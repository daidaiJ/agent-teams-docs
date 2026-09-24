# bubblewrap 进程级沙箱实证验证报告

### 能力清单 · 多会话工作空间隔离 · user namespace 边界 · 节点/Pod 姿态 · K8s 非管理员可及范围 · 与 smolvm 的成本对照

| | |
|---|---|
| 文档版本 | v1.1（2026-09-24；v1.0 的 userns 与 sysctl 两条结论已在本版修正，见 §1.2） |
| 验证环境 | k3s v1.30.5 双节点集群（Ubuntu 24.04.2 / kernel 6.17 + CentOS Stream 9 / kernel 5.14）、Docker 29.6.1、Alpine 3.20、**bubblewrap 0.10.0（Alpine 构建）**、16 vCPU / 64 GB、cgroup v2 |
| 报告性质 | 一手实测为主。凡引用文档/官方声明均标注；未能实测的部分集中列在 §12 |
| 关联 | [bubblewrap-isolation-model-and-boundaries.md](bubblewrap-isolation-model-and-boundaries.md)（原理与边界介绍）；smolvm 对照的运行时行为未实测（本环境无 KVM） |
| 环境前提 | 沙箱镜像 `alpine:3.20` + apk 安装 bubblewrap 0.10.0；宿主无 `/dev/kvm`、无 `vmx/svm`；宿主 `kernel.apparmor_restrict_unprivileged_userns=1` |

---

## 1. 结论摘要

### 1.1 四个问题的直接答案

| 问题 | 结论 |
|---|---|
| **bubblewrap 能支持哪些隔离？** | 挂载视图（精确文件系统可见面）+ PID/UTS/IPC/NET 命名空间 + 能力裁剪 + seccomp + `no_new_privs`。**没有 rlimit 参数**（该 build 与上游现行 man page 均无），资源上限靠启动器 `ulimit` 或容器 cgroup。完整实测清单见 §4。 |
| **能不能对多会话隔离工作空间？** | **能，实测通过**。机制是"每个会话一份 bind 挂载"：同一路径 `/workspace` 各自独立、会话间互不可见、进程互不可见不可杀、可同时绑同一端口、环境可切断、工作区可持久可一次性。bwrap 没有"会话"概念，目录/并发/超时/清理/配额要业务层自己实现。见 §5。 |
| **我们这种节点环境支持哪些能力？** | Ubuntu 节点最小集合 = `SYS_ADMIN + NET_ADMIN` **且** `AppArmor Unconfined`（该节点的 `cri-containerd.apparmor.d` 会拦 `mount`）；CentOS worker 只要 `SYS_ADMIN + NET_ADMIN`（无 AppArmor）。默认姿态（无能力）下 bwrap **完全不可用**。见 §6。 |
| **K8s 非管理员开发能封装到什么程度？** | **零自助**：bwrap 所需的能力与 `AppArmor: Unconfined` 被 baseline/restricted 两档 Pod Security 明确拒绝（集群内已复现原文，见 §9.1）。必须平台侧放行，或由平台提供"沙箱启动器"旁车代跑。 |
| **user namespace 能不能用？**（v1.1 修正） | **取决于容器的能力集，不是"一律不能用"**：容器保留默认能力位并加 `SYS_ADMIN/NET_ADMIN` 时可用（沙箱拿到独立 userns，`uid_map 0 0 1`；`--uid 1000` 时以 uid 1000 运行且 `CapEff=0`）；`drop: [ALL]` 的姿态下报 `setting up uid map: Operation not permitted`。Docker 容器与 k3s Pod 内结果一致。见 §7。 |

### 1.2 对 v1.0 的两处修正

| 原结论 | 修正后的结论 | 证据 |
|---|---|---|
| "容器内一律拿不到 user namespace，沙箱永远是真 uid 0" | 只在 **`drop: [ALL]` + 显式加能力** 这一档姿态成立。容器保留运行时默认能力位时，userns 可用（`--unshare-user` 成功、`uid_map 0 0 1`；`--unshare-user --uid 1000` 得到 uid 1000 + `CapEff=0`） | §7.1、§7.2 逐姿态实测 |
| "沙箱内 uid 0 能写全局 sysctl，唯一有效封堵是 `--remount-ro /proc`" | 该漏洞**依赖容器 `/proc/sys` 是否只读**：`--privileged` 容器里可复现（即便 `--cap-drop ALL` 也能写，`--remount-ro /proc` 可封）；**普通 Docker 容器与本次 k3s Pod 里不可复现**——运行时自带的只读 `/proc/sys` 子挂载会保留进沙箱。`--remount-ro /proc` 仍是跨姿态的稳妥做法 | §8 四姿态对照 |

### 1.3 选型结论：容器内进程级沙箱用 bubblewrap，不用 smolvm

