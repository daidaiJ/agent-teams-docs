# Kubernetes requests / limits 对调度、namespace 配额与集群容量的影响

> 文档版本: v1.0
> 日期: 2026-10-21
> 状态: Draft
> 关联: `opensandbox-vs-cubesandbox-selection.md`（业务约束）、OpenSandbox 仓库 `docs/guides/multi-tenancy.md`（租户配额与 LimitRange 实践）

---

## 0. 结论

- **requests 决定 Pod 能落在哪、占多少额度和多少集群承诺容量。**
- **limits 决定 Pod 运行时实际能用的上限，以及超卖后出问题的形式**（CPU 是 throttle，内存是 OOMKill）。
- **namespace 的 ResourceQuota 统计的是声明值**，准入时累加，与容器真实用量无关；配额够不等于集群放得下。

三者关系可以压缩成一条链路：

```
Pod 声明 requests/limits
   ├─ 调度器：Σrequests ≤ Node Allocatable        → 绑定节点
   ├─ kubelet：cgroup 按 limits 限流/限额          → 运行时限
   ├─ ResourceQuota：按 requests.* / limits.* 记账 → namespace 准入
   └─ Autoscaler / HPA：按 requests 计算           → 扩容决策
```

---

## 1. requests 与 limits 的本质区别

| 维度 | requests | limits |
|---|---|---|
| 作用阶段 | 调度、准入、配额 | 运行时 cgroup |
| 调度器是否使用 | 是，判断节点余量 | 否，不参与放置 |
| namespace 配额键 | `requests.cpu` / `requests.memory` | `limits.cpu` / `limits.memory` |
| CPU 超限行为 | 争抢时按 `cpu.shares` 分权重 | CFS quota 限流（throttle），不会被杀 |
| 内存超限行为 | 不强制 | 容器被 OOMKill |
| 缺省规则 | 只写 limits、不写 requests 时，request 在准入阶段被复制为 limit | 都不写则无上限，QoS 为 BestEffort |
| 能否小于实际需求 | 能，代价是超卖风险 | 不能，等于硬上限 |

补充两条容易忽略的规则：

1. **requests 也影响运行时 CPU 权重**。节点 CPU 争抢时，`cpu.shares` 由 requests 换算，所以 requests 低的 Pod 在拥塞时天然吃亏。内存 requests 只用于调度和驱逐排序，运行时不做限制。
2. **扩展资源必须 requests == limits**。GPU、`hugepages-*` 等资源不写 requests 或两者不等会被 API 拒绝。

---

## 2. 对调度的影响

### 2.1 Pod 有效 requests 的计算

调度器看的是 Pod 级聚合值，按每种资源分别计算：

```
pod_request(r) = max( Σ 普通容器与 sidecar 的 request(r),  max(init 容器 request(r) ) ) + pod overhead
```

因此：

- 多个 sidecar 的 requests 会叠加，egress sidecar 不显式声明就会继承 LimitRange 默认值。
- init 容器取最大值而非求和，但重启型 sidecar（native sidecar）算进普通容器那一侧。

### 2.2 过滤与打分

- **Filter**：`NodeResourcesFit` 检查 `Allocatable − Σ已分配 requests ≥ pod_request`。放不下就是 Pending，事件为 `Insufficient cpu/memory`。
- **Allocatable** 不是物理容量：`Allocatable = Capacity − kubeReserved − systemReserved − 驱逐阈值预留`，DaemonSet 与 kube-system 的 requests 也先扣掉。
- **Score**：默认 `LeastAllocated` + `NodeResourcesBalancedAllocation`，倾向于把 Pod 摊开并平衡 CPU/内存比例。高密度场景可切到 `MostAllocated` 做装箱，代价是碎片和故障域变大。
- **抢占**：高优先级 Pod 触发抢占时，比较的是“驱逐受害者后释放的 requests 是否够用”，与 limits 无关。
- **碎片**：requests 越大越容易产生“单项资源够、另一项不够”的不可用节点。典型现象是集群总余量看着不少，新 Pod 却长期 Pending。

### 2.3 调度只看声明，不看历史

