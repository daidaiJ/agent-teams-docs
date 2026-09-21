# Google ax + substrate 三个月演进盘点，兼论容器快照能否增量存储

> 文档版本: v1.0
> 日期: 2026-09-21
> 状态: Draft
> 关联: 博客 [Google ax + substrate：智能体运行时调度架构分析](https://daidaij.github.io/p/google-ax-agent-runtime/)（最后更新 2026-06-07）
> 方法: 取 6/6 的 head `600320f2`（PR #190）与 9/20 的 head（PR #1742）两棵源码树做 A/B diff；运行时能力以 runc / gVisor 上游源码、Firecracker / Cloud Hypervisor 官方文档为准
> 结论标记: 【源码】= 直接读过上游源码；【文档】= 官方文档；【检索】= 二手信息，未逐条核对

---

## 1. 结论摘要

1. **博客对当时代码的描述基本准确**。6/6 的树里 `redis` 命中 167 处，当月的 roadmap 里还挂着一条待决问题“Decide: Is Redis/ValKey the right answer for API storage?”。所以后面这些差异不是博客写错，而是**这三个月真实发生的变化**。
2. **规模**：非 vendor 文件 527 → 1471；`docs/` 4 → 20+；`cmd/` 二进制 7 → 10；`demos/` 4 → 8；PR 编号 #190 → #1742（106 天，约 15 个/天）。
3. **三条主线变化**：状态库 Redis → PostgreSQL；新增 `atespace` / `tag` / `pause` 这套资源与生命周期模型；microVM（Kata + Cloud Hypervisor）从 roadmap 愿景变成可用沙箱类。
4. **仍未交付**：**增量快照**（6 月 roadmap 已列，9 月仍列）、storage tiering、空闲 actor 的 TTL 自动回收、内置 worker 扩容。
5. **博客有两处技术前提需要更正**：
   - gVisor `runsc checkpoint` **没有** `--parent` 增量模式【源码】，所以“启用 `--parent`”这条建议不可执行；能立刻用的是 `--exclude-committed-zero-pages`、`--direct`。
   - Cloud Hypervisor **没有**原生 diff snapshot【文档】。substrate 自己在它上面合成了 Firecracker 风格差分（OnDemand 恢复 + 稀疏 delta + merge），但**落盘仍是完整镜像**，不存链。
6. **能否对容器归档做增量存储**：分三层看——镜像/导出归档层天然可增量且生态成熟；文件系统层最便宜；**内存层取决于运行时是否暴露脏页跟踪**：CRIU 有、Firecracker 有、Cloud Hypervisor 与 gVisor 没有。运行时都不支持时，可用与运行时解耦的**后置差分压缩**（`zstd --patch-from`）兜底。代价是父链使 GC 变成图问题、恢复延迟随链长增长。

---

## 2. 基线与方法

| 项 | 6/6（`600320f2`, PR #190） | 9/20（PR #1742） |
|---|---|---|
| 非 vendor 文件 | 527 | 1471（+1248 / −304） |
| `docs/` | api-guide、architecture、observability、roadmap（4 个） | 20+（新增 glossary、upgrade、request-parking、threat-model、authentication、egress-trust-bundle、csi-volumes、network-egress、api-style-guide、api-validation、metrics registry、dev/best-practices…） |
| `cmd/` | ateapi、atecontroller、atelet、atenet、ateom-gvisor、kubectl-ate、podcertcontroller | 上列 + **ateom-microvm、ate-setup、benchmarking** |
| `demos/` | agent-secret、claude-code-multiplex、counter、sandbox | agent-secret 已删；新增 **parking、autoscaled-workerpool、egress、jupyter、multi-template、counter-microvm** |
| 新增依赖 | — | `jackc/pgx/v5`、`aws-sdk-go-v2/service/s3`、`openfga/openfga`、`otel`（log+metric+trace）、`envoyproxy/go-control-plane` |

复现方式（`git clone` 在本机被墙时可用）：

```bash
# 1) 取某个时间点的 head
curl -s "https://api.github.com/repos/agent-substrate/substrate/commits?until=2026-06-07T23:59:59Z&per_page=1"
# 2) 用该 SHA 下载 tar（codeload 可达）
curl -L -o june.tar.gz https://codeload.github.com/agent-substrate/substrate/tar.gz/<sha>
# 3) 与今日 main 做文件级/关键词级 diff
```

---

## 3. 逐项对照：哪些是博客之后才有的

按下表统计关键词在两棵树（排除 vendor）中的命中数：

| 关键词 | 6/6 | 9/20 | 结论 |
|---|---|---|---|
| redis | **167** | 20（全是 `rediscover` / `redistribute` 误命中） | 博客写的 Redis 在当时成立；现已移除 |
| atespace | 0 | 4516 | 全新：隔离边界 + `(atespace, name)` 寻址 |
| tag / goldenTag | 0 | 208 | 全新：不可变、保留钉、可跨 atespace 共享 |
| PauseActor / onPause | 0 | 260 | 全新：节点本地 checkpoint + 节点钉住 |
| durableDir | 0 | 215 | 全新 |
| systemInfo 卷 | 0 | 366 | 全新（actorMetadata / trustBundle） |
| CSIDriverConfig / externalVolumeTemplate | 0 | 319 | 全新（CSI 卷） |
| parking（parked request / parking lot） | 0 | 61 | 全新：替代“池满即 503” |
| atunnel | 0 | 396 | 全新：worker 内 mTLS 隧道 |
| SandboxConfig | 0 | 509 | 全新：沙箱二进制与 pause image 外置为集群资源 |
| DrainWorker | 0 | 81 | 全新：配合滚动升级 |
| RevertActor | 0 | 119 | 全新：回滚到外部快照 |
| openfga / authz | 0 | 71 | 全新：内嵌 OpenFGA，9/18 落地 |
| imagecache | 2 | 293 | 基本全新：节点级镜像缓存 |
| sparseZstd / ATESPRSE / SEEK_DATA | 0 | 49 | 全新：稀疏区段压缩上传格式 |
| microvm / Kata | 7（仅 roadmap 愿景） | 472 | 已落地 |
| incremental snapshot | 1（仅 roadmap） | 2（roadmap + 压测注释） | **仍未实现** |

状态库这次的直接证据：`cmd/ateapi/internal/store/atepg`（pgx v5）+ `cmd/ate-setup/internal/steps/postgres.go`；而 6 月的 roadmap 还在问是否继续用 Redis，性能条目也还是“用 Redis Hash Tags 做分片”。

---

## 4. 6 月 roadmap → 9 月交付情况

| 6 月列的计划 | 9 月状态 |
|---|---|
| 是否继续用 Redis 做 API 存储 | **已决断：换 PostgreSQL** |
| “Namespaces 或类似概念” | **已交付 = Atespace** |
| 除 gVisor 外支持至少一种 microVM | **已交付 = ateom-microvm（Kata + Cloud Hypervisor）** |
| S3 支持（via plugin） | **已交付**（in-tree `internal/objectstore/s3.go`）；但 9 月 roadmap 仍列在 Coming Soon → 文档滞后 |
| gVisor 快照/恢复优化 | **部分**：稀疏区段压缩、并行 zstd、分片上传、节点镜像缓存 |
| Prometheus 指标 / Session-Aware 遥测关联 | **基本**：OTLP 三件套 + actor 生命周期事件 + 每 actor 用量事件；跨进程的完整关联仍留 issue（#853/#761） |
| 系统组件间 mTLS | **已交付**：podcertcontroller 签发短期 pod 证书 |
| 凭证注入 / actor identity（via proxy） | **已交付但形态不同**：ActorIdentity 服务 `MintJWT`/`MintCert`（SPIFFE URI，仅同节点 atelet 可代签） |
| 用户授权 | **进行中**：OpenFGA 内嵌服务落地（`can_suspend`/`can_revert`/`can_set_policy`…） |
| Worker 水平扩容 | **部分**：WorkerPool 有 scale 子资源 + HPA demo（prometheus-adapter 喂 assigned worker 数），内置扩容仍未做 |
| KinD 本地开发 | **已交付** |
| 空闲 actor 的 TTL 自动回收 | **未做** |
| Disk-Only Resume Policy | **部分**：以 `Data` scope + `onResume.fromData: ColdBoot` 形式出现 |
| incremental snapshots / storage tiering / ConfigMaps as volumes | **未做** |
| Standardized DNS Mesh | **计划被替换**：改为 per-request 的 actor 引用头（`ate-target-actor`） |

---

## 5. 博客“问题清单”对账

| 博客指出的问题 | 现状 |
|---|---|
| 1. 无空闲检测/冷却期，每次请求结束即挂起 | **仍未解决**（最值得注意的遗留项）。substrate 只有显式 `SuspendActor` 与驱逐窗口；ax 侧改为声明式 `spec.suspend`。全仓库找不到 idle timeout / autosuspend / cooldown |
| 2. 挂起是 fire-and-forget，失败不回退 | **基本解决**：suspend 是服务端幂等可重入 workflow；worker 驱逐给 30 分钟挂起窗口；router 对饱和改为 park 而非 503 |
| 3. 无快照 GC | **已解决**：一个 Actor 同时只拥有一个外部快照，下次 suspend 成功后释放上一个；tag 是显式保留机制（自带副本、活得比 Actor 长）；delete/revert 清理 in-progress 残留 |
| 4. 无自动扩缩容 | **部分**：scale 子资源 + HPA demo + 容量指标（desired/ready 反 windup）；无 pending 队列的问题被 parking 补上 |
| 5. worker 死亡时状态恢复有竞态 | **大幅改善**：`CRASHED` 终态、`RevertActor`、`DrainWorker`、按 node name 解析 atelet（pod 已删也能 `Terminate`）、ateom 目录泄漏的 janitor |
| 6. ax v2 Controller 缺执行恢复 | **不复存在**：那套 controller 已被整体重写 |
| 7. 无增量快照 | **仍未做**；且博客的配方本身不成立（见 §7.2） |

---

## 6. 新代码里值得注意的实现细节

- **稀疏区段压缩上传**（`cmd/atelet/internal/ategcs/sparsezstd.*`）：自定义格式 `ATESPRSE` + 版本号在明文，之后是单个 zstd 流，元数据与**仅有的有效区段**交错写入，用 `SEEK_DATA/SEEK_HOLE` 增量发现 extents，`-1` 作结束哨兵以保持流式；配多 worker 并行 zstd。它就是为 microVM 的 `memory-ranges`（逻辑上数 GiB、绝大部分是洞）设计的。
- **microVM 差分**（`cmd/ateom-microvm/internal/ch/merge.go`）：CH 无原生 diff snapshot，于是 OnDemand（userfaultfd）恢复后新快照只含 fault 到的页，再用 `MergeSparseOverlay` 覆写回 base 的稀疏拷贝，产出**完整**镜像才上传。注释明说这是“a Firecracker-style differential snapshot implemented on top of CH”。merge 时 `remove` 旧文件再 `rename` 到不存在的名字，绕开 ext4 `data=ordered` 对“覆盖式 rename”的同步回写：约 1140ms → 115ms。
- **请求 parking**（`docs/request-parking.md`）：`ResourceExhausted`/`FailedPrecondition`/`Unavailable` 视为可重试；park budget 默认 5s、parking lot 默认 1024 槽（满则 shed 503）、同 actor 的并发请求用 singleflight 合并成一次 `ResumeActor`；ext_proc 断路器容量按 lot 两倍派生以保留快路径余量；router 优雅停机时 parked 请求仍能拿到结论。
- **驱逐窗口对齐**：worker pod 被驱逐时给 Actor `SIGTERM` + **30 分钟**完成挂起，超时转 `CRASHED`（终态，只能删）；router 的 route/idle timeout 也设为 30m + margin，两者刻意对齐。
- **快照所有权**：对象路径按 owner UID 分前缀（`atespaces/<ns>/actors/<uid>/snapshots/<name>`、`.../tags/<uid>`），删除只能作用于自己的前缀，于是“借用 tag 快照”天然安全，也能按前缀给租户做对象存储 IAM 条件。

---

## 7. 容器运行时能否对容器导出归档做增量快照存储

### 7.1 先分层，三层的答案完全不同

| 层 | 增量可行吗 | 机制 | 代价 |
|---|---|---|---|
| **A. 镜像/导出归档**（OCI 层、`docker save`/`export` 的 tar） | 可以，且最成熟 | OCI 层本身就是内容寻址的 fs diff，导出侧增量 = 只传新层 blob + manifest（registry push 即如此；`crane`/`skopeo`/`oras`）；近似 tar 可做二进制差分 `zstd --patch-from` / `xdelta3`；跨镜像去重用内容定义分块（`casync`/`desync`）或 restic/borg 式 chunk 去重 | 低 |
| **B. 文件系统/rootfs 增量** | 可以，最便宜 | overlayfs 的 upper 目录本身就是 diff；`tar --listed-incremental`；`rsync --link-dest` 硬链接快照；btrfs/zfs `snapshot` + `send -i/-p`；K8s 侧直接 CSI 卷快照 | 低 |
| **C. 进程/内存快照** | **取决于运行时是否暴露脏页跟踪** | 见 7.2 | 高（父链 GC、恢复延迟、locality） |

### 7.2 内存层：各运行时能力对照（含证据等级）

| 运行时 | 原生增量 | 机制 | 证据 |
|---|---|---|---|
| **CRIU**（runc/docker/containerd checkpoint 的底座） | **有** | soft-dirty 跟踪（`/proc/<pid>/pagemap` + `clear_refs`）+ `--prev-images-dir` 父镜像链 + `pre-dump` + `page-server` + `--auto-dedup`。runc CLI 暴露 `--parent-path`（“path for previous criu image files in pre-dump”）、`--pre-dump`、`--page-server`、`--auto-dedup`、`--lazy-pages` | 【源码】runc `checkpoint.go`；机制细节【文档】criu.org |
| **Firecracker** | **有** | `snapshot_type: Diff` + `track_dirty_pages`（KVM dirty log，退化用 `mincore`），输出稀疏文件；**不能单独恢复**，必须按创建顺序 merge 回 base；官方仍标注 developer preview；UFFD 只用于 restore 侧懒加载 | 【文档】`docs/snapshotting/snapshot-support.md` |
| **Cloud Hypervisor**（substrate 的 microVM 用它） | **无原生** | 文档化的只有 `memory_restore_mode=eager|ondemand(userfaultfd)|copyonwrite`；脏页跟踪只服务 live migration。稀疏快照属 PR 级改进（PR #8113），未进文档 | 【文档】`docs/snapshot_restore.md`；稀疏快照【检索】 |
| **gVisor (runsc)** | **无** | 不走 CRIU，自带 sentry 保存/恢复，拿不到 soft-dirty。checkpoint 全部 flag 为：`--image-path`、`--leave-running`、`--exclude-committed-zero-pages`、`--direct`、`--cuda-checkpoint-path`、`--cuda-checkpoint-sequential`、`--save-restore-exec-argv`、`--save-restore-exec-timeout`、`--fs-checkpoint-paths`。**没有 `--parent`** | 【源码】`runsc/cmd/checkpoint.go` |

补充：substrate 目前跑 gVisor 只带 `-allow-connected-on-save`，**上述省空间 flag 一个都没用**（`fscheckpoint` 的封装甚至标着 `nolint:unused`），而它把力气花在存储侧（稀疏区段 + 并行 zstd + 分片上传）。这是一处零运行时改动的优化空间。

### 7.3 运行时都不支持时的兜底：后置差分压缩

保留一份 base 全量镜像，之后每次用 `zstd --patch-from=base new.img`（或 `xdelta3`）做差分压缩。不需要运行时配合，对相似度高的内存镜像收益很大；代价是 CPU，以及 base 必须保留（于是又回到父链 GC 问题）。这也是唯一能同时覆盖 gVisor 与 CH 的路径。

### 7.4 增量的真实代价

1. **GC 从“删一个对象”变成图问题**：base 在有子快照时不能删，链尾删除、孤立分支回收都要专门控制器；
2. **恢复延迟随链长增长**，且恢复前必须把整条链拉到就近位置（locality 变成调度约束）；
3. **链断裂 = 快照永久不可恢复**，需要校验与 rebase/compact；
4. 成熟实现都会定期把链合并成新 base——用 CPU 和一次全量写换长期的小增量。

### 7.5 若要在 substrate 上做，成本从低到高

1. 给 gVisor 打开 `--exclude-committed-zero-pages`，复用已有的稀疏上传路径（零运行时改动）；
2. 上传前做 base 差分的**后置**压缩（不动快照语义，只加缓存层，可覆盖两种 sandbox class）；
3. 才轮到 microVM 侧存真正的 delta 链 + compaction（对应 roadmap 的 incremental snapshots / storage tiering）。

---

## 8. 参考

- 博客：[Google ax + substrate：智能体运行时调度架构分析](https://daidaij.github.io/p/google-ax-agent-runtime/)（2026-06-07）
- 仓库：[agent-substrate/substrate](https://github.com/agent-substrate/substrate)、[google/ax](https://github.com/google/ax)
- substrate 关键文件：`docs/{roadmap,architecture,glossary,api-guide,request-parking,upgrade,observability,authentication}.md`、`cmd/atelet/internal/ategcs/{objects,sparsezstd}.go`、`cmd/ateom-microvm/internal/ch/merge.go`、`cmd/ateapi/internal/{controlapi/workflow_*,store/atepg}`、`pkg/api/v1alpha1/workerpool_types.go`
- 上游：`opencontainers/runc` `checkpoint.go`、`google/gvisor` `runsc/cmd/checkpoint.go`、Firecracker `docs/snapshotting/snapshot-support.md`、Cloud Hypervisor `docs/snapshot_restore.md`
