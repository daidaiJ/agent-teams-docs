# Team Controller 调谐耗时定位与观测排查方案

> 文档版本: v1.0
> 日期: 2026-08-05
> 基线代码: `AgentTeams` HEAD（`48ce4aa` / `c01aaec` 修复后，含 per-step timing logs 与可配置并发）
> 关联文档: [team-controller-performance.md](team-controller-performance.md)（修复前分析）、[team-controller-defects.md](team-controller-defects.md)（缺陷清单）

---

## 1. Step 4 耗时 3m10s 定位

### 1.1 Step 4 代码路径

`team_controller.go:440` 的 Step 4 对每个 member（leader 优先，**串行**）执行 `reconcileMember`，其 5 个子阶段：

| 子阶段 | 调用链 | 特征 |
|--------|--------|------|
| infra | `ReconcileMemberInfra` → `ProvisionWorker` / `RefreshWorkerCredentials` | 首次含固定 2s sleep（`provisioner.go:455`，WASM key-auth 同步）；稳态走缓存 token，无网络调用 |
| service-account | `EnsureMemberServiceAccount` | 秒级 |
| config | `ReconcileMemberConfig` → `DeployWorkerConfig`（OSS 推送） | **每次调谐全量执行**，10~20 次 mc 子进程 |
| container | `createMemberContainer` → `wb.Create` | k8s 异步提交 Pod；docker 后端含 `ensureImage` 同步拉镜像 |
| expose | `ReconcileExpose` | 稳态 no-op（只处理新增/删除端口） |

### 1.2 已验证的排除项

| 候选 | 结论 | 依据 |
|------|------|------|
| Higress `AuthorizeAIRoutes` 遍历路由 | 排除，O(1) | AI 路由固定 1 条（`default-ai-route`，initializer 创建），`modifyAIRoutes` 对其 GET+PUT；409 重试 ≤5×3s 且仅冲突时 |
| Matrix orphan 登录重试 | 排除，≤~7.5s | `client.go:222`，5 次 × 0.5~2.5s，仅登录失败时触发 |
| Higress HTTP 超时 | 排除，30s 上限 | `higress.go:30` |
| K8s backend `Status` | 排除，O(1) | 单 Pod Get（`kubernetes.go:345`） |
| Docker backend `Status` | 排除，O(1) | 单容器 inspect |
| `seedLocalAgentFiles` | 排除，O(本地文件数) | 只遍历成员自己的目录，每文件 1 次 GetObject 探活 |
| `pushBuiltinTopLevelFiles` | 排除，seed-only | 已存在即跳过（`deployer.go:859`） |
| `ReconcileExpose` | 排除，稳态 no-op | `provisioner_expose.go:25` |
| 凭据存储 | 排除，O(1) | `SecretCredentialStore` 按 worker 独立 Secret |
| 控制器内全局锁 | 排除 | controller 包无 mutex；`LegacyCompat.mu` 只串行 registry 文件操作（每 op 秒级） |

### 1.3 最可能路径：mc CLI 子进程风暴（OSS 层）

**核心事实**：`oss/minio.go` 的每个 `GetObject` / `PutObject` / `PutFile` 都是 `exec.CommandContext` fork 一个 `mc` 子进程（无连接池、无超时、无日志）。

**每成员每次调谐的 mc 调用次数**（`DeployWorkerConfig` 每次调谐全量执行，非仅首次）：

| 操作 | mc 调用数 |
|------|-----------|
| `seedLocalAgentFiles`：每个本地 seed 文件 1 次 `mc cat` 探活 | F（本地文件数，约 5~15） |
| `openclaw.json`：IsUpdate 时 GetObject + PutObject | 2 |
| `SOUL.md`：GetObject 探活（+首次 PutObject） | 1~2 |
| `prepareAndPushAgentsMD`：GetObject + PutObject | 2 |
| `pushBuiltinTopLevelFiles`：HEARTBEAT.md GetObject | 1 |
| mcporter.json PutObject | 1 |
| `PushOnDemandSkills`（按需） | 0~5 |
| **合计** | **约 10~20 次/成员/次调谐** |

