# bubblewrap 的原理与功能边界

> 文档版本: v1.0
> 日期: 2026-09-24
> 状态: Draft
> 关联: [bubblewrap-empirical-verification-report.md](bubblewrap-empirical-verification-report.md)（同一轮实测的完整数据与复现方式）
> 定位: 介绍性文档。讲清楚 bubblewrap 是怎么工作的、能隔离什么、在哪里失效，供"要不要用它、怎么用它"做判断
> 证据标记: 【实测】= 在本项目节点/容器/Pod 里跑出来的结果；【上游】= bubblewrap 官方 README/man page 或发行版包元数据；【推断】= 未直接验证，按机制推导

---

## 1. 结论摘要

1. **bubblewrap 是"沙箱构造器"，不是一个成品沙箱。** 上游原话：它"不是一个完整的、现成带特定安全策略的沙箱"，保护强度**完全由传入的参数决定**【上游】。这句话是所有边界讨论的起点——安全责任在调用方。
2. **它的隔离手段是"命名空间 + 挂载视图 + 能力裁剪 + seccomp"**，共享内核。核心机制：新建 mount namespace，**根是一个宿主看不见的 tmpfs**，再用参数把需要的东西挂进去；在此之上可选开 user/PID/UTS/IPC/NET/cgroup 命名空间。
3. **它解决的是"同一宿主里多个进程互不干扰"，不是"防恶意代码逃逸"。** 防后者需要 VM 级边界（Kata/Firecracker/gVisor/smolvm），成本高一个数量级。
4. **最大的环境前提：能创建 user namespace，或者调用方已有 CAP_SYS_ADMIN。** 历史上还有 setuid 模式，**已被移除**（现行版本会拒绝以 setuid 运行）【上游】。在容器里就退化成"必须有 CAP_SYS_ADMIN"——这也是**唯一**一条真正难绕过的门槛，其余能力（网络、PID、UTS、IPC 隔离）都只是再加一两个能力位的事。
5. **功能性缺口（不是安全缺口，但决定要不要选它）**：没有 rlimit 参数（资源上限要靠启动器 `ulimit` 或 cgroup）、没有"会话"概念（目录分配/并发/超时/清理/配额都由调用方实现）、容器内通常给不了每沙箱的 cgroup 配额、不是审计/多租户组件。
6. **两条最贵的使用教训**：`--ro-bind / /` 既漏文件又让新挂载点建不出来（要"精选 rootfs"）；不显式 `--cap-drop ALL` 时沙箱会继承调用方的能力位【实测】。

---

## 2. 原理：它是怎么工作的

### 2.1 一句话机制

`bwrap <一堆参数> <命令>`：先按参数**构造一个全新的文件系统视图与命名空间集合**，再 exec 你的命令，把它的退出码当作自己的退出码返回。

### 2.2 启动时序（按代码路径顺序）

| 步骤 | 做什么 | 依据 |
|---|---|---|
| 1 | 打印参数解析、准备 fd（`--seccomp`/`--args`/`--json-status-fd` 等都从 fd 读） | 【上游】 |
| 2 | **新建 mount namespace**，根换成一个**宿主不可见、最后一个进程退出后自动清理的 tmpfs** | 【上游】 |
| 3 | 按命令行**顺序**执行文件系统操作（`--ro-bind`/`--bind`/`--tmpfs`/`--dir`/`--symlink`/`--proc`/`--dev`/`--remount-ro`…），缺父目录自动补建 | 【上游】 |
| 4 | 可选命名空间：`--unshare-user`/`-ipc`/`-pid`/`-net`/`-uts`/`-cgroup`（`--unshare-all` = user-try + ipc + pid + net + uts + cgroup-try） | 【上游】 |
| 5 | 若开了 PID ns：bwrap 自己当 **pid 1**，只做收尸（reap zombies），并监视 pid 2（你的命令）；沙箱内无进程时 pid 1 退出以完成清理 | 【上游】 |
| 6 | 设置 `no_new_privs`（PR_SET_NO_NEW_PRIVS），**关掉 setuid 提权链** | 【上游】 |
| 7 | 按 `--cap-add/--cap-drop` 处理能力位；`--new-session` 调 setsid 断开控制终端（防 TIOCSTI 注入）；`--die-with-parent` 设 PR_SET_PDEATHSIG | 【上游】 |
| 8 | 装载 `--seccomp` 给的 cBPF 过滤器（**必须是 libseccomp `seccomp_export_bpf` 的产物**） | 【上游】【实测】 |
| 9 | `--clearenv`/`--setenv` 处理环境，`--chdir` 换目录，exec 目标命令 | 【上游】 |