| | bubblewrap | smolvm |
|---|---|---|
| 隔离边界 | namespace + seccomp + 能力裁剪（**共享内核**） | 独立 guest 内核 + KVM（硬件边界） |
| 本环境能否运行 | ✅（需 §6 的权限组合） | ❌ 宿主无 `/dev/kvm`、CPU 无 `vmx/svm` |
| 单实例开销 | **≈1.2 MiB 内存 / ≈4 进程**（cgroup 直读） | **250m CPU / 160 MiB**（其 RuntimeClass 自述） |
| 冷启动 | **9.0 ms**（实测 n=200，p95 10.8 ms） | `<200 ms`（官方 README，本环境未能实测） |
| 500 并发内存 | **≈0.6 GiB**（实测 650.9 MiB） | 折算 **≈78 GiB** |
| 容器/Alpine 内 | ✅ Alpine 原生包（1 个 58 KB 二进制 + 1 个 34 KB libcap） | ❌ 仅 glibc 二进制 |
| 适合 | **同业务多并发的进程级隔离**、高密度低成本 | 不信任代码的硬件级隔离、节点级"一 Pod 一 VM" |

**一句话**：本轮问题是"同一业务的多并发任务互不干扰"，namespace 隔离是最优成本/收益点；只有在需要"硬件级边界且节点有 KVM"时才换 smolvm/Kata 一类方案。

### 1.4 落地要做的三件事

1. **平台侧开权限**：业务命名空间给 PSA 例外，允许 `capabilities.add: [SYS_ADMIN, NET_ADMIN]` + `allowPrivilegeEscalation: true`，Ubuntu 节点再加 `appArmorProfile: Unconfined`；容器级 `--memory/--cpus/--pids-limit` 由平台兜底。
2. **更干净的替代**：业务容器保持 restricted 合规，同 Pod 内加一个**沙箱启动器旁车**（只有它带特权），业务容器经本地 socket 请它 fork 沙箱（§9.4）。
3. **沙箱参数按 §11.2 模板**：精选 rootfs（不要 `--ro-bind / /`）+ `--cap-drop ALL` + `--clearenv` + 各命名空间隔离 + 启动器 `ulimit`。

---

## 2. 背景

### 2.1 业务场景

业务侧需要在**一个容器内**为"同一业务的多并发任务/会话"提供隔离的工作空间与执行环境。要防的风险是：任务之间互相看见文件、抢占端口、抢内存/CPU、读到不该读的密钥、fork 炸弹拖垮同伴。**不要求**防"恶意代码利用内核漏洞逃逸"。

### 2.2 为什么考虑 bubblewrap

bubblewrap 是 Flatpak 等场景使用的改名空间沙箱工具，被大量"AI 代码执行沙箱"采用：单二进制、按参数拼装隔离、每次调用开销极小，天然适合"每个任务一条命令"的形态。

### 2.3 为什么同时评估 smolvm

smolvm 走另一条路线：每个负载一个 microVM（libkrun + KVM），主打"不信任代码给硬件边界、亚秒冷启动"。需要确认在**我们的节点环境**里它是否可用、以及两者在多并发场景下的成本差异。

### 2.4 为什么还要问"K8s 非管理员能做什么"

业务侧开发通常只有 namespace 级权限，而 Pod Security Standards 会限制 `securityContext`。所以"能不能用"不取决于技术可行性，而取决于**平台是否放行**。这一点必须实测。

---

## 3. 验证环境与方法

| 项目 | 值 |
|---|---|
| 集群 | k3s v1.30.5+k3s1，2 节点；containerd 1.7.21-k3s2 |
| 节点 A | `extvdiadmin-vmware-virtual-platform`，Ubuntu 24.04.2 LTS，kernel 6.17.0-14，**AppArmor 启用且容器应用 `cri-containerd.apparmor.d`（enforce）**，控制面 |
| 节点 B | `k3s-103`，CentOS Stream 9，kernel 5.14，**无 AppArmor，SELinux `spc_t`**，worker |
| 宿主资源 | 16 vCPU / 64 GB RAM / cgroup v2；Docker 29.6.1（systemd cgroup driver） |
| KVM | **不存在**：无 `/dev/kvm`、`/proc/cpuinfo` 无 `vmx/svm`、宿主 sudo 需密码（无法开嵌套虚拟化） |
| 宿主非特权 userns | **受限**：`kernel.apparmor_restrict_unprivileged_userns=1`，非 root `unshare -U -r` 被拒 |
| 验证路径 | ① 容器内（Docker + apk 安装的 bwrap）② Pod 内（k3s + `alpine:3.20` + apk） |
| 性能测量 | cgroup v2 文件直读（`memory.current`/`pids.current`），不用 `docker stats` |

**可信度声明**：本文"实测"均为上述环境的一次执行结果；官方/文档数据均标注来源；未验证项见 §12。

---

## 4. 能力清单与实测状态