调度器不知道容器过去用了多少。一个 requests=2C 但常年只用 100m 的 Pod，会一直占着 2C 的调度额度。这类“声明虚高”是集群密度损失的首要来源，需要用 VPA 或人工压测来收敛。

---

## 3. 对集群容量与超卖的影响

### 3.1 承诺容量与理论上限

```
承诺容量 = Σ Pod requests        ≤ Σ Node Allocatable
理论上限 = Σ Pod limits         可以远大于物理容量
超卖比   = Σ limits / Σ requests
```

- 调度器保证的是 **Σrequests 不超卖**；limits 之和可以任意超过物理资源，超额部分靠内核争抢与限流兜底。
- **CPU 超卖相对安全**：争抢表现为延迟升高、throttle 增加，不会直接杀进程。对短任务 Q&A 型负载，2 倍 CPU 超卖通常是可接受的。
- **内存超卖会把风险直接转成故障**：容器触到自己的 limit 就 OOMKill；节点整体内存吃紧则触发 kubelet 驱逐。表现为随机失败、重试和尾延迟抖动。

### 3.2 节点压力与驱逐

kubelet 的驱逐依据是**真实用量**（`memory.available`、`nodefs.available`、`imagefs.available`、`pid.available` 等阈值），与 requests 无直接关系；requests 参与的是受害者排序：

1. QoS 类别：BestEffort → Burstable → Guaranteed。
2. 同类中优先驱逐“用量已超出自身 requests”的 Pod。
3. 再看 Priority、以及用量相对 requests 的偏离程度。

QoS 判定：

| QoS | 条件 | 调度密度 | 风险 |
|---|---|---|---|
| Guaranteed | 所有容器 CPU、内存的 requests == limits | 最低 | 最稳，最后被驱逐 |
| Burstable | 设置了 requests，且至少一项 request < limit | 中高 | 内存超卖时可能 OOM |
| BestEffort | 完全未设置 | 最高 | 最先被驱逐 |

### 3.3 存储与临时盘

- `requests.ephemeral-storage` 参与调度，决定节点本地盘能否容纳。
- `limits.ephemeral-storage` 超限会触发 Pod 驱逐（而非容器 OOM），日志和临时文件多的沙箱要关注。

---

## 4. 对 namespace 配额的影响

### 4.1 记账方式

ResourceQuota 由 apiserver 的准入控制器在 **Pod 创建时**按声明值累加，Pod 进入终态并被删除后才释放：

- 配额里写了 `requests.cpu`，则每个容器必须显式声明 requests，否则拒绝；写了 `limits.cpu`，同理必须声明 limits。
- 典型拒绝信息：`exceeded quota: tenant-quota, requested: requests.cpu=1, used: requests.cpu=8, limited: requests.cpu=8`。
- 配额统计包含 init 容器的最大值口径与 sidecar，与调度口径一致。
- 终止中的 Pod 在完全删除前通常仍占额度，滚动发布时会出现瞬时配额不足。
- 可用 `scopeSelector` 按 `BestEffort` / `NotBestEffort` / `Terminating` / `NotTerminating` / PriorityClass 分别限额；`count/<resource>.<group>` 可以按 CR 数量限额。

### 4.2 配额与集群容量是两层约束

- 配额是 **namespace 级上限**，不预留集群容量。所有租户配额之和超过集群可分配量时，Pod 能通过配额检查但会一直 Pending。
- 反过来，配额打满时即使集群很空也创建不了新 Pod。
- 配额按声明值计算，所以**调大 requests 等于按比例消耗配额**；超卖比体现在 `limits.*` 配额上，两者要分别设定。

### 4.3 LimitRange 的补位作用

LimitRange 解决“配额的必填项没人填”的问题：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
  namespace: sandbox-alpha
spec:
  limits:
  - type: Container
    default:
      cpu: "1"
      memory: 2Gi
    defaultRequest:
      cpu: 250m
      memory: 512Mi
    max:
      cpu: "4"
      memory: 8Gi
    maxLimitRequestRatio:
      cpu: "4"
      memory: "2"
