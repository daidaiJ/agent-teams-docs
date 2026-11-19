# agentgateway 扩展路径与凭证注入最佳实践

> 文档版本: v1.0
> 日期: 2026-11-19
> 范围: agentgateway（Apache-2.0，Rust 数据面 + Go 控制面）在「按 host + path 前缀注入上游凭证」场景下的扩展点选型、组合方式与落地约束
> 证据: 官方文档 standalone / kubernetes latest（2026-11-19 抓取：ExtProc、External authorization、UI）；未做部署验证与源码级核对，PoC 清单见第 9 节
> 关联: OpenSandbox `wiki/opensandbox-credential-gateway-industry-designs.md`、`wiki/opensandbox-external-egress-control-plane-and-credentials.md`、本目录 `llm-upstream-failover-higress-design.md`

---

## 1. 结论

1. 自定义逻辑的官方通道是**外部服务**：ExtProc（Envoy `ext_proc` 兼容）与 extAuthz（Envoy External Authorization 兼容）。CEL 负责纯表达式变换，不承担 Redis/PG 访问。
2. 凭证注入的默认组合是 **CEL 剥离 + ExtProc 注入**，ExtProc 只开请求头阶段。需要更低开销时换成 **CEL 剥离 + extAuthz**（gRPC 可带缓存，HTTP 无缓存）。
3. `Authorization` 保持**单一写者**。同时配 ExtProc、extAuthz、backendAuth 会引入未文档化的覆盖顺序，属于 PoC 项，不进入生产配置。
4. 拒绝语义用 `ImmediateResponse`。外部服务正常关闭 gRPC 流会被当作「处理完成」，请求继续发往上游，`failClosed` 也不会拦截。
5. 凭证服务只做「查表 + 解密 + 回注 header」，存储分层为 PG 真相源、Redis 热路径、服务内 1–5s 本地缓存。

## 2. 扩展点盘点

### 2.1 ExtProc

| 项目 | 事实 |
|---|---|
| 协议 | Envoy `ext_proc` gRPC 服务，双向流，每 exchange 一条 |
| 能力 | 读取并修改请求/响应的 headers、body、trailers；可终止请求 |
| 挂载点 | Listener、Route；支持 `conditional` 按请求选择处理器 |
| 默认阶段 | 请求头、响应头、请求 trailer、响应 trailer 全部发送；请求/响应 body 默认全双工流式 |
| 失败模式 | `extProc.failureMode`：`failClosed`（默认）/ `failOpen` |
| 建流超时 | 默认 10s，`extProc.policies.http.requestTimeout` 可改 |
| body 缓冲 | `buffered` 超过网关缓冲上限（默认 2 MiB）报错；`bufferedPartial` 只发前缀 |
| 拒绝 | `ImmediateResponse` 支持请求头、请求体、响应头、响应体四个阶段 |

`processingOptions` 默认值与取值：

| 字段 | 默认 | 取值 |
|---|---|---|
| `requestHeaderMode` | `send` | `send` / `skip` |
| `responseHeaderMode` | `send` | `send` / `skip` |
| `requestBodyMode` | `fullDuplexStreamed` | `none` / `buffered` / `bufferedPartial` / `fullDuplexStreamed` |
| `responseBodyMode` | `fullDuplexStreamed` | 同上 |
| `requestTrailerMode` | `send` | `send` / `skip` |
| `responseTrailerMode` | `send` | `send` / `skip` |
| `allowModeOverride` | `false` | `true` / `false` |

纯 header 注入的最小配置：

```yaml
routes:
- backends:
  - host: api.example.com
    port: 443
  policies:
    extProc:
      host: "127.0.0.1:9000"
      failureMode: failClosed
      processingOptions:
        requestHeaderMode: send
        responseHeaderMode: skip
        requestBodyMode: none
        responseBodyMode: none
        requestTrailerMode: skip
        responseTrailerMode: skip
```

### 2.2 extAuthz

