# OpenSandbox aio 沙箱隔离现状与 OpenShell 机制移植建议（K8s 池模式）

> 文档版本: v1.1
> 日期: 2026-09-30
> 状态: Draft
> 变更: v1.1 讨论范围收敛为 **K8s 池模式（BatchSandbox / Pool CRD）**，Docker runtime 不在范围内；v1.0 见 git 历史
> 关联: `openshell-container-sandbox-isolation-hardening.md`（OpenShell 机制解析）、`dsec-erofs-image-on-demand-pull-for-aio-sandbox.md`（镜像分发优化）
> 调研对象: OpenSandbox（D:\pro\OpenSandbox，Go/Python 混合）、OpenShell（D:\pro\openshell）

---

## 1. 范围与 aio 沙箱链路（池模式）

本文只讨论 OpenSandbox 的 **K8s 池模式**：BatchSandbox 声明 `poolRef`，从预热的 Pool Pod 里认领，Docker runtime 不在讨论范围内。

aio（All-in-One）沙箱在池模式下的完整链路：

1. **Pool 模板由用户原样提交**（`Pool.spec.template` 是 `corev1.PodTemplateSpec`，`kubernetes/apis/sandbox/v1alpha1/pool_types.go:49-68`），server 不做合成——execd-installer init container、task-executor 主命令、安全上下文、imagePullSecret **全都要用户自己写进模板**（约定见 `exporter/pool-pod-template-cookbook.md`）
2. Pool 控制器按模板建预热 Pod，只加两个 label（`pool_controller.go:1343-1366`）；模板变更触发 revision 重算 + 滚动重建（`pool_update.go:29-51`，`maxUnavailable` 默认 25%）
3. 创建沙箱：BatchSandbox spec 只含 `replicas:1 + poolRef + taskTemplate`（`batchsandbox_provider.py:158-183, 378-428`）；Pool 控制器用 Allocator 做分配（`pool_controller.go:996-1060`），**claim 是注解记账式**——`alloc-status` 注解 + `pool-allocation` finalizer（`apis.go:27, 35`），**Pod 本身不改 label、不重建**
4. BatchSandbox 控制器通过 HTTP 调预热 Pod 内 5758 端口的 **task-executor** 下发 taskTemplate（`batchsandbox_controller.go:430-497`），由 task-executor 在容器内启动用户进程（`/opt/opensandbox/bootstrap.sh + entrypoint`）
5. 客户端访问容器内 execd(44772)/AIO 门户(8080)：gateway 模式走 ingress gateway → BatchSandbox 注解里的 podIP 直连（`components/ingress/pkg/sandbox/batchsandbox_provider.go:150-158`）；非 gateway 模式 server 直接返回 `podIP:port`，要求网络可达（`batchsandbox_provider.py:942-949`）

aio 镜像为外部 `ghcr.io/agent-infra/sandbox:latest`（root 运行，内含 shell/file/browser/jupyter 全套），在池模式下作为 Pool 模板的镜像字段出现，整个"容器里装了什么"完全由该镜像 + 用户模板决定。

**池模式的特殊性**：预热 Pod 是"带着 execd/task-executor 的活容器"等待认领，隔离边界 = Pod 边界 + 用户模板自带的安全配置，平台层唯一保证的硬默认是 `automountServiceAccountToken: False`（`batchsandbox_provider.py:231`）。

## 2. 现有的隔离机制（池模式视角）

| 机制 | 位置 | 说明 |
|---|---|---|
| SA token 不自动挂载 | `batchsandbox_provider.py:231` | 平台层唯一的硬默认 |
| 模板级 securityContext merge | `batchsandbox_provider.py:79-96, 430-514` | 用户可写，叶子值运行时覆盖，**但无默认值** |
| imagePullSecret 自动建 + 失败回滚 | `batchsandbox_provider.py:263-349` | image.auth → secret |
| secureAccess 签名路由 | `services/endpoint_auth.py:32-34`、`create_helpers.py:73-76` | HMAC token 写 BatchSandbox 注解，ingress 校验（fail-fast 401）+ 访问续期（OSEP-0009），**仅 gateway 模式** |
| egress sidecar / credentialProxy | `egress_helper.py:81-160`、`provider_common.py:204-205` | **池模式明确拒绝**（`kubernetes_service.py:433-462`、`batchsandbox_provider.py:169-173`） |
| bwrap isolation（opt-in） | `provider_common.py:205-221` | 主容器加 `SYS_ADMIN` + seccomp/AppArmor Unconfined |
| IPv6 disable 时 init 特权 | `egress_helper.py:42-59` | execd init 容器 `privileged: true` 写 /proc/sys |
| pause/resume | `batchsandbox_pause_resume.go:430-473` | pause 把 Pod template 固化进 BatchSandbox 并脱离池，snapshot 走 SandboxSnapshot CR |

