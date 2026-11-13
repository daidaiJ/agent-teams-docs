# Kubernetes 渐进式发布：从 5xx 检测到安全回滚的 runbook

> 文档版本: v1.0
> 日期: 2026-11-13
> 状态: Draft
> 适用范围: Deployment / Argo Rollouts / Flagger 这类滚动或金丝雀发布；HTTP/gRPC 无状态服务为主

---

## 0. 结论

- **就绪探针通过只证明"进程活着、端口能连"**，它是可接流量的代理指标，不证明业务链路可用。真实请求的正确性必须由**入口侧合成探测 + 金丝雀真实流量分析**来守。
- **部署完成 ≠ 发布成功**：发布的最后一个门是"观察窗口内 SLO 未劣化"。观察窗口要覆盖 10–15 分钟，覆盖延迟爆发型故障（连接池耗尽、缓存/证书过期、GC、下游限流）。
- **止血顺序**：先停扩散（pause/abort），再回滚（按 digest），最后查因；数据库迁移必须 expand/contract，保证"回滚应用"不依赖"回滚数据"。
- **滚动更新不会中断已建立的连接**，但 Endpoints 摘除是异步的、SIGTERM 与摘流量存在竞态；靠 `preStop` 排空、`terminationGracePeriodSeconds`、graceful shutdown 与 GOAWAY 覆盖。

---

## 1. 现场：部署 5 分钟后 5xx + 延迟 +300%

这类"先绿后炸"的形态通常是延迟爆发（连接池打满、缓存/证书过期、内存泄漏、下游限流），而不是启动即失败。检测按信号优先级：

| 信号 | 看什么 | 为什么 |
|---|---|---|
| 入口 SLO | 5xx 比例（按 route + version 分组）、p99、超时率 | 全局均值会被健康旧副本拉平，必须分版本对比 |
| 合成探测 | 从集群外经 Ingress 跑带鉴权的事务（登录→读→写），30–60s 一次 | 唯一能覆盖"探针过、真实请求挂"的手段 |
| 依赖水位 | DB 连接池、下游超时、队列积压 | 延迟 +300% 先看池是否打满 |
| 运行时 | CPU throttling、重启次数、内存 RSS 曲线 | 泄漏与限流的形式差异 |

告警配置要点：

- 告警窗口 10–15 分钟，**不能只看部署完成那一刻**；
- 以 SLO 烧速（burn rate）为主，避免固定阈值在低流量下误报；
- 每条告警带 `version`/`revision` 标签，直接回答"是不是新版本"。

## 2. 止血与回滚

顺序：**停扩散 → 回滚 → 保留现场 → 查因**。

```bash
# 1) 停住滚动，不再推进
kubectl rollout pause deploy/<svc>

# 2) 回滚到上一个 ReplicaSet（或 Argo Rollouts）
kubectl rollout undo deploy/<svc>
kubectl argo rollouts abort <rollout>     # abort 会自动回滚到 stable
kubectl argo rollouts undo <rollout>

# 3) 确认恢复
kubectl get pods -l app=<svc> -o wide
# 看错误率归零、p99 回基线、连接池水位下降
```

约束与细节：

- **按镜像 digest 回滚**，不按 tag（tag 可能已被覆盖）。发布单里记录"上一个已知良好 digest"。
- **数据库迁移 expand/contract**：新增列/表先兼容双写，破坏性变更（删列、改类型）拆到后续发布。否则回滚应用等于回滚数据。
- **feature flag 优先**：能关代码路径就不要回滚镜像，速度更快、影响面更小。
- **保留现场**：回滚前先抓 canary 指标、Pod 日志、`kubectl describe`，否则复盘没有证据。
- 回滚本身也是需要演练的路径：定期做一次生产回滚演练，确认旧镜像、迁移兼容、配置开关都还可用。

## 3. 为什么就绪探针没拦住

| 漏洞 | 表现 | 修法 |
|---|---|---|
| 探针只查 `/healthz` 返回 200 | 进程活着，依赖全挂 | readiness 检查关键依赖（DB/缓存/下游），单次 1–2s 超时 |
| 探针从集群内发起 | 绕过 Ingress、鉴权、TLS、限流 | 增加入口侧合成探测，走完整链路 |
| 启动即通过 | 预热/缓存未建立，前几分钟走冷路径 | `startupProbe` + 预热完成标志 + `minReadySeconds`（如 30s） |
| 只验证连通性 | 200 但响应内容错误/为空 | 事务级探测校验业务语义 |
| 故障只在负载下出现 | 连接池、并发、泄漏 | 探针无法覆盖，只能靠金丝雀真实流量分析 |

原则：**就绪探针负责"能不能接流量"，正确性由入口探测与金丝雀分析负责**。两者不能互相替代。