单次 mc 调用 = 进程 exec（20~50ms）+ mc 启动/配置加载 + HTTP 往返。本地 MinIO 约 200~400ms/次，云 OSS 约 400~800ms+/次。

**3m10s 的构成推演**：OSS 端变慢到 1~1.5s/次时，10 成员团队 = 10 × 15 × 1.25s ≈ 190s，与观察值吻合。

### 1.4 验证方法（零改动，用现有日志）

1. 同一轮 step 4 内对比各成员 `member reconcile: infra / service-account / config / container / expose` 的 `elapsed`——`config` 占大头即实锤；
2. c01aaec 加的 `deploy worker config: seed agent files / openclaw.json / ...` 细粒度日志定位具体 OSS 操作，并数出每成员 mc 调用次数；
3. 实验：`time mc cat hiclaw/agents/<worker>/openclaw.json` 连跑 10 次测单次延迟；`mc ls --recursive` 统计对象总数与 team 数关联。

### 1.5 补充：docker 后端的分钟级阻塞点

嵌入式（docker）部署下另有**无超时**的同步镜像拉取：`backend/docker.go:365` `ensureImage`，`io.Copy(io.Discard, pullResp.Body)` 读完整个拉取流，仅受 reconcile ctx 约束（`ReconcileTimeout` 默认关闭）。属首次一次性，但 docker daemon 对 pull 有全局锁，多 team 首次建队会排队。确认方式：日志搜 `[Docker] Image not found locally, pulling:`。

---

## 2. Team 越多、单次调谐越慢的机制

> 澄清：指**单次调谐的耗时本身**随全局 team 数增长（非排队导致的墙钟时间）。

### 2.1 代码层面已排除 O(N) 循环

逐个排查 step 4 单成员路径，所有操作都是 O(1) 或 O(该成员自身资源数)，**不存在遍历全局 team/worker 的循环**（Higress AI 路由固定 1 条、K8s/Docker Status 单资源查询、OSS 操作只碰自己的 prefix）。

### 2.2 真正机制：存储规模正反馈

```
team 数 ↑ → MinIO 对象总数线性增长（每成员几十个小对象）
          → MinIO 小对象元数据/IO 性能下降；云 OSS 受 QPS/限流影响
          → 单次 mc 调用延迟上升
          → 每个 team 的 step 4（成员数 × 每成员 10~20 次 mc）整体变慢
```

### 2.3 叠加的全局串行点

| 串行点 | 位置 | 影响 |
|--------|------|------|
| `LegacyCompat.mu` 全局互斥 | `legacy.go:279` | 所有 Worker/Team/Manager reconciler 的 workers-registry 读改写（OSS 全量 GET+PUT）互相排队 |
| `HICLAW_TEAM_MAX_CONCURRENT_RECONCILES` 默认 1 | `config.go:338` | 所有 team 调谐串行，单个慢调谐阻塞全部 |
| Higress console 409 重试 | `higress.go:203` | 并发调谐写路由冲突时 sleep 1~3s ×5 叠加 |

---

## 3. 方案：带 team id 的耗时监测日志

### 3.1 设计原则

1. **team id 自动传播**：`Reconcile` 入口构造 team-scoped logger 注入 ctx，下游所有 `log.FromContext(ctx)` 自动带 `team=<name>`，无需给函数加参数；
2. **统一 timed helper，失败也打日志**（现有日志最大缺口：`reconcileMember` 各阶段失败直接 return，无 elapsed）；
3. **异常全覆盖**：错误提前返回、ctx 超时、panic、卡死调用。

### 3.2 代码实现