| # | 隔离维度 | 关键参数 | 本环境可用性 |
|---|---|---|---|
| 1 | **挂载 ns / 文件系统视图** | `--ro-bind --bind --dev-bind --tmpfs(±--size/--perms) --dir --symlink --chmod --remount-ro --file --bind-data` | ✅ 根只读、`/tmp` 私有、`/workspace` 受控，容器内其它文件不可见 |
| 2 | **PID ns** | `--unshare-pid --as-pid-1 --pidns` | ✅ 沙箱内仅 4~5 个进程，pid1 = bwrap reaper；看不到也 `kill` 不到容器内其它进程 |
| 3 | **UTS ns** | `--unshare-uts --hostname` | ✅ `--hostname job-1` 生效且不影响容器。**只加 `--unshare-uts` 不改主机名** |
| 4 | **IPC ns** | `--unshare-ipc` | ✅ SysV 共享内存/信号量隔离 |
| 5 | **NET ns** | `--unshare-net --share-net --unshare-all` | ✅ 只有 `lo`；出网/DNS/K8s API 全不可达；多会话可同端口并发 |
| 6 | **Cgroup ns** | `--unshare-cgroup[-try]` | ⚠️ 未单独实测（容器内 cgroup 只读，意义有限） |
| 7 | **User ns** | `--unshare-user --uid/--gid --userns/--userns2 --disable-userns --assert-userns-disabled` | ⚠️ **视姿态而定，见 §7** |
| 8 | **能力集** | `--cap-add/--cap-drop` | ✅ `--cap-drop ALL` 后沙箱内 `CapEff=0`；**不写则继承调用方能力位**（实测 `0x201000` / `0xa82435fb`） |
| 9 | **seccomp** | `--seccomp FD --add-seccomp-fd FD` | ✅ `Seccomp_filters:1`，`mkdir()` 被 EPERM，其它 syscall 正常（须用 libseccomp `seccomp_export_bpf` 产物） |
| 10 | **no_new_privs** | 强制 | ✅ `NoNewPrivs: 1` |
| 11 | **资源上限** | **无 `--rlimit-*`** | ⚠️ 靠启动器 `ulimit`：`AS/CPU/FSIZE/NOFILE` 生效；`NPROC` 对 root 无效（内核豁免，沙箱内外一致） |
| 12 | **会话生命周期** | `--new-session --die-with-parent --lock-file --sync-fd --json-status-fd` | ✅ `--json-status-fd` 实测输出 `{"child-pid":…,"pid-namespace":…}` 与退出码 |
| 13 | **环境与其它** | `--clearenv --setenv --unsetenv --chdir --argv0` | ✅ `--clearenv` 可切断调用方环境变量泄漏 |
| 14 | **一次性工作区** | `--tmpfs`（`--tmp-overlay` 该版本没有） | ✅ 退出即消失，底层目录无变化 |

**两条必须记住的性质**：① 上游自述 bubblewrap **不是**成品策略，"保护强度完全取决于传入参数"；② 拿不到 userns 时，**沙箱进程在容器内是真 uid 0**——所以**绝不能** `--ro-bind / /`。

---

## 5. 多会话工作空间隔离（两节点各跑一遍，结果一致）

**形态**：业务 App 作 launcher；每次会话一条 bwrap：精选只读系统目录 + 私有 `/tmp` + 会话专属 `/workspace`（bind 到 `/srv/sessions/<sid>`）+ 只读共享数据 + 受控可写交换目录 + 全套命名空间隔离。

| 测试项 | 结果 |
|---|---|
| 同一路径 `/workspace` 内容互不干扰 | ✅ 两个会话都看到 `/workspace/id.txt`，内容分别是 `s1-content` / `s2-content` |
| 会话看不到别的会话工作区 | ✅ 沙箱内 `/srv/sessions` 不存在；`/` 下只有 `bin dev etc lib proc shared tmp usr workspace` |
| 共享只读数据集 | ✅ 可读；写入被拒（`Read-only file system`） |
| 受控可写交换目录 | ✅ 会话写入后 launcher 可见；未挂载该目录的会话读不到 |
| 端口/网络隔离 | ✅ 两会话**同时**监听 tcp/8080 均成功；互相探测不到 |
| 环境变量隔离 | ✅ `--clearenv` 后 launcher 的 `SECRET_TOKEN` 不可见；**不加则继承** |
| 进程隔离 | ✅ launcher 能看到沙箱进程；兄弟会话既看不到（`/proc/<pid>` 不存在）也不能 signal |
| 工作区持久化复用 | ✅ 会话退出后同目录再次挂载可读到上次写入 |
| 一次性工作区 | ✅ `--tmpfs /workspace` 退出即消失 |
| 并发正确性 | ✅ 8/8、16/16 会话各自内容正确、netns 各不相同（全套隔离检查 18 passed / 0 failed） |
| 会话内资源上限 | ✅ 启动器 `ulimit -v 100M` 在会话内仍生效（150 MB 分配被拒） |
| 16 并发 wall time | 亚秒级，无失败 |

**四个必须知道的坑**：

1. **`--ro-bind / /` 让任何新挂载点都建不出来**（实测 `bwrap: Can't mkdir /workspace: Read-only file system`，加 `--dir` 亦然）。要么精选 rootfs（推荐），要么镜像里预先存在该目录。
2. 会话内是 **uid 0**（未开 userns 时），能读容器里 root 能读的一切。
3. `--unshare-uts` 要配 `--hostname` 才真正换名。
4. bwrap 没有会话管理：目录、并发、超时、清理、配额都要业务层实现。