## 4. 金丝雀配置与回滚阈值

流量切分：`1% → 5% → 25% → 50% → 100%`，每步观察 **10–30 分钟**；关键服务最后一步加人工门。低流量服务要么拉长窗口凑样本量，要么用影子流量（mirror）先验证。

自动分析阈值（相对基线，避免固定值漂移）：

| 指标 | 阈值 |
|---|---|
| 5xx 率 | canary ≤ baseline + 0.5pp，且绝对值 < 1% |
| p99 延迟 | ≤ baseline × 1.2（或绝对 +200ms） |
| 合成事务成功率 | ≥ 99.5% |
| CPU throttling | < 5% |
| 连接池使用率 | < 80% |
| 样本量 | 每步 ≥ 100 请求或 ≥ 5 分钟 |
| 失败判定 | 连续 2 个测量周期失败 → 自动 abort |

Argo Rollouts 最小骨架：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 1
        - pause: { duration: 10m }
        - setWeight: 5
        - pause: { duration: 10m }
        - setWeight: 25
        - pause: { duration: 15m }
        - setWeight: 50
        - pause: { duration: 15m }
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
        args:
          - name: service-name
            value: <svc>
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: error-rate
      interval: 1m
      count: 5
      failureLimit: 2
      successCondition: result[0] < 0.01
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",version="canary",code=~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}",version="canary"}[2m]))
```

阈值必须进 Git（GitOps），与 Rollout 同仓库评审；**不允许人肉判断"看起来还行"**。

## 5. 滚动更新期间的活动连接

实际时序：

```
新 Pod 启动 → readiness 通过 → 加入 Endpoints
旧 Pod 标记 Terminating → 从 Endpoints 摘除（异步）→ SIGTERM
```

竞态与对策：

| 问题 | 原因 | 对策 |
|---|---|---|
| 摘流量前已收到 SIGTERM，仍接新连接 | Endpoints/kube-proxy/Ingress 传播延迟 | `preStop: sleep 5–15s` 覆盖传播延迟后再退出 |
| 在途请求被切断 | 进程收到 SIGTERM 立即退出 | graceful shutdown：停止 accept，处理完在途请求；`terminationGracePeriodSeconds` > 最长请求耗时（30–60s） |
| HTTP/2、gRPC 长连接 | 连接复用到旧 Pod | 发 GOAWAY，客户端重连 |
| WebSocket | 长连接不会自己断 | 服务端 close 帧（1001），客户端重连策略 |
| 两端同时不可用 | `maxUnavailable` 过大 | `maxUnavailable: 0` + `maxSurge: 25%`；有状态/长事务用 PDB + 蓝绿 |
| 预热抖动 | 刚 ready 就被打满流量 | `minReadySeconds`，必要时用 slow-start（Istio/NGINX） |

客户端侧前提：滚动期间偶发 502/连接重置属正常，**幂等重试 + 退避**必须存在；否则再平滑的服务端策略也会放大成用户可见故障。

另外两个容易漏的点：

- **PDB 保护的是驱逐，不保护滚动**：`kubectl rollout` 不受 PDB 限制，滚动参数要自己算；
- **readiness 失败会摘流量但不重启**，liveness 失败才重启。把依赖检查放进 liveness 会导致下游抖动时整批重启，是常见事故放大器。

## 6. 防复发清单

1. 把这次故障写成**可执行的发布门禁**：合成探测用例、SLO 查询、阈值配置，全部进仓库。
2. **生产变更默认走金丝雀**，禁止直接 100%；紧急通道也要记录并事后补分析。
3. **发布后观察窗口**写进流程（如 30 分钟值班盯板），窗口内不接新发布。
4. 探针分层：`startupProbe`（预热）/ `readinessProbe`（依赖 + 语义）/ `livenessProbe`（只判死锁与僵死）。
5. 错误预算策略：SLO 烧速超阈值自动冻结发布，不靠人记得。
6. 复盘落在机制上：告警加版本维度、回滚演练定期做、迁移规范（expand/contract）落到 CI 检查。

---

## 附：故障当天操作清单

- [ ] 确认时间线与版本：`kubectl rollout history`、Pod 标签、Ingress 日志
- [ ] 暂停发布：`kubectl rollout pause` / `argo rollouts abort`
- [ ] 保留证据：canary 指标快照、Pod 日志、`describe`、下游水位
- [ ] 回滚到已知良好 digest，确认 5xx 与 p99 恢复
- [ ] 若涉及 schema：确认迁移可双向兼容，不需要回滚数据库
- [ ] 关闭相关 feature flag，缩小影响面
- [ ] 通知值班与业务方，给出恢复时间点
- [ ] 24 小时内出复盘，登记门禁项与负责人
