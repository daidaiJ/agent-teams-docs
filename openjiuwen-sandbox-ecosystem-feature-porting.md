# openJiuwen 生态沙箱能力测绘与 OpenSandbox 特性移植设计

> 调研时间：2026-09-30。调研对象：openJiuwen-ai 组织（GitHub）下与 Agent 运行时/代码执行/沙箱隔离相关的开源仓库，重点 jiuwenswarm（develop 分支）、agent-core、agent-dx、agent-runtime。
> 目的：为 OpenSandbox 寻找可移植的沙箱特性设计，并与已有的 openshell 移植计划（`wiki/opensandbox-openshell-portable-policy-design.md`，基于 openshell commit 1358941）衔接。
> 结论均基于源码核查，证据标注为「仓库相对路径:行号」（jiuwenswarm 均指 develop 分支）。

## 0. TL;DR

1. openJiuwen 的沙箱能力**不在 jiuwenswarm 的 main 分支**（main 是旧产品 JiuwenClaw，命令直接 `subprocess` 跑宿主机，仅正则黑名单防护），而在 **develop 分支的 `jiuwenbox/` 子目录**——一个基于 bubblewrap + Landlock + seccomp + netns + cgroup 的进程级 Linux 沙箱服务。
2. 「同组织其他运行时项目提供沙箱隔离能力」的判断成立：agent-core（沙箱抽象层）、agent-dx（FaaS 沙箱容器 API）、agent-runtime（会话编排）共同构成完整体系。
3. 可移植的价值点是**策略模型、API 契约、生命周期语义**，而非隔离机制本身（OpenSandbox 的容器 + gVisor/Kata/Firecracker 路线更强）。
4. 最高性价比三件事：**egress 契约升级**（openshell audit/enforce 灰度 + jiuwenbox 域名级规则合流到一个 PR）、**jiuwenbox REST 兼容适配层**（零框架改动接入 openJiuwen 生态）、**OSEP-0021 异步池预热吸收 agent-runtime 的预热池设计**。

## 1. openJiuwen 组织全景（22 个仓库，沙箱相关度分级）

| 仓库 | 定位 | 沙箱相关度 |
|---|---|---|
| jiuwenswarm (6k★, Python) | 多 Agent 蜂群协作（develop 分支）；内含 jiuwenbox 沙箱服务子目录 | **高** |
| agent-core (438★, Python) | Agent SDK；含 `sys_operation/sandbox` 完整沙箱抽象层（gateway/launcher/provider） | **高** |
| agent-dx (3★, Python) | 分布式执行基座；`executor/` 含 YuanRong FaaS 沙箱容器 API | **高** |
| agent-runtime (15★, Python) | Agent 运行与部署管理平台；session 编排层有 K8s/Docker/进程三种 handler + `CODE_SANDBOX_URL` 外接代码沙箱 | **高** |
| agent-runtime-java (32★) | Java 版分布式 Runtime；`manager/*` 尚未交付，无沙箱代码 | 低 |
| agent-protocol (99★, C++) | MCP/A2A SDK + A4P 认证鉴权审计协议（授权边界，非执行隔离，可互补） | 低-中 |
| agent-studio / agent-tools / agent-infer / deepsearch / sciencediscovery / iCode / agent-memory / jiuwensymbiosis / skillhub / model-router / relay / docs / community 等 | 无沙箱基础设施 | 低 |

## 2. jiuwenbox：核心沙箱实现（jiuwenswarm develop 分支）

### 2.1 定位与架构

「lightweight Linux sandbox service for running agent tools and code snippets with layered isolation」。FastAPI 管理面（`jiuwenbox/src/jiuwenbox/server/app.py`）+ bubblewrap 逐沙箱拉起（每沙箱一个长驻 daemon + 按需后台命令）。隔离全部为 Linux 原生进程级机制，无 Docker/gVisor/Kata/Firecracker 依赖。

### 2.2 隔离技术栈