```go
// team_controller.go Reconcile 入口（现有 logger 之后）
ctx = log.IntoContext(ctx, logger.WithValues(
    "team", req.NamespacedName.Name,
    "namespace", req.NamespacedName.Namespace,
))

// internal/controller/timing.go
func timed(ctx context.Context, phase string, fn func() error) error {
    logger := log.FromContext(ctx)
    start := time.Now()
    err := fn()
    elapsed := time.Since(start).Truncate(time.Millisecond)
    if ctx.Err() != nil {
        logger.Error(ctx.Err(), "timed-call cancelled", "phase", phase,
            "elapsed", elapsed.String())
    } else if err != nil {
        logger.Error(err, "timed-call failed", "phase", phase,
            "elapsed", elapsed.String())
    } else {
        logger.Info("timed-call", "phase", phase, "elapsed", elapsed.String())
    }
    return err
}

// panic 兜底（reconcileMember 与 Reconcile 顶层）
defer func() {
    if p := recover(); p != nil {
        logger.Error(nil, "reconcile panic", "phase", currentPhase, "panic", p)
    }
}()
```

### 3.3 监测点清单（按优先顺序）

| # | 位置 | 加什么 | 价值 |
|---|------|--------|------|
| 1 | `oss/minio.go` `runMC` | 每次 mc 调用打 `mc <args> elapsed`（建议 >300ms 阈值） | **最关键**：直接量化 config 阶段每次 OSS 操作耗时 |
| 2 | `reconcileMember`（team_controller.go:585） | 5 个阶段全部改 timed 包装（当前失败路径无 elapsed） | 失败也能定位卡点 |
| 3 | `reconcileLegacyMember`（step 4 内） | 前后 elapsed | registry 写耗时（受 `LegacyCompat.mu` 排队影响） |
| 4 | `ProvisionWorker` 内部 | Matrix 注册/建房/join、MinIO、`EnsureConsumer`、`AuthorizeAIRoutes`、2s sleep 各段 elapsed | 首次 provision infra 拆解 |
| 5 | `DeployWorkerConfig` 内部 | 补齐 SOUL.md / `prepareAndPushAgentsMD` / `pushBuiltinTopLevelFiles` / mcporter 各段 | config 阶段完整画像 |
| 6 | `backend/docker.go` `ensureImage` | 拉取前打 `pulling image=<image>`，结束后打 elapsed | 卡死/慢拉取可见 |
| 7 | backend `Create`/`Delete`/`Status`（docker+k8s） | elapsed | container 阶段 |
| 8 | `modifyAIRoutes`（higress.go） | 路由数 + 整体 elapsed + 409 重试次数 | Higress 侧延迟与冲突 |
| 9 | `ReconcileExpose` | 前后 elapsed | expose 阶段 |
| 10 | Step 3 `ReconcileMemberDelete` | elapsed | 删除路径 |
| 11 | `summarizeBackendReadiness`（step 6） | 每成员 Status elapsed | step 6 拆解 |
| 12 | `Status().Patch` | 前后 elapsed | status 写入延迟 |

### 3.4 检索命令

```bash
# 定位某 team 的调谐热点（所有日志统一带 team=<name>）
kubectl logs -n <ns> deployment/hiclaw-controller --tail 1000 | grep "team=<team-name>"

# 只看耗时记录
kubectl logs -n <ns> deployment/hiclaw-controller --tail 1000 \
  | grep "team=<team-name>" | grep -E "timed-call|member reconcile|deploy worker config"
```

**噪音控制**：`runMC` 用阈值（>300ms）过滤；其余每阶段必打。单成员一次调谐约新增 15~25 条，`--tail 1000` 足够覆盖完整调谐。

---

## 4. 方案：CRD 状态字段写入问题排查

### 4.1 关键发现：yaml 与 Go 类型不同步

- Go `TeamStatus`（`types.go:377`）含 `reconcileAttempt`（每次调谐 +1，`team_controller.go:287`）和 `phaseTransitionTime`（phase 变化时写）；
- **`helm/hiclaw/crds/teams.hiclaw.io.yaml` 的 status 里没有这两个字段**；
- 后果：集群 CRD 若来自该 yaml，controller 写入时这两个字段被 apiserver **结构 schema pruning 静默裁剪**——写了也看不到。

### 4.2 排查步骤（前 3 步定位 90% 案例）

