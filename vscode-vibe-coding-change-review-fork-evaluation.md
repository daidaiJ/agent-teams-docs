# VS Code 变更审查体验：上游机制调研、可移植结论与 fork 选型评估（vibe coding 场景）

> 文档版本: v1.0
> 日期: 2026-10-29
> 状态: Draft
> 关联: 上游仓库 thinkrail / jetbrains-version-control / vs-idea-gittool / svyatogor/crit（版本与克隆路径见 §7）；VS Code 官方文档 Review and revert agent changes、Language Model Tool API、1.99 发布说明

---

## 0. 结论

1. **审查的对象必须是"agent 的一轮改动"，不是整个工作区。** ThinkRail 用 `GitDiffScope`（`branch | uncommitted | commit(sha) | pinned(baseRef) | turn(baseTree,headTree)`）表达这件事，并在审查会话创建时把 base 侧 ref 钉成 commit oid。这是全篇最可移植、也最容易被忽略的一条。
2. **"看过"和"失效"要分开，且失效必须自动。** ThinkRail 用 `stale`（审阅后代码被覆盖）与 `unreviewedShas`（新提交使该步重新变未审）；Crit 用"标记记录被审时内容的 sha256"实现同一语义，并给出 `Show Changes Since I Reviewed` 的增量 diff。VS Code 原生（Agents 窗口）也已做到"文件再被改动时已审状态自动清除"。
3. **评论闭环在 VS Code 扩展里不成立，除非把审查状态变成 agent 能调用的接口。** ThinkRail 的评论之所以有用，是因为它自己就是 agent 宿主（pi 在进程内，评论由宿主渲染成上下文包投进会话）。VS Code 扩展不拥有 agent，可行通道只有三条：**MCP server**（跨 agent 通用）、**Language Model Tool**（仅 Copilot agent mode）、**诊断/文件导出**（零 API 依赖）。
4. **生态里已经有实现，且是 MIT。** `svyatogor/crit`（26 个源文件 / 5.6k 行，15 个测试文件）已具备：逐文件打钩 + `4/9 reviewed` 进度、审后变更失效、行内评论带 git diff hunk 上下文、批量提交给 Copilot / Claude Code、MCP server、4 个 Copilot LM tool、把评论发布为 `Crit` 来源诊断、AI 自审。
5. **fork 选型：做"审查工具"选 Crit；做"IntelliJ 风格 git 客户端顺带把审查做好"选 vs-idea-gittool。** 明确不选 `mhutchie/vscode-git-graph`（tip 停在 2021-09-19）、GitLens（`plus/` 目录专有）、`microsoft/vscode-pull-request-github`（83k 行 src + GitHub API 绑定）。

---

## 1. 问题定义

vibe coding 下"审查 AI 改的代码"有三个具体痛点，它们对应不同的机制，不能用一个功能一起解决：

| 痛点 | 表现 | 需要的机制 |
|------|------|-----------|
| **看不完** | 一轮改动几十个文件，diff 一屏放不下 | 范围收窄（按轮/按提交）+ 一次一文件模式 + 进度计数 |
| **看不出重点** | 文件平铺，不知道先看哪个、为什么改 | 阅读顺序导读（每文件一句 why）+ 大范围提示 + 隐藏空白 |
| **说了没用** | 想改就得回到对话里复述一遍；写的评论 agent 读不到 | 审查状态变成 agent 可调用的接口（MCP / LM tool / 诊断） |

第三个痛点决定了一切：**凡是"只写给人看"的审查 UI，在 vibe coding 里都会退化成自娱自乐。** 本文后续所有建议都以此为前提——审查产物必须落到 agent 能读到的地方。

---

## 2. ThinkRail 的审查设计（可移植机制）

ThinkRail 是 Electron + Bun 的 agentic IDE，审查层在 `packages/server/src/reviews/`（服务端）+ `apps/web/src/panels/`（客户端），契约在 `packages/contracts/src/domain.ts`。它的设计密度明显高于另外两个仓库，以下五条是可移植的。

### 2.1 锚点模型与重锚定