| 项目 | 事实 |
|---|---|
| 协议 | gRPC（Envoy External Authorization，API 兼容）或 HTTP |
| 决策语义 | gRPC 返回 OK/denied；HTTP 2xx 放行，非 2xx 拒绝 |
| header 注入 | gRPC 的响应 header 修改由 agentgateway 以 `insert` 应用；HTTP 用 `protocol.http.includeResponseHeaders` 复制到后端请求 |
| 请求头透传 | gRPC 未设 `includeRequestHeaders` 时发送全部请求头；HTTP 默认只发 `Authorization` |
| 请求体 | `includeRequestBody.maxRequestBytes` 默认 8192，可 `allowPartialMessage` |
| 缓存 | **仅 gRPC**：`cache.key`（1–16 个 CEL）、`cache.ttl`、`cache.maxEntries`（默认 10000）；HTTP 每次都调用 |
| 超时 | 默认 2s，`policies.http.requestTimeout` 可改 |
| 挂载点 | Listener、Route、Backend；backend 级在选中后端之后执行，官方定位就是「插入 token」这类改写 |

```yaml
extAuthz:
  host: credential-service:9001
  includeRequestHeaders:
    - x-client-id
    - ":authority"
    - ":path"
  protocol:
    grpc: {}
  cache:
    key:
      - request.headers["x-client-id"]
      - request.headers[":authority"]
      - request.path
    ttl: '"2s"'
    maxEntries: 20000
```

### 2.3 CEL transformation

- 纯表达式，作用于请求上下文与 JWT claims，不发起网络或存储调用。
- 适合：剥离客户端同名头、header 改名、静态映射、条件判断、构造 extAuthz 的 `path`、拼 `cache.key`。
- 不适合：查 Redis/PG、解密、调用 KMS。

### 2.4 backendAuth

内建的上游认证：静态 key、passthrough、GCP/AWS/Azure 凭证、Signed JWT、OAuth token exchange（RFC 8693、RFC 7523、Entra OBO）。这是**交换模型**：请求到来时换取短期 token。适合上游是标准 OAuth 资源服务器、且不需要按 subject 查自定义注册表的场景。

### 2.5 配置入口与 UI

- standalone：UI 可编辑路由、模型、MCP server、虚拟 API key，写本地配置文件或内置 DB。
- Kubernetes：UI 只读，`agentgateway-proxy` 的 15000 端口，`kubectl port-forward` 访问，用于核对 xDS 下发的配置。
- UI 不是凭证存储：虚拟 API key 面向 LLM 客户端，与上游凭证注入是两套对象；没有 RBAC、审计与多租户隔离。

## 3. 凭证注入的三档模式映射

| 档位 | 语义 | agentgateway 落点 |
|---|---|---|
| L1 静态 | 固定 header 或固定上游 key | `backendAuth.key`、CEL 静态映射 |
| L2 查表 | 按 `(subject, host, path)` 从 Redis/PG 取密文、解密、注入 | **ExtProc**（首选）或 extAuthz |
| L3 交换 | 每请求换短期 token | `backendAuth.oauthTokenExchange`、云 STS |

OpenSandbox 的注册式租约属于 L2：凭证在有效期内不变，替换发生在注册/轮换时刻，不需要每请求换 token。L2 的数据面就是本文的重点。

## 4. 推荐组合

| 需求 | 组合 | 说明 |
|---|---|---|
| 每请求查表并注入一个 header | **CEL + ExtProc（header-only）** | 语义明确，无缓存隐藏状态，可 `ImmediateResponse` |
| 同上，允许 1–5s 决策缓存 | **CEL + extAuthz（gRPC + cache）** | 更轻，缓存内建；HTTP 模式无缓存 |
| 只做剥离、改名、静态映射 | **CEL 单用** | 零 I/O |
| 响应擦除、body 改写、动态选后端 | **ExtProc** | 唯一覆盖响应与 body 阶段的扩展点 |
| 上游是标准 OAuth 资源服务器 | **backendAuth** | 内建交换，无需外部服务 |

组合原则：

1. **单写者**：`Authorization` 只由一个环节写。默认让 ExtProc 写；不要同时让 extAuthz 和 ExtProc 写同一个 header。
2. **阶段裁剪**：只做注入时，body 与 trailer 全部 `none`/`skip`，避免 2 MiB 缓冲和流式转发成本。
3. **剥离在前**：CEL 先删除客户端自带的 `Authorization`、`X-Api-Key`、`Cookie`，再交给外部服务注入，避免双 token 与「insert 还是覆盖」的歧义。
4. **拒绝用 ImmediateResponse**：返回稳定错误码（`CRED_NOT_FOUND` / `CRED_EXPIRED` / `CRED_REVOKED` / `SUBJECT_MISMATCH`），响应体不含任何密钥材料。

## 5. 凭证服务设计