| 维度 | 设计 | 证据 |
|---|---|---|
| Namespace | bubblewrap：user/pid/ipc/cgroup/uts | `jiuwenbox/src/jiuwenbox/supervisor/bwrap.py` |
| Landlock | 三档：best_effort / hard_requirement / disabled，作为 fs policy 之上的二次兜底 | `supervisor/{landlock,landlock_launcher}.py`；`models/policy.py:417-443` |
| seccomp | 按架构黑名单，x86_64/arm64 各约 30 个敏感 syscall（ptrace/mount/bpf/io_uring 等） | `configs/default-policy.yaml` seccomp 全表 |
| 文件系统 | read_only（默认 `/` `/bin` `/usr`…）+ read_write（`/home` `/tmp`）+ bind_mounts（ro/rw）+ bind_root_entries include/exclude 通配 + device 白名单（默认仅 `/dev/urandom` `/dev/null`） | `models/policy.py:281-388`（BindMount/BindRootEntries/DeviceMount/FilesystemPolicy） |
| 网络 | `network.mode: isolated` 时创建独立 netns + veth + NAT uplink（自动避开冲突私网段）+ iptables/nftables 规则；规则支持 default deny/allow × {allowed/blocked_domains, allowed/blocked_ips(CIDR), allowed/blocked_ports}，**block 优先于 allow**；默认策略显式封锁 `169.254.169.254`（云元数据端点） | `policy.py:445-654`；`supervisor/network.py` |
| 资源 | cgroup v2 优先、v1 回退：`memory_max`（字节或 256M/1G 后缀）、`cpu_max`（小数核数或 quota/period）、`pids_max`；cgroup 不可写且配了限额则创建失败，全空则跳过 | `supervisor/cgroup.py`；`policy.py:53-278` |

### 2.3 生命周期与 API

- 状态机 PROVISIONING → RUNNING → STOPPED/ERROR（`models/sandbox.py:68-92`）。
- REST API：`POST /api/v1/sandboxes`（可带 policy/policy_mode/sandbox_runtime）、`GET/DELETE /sandboxes[/{id}]`、`start/stop/restart`、`exec`、`exec_background`（后台任务 list/get/kill）、`logs`、`upload/download`、`files`、`search`（`server/routes/sandbox.py:62-243`）。
- **空闲回收 reaper**：根 policy 配 `timeout.idle_timeout` + `idle_check_interval`，后台轮询淘汰；「空闲 = 自最后一次 exec/file IO 起」（查询类调用不刷新计时器）；审计日志带 `reason: idle_timeout`（`server/sandbox_manager.py:135-504`）。
- 崩溃恢复：状态 JSON 持久化到 `~/.jiuwenbox/sandboxes`，但**重启显式视为 ephemeral、注册表清空**（`sandbox_manager.py:165-174` 注释 "Sandbox state is treated as ephemeral"）。无任何 snapshot/checkpoint/热恢复实现。

### 2.4 Swarm 集成方式（嵌入式沙箱管理范本）

- `startup_mode: internal`（AgentServer spawn uvicorn 子进程，PR_SET_PDEATHSIG 防残留、随机空闲端口、随机 Bearer token 不落盘）或 `external`（只做健康检查）（`jiuwenswarm/server/sandbox/jiuwenbox_runner.py`；`server/agent_ws_server.py:921-1660`）。
- `fallback_on_failure: true` 时沙箱 exec 异常回退本地执行（`common/config.py:3207`；另有禁回退变体 `no_host_fallback_jiuwenbox.py`）。
- 非 Linux 平台直接拒绝启用（`_require_sandbox_supported`）；Windows 沙箱版在企业分支（issue #3782 自述）。
- 沙箱共享粒度由 agent-core 的 `SandboxIsolationConfig(container_scope=CUSTOM, custom_id=按 project_dir 派生)` 决定（`server/runtime/agent_adapter/sysop_builder.py:571-680`）。

## 3. 同组织运行时项目的沙箱设计

### 3.1 agent-core：沙箱抽象层（`openjiuwen/core/sys_operation/`）