`ReviewAnchor`（`packages/contracts/src/domain.ts:1330-1337`）= `path` + `side`（`base` / `worktree`）+ `contentHash`（文件字节的 sha-256）+ 有序 `selectors` 回退链：

| selector | 用途 |
|----------|------|
| `lineRange` | 主锚点，行号区间 |
| `textQuote`（`exact` + 前后各 32 字符 `prefix`/`suffix`） | 重锚定的证据 |
| `diffHunk` | 记录当时的 hunk header |
| `structural`（scheme + ref） | 文档节点（表格单元格、notebook cell） |
| `region`（归一化 0..1 几何 + page） | 图像/PDF 区域 |

重锚定是纯函数（`packages/server/src/reviews/anchoring.ts:186-224`）：

1. `contentHash` 未变 → `anchored`；
2. 否则按 `textQuote.exact` 搜索，用 prefix/suffix 消歧，唯一命中 → 更新 `lineRange`，状态 `moved`；
3. 无命中 / 有歧义 / 文件消失 → `outdated`；
4. 没有可搜索文本的**定位型**锚点（`region`/`structural`）一律 `outdated`，而**整文件级**评论保持 `moved`。

两个容易抄错的细节：

- **`moved` 是粘性的**：重钉时刷新 `contentHash`，否则下一轮会因 hash 相同被降级回 `anchored`，丢掉"自创建以来已漂移"这一事实。
- **状态与生命周期正交**：`anchorState`（`anchored`/`moved`/`outdated`）与 `status`（`draft`/`sent`/`resolved`/`dismissed`）互不覆盖——"是否讨论过"和"锚点还活着吗"是两件事。

### 2.2 审阅状态与失效

- **审查会话**：每个工作区一个 `Review`，`baseSha` 在创建时钉成 `merge-base(diff 目标, HEAD)` 的完整 oid。SPEC 里写明理由：`originalRef` 对 `uncommitted` 是符号 `HEAD`，原样存储意味着"用户下一次提交就把行号指向别的内容"。
- **sidecar 存储**：plan 步骤的审查状态 `TodoReviewInfo { state, reviewedBy: user|agent, revision, unreviewedShas }`（`domain.ts:349-368`）明确**存在 sidecar，从不写进 agent 可写的 plan 文件**——"so an agent re-plan can't flip a review decision"。
- **`stale`**：`domain.ts:1352` 注释直说 "Server-derived for the client, never persisted: the reviewed code was overwritten after review"。
- **完整性规则**（`domain.ts:1491-1510`）：`request_changes` 必须至少带一个可执行 finding（否则 worker 会被晾住）；带着未解决 finding 的 `approve` 会被拒绝并记 `blockedByOpenFindings`。**AI 自审必须有互相约束，否则退化成盖章。**
- **未归属改动单列**：`unattributed` / `adoptedCommits`（`domain.ts:390-398`）把"不属于任何 plan 步骤的改动"显式列出，注释称之为 "the honesty section of the review map"。对应现实里"agent 在计划外顺手改的文件"。

### 2.3 范围模型

```ts
export type GitDiffScope =
  | { kind: "branch" } | { kind: "uncommitted" }
  | { kind: "commit"; sha: string } | { kind: "pinned"; baseRef: string }
  | { kind: "turn"; id: string; baseTree: string; headTree: string; startedAt: number };
```
（`packages/contracts/src/domain.ts:689-694`）

`turn` 是 agent 的一轮改动（前后两棵树），`commit` 是单次提交，`pinned` 是固定的 baseRef。审查面板据此只渲染该范围的 diff，并显示"该提交已不在本分支"/"该轮快照已不在仓库"这类失效提示。

### 2.4 结构化上下文包

`packages/server/src/reviews/packageRender.ts` 把评论渲染成一段结构化文本（这是"发给 agent"的载体，不是给人看的评论 UI）：

```xml
<review id branch base comments>
  <comment id kind path side base-ref lines anchor anchor-kind>
    <locator>image region x=… y=… w=… h=… of <path> (sha256 …)</locator>
    <fragment>…创建时的原文…</fragment>
    <context lines="…" side="base">…±10 行…</context>
    <text>…用户写的正文…</text>
  </comment>
  <instructions>…</instructions>
</review>
```