```

- `default` / `defaultRequest` 会作用到**所有未声明资源的容器**，包括 sidecar 和注入的辅助容器。
- `min` / `max` / `maxLimitRequestRatio` 在准入阶段直接拒绝过大或超卖比过高的 Pod，是防止单租户吃掉整台节点的有效手段。
- 副作用：默认值对主容器合适，对 sidecar 往往过大，需要在工作负载模板里显式覆盖。

---

## 5. 对弹性伸缩的影响

- **Cluster Autoscaler** 依据 Pending Pod 的 requests 判断需要几个什么样的节点；缩容时也按节点的 requests 占用判断是否可回收。limits 不参与。
- **HPA** 的 CPU/内存利用率 = 实际用量 / requests。把 requests 调低会让利用率虚高、更早扩容；调高则更晚扩容，甚至永远达不到阈值。
- **VPA** 直接改 requests（需要重建 Pod），是收敛“声明虚高”的主要手段；与 HPA 同用时有互相干扰的风险。
- **Descheduler** 依据 requests 与节点实际负载做再平衡，适合缓解长期运行后的碎片。

---

## 6. 常见组合的效果

| 配置 | QoS | 调度密度 | 主要风险 | 适用 |
|---|---|---|---|---|
| requests = limits（CPU、内存都相等） | Guaranteed | 低 | 容量浪费，CPU 无法突发 | 长会话、内存敏感服务 |
| requests < limits，仅 CPU 超卖 | Burstable | 高 | 延迟抖动、throttle | 短任务 Q&A、CPU 突发型 |
| requests < limits，内存也超卖 | Burstable | 最高 | OOMKill、节点驱逐 | 可重试、幂等的批处理 |
| 只写 limits | 等价 Guaranteed | 低 | request 被抬到 limit，容易 Pending | 不推荐 |
| 都不写 | BestEffort | 最高 | 最先被驱逐；有配额时直接拒绝 | 仅限临时调试 |

---

## 7. 落到 OpenSandbox 的配置建议

面向“节点资源有限、服务人数优先、短任务为主”的内网集群：

1. **内存 requests ≈ limits**，避免随机 OOM 导致的失败重试；这是短任务场景最划算的一条。
2. **CPU 用低 requests + 较高 limits**（如 request 250m / limit 1C），换取突发能力和调度密度；监控 throttle 指标后再收敛。
3. **sidecar 单独定量**。egress sidecar 在 server 的 `[egress]` 里配置 `requests = 25m/64Mi`、`limits = 250m/256Mi`；否则会继承 LimitRange 面向主容器的默认值，白白消耗配额。
4. **配额按“并发用户数 × 单沙箱 requests + 余量”估算**，`limits.*` 配额按可接受的超卖比（如 2 倍）设置，两层都要留出滚动发布余量。
5. **Pool 模板里固化资源值**，让池内 Pod 的 requests 稳定可预测；`taskTemplate` 只注入环境变量等轻量差异，避免每个租户各自声明导致配额口径漂移。
6. **配额打满时的行为**：`agent-sandbox` provider 会快速返回 `403 KUBERNETES::QUOTA_EXCEEDED`；默认 `batchsandbox` provider 目前拿不到准入拒绝，会等到创建超时返回 `POD_READY_TIMEOUT`。需要快速失败语义时优先选前者。

---

## 8. 排查命令

```bash
# 节点可分配量与已分配 requests
kubectl describe node <node> | sed -n '/Allocatable/,/Events/p'

# namespace 配额使用情况（used 为声明值之和）
kubectl get resourcequota -n sandbox-alpha -o yaml

# Pod 的 QoS 与有效 requests/limits
kubectl get pod <pod> -n sandbox-alpha -o jsonpath='{.status.qosClass}{"\n"}'
kubectl describe pod <pod> -n sandbox-alpha | sed -n '/Requests/,/QoS/p'

# Pending 原因
kubectl get events -n sandbox-alpha --field-selector reason=FailedScheduling

# 真实用量与限流（需 metrics-server）
kubectl top pod -n sandbox-alpha
```

判断顺序：先看 Pending 是配额拒绝还是调度失败；再看节点 `Allocatable` 与已分配 requests 的差额是否被碎片吃掉；最后用 `kubectl top` 对比真实用量，确认是否 requests 虚高或 limits 过紧。
