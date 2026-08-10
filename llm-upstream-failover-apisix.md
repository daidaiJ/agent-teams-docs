# APISIX LLM 上游端点错误容忍与自动切换机制

> 文档版本: v1.0
> 日期: 2026-08-10
> 基线代码: apisix master（`a2422829`，含 ai-proxy-multi v0.5）
> 关联文档: [llm-upstream-failover-higress-design.md](llm-upstream-failover-higress-design.md)（Higress 侧方案设计）

---

## 1. 插件总览

错误容忍/自动切换的核心是 **`ai-proxy-multi`** 插件（`apisix/plugins/ai-proxy-multi.lua`，priority 1041，version 0.5），它扩展 `ai-proxy`（`apisix/plugins/ai-proxy.lua`），把多个 LLM 端点（不同 provider / model / endpoint）组织成实例池：

| 维度 | 机制 | 配置位置 |
|------|------|----------|
| 实例定义 | `instances[]`（provider、override.endpoint、auth、options） | `ai-proxy-multi` schema |
| 实例调度 | `instances.priority`（优先级，高者先用）+ `instances.weight`（同级轮询权重） | 同上 |
| 节点解析 | 每实例 endpoint 做 DNS 解析出多节点（`_dns_nodes`），随机挑选 | `resolve_endpoint`（ai-proxy-multi.lua:372） |
| 健康检查 | 每实例可配 `checks.active` 主动探测（复用 APISIX healthcheck 基础设施） | `instances.checks` |
| 请求级容错 | `fallback_strategy`（http_429 / http_5xx / rate_limiting）+ `max_retries` + `retry_on_failure_within_ms` | 插件顶层属性 |
| 负载均衡算法 | `balancer.algorithm`：roundrobin（默认）/ chash / semantic | `balancer` |

容错分三层：**节点级**（DNS 多节点 + 健康过滤）、**实例级**（优先级/权重 + 健康/配额过滤选实例）、**请求级**（429/5xx/配额触发重试换实例）。

---

## 2. 第一道防线：选实例时的健康过滤（access 阶段）

入口 `_M.access`（ai-proxy-multi.lua:885）→ `pick_ai_instance`（:870）→ `pick_target`（:594）。

### 2.1 server picker 构建与缓存

- `create_server_picker`（:560）：按 `balancer.algorithm` 加载 `apisix.balancer.<algo>`，实例集按 priority 分组后交给 `apisix.balancer.priority`（多优先级）或直接单层 picker；
- picker 缓存在 `lrucache_server_picker`，**cache key = route key + conf version**（`plugin.conf_version`），配置变更立即失效；
- 非 chash 算法在构建 picker 时即做健康过滤：`fetch_health_instances`（:516）只把健康实例放入候选集。

### 2.2 健康检查接入

- 每个配置了 `checks` 的实例通过 `healthcheck_manager.fetch_checker(resource_path, resource_version)` 获取 checker，`resource_path` 形如 `route#plugins['ai-proxy-multi'].instances[i]`（i 为 0 基索引）；
- `create_health_status`（:447）对该实例每个 DNS 节点调用 `healthcheck_manager.fetch_node_status(checker, host, port, host_header)`，只保留健康节点为 `_healthy_dns_nodes`；
- **全挂 fail-open**：所有实例均无健康节点时返回 `{all_unhealthy=true}`，回退为"全量可用"，不直接 503（避免把实例永久打死）。

### 2.3 选实例循环（跳过不健康/限流实例）

```
for _ = 1, #conf.instances do
    instance_name, err = server_picker.get(ctx)     -- 按 priority 从高到低
    if not health_status or health_status[instance_name] then
        if not check_rate_limiting or ai_rate_limiting.check_instance_status(...) then
            break                                    -- 选中
        end
    end
    server_picker.after_balance(ctx, true)           -- 跳过当前实例
end
```

- `server_picker.get` 内部（`apisix/balancer/priority.lua`）：从 `priority_balancer_picker_idx` 开始按优先级取，某优先级内 `picker.get` 失败则 `before_retry_next_priority` 重置状态换下一优先级；
- `fallback_strategy` 为 `rate_limiting` / `instance_health_and_rate_limiting` 时额外检查 `ai-rate-limiting.check_instance_status`（ai-rate-limiting.lua:316）：实例 token 配额耗尽视为不可用，**无视优先级**切到低优先级实例；
- 全部尝试完 → 返回 `503 all servers tried`。

### 2.4 注意点

- **chash 不参与健康过滤**（`create_server_picker` 中 chash 走 `fetch_all_instances`），为保持一致性哈希稳定；
- 健康快照缓存 `lrucache_health_status` 的 key 额外拼接 `get_health_status_ver`（每实例 `nodes_ver` + checker `status_ver`），健康状态翻转、DNS 变化立即失效。

---

## 3. 第二道防线：请求失败后的自动切换（before_proxy 阶段）

### 3.1 重试循环

`_M.before_proxy`（ai-proxy-multi.lua:1030）→ `base.before_proxy`（ai-proxy/base.lua:179）是 **`while true` 循环**：

```
while true do
    按 ctx.picked_ai_instance 构造请求（协议转换、鉴权、body）
    发送（transport_http，cosocket）
    if 连接失败/超时 → 返回错误码 → retry_on_error 判断
    if 429 / 5xx → 读 upstream error body → retry_on_error 判断
    else → 正常响应（流式/非流式）
end
```

每个失败分支都会调用 `on_error` 回调（即 `retry_on_error`）；返回 nil 表示"已切换实例，继续循环"。

### 3.2 切换判定 `retry_on_error`（ai-proxy-multi.lua:915）

