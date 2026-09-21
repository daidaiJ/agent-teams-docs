# Agent 沙箱状态持久化与快照选型：microVM 运行时成本、对象存储友好性、逐轮持久化

> 文档版本: v1.0
> 日期: 2026-09-21
> 状态: Draft
> 关联: [google-ax-substrate-drift-and-snapshot-incrementality.md](google-ax-substrate-drift-and-snapshot-incrementality.md)（同一轮 substrate 调研的演进盘点部分）
> 约束前提（本文讨论的输入）: 不需要保留内存态；存储 I/O 要低；快照与恢复要快；绝大多数 actor 来自**同一个基座镜像模板**；可写子目录基本**只有一个写来源**；可能要求**每个问答存一次**
> 结论标记: 【源码】= 读过上游/本仓源码；【文档】= 官方文档；【检索】= 二手信息未逐条核对

---

## 1. 结论摘要

1. **"microVM 是不是重复挂起/恢复成本最低"要分维度答**：单次挂起的写量与恢复的启动路径上，KVM 系 microVM 通常最低（VMM 掌控 guest RAM，可稀疏 + userfaultfd 按需）；但**"挂起 N 次之后的总传输体积"最低的是能把差分落盘的方案**（Firecracker diff、CRIU 父链），而不是"microVM"这个类本身。
2. **substrate 的 microVM = Kata Containers + Cloud Hypervisor**（不是 Firecracker），默认 sandbox class 反而是 gVisor。
3. 在"不要内存态"的前提下，**推荐配置是 `onCommit: DATA` + `onResume.fromData: GOLDEN`（或 `COLD_BOOT`）**；每 actor 快照只搬卷的 tar，最贵的那一环（CH 快照 + `MergeSparseOverlay` + 上传常驻集）被整段砍掉。
4. **每 actor 打包整个可写目录对对象存储是"形态友好、字节浪费"**：单对象顺序 PUT 是对象存储最喜欢的形态，痛点是每次全量重传（对象不可变、无服务端去重）。容量不累积（一 actor 只留一份快照），痛在写入字节与 suspend 关键路径时长。
5. **出路阶梯**：状态外置（CSI 卷，快照归零）→ 分层可变性 + 排除项 → 客户端 CDC 去重 → 对上一版做差分压缩 → overlay upper 作下一次的 lower。
6. **"rsync 到 S3 前缀"可行，且有成熟开源实现**：文件级（s5cmd/rclone/aws s3 sync、rsync-object-storage、kscb）、块级内容寻址（desync、restic、kopia）、直接挂载（JuiceFS、s3fs、Mountpoint for S3）。单写者前提让文件级方案零设计成本。
7. **"每个问答存一次"不该做成"每轮快照一次"，而应做成"每轮追加一个不可变小对象 + 周期性 compaction"**，否则成本随轮数二次增长；且请求数账单（PUT 次数）通常比字节账单先爆。

---

## 2. 快照 / 恢复成本：四方案 × 五维度

| 维度 | microVM（Firecracker） | microVM（CH，substrate 现状） | gVisor | CRIU（容器） |
|---|---|---|---|---|
| 单次**制作** | 只写脏页（KVM dirty log）【文档】 | 本地只写被 fault 过的页（不 densify），但**上传前要 merge 成完整常驻集镜像**【源码】 | 全量 checkpoint；substrate 未开 `--exclude-committed-zero-pages`【源码】 | 全量，除非用 `pre-dump` + `--prev-images-dir`【源码】 |
| 单次**恢复** | UFFD 按页 fault【文档】 | `memory_restore_mode=ondemand`（userfaultfd）【文档】 | 顺序读回镜像，**无按需分页**；`--background` 只是后台继续加载且要求未压缩 checkpoint【源码】 | 按父链读回 |
| **N 次之后**落盘体积 | 真增量（diff 稀疏文件；须 merge 回 base；dev preview）【文档】 | **常驻集量级**（merge 出的完整稀疏镜像），不随 delta 缩小【源码】 | 常驻集量级 | 只传脏页（父链）【源码/文档】 |
| 稳态开销 / 密度 | 每 VM 几 MB 级 VMM 开销，无 syscall 拦截税【检索】 | 同左；rootfs 写在 host overlay upper 上，经 virtio-fs 共享，成本落在 host page cache | 每沙箱一套 sentry/gofer + 每次系统调用的用户态内核税 | 与普通容器同 |
| 工程前提 | CPU 型号/内核兼容要求严；无 KVM 不可用 | CH 官方不保证快照跨 CH 版本可恢复；资产五件套 + `/dev/kvm` | 一个 `runsc` + pause image，OCI 直跑，无需 KVM | 依赖 runc/CRIU；容器运行时通常不暴露父链 |