要点：`CONTEXT_LINES = 10`；每条带 `anchor`（状态）与 `anchor-kind`（`line`/`file`/`region`/`structural`），让 agent 知道该按行还是按几何理解；**所有动态属性做实体转义**（`& < > "`、CR/LF 转数字引用），正文与片段保持原样——因为仓库路径是外部可控文本，否则可以闭合标签伪造一条用户没写过的评论。

### 2.5 审阅引导与 AI 自审

- **Review guide**：`ReviewGuide { summary, readingOrder[], verdict, todoId, reviewedSha }`（`domain.ts:1366-1378`），`readingOrder` 每项是 `{ path, why }`，上限 12 步（`REVIEW_READING_ORDER_LIMIT`）；客户端按步走读，支持 n/p 键（`apps/web/src/panels/ChangesReviewGuide.tsx`）。
- **审阅进度 UX**：`packages/locales/en.json:49-81` 给出完整词汇——`Mark viewed` / `Mark all viewed` / `{{viewed}} of {{total}} reviewed` / `Next unreviewed` / `Stacked` 与 `One file` / `Hide whitespace changes` / `{{count}} files in this scope — reviewing one file at a time may be easier` / `Send review`。

### 2.6 明确不要移植的部分

| 机制 | 为什么不要 |
|------|-----------|
| 评论完整生命周期（draft/sent/resolved、`sent` 即不可删、`rollbackSend`） | 依赖"宿主拥有 agent 会话"与 id 语义；扩展里没有消费者 |
| per-file 会话钉住（`Review.fileSessions`）、`resolve_comment` 工具 | 同上，属于 agent 宿主能力 |
| worktree 隔离、per-workspace 锁、`review.changed` 推送收敛 | 为多客户端 + 多会话设计，单进程扩展是纯负担 |
| base/worktree 双锚点空间、`region`/`structural` 锚点 | 为多模态资源（图像、notebook）设计，与 git 审查无关 |

---

## 3. 另两仓可借鉴的局部设计

### 3.1 jetbrains-version-control（JGC，MIT）

| 设计 | 出处 | 价值 |
|------|------|------|
| 评论控制器当**浮层卡片**（回复框复用为提交信息输入，`comments/commentThread/context` 菜单项接提交动作），文档变更或光标移出即关闭 | `src/decorations/markerPopup.ts:23-100` | 想在原生编辑器里放审查入口，这是唯一可行姿势 |
| 光标行 blame 只更新一个 decoration，blame 按 document version 缓存 | `src/blame/caretBlame.ts:137-214` | 审查视图里的行级元信息可复用此低成本路径 |
| 图布局是纯算法 + `LaneSnapshot` 跨页稳定；webview 只做滚动窗口虚拟化（overscan 8） | `src/git/graphLayout.ts:12`、`webview/src/panel/components/GitGraphSvg.tsx:12,414-438` | 审查清单若要显示提交列可直接复用 |
| git 调用统一 `execGit` + 5s TTL 缓存 | `src/git/gitService.ts:66-85`、`src/git/cache.ts:1-32` | 审查需要频繁读 blob/文件字节，缓存是前提 |
| AI 提交信息的**上下文选择规则**（未跟踪→文件内容；已暂存→`git diff --cached`；未暂存→`git diff`；风格参考取作者近 10 条提交） | `src/ai/commitMessage.ts`、`src/ai/prompt.ts:35-83` | 若"AI 导读"走 `vscode.lm`，这套裁剪规则可直接搬 |

JGC 的短板同样明确：diff 走 VS Code 内置编辑器（`src/views/diffEditorManager.ts`），**没有本地快照层**，因此"这一轮改了什么"没有现成边界；测试只有 `vscode-test`。

### 3.2 vs-idea-gittool（IdeaGitTool）

它的价值不在审查（没有任何评论/审阅代码），而在**宿主机制齐备**：

