# 沙箱平台选型：OpenSandbox vs CubeSandbox（企业内部智能体服务场景）

> 文档版本: v1.0
> 日期: 2026-08-22
> 状态: Draft
> 关联: `AGENTS.md`（业务背景）、OpenSandbox 仓库 `wiki/`（机制调研）、CubeSandbox 官方 `docs/`

---

## 1. 背景与约束

### 1.1 场景

大型企业内部智能体服务：多部门/多团队 agent 协作，运行在**企业内网 K8s 集群**上。

### 1.2 硬约束（按优先级）

| 约束 | 含义 |
|------|------|
| **资源有限** | 节点 CPU/内存不充裕，无无限云端资源 |
| **更多用户 > 售出时长** | 单位资源服务更多用户比单用户会话时长更重要 |
| **服务质量与稳定性** | 回收不能导致体验崩溃；故障域要小 |
| **业务层租户控制** | 租户/配额/审计由上层业务系统管理，不依赖沙箱平台的多租户功能 |
| **namespace 审批制** | 部门 namespace 是审批制资源，静态、受控、数量有限 |
| **内网/离线** | 私有镜像仓库、无公网依赖 |
| **数据安全** | 企业敏感数据需强隔离 + 审计 |

### 1.3 候选平台

| | OpenSandbox（阿里） | CubeSandbox（腾讯云） |
|---|---|---|
| 本质 | 通用沙箱平台：统一 API + Docker/K8s 双运行时 | Agent 专用微VM 底座：RustVMM/KVM 硬件隔离 |
| 隔离级别 | 容器级（可选 gVisor/Kata/Firecracker 加固） | 每沙箱独立内核（KVM MicroVM） |
| 冷启动 | 秒级（Pool 预热后毫秒级分配） | <60ms（模板内存快照 + FICLONE） |
| 单沙箱开销 | ~100~300MB 内存 + 0.15~0.4C | <5MB VMM 开销 + 共享模板页 |
| K8s 形态 | 原生 Operator（BatchSandbox/Pool/SandboxSnapshot CRD） | v0.6 起 Helm 部署（规划 CRD/Operator 化） |
| 部署前提 | 无特殊要求 | **裸金属 KVM（不支持嵌套虚拟化）** |

---

## 2. 逐维度对比

### 2.1 K8s 部署与多租户

| 维度 | OpenSandbox | CubeSandbox |
|---|---|---|
| K8s 形态 | 原生 Operator：BatchSandbox（批量 O(1) 交付 100 沙箱 0.92s）、Pool（预热池）、SandboxSnapshot | Helm 部署：控制面 Deployment/StatefulSet + 计算面 4 个 DaemonSet（cube-node/installer/bootstrap/PVM） |
| 多租户 | 原生 `[tenants]`（API Key → namespace 路由 + ResourceQuota）——**但本项目不用**，租户控制上移业务层 | 无租户模型，认证走外部回调（`--auth-callback-url`） |
| 租户配额 | ResourceQuota/LimitRange 原生 | 无，需自建准入层 |
| 与业务层租户控制的配合 | server 退化为执行引擎：每部门一个 server 实例（或 tenants 映射），业务层管用户→部门→namespace | 认证回调可对接 IAM，但配额/隔离需完全自建 |

### 2.2 密度与资源回收（资源有限场景核心）

| 维度 | OpenSandbox | CubeSandbox |
|---|---|---|
| 单沙箱开销 | ~100~300MB + 0.15~0.4C（最小配置：无 sidecar、无 Jupyter） | <5MB VMM + guest 私有脏页（模板页 FICLONE 共享） |
| 32C/128GB 节点容量 | ~100 并发（CPU 瓶颈），超卖 + 短 TTL 可到 ~150~200 | 内存共享 + AutoPause 后 CPU/内存归零，密度数量级更高 |
| 空闲回收 | TTL 到期销毁（需业务层主动调 pause 或靠 TTL） | **AutoPause 自动**（`on_timeout="pause"` + `auto_resume=True`），空闲即冻结，请求到达透明唤醒 |
| 配额释放 | pause 后 Pod 完全释放（rootfs 快照在 registry，节点零占用） | `paused_resource_release_ratio`（0~1）可调；**暂停快照吃磁盘 2~3GB/个**（内存换磁盘） |
| 密度优化手段 | 用完即焚 + Pool bufferMin=0 + 去 egress sidecar（NetworkPolicy 替代）+ 镜像瘦身 + 超卖 | 模板克隆 + AutoPause + ratio 调节 |