要点：**"根是空的 tmpfs"是理解它的关键**——沙箱默认什么都看不见，**看见的每一样东西都是显式挂进去的**。所以"最小可见面"是靠参数写出来的，不是默认给的。

### 2.3 两条特权路径（为什么容器里需要 SYS_ADMIN）

| 路径 | 前提 | 结果 |
|---|---|---|
| 非特权路径 | 能创建 user namespace | 普通用户即可用；沙箱内是"假 root"（uid 映射后） |
| root / 已授权路径 | 没有 userns 可用，但调用方有 **CAP_SYS_ADMIN** | 沙箱直接以真实 uid（容器里就是 0）运行，靠挂载 + 命名空间 + 能力裁剪隔离 |

* man page 原文：**"如果 bwrap 不是以 root 运行，则 user namespace 是必需的"**【上游】。
* **setuid 模式已废弃并移除**：历史版本支持 setuid-root 作为 userns 不可用时的兜底，现行版本默认拒绝 setuid 运行；发行版现在也都是 0755 非 setuid【上游】。相关 CVE-2026-41163 只影响"以 setuid 安装的 0.11.0/0.11.1"（ptrace 抢占 setup 阶段去调 overlay 挂载），0.11.2 起在 setuid 下直接拒绝启动【上游】。
* 由此得出容器内的现实：**要么容器允许非特权 userns，要么容器带 CAP_SYS_ADMIN**。本项目的节点上，后者是可行路径，前者取决于容器的能力集【实测，见 §3.4 与实证报告 §7】。

### 2.4 生命周期与状态回传

* `--json-status-fd FD`：按 JSON Lines 写事件——启动后写 `{"child-pid": …, <各命名空间 id>}`，子进程退出后写 `{"exit-code": …}`【上游】。这是把沙箱纳入调用方会话管理（超时、强杀、清理）的正规接口。
* `--lock-file`：沙箱存活期间持有文件锁，可用于"这个工作区正在被用"。
* `--sync-fd`/`--block-fd`：握手用，便于调用方等"沙箱已就绪"再投喂任务。
* 「**用户可见的退出码就是沙箱内那条命令的退出码**」——实测杀掉沙箱内进程时得到 `137`，符合被杀语义【实测】。

---

## 3. 功能边界：能隔离什么、不能隔离什么

### 3.1 隔离面清单（含前提与实测状态）

| 维度 | 关键参数 | 前提 | 实测结论（本项目环境） |
|---|---|---|---|
| 挂载视图 | `--ro-bind/--bind/--dev-bind/--tmpfs/--dir/--symlink/--chmod/--remount-ro/--file/--bind-data` | mount ns（bwrap 总是建） | ✅ 只读根、私有 `/tmp`、会话专属 `/workspace`；未挂载的容器文件看不见 |
| USER ns | `--unshare-user/--uid/--gid/--userns/--userns2/--disable-userns/--assert-userns-disabled` | **要看容器能力集**（见 §3.4） | ⚠️ 有条件可用：容器保留默认能力位时可用（`uid_map 0 0 1`）；`drop: [ALL]` 的姿态下报 `setting up uid map: Operation not permitted` |
| PID ns | `--unshare-pid/--as-pid-1/--pidns` | 无（有 SYS_ADMIN 即可） | ✅ 沙箱内 4~5 个进程，容器内其它进程看不见也杀不到 |
| UTS ns | `--unshare-uts/--hostname` | 无 | ✅ 生效；**只加 `--unshare-uts` 不改名**，新 UTS ns 会复制容器主机名 |
| IPC ns | `--unshare-ipc` | 无 | ✅ SysV 共享内存/信号量隔离 |
| NET ns | `--unshare-net/--share-net` | **CAP_NET_ADMIN** | ✅ 只剩 `lo`；无出网/DNS/API server；因此多会话可并行绑同一端口 |
| cgroup ns | `--unshare-cgroup[-try]` | 无 | ⚠️ 未单独验证；容器内 `/sys/fs/cgroup` 只读，意义有限 |
| 能力集 | `--cap-add/--cap-drop` | 无 | ✅ `--cap-drop ALL` 后沙箱内 `CapEff=0`；**不写则继承调用方能力位**（实测继承到 `SYS_ADMIN`） |
| seccomp | `--seccomp FD`（`--add-seccomp-fd FD` 可叠加） | 无 | ✅ 用 libseccomp 生成的 cBPF 生效（`Seccomp_filters:1`）；**手写 cBPF 会因跳转错位把 syscall 全拒** |
| no_new_privs | 强制 | 无 | ✅ `NoNewPrivs: 1` |
| 资源上限 | **无 `--rlimit-*` 选项**（0.10.0 实测 + 现行 man page 复核） | — | ⚠️ 只能靠启动器 `ulimit`（`AS/CPU/FSIZE/NOFILE` 生效）或容器级 cgroup；`NPROC` 对 root 无效（内核豁免） |
| 环境变量 | `--clearenv/--setenv/--unsetenv` | 无 | ✅ 不加 `--clearenv` 则默认继承调用方环境 |
| 会话生命周期 | `--new-session/--die-with-parent/--lock-file/--sync-fd/--json-status-fd` | 无 | ✅ 均生效 |
| 一次性工作区 | `--tmpfs DEST` | 无 | ✅ 退出即消失（0.10.0 **没有** `--tmp-overlay`，overlay 系列选项是 0.11.0 才加的上游特性） |
| 安全绑定（防 TOCTOU） | `--bind-fd/--ro-bind-fd` | 无 | 0.10.0 起提供（Flatpak CVE-2024-42472 的修复手段：调用方先把源目录开成 fd，bwrap 用 `/proc/self/fd/N` 挂载并比对 `st_dev/st_ino`）【上游】 |
| SELinux 标签 | `--exec-label/--file-label` | SELinux 环境 | 未验证 |