| 条件 | 行为 |
|------|------|
| `429` 且 `fallback_strategy` 含 `http_429` | 可重试 |
| `500 <= code < 600` 且含 `http_5xx` | 可重试 |
| 超过 `retry_on_failure_within_ms`（从 `ctx.llm_request_start_time` 度量，base 每次尝试前重置） | **不重试**，直接返回该 code（慢失败保护，避免 fallback 让客户端等两倍时间） |
| 超过 `max_retries`（`ctx.ai_retries` 计数） | 不重试，返回该 code |
| 重试上限内 | `pick_ai_instance` 换实例，更新 `ctx.balancer_ip / picked_ai_instance`，返回 nil 继续循环 |
| `server_picker` 不存在（如单实例） | 原样返回 code，不重试 |

失败实例的 error body 只记 error log（后续尝试的响应会替代它发给客户端）；**不匹配策略的失败（如普通 4xx）直接透传 upstream status + error body 给客户端**，并保留 upstream Content-Type。

### 3.3 触发条件与代码位置

- 连接错误/超时：`base.lua` do_request 中 `transport_http.request` 失败 → `transport_http.handle_error` 返回码；
- 429/5xx：`base.lua` 在响应头到达后、body 转发前拦截，先 `read_upstream_error_body`（base.lua:64）再返回状态码；
- **重试窗口只覆盖响应头到达之前**；流式响应中途断流不切换（由 `max_stream_duration_ms` / `max_response_bytes` 兜底截断）。

---

## 4. 健康状态闭环

```
请求结果 → apisix_upstream.push_upstream_state（被动数据源）
        + instances.checks.active 主动探测（默认 http_path="/"，不健康状态码 [429,404,500-505]，连续失败 5 次判死，成功 2 次复活）
                    ↓
        resty.healthcheck（lua-resty-healthcheck）+ "upstream-healthcheck" 共享字典
                    ↓
        checker.status_ver 递增 → lrucache（picker / 健康快照）key 失效
                    ↓
        后续请求不再选到坏实例/坏节点（"自动切换"）
```

- 主动探测配置经 `_M.construct_upstream`（ai-proxy-multi.lua:969）合入实例 auth header/query（探测请求也要带鉴权）；
- 健康检查对 OpenAI/DeepSeek/AIMLAPI 等无官方健康端点的 provider 意义有限，文档建议用于 `openai-compatible` 自定义端点。

---

## 5. 其他切换路径

| 路径 | 触发条件 | 行为 | 参与健康/重试？ |
|------|----------|------|-----------------|
| `semantic` 算法（`pick_semantic_instance`:773） | prompt 与实例 `examples` 的嵌入相似度排序，首超阈值者胜出 | 无实例过阈值 / embedding 失败 / 维度不匹配 → fail-open 到 `semantic_opts.fallback`（默认第一个实例） | **不参与**——上游失败直接返回客户端 |
| 单实例模式（`#instances == 1`） | 无 | 直接使用该实例；容错仅剩节点级（DNS 多节点随机 + 健康过滤） | 部分 |
| `rate_limiting` fallback | 高优先级实例配额耗尽 | 跳过当前实例，无视优先级切低优先级实例 | 配额维度 |

---

## 6. 关键设计取舍

| 机制 | 行为 | 理由 |
|------|------|------|
| 全挂 fail-open | 所有实例不健康仍走默认实例集，不直接 503 | 避免误判把实例永久打死；宁可试错 |
| 慢失败不重试 | 超 `retry_on_failure_within_ms` 直接回客户端 | 失败已让客户端等待，fallback 会让等待翻倍 |
| 缓存版本化 | cache key = route + conf version + `status_ver` + `nodes_ver` | 配置/健康/DNS 变化秒级生效，无需清缓存 |
| 重试只发生响应头前 | 流中失败不切换 | 流已启动无法无损重发 |
| 健康按节点而非实例 | 实例 = DNS 节点集，逐节点判活 | 一个端点多 IP 时容忍单 IP 故障 |

---

## 7. 底层实现说明（非 Envoy）

APISIX 基于 **OpenResty（Nginx + LuaJIT）**，插件为纯 Lua 运行在 Nginx worker 内，与 Envoy/WASM/ext_proc 无关：

| 能力 | 实现 |
|------|------|
| 发请求到 LLM 端点 | `lua-resty-http` cosocket（`ai-proxy/transport_http.lua`） |
| 主动健康检查 | `resty.healthcheck`（healthcheck_manager.lua:97）+ `upstream-healthcheck` 共享字典跨 worker 同步 |
| 被动健康 | 每请求 `push_upstream_state` 写入同一套状态 |
| 选实例/跳过/重试 | 纯应用层 Lua 逻辑（`pick_target` 循环 + `base.before_proxy` while 重试） |

决策全部进程内完成，无外部 RPC 跳数；`while true` 同步式重试循环依赖 cosocket 的同步语义，这是与 Envoy WASM（异步回调）模型最大的结构性差异。

---

## 附录：关键配置项速查

| 配置 | 默认 | 说明 |
|------|------|------|
| `fallback_strategy` | 无 | `http_429` / `http_5xx` / `rate_limiting`（或组合数组） |
| `max_retries` | 无（试完为止） | 单请求最多换实例数 |
| `retry_on_failure_within_ms` | 无（不设限） | 慢失败保护阈值 |
| `instances.priority` | 0 | 高优先先用 |
| `instances.weight` | 0 | 同级轮询权重 |
| `instances.checks.active.*` | 见文档 | 主动探测间隔/状态码/判死复活阈值 |
| `balancer.algorithm` | roundrobin | roundrobin / chash / semantic |
| `timeout` | 30000ms | LLM 请求超时（按 socket 操作计） |