**关键 trade-off**：CubeSandbox 的 `paused_resource_release_ratio` 是"可用性 vs 利用率"权衡——ratio=1 密度最大但 resume 可能 409 失败；ratio=0 保可用性但暂停不释放配额。服务质量优先 → 低 ratio，密度优势打折。

### 2.3 持久化与产物

| 维度 | OpenSandbox | CubeSandbox |
|---|---|---|
| 卷模型 | `host` / `pvc`（→K8s PVC）/ `ossfs`；**支持 subPath + readOnly**（非池化） | host mount（节点本地，需共享文件系统跨节点）+ Volume 框架（v0.6，CSI 风格插件，可对接 COS/NFS） |
| 池化模式卷 | **API 拒绝 volumes**（预热 Pod 无法动态加卷）→ 产物走 S3 中间层（见 §3.3） | 无池化概念（模板克隆即预热），nested mount 原生支持"共享只读 + 每沙箱可写子目录" |
| 细粒度权限 | 非池化：subPath + readOnly 可做到"共享 PVC + 每沙箱写自己子目录"；池化：做不到 | nested mount 原生（"shared-readable, writer-scoped"），但 host mount 节点本地 |
| 快照 | rootfs 快照（OCI 镜像，推集中 registry） | 全状态快照（内存+文件系统，CubeCoW 增量脏页） |

### 2.4 暂停/恢复与节点故障（修正版对比）

| 维度 | OpenSandbox | CubeSandbox |
|---|---|---|
| 暂停语义 | rootfs 提交 OCI 镜像，**进程/内存不保存**；释放 Pod | **全状态冻结**（CPU 寄存器、进程内存、TCP 状态、文件系统） |
| 恢复语义 | 重写模板镜像重建 runtime，sandboxId 稳定；进程需重启 | 从快照 restore，进程原样继续，sub-second |
| 自动暂停 | 无（需 API 调用） | AutoPause/AutoResume 原生 |
| **节点故障：运行中沙箱** | **丢**（进程/内存/可写层） | **丢**（进程/内存） |
| **节点故障：已暂停沙箱** | ✅ **不丢**：快照在集中 registry，resume 可在任意节点重建 | ❌ **丢**：快照在节点本地盘，随节点消失 |
| **节点故障：重建能力** | ✅ 有源可重建（镜像集中 + PVC 跨节点 + K8s 自动调度） | ⚠️ 只能从模板重建全新沙箱，暂停态无法恢复 |
| 跨节点恢复 | 天然支持（K8s 调度） | 单机维度（官方 roadmap：跨节点迁移 + Volume 框架是起点） |

> **核心差异**：OpenSandbox 的丢失是"可重建的丢失"（暂停态安全、卷数据安全、重建有源）；CubeSandbox 的丢失是"不可恢复的丢失"（暂停快照在本地盘，节点故障连暂停态一起消失）。

### 2.5 可扩展性 / 二次开发

| 维度 | OpenSandbox | CubeSandbox |
|---|---|---|
| 协议 | **Sandbox Protocol 公开契约**（specs/ 4 份 OpenAPI），SDK 从 spec 生成 | E2B 兼容 API（换环境变量迁移） |
| 运行时 | `SandboxService` 接口可插拔（Docker/K8s/agent-sandbox provider） | 控制面闭源二进制，配置 + 插件 |
| 插件 | 官方 OpenClaw 工具插件（13 个 sandbox_* 工具，自包含离线安装）；EvictionHandler 接口；image-committer 可定制 | Volume Plugin 框架（CSI 风格双角色 Hook）；认证回调 |
| 二次开发面 | 协议级扩展 + provider/execd/committer 可替换 | 主要靠 Volume 插件 + 配置 |

### 2.6 卷权限控制边界（池化模式）

| 模式 | OpenSandbox | CubeSandbox |
|---|---|---|
| 非池化 | ✅ `volumes[].sub_path` + `readOnly` 可实现"共享 PVC + 每沙箱写自己子目录"（`volume_helper.py`） | ✅ nested mount 原生 |
| 池化 | ❌ API 拒绝 volumes（预热 Pod 无法动态加卷）→ **S3 中间层方案**（见 §3.3） | 无池化概念，模板克隆即预热 |

---

## 3. 推荐架构与结论

### 3.1 总体结论

**"更多用户 + 服务质量 + 业务层租户控制"导向下，OpenSandbox 是主选**：

