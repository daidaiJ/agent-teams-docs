# Grok Build 记忆管理设计研究

> 调研对象：grok-build-proxy 本地 fork（`D:\pro\grok-build-proxy`），上游同步快照 2026-09-22。
> 核心实现：`crates/codegen/xai-grok-memory`（25 个源文件）；会话侧接线在 `xai-grok-shell/src/session/`。

## 1. 总体架构：双模式（Legacy / V2）

由 `MemoryMode`（`xai-grok-config-types/src/memory.rs`）一次性解析并固定，运行中的会话不可切换：

| | Legacy（默认） | V2（在建） |
|---|---|---|
| 根目录 | `~/.grok/memory/` | `~/.grok/memory-v2/`（完全隔离，互不读写） |
| 内容形态 | 自由 Markdown 摘要 | topics + 不可变 observation 流 + 生成的 manifest |
| 检索 | 混合搜索（FTS5 + 向量） | 预算化 manifest 直接进上下文 + 词法索引 |

开关：`GROK_MEMORY` 环境变量、`[memory] enabled`、远程设置。禁用时整个 crate 不初始化。

## 2. 数据布局

**Legacy**：

```text
~/.grok/memory/
  ├── MEMORY.md                          # 全局精选知识（跨项目）
  └── {project-slug}-{hash8}/            # 工作区目录名如 xai-a3f7b2c9（blake3 前 8 位）
      ├── MEMORY.md                      # 项目级精选知识
      └── sessions/YYYY-MM-DD-{slug}-{sid8}.md   # 会话日志
```

**V2**（每个 scope——global 或 workspace——一套）：

```text
{scope}/
  ├── MEMORY.md                  # 生成的 manifest（只读标记 "Do not edit"）
  ├── topics/                    # 主题化知识
  ├── observations/_inbox/       # 不可变 observation 收件箱
  ├── archive/
  ├── memory_state.sqlite        # 捕获队列/持久状态（schema v3）
  └── index.sqlite               # 词法索引
```

## 3. Legacy 管线：写入侧

三个写入源，全部产出 Markdown：

- **Pre-compaction flush**（`flush.rs`）：token 用量到达"压缩阈值 − soft headroom"时触发，
  每个压缩周期最多一次。系统提示要求模型从对话中提取"决策与理由 / 技术上下文 /
  调试技巧 / 问题与解法"，明确排除用户偏好（归全局记忆）和瞬时进度，无事可写回 `NO_REPLY`。
- **Session-end 保存**（`save_on_end`）：会话结束时写摘要。
- **autoDream 整合**（`dream.rs` + `dream_lock.rs`）：三重门控（enabled → 距上次整合 ≥
  `min_hours` → 累计会话数 ≥ `min_sessions`，先查最便宜的），跨进程 `DreamLock` 锁定，
  把多个 session 日志蒸馏回 MEMORY.md。这是"记忆去噪/合并"环节。

**Ephemeral 防御**：CWD 是临时目录（temp）时 workspace 写入静默跳过，避免污染记忆库。

## 4. Legacy 管线：索引与检索

**索引**（`schema.rs` / `index.rs`）：chunker 切块 → blake3 内容哈希去重 → SQLite（WAL 模式）
三张表：`meta`（维度/版本）、`chunks`（含 access_count 访问计数）、`chunks_fts`（FTS5
contentless，BM25）；sqlite-vec 可用时再加 `chunks_vec`（vec0 虚拟表，KNN）。rusqlite 连接
`!Send`，所以每次查询新开连接。

**混合搜索**（`search.rs`，8 步流水线）：

1. FTS5 关键词（永远可用）
2. 向量 KNN（可选）
3. 按 chunk_id 合并、分数归一化
4. 跳过空模板块
5. 时间衰减：global/workspace 是"常青源"免衰减，session 块按指数半衰期衰减
6. 来源权重 + 访问频率加成，`min_score` 过滤
7. MMR 多样性重排（惩罚冗余）
8. 截断到 `max_results`

**优雅降级**：向量不可用时退回纯 FTS。embedding 凭证通过 `EndpointScopedCredentials`
**fail-closed**——只保留给可信的第一方 endpoint，第三方网关下会话凭证直接丢弃（防止密钥外泄）。

**外部编辑同步**（`watcher.rs`）：notify 线程监听 `.md` 变更，`ArcSwap<HashSet>` 无锁累加
脏路径；搜索前查 `is_dirty`（一次原子读），对脏文件 reindex / 删 stale chunks。
另有 `reindex_claim` 元数据 + stale claim 秒数，防止跨进程重复 reindex 死锁。

