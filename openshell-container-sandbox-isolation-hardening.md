# OpenShell 容器型沙箱隔离加固机制解析

> 文档版本: v1.0
> 日期: 2026-09-30
> 状态: Draft
> 关联: `opensandbox-aio-isolation-gap-analysis-and-porting.md`（移植建议）、`opensandbox-vs-cubesandbox-selection.md`（选型背景）
> 调研对象: NVIDIA OpenShell（D:\pro\openshell，Rust 工作区），commit 1358941

---

## 1. 总体模型：三明治隔离

OpenShell 不是 All-in-One 沙箱，它的隔离建立在**三个独立信任域**之上：

| 信任域 | 角色 | 信任级别 |
|---|---|---|
| workload 容器 | 跑受 seccomp/Landlock 加固的 `openshell-sandbox` 运行时（作为容器 init），用户代码在其内 fork/exec | 完全不可信 |
| supervisor 容器/Pod | 跑 egress 代理、DNS relay、策略引擎（OPA）、凭据 resolver、进程监管 | 可信基础设施 |
| gateway | 控制面：API、沙箱生命周期、认证边界 | 集群/主机外 |

协议契约统一在 `openshell-sandbox-backend/src/boundary_protocol.rs:1-20`（"Drivers choose and provision the transport, but they do not redefine the process lifecycle, streaming, identity, or authentication messages"），隔离后端契约由 RFC 0012（`rfc/0012-isolation-backend`）定义。

核心原则：**workload 容器没有直接出口**，所有出网流量必须经过独立信任域的 supervisor 代理；容器内的运行时通过内核机制（seccomp + Landlock）自加固；每一层都有可验证的 fail-closed 语义。

---

## 2. 外层围栏：计算驱动强制的容器配置

三个容器驱动（Docker / Podman / Kubernetes）在创建 workload 容器时强制以下配置：

### 2.1 网络零化

- Docker 强制 `network_mode: "none"`（`crates/openshell-driver-docker/src/lib.rs:5797`）
- Podman 强制 `netns: "none"`，并清空 networks/portmappings/hostadd（`crates/openshell-driver-podman/src/container.rs:1549-1553`）
- K8s NetworkPolicy 对 workload Pod **Ingress+Egress 双隔离，egress 为空列表——连 DNS/K8s API 都拒**，仅允许 supervisor Pod 经 boundary port 入站（`crates/openshell-driver-kubernetes/src/isolation.rs:104-179`）

### 2.2 身份强制

- non-root 数字 uid/gid，从 pinned 镜像的 `/etc/passwd` 解析；**USER-less 镜像直接拒绝创建**（`crates/openshell-driver-docker/src/lib.rs:5748-5751, 641-655`）
- K8s 侧：`runAsNonRoot` + 固定 uid/gid + `fsGroup+OnRootMismatch` + `supplementalGroups: []` + `supplementalGroupsPolicy: "Strict"`（`crates/openshell-driver-kubernetes/src/driver.rs:5706-5718`）

### 2.3 权限与资源收敛

- `cap_drop: ["ALL"]` 且无任何 cap_add；`no_new_privileges: true`（`docker lib.rs:5784-5796`）
- 可选 AppArmor profile，创建前校验 daemon 是否支持，不支持则 fail（`docker lib.rs:5788-5794, 5915-5937`）
- 资源限制：nano_cpus、memory、pids_limit（默认 2048，`crates/openshell-core/src/config.rs:35`）
- GPU 仅经 CDI `device_requests` 白名单注入，无 `/dev` 通配（`docker lib.rs:5698-5704`）
- bind mount 默认禁用，需显式 `enable_bind_mounts=true` 且拒绝 bind-backed volume；SELinux `z` 标签支持（`docker lib.rs:212-215, 1131-1160`）
- tmpfs 以 `rw,noexec,nosuid,nodev` + 精确 uid/gid/mode 挂载（`docker lib.rs:5803-5809`）
- sysctl 仅放行 `net.ipv4.ip_unprivileged_port_start=0`（配合 cap-drop 免 root 绑低端口，`docker lib.rs:5810-5813`）

### 2.4 ENTRYPOINT 强制覆盖

三个驱动都把容器入口替换为受信的 `openshell-sandbox` 运行时（Docker `lib.rs:5756-5761`、Podman `container.rs:1196`、K8s `driver.rs:5775-5781`）。**镜像自带的 ENTRYPOINT/CMD 无法绕过**——这是对"镜像里什么都跑"类 All-in-One 场景最关键的一刀。

### 2.5 防篡改回读校验

K8s 驱动创建 Pod 后**重新读回 securityContext 校验未被篡改**：uid/gid/runAsNonRoot/seccomp/supplementalGroups/Strict/sysctl/各容器 cap drop，任一变更即失败（`crates/openshell-driver-kubernetes/src/driver.rs:2150-2240`）。

---

## 3. 中间层：supervisor 代理（关键差异点）

### 3.1 L4/L7 代理与策略决策