| 机制 | 出处 | 对审查的意义 |
|------|------|-------------|
| 自绘 diff：side-by-side / unified，**双侧行号** + block id + 展开空隙 | `src/webview/log/diffView.tsx:25-45`、`src/webview/log/diffAlign.ts` | 内置 diff 无法注入 UI，审查界面只能落在这里，而这里现成 |
| Local History：按文件记录 before/after，带 `group` 与 `external: true`（git 操作）标记，`putLabel` 可打"Before Update" | `src/features/localHistory/recorder.ts:142-280`、`src/features/localHistory/store.ts:13` | **"agent 一轮"的现成边界**，不需要另建快照层、不写 git 对象 |
| 合并窗口用 CodeMirror 6（行 gutter / decorations / 折叠） | `src/webview/merge/*`、`package.json:1794-1798` | 行级 UI 的第二套参考实现 |
| webview 是 Preact + 自研 RPC（`{type:'req'|'res'|'evt'}`） | `src/webview/lib/rpc.ts`、`src/host/webviewHost.ts` | 新增审查面板的接入成本已知 |
| 变更列表（changelist）概念：命名分组 + 活动组 + 勾选集 | `src/features/commit/changelists.ts:15,29-47` | 可用来表达"这一轮的改动" |
| 验证台：`node:test` + vscode stub、`scripts/preview.mjs`（Chromium 渲染出图/矩形）、`scripts/layout-check.mjs`（元素盒子回归）、`scripts/e2e.mjs`（真实 VS Code） | `test/` 与 `scripts/`（6.7k 行） | 新增 UI 的验证路径明确；改布局会让 layout-check 报错，这是设计意图 |

---

## 4. 生态现状：已有实现与原生能力

### 4.1 第三方扩展（均已确认存在）

| 扩展 | 许可证 | 与目标的关系 |
|------|--------|-------------|
| **svyatogor/crit**（"Crit"） | MIT（© 2026 Jinqi Shen） | 最贴：逐文件打钩 + 进度 + stale + 增量 diff + 行内评论 + MCP + LM tool + 诊断通道 + AI 自审 |
| `JinqiShen.diff-review`（"Diff Review with AI Agent"） | 未核实源码仓库 | 与 Crit 描述几乎相同，疑同一作者的另一条产品线 |
| `wushuzhilu.change-review`（"Change Review"） | 未核实 | 轻量：git/svn/无版本管理，逐文件打钩 + 按块 diff 审查 |
| `changAIs` | 商业/闭源 | 展示 AI 会话改动的文件 + AI 注释贴到对应行 |
| `ethanito.document-oriented-vibing`（DOV） | 未核实 | 按 hunk 接受/拒绝，`.reviews/*.diff` + `.state.json`，与 Codex 线程绑定 |
| `tomasz-tomczyk/crit`（同名但不同产品） | MIT | Go 单二进制 + 浏览器审查层（plan.md / diff / 运行中的 app），集成 18 种 agent。**不是 VS Code 扩展**，只能当产品形态参考 |

### 4.2 VS Code 原生已覆盖的部分

来自官方文档 "Review and revert agent changes"：

- **working set**：Copilot / Local harness 的编辑记为待审改动，逐处 **Keep / Undo**，支持文件级与全部；`chat.editor.workingSet.enabled`。不需要 git，Keep 不等于 stage/commit。
- **Agents 窗口的 Changes 视图**：有 **Branch Changes** 变更集下拉；每文件 **Mark as Reviewed**；**文件再被改动时已审状态自动清除**。
- **已知边界**：working set 只跟踪"受支持的编辑工具"，通过终端命令 / 脚本改的文件不在其中；不覆盖非 git 的变更集语义。

**推论**：扩展的生存空间只有三个角落——git 语义的变更集（未提交 / 分支 vs base / 单提交 / agent 一轮）、VS Code harness 之外的 agent（Claude Code、Codex 等在终端里改文件）、AI 导读（阅读顺序）。重做原生已有的东西没有意义。

### 4.3 三条"agent 读得到"的通道