只比"单次快照字节数"：**Firecracker(diff) < CRIU(父链) < CH(substrate) ≈ gVisor(全量)**。

**关于 Firecracker 的补充**（它的 diff 更低，但有代价）【文档】：

- diff snapshot 保存"自上次快照以来被访问的内存页"的稀疏文件，**官方承认可能包含多于严格脏页的页**；
- **一般不能单独恢复**，必须按创建顺序 merge 回 base（`snapshot-editor edit-memory rebase`），且 **merge 是原地更新 base**（"base is constantly updated with information from previously merged layers"）→ 想给多个 actor 分叉共享模板 base 时必须先 copy/reflink；
- diff snapshot 仍是 **developer preview**；
- 设备模型只有 VirtIO **Net / Block / Vsock**（+ serial / MMDS / balloon），上游 CHANGELOG 全历史 **virtiofs 零命中** → **没有 virtio-fs**，rootfs 只能走块设备（host 文件需预格式化）。对 substrate 这套"host overlay + virtio-fs 共享目录"的设计是硬伤，所以它选了 CH。

---

## 3. substrate 侧的具体事实（源码核对）

| 事实 | 证据 |
|---|---|
| microVM 栈 = **Kata Containers + Cloud Hypervisor**，VMM 走 unix `api-socket` 的 REST API（快照/恢复同路） | `cmd/ateom-microvm/internal/{ch/api.go,ch/ch.go}` |
| 沙箱资产（按架构各一套、带 sha256）：`cloud-hypervisor`、`virtiofsd`、`vmlinux`、`rootfs.img`、`configuration-clh.toml`（+ `pauseImage`） | `manifests/microvm/sandboxconfig-microvm.yaml.tmpl`、`hack/install-microvm-deps.sh` |
| **WorkerPool 默认 sandbox class 是 `gvisor`**，microVM 是按池/模板 opt-in | `pkg/api/v1alpha1/workerpool_types.go`（`+kubebuilder:default=gvisor`） |
| 全仓 "firecracker" 仅出现一次，且是形容词（"a Firecracker-style differential snapshot"），无 FC 支持 | `cmd/ateom-microvm/internal/ch/merge.go` |
| substrate 只接了 CH 的 `OnDemand` 与 `Copy`（eager），**未接 `copyonwrite`** | `cmd/ateom-microvm/restore.go`、`internal/ch/prefault.go` |
| 快照格式是**固定文件集**（`durable-dir.tar`、rootfs upper tar、manifest）；atelet 会从本地 FULL checkpoint 里"切出" durable tar 只上传 DATA | `internal/ateompath/ateompath.go`（`DurableDirTarFile`）、`cmd/ateom-microvm/durable.go` |
| durable / rootfs upper 的快照都是**整目录 tar，无增量、无 mtime 过滤、无块去重**（注释原文："Snapshots carry the contents as a tar of the whole per-actor directory"） | `cmd/ateom-microvm/{durable.go,rootfsupper.go}` |
| 可写目录在 host 上是普通目录，virtiofsd **write-through**，guest 一暂停 host 侧就完整可见 | `cmd/ateom-microvm/durable.go` |
| **节点级 golden 缓存代码已写但未接线**：`cmd/atelet/internal/filecache` 全仓无调用方；其文档示例正是 `URIKey(goldenSnapshotURI, fileName)` | 全仓非 vendor grep `filecache` 仅命中该包自身 |
| 节点级 **OCI 层缓存已生效**：解压一次/节点，全 actor 共享，落盘可跨 atelet 重启/节点重启，共享 page cache | `internal/imagecache/README.md` |
| golden 快照 = 模板创建时的 Full 捕获，存于保留 atespace `ate-golden` 的 tag；`PUBLISHED` tag 可跨 atespace 复用而不复制 | `docs/glossary.md`、`docs/api-guide.md` |
| 驱逐语义：worker pod 被驱逐时给 Actor `SIGTERM` + **30 分钟**完成挂起，超时转 `CRASHED`（终态，只能删） | `docs/api-guide.md` |