- workload 无网络，唯一出口是 supervisor 的 egress proxy；每个连接先做 procfs 二进制身份解析，再由内嵌 OPA/regorus 引擎 + baked Rego 规则求值决策（`crates/openshell-supervisor-network/src/opa.rs:1-14`）
- L7 深度检查覆盖 HTTP/GraphQL/JSONRPC/MCP/REST/WebSocket/TLS（`crates/openshell-supervisor-network/src/l7/`），含路径规范化防 `../` 绕过（`proxy.rs:10087-10093`）
- 策略代际原子快照，失效代际进入 quarantine，fail-closed（`opa.rs:86-101`）

### 3.2 凭据占位符注入（最可移植的机制）

- workload 环境变量里放的是**占位符** `openshell:resolve:env:v{revision}_API_KEY`，不是真凭据
- 真值只在 supervisor 的 SecretResolver；仅当流量**命中该凭据绑定的 host:port:path** 时才在代理侧重写（`crates/openshell-core/src/provider_credentials.rs:195-256`，原文："Only the prepared environment crosses an isolation boundary; resolver material remains in the supervisor"）
- 占位符含 revision，防 cross-generation 复用；端点错配 → 拒绝 + OCSF 审计事件（`proxy.rs:168-201`）
- 动态 token grant：SPIFFE JWT-SVID 作为 OAuth2 client assertion 按需换 access token 并注入 Authorization 头（`crates/openshell-supervisor-network/src/token_grant.rs:1-40`）

### 3.3 procfs 二进制身份防伪造

- 从 procfs 解析进程可执行身份 + 完整祖先链 + cmdline 引用路径，绝不报告 namespace 外宿主祖先（`crates/openshell-binary-identity/src/lib.rs:99-135`）
- 防伪造：打开 `/proc/<pid>/exe` 的 fd 后对**活对象**做 SHA256，快照含 device/inode/size/mtime/ctime 五元组；`validate_process_snapshot` 校验快照期间进程未换身（`lib.rs:170-216`）
- supervisor 侧 TOFU 缓存：同一路径二进制 hash 突变（被中途替换）→ 拒绝请求（`crates/openshell-supervisor-network/src/identity.rs:1-9`）
- **未解析身份的连接一律拒绝**（`crates/openshell-supervisor-network/src/identity_source.rs:6-9`）

### 3.4 bypass 检测与 Policy DNS

- bypass 检测：扫描 `/proc/<pid>/fd` 找 socket owner，只有入口进程树后代（Descendant）才算正当；只能从全 /proc 兜底扫到（ProcFallback）的连接被视为非受管来源（`crates/openshell-supervisor-network/src/procfs.rs:440-490`）
- Policy DNS：supervisor 持有 DNS eligibility 与合成映射（198.18.0.0/15），DNS 答案只对策略声明过的 endpoint 发布，TTL 1-30s，映射与策略 generation 绑定（`policy_dns/mod.rs:9-45`）；容器内 resolv.conf 指向 127.0.0.53 relay，`dns_search="."` 防短名外泄

---

## 4. 内层：容器内沙箱运行时自加固

### 4.1 Landlock 双层

- **强制 capability-free 基线**：逐个顶层目录项授权，刻意排除 driver 私有 `/.openshell` 层级；用户策略作为第二条 ruleset **相交**应用——用户策略只能收窄，永远打不开私有层级（`crates/openshell-sandbox/src/sandbox/linux/mod.rs:45-63`）
- prepare 在 root 阶段开 PathFd，enforce 在降权后 `restrict_self`（`mod.rs:26-42`）
- 启动时探测 Landlock ABI（要求 ≥v3）与 allow/deny 能力，不支持即 fail-closed（`src/main.rs:215-217, 307-330`）

### 4.2 seccomp 过滤

无条件封锁（`crates/openshell-sandbox/src/sandbox/linux/seccomp.rs:98-255`）：

- socket 域：AF_PACKET、AF_BLUETOOTH、AF_VSOCK；代理模式下连 AF_INET/AF_INET6 也封（全部流量走 seccomp-notify broker）
- AF_NETLINK 仅放行 NETLINK_ROUTE 协议 0
- memfd_create（fileless exec）、ptrace、bpf、process_vm_readv/writev、pidfd_getfd/send_signal、io_uring_setup、mount 全家族（fsopen/fsconfig/fsmount/fspick/move_mount/open_tree）、setns、umount2、pivot_root、userfaultfd、module 加载、kexec、clone3、perf_event_open
- 条件封锁：`execveat+AT_EMPTY_PATH`、`unshare+CLONE_NEWUSER`、`seccomp(SET_MODE_FILTER)`
- NO_NEW_PRIVS 由沙箱自己 set；supervisor prelude filter 经 TSYNC 对全线程加装（`seccomp.rs:56-64`）

### 4.3 seccomp-notify 网络 broker（免特权的逐连接决策）