- **双模式操作抽象**：`OperationMode.LOCAL/SANDBOX`，fs/shell/code 三类操作统一接口；LOCAL 模式带 shell_allowlist（默认 27 个命令前缀）+ sandbox_root 路径边界（`core/sys_operation/base.py:15-19`、`config.py:15-40`）。
- **三层结构**：`SandboxGateway`（单例）按 `{op_type, method, params, isolation_key}` 路由 → 按 isolation_key 缓存 provider → provider 调用；provider 按 `sandbox_type` 从 SandboxRegistry 创建（内置 jiuwenbox/yuanrong/aio）（`sandbox/gateway/gateway.py:32-158`）。
- **Provider 接口**（API 形态，比裸 exec 更贴 Agent 使用习惯）：`BaseFSProvider`（read/write/upload/download/stream 分块、list/search，含 head/tail/line_range）、`BaseShellProvider`（execute_cmd(+stream)、cwd/timeout=300/env）、`BaseCodeProvider`（execute_code(+stream)，python/javascript）（`providers/base_provider.py`）。
- **隔离粒度三档**：`ContainerScope.SYSTEM`（全局一沙箱）/ `SESSION`（默认，同 session 共享）/ `CUSTOM`（context.id 直作 key）+ prefix 命名空间；`isolation_key_template: "{session_id}"` 占位符运行时从 ContextVar 解析；isolation_key_template 全局唯一，冲突即拒绝注册（`config.py`；`sysop_manager.py:38-48`）。
- **生命周期语义**：`on_stop: delete/pause/keep`；`idle_ttl_seconds` 空闲即删优先于 on_stop；endpoint 恢复链 RUNNING 复用 → PAUSED 调 resume → 其他重建；`container_config_hash`（image/env/volumes/resource_limits/network/service_port 的 SHA）用于配置变更检测（`gateway.py:34-57,255-289`）。
- **jiuwenbox provider 即官方客户端契约**：`POST /api/v1/sandboxes`、`PUT /api/v1/timeout`（idle_timeout 部分更新）、exec/upload/download/files/search；沙箱丢失后 `_recreate_sandbox_after_loss` 重建（`extensions/.../providers/jiuwenbox.py:392-578`）。
- **上游未实现（宣称但不存在的部分）**：Redis 沙箱存储（`sandbox_store.py` 仅 InMemory，phase 1 only supports memory）；自启动 Launcher（仅 PreDeployLauncherConfig，"launcher simply returns the provided base_url"）。

### 3.2 agent-dx：FaaS 沙箱容器 API（`executor/`）

- Executor HTTP Server（默认 :18093）**仅接受 loopback 来源**（否则 403），同机进程经它创建/操作/销毁远端子沙箱容器。
- 创建参数：`image`、`cpu`（millicores）、`memory`（MB）、`sandbox_type: ""/"supervisor"/"docker"`、`ports`（`tcp:8080` 声明式端口转发）、`upstream`、`working_dir`、`env`、`idle_timeout`（默认 300s 必须>0）、`user`、`trace_id`；**创建同步阻塞，200 即就绪含 readiness 校验，无需轮询**；DELETE 幂等（不存在也 200）。
- 背压：全局最多 64 并发（超出 503 `executor is busy`）；请求/响应各 512 MiB 上限（413）；trace_id 全链路透传（1-128 字符白名单）。
- 核心隔离在闭源 openYuanrong 侧（`yr.agentexecutor.sandbox`）；issue #3798 用户环境显示 k3s+QEMU+libvirt 栈，但开源代码中无 microVM 调用，不可核查。

### 3.3 agent-runtime：会话编排与部署

- **Session 编排设计文档**（`management/openjiuwen_runtime/management/session/DESIGN.md`，全中文含 mermaid）：双队列（系统事件优先）调度；服务实例=部署单元+WSS 多路复用通道；**session 亲和**（session_id→service_id，TTL 滑动续期）；两级并发控制（服务级+会话级 BoundedSemaphore）；**`min_idle` 预热池 + `max_services` 上限 + `service_ttl` 空闲缩容**；资源满返回错误码 100001。
- K8s 后端真实可用：等 Pod Ready、按 label 批量清理、ContainerSpec 含 request/limit 与 securityContext（`k8s_service_handler.py:29-290`）。
- **Docker 后端是桩**：`DockerServiceHandler.deploy()` 仅生成假 `ctr-<uuid>`（"deploy(桩)"），session 编排层 Docker 隔离未真正接线（`docker_service_handler.py:16-69`）。
- 外接代码沙箱：工作流代码节点经 `CODE_SANDBOX_URL`（默认 `http://127.0.0.1:8188/run`，compose 示例 `code-sandbox:8080/run`）调用外部 HTTP 沙箱执行 Python（RemoteCodeRunner 来自闭源 openjiuwen_studio 包；`.env.example:34-36`）。

