# Higress LLM 上游容错方案设计（Envoy Go HTTP Filter 路线）

> 文档版本: v1.0
> 日期: 2026-08-10
> 关联文档: [llm-upstream-failover-apisix.md](llm-upstream-failover-apisix.md)（APISIX 基线机制研究）
> 参考示例: [envoyproxy/examples golang-http](https://github.com/envoyproxy/examples/tree/main/golang-http)（Envoy Go HTTP filter 挂起/恢复模式）
> 设计目标: 在 Higress（Envoy 数据面）上实现与 APISIX `ai-proxy-multi` 等价的错误容忍/自动切换语义

---

## 1. 目标语义（对照 APISIX ai-proxy-multi）

| 能力 | APISIX 实现 | 本文方案 |
|------|-------------|----------|
| 多端点（provider/模型）实例池 | `instances[]` | providers 配置 |
| 优先级 + 权重调度 | `priority.lua` + roundrobin | Go filter 内实现（Envoy LB 不支持实例级优先级） |
| 429/5xx 失败重试换实例 | `retry_on_error`（响应头前） | Go filter 重试循环（响应头前） |
| 慢失败保护 | `retry_on_failure_within_ms` | 同语义 |
| 重试上限 | `max_retries` | 同语义 |
| 配额耗尽降级 | `rate_limiting` fallback | 插件内配额计数 |
| 健康过滤 | healthcheck + status_ver | 见第 4 节（两级分工） |
| 流式 | 响应头前可重试，流中不切 | 同取舍（见第 5 节） |

---

## 2. Higress 现状盘点（可复用的部分）

### 2.1 架构定位

- Higress = Envoy 数据面 + Istio 控制面；插件体系为 **proxy-wasm**（Go SDK `higress-group/wasm-go`，TinyGo 编译，VM 每 worker 线程一个，**无 goroutine/阻塞**，异步回调模型）；
- 外部 LLM 端点通过 **McpBridge**（DNS 类型）注册为 Envoy cluster（指标可见 `outbound|443||qwen.dns` 命名）；
- 每个 provider cluster 可挂 Envoy 原生 active health check / outlier detection / priority（需控制面支持或 EnvoyFilter patch）。

### 2.2 现有 ai-proxy 插件的能力与缺口

- 已支持 `providers[]` 多提供商 + `activeProviderId` 切换 + API Token failover（`plugins/wasm-go/extensions/ai-proxy/config/config.go` 的 `SetApiTokensFailover`）；
- 已支持 failover 触发配置（社区 issue #3531 佐证 `failoverOnStatus` 可配 400/429 触发切换）；
- **已知缺陷**：failover 重发请求时 body 丢失（`Content-Length: 0`，issue #3531，v2.1.9 未修复）——重试必须自持 body；
- **能力缺口**：多实例优先级/权重轮询、max_retries、慢失败保护、配额感知降级均无。

---

## 3. 方案选型对比

| 方案 | 开发量 | 能力对齐度 | 主要坑 |
|------|--------|-----------|--------|
| A. 现成 ai-proxy 插件（failoverOnStatus） | 零 | 低（仅主备 + 状态码） | body 丢失 bug 未修；无重试上限/慢失败保护 |
| B. WASM 插件自研 | 高 | 中 | 重试循环须改异步回调状态机；跨 worker 健康状态要 shared queue 聚合（VM 每线程一份，共享数据不跨线程一致） |
| **C. Envoy Go HTTP filter（推荐）** | 中 | 高（近乎 1:1 移植） | 需数据面镜像启用 `envoy.filters.http.golang`（Higress 默认未必开启，见第 6 节） |
| D. ext_proc 外部决策 | 中 | 中 | 每请求多一跳 RPC；流式需 buffering 模式；决策服务自己实现容错 |

---

## 4. 推荐方案：两层容错

### Level 1 — Envoy 原生机制（节点级，零代码）

每个 provider 一个 cluster（McpBridge 注册），配：

| 机制 | 配置 | 作用 |
|------|------|------|
| active health check | cluster `health_check`（HTTP 探测） | 主动剔除坏节点 |
| outlier detection | `outlier_detection`（consecutive_5xx / success_rate / base_ejection_time） | 被动剔除；**主线程集中决策，跨 worker 一致** |
| priority failover | 单 cluster 内 priority 0 = 主 provider host、priority 1 = 备 provider host | 主 provider 全部不可用（健康检查/驱逐）时自动切备 |

局限：只在"节点全挂/被驱逐"时切换；**429 配额耗尽、单 provider 部分降级**触发不了 → 需要 Level 2。

### Level 2 — Go HTTP filter（实例级，对应 golang-http 模式）

`envoy.filters.http.golang` 插件，进程内 .so，**真 Go runtime**（goroutine / mutex / channel 可用）。核心流程：

```
DecodeData(endStream=true)          // 缓冲完整请求 body（重试重发的唯一来源）
    └─ return api.Running           // 挂起
        └─ go func() {              // 后台 goroutine（示例推荐模式）
            defer RecoverPanic()
            for attempt := 0; ; attempt++ {
                p := pickProvider(providers)   // priority → weight 轮询 → 健康/配额过滤
                if p == nil { SendLocalReply(503, "all providers failed"); return }
                t0 := now()
                resp, err := http.Call(p.Cluster, buildHeaders(p), body, p.TimeoutMs)
                if err != nil { recordFailure(p); /* 连接失败/超时视为 5xx */ }
                else if contains(FallbackOnStatus, resp.StatusCode) {
                    recordFailure(p)            // 429/5xx
                } else {
                    SendLocalReply(200, resp.Body); return   // 成功
                }
                if attempt >= MaxRetries || now()-t0 > RetryOnFailureWithinMs {
                    SendLocalReply(resp.StatusCode, resp.Body); return
                }
                // 否则换下一个 provider，继续循环
            }
        }()
```

关键点：

- **`http.Call(cluster, headers, body, timeout)` 为阻塞式调用**（返回完整响应头+体），等价 APISIX cosocket 同步语义 → `retry_on_error` 逻辑可 1:1 移植；
- **跨 worker 状态直接用 Go 全局变量 + mutex**（.so 单实例加载，全局可见），对比 WASM 的 shared queue 聚合是结构性优势：

```go
var healthState = struct {
    sync.Mutex
    failures map[string]*slidingWindow   // 每 provider 失败滑动窗口（被动健康）
    probes   map[string]*probeResult     // 主动探测结果（goroutine ticker 维护）
    quota    map[string]*tokenBucket     // 配额余量（rate_limiting 降级）
}{}
```

- 主动探测：插件内 `go func(){ for range ticker { http.Call(healthProbe...) } }()`，无需 OnTick（那是 WASM 的限制）；
- 配置结构：

```go
type ProviderConfig struct {
    Name        string            // 唯一 ID
    Cluster     string            // McpBridge 定义的 provider cluster
    Priority    int               // 高优先先用完才用低优先级
    Weight      int
    Model       string
    AuthHeader  string
    TimeoutMs   int
    HealthProbe *HealthProbeConfig // 可选主动探测
    Quota       *QuotaConfig       // 可选配额，耗尽即不可用
}
type PluginConfig struct {
    Providers          []ProviderConfig
    FallbackOnStatus   []int   // [429, 500, 502, 503, 504]
    MaxRetries         int     // 0 = 试完所有 provider
    RetryOnFailureWithinMs int // 慢失败不重试
    MaxBodySize        int     // body 缓冲上限
}
```

---

## 5. 流式（SSE）响应策略

`http.Call` 缓冲完整响应体，**不适用于 streaming**。两条路线：

| 模式 | 做法 | 代价 |
|------|------|------|
| 透传模式（流式请求用） | `DecodeHeaders` 选好 provider 后改写 `:authority`/`:path` 指向对应 cluster，`api.Continue` 交给 Envoy router | 流式零缓冲零延迟；但 **429/5xx 无法跨 provider 切换**，只靠 Level 1（outlier/priority） |
| 混合模式（推荐） | `stream: false` 走 goroutine 重试循环；`stream: true` 走透传 + router `retry_on: retriable-status-codes`（同 cluster 重试）+ Level 1 兜底 | 与 APISIX 取舍一致：**重试只在响应头到达前**，流中失败不切换 |

流式判断需先读 body：在 `DecodeData(endStream=true)` 统一决策。

---

## 6. 落地路径与坑

| 坑 | 说明 |
|----|------|
| **Go filter 在 Higress 默认不可用** | `envoy.filters.http.golang` 是 Envoy contrib 扩展；需确认 Higress 数据面镜像（higress-group/envoy 分支）是否编译启用，未启用需自编译镜像——**此路最大成本** |
| 重试必须自持 body | Higress ai-proxy 的教训（issue #3531：failover 时 `Content-Length: 0`）；Go filter 里 `http.Call` 的 body 必须用自己缓冲的那份，且受 `MaxBodySize` 约束 |
| `SendLocalReply` 不能用于流式 | 本地回复只能是完整 body；流式必须走 router 透传 |
| goroutine 生命周期 | `OnDestroy` 后不可再访问 `callbacks`（示例注释明确警告）；goroutine 内必须 `RecoverPanic` |
| cluster 必须预存在 | `http.Call` 按 cluster 名寻址，provider cluster 需先经 McpBridge/EnvoyFilter 定义 |

---

## 7. 结论与建议

- **只做主备 + 状态码容错、不动数据面**：用现成 ai-proxy 插件 `providers` + failover（等官方修 body 缓冲 bug，或自行打补丁）；
- **要完整语义、可掌控数据面镜像**：**Go HTTP filter（第 4 节方案）**——代码可对照 `ai-proxy-multi.lua` 的 `pick_target`（:594）+ `retry_on_error`（:915）直接移植，开发量最低、语义对齐度最高；
- **不动数据面镜像但要完整语义**：WASM 插件，需将重试循环改为异步回调状态机（挂起 → dispatch → 回调内再 dispatch），健康状态用 shared queue + 主线程单例聚合，开发量最大；
- 无论哪条路线，**Level 1（Envoy 原生 health check + outlier detection + priority failover）都建议先落地**：零代码拿到节点级容错，插件只负责实例级语义。