---

## 6. 节点/Pod 姿态矩阵

| Pod 姿态 | Ubuntu 节点（AppArmor enforce） | CentOS worker（无 AppArmor） |
|---|---|---|
| 默认（无能力，无 seccomp 字段） | ❌ `Creating new namespace failed: Operation not permitted`（CapEff=0） | ❌ 同上 |
| drop ALL + seccomp Unconfined + apparmor Unconfined | ❌ `setting up uid map: Permission denied`（无能力，命名空间建不起来） | — |
| + `SYS_ADMIN`+`NET_ADMIN`，**AppArmor 仍 enforce** | ❌ `bwrap: Failed to make / slave: Permission denied`（AppArmor 拦 `mount`） | ✅ **能跑** |
| + `SYS_ADMIN`+`NET_ADMIN`，**AppArmor Unconfined** | ✅ **能跑**（含 `--unshare-net`） | ✅ 能跑 |
| 上述再加 seccomp Unconfined | ✅ 能跑 | ✅ 能跑 |
| `hostUsers: false`（K8s 原生 userns）+ unconfined | ⚠️ API 接受但**未生效**：`uid_map=0 0 4294967295`、`ns/user` 仍指向初始 userns | — |

**要点**：

* Ubuntu 节点最小集合 = `capabilities.add: [SYS_ADMIN, NET_ADMIN]` **且** `appArmorProfile: Unconfined`。**seccomp 不用动**（不写字段 = 无过滤器）。
* CentOS worker 最小集合 = 只要 `SYS_ADMIN + NET_ADMIN`。
* `hostUsers: false` 在本集群不生效（Kubernetes 特性门槛/运行时未启用）。

### 6.1 沙箱内的残留权限

* 不加 `--cap-drop ALL` 时沙箱**原样继承**调用方能力位：`drop ALL + SYS_ADMIN + NET_ADMIN` → `CapEff=0x201000`；保留运行时默认能力位 + 两个能力 → `CapEff=0xa82435fb`。此时沙箱内可 `mount -t tmpfs`、可 `mknod`。
* 加 `--cap-drop ALL` 后 `CapEff=0`，`mount`/`mknod` 全被封。
* 容器内 `/sys/fs/cgroup` 只读 → **给不了 per-session 内存/CPU 配额**，只能靠容器级 `--memory/--cpus/--pids-limit`（实测本容器 `pids.max=77064`、`memory.max=max`）。

---

## 7. user namespace 边界（v1.1 逐姿态实测）

### 7.1 能力集决定 userns 能否建立

| 容器能力集 | 容器 `CapEff` | `--unshare-user` 结果 |
|---|---|---|
| 运行时默认能力位 + `SYS_ADMIN`+`NET_ADMIN` | `0xa82435fb` | ✅ **可用**：`ns_user` 变为新 userns，`uid_map = 0 0 1` |
| `drop: [ALL]` + `SYS_ADMIN`+`NET_ADMIN` | `0x201000` | ❌ `bwrap: setting up uid map: Operation not permitted` |
| `drop: [ALL]` + `SYS_ADMIN`+`NET_ADMIN`+`SETUID`+`SETGID` | `0x2010c0` | ❌ 同上（**加回 SETUID/SETGID 不足以修好；确切起作用的那个能力位未定位**） |
| 无任何能力 | `0` | ❌ `Creating new namespace failed: Operation not permitted` |

**Pod 内一致**：k3s Ubuntu 节点上，`capabilities.add: [SYS_ADMIN, NET_ADMIN]`（不 drop）→ userns 可用；`drop: [ALL] + add [SYS_ADMIN, NET_ADMIN]` → `setting up uid map` 失败。即"§6 的可用姿态"与"userns 是否可用"是两件事。

### 7.2 userns 里能拿到什么

| 参数 | 沙箱内 `id -u` | `uid_map` | `CapEff` |
|---|---|---|---|
| 无 userns（对照） | 0 | `0 0 4294967295`（初始 userns） | 继承容器能力 |
| `--unshare-user` | 0 | `0 0 1` | 继承容器能力（但在新 userns 内） |
| `--unshare-user --uid 1000` | **1000** | `1000 0 1` | **0** |
| `--unshare-user --cap-drop ALL` | 0 | `0 0 1` | 0 |

**判读**：`--uid 1000` 让沙箱以 uid 1000 运行且**能力清零**（比"假 root + 继承能力"干净）；但由于子 userns 的 1000 映射到容器 uid 0，**按 DAC 判定的文件访问仍等价于容器 root**——所以这不是文件权限上的降权，**精选 rootfs 依旧是必需项**。

### 7.3 `--unshare-user` 与 `--unshare-pid` 不能与 `--proc /proc` 同用