- workload 子进程的 TCP connect/UDP/DNS 被截获，broker 基于进程二进制身份请求 supervisor 决策（TcpOpenDecision），决策超时 30s fail-closed（`crates/openshell-sandbox/src/network_broker.rs:1-60`）
- **per-thread capability-free launcher**：seccomp filter 是 per-thread 的，launcher 线程持有带 notify listener 的过滤器并串行化所有 fork/exec，子进程继承过滤器，broker/生命周期线程保持无过滤——实现逐连接决策**不需要 CAP_SYS_ADMIN**（`crates/openshell-isolation-interface/src/linux/workload_launcher.rs:1-25`）

### 4.4 降权与自检

- root 且无配置时强制 fallback 到 `sandbox:sandbox` 而非留 root；降权后**验证 effective uid/gid 确已变更**（CWE-250/POS37-C）、清理 bounding set、进程设为 non-dumpable（`crates/openshell-sandbox/src/process.rs:1938-2080, 2696`）
- 运行时自检：探测 Landlock ABI、seccomp notification round-trip、addfd、listener killable 模式，结果作为 `SeccompEvidence`/`CapabilityEvidence` 上报；capability-free 从 `/proc/<pid>/status` 量测 5 类 capability，**全零才算数**（`boundary_protocol.rs:74-95`）

---

## 5. 认证边界：Sandbox Protocol

- 传输：Docker/Podman 用受保护 Unix socket，K8s 用 mTLS TCP；每代生成专用 TLS 材料（自管 CA，`generate_sandbox_tls_material`）
- 认证：Bearer JWT 绑定 sandbox_id + runtime generation + credential epoch（`crates/openshell-sandbox-backend/src/sandbox_auth.rs:43-120`）；连接后分配 server-local `SandboxConnectionId`——"cannot be supplied by a compute driver or workload"，请求必须绑到执行过 attach 的连接
- 双向绑定：`BoundaryConfig` 与 `SandboxRuntimeDescriptor` 从同一不可变坐标生成；K8s bootstrap Secret **immutable 且按 generation 命名**，重建对象无法复用（`sandbox_runtime.rs:517`）；resource_claims（container_id/pod UID/policy UID/resourceVersion）双侧一致校验
- K8s 侧用 PodSchedulingGate 把 Pod 扣住直到 bootstrap Secret 安装完毕；supervisor Pod `automountServiceAccountToken: false`，改用 audience 限定的短期 projected SA token（`sandbox_runtime.rs:283-296, 465-468`）

---

## 6. 可验证的失败闭合（OuterFenceGuarantees）

四条保证（`crates/openshell-isolation-interface/src/contract.rs:400-460`）：

1. **DefaultDenyEgress** — network=none / 空 egress NetworkPolicy
2. **NoUnmanagedEgressPath** — 无绕过代理的出口
3. **RevocationVerified** — 策略撤销已被验证
4. **ControllerLossFailsClosed** — 控制器失联即冻结

每条必须由驱动从原生事实投影并 **SHA256 绑定策略 generation**，缺一条边界就不许建立。容器内的边界状态机对应 ACTIVE/FROZEN/TERMINATED/ENFORCEMENT_LOST 四态，supervisor 控制连接丢失时冻结（`crates/openshell-sandbox/src/boundary_io.rs:14-60`）。

---

## 7. 审计与可观测

- OCSF v1.8.0 结构化审计，8 个事件类（Network/HTTP/SSH/Process/Detection Finding/App Lifecycle/Device Config 等），双格式（人类 shorthand + JSONL）（`crates/openshell-ocsf/src/lib.rs:4-19`）
- deny 事件去重聚合并上报 gateway（`crates/openshell-sandbox/src/denial_aggregator.rs:4-13`）
- 安全发现**双发**：一个领域事件 + 一个 DetectionFinding（如 nonce replay 同时走 SshActivityBuilder 和 DetectionFindingBuilder）
- 红线：OCSF message 不落 secrets/凭据/查询参数（JSONL 可能外发）

---

## 8. 对 All-in-One 类沙箱的启示

OpenShell 模型的可移植内核，按优先级：

1. **ENTRYPOINT 强制覆盖**为受信运行时——镜像自带入口不可信
2. **不可变 bootstrap + 双侧 resource-claim 绑定**——防 attach 错容器/重建对象复用
3. **凭据占位符 + 代理端点绑定注入**——workload 永远拿不到真凭据
4. **Landlock 强制基线 + 用户策略相交**——策略只能收窄不能放开
5. **seccomp-notify broker + per-thread launcher**——免 CAP_SYS_ADMIN 的逐连接网络决策
6. **procfs 二进制身份（SHA256 五元组 + TOFU 突变拒绝）**——按"哪个二进制"授权
7. **OuterFenceGuarantees 证据化**——把 network=none 等原生事实投影为可验证保证，缺一即拒绝建边界

具体到 OpenSandbox aio 沙箱的移植建议，见 `opensandbox-aio-isolation-gap-analysis-and-porting.md`。