### 3.2 它明确不提供的东西

| 不是 | 说明 |
|---|---|
| 不是多租户安全边界 | 共享内核。防的是"同业务任务互相踩"，不防内核漏洞逃逸；要后者得上 VM 级边界 |
| 不是 uid 隔离 | 拿不到 userns 时沙箱就是**真 uid 0**，能读容器里 root 能读的一切；因此**精选 rootfs 是必需项，不是优化项** |
| 不是配额/治理组件 | 无 per-sandbox cgroup 配额（容器内 `/sys/fs/cgroup` 只读）、无 CPU/内存上限语义 |
| 不是会话管理器 | 没有"会话"概念：目录分配、并发度、排队、超时、强杀、清理、审计都由调用方写 |
| 不是网络策略工具 | `--unshare-net` 是"要么全有要么全无"；需要受限出网得自己做代理/网络策略 |
| 不做路径白名单 | 上游明确反对白名单路径的做法（路径操纵面太大），选择了"只保留少数能力位 + 始终以调用者 uid 访问文件"【上游】 |

### 3.3 使用边界（什么时候不该用）

* 需要**防恶意代码**（不信任的多租户代码、有内核提权风险）→ 用 VM 级（Kata/gVisor/Firecracker/smolvm），并确认节点有 KVM。
* 需要**每任务硬配额**（内存/CPU 精确切分）→ cgroup 或 VM。
* 需要**跨节点调度/镜像分发/审计** → 平台组件（K8s + 沙箱平台），bwrap 只是进程内一层。
* 沙箱内**必须是 uid 0** 而又不想暴露容器文件 → 精选 rootfs 是硬要求。

### 3.4 决定"能不能用"的环境前提（按踩坑顺序）

| 前提 | 缺失时的现象 | 补法 |
|---|---|---|
| mount ns / 建命名空间的能力 | `Creating new namespace failed: Operation not permitted` | 容器加 `--cap-add SYS_ADMIN` |
| AppArmor 允许 `mount` | `bwrap: Failed to make / slave: Permission denied` | `apparmor=unconfined`（Pod：`appArmorProfile: Unconfined`）；无 AppArmor 的节点（如 CentOS+SELinux）不需要 |
| `CAP_NET_ADMIN`（只在进行网络隔离时） | `loopback: Failed RTM_NEWADDR` | 加 `--cap-add NET_ADMIN` |
| userns 可用**或**有 SYS_ADMIN | `setting up uid map: Operation not permitted`（userns 建了一半）/ `Creating new namespace failed`（完全没有能力） | 二选一；容器里通常走 SYS_ADMIN 路径 |
| `--proc` 与 `--unshare-user` 同时用 | `bwrap: Can't mount proc on /newroot/proc: Operation not permitted` | 二者不要同用：要么不开 userns，要么把 `--proc /proc` 换成 `--ro-bind /proc /proc`【实测，见实证报告 §7】 |

---

## 4. 版本与构建差异（选包时先看这张表）