| 组合（userns 可用姿态下） | 结果 |
|---|---|
| `--unshare-pid --proc /proc`（不开 userns） | ✅ 正常 |
| `--unshare-user --unshare-pid`（不挂 proc） | ✅ 正常（`ns_user`/`ns_pid` 均为新命名空间） |
| `--unshare-user --unshare-pid --proc /proc` | ❌ `bwrap: Can't mount proc on /newroot/proc: Operation not permitted` |
| `--unshare-user --unshare-pid --ro-bind /proc /proc` | ✅ 正常（代价：沙箱内能看到容器的进程列表，PID 隔离的可见性收益消失） |
| `--unshare-user-try --unshare-pid --proc /proc` | ❌ 同样失败（`-try` 在这里并没有被跳过） |
| `--unshare-all --proc /proc` | ❌ 同样失败（`--unshare-all` 含 userns） |

**结论**：开 userns 就必须放弃 `--proc`（或接受绑定容器 `/proc`）；本项目 §11.2 的模板不开 userns，因此不受影响。Docker 容器与 k3s Pod 上表现完全一致。

---

## 8. 全局 sysctl 洞的姿态依赖性（v1.1 逐姿态实测）

探针只把 `/proc/sys/kernel/sysrq` **写回它当前的值**（176），不改宿主状态；四次测试后宿主 `sysrq` 仍为 176。

| 姿态 | 容器内 `/proc/sys` 子挂载 | 沙箱（`--proc /proc`，不加 `--cap-drop ALL`） | 加 `--remount-ro /proc` |
|---|---|---|---|
| `--privileged` 容器 | 无（可写） | **WRITABLE**（即使再 `--cap-drop ALL` 也能写） | **blocked** ✅ |
| Docker 默认能力 + `SYS_ADMIN`+`NET_ADMIN` | 1 个只读 | blocked（沙箱内可见 2 个 `/proc/sys` 挂载） | blocked |
| k3s Pod，默认能力 + 两个能力 | 1 个只读 | blocked | blocked |
| k3s Pod，`drop: [ALL]` + 两个能力 | 1 个只读 | blocked | blocked |

**判读**：bwrap 的 `--proc /proc` 确实会挂一个**可写的全新 procfs**，但运行时自带的只读 `/proc/sys` 子挂载会一起保留进沙箱并继续挡住写入；只有在 `/proc/sys` 本身可写的容器里（如 `--privileged`），"全新 procfs ⇒ uid 0 可写全局 sysctl"才成立——**此时它与能力无关（DAC 判定）**，`--remount-ro /proc` 是有效封堵。结论：把 `--remount-ro /proc` 当作**跨姿态的廉价保险**保留在模板里。

---

## 9. K8s 非管理员的真实可及范围

### 9.1 实测证据（集群内建 baseline / restricted 命名空间提交 Pod）

| 提交姿态 | baseline 命名空间 | restricted 命名空间 |
|---|---|---|
| `SYS_ADMIN`+`NET_ADMIN`+AppArmor/Seccomp Unconfined（bwrap 可用姿态） | ❌ `non-default capabilities (…must not include "NET_ADMIN", "SYS_ADMIN")`、`forbidden AppArmor profiles (…must not set AppArmor profile type to "Unconfined")`、`seccompProfile (…must not set … "Unconfined")` | ❌ 上述全部 + `allowPrivilegeEscalation != false`、`runAsNonRoot != true`、`runAsUser=0` |
| 只加 seccomp Unconfined | ❌ `seccompProfile (pod must not set … to "Unconfined")` | ❌ 同上 + 非 root 要求 |
| 只加 apparmor Unconfined | ❌ `forbidden AppArmor profiles` | ❌ 同上 + 非 root 要求 |
| drop ALL + RuntimeDefault 双 profile（合规姿态） | ✅ 可创建 | ❌ 需 `runAsNonRoot: true` 且 `runAsUser≠0` |
| 非 root + drop ALL + RuntimeDefault | — | ✅ 可创建 |

### 9.2 政策依据（Kubernetes Pod Security Standards）

* Baseline 能力白名单仅：`AUDIT_WRITE CHOWN DAC_OVERRIDE FOWNER FSETID KILL MKNOD NET_BIND_SERVICE SETFCAP SETGID SETPCAP SETUID SYS_CHROOT` —— **不含 SYS_ADMIN / NET_ADMIN**。
* Baseline AppArmor 仅允许 `Undefined/RuntimeDefault/Localhost` —— **禁止 Unconfined**；Seccomp `must not be explicitly set to Unconfined`。
* Restricted 追加：`allowPrivilegeEscalation=false`、`runAsNonRoot=true`、`runAsUser≠0`、`drop: ALL`（仅可加回 `NET_BIND_SERVICE`）。

### 9.3 非管理员可及范围阶梯