| 通道 | 适用 agent | 关键约束 | 实例 |
|------|-----------|---------|------|
| **MCP server** | 所有支持 MCP 的客户端（Copilot agent mode、Claude Code、Cline、Codex、Cursor、Windsurf…） | 通用性最好；需要进程间寻址（Crit 用每窗口一个本地 HTTP server + `${TMPDIR}/crit/instances/<sessionRef>.json` 描述符） | Crit 的 `listDiffComments` / `addDiffComment` / `replyToDiffComment` / `resolveDiffComment` / `deleteDiffComment` |
| **Language Model Tool** | 仅 Copilot agent mode | `contributes.languageModelTools` + `vscode.lm.registerTool`；需设 `toolReferenceName` 与 `canBeReferencedInPrompt` 才在 agent mode 自动可用；**ask / edit 模式不自动调用**，用户要 `#` 手动引用 | Crit 的 4 个 `crit_*` 工具 |
| **诊断 / 文件导出** | 任意能读诊断或文件的 agent | 零 API 依赖；诊断需要 agent 侧主动读 | Crit 把每条未解决线程发布为 `source: Crit` 的 Information 诊断 |

补充事实：VS Code 1.99 发布说明写明"扩展贡献语言模型工具的 API 已定稿，现在可在 agent mode 中使用"；本仓 IdeaGitTool 的 `@types/vscode` 是 `^1.90.0`（`package.json:1801`），**不含该 API，使用前要抬版本并在运行时 feature-detect**。

---

## 5. fork 选型评估

### 5.1 候选与实测数据

所有仓库均为 depth-1 克隆后本地统计（2026-10-29）。

| 候选 | 许可证 | 栈 / 体积 | 审查能力现状 | 测试 | tip 提交 |
|------|--------|-----------|-------------|------|---------|
| **svyatogor/crit** | MIT | TS + esbuild，**纯原生 API（无 webview）**，src 26 文件 / 5.6k 行，`engines: vscode ^1.110`、node ≥22.18 | 打钩+进度、stale、增量 diff、行内评论+diff hunk、批量提交、MCP、4 LM tool、诊断通道、AI 自审 | 15 文件 / 3.0k 行（`node --test` + vscode stub） | 2026-08-19（v0.5.1） |
| PGshen/vs-idea-gittool | package.json 与 README 写 Apache-2.0，**LICENSE 文件是 JetBrains Open-Source Build Terms v1.3（矛盾）** | Preact + esbuild，src 25.4k 行（webview 10.6k），`engines: vscode ^1.90` | 无审查层；但有自绘 diff + Local History + CodeMirror 合并窗 + l10n | test+scripts 6.7k 行，含 preview / layout-check / e2e | 2026-10-28（0.4.2） |
| witt-bit/jetbrains-version-control | MIT | React 19 + Vite + zustand，9.9MB | 无审查层；diff 走内置编辑器；有 `vscode.lm` 提交信息 | 仅 `vscode-test` | 2026-10-25（1.1.3） |
| aotemj/jetbrains-git-graph（JGC 上游） | MIT | React 19 + Vite，21.7k 行，`engines ^1.85` | 同上但更小更干净 | 仅 `vscode-test` | 2026-06-23（0.4.17） |
| zhyc9de/jet-git | MIT | React 19 + Vite，7.4k 行 | 更早的上游，功能少 | 仅 `vscode-test` | 2026-03-02（0.0.4） |
| mhutchie/vscode-git-graph | MIT | 无框架 webview + jest，18.3k 行，`engines ^1.38` | 图 + 提交详情，diff 走原生 | jest（完整） | **2021-09-19** |
| microsoft/vscode-pull-request-github | MIT | React 16，src 83k 行 / 113MB | PR 审查全套（评论线程 webview、viewed 状态） | 完整 | 2026-10-29 |
| GitLens | MIT，但 `plus/` 目录另许（`LICENSE.plus`） | 超大 | 注解、图、审查辅助 | 完整 | 活跃 |

### 5.2 结论与理由

**（A）目标是"做出这个审查体验" → fork `svyatogor/crit`。**
理由：它是唯一已经实现了 §0 第 2、3 条全部内容的开源实现，且 MIT。要改的只有三块：