## 3. 明确短板（按危险程度排序）

### 3.1 execd API 默认无鉴权（最严重）

- execd 访问令牌 `EXECD_ACCESS_TOKEN` 默认空（`components/execd/pkg/flag/parser.go:41, 60-62`），token 非空时才校验 `X-EXECD-ACCESS-TOKEN` 头（`pkg/web/router.go:160-163`）
- **server 的 K8s 创建路径不注入该 token**（server/ 全仓库 grep 无引用）；池模式下要启用，得用户自己在 Pool 模板 env 里加
- gateway 模式下 ingress 有 secureAccess 校验，但仅当用户启用 secureAccess；非 gateway 模式客户端直连 podIP:port，无任何认证跳板
- **预热期放大**：池里每个空闲预热 Pod 都是带 execd 的活容器，任何能到达 podIP:44772 的主体都可直接 exec/读写文件——预热池越大、暴露面越大

### 3.2 平台层零默认硬化

- securityContext 全部来自用户 Pool 模板，**无 runAsNonRoot/fsGroup/seccomp 默认值**；server 只做功能性最小覆盖（3.3）
- charts 里完全没有 PSA（Pod Security Admission）label（`kubernetes/charts/` 全目录 grep `pod-security.kubernetes.io` 无结果）——namespace 的 PSA 级别全靠用户自理，而 bwrap isolation（SYS_ADMIN + Unconfined）与 restricted PSA 直接冲突，两者叠加时用户没有中间选项
- aio 镜像本身 root 运行、内含全套 agent 服务，容器内无任何身份约束

### 3.3 池模式下出网控制缺位（比 Docker 模式更弱）

- 池模式**明确拒绝** `networkPolicy` 和 `credentialProxy`（`kubernetes_service.py:433-462`——池 Pod 已预建，无法事后注入），也拒绝 `volumes`/`egress_settings`/`platform`（`batchsandbox_provider.py:158-183`）
- 即：**池模式下沙箱出网完全不受控，也没有 per-sandbox NetworkPolicy 对象**，Pod 间网络隔离只有 CNI 默认（全通）
- egress sidecar 的 iptables 方案与池模式在架构上互斥（sidecar 要在 Pod 创建时注入），需要一个"claim 时才落地"的机制替代

### 3.4 bwrap 内层隔离以外层降壁为代价

opt-in isolation sessions 给主容器加 `SYS_ADMIN` + seccompProfile/AppArmor **Unconfined**（`provider_common.py:205-221`）。内层加固换来外层裸奔——方向反了，且使该 Pod 无法满足任何 PSA restricted 命名空间。

### 3.5 凭据隔离弱

- credentialProxy（mitmproxy）在池模式不可用（见 3.3），aio 场景下用户 LLM API key 若注入 Pool 模板 env，则**全池预热 Pod 共享同一份明文凭据**，任何一个 Pod 的 RCE = 全池凭据泄漏
- secureAccess token、ingress HMAC key 管理在 server/ingress 侧，容器内不涉及——这点是好的
- mitm CA 被装进沙箱信任库实现 TLS 拦截（`bootstrap.sh:243-287`），本身即沙箱内可被滥用的信任锚

### 3.6 其他

- Windows 平台沙箱 `privileged: True`（`services/k8s/windows_profile.py:143`）
- snapshot 恢复容器需 CAP_SYS_PTRACE（`sandboxsnapshot_lifecycle.go:351-362`）
- 无审计：eBPF 审计仅 isolation 会话 opt-in，池模式下 claim/释放/到期无安全事件留痕

## 4. OpenShell 可移植机制清单（池模式插入点）

### P0 — 改动小、封住最大的洞