| 档位 | 前提 | 能得到的隔离 |
|---|---|---|
| **L0** 零审批（restricted 合规） | 什么都不加 | ❌ **进程级命名空间隔离为零**：连 `unshare -m` 都被拒（无 CAP_SYS_ADMIN）。只剩 rlimit、环境清理、`emptyDir` 目录隔离。要会话隔离只能"**一会话一个 Pod**"，代价是每会话一份 Pod 开销 |
| **L1** 平台放行（本环境可用档） | PSA 例外 + RBAC 允许 `SYS_ADMIN/NET_ADMIN` + `allowPrivilegeEscalation`（Ubuntu 节点再加 AppArmor Unconfined）；容器级配额由平台设 | ✅ **bwrap 在本环境的全部可用能力**：文件系统视图、私有 `/tmp`、PID/UTS/IPC/NET 隔离、能力裁剪、seccomp、rlimit、多会话同端口并发。仍无 per-session cgroup 配额 |
| **L2** 平台支持 userns | 容器保留运行时默认能力位（不要 `drop: ALL`），并允许非特权 userns（本集群 `hostUsers:false` 实测未生效） | 可拿"假 root"或 `--uid 1000` 零能力沙箱；仍共享内核，且需放弃 `--proc /proc`（§7.3） |
| **L3** 平台提供 RuntimeClass | kata / gVisor / microVM（需节点能力，如 KVM；本环境无 `/dev/kvm`） | 内核/硬件级边界；成本最高 |

### 9.4 给平台的申请方案（二选一）

**方案 A（简单）**：业务命名空间加 PSA 例外（或指定独立"沙箱专用命名空间"），允许 `capabilities.add: [SYS_ADMIN, NET_ADMIN]`、`allowPrivilegeEscalation: true`，Ubuntu 节点加 `appArmorProfile: Unconfined`；容器级 `--memory/--cpus/--pids-limit` 由平台设置。

**方案 B（推荐，权限收敛）**：业务容器保持 restricted 合规；同 Pod 内加**沙箱启动器旁车**（只有它带特权与 unconfined profile），业务容器经本地 socket 请求它启动/管理沙箱。高权限只落在一个受控容器，业务代码零特权，审计面最小，也不需要放开整个命名空间。

---

## 10. 成本数据（实测）

| 指标 | 数值 |
|---|---|
| 沙箱冷启动（完整命名空间 + 私有 `/tmp`，n=200） | **avg 9.01 ms / p50 8.73 ms / p95 10.80 ms / max 16.94 ms** |
| 基线 | 裸 `fork+exec /bin/true` 1.45 ms；`unshare -p --mount-proc` 1.98 ms |
| 并发 64（每任务固定 CPU 量） | 沙箱 0.78 s vs 无沙箱 0.66 s（**+18%** wall，CPU 8.4 s vs 7.7 s） |
| 并发 256 | 沙箱 2.84 s vs 无沙箱 2.46 s（**+15%** wall，CPU 34.8 s vs 32.3 s，+8%），每任务 ≈11 ms |
| 单沙箱常驻内存（cgroup v2 直读） | 空容器 38.1 MiB → 100 沙箱 162.0 MiB → **1.24 MiB/沙箱**（anon 0.68 MiB）；500 沙箱 650.9 MiB（1.23 MiB/沙箱），**≈4 进程/沙箱** |
| 256 并发计算任务 | 沙箱 210.7 MiB vs 无沙箱 106.7 MiB |
| smolvm 参照（官方 RuntimeClass） | `podFixed: cpu 250m / memory 160Mi` → 500 并发折算 ≈78 GiB |

---

## 11. 落地配置建议

### 11.1 容器/Pod 侧（L1 档）

```yaml
# Pod（L1 档；Ubuntu 节点）
spec:
  securityContext:
    runAsUser: 0
    appArmorProfile: {type: Unconfined}       # Ubuntu 节点必需，CentOS 不需要
    seccompProfile: {type: Unconfined}        # 本集群非必需；跨集群默认过滤器时才需要
  containers:
  - name: biz
    securityContext:
      allowPrivilegeEscalation: true
      capabilities: {add: ["SYS_ADMIN", "NET_ADMIN"]}   # 注意：要 userns 就别 drop ALL（§7.1）
```

容器运行参数兜底：`--memory`、`--cpus`、`--pids-limit`（沙箱内设不了 cgroup 配额）。

### 11.2 每个会话的 bwrap 命令（本环境实测可用）

```bash
SID=job-1; WS=/srv/sessions/$SID; mkdir -p "$WS"

bwrap \
  --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib \
  --ro-bind /etc/passwd /etc/passwd --ro-bind /etc/group /etc/group \
  --proc /proc --remount-ro /proc \
  --dev /dev --tmpfs /tmp --size 134217728 \
  --dir /workspace --bind "$WS" /workspace --chdir /workspace \
  --ro-bind /srv/dataset /shared \
  --unshare-pid --unshare-uts --unshare-ipc --unshare-net \
  --hostname "sandbox-$SID" \
  --die-with-parent --new-session --cap-drop ALL \
  --clearenv --setenv HOME /workspace --setenv SESSION_ID "$SID" \
  --setenv PATH /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  --json-status-fd 3 3>"/var/run/session-$SID.json" \
  sh -c 'ulimit -v 524288 -f 102400 -t 300; exec /opt/app/worker'
```