---

## 4. 推荐配置（两条，按"冷启动够不够快"选）

**档位 A —— I/O 下限**

```yaml
snapshotsConfig:
  onPause:  SNAPSHOT_CONTENT_SCOPE_DATA
  onCommit: SNAPSHOT_CONTENT_SCOPE_DATA
  onResume: {fromData: RESUME_SOURCE_COLD_BOOT}   # 或省略 onResume
```

每 actor 只搬 `durableDir` 的 tar；恢复靠**已共享好的节点级镜像层**冷启动。无 golden 下载、无内存镜像。

**档位 B —— 速度优先，I/O 仍小**

```yaml
  onResume: {fromData: RESUME_SOURCE_GOLDEN}      # 目前 microvm-only
```

每 actor 仍是只搬 data tar，但恢复先拉 golden（Full：内存 + rootfs delta），是"共享的 warm 起点 + 我这一小包数据"。

**必须知道的约束**（均来自源码/官方文档）：

1. **`DATA` scope 不保 rootfs**：状态必须写在 `durableDir`（host-backed virtio-fs）或 CSI 卷里；写在容器根文件系统上的东西会丢。
2. **`onCommit` 没有默认值、必须显式写**，且必须是 `onPause` 的子集（`DATA` on_pause + 未设 on_commit 会同时报 required 与 subset 两个错）。
3. **`onResume.fromData: GOLDEN` 目前 microvm-only**。
4. **gVisor 模板只允许 1 个 `durableDir`**，microVM 可多个 → "多个工作目录"这一常见需求本身就倾向 microVM。
5. 所有容器都声明 `readyz` 时，模板控制器会跳过 golden 的默认约 20s "等稳定"窗口（golden 更快就绪，后续 GOLDEN 恢复也更快）。
6. 上行字节由 `onCommit` 决定、本地 I/O 由 `onPause` 决定，可分开配：`onPause: FULL` + `onCommit: DATA` 时 atelet 从本地 FULL 集里切出 durable tar 上传，**上行不增加**，多花的是本地盘 I/O 与 pause 时长，换来同节点热恢复。

---

## 5. "一个基座模板"能拿到哪些共享

| 共享物 | 粒度 | 机制 | 状态 |
|---|---|---|---|
| golden 快照 | 每模板一份对象，全 actor 引用同一 URI；`PUBLISHED` tag 可跨 atespace 复用 | 对象存储 | 已生效 |
| OCI 镜像层 | 节点级，解压一次全 actor 共享、共享 page cache、可跨重启 | `internal/imagecache` | 已生效 |
| golden 快照对象 | 节点级缓存 | `internal/filecache`（**未接线**） | 缺口 |
| 同模板 actor 的 guest 内存页 | 节点级 CoW 共享 | CH `memory_restore_mode=copyonwrite`（substrate 未接） | 缺口 |

这两个缺口正是"一个基座模板 + 同节点 N 个 actor"模式下最值得做的两件事：前者把"每次恢复各下一遍 golden"变成"每节点一次"，后者让同节点同模板 actor 的驻留内存不再线性增长。

---

## 6. 对象存储友好性：整目录 tar 到底哪里疼

| 轴 | 表现 |
|---|---|
| **请求形态** | **友好**。单对象、顺序流式 PUT，无小文件海量 PUT、无随机写，可压缩、可 ranged/multipart。上传层还做了稀疏区段 + 并行 zstd + 分片。 |
| **字节重复** | **不友好**。对象不可变、无服务端去重；每次挂起都是一次全量写出。代价 = 写出字节 + suspend 关键路径时长（读目录 + 压缩 + 上传都是 O(目录大小)），而 suspend 时长决定 worker 何时能回收。 |
| 容量 | **不累积**：一 actor 只保留一份快照（下次 suspend 成功后删上一份），稳态 ≈ 每 actor 一份目录大小。 |