### 5.1 请求链路

```
sandbox → egress（L3/L4）→ agentgateway（host + path 前缀）
  │ 1. CEL：剥离 authorization / x-api-key / cookie
  │ 2. ExtProc gRPC：请求头阶段
  │      ├─ 读取 :authority、:path、:method、x-client-id
  │      ├─ 本地 LRU → Redis → PG（最长前缀匹配）
  │      ├─ KMS 解封 DEK → AES-GCM 解密（仅内存）
  │      └─ 返回 header mutation：set authorization
  │ 3. 转发上游
  └ 4. 可选：响应头阶段擦除上游回显的凭证
```

### 5.2 查找键与身份来源

| 模型 | 身份键 | 适用 |
|---|---|---|
| 路由即身份 | 每条 route 固定 subject；凭证服务只按 `(host, path)` 查 | 每客户端独立域名/前缀，配置量随客户端线性增长 |
| 共享域名 | `x-client-id`（沙箱注入）+ `(host, path)` | 高基数，网关配置保持静态 |
| 强身份 | mTLS 身份或 JWT `sub` + `(host, path)` | 需要不可伪造的主体 |

共享域名方案依赖 egress 侧或 taskTemplate 注入的 `x-client-id`。若要求不可伪造，用 mTLS 或 JWT；源 IP 在 NAT 与池化场景下不稳定，只作为辅助信号。

### 5.3 匹配规则

- `host` 精确匹配，统一小写、去掉默认端口。
- `path_prefix` 按最长前缀命中；同长度时拒绝注册，避免歧义。
- 方法、scheme、port 作为可选条件，命中规则与 OpenSandbox `CredentialBinding` 保持一致。
- 未命中不回落默认凭证，直接拒绝。

### 5.4 存储分层

| 层 | 内容 | 失效方式 |
|---|---|---|
| PG | 真相源：绑定、密文、`expires_at`、`revoked_at`、`revision`、`retire_at` | 管理 API 写入 |
| Redis | 热路径：`cred:{subject}:{host}:{prefix}`，值为密文 + 元数据，`EX` 对齐 `expires_at` | TTL 自然过期 + pub/sub 主动失效 |
| 服务内 LRU | 1–5s 短缓存，按 `revision` 作 key；负缓存 ≤ 1s | 进程内 TTL |

PG 只在缓存 miss 时回源。高基数下避免每请求查 PG；`(subject, host)` 建索引，`expires_at` 建部分索引供 reaper 扫描。

### 5.5 TTL 与轮换

- 到期判断在服务端完成：`now >= expires_at` 或 `revoked_at IS NOT NULL` 一律拒绝，不依赖缓存 TTL。
- 轮换采用 revision 重叠：新 revision 立即生效，旧 revision 设 `retire_at = now + 重叠窗口`（建议 5 分钟，覆盖 in-flight 请求），到期后退役。
- 撤销写 `revoked_at`，发布 Redis 失效消息；缓存 TTL 决定撤销的最大可见延迟，建议 ≤ 5s。
- extAuthz 的 `cache.ttl` 必须钳到凭证剩余生命周期的较小值，或直接关闭缓存；gRPC 缓存键无法包含凭证 revision。

### 5.6 加密与密钥

- 信封加密：每条绑定一个 DEK（或每租户一个），KEK 放 KMS；PG 存 `ciphertext`、`nonce`、`key_id`。
- 网关不持密钥，只看到最终注入的 token。
- 凭证服务用独立 ServiceAccount，只读自己的 Secret 与 KMS 权限。
- 日志、trace、错误体只记录 fingerprint 与 `revision`。

## 6. 部署与安全

| 项 | 建议 |
|---|---|
| 拓扑 | 凭证服务与 agentgateway 同 Pod sidecar，或同节点 DaemonSet + localhost；跨节点会增加一跳与 TLS 配置面 |
| 网络 | NetworkPolicy 只放行网关 → 凭证服务的 gRPC 端口；凭证服务不暴露 Ingress |
| 连接 | 同 Pod 用 localhost；跨 Pod 用 mTLS，SNI 固定服务名 |
| 超时 | ExtProc 建流超时与 extAuthz 调用超时都显式设置（建议 1s），小于上游请求超时 |
| 资源 | 每请求一跳 gRPC，凭证服务 2C4G、本地命中率 > 99% 时可支撑数千 QPS；以压测为准 |
| 灰度 | 先按 route 灰度，再按 host 扩大；保留 `failClosed` 的快速回滚开关（临时切 `failOpen` 需安全审批） |