要点：精选 rootfs（不要 `--ro-bind / /`）、`--cap-drop ALL`、`--clearenv`、`--remount-ro /proc`、`--hostname` 与 `--unshare-uts` 配对、资源上限走启动器 `ulimit`。要 seccomp 时用 `--seccomp 4 … 4<filter.bpf`（`seccomp_export_bpf` 产物）。

### 11.3 业务层需要自己实现的部分

会话目录分配与回收、并发度控制与排队、超时与强杀（配合 `--die-with-parent`）、会话日志与状态（`--json-status-fd`）、配额核算（容器级配额 ÷ 并发数）、异常退出后的残留清理。

---

## 12. 风险、边界与未验证项

### 12.1 必须接受的边界

1. **bwrap 不是多租户安全边界**（共享内核）：防"同业务任务互相干扰"，不防内核漏洞逃逸。
2. 未开 userns 时沙箱在容器内是 **uid 0**：能读容器内一切 → **精选 rootfs 是必需项**。开了 userns 但映射为"子 uid → 容器 0"时，DAC 语义同样等价于容器 root（§7.2）。
3. `NPROC` 对 root 无效 → 单沙箱进程数只能靠容器级 `pids.max` 兜底。
4. 容器内无 cgroup 写权限 → 无 per-session 内存/CPU 硬上限。
5. bubblewrap 0.10.0 无 `--tmp-overlay`（0.11.0 才进上游）、无 `--rlimit-*`（上游从来没有）；换发行版需按实际版本复核。

### 12.2 本次未验证

1. **让 userns 可用的确切能力位未定位**：默认能力位（含 SYS_ADMIN/NET_ADMIN）可用、`drop ALL` + 两个能力不可用、再加 SETUID/SETGID 仍不可用；机制解释为推断。
2. **CentOS worker 上的 userns 行为未测**（该节点本轮只做过只读排查，未创建资源）。
3. `--unshare-cgroup`、`--add-seccomp-fd`、`--file/--bind-data`、`--bind-fd/--ro-bind-fd`、`--userns/--pidns`（复用已有命名空间）、`--disable-userns` 未逐一验证；`--overlay*` 该版本不支持。
4. **非 root 会话（`runAsUser≠0`）未测**：apk 安装需要 root，且 drop ALL 的 Pod 里 `su` 也失败（`su: can't set groups`）。
5. **smolvm 的运行时行为全部未实测**（无 KVM）：`<200ms` 冷启动、弹性内存、网络/凭据策略均为官方文档数据。
6. 未检查集群是否另装 Kyverno/Gatekeeper 等准入策略（本次只按 PSA 判定）。
7. 集群现状提示：**所有既有命名空间都没打 PSA 标签**（等于 privileged），"能否建成"当前实际取决于 RBAC 与平台约定；一旦按标准打标签即为 §9 结论。
8. 生产语义（配额治理、镜像分发、审计日志、跨节点漂移、升级路径）不在本次范围。

---

## 13. 复现方式（内联命令，无需脚本包）

前置：一个带所需姿态的容器（`--cap-add SYS_ADMIN --cap-add NET_ADMIN --security-opt apparmor=unconfined --security-opt seccomp=unconfined`），内装 bwrap（`apk add --no-cache bubblewrap`）。

```bash
# ① 姿态自检：哪几行 OK
unshare -m --propagation private /bin/true && echo "mount ns OK"
bwrap --ro-bind / / --dev /dev --tmpfs /tmp --unshare-pid --proc /proc /bin/true && echo "沙箱 OK"
bwrap --ro-bind / / --dev /dev --tmpfs /tmp --unshare-pid --proc /proc --unshare-net /bin/true && echo "网络隔离 OK"

# ② 隔离断言
bwrap --ro-bind / / --dev /dev --proc /proc --tmpfs /tmp --unshare-pid --die-with-parent sh -c 'touch /pwn' 2>&1 | tail -1   # 期望 Read-only file system
bwrap --ro-bind / / --dev /dev --proc /proc --unshare-net --die-with-parent sh -c 'kill -0 1' 2>&1 | tail -1                # 期望看不到容器进程

# ③ userns 姿态（对照 §7）
bwrap --ro-bind / / --dev /dev --die-with-parent --unshare-user sh -c 'readlink /proc/self/ns/user; cat /proc/self/uid_map'
bwrap --ro-bind / / --dev /dev --die-with-parent --unshare-user --uid 1000 sh -c 'id -u; grep CapEff /proc/self/status'
bwrap --ro-bind / / --dev /dev --proc /proc --unshare-user --unshare-pid --die-with-parent /bin/true 2>&1 | tail -1     # 期望 proc 挂载失败

# ④ 多会话：同路径不同内容 / 同端口并行 / 进程互不可见
for s in s1 s2; do mkdir -p /srv/$s; echo "$s-content" > /srv/$s/id.txt; done
for s in s1 s2; do bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib \
  --proc /proc --dev /dev --tmpfs /tmp --dir /workspace --bind /srv/$s /workspace \
  --unshare-pid --unshare-uts --unshare-ipc --unshare-net --cap-drop ALL --die-with-parent \
  sh -c 'cat /workspace/id.txt' & done; wait

# ⑤ 全局 sysctl 洞（探针只写回原值，不改宿主状态；对照 §8）
bwrap --ro-bind / / --dev /dev --tmpfs /tmp --proc /proc --die-with-parent \
  sh -c 'V=$(cat /proc/sys/kernel/sysrq); echo "$V" > /proc/sys/kernel/sysrq 2>/dev/null && echo WRITABLE || echo blocked'
bwrap --ro-bind / / --dev /dev --tmpfs /tmp --proc /proc --remount-ro /proc --die-with-parent \
  sh -c 'V=$(cat /proc/sys/kernel/sysrq); echo "$V" > /proc/sys/kernel/sysrq 2>/dev/null && echo WRITABLE || echo blocked'
```