**经验判据**：目录 ≲ 几百 MB、变更比 ≳ 10~20% → 整包最划算（一次顺序写，省掉索引与链）；目录 ≫ 变更量（如 5GB 目录每次只改几十 MB）→ 明显浪费。

**出路阶梯**（代价从低到高）：

1. **不让它进快照**：状态放 `externalVolumeTemplate`（CSI 卷，substrate 已支持）或应用直写数据库/对象存储 → 挂起零字节。
2. **分层可变性 + 排除项**：只读内容进基座镜像/golden；可写目录只留真状态，cache/构建产物/临时目录排除。`tarutil.CreateFiltered` 的 skip 机制已在用（`skipWorkdir` 是先例），**扩展排除列表接近零改动**。
3. **客户端 CDC 去重**（casync/desync、restic 式 pack + index）：只上传缺失块。改 1% 的 10GB 从 10GB 降到百 MB 级。代价：chunk index、PUT 次数上升（用 pack 收拢）、恢复要拉 index。
4. **对上一版做差分压缩**（`zstd --patch-from=prev` / xdelta3）：链长=1，上传"相对上次的补丁"，定期 compact。前提是**保留上一版对象**（现在 suspend 成功会删掉上一版）且 base 本地可 mmap。
5. **上层 upper 作下一层的 lower**（架构级）：overlayfs 支持多 lower，substrate 镜像装配已在用（`internal/imagecache/bundle_linux.go` 提到 incremental `lowerdir+`）；把上次快照的 upper 作为下次 lower、只快照本轮新增上层 → rootfs 天然按周期增量。代价是回到链 + compaction。

---

## 7. rsync 到 S3 特定前缀：可行，且有成熟开源设计

**前提红利**：在 microVM 里可写目录本来就是 host 上的普通目录（virtiofsd write-through），**不需要 `docker cp`**；`docker cp` 只在"状态留在容器 rootfs、host 无路径"时才需要，而那本就该改成挂卷。

**"增量"有三种含义，先分清**：文件级（size+mtime/校验和）｜块级（CDC 去重，字节真正只传变化）｜仅扫描加速（如 restic `--parent` 只省读目录，去重是全仓级的）。

| 类别 | 代表项目（README 均已核对存在） | 增量粒度 | 单写者子目录匹配度 |
|---|---|---|---|
| 文件级 sync | `aws s3 sync`、**`s5cmd sync`**（并行度最高）、**`rclone sync`**（`--checksum`/`--backup-dir`/`--immutable`）、`mc mirror` | 文件 | ★★★★★ 无锁、last-writer-wins 可接受 |
| 块级内容寻址 | **`desync`**（casync 的 Go 实现，README 原话 "transferring only the parts that changed"；S3/GCS/Azure/HTTP 后端，chunk 扁平命名 + `.caibx/.caidx` 索引，`--seed` 复用旧数据，`prune` 回收）、`casync`、**`restic`**、`kopia`、`rustic` | CDC 分块 | ★★★★☆ 字节最省；但 restic/kopia 有**仓库级锁**，多 actor 共用一仓会争锁 → 每 actor 一仓，或用 desync 这种无状态 chunk 池 |
| 事件驱动 / 边车 | **`rsync-object-storage`**（inotify 实时 + 周期全扫，忽略规则/hot-file 合并）、**`kscb`**（rclone + crontab 轻量边车）、`VolSync`（PV 级，偏跨集群 DR） | 文件（事件触发） | ★★★★★ 天然单写者模型 |
| 直接挂载（取消 sync） | **`JuiceFS`**（数据在 S3、元数据在 Redis/SQL/TiKV；chunk→slice→block；支持 flock、close-to-open、原子 rename）、`s3fs`、**`Mountpoint for S3`**、`SeaweedFS`、`yandex k8s-csi-s3` | 无（写入即持久） | ★★★☆☆ 单写者可省掉大量语义妥协 |
| K8s 存储编排 | CSI S3 驱动 / CSI volume snapshot | 视底层 | ★★★☆☆ 对应 substrate 的 `externalVolumeTemplate` |