**Step 1：确认集群 CRD schema 实际字段**（注意：`helm upgrade` **不会**更新已安装的 CRD，`crds/` 目录只在首次 `helm install` 应用）：

```bash
kubectl get crd teams.hiclaw.io \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.status.properties}' | jq 'keys'
```

**Step 2：写探针实验——区分 CRD 问题 vs controller 问题**：

```bash
kubectl patch team <name> --subresource=status --type=merge -p '{"status":{"reconcileAttempt":999}}'
kubectl get team <name> -o jsonpath='{.status.reconcileAttempt}'
```

- 返回空 → **CRD schema 缺字段**（pruning）：先把字段补进 `helm/hiclaw/crds/teams.hiclaw.io.yaml`，再 `kubectl apply -f helm/hiclaw/crds/teams.hiclaw.io.yaml`；
- 返回 999 → schema 正常，问题在 controller 侧，继续 Step 3。

**Step 3：核对 controller 镜像版本**：

```bash
kubectl get deploy -n <ns> hiclaw-controller -o jsonpath='{.spec.template.spec.containers[0].image}'
# 对比写入逻辑（ReconcileAttempt++ / PhaseTransitionTime）所在 commit 是否早于镜像构建时间
```

**Step 4：确认调谐发生 + 无 patch 报错**：

```bash
kubectl logs -n <ns> <controller-pod> | grep "team=<team-name>" | grep -E "pass complete|failed to patch team status"
```

**Step 5：字段写入时机（部分字段看不到属正常）**：

| 字段 | 写入时机 |
|------|----------|
| `reconcileAttempt` | 每次调谐 +1，应总能看到 |
| `phaseTransitionTime` | **仅 phase 变化时**更新，稳态 Active 不刷新 |
| `observedGeneration` | 仅成功 pass 写入 |
| `maxRetriesReached` | 仅触发重试上限时置 true，默认不出现 |

**Step 6：观察方式**：status 是子资源，用 `kubectl get team -o yaml` 或 `-o jsonpath='{.status}'`；`kubectl get team` 表格只显示 `additionalPrinterColumns` 配置的列，不能用于判断字段是否存在。

---

## 5. 方案：控制器 pprof 采样（构建时开关）

### 5.1 设计目标

- **构建时显式控制**：Dockerfile 通过 `--build-arg ENABLE_PPROF=true` 决定是否编译 pprof 代码；
- **默认不开启**：默认镜像不含 pprof 逻辑，零额外端口、零安全面；
- 采样方式：`kubectl port-forward` + `go tool pprof`。

### 5.2 Go 代码（build tag 条件编译）

```go
// cmd/controller/pprof.go  (//go:build pprof)
package main

import (
    "context"
    "fmt"
    "net/http"
    _ "net/http/pprof"
    "os"
    "runtime"
    "time"
)

// maybeStartPprof 仅在 -tags pprof 构建时生效；监听地址可用
// HICLAW_PPROF_ADDR 覆盖（默认 0.0.0.0:6060，kubectl port-forward 需要 pod IP 可达）。
func maybeStartPprof(ctx context.Context) {
    // net/http/pprof 默认 block/mutex 采样率为 0（无数据），调试版显式开启：
    // block：goroutine 阻塞等待（锁、channel、IO 排队）；mutex：锁竞争持有时间。
    // 仅调试镜像开启，采样开销在可接受范围。
    runtime.SetBlockProfileRate(1)
    runtime.SetMutexProfileFraction(1)

    addr := os.Getenv("HICLAW_PPROF_ADDR")
    if addr == "" {
        addr = "0.0.0.0:6060"
    }
    srv := &http.Server{Addr: addr, ReadHeaderTimeout: 5 * time.Second}
    go func() {
        <-ctx.Done()
        sctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
        defer cancel()
        _ = srv.Shutdown(sctx)
    }()
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            fmt.Fprintf(os.Stderr, "pprof server: %v\n", err)
        }
    }()
}

// cmd/controller/pprof_stub.go  (//go:build !pprof)
package main

import "context"

func maybeStartPprof(context.Context) {}
```