| 事实 | 含义 |
|---|---|
| **0.10.0 没有 `--tmp-overlay`/`--overlay`/`--ro-overlay`** | overlay 系列是 0.11.0 才进上游的；报 `Unknown option` 是版本问题，不是构建裁剪 |
| **上游从无 `--rlimit-*`** | 资源上限别指望 bwrap 参数 |
| 0.10.0 起有 `--bind-fd/--ro-bind-fd` | 需要"防路径被换掉"时用它 |
| Alpine 3.20 = bubblewrap 0.10.0（musl）；Alpine 3.23/edge 已是 0.13.x | 想要 overlay 及新修复就升 Alpine |
| Debian bookworm = 0.8.0-2+deb12u1 | 依赖 `libc6 + libcap.so.2 + libselinux.so.1`，文件含 `/usr/lib/sysctl.d/50-bubblewrap.conf`（用于放开非特权 userns）与 shell 补全 |
| **musl 二进制不能拷进 glibc 镜像** | Alpine 包只有 `usr/bin/bwrap` 一个文件（依赖 `libc.musl-x86_64.so.1` + `libcap.so.2`）；glibc 侧要用同发行版 `.deb` |

---

## 5. 与相邻方案的位置关系

| 方案 | 边界 | 适用 |
|---|---|---|
| **bubblewrap** | 命名空间 + 挂载视图 + 能力/seccomp（共享内核） | 同业务多并发的进程级隔离；单实例约 1 MiB / 数毫秒 |
| nsjail / Firejail | 同类（namespace 沙箱）；Firejail 走 setuid + 桌面特性 | 需要现成策略集、可接受 setuid 依赖 |
| runc / OCI 容器 | 命名空间 + cgroup（有配额），面向"一个容器一个负载" | 需要 cgroup 配额、镜像分发、被编排 |
| gVisor | 用户态内核（syscall 拦截），边界强于 namespace，弱于 VM | 不信任代码但节点无 KVM；有 syscall 税 |
| Kata / Firecracker / Cloud Hypervisor / **smolvm** | 独立 guest 内核（KVM 硬件边界） | 真硬件级隔离；单实例百 MiB 级、百毫秒级，节点必须有 `/dev/kvm` |

判断顺序建议：**先问"要不要防恶意代码"** → 答案是要，则看节点有没有 KVM，有就 VM 级、没有就 gVisor；答案是"只需任务间不互相踩"，则 bubblewrap 是成本最优解。

---

## 6. 最容易踩的坑（按代价排序）

1. **`--ro-bind / /`**：既把容器里所有文件（含密钥、别的会话数据）暴露给沙箱，*又*让任何新挂载点建不出来（实测 `bwrap: Can't mkdir /workspace: Read-only file system`，加 `--dir` 也一样）。正确做法是精选 rootfs（只挂 `/usr /bin /lib`、必要的 `/etc/passwd`、`/etc/group`）。
2. **忘了 `--cap-drop ALL`**：沙箱继承调用方能力位（实测继承到 `SYS_ADMIN`/`NET_ADMIN`，`CapEff=0x201000` 或 `0xa82435fb`），于是能 `mount -t tmpfs`、能 `mknod`。注意上游 man page 写着"默认不保留能力"，但**在容器内以已有能力运行的路径上实测会继承**——不要依赖默认值，显式写 `--cap-drop ALL`。
3. **忘了 `--clearenv`**：调用方的 token/口令默认继承进沙箱（实测不加即可见）。
4. **以为有 uid 隔离**：容器内能否拿到 userns 取决于容器能力集；拿不到时沙箱就是真 uid 0。
5. **`--proc /proc` 与 `--unshare-user` 同用**：实测直接失败（`Can't mount proc on /newroot/proc: EPERM`）；两者取一。
6. **`--unshare-uts` 不配 `--hostname`**：只是复制了容器主机名，等于没改名。
7. **手写 seccomp cBPF**：实测会把 syscall 全拒、沙箱卡死；只用 `seccomp_export_bpf` 的产物。
8. **拿 `--tmp-overlay` 当一次性工作区**：0.10.0 没有这个选项，用 `--tmpfs`。

---

## 7. 参考

* bubblewrap 上游 README（sandbox security / limitations 两节）与 `bwrap(1)` man page（`containers/bubblewrap`）
* Flatpak 安全公告 GHSA-7hgv-f2j8-xw87（CVE-2024-42472，`--bind-fd`/`--ro-bind-fd` 的由来）；bubblewrap GHSA-xq78-7hw4-5jvp（CVE-2026-41163，setuid 模式相关）
* Kubernetes Pod Security Standards（baseline/restricted 的能力与 AppArmor/seccomp 约束）
* 本仓实测数据与本项目环境复现方式：[bubblewrap-empirical-verification-report.md](bubblewrap-empirical-verification-report.md)
