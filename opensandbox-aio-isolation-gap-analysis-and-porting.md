# OpenSandbox aio 沙箱隔离现状与 OpenShell 机制移植建议

> 文档版本: v1.0
> 日期: 2026-09-30
> 状态: Draft
> 关联: `openshell-container-sandbox-isolation-hardening.md`（OpenShell 机制解析）、`dsec-erofs-image-on-demand-pull-for-aio-sandbox.md`（镜像分发优化）
> 调研对象: OpenSandbox（D:\pro\OpenSandbox，Go/Python 混合）、OpenShell（D:\pro\openshell）

---

## 1. aio 沙箱是什么

aio（All-in-One）是 OpenSandbox 的完整 AI agent 工作负载沙箱示例：

- 镜像为外部镜像 `ghcr.io/agent-infra/sandbox:latest`（Dockerfile 不在 OpenSandbox 仓库），单容器内含 shell/file/browser/jupyter 全套服务，root 运行（`examples/aio-sandbox/main.py:65`）
- 入口被强制改为 `bootstrap.sh`：先启动 execd（控制 API，容器内 44772 端口），再后台启动用户命令；`EXECD_INIT=1` 时 execd exec 成 PID 1（`components/execd/bootstrap.sh:455, 525-527`）
- AIO 门户暴露 8080 端口；容器固定暴露 `["44772", "8080"]`（`server/opensandbox_server/services/docker/docker_service.py:800`）
- 健康检查轮询容器内 8080 的 `/v1/shell/sessions`（`main.py:43-56`）
- 客户端经 `agent-sandbox` SDK 直连 AIO HTTP 门户做 shell.exec_command / file.read_file / browser.screenshot（`main.py:82-94`）

本质上 aio 是「外部重型镜像 + OpenSandbox Docker runtime 默认参数」的组合，隔离形态是"外层宽松容器 + opt-in 内层 bwrap"。

## 2. 现有的隔离机制

| 机制 | 位置 | 说明 |
|---|---|---|
| no-new-privileges 默认开 | `server/.../docker/config.py:1068-1071` | HostConfig 层面 |
| 9 项 capability 默认 drop | `config.py:1046-1061` | AUDIT_WRITE/MKNOD/NET_ADMIN/NET_RAW/SYS_ADMIN/SYS_MODULE/SYS_PTRACE/SYS_TIME/SYS_TTY_CONFIG |
| pids_limit 4096 | `config.py:1097-1101` | 防 fork 炸弹 |
| cgroup 资源限制 | `container_ops.py:335-366` | 仅当请求带 resourceLimits 时生效 |
| 到期自动回收 | `docker_service.py:297-323` | 按 expires_at 起 daemon Timer；无 timeout 则打 `manual-cleanup` 标签**永不回收** |
| 可选 gVisor/Kata runtimeClass | `config.py:990-997` | docker_runtime / k8s_runtime_class 配置项 |
| 可选 egress sidecar | `docker_service.py:807-869`、`components/egress/nft.go:42-66` | nftables + DNS 解析动态白名单 |
| 可选 bwrap 内层隔离 | `components/execd/pkg/isolation/config.go:23-120` | 每 session 独立 namespace + seccomp denylist + Landlock 白名单 + eBPF 审计 + env 黑名单 |
| K8s 签名路由 secureAccess | `services/endpoint_auth.py:27-34`、`components/ingress/main.go:63-77` | HMAC token，**仅 K8s runtime 支持** |
| server 侧可选 API key | `middleware/auth.py:49, 82-87` | 未配 api_key 且无 tenant provider 时跳过全部鉴权 |

## 3. 明确短板（按危险程度排序）

### 3.1 默认 host 网络

`config.py:1031-1034` 默认 `network_mode = "host"`。aio 默认部署下容器与宿主共享 netns，44772/8080 直接绑在宿主上，无 netns 隔离。

### 3.2 execd API 默认无鉴权（最严重）

- execd 访问令牌 `EXECD_ACCESS_TOKEN` **默认空**（`components/execd/pkg/flag/parser.go:41, 60-62`），token 非空时才校验 `X-EXECD-ACCESS-TOKEN` 头（`pkg/web/router.go:160-163`）
- **server 端没有任何地方注入 EXECD_ACCESS_TOKEN**（server/ 全仓库 grep 为空）
- 叠加单租户模式下 `/sandboxes/{id}/proxy/{port}` 代理路径**显式豁免鉴权**（`middleware/auth.py:82-83`）
- 结果：拿到宿主端口即可在沙箱内任意 exec / 读写文件

### 3.3 root 运行 + 可写 rootfs + 无 seccomp profile

- create_container 无 `user` / `read_only` 参数（`container_ops.py:433-451`），AIO 镜像内 agent 服务全部 root 身份
- 无自定义 seccomp/AppArmor profile（`config.py:1072-1077` 默认 None），只落到 Docker 默认；syscall 过滤仅 opt-in bwrap denylist 有

### 3.4 无出网默认控制

networkPolicy 是 opt-in 且与 host 网络互斥（`networking.py:152-163`）；无 networkPolicy 时沙箱出网完全不受控，`OPENSANDBOX_EGRESS_*` env 被直接丢弃（`docker_service.py:759-767`）。