`main.go` 在 `application.Start(ctx)` 前调用：

```go
maybeStartPprof(ctx)   // 默认 no-op；-tags pprof 构建时启用
```

### 5.3 Dockerfile 改动

```dockerfile
# hiclaw-controller/Dockerfile builder 阶段
ARG ENABLE_PPROF=false
RUN if [ "$ENABLE_PPROF" = "true" ]; then \
        CGO_ENABLED=1 go build -tags pprof -ldflags '-extldflags "-static"' -o /hiclaw-controller ./cmd/controller/; \
    else \
        CGO_ENABLED=1 go build -ldflags '-extldflags "-static"' -o /hiclaw-controller ./cmd/controller/; \
    fi
```

`Dockerfile.embedded`（embedded 模式）同样处理（或复用同一 builder stage 逻辑）。

### 5.4 Makefile 用法

现有 `DOCKER_BUILD_ARGS` 已透传 build args，无需改 Makefile：

```bash
# 默认构建（不含 pprof）
make build-hiclaw-controller

# pprof 调试版本
make build-hiclaw-controller DOCKER_BUILD_ARGS="--build-arg ENABLE_PPROF=true"
```

### 5.5 容器内采样与数据导出

**前置**：controller 镜像已含 `curl`（Dockerfile `apk add bash jq curl unzip kubectl`），两种方式都可行。

**方式 A（推荐）：kubectl port-forward + 本地 curl，数据直接落本地**

```bash
kubectl port-forward -n <ns> <controller-pod> 6060:6060 &
PF_PID=$!

# CPU 采样 120s（覆盖一次完整调谐）
curl -s -o /tmp/cpu.pprof  "http://127.0.0.1:6060/debug/pprof/profile?seconds=120"
# 内存/协程/锁快照（瞬时）
curl -s -o /tmp/heap.pprof   "http://127.0.0.1:6060/debug/pprof/heap"
curl -s -o /tmp/goroutine.txt "http://127.0.0.1:6060/debug/pprof/goroutine?debug=2"
curl -s -o /tmp/block.pprof  "http://127.0.0.1:6060/debug/pprof/block"
curl -s -o /tmp/mutex.pprof  "http://127.0.0.1:6060/debug/pprof/mutex"
# 执行轨迹 60s（事件时间线，看等待/阻塞序列）
curl -s -o /tmp/trace.out   "http://127.0.0.1:6060/debug/pprof/trace?seconds=60"

kill $PF_PID
```

**方式 B：容器内采样 + kubectl cp 导出**（无法开 port-forward 时）

```bash
kubectl exec -n <ns> <controller-pod> -- \
  curl -s "http://127.0.0.1:6060/debug/pprof/profile?seconds=120" -o /tmp/cpu.pprof
kubectl cp -n <ns> <controller-pod>:/tmp/cpu.pprof ./cpu.pprof
kubectl exec -n <ns> <controller-pod> -- rm -f /tmp/cpu.pprof
```

**数据端点清单与用途**：

| 端点 | 文件 | 用途 | 本项目分析价值 |
|------|------|------|----------------|
| `/debug/pprof/profile?seconds=N` | cpu.pprof | CPU 热点（函数级 self/cum） | 定位 mc 子进程 exec、JSON 编解码、hash 计算热点 |
| `/debug/pprof/heap` | heap.pprof | 内存分配/常驻 | 大对象、缓存泄漏 |
| `/debug/pprof/goroutine?debug=2` | goroutine.txt | 全量 goroutine 栈（文本） | **卡死/慢调用首选**：直接看所有 goroutine 阻塞在哪个调用（如 `ensureImage` io.Copy、`exec.Command` Wait、HTTP 等待） |
| `/debug/pprof/block` | block.pprof | goroutine 阻塞等待（需 `SetBlockProfileRate`） | `LegacyCompat.mu` 排队、channel 等待、IO 阻塞 |
| `/debug/pprof/mutex` | mutex.pprof | 锁竞争持有时间（需 `SetMutexProfileFraction`） | registry 全局锁、客户端内部锁 |
| `/debug/pprof/trace?seconds=N` | trace.out | 执行轨迹（goroutine 调度/事件时间线） | 串行依赖、GC 停顿、调度延迟 |