1. **git 侧范围**：它只覆盖"与 HEAD 的差异"（Changes 视图即工作区 vs HEAD），没有分支 vs base、单提交、staged 的语义，也没有"agent 一轮"的边界；
2. **AI 导读**：它做的是"AI 审我的代码"（`Crit: Ask AI to Review Changes`），不做"AI 帮我排阅读顺序"；
3. **diff 内呈现**：它全靠原生 diff，无法在 diff 里做行级进度/折叠——若要行内审查面板，可借 IdeaGitTool 的自绘 diff。

**（B）目标是"IntelliJ 风格 git 客户端顺带把审查做好" → 基座用 `vs-idea-gittool`，Crit 当参考实现抄。**
理由：自绘 diff 已就绪（`src/webview/log/diffView.tsx`）、"一轮"边界现成（Local History 的 `group` + `external`）、验证台最完整（preview / layout-check / e2e）、有 l10n。代价是审查层要从零写，且有两个前置条件：

- **先修 LICENSE 矛盾**：`package.json:7` 与 README 声明 Apache-2.0，`LICENSE` 文件却是 JetBrains Open-Source Build Terms（JetBrains IDE 的条款，明显误拷）。fork / 分发前必须澄清归属。
- **补 agent 通道**：当前 `src/` 内 `createCommentController` / `CommentThread` / `vscode.comments` 零命中，也无任何 `vscode.lm` 调用；`contributes` 里没有 `languageModelTools`。

**（C）不要选**：
- `mhutchie/vscode-git-graph`：tip 停在 2021-09-19，`engines ^1.38` 基线太老，无框架 webview 改造面大。
- GitLens：`plus/` 目录专有，且与 GitKraken 服务耦合。
- `microsoft/vscode-pull-request-github`：83k 行 src + GitHub API 绑定，形态不符；但它的评论线程 webview 与 viewed 状态是很好的 UI 参考。

### 5.3 IdeaGitTool 的硬约束（已核实）

| 约束 | 证据 | 影响 |
|------|------|------|
| 内置 diff 编辑器不可注入 UI | 扩展只调 `vscode.diff` / `vscode.changes`：`src/host/gitContent.ts:109,128`、`src/features/commit/view.ts:669-683`、`src/features/commit/mergeTool.ts:117` | 审查界面只能落在自绘 webview 或独立面板 |
| 无评论 API 使用 | `grep -rn "createCommentController\|CommentThread\|vscode.comments" src` 零命中；唯一 CodeLens provider 是冲突标记 `src/features/commit/conflictEditor.ts:171` | 行内评论要么走 Comment API（新引入），要么自绘 |
| 无 LM / agent 集成 | `grep -rn "vscode.lm\|LanguageModel" src` 零命中 | 导读与通道都要新建 |
| `@types/vscode ^1.90.0` | `package.json:1801`，`engines: vscode ^1.90.0`（`package.json:18`） | 用 LM tool 需抬版本 + 运行时 feature-detect |
| 改布局会触发回归检查 | `scripts/layout-check.mjs` 比对元素盒子（0.5px 容差） | 新增面板需同步更新基线，属设计意图 |

---

## 6. 规划（依据 → 动作 → 成本 → 验证）

以下按"先证明价值，再补能力"排序。P1–P2 不依赖任何 agent 集成，可独立验证。

### P1 · "本轮改动"视图（约 2–3 天）

- **依据**：§2.3 范围模型 + §3.2 Local History 的 `group`/`external` 标记。Local History 已在保存/创建/删除/重命名/git 操作时按文件记录 before/after（`recorder.ts:142-280`），把一段连续的外部变更组当"一轮"即可，不写 git 对象、不动索引。
- **动作**：新增命令 `Review this turn`，打开只含本轮改动的视图（文件列表 + 每文件 diff stat + 状态字母）。
- **验证**：`scripts/preview.mjs` 出图确认渲染；`scripts/e2e.mjs` 跑一条"agent 改文件 → 打开本轮视图"的场景。

### P2 · 进度与自动失效（约 1–2 天）