| # | 机制 | OpenShell 参考 | 池模式插入点 |
|---|---|---|---|
| 1 | **execd 强制 token**：claim 时由 controller/task-executor 注入随机 token，ingress 转发时带 `X-EXECD-ACCESS-TOKEN` 头；预热期 Pod 在 claim 前对 44772 默认拒绝 | Session JWT 绑定 sandbox_id + generation + epoch（`sandbox_auth.rs:43-120`）——OpenSandbox 不需要那么重，但"默认空 token + 预热期裸奔"必须消灭 | taskTemplate env 注入已有通道（`batchsandbox_provider.py:516-551`）；ingress 校验已有 secureAccess 骨架可复用 |
| 2 | **claim 时安全基线校验/加固**：认领沙箱时校验 Pod 的 securityContext（runAsNonRoot、cap drop、seccomp）未被篡改，不符即拒绝 claim | K8s 驱动创建 Pod 后回读 securityContext 全项校验（`driver.rs:2150-2240`） | Allocator 分配决策处（`pool_controller.go:996-1060`），天然适合做"符合基线才可被认领" |
| 3 | **PSA / chart 默认加固**：charts 补 PSA label；Pool 模板文档给 restricted 兼容的基线模板（去掉 bwrap 的 SYS_ADMIN+Unconfined 才能过 restricted） | K8s 驱动的 RuntimeDefault seccomp + PSA 组合 | `kubernetes/charts/opensandbox-server/`、cookbook |

### P1 — 结构性移植，解决"AI 沙箱拿凭据"的根本问题

| # | 机制 | OpenShell 参考 | 池模式落点 |
|---|---|---|---|
| 4 | **凭据占位符 + 代理端点绑定注入**：沙箱 env 只放 `opensandbox:resolve:vN_LLM_API_KEY` 占位符，真值在代理侧，仅当流量命中绑定的 host:port 才重写 | `provider_credentials.rs:195-256`；占位符带 revision 防 cross-generation 复用 | 需要设计**池模式兼容的 egress**：现有 sidecar/credentialProxy 因"Pod 预建"被拒，可改为 claim 时由 controller 创建 per-claim NetworkPolicy + 代理（OpenShell 的 NetworkPolicy fence：workload Pod 空 egress、仅代理入站，`openshell-driver-kubernetes/src/isolation.rs:104-179`，正是"运行时下发、与 Pod 生命周期解耦"的形态） |
| 5 | **egress 决策按二进制身份**：从"纯域名白名单"升级为"哪个二进制允许访问哪个域名" | procfs 身份解析（SHA256 五元组 + TOFU 突变拒绝，`openshell-binary-identity`）；未解析身份一律 deny | egress/代理组件内实现；execd 已有 eBPF exec 审计可提供二进制信息 |

### P2 — 深水区，借鉴思路而非代码

| # | 机制 | OpenShell 参考 | 池模式落点 |
|---|---|---|---|
| 6 | **修正 bwrap 降壁悖论**：外层保持 cap_drop ALL + RuntimeDefault seccomp（过 restricted PSA），内层用 seccomp-notify broker 替代需要 SYS_ADMIN 的 bwrap mount 操作 | per-thread seccomp-notify + capability-free launcher（`workload_launcher.rs`），免特权实现逐连接决策 | execd isolation sessions 演进方向 |
| 7 | **claim 生命周期安全事件**：claim/释放/到期/基线校验失败发结构化审计，deny 聚合上报 | OCSF deny 聚合上报（`denial_aggregator.rs`）；安全发现双发（领域事件 + DetectionFinding） | controller 侧事件 + execd eBPF 审计 JSONL 已有底子 |

## 5. 结论

池模式的架构（预建 Pod + 注解记账 claim）带来一个 Docker 模式没有的结构性约束：**任何"Pod 创建时注入"型的加固（egress sidecar、credentialProxy、token env）都无法在 claim 后补做**——这正是 3.3 出网控制缺位的根因。因此移植 OpenShell 机制时，插入点优先选 claim 时刻：

1. P0 的 execd token 与基线校验都搭 claim 通道（taskTemplate 已有 env 注入、Allocator 已有决策点），不引入新机制；
2. P1 的凭据占位符注入依赖"claim 时下发 per-claim NetworkPolicy + 代理"，OpenShell 的 K8s 驱动已验证该形态可行（NetworkPolicy 与 Pod 解耦、运行时创建、SHA256 绑定 generation）；
3. 预热期裸奔（空闲 Pod 带 execd 无鉴权）是池模式特有暴露面，P0 #1 的"claim 前默认拒绝"是必须项而不是优化项。

aio 场景里用户 LLM API key 是最高价值目标，P1 #4（占位符注入）是收益最大的单项结构改进。