**Mountpoint for S3 的硬限制**（官方 SEMANTICS.md）【文档】：写入必须**从 offset 0 顺序写**；无 rename、无 chmod/chown/xattr、无 flock；覆盖需 `--allow-overwrite` 且必须 `O_TRUNC`；**追加剧仅支持 S3 Express One Zone**（`--incremental-upload`）；正在写的文件在 close 前对其他读者不可见。→ 只适合"只追加、不改名"的目录；通用 agent workspace 要用 JuiceFS/s3fs 这类带元数据引擎的实现。

**一致性的提交设计**（工具替你解决不了）：S3 自 **2024-08** 支持条件写 `If-None-Match`，**2024-11** 增加 `If-Match`（ETag CAS），**2025-10** 扩展到 `CopyObject`【检索/AWS 公告】；GCS 侧对应 `ifGenerationMatch` 前置条件。配套做法：**每轮写新 generation 前缀 → 最后条件写 `CURRENT` 指针 → 旧代按前缀/生命周期回收**。这同时解决了"半同步的树被读到"的读侧一致性问题。

**对象存储特有的坑**：请求数（10 万小文件 = 10 万 PUT + LIST，账单按请求走，需要 pack 收拢）、**没有 rename**（改名 = copy+delete）、元数据（属主/mtime/xattr/symlink/hardlink/稀疏文件，以及 overlay 的 **whiteout**）需要 sidecar 表达、跨文件压缩优势丢失（per-file 各自压）、以及它**替代不了** rootfs upper 的 whiteout 语义与内存态。

---

## 8. 每个问答存一次（per-turn 持久化）

**先纠正一个前提**：substrate **不会**为每个问答做快照（无 idle / 每请求自动挂起机制）；博客里"每请求挂起"是旧 ax 的行为，而现在的 ax store 是"资源存储 + 带 ack 的事件队列"（`SaveTask/...`、`EventQueue`），那套 conversation_log/execution_log 逐轮事件日志在重写后已不存在【源码】。所以逐轮持久化是**应用层**的事。

**要避开的反面模式**：

| 反面模式 | 代价 |
|---|---|
| 每轮 tar 整个可写目录 | T 轮 × 目录大小 D，二次增长；每轮延迟 O(D) |
| 每轮整目录 sync（s5cmd/rclone） | 每轮一次全树 LIST/比对，请求数与 CPU 随目录大小走 |
| 每轮在 substrate 层 `SuspendActor` | 等价于上面两条，还会打散 worker 池 |
| 每轮重写一个越来越大的 `conversations.json` | 对象不可变 ⇒ 每轮 O(size) PUT，同样二次 |

**推荐架构：一轮一个不可变对象 + 完成标记 + 定期 compaction**

```
<bucket>/actors/<atespace>/<actor-uid>/
  turns/000123-<sha8>.jsonl      # 一轮一个不可变小对象（对话/工具调用/产物引用）
  blobs/<sha256>                 # 本轮新增/改动的文件块（内容寻址，天然去重）
  manifests/000124.json          # 完成标记：列出本轮引用的 blob
  CURRENT                        # 指向最近一次 compaction 的基准代（可选，条件写）
```

1. **一轮内的原子性**：先写 `blobs/` 与 `turns/`，**最后写 `manifests/<seq>.json`**；读者只认有 manifest 的轮 → 半途崩溃的轮被忽略，而不是读到半新半旧的树。键里带内容哈希则连 CAS 都不需要。
2. **compaction**：每 K 轮或按大小/时间阈值把 tail 合并成新基准，旧对象按前缀/生命周期回收 → 恢复要拉的对象数不随会话长度线性增长。
3. **工作区文件不要"每轮全量"**：单写者意味着应用自己知道改了哪些文件，只 PUT 这些（内容寻址）；或直接把工作区挂成对象存储文件系统（JuiceFS 类），写入即持久、**完全没有扫描/比对步骤**。
4. **持久点分层**：本地盘 fsync（抗进程崩溃，µs~ms）+ 异步 ship（抗节点丢失）｜或同步写对象存储（把一次 RTT 放进每轮关键路径）｜若要"一轮都不丢"，就必须同步写对象存储或冗余层。

