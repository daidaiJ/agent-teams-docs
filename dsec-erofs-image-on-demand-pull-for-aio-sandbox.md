# aio 沙箱镜像按需拉取：DSec/EROFS 调研与落地可行性

> 文档版本: v1.0
> 日期: 2026-09-30
> 状态: Draft
> 关联: `opensandbox-aio-isolation-gap-analysis-and-porting.md`（aio 现状）、`opensandbox-vs-cubesandbox-selection.md`（选型背景）
> 问题: 能否把 aio（All-in-One）Docker 沙箱镜像改成 EROFS 实现按需拉取？参考 DeepSeek DSec 的相关设计。

---

## 1. DSec 是什么（事实核查）

**DSec = DeepSeek Elastic Compute**，是论文描述的生产系统，**不是开源仓库**：

- 出处：论文 "DeepSeek Elastic Compute (DSec): A Sandbox Infrastructure for Effective Agentic Training at Scale"（[arXiv 2609.22978](https://arxiv.org/html/2609.22978v1)，31 页，梁文锋署名，DeepSeek-AI + 清华）
- 用途：DeepSeek-V4 的 Agent 训练/评测沙箱基础设施，暴露 FnCall / container / microVM / full-VM 四种沙箱接口
- 规模：约 300 万沙箱/天，峰值并发 380k+，每秒创建 5000+ 沙箱
- **GitHub 上不存在官方 `deepseek-ai/dsec` 仓库**；搜到的同名项目均为无关（`ImJoke/dsec` 是第三方 CLI 工具，DSEC 另有一个双目事件相机数据集）
- 已开源部分：Rust 写的 OverlayBD/ublk 存储组件（[AgentENV/storage/overlaybd](https://github.com/kvcache-ai/AgentENV/tree/main/storage/overlaybd)）和 [3FS](https://github.com/deepseek-ai/3fs) 本身

## 2. DSec 的镜像分发设计（论文要点）

1. **不用 registry + P2P**：镜像直接放自研 3FS（Fire-Flyer 分布式文件系统），复用训练存储基础设施，省掉独立镜像分发层。3FS 顺序大 IO 强、小随机 IO 差 → 设计原则："Reads are on-demand and bulk"、"Metadata is preferably kept local"
2. **OCI → EROFS 离线转换**："Container images are converted offline from OCI into EROFS, which separates metadata from data so that metadata is kept local while image data remains in 3FS."（EROFS 的 inode 紧凑布局使元数据天然集中，本地只需存元数据；文件数据块按需从 3FS 读）
3. **层折叠**：连续的、总大小在阈值（约 3 GB）内的层离线合并成一对 metadata/data，同时保留 overlayfs whiteout 语义
4. **file-backed mount**：用 EROFS 的文件挂载模式直接挂镜像文件，消除 loop device 块映射层
5. **microVM 走另一条路**：OverlayBD 格式 + ublk 暴露，256 KiB chunk 拉取进二级本地缓存；只读基础层用 EROFS 块设备，guest rootfs = overlayfs(EROFS lower + ext4 writable upper)
6. **benchmark（论文第 8 节）**：
   - 运行时实际只访问镜像数据的 **4.2%–13.3%**；镜像周产出 >130 TB（单节点装不下）
   - 8192 容器突发创建：按需 EROFS 与全量本地基线同速（约 35 min），**eager 全拉慢 1.71×**（>60 min），磁盘写少约 57%（700 GB vs 1600 GB per node）
   - EROFS vs tar.gz 工作区 provisioning：45 vs 79 min（1.76×），tar 写流量 5.5×

## 3. EROFS 按需拉取技术版图

### 3.1 内核能力

- EROFS 自 4.19 入主线，只读 FS
- **5.16**：fscache/cachefiles 后端合入
- **5.17/5.18**：on-demand 读模式合入（`/dev/cachefiles` 的 `OPEN`+`copen` 协议，数据缺失时内核回调用户态 daemon 按需填充缓存）——即 "EROFS over fscache" 按需加载
- 注意：**fscache 按需模式正在被弃用**，迁移到 EROFS 原生按需机制（[nydus#1826](https://github.com/dragonflyoss/nydus/issues/1826)），追新内核有 API 漂移风险
- 新内核还支持 file-backed mount（直接挂镜像文件，无需块设备/loop）与 FSDAX

### 3.2 生态方案对比

| 方案 | 格式/原理 | 需转镜像 | 内核/依赖 | K8s 落地 | 基准 |
|---|---|---|---|---|---|
| **Nydus**（阿里/Dragonfly，containerd 子项目） | RAFS v6，可跑在 EROFS over fscache 上，metadata/data 分离 | 需要（nydusify / acceld 转换） | 5.16+，或 FUSE 兜底 | nydus-snapshotter 插件 | 冷启动显著优于全量（[nydus.dev](https://nydus.dev/)） |
| **eStargz + stargz-snapshotter** | OCI 兼容，层内文件重排 + 按文件懒拉 | 需要（重排破坏层压缩完整性） | 无特殊要求 | containerd snapshotter，生态最普及 | 社区实测约 10× 冷启动提升；启动后随机读性能退化 |
| **SOCI snapshotter**（AWS） | **不改镜像格式**：生成 index manifest，启动后经 overlayfs 按需从 registry 拉文件 | **不需要** | 无特殊要求 | containerd snapshotter；Fargate 原生集成 | 冷启动 20s→2.8s（7.4×），拉取时间与镜像大小基本无关 |
| **DADI / OverlayBD** | 块设备格式（Zfile + 索引），TCMU/ublk 虚拟块设备，range 拉取 | 需要（内置 convertor；turboOCI 可免转换） | TCMU/configfs 或 ublk | overlaybd-snapshotter 插件 | 块级随机 IO 好，适合 DB 类负载 |
| **DSec 自研** | OCI 离线转 EROFS，元数据本地 + 数据在 3FS 按需读，file-backed mount | 自研转换管道（层折叠、whiteout 保留） | file-backed mount + 3FS FUSE 客户端 | 非标准 K8s，自建调度 | 见第 2 节 |
| **纯 EROFS 自研**（无 fscache） | mkfs.erofs 打每层/合并层为 .erofs 文件，自定义 daemon 供数据 | 自研 | 5.16+ | 需自写 containerd snapshotter | 无现成开源 snapshotter，工程量最大 |

## 4. 关键约束清单

1. **EROFS 只读**：rootfs 必须叠可写层——overlayfs upper（tmpfs 或本地盘 ext4）。DSec 的 microVM 就是 overlayfs(EROFS lower + ext4 upper)；这会额外占写盘，需容量规划
2. **内核版本**：fscache 按需 ≥5.17/5.18；file-backed mount、FSDAX 需更新内核；fscache 按需模式正被弃用，长期方案存在内核 API 漂移风险
3. **底层存储随机小读性能**：DSec 明确指出 3FS "performs poorly on small random I/O"，所以按需读要 "bulk"（大块读 + 元数据预取 + 本地二级缓存）。若后端是普通 registry/HDD，小随机读放大更严重——启动后首跑的 latency tail 不可忽略（eStargz 在 Grab 的实测 25s vs SOCI 5s，随机读密集负载的运行期退化是真实风险）
4. **访问模式决定收益**：DSec 运行时只访问 4.2%–13.3% 镜像数据；但如果负载首次运行就要全量读（如装 wheel、跑浏览器全量资产），按需加载退化为更慢的全量下载
5. **K8s 落地**：标准路径是 containerd 的 remote snapshotter 插件（nydus/stargz/soci/overlaybd 都是这个模式），节点需装插件 + 转换/索引工具，runtime 是 CRI-containerd 而非 dockershim
6. **工程成本**：纯 EROFS 自研 = 镜像转换管道（whiteout/层折叠）+ 元数据服务/预取 + 缓存淘汰 + snapshotter 插件；DSec 是在 3FS / 38 万并发规模下才值得自研

## 5. aio 沙箱的落地路径

### 5.1 现状

OpenSandbox 目前**没有任何按需拉取代码**（全仓库 grep 无 EROFS/nydus/stargz/dentry 命中）。现有提速手段只有：

- 本地镜像缓存复用（`container_ops.py:218-285`，inspect 本地缓存、平台不匹配才重拉）
- 池预热（BatchSandbox pool + OSEP-0021 异步客户端侧预热）
- 暂停/恢复的镜像 commit（`kubernetes/charts/opensandbox/values.yaml:16-27` image-committer）
- Fastlet 预热（OSEP-0007，按 image-cache affinity 排序的 Top-K 调度）

插入点：Docker 路径在 `container_ops.py:182-216` 的 `_pull_image`；K8s 路径在 BatchSandbox 模板 + runtimeClassName（`batchsandbox_provider.py:261`）。

### 5.2 关键前提：Docker runtime 拿不到按需拉取

普通 dockerd 无法挂 remote snapshotter（SOCI/Nydus 都是 containerd 插件）。**按需拉取实质上要求 aio 生产部署走 K8s/containerd**——这与 OpenSandbox 自身"secureAccess 仅 K8s 支持"的走向一致，不构成额外架构分叉。

### 5.3 推荐路线

| 路线 | 改造成本 | 建议 |
|---|---|---|
| **SOCI snapshotter** | 零镜像转换，只需对现有 aio 镜像生成 index；需 K8s + containerd | **推荐起步**：冷启动收益与镜像大小解耦，几 GB 的 aio 镜像受益最大，Fargate 生产验证 |
| **Nydus** | `nydusify convert` 一条命令，containerd 插件即插即用 | **推荐的 EROFS 路线**：这是 "EROFS 按需加载" 唯一不用自研的产品化闭环（RAFS on EROFS over fscache），吃到 EROFS 元数据局部性/去重 |
| eStargz | 生态最普及 | 兜底选项；对浏览器/python 启动后随机读密集的负载，运行期退化风险最高 |
| DSec 式自研（OCI→EROFS + 按需数据源） | 最高 | 仅当有自建 3FS 级别高吞吐存储后端时考虑 |

### 5.4 结论

**可行，且与 DSec 场景高度同构**（海量短生命周期沙箱、单容器大镜像、运行时只碰 ~10% 镜像数据、rootfs 只读 + 可写 upper），论文数据已验证收益（eager 全拉慢 1.71×、写流量省 57%）。但对 OpenSandbox，**建议先用 SOCI/Nydus 拿到 80% 收益**，把 "OCI→EROFS + 元数据本地/数据按需" 留作有自建高吞吐存储后端时的进阶路线。

上线前必须做两件事：

1. **实测 aio 镜像首跑随机读 profile**——确认运行时访问比例接近 DSec 的 4%-13%，而不是首跑就要全量读
2. **规划可写 upper 层的容量**——EROFS 只读，overlayfs upper 落 tmpfs 或本地盘

## 6. 参考资料

- DSec 论文: https://arxiv.org/html/2609.22978v1
- DSec 开源 OverlayBD/ublk 组件: https://github.com/kvcache-ai/AgentENV/tree/main/storage/overlaybd
- 3FS: https://github.com/deepseek-ai/3fs
- EROFS 内核文档: https://docs.kernel.org/filesystems/erofs.html
- EROFS fscache 按需读（LWN）: https://lwn.net/Articles/892551/
- fscache 弃用迁移: https://github.com/dragonflyoss/nydus/issues/1826
- Nydus 演进（EROFS over fscache）: https://d7y.io/blog/2022/06/06/evolution-of-nydus/
- nydus-snapshotter: https://github.com/containerd/nydus-snapshotter
- DADI/accelerated-container-image: https://github.com/containerd/accelerated-container-image
- stargz-snapshotter: https://github.com/containerd/stargz-snapshotter
- 懒拉取实测（~10×）: https://blog.zmalik.dev/p/lazy-pulling-container-images-a-deep
- 同期参考: WeEnv 论文（agentic RL 环境）也把 Slacker/overlaybd 列为 lazy image distribution 的最近工作，该方向在 AI 沙箱赛道是公认痛点