1. 租户配额 + K8s 稳定性（故障域小、暂停态跨节点存活）匹配"服务质量稳定"
2. 暂停零节点占用（快照在 registry）匹配"资源有限"
3. 协议开放 + 可插拔组件匹配"业务层二次开发"
4. 无部署前提（CubeSandbox 需裸金属 KVM）

**CubeSandbox 仅当满足全部条件才考虑**：有裸金属 KVM 节点 + 长会话重状态为主 + 磁盘充裕（暂停快照 2~3GB/个）+ 愿意自建租户配额层 + 接受节点故障=沙箱丢失。

### 3.2 架构（业务层租户控制 + namespace 审批制）

```
业务层（自研，租户控制核心）
  · 用户认证（IAM/SSO）→ 用户 → 部门 → namespace 映射（审批制，静态）
  · 配额：部门级 ResourceQuota（审批时定）+ 用户级业务层记账
  · 沙箱编排：创建/暂停/销毁/恢复，注入用户信息（taskTemplate env）
  · 产物管理：S3 用户目录（中间层静默同步）
  · 审计
        │
OpenSandbox Server（执行引擎，无租户逻辑）
  · 每部门一个 server 实例（推荐，配额/密钥/审计随 namespace 走）
  · 或单实例 + tenants 映射（部门多时省资源）
        │
K8s：审批过的部门 namespace（静态、受控）
  · ResourceQuota + LimitRange（审批时定）
  · 共享 PVC（只读数据：模型/数据集，静态预置 Pool 模板）
```

### 3.3 池化模式产物持久化（卷权限受限的出路）

| 方案 | 做法 | 适用 |
|---|---|---|
| **A. S3 中间层**（推荐） | 产物走 S3 用户目录，中间层在 create/delete 时静默同步（恢复 + 回写），权限由中间层控制 | 池化主路径 + 动态用户产物 |
| **B. 非池化 + subPath** | 低频长会话用非池化 BatchSandbox，动态模板带 subPath 挂载（注意：subPath 目录需预创建） | 吞吐要求低、需要真卷隔离 |
| **C. 混合** | 共享只读数据静态预置 Pool 模板；用户产物走 S3 | 池化主路径 + 静态共享数据 |

### 3.4 资源有限下的密度策略（OpenSandbox 路径）

1. 用完即焚模式（`poolRef + replicas=1 + taskTemplate + expireTime + policy=Release`，不走 pause/resume/快照）
2. Pool bufferMin=0（不常驻预热），poolMax 上限 + 按需扩容
3. 去 egress sidecar → K8s NetworkPolicy（内网出站目标可控，IP/CIDR 够用）
4. 镜像瘦身 + 共享解释器卷（GB 级 → 数百 MB）
5. 超卖：requests 小 / limits 大（内部可信负载 2~3 倍）
6. TTL 15~30 分钟 + postStop 钩子导出产物
7. 客户端池关闭

---

## 4. 引用文档

### OpenSandbox 仓库 wiki（机制调研）

- `wiki/opensandbox-ephemeral-sandbox-orchestration-pattern.md` — 用完即焚编排
- `wiki/opensandbox-shared-storage-interpreter-minimal-image.md` — 镜像瘦身
- `wiki/opensandbox-k8s-networkpolicy-vs-egress-sidecar.md` — 网络隔离替代
- `wiki/opensandbox-pooled-session-s3-sync-middleware.md` — S3 中间层（已部分实施）
- `wiki/opensandbox-task-template-user-info-injection-example.md` — 用户信息注入
- `wiki/opensandbox-shardtaskpatches-mechanism-and-examples.md` — 异构任务分发
- `wiki/opensandbox-create-sandbox-params-reference.md` — 池化参数说明书
- `wiki/opensandbox-sandbox-lease-and-manual-cleanup.md` — TTL/续约

### CubeSandbox 官方文档

- `docs/architecture/overview.md` — 架构
- `docs/guide/lifecycle.md` — 生命周期/AutoPause
- `docs/guide/snapshot-rollback-clone.md` — 快照/回滚/克隆
- `docs/guide/persistent-storage.md` — host mount / nested mount
- `docs/guide/volume-plugin.md` — Volume 插件框架
- `docs/guide/kubernetes/architecture.md` — K8s 部署
- `docs/zh/blog/posts/2026-08-13-cubesandbox-agent-infra-interview.md` — Agent Teams 方向访谈