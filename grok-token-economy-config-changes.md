# Grok（grok-build fork）token 经济性配置变更

> 文档版本: v1.0
> 日期: 2026-11-13
> 状态: Applied（本机已生效，新会话起）
> 关联: 本仓库无；明细审计与对照见 grok-build-proxy 仓库
> `docs-local/token-economy-audit-2026-11/`（README、follow-up-2026-11-13、
> cli-design-port-eval-2026-11-13）

---

## 0. 结论

- 目标是**省 token 且不动 agent 行为**：只改请求形态与工具输出上限，不改工具面、
  不降推理档位、不启用有损压缩。
- 已落三组改动：bash 输出上限 40k；四个大窗口模型的工具结果裁剪改追加式
  （`prune_mode = "append"`）；其中主模型与三个备选模型的 system 提示变更改
  追加式（`system_prompt_update = "in_history"`）。
- 稳态命中率不会因此变化（实测 98.3% 已经是上限）；收益是**避免少数场景的缓存
  断崖**：长会话越过窗口 50%、会话中途换模型/重建 agent、单次 bash 输出过大。
- 一个必须记住的坑：配置枚举值是 snake_case。11-12 写的 `"in-history"` 是非法值
  （`grok inspect` 报 unknown variant），该能力位一直没生效；今天是改成
  `"in_history"` 后才真正启用。

---

## 1. 背景：为什么只调这三处

本机 10 天账本（`~/.grok/cache/model-usage.jsonl`，1910 次主回合）：

| 指标 | 数值 |
|---|---|
| prompt tokens | 277.3M |
| 命中（cached） | 272.7M（98.3%） |
| 未命中 | 4.65M（约 2.4k/轮，主要是新增工具输出与上一轮回复） |
| 输出 | 1.36M（其中 reasoning 0.82M） |
| 估算成本 | cached $0.82 + uncached $0.70 + output $0.82 ≈ $2.33 / 10 天 |

首轮请求约 55k tokens（项目规则约 27k、工具说明约 11k、system 约 3k、skills
约 5k）。结论是：命中率没有可挖的空间，剩下的都是**新增内容**和**基座体积**；
配置层能安全动的是工具输出上限与"历史中途改写"这两类会制造断崖的开关。

---

## 2. 变更清单

### 2.1 bash 单次输出上限（2026-11-12）

```toml
[toolset.bash]
output_byte_limit = 40000
```

- 缺省 200k 字符（约 50k+ tokens）全量进历史；40k 约 10k tokens（中文约 13k）。
- 截断保留头尾，并在结果里给出全量输出文件路径，需要细节时用 `read_file`/grep
  取回，不丢信息。
- 影响的是**新增内容体积**，不影响命中率。

### 2.2 工具结果裁剪：追加式（2026-11-12，主模型）

```toml
[model."relay/deepseek-flash"]
prune_mode = "append"
```

- 默认 `rewrite`：越过窗口 50% 后，每轮请求把旧 tool result 原地改写成头尾截断
  或占位符。第一个被改写点之后的前缀整段作废，会话剩余轮次全部断崖。
- `append`：旧 tool result 原字节保留，只在其后追加一条固定失效标记；前缀保持
  只追加，缓存可续命。
- 本机 1M 窗口、10 天最大单轮 345k，**尚未触发**（512k 才越线）；价值在更长
  会话与切到小窗口模型时兜底。
- 注意：`append` 不释放 token 数，只用标记告知模型"该结果已失效"。窗口小的
  模型（如 200k）不建议开，否则上下文会一路涨到压缩阈值。

### 2.3 system 提示变更：追加式（11-12 配置，11-13 修正拼写后生效）

```toml
system_prompt_update = "in_history"
```

- 默认 `off`：system 提示一旦变化（换模型、重建 agent、运行中更新），直接改写
  `messages[0]`，整段前缀作废。
- `in_history`：把新 system 快照追加到历史尾部，端点读取最新 system 为准；
  旧前缀字节不变。
- 11-12 的配置写成 `"in-history"`，`grok inspect` 报 `unknown variant`，实际
  一直是 `off`；改成 `in_history` 后 `inspect` 零相关告警。
- 该能力位对 `messages`（Anthropic）后端会自动回退为旧行为（代码内守卫），
  不需要额外判断。

### 2.4 能力位扩展到其他大窗口模型（2026-11-13）

| 模型 | 窗口 | prune_mode | system_prompt_update | 说明 |
|---|---|---|---|---|
| `relay/deepseek-flash` | 1M | append | in_history | 主模型 |
| `relay/glm-5.3-flash` | 1M | append | in_history | 本次补 |
| `relay/gpt-6-luna` | 1.05M | append | in_history | 本次补 |
| `ocgo-deepseek-v4.1-flash` | 1M | append | in_history | 本次补（responses 后端，probe 已过） |
| `lightvela/auto` | 200k | 默认 rewrite | 默认 off | 小窗口，append 不释放 token，保持默认 |

动机：切换模型时（`/model`、子 agent 绑定）不应回到"原地改写"的旧行为。

---

## 3. 生效范围与验证

- 配置在会话启动时读取；**已在运行的会话保持旧快照**，新会话起生效。
- 静态校验：`grok2 inspect` → `Config Warnings` 应只剩既有的 `[consent]`
  unrecognized 一条。
- 首轮缓存（每条新会话的第一轮）：`~/.grok/cache/model-usage.jsonl` 里该轮
  `cachedPromptTokens` 应覆盖 system + 全部工具说明（本机约 14–16k），而不是
  停在旧前缀断点。
- 中段命中：同一会话逐轮 `cachedPromptTokens / promptTokens` 应保持在 98% 左右，
  越过窗口 50% 的会话不应出现整段跌到 0 的轮次。
- 换模型场景：切到上表任一模型后继续同一会话，观察切换后第一轮命中是否只丢
  新增块，而不是整段前缀。

---

## 4. 回滚

```bash
cp ~/.grok/config.toml.bak-token-econ-20261113-100330 ~/.grok/config.toml
```

（同日更早备份 `config.toml.bak-token-econ-20261112-124324` 对应 11-12 状态。）

最小回滚：删掉 `[model."<id>"]` 里的 `prune_mode` / `system_prompt_update` 两行，
或把 `output_byte_limit` 改回 200000。

---

## 5. 明确没做的配置

| 项 | 原因 |
|---|---|
| `[features] turn_summary = false` | 只影响 dashboard 摘要，属 UX 取舍；需要先量化每回合一次的 sidecar 成本再定 |
| `[tool_output_compression] enabled = true` | 一期是有损 head/tail 压缩，可能破坏唯一型输出；等无损层 |
| 全局降 `reasoning_effort` | reasoning 占输出 60%，但降档换任务质量，主线程保持 `high` |
| 缩小工具面 / 延迟工具声明 | 本机 12.5k tokens，省约 $0.03/10 天，发现质量风险更大 |
| 调小 prune 阈值 | 1M 窗口下本就不触发；调小等于主动制造失效标记 |

---

## 6. 一句话给同事

这三处配置的共同点是**只改请求形态、不改模型看到的信息量**：bash 上限让大输出
可回读，append/in_history 让"历史中途改写"不再一改就全断。命中率不会再涨，
但长会话和切模型时的成本抖动会明显变小。