## 4. 上游反馈取证（GitHub issues，均为 bot 自 GitCode 同步，抽样未见官方回复）

| Issue | 状态 | 结论 |
|---|---|---|
| jiuwenswarm#3782 | OPEN | Windows 沙箱版 jiuwenbox 只在企业分支，开源分支仅 Linux/bwrap |
| jiuwenswarm#5113 | OPEN | k8s 环境 200+ 沙箱进程残留，"缺少空闲主动回收逻辑"——develop 已有 idle reaper 代码但 issue 仍 open，reaper 设计值得对照测试 |
| jiuwenswarm#3798 | OPEN | 销毁沙箱 A 后后台进程泄漏到新沙箱 B（孤儿进程）；用户环境 k3s+QEMU+libvirt（社区部署形态，代码不可核查） |
| jiuwenswarm#6755/#4159/#6434 | OPEN | 存在闭源 AgentOS 云版沙箱形态，管理面不在开源仓库 |
| agent-core#648/#1173/#1437 | CLOSED | yuanrong sandbox 已实现；三条观测 Span 补全；沙箱重建丢 policy 键 bug 已修（链路活跃维护） |

## 5. 文档/配置宣称但代码未实现的能力（移植前需知）

1. agent-runtime 的 Docker 部署后端是桩，宣称的三后端抽象仅 K8s 与本地进程真实可用。
2. agent-core 的 Redis 沙箱存储未实现（多网关实例共享注册表能力不存在）。
3. agent-core 的自启动 Launcher 未实现（仅 pre_deploy 外部直连）。
4. 开源分支无 Windows/macOS 沙箱。
5. **无快照/热恢复/热迁移**：jiuwenbox 与 agent-core 全部源码无 sandbox checkpoint/restore；`on_stop: pause` 只是状态语义占位。
6. RemoteCodeRunner 与 AgentOS 沙箱管理面不在开源仓库，只能看到调用契约。

## 6. 移植到 OpenSandbox 的特性清单

### 6.1 A 档：直接新增价值

| # | 特性 | 来源 | 落点 | 说明 |
|---|---|---|---|---|
| A1 | **域名级 egress 规则 + 云元数据封锁** | jiuwenbox NetworkRulePolicy | `specs/egress-api.yaml` NetworkRule（现仅 `action: allow\|deny`） | allowed/blocked_domains × allowed/blocked_ips(CIDR) × ports，block 优先；默认封 `169.254.169.254`。与 openshell 移植文档的 `enforcement: audit\|enforce` 灰度**合流为一次契约升级** |
| A2 | **声明式沙箱策略 schema** | jiuwenbox policy.yaml | sandbox spec 扩展段 | namespace/landlock/seccomp/cgroup/fs/network/idle-timeout 一体化声明；执行层映射到 K8s/Docker 原生机制（securityContext、NetworkPolicy、limits），**只取 schema 不取 bwrap 执行层** |
| A3 | **空闲回收精确语义** | jiuwenbox idle reaper | 生命周期服务 | 「空闲 = 最后一次 exec/file IO」判定 + 回收审计 `reason: idle_timeout`；对照测试 issue #5113 场景 |
| A4 | **session 隔离粒度三档模板** | agent-core ContainerScope | 多租户（OSEP-0014）之上的 session 绑定层 | SYSTEM/SESSION/CUSTOM + isolation_key_template 占位符 + key 冲突拒绝注册 |
| A5 | **config-hash 变更检测** | agent-core container_config_hash | 池化/快照路径 | image/env/volumes/limits 的 SHA；配置变更触发重建而非复用，适配池化镜像升级 |
| A6 | **同步创建即就绪 + 幂等删除 + 背压** | agent-dx executor API | lifecycle 契约打磨 | 200 即 readiness 校验通过；DELETE 幂等；并发上限 503 背压；trace_id 透传 |
| A7 | **生命周期语义细节** | agent-core | 生命周期服务 | `on_stop: delete/pause/keep` + idle_ttl 优先级 + PAUSED→resume→重建恢复链（OpenSandbox pause/resume 已有，补语义优先级） |