### 3.5 bwrap 内层隔离以外层降壁为代价

开启 isolation sessions 时，server 反而给容器 **CAP_SYS_ADMIN + apparmor=unconfined + seccomp=unconfined** 加 tmpfs（`docker_service.py:914-932`，K8s 同款 `provider_common.py:205-215`）。内层加固换来外层裸奔——方向反了。

### 3.6 K8s 部署缺默认加固

- 沙箱 Pod securityContext 完全取决于用户 CRD 模板（`batchsandbox_types.go:101`），server 只做 merge，**无 runAsNonRoot/只读 rootfs 默认值**
- Windows 平台沙箱 `privileged: True`（`services/k8s/windows_profile.py:143`）
- server chart `values.yaml:61` podSecurityContext 为空；无 PSA 标签；network-policy 默认未启用（`config/default/kustomization.yaml` 中被注释）
- snapshot 恢复容器需 CAP_SYS_PTRACE（`sandboxsnapshot_lifecycle.go:351-362`）

### 3.7 凭据隔离弱

- image auth 明文经请求传递（`container_ops.py:322-333`）
- bwrap 开启时外层 apparmor/seccomp unconfined + SYS_ADMIN（见 3.5）
- mitm CA 被装进沙箱信任库实现 TLS 拦截（`bootstrap.sh:243-287`），本身即沙箱内可被滥用的信任锚

### 3.8 无审计

Docker runtime 无 secureAccess/签名路由；eBPF 审计仅 isolation 会话 opt-in。

## 4. OpenShell 可移植机制清单（按投入产出比）

### P0 — 改动小、封住最大的洞

| # | 机制 | OpenShell 参考 | OpenSandbox 插入点 |
|---|---|---|---|
| 1 | **容器安全默认值收紧**：host 网络→bridge；create_container 加 `user`；cap_add 收敛为显式白名单；`read_only` rootfs + tmpfs 写位；默认 seccomp profile | 外层围栏（cap_drop ALL、non-root 数字身份、network=none） | 全部在 `container_ops.py:368-451`，半天工作量 |
| 2 | **execd 强制 token**：创建容器时生成随机 `EXECD_ACCESS_TOKEN` 注入 env，代理转发带 `X-EXECD-ACCESS-TOKEN` 头 | Session JWT（绑定 sandbox_id + generation + epoch，`sandbox_auth.rs:43-120`）——OpenSandbox 不需要那么重，但"默认空 token"必须消灭 | execd flag parser + server 容器创建 + proxy 转发链路 |
| 3 | **bootstrap/attach 双侧 resource-claim 绑定**：一次性 bootstrap 材料 + 生成代绑定，防重建同名对象错挂 | K8s bootstrap Secret immutable 按 generation 命名（`sandbox_runtime.rs:517`）；resource_claims（container_id/pod UID/resourceVersion）双侧校验 | execd token 可升格为"一次性 bootstrap 材料 + 生成代绑定" |

### P1 — 结构性移植，解决"AI 沙箱拿凭据"的根本问题

| # | 机制 | OpenShell 参考 | OpenSandbox 落点 |
|---|---|---|---|
| 4 | **凭据占位符 + 代理端点绑定注入**：沙箱 env 只放 `opensandbox:resolve:vN_LLM_API_KEY` 占位符，真值保存在 server/egress sidecar，仅当流量命中绑定的 host:port 才在代理侧重写 | `provider_credentials.rs:195-256`；占位符带 revision 防 cross-generation 复用 | 天然契合 aio 场景（用户 LLM API key 是最高价值目标），egress sidecar 已有，可承载重写逻辑 |
| 5 | **egress sidecar 默认化 + 二进制身份授权** | procfs 身份解析（SHA256 五元组 + TOFU 突变拒绝，`openshell-binary-identity`）；未解析身份一律 deny | networkPolicy 从 opt-in 变默认；egress 决策从"纯域名白名单"升级为"哪个二进制允许访问哪个域名" |

### P2 — 深水区，借鉴思路而非代码

| # | 机制 | OpenShell 参考 | OpenSandbox 落点 |
|---|---|---|---|
| 6 | **修正 bwrap 降壁悖论**：外层保持 cap_drop ALL + 默认 seccomp，内层用 seccomp-notify broker 替代需要 SYS_ADMIN 的 bwrap mount 操作 | per-thread seccomp-notify + capability-free launcher（`workload_launcher.rs`），免特权实现逐连接决策 | execd isolation sessions 演进方向 |
| 7 | **OCSF 化审计**：默认关闭的审计变成随 deny 事件触发、聚合上报 | deny 聚合上报（`denial_aggregator.rs`）；安全发现双发（领域事件 + DetectionFinding） | execd eBPF 审计 JSONL 已有底子 |

## 5. 结论

aio 沙箱的三个洞（host 网络、execd 无鉴权、root + 无 seccomp）都可以在 P0 层面用容器参数 + token 修复，不依赖架构改造；凭据占位符注入（P1 #4）是对 AI agent 场景收益最大的一项结构改进，因为 aio 场景里用户 LLM API key 直接躺在容器 env 中，任何 RCE 都等于凭据泄漏。P2 项建议作为 execd isolation sessions 的长期演进方向跟踪。