## 7. 可观测性

最小指标集：

- `extproc_requests_total{result}`、`extproc_duration_seconds`（p50/p95/p99）
- `cred_lookup_total{layer="local|redis|pg", result}`
- `cred_rejected_total{reason}`
- `cred_remaining_seconds` 直方图（在注入时记录）
- `cred_reaper_last_success_timestamp`

告警建议：p99 外部调用 > 20ms、拒绝率突增、24h 内到期凭证数超阈值、reaper 超过两个周期未成功。

## 8. 反模式

1. 用 CEL 查 Redis/PG 或调 KMS —— 表达式语言没有 I/O。
2. 同时配 extAuthz 与 ExtProc 写同一个 `Authorization` —— 覆盖顺序未文档化。
3. 用「关闭 gRPC 流」表示拒绝 —— 网关视为处理完成并放行。
4. `cache.key` 漏掉决策输入（如 `x-client-id`、path）—— 跨主体复用决策。
5. `cache.ttl` 长于凭证剩余 TTL —— 过期或撤销后仍放行。
6. 每请求直查 PG —— 高基数下连接池与 p99 双崩。
7. 把 standalone UI 当凭证管理台 —— 无 RBAC、审计、多租户与加密存储。
8. 让网关持有 KEK 或落盘明文 —— 数据面被攻破即全量泄露。

## 9. PoC 验收清单

| # | 验证项 | 通过标准 |
|---|---|---|
| 1 | header-only ExtProc | 日志确认身阶段只发 header，body 不经过服务 |
| 2 | 剥离 + 注入 | 客户端自带 `Authorization` 不可达上游，上游只收到一个 token |
| 3 | fail-closed | 停掉凭证服务，请求返回 5xx/403，不落到上游 |
| 4 | 拒绝语义 | 未注册 host/path 返回 `ImmediateResponse` 403，错误码稳定 |
| 5 | 过期与撤销 | 过期/撤销后 ≤ 1s 网关侧拒绝（含缓存失效） |
| 6 | 轮换重叠 | 重叠窗口内新旧 revision 均可命中，退役后旧 token 不再注入 |
| 7 | 顺序 | ExtProc 与 backendAuth 共存时的覆盖顺序（仅验证，不默认启用） |
| 8 | 响应擦除 | 上游回显的凭证在响应头阶段被移除 |
| 9 | 高基数 | 1000 subject × 10 host 下本地命中率 > 99%，p99 增量可接受 |
| 10 | 超时 | 建流超时与请求超时的组合下无悬挂连接 |

## 10. 与 Higress / Envoy 的对照

| 维度 | agentgateway ExtProc | agentgateway extAuthz | Higress ext-auth | Envoy ext_authz |
|---|---|---|---|---|
| 协议 | Envoy `ext_proc` gRPC | Envoy External Authorization / HTTP | HTTP，回传 header 白名单 | Envoy External Authorization |
| 注入方式 | header mutation | header `insert` / `includeResponseHeaders` | `allowed_upstream_headers` | header mutations |
| 响应改写 | 支持 | 不支持 | 需 Wasm | 不支持 |
| body 访问 | 支持（可选阶段） | 仅请求体可选 | 需 Wasm | 不支持 |
| 内建缓存 | 无 | gRPC 有 | 无 | 无 |
| 数据面 | Rust | Rust | Envoy | Envoy |

凭证服务的外部契约在 extAuthz 与 Higress ext-auth 之间高度相似（一次调用 + 回传 header），迁移成本主要在配置层。若只需要通用 HTTP 注入且已有 Higress 体系，Higress 更省运维；若同一条数据面还要承载 LLM/MCP 流量，agentgateway 的统一策略模型更有价值。

## 11. 未核实项

- ExtProc 与 `backendAuth` 的执行顺序与覆盖关系，官方文档未写明。
- extAuthz 的 gRPC 缓存是否连同 header 注入结果一起缓存（文档只描述「授权结果」）。
- `cache.ttl` 的 CEL 表达式能否读取鉴权响应写入的 header；若可以，可让凭证服务回传剩余 TTL 自动钳制。
- ExtProc 跨节点部署时的 TLS 配置字段，需对照配置参考。
- 高并发下的 p99 与连接复用行为，需按实际拓扑压测。