**K8s 侧不落地的准入探测**（服务端 dry-run，不调度）：

```bash
kubectl create ns sandbox-probe
kubectl label ns sandbox-probe pod-security.kubernetes.io/enforce=baseline pod-security.kubernetes.io/enforce-version=latest
kubectl apply -n sandbox-probe --dry-run=server -f- <<'YAML'
apiVersion: v1
kind: Pod
metadata: {name: sandbox-posture}
spec:
  restartPolicy: Never
  securityContext: {runAsUser: 0, appArmorProfile: {type: Unconfined}, seccompProfile: {type: Unconfined}}
  containers:
  - name: c
    image: alpine:3.20
    command: ["true"]
    securityContext:
      allowPrivilegeEscalation: true
      capabilities: {add: ["SYS_ADMIN", "NET_ADMIN"], drop: ["ALL"]}
YAML
kubectl delete ns sandbox-probe
```

`forbidden: violates PodSecurity …` 表示该姿态在被标签约束的命名空间里不可达；输出 `pod/… created (server dry run)` 表示可达。把 `enforce=restricted` 再跑一次可看到追加的违规项。

---

## 14. 离线部署包与资源状态

### 14.1 bubblewrap 离线包（Alpine/musl x86_64）

`bwrap-offline-x86_64.tar.gz`（99,195 B，SHA256 `93c09c61…7efd56`，共 9 个文件）：

| 文件 | 大小 | 作用 |
|---|---|---|
| `bin/bwrap` | 59,528 B | bubblewrap 0.10.0（Alpine/musl x86_64，**非 setuid**） |
| `lib/libcap.so.2` | 34,632 B | 唯一的非 libc 依赖（Alpine 包名 `libcap2`） |
| `apks/bubblewrap-0.10.0-r0.apk` | 26,902 B | 官方签名包（可离线安装） |
| `apks/libcap2-2.78-r0.apk` | 21,849 B | 依赖包原件 |
| `seccomp/example-mkdir-block.bpf` / `.c` | 72 B / 1,090 B | 示例 cBPF（拦 `mkdir`→EPERM）与生成源码 |
| `scripts/smoke-test.sh` / `session-sandbox.sh` | 4,856 B / 3,121 B | 可选自检与多会话启动器参考实现（可删） |
| `README.md` / `manifest.txt` | 8,046 B / 924 B | 用法与边界 / 清单与 SHA256 |

**验证（无网络 `--network none` 的纯净 `alpine:3.20`）**：

| 场景 | 结果 |
|---|---|
| 只解压不装包（`SYS_ADMIN/NET_ADMIN/apparmor/seccomp unconfined`） | 自检 **9 passed, 0 failed**，exit 0 |
| 包内 apk 离线安装（`apk add --allow-untrusted --no-network ./apks/*.apk`） | 装到 `/usr/bin/bwrap`，**9 passed, 0 failed** |
| 无权限容器（默认姿态） | 立即 `[ABORT]` 并提示所需权限，exit 1（**不假通过**） |

**关键结论**：Alpine 目标上"必要的 bubblewrap"实际只有 **1 个 58 KB 二进制 + 1 个 34 KB 的 libcap**；glibc 发行版不能复用该包（Debian bookworm 实测为 0.8.0，依赖 `libc6 + libcap.so.2 + libselinux.so.1`）。

### 14.2 测试资源状态（已释放）

* Ubuntu 测试机：测试容器（含本轮 `bwuserns`/`bwpriv`/`bwcapsA-C` 等）与临时探针脚本全部删除；无遗留进程；宿主 `kernel.sysrq=176`（本轮探针只把该值写回自身，未改变宿主状态；176 亦为 `/etc/sysctl.d/10-magic-sysrq.conf` 的设定）；剩余磁盘 41 G。
* k3s 集群：`bw-verify`、`bw-sbx-test`、`bw-sbx-baseline`、`bw-sbx-restricted` 等探针命名空间与全部 Pod 已回收，无残留对象，两节点 `Ready`。
* CentOS 节点：仅做过只读排查，未创建任何资源。
* 保留：`alpine:3.20` 基础镜像（12 MB，便于复跑）。