**成本按请求数算（比字节更容易先爆）**：1 PUT/轮/actor；1 万 actor × 每分钟一轮 ≈ 4.3 亿 PUT/月，按 S3 PUT 约 $0.005/千次即每月数千美元量级。压它只有两条路：**批量打包**（每 K 轮一个对象，代价是最多丢一个 pack 的恢复粒度）或放宽轮粒度。

**与 substrate 的分工**：基座/冷启动交给 golden + 节点级镜像缓存；**偶发全量**（会话结束或长空闲时一次 `Data` scope 工作区 tar）作为恢复起点；**逐轮持久**交应用自己的 append-only 日志。这与 OpenSandbox 那边 `postStop` 回写的分工一致，只是从"结束才回写"升级为"每轮追加 + 结束时全量收口"。

---

## 9. 落地顺序与验收指标

**顺序**：① 每轮一次大写入 → 一轮一个不可变小对象 + 完成标记（这一步就把二次成本变线性）；② 量 PUT 次数/每轮延迟/恢复需拉的对象数；③ 到阈值再加 compaction；④ 工作区文件改成"应用声明改动列表"或换挂载；⑤ 才考虑 CDC/delta 链。

**指标**：

- `atelet.snapshot.size`：`populated_bytes` 应随卷大小走、与 guest RAM 无关；若长期持平说明变更比低、整包在浪费。
- `ate.actor.restore.duration{ate.snapshot.phase}`：`download` 占比高 = golden 被重复下载（待 `filecache` 接线）。
- `ate.actor.checkpoint.duration{ate.snapshot.phase="persist"}`：suspend 中上传占比。
- `ate.imagecache.requests{outcome="hit"}`：单基座模式下应接近 1。
- `ate.actor.lifecycle.operation.duration{operation="resume", ate.snapshot.scope="data_on_golden"}`：A/B 两档配置的对照口径。
- 自建口径：每轮增量字节 / 目录大小（决定要不要 CDC）、PUT 次数/月（决定要不要打包）。

---

## 10. 参考

- 本仓前篇：[google-ax-substrate-drift-and-snapshot-incrementality.md](google-ax-substrate-drift-and-snapshot-incrementality.md)
- substrate：`docs/{glossary,api-guide,architecture,request-parking,observability}.md`；`cmd/ateom-microvm/{durable.go,rootfsupper.go,restore.go,internal/ch/{ch,api,prefault,merge}.go}`；`cmd/atelet/internal/filecache/`、`internal/imagecache/README.md`；`internal/ateompath/ateompath.go`；`pkg/api/v1alpha1/workerpool_types.go`；`demos/counter/counter-microvm{,-csi-test}-template.yaml`、`demos/autoscaled-workerpool/`
- 上游运行时：`opencontainers/runc` `checkpoint.go`；`google/gvisor` `runsc/cmd/{checkpoint,restore}.go`；Firecracker `docs/{design.md,snapshotting/snapshot-support.md}`；Cloud Hypervisor `docs/snapshot_restore.md`
- 开源同步/持久化：`peak/s5cmd`、`rclone/rclone`、`jorben/rsync-object-storage`、`dkruyt/kscb`、`folbricht/desync`、`restic/restic`、`kopia/kopia`、`juicedata/juicefs`、`backube/volsync`、`yandex-cloud/k8s-csi-s3`、`awslabs/mountpoint-s3`（`doc/SEMANTICS.md`）
- AWS S3 条件写公告（2024-08 `If-None-Match`、2024-11 `If-Match`、2025-10 `CopyObject`）
- 本仓相关：`OpenSandbox/changes/pooled-session-s3-sync.md`（池化会话 S3 静默同步：Ready 后注入回写脚本 + 后台 `aws s3 sync` + postStop 回写同一前缀）