- **依据**：§2.2 的 `stale` / `unreviewedShas`；Crit 的 sha256 实现；原生 Agents 窗口也这么做。
- **动作**：viewed 标记存 `workspaceState`（键 = path + blob sha256）；显示 `N of M reviewed` 与 `Next unreviewed`；隐藏空白开关；大范围时提示"一次一文件"；**文件在本轮内再次被改动时自动退回未审**。
- **验证**：单测覆盖"标记 → 改文件 → 标记失效"；文案可直接借 `packages/locales/en.json:49-81` 的词汇。

### P3 · AI 导读（约 1–2 天）

- **依据**：§2.5 的 `ReviewGuide.readingOrder`（每文件一句 why，上限 12）+ JGC 的 `vscode.lm` 用法与裁剪规则（`src/ai/commitMessage.ts`、`src/ai/prompt.ts:35-83`）。
- **动作**：对本轮 diff 生成"建议阅读顺序 + 摘要"，贴在视图顶部，支持 n/p 步进。
- **验证**：无 LM 可用时（无 Copilot 订阅）退化为启发式排序（改动行数 / 文件类型），不得阻塞视图。

### P4 · 闭环通道（1–3 天，按所用 agent 选一条）

- **依据**：§4.3 三条通道 + §0 第 3 条。
- **动作**：任选其一 —— MCP server 暴露 `get_changeset(scope)` / `get_review_state()` / `list_unreviewed()`；或 LM tool（需抬 `engines`）；或导出 `.ideagittool/review.md` + 剪贴板。
- **验证**：用一个真实 agent 客户端（Claude Code / Copilot agent mode）跑通"agent 查询未审文件 → 按结果继续改"的往返。

### P5 · 行级标记（可选，仅当需要"在这一行觉得不对"时）

- **依据**：§2.1 锚点模型。**只做锚点 + 状态，不做评论生命周期**；让标记本身成为 agent 能查的状态（即 P4 的读通道），而不是投进虚空里的评论。

---

## 7. 复现信息与未验证项

### 7.1 克隆与版本

GitHub 直连在本机不通（`github.com:443` 超时），全部经 `ghfast.top` 镜像 depth-1 克隆：

| 仓库 | revision | 本地路径 |
|------|----------|---------|
| JetBrains/thinkrail | `940d15c`（2026-10-28） | `%TEMP%\review-research\thinkrail` |
| PGshen/vs-idea-gittool | `8bd81d2`（2026-10-28） | `D:\pro\git-vscode\repo` |
| witt-bit/jetbrains-version-control | `d27522c`（2026-10-25） | `%TEMP%\review-research\jvc` |
| svyatogor/crit | v0.5.1（2026-08-19） | `%TEMP%\review-research\cand-critvscode` |
| 其他候选（mhutchie / aotemj / zhyc9de / microsoft / tomasz） | 见 §5.1 tip 提交列 | `%TEMP%\review-research\cand-*` |

### 7.2 已核实 vs 未验证

- **已核实（读代码/读官方文档）**：ThinkRail 的契约与 `anchoring.ts` / `packageRender.ts` 实现；Crit 的许可证、`contributes`（37 命令 / 2 视图 / 4 LM tool）、`vscode.comments.createCommentController` 调用、MCP SDK 依赖、测试文件清单；IdeaGitTool 的 diff 打开点、Local History 记录点、零评论 API / 零 LM 调用；VS Code 官方对 working set 与 Agents 窗口 Mark as Reviewed 的描述。
- **未验证**：Crit 的**运行时行为**（"打钩 → agent 改文件 → 变 stale → 增量 diff"链路未实跑）；`JinqiShen.diff-review`、`wushuzhilu.change-review`、DOV 的源码仓库与许可证；GitLens `plus/` 的边界细节；IdeaGitTool LICENSE 矛盾的成因（需作者确认）。
- **选型建议的前置动作**：装一次 Crit 跑通上述链路（约 10 分钟），同时确认 vs-idea-gittool 的 LICENSE 归属。这两件事定了，选型就只剩一句话。