### 5.6 与调谐联动的采样流程

调谐是事件驱动的，采样窗口必须覆盖一次真实调谐。**触发方式选择**（当前 HEAD 已有 Active 短路 `team_controller.go:256`：`Phase==Active && Generation==ObservedGeneration` 时跳过全量调谐）：

| 触发方式 | 效果 | 适用场景 |
|----------|------|----------|
| **删一个成员 pod**（推荐） | 成员不健康 → 短路检查失败 → 走全量调谐；不改 spec、不重建容器（IsUpdate=true 且 spec 未变），采到的正是**存量调谐**热点 | step 4 耗时 3m10s 场景 |
| 创建临时 Team | 全新首次收敛（耗时最长的场景），不污染现有环境 | 首次建队耗时分析 |
| 修改 spec（如 leader.env） | generation bump → 全量调谐，但会触发容器重建 | 容器重建/镜像拉取耗时分析 |
| 重启 controller pod | informer re-sync 全量入队，但 Active 且未变的 team 走短路（秒级） | 不适合采 step 4 热点 |

**推荐时序**（删成员 pod 方式）：

```bash
# 1. 确认目标 team 与成员 pod
kubectl get pods -n <ns> -l hiclaw.io/team=<team-name>

# 2. 开采样（后台 120s，覆盖完整调谐窗口）
kubectl port-forward -n <ns> <controller-pod> 6060:6060 &
curl -s -o /tmp/cpu.pprof "http://127.0.0.1:6060/debug/pprof/profile?seconds=120" &
curl -s -o /tmp/trace.out "http://127.0.0.1:6060/debug/pprof/trace?seconds=120" &

# 3. 触发调谐（采样开始后立即执行）
kubectl delete pod -n <ns> <member-pod>

# 4. 采样结束后确认调谐窗口被覆盖（看耗时日志）
kubectl logs -n <ns> <controller-pod> --tail 200 | grep "team=<team-name>" | grep "step 4"

# 5. 补拍协程快照（若怀疑卡死/阻塞）
curl -s "http://127.0.0.1:6060/debug/pprof/goroutine?debug=2" -o /tmp/goroutine.txt
```

### 5.7 数据分析命令（本地执行）

```bash
# 火焰图 / 交互界面（浏览器）
go tool pprof -http=:8081 /tmp/cpu.pprof

# 文本 top（按 self 耗时排序，前 30）
go tool pprof -top -nodecount=30 /tmp/cpu.pprof

# 完整调用栈轨迹（-cum 按累计耗时排序，看"谁调用了谁"）
go tool pprof -traces -cum /tmp/cpu.pprof

# 定位某个函数的行级耗时（正则匹配，如 mc/exec/ReconcileMemberConfig）
go tool pprof -list 'ReconcileMemberConfig|runMC|exec' /tmp/cpu.pprof

# 内存：常驻 vs 分配
go tool pprof -sample_index=inuse_space -top /tmp/heap.pprof
go tool pprof -sample_index=alloc_space  -top /tmp/heap.pprof

# 锁/阻塞
go tool pprof -top /tmp/mutex.pprof
go tool pprof -top /tmp/block.pprof

# 协程文本快照：直接读，找阻塞调用（grep 关键帧）
grep -B2 -A8 'os/exec|mc |io.Copy|ListenAndServe|reconcile' /tmp/goroutine.txt

# 执行轨迹：本地起 trace 查看器
go tool trace /tmp/trace.out
```

**本项目常见热点的 profile 特征**：