## 5. 记忆进入上下文的三条路径

这是设计里最讲究 prompt-cache 的部分：

1. **首轮注入**：会话开始按首条用户消息搜索（问候语检测 `is_greeting`——太泛则退化为更宽的
   项目上下文查询），结果格式化为 `<system-reminder>` 块**持久化**进首条 system 消息。
   `conversation_has_memory_context` 保证已持久化的块原样复用、不再重打分——因为重新搜索会
   改变 system prompt 前缀，**打爆整个下游对话的 KV cache**。
2. **压缩后恢复**：compaction 之后重新搜索再注入（`compaction.rs`；V2 模式则重新生成
   manifest 注入）。
3. **模型主动调用**：`memory_search`（混合搜索工具）+ `memory_get`（按路径+行号读原文）两个
   工具。`MemoryBackend` trait 定义在 `xai-grok-tools`，由 shell 以 ephemeral resource 注入
   `ToolBridge`。

注入文本自带纪律声明："记忆是历史上下文而非当前计划，回溯的路径/命令/仓库状态要用活工具验证"。

## 6. V2 管线：crash-safe 捕获队列（在建）

`v2_capture.rs` 是 V2 的核心创新——**跨进程持久化的 observation 捕获**：

- Observation 类型：`User` / `Feedback` / `Project` / `Reference`，含 topic 提示、
  statement（≤1KB）、keywords、aliases、提取模型与 prompt 版本（可审计/重放）。
- **崩溃一致性协议**：observation 文件先落盘（不可变、带 job id 和期望文件数）、outcome 行后
  提交、索引游标最后推进；两个崩溃窗口都能靠 reconcile 确定性重放恢复（孤儿文件只有哈希一致的
  完整集合才被收养）。
- **manifest 预算化**：`MEMORY.md` 由 `regenerate_scope_manifest` 生成，上限 8KiB / 64 条 /
  描述 200 字节——保证注入上下文体积恒定可控（对比 Legacy 的自由增长）。
- **边界防御**：symlink 组件逐一拒绝、目录条目数上限（1 万文件/10 万条目）、网络文件系统上
  禁用捕获（共享 coordination DB 不可靠）、单 observation 16KiB。
- **接线状态**：存储层和捕获队列已实现并有完整测试（v2_tests / v2_capture_tests /
  v2_access_tests），shell 已在压缩恢复路径消费 `format_v2_memory_context`，但
  `V2CaptureStore` 在 shell 侧**尚无调用点**——捕获（会话中自动提取 observation）还没接上，
  V2 处于"地基完成、写侧待接线"阶段。

## 7. 配置面与可观测性

`[memory.*]` 全部可调：

| 配置组 | 字段 |
|---|---|
| `index` | chunk 大小 / 重叠 |
| `embedding` | provider / model / 维度 |
| `search` | 权重 / 衰减半衰期 / MMR / 来源权重 / min_score / max_results |
| `initial_injection` | enabled / min_score |
| `session` | save_on_end |
| `watcher` | enabled / stale_claim_secs |
| `gc` | max_age_days |
| `dream` | enabled / min_hours / min_sessions |
| `flush` | enabled / soft_threshold_tokens |

可观测性走 `MemoryObservationSink` trait（shell 里实现为 telemetry sink），记录每次搜索的
检索模式（FtsOnly/Hybrid）、错误分类等。

## 8. UI 层

- `/memory` 弹窗（`xai-grok-pager/src/views/memory_modal.rs`）：Global / Workspace /
  Sessions 三区浏览，支持过滤、预览（≤1MiB）、开关记忆。
- `memory_search` 工具调用在 scrollback 里有专门渲染块
  （`xai-grok-pager/src/scrollback/blocks/tool/memory_search.rs`，从工具输出解析
  score/source/path/snippet）。

## 9. 设计上值得注意的取舍

- **KV cache 优先**：记忆块持久化 + 原样复用，是全代码库里对 prompt cache 最敏感的一处设计。
- **每一层都有降级路径**：向量→FTS、V2/Legacy 隔离互不踩、watcher 失败→仅日志。
- **安全边界细**：凭证 fail-closed、ephemeral CWD、symlink 拒绝、全字段字节上限。
- **V2 的方向**：从"自由 Markdown + 搜索"转向"结构化 observation 流 + 生成式预算 manifest"，
  检索成本从每次查询转移到 manifest 生成，上下文体积可精确控制。