### 6.2 B 档：生态对接（让 OpenSandbox 被 openJiuwen 直接挂载）

| # | 对接面 | 说明 |
|---|---|---|
| B1 | **jiuwenbox REST 兼容适配层** | `/api/v1/sandboxes` CRUD + `/exec`（含 exec_background）+ `/upload`/`/download`/`/files`/`/search` + `PUT /api/v1/timeout`。openJiuwen 的 agent-core provider 按此契约调用，零框架改动接入 |
| B2 | **agent-runtime `/run` 代码执行端点** | 工作流代码节点经 `CODE_SANDBOX_URL` 调外部 HTTP 沙箱执行 Python；execd `/code` API 加 shim 即可对接 |

### 6.3 C 档：领先项（无需移植）

- 进程隔离机制（容器 + gVisor/Kata/Firecracker 路线更强）。
- pause/resume + 快照（jiuwenbox 无任何 snapshot/checkpoint，状态 ephemeral）。
- 凭证管理（Credential Vault OSEP-0012 已实现）。
- 多租户（OSEP-0014 已实现）。

### 6.4 可选参考（谨慎采纳）

- **fallback_on_failure 本地回退**：沙箱异常回退宿主机执行。实用但等于放弃隔离，与 OpenSandbox 安全底线冲突，如采纳必须默认关闭。
- **internal/external 双模式 + PDEATHSIG + 随机 token**：嵌入式沙箱管理的完整工程方案，适合 OpenSandbox SDK 内嵌场景参考。

## 7. 与 openshell 移植计划的衔接

已有 `D:\pro\OpenSandbox\wiki\opensandbox-openshell-portable-policy-design.md`（2026-09-28，方案设计未实施），结论是移植 openshell 的 "honest"（audit 灰度）而非 fail-closed 契约。三方合流共识：

1. **egress 一次升级**：openshell 贡献 L7/OPA 评估 + `enforcement: audit|enforce` 灰度；jiuwenbox 贡献域名级规则 + 元数据 IP 封锁基线；都落在 `specs/egress-api.yaml` 的 NetworkRule 扩展（当前 `egress-api.yaml:387-400`）。
2. **策略热更新与 prover 审批**（openshell 独有）维持原 A 档计划不变。
3. **agent-runtime 的 session/DESIGN.md**（亲和 + TTL 滑动续期 + min_idle 预热池 + 双队列）作为 OSEP-0021 异步池预热（现为 draft）的设计蓝本。

## 8. 建议优先级

1. **P0**：egress 契约升级（openshell audit/enforce + jiuwenbox 域名规则/元数据封锁，一个 PR 解决两份输入）。
2. **P0**：jiuwenbox REST 兼容适配层（B1）——成本最低的生态对接。
3. **P1**：声明式 policy schema 扩展（A2）+ config-hash 变更检测（A5）；session 隔离粒度三档（A4）。
4. **P2**：空闲回收语义细化（A3）；OSEP-0021 吸收 agent-runtime 预热池设计；B2 shim。

## 9. 注意事项

1. **粒度错位**：jiuwenbox 的 fs/network policy 语义建立在进程级沙箱上（如 bind_mount 直通宿主路径），移植时只取 schema 与 API 形态，执行层映射到容器边界（volume、NetworkPolicy、securityContext）。
2. **上游短板即差异化机会**：Docker 部署桩、Redis 存储缺失、无快照/热恢复、无 Windows 沙箱——这些恰是 OpenSandbox 已覆盖或可覆盖的点，可在生态对接时作为卖点。
3. **组织内未统一沙箱 API 规范**：jiuwenswarm/agent-core/agent-runtime/agent-dx 四条路径各有沙箱入口；生态对接应以 agent-core 的 `BaseFSProvider/BaseShellProvider/BaseCodeProvider` 接口为最小公约数。