| 症状 | 特征 | 对应问题 |
|------|------|----------|
| `os/exec` + `syscall` 频繁出现 | CPU profile 里大量 exec 相关帧 | mc 子进程风暴（第 1.3 节） |
| goroutine 大量阻塞在 `exec.(*Cmd).Wait` / `io.Copy` | goroutine.txt 同类栈成百上千 | mc 子进程 / `ensureImage` 卡死 |
| `sync.(*Mutex).Lock` 排队 | mutex.pprof 高持有时间 | `LegacyCompat.mu` registry 串行（第 2.3 节） |
| goroutine 阻塞在 HTTP roundtrip | goroutine.txt 中 `net/http` 等待帧 | Matrix / Higress / MinIO 网络往返 |

### 5.8 供 agent 深度分析的数据包

将以下文件放入同一目录（如 `pprof-data/<日期>/`），连同调谐日志一起交给分析 agent：

```
pprof-data/
├── cpu.pprof           # 二进制（agent 可用 go tool pprof 文本化后分析）
├── cpu-top.txt         # go tool pprof -top -nodecount=50 cpu.pprof
├── cpu-traces.txt      # go tool pprof -traces -cum cpu.pprof
├── cpu-list-config.txt # go tool pprof -list 'ReconcileMemberConfig|runMC|DeployWorkerConfig' cpu.pprof
├── heap.pprof
├── mutex.pprof
├── block.pprof
├── goroutine.txt       # 文本，直接可读
├── trace.out
└── reconcile.log       # kubectl logs --tail 1000 | grep "team=<team-name>"
                        # （含 step/成员阶段 elapsed，与 profile 时间轴对照）
```

**给 agent 的分析提示模板**：先读 `goroutine.txt` 找阻塞点 → 用 `cpu-top.txt`/`cpu-traces.txt` 确认 CPU 热点 → 对照 `reconcile.log` 的 elapsed 验证哪个阶段与 profile 热点对应 → 用 `-list` 输出下钻到具体函数行。

### 5.9 容器内一键采集脚本 collect-pprof-inpod.sh

只对 controller 进程采样并打包，**不管 team/pod**；CPU 与阻塞相关采样（goroutine 多时间点 / block / mutex / trace）一次在脚本内自动完成。目标场景由你在采样窗口内自行触发，**调谐日志自行 grep，脚本不处理**。

**采样时长配置**：

| 参数 | 说明 |
|------|------|
| 位置参数 1（秒数） | 采样总时长，默认 120。**建议 = 目标场景预计耗时 + 60s 余量**：已知调谐约 3m10s → `260`；不确定时先用 120 试采，对比日志 step 4 的 elapsed 确认窗口覆盖，不够再加大 |
| 位置参数 2（快照次数） | goroutine/heap 周期快照次数，默认 4；快照间隔自动 = 时长/次数（最小 10s），均匀覆盖整个采样窗口 |

```bash
bash /tmp/collect-pprof-inpod.sh              # 120s，4 次快照
bash /tmp/collect-pprof-inpod.sh 260          # 覆盖 3m10s 的调谐
bash /tmp/collect-pprof-inpod.sh 260 6        # 快照更密（每 ~43s 一次）
```

**执行流程**（放置、执行、拷包、抓日志由你自行操作）：

```bash
# 1. 节点上：把脚本放进容器并执行
kubectl cp collect-pprof-inpod.sh <ns>/<controller-pod>:/tmp/
kubectl exec -it <ns>/<controller-pod> -- bash /tmp/collect-pprof-inpod.sh 260

# 2. 采样期间（脚本提示后），在节点上自行触发目标场景，例如：
kubectl delete pod -n <ns> <member-pod>       # 触发一次 Team 调谐

# 3. 拷出 tar 包 + 自行抓调谐日志
kubectl cp <ns>/<controller-pod>:/tmp/pprof-<时间戳>.tar.gz ./
kubectl logs -n <ns> <controller-pod> --tail=5000 | grep "team=<team>" > reconcile.log
```

脚本内容：

```bash
#!/usr/bin/env bash
# collect-pprof-inpod.sh — 容器内对 controller 进程做性能/阻塞采样并打包
# 用法: bash collect-pprof-inpod.sh [采样秒数=120] [快照次数=4]
#   采样秒数: 建议 = 目标场景预计耗时 + 60s 余量（如调谐 3m10s → 260）
#   快照次数: goroutine/heap 周期快照次数，间隔自动 = 秒数/次数（最小 10s）
set -euo pipefail

DUR=${1:-120}
SNAPS=${2:-4}
P="http://127.0.0.1:6060/debug/pprof"
OUT=/tmp/pprof-$(date +%Y%m%d-%H%M%S)
mkdir -p "$OUT"
JOBS=()

# --- 基线快照（采样开始前，瞬时） ---
curl -s -o "$OUT/goroutine-0.txt" "$P/goroutine?debug=2"
curl -s -o "$OUT/heap-0.pprof"    "$P/heap"

# --- 全程采样（CPU + trace，阻塞 DUR 秒） ---
curl -s -o "$OUT/cpu.pprof" "$P/profile?seconds=$DUR" & JOBS+=($!)
curl -s -o "$OUT/trace.out" "$P/trace?seconds=$DUR"  & JOBS+=($!)

# --- 采样中周期快照（goroutine/heap，间隔 = DUR/SNAPS） ---
SNAP=$((DUR / SNAPS)); [[ $SNAP -lt 10 ]] && SNAP=10
for i in $(seq 1 $SNAPS); do
  T=$((i * SNAP))
  [[ $T -ge $DUR ]] && break
  ( sleep $T
    curl -s -o "$OUT/goroutine-$i.txt" "$P/goroutine?debug=2"
    curl -s -o "$OUT/heap-$i.pprof"    "$P/heap" ) & JOBS+=($!)
done

# --- 阻塞相关：block/mutex（首尾各一次，需 5.2 节开启采样率） ---
( sleep 3
  curl -s -o "$OUT/block-1.pprof" "$P/block"
  curl -s -o "$OUT/mutex-1.pprof" "$P/mutex" ) & JOBS+=($!)
( sleep $((DUR - 5))
  curl -s -o "$OUT/block-2.pprof" "$P/block"
  curl -s -o "$OUT/mutex-2.pprof" "$P/mutex" ) & JOBS+=($!)

echo "==> CPU/trace 采样 ${DUR}s 进行中，请触发目标场景（如删成员 pod 触发调谐）..."
wait "${JOBS[@]}" || true
sleep 2

tar -czf "$OUT.tar.gz" -C /tmp "$(basename "$OUT")"
echo "==> 完成: $OUT.tar.gz"
echo "    文件: $(ls "$OUT" | tr '\n' ' ')"
echo "    kubectl cp <controller-pod>:$OUT.tar.gz ./"
```

### 5.10 安全与运维说明

| 项 | 说明 |
|----|------|
| 默认行为 | 无 pprof 代码编译，无监听端口（`!pprof` stub no-op） |
| 开启方式 | 构建时 `--build-arg ENABLE_PPROF=true`，**仅限调试镜像**，不得进生产发布流程 |
| 端口 | 6060（env `HICLAW_PPROF_ADDR` 可改） |
| 生命周期 | 随进程退出；ctx 取消后 3s 优雅关闭 |
| 对照方案 | 运行时 env 开关（`HICLAW_PPROF_ENABLED`）实现更简单但镜像含 pprof 代码；如需按需热启可后续补充 |

---

## 6. 后续行动建议

| 优先级 | 事项 | 工作量 |
|--------|------|--------|
| P0 | 部署带监测日志的版本（第 3 节），收集 step 4 各子阶段真实耗时数据 | 半天 |
| P0 | 用第 4.2 节 Step 2 写探针确认 CRD 字段问题归属，补齐 `reconcileAttempt`/`phaseTransitionTime` 到 helm CRD yaml | 10 分钟 |
| P1 | 依据数据决定：`runMC` 阈值日志 → minio-go SDK 替换（消除子进程风暴） | 1~2 天 |
| P1 | pprof 构建开关落地（第 5 节），采样调谐热点验证 | 半天 |
| P2 | `LegacyCompat.mu` registry 读改写加内存缓存 | 半天 |
