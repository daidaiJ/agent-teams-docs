# Team Controller 调谐耗时定位与观测排查方案

> 文档版本: v1.2
> 日期: 2026-08-05
> 基线代码: `AgentTeams` HEAD（`48ce4aa` / `c01aaec` 修复后，含 per-step timing logs 与可配置并发）
> 关联文档: [team-controller-performance.md](team-controller-performance.md)（修复前分析）、[team-controller-defects.md](team-controller-defects.md)（缺陷清单）
> v1.1 新增: S3 层根因复现基准（第 6 节）、控制器调用方式 vs S3 基座两轨优化对照（第 7 节）
> v1.2 更新: 适配云 S3 部署（底层为服务商 OSS）——VPC 内网 endpoint 优先（E6b）、限流排查（E8）、SeaweedFS 降级为自建评估

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

## 6. mc 确认后：S3 层根因定位与复现基准（bench）

**前提**：第 1.3/1.4 节已确认 config 阶段耗时为 mc 调用主导（runMC 阈值日志显示单次 >300ms），第 2.2 节"存储规模正反馈"机制成立。本节用**可控复现基准**把瓶颈拆到具体层，回答三个问题：

1. 瓶颈在**调用模式**（exec fork / 新 TLS 连接 / 无连接池）还是 **S3 服务端**？
2. 瓶颈是否随**对象规模 / 对象大小 / 并发**放大（复现"team 越多越慢"）？
3. 自建 SeaweedFS 替换云 S3 是否值得（默认不做，降级动作）？

### 6.1 复现负载：对齐真实调用模式

代码事实（当前 HEAD）：

| 事实 | 位置 |
|------|------|
| 探活走 `GetObject` = `mc cat` **全量下载**，内容丢弃 | `deployer.go:479`（seedLocalAgentFiles）、`:263`（SOUL.md）、`:362`（AGENTS.md）、`:862`（pushBuiltinTopLevelFiles） |
| `PutObject` = 写临时文件 + `mc cp` | `minio.go:88~96` |
| 每成员每轮调谐 10~20 次调用，**GET 为主** | 见 1.3 节表 |

**bench 负载**：每"成员轮次" = 12 GET + 3 PUT + 2 STAT + 1 LIST（18 次调用，GET 主导），轮次墙钟耗时直接对标真实 config 阶段。

**测试量不需要大**：环境里其他 team 的 worker 每 5min 周期调谐、事件触发调谐，本身就在向 S3 持续打负载。bench 是**搭车测量**——用小样本（默认 rounds=10、workers=1,5）测"真实背景负载叠加下"的单次调用延迟，而非自己制造负载。加大 rounds/workers 只会增加对环境的冲击，不增加信息量；更有意义的变量是**时段**（调谐高峰 vs 低谷各跑一次对比）。

### 6.2 bench_s3.go：双驱动 + 真实桶复用

单文件 Go 程序，两个驱动跑**同一份负载**：

| 驱动 | 实现 | 对应生产路径 |
|------|------|-------------|
| `mc` | 每次调用 `exec.CommandContext("mc", ...)`（PUT 先写临时文件） | `minio.go` 现状 |
| `sdk` | minio-go 连接池直连 | performance 文档 6.3.1 优化方案 |

**复用场景桶，不预置负载**（省 seed 成本、零污染）：

| 空间 | 范围 | 操作 | 说明 |
|------|------|------|------|
| 读空间 | `-prefix` 下采样的真实 key（`-probe-n` 个） | stat / get / list | 只读，不修改任何数据；key 大小/分布即真实 |
| 写空间 | `bench-probe/` 前缀（固定 16 个 key） | put | 前缀隔离，跑完 `-clean` 清理；`-write=false` 完全关闭 |
| 规模 | `background` 列 | — | 云 S3 全桶 LIST 分页贵且耗 QPS，默认 `-count=false` 用控制台对象数；换 `-bucket/-prefix` 即规模对比实验 |

实验维度由 flag 控制：操作类型（`-ops`）、并发 worker（`-workers`）、对象大小（`-write-size`，写空间）、网络/后端（`-endpoint`：公网 vs OSS VPC 内网 `-internal` endpoint，或自建评估时才换引擎）。

```go
// bench_s3.go — S3 层瓶颈复现基准 v2（复用真实桶，不预置负载）
//
// 设计: 不向桶里压入测试对象（省 seed 成本），直接复用场景中已有的 S3 桶:
//   - 读空间: 从 -prefix 下采样真实 key 池，做 stat/get/list（只读，零污染）
//   - 写空间: bench-probe/ 前缀，做 put（-write=false 关闭），跑完 -clean 清理
//   - 操作类型对比（主目标）: 在环境背景调谐压力下，对比 stat/get/put/list
//     各类型的单次调用延迟分布，直接暴露"哪种类型负载最耗时"（E7 主实验）
//   - 调用模式对比: mc vs sdk 同负载对比，量化 exec/TLS/无连接池开销（E1 辅助）
//   - 背景规模: 启动时全桶统计对象数记入 CSV（background 列）；云 S3 全桶 LIST
//     分页开销大且耗 QPS，默认关闭（-count=false），用 OSS 控制台对象数代替；
//     用同一命令换 -bucket/-prefix 指向不同规模桶即规模效应实验（E2）
//   - 测试量小: 环境里其他 team 的 worker 每 5min 周期调谐/事件调谐本身就在
//     持续打 S3 负载，bench 只做小样本"搭车测量"，无需大 rounds/workers
//
// 用法:
//   go mod init bench-s3 && go get github.com/minio/minio-go/v7
//   # 对场景桶跑完整负载（mc vs sdk，含写路径，跑完自动清 bench-probe/）:
//   go run bench_s3.go -endpoint http://127.0.0.1:9000 -ak minioadmin -sk minioadmin \
//     -bucket hiclaw -prefix agents/ -mc /usr/local/bin/mc -alias bench \
//     -drivers mc,sdk -workers 1,5 -rounds 10
//   # 只读探测（不改动桶任何数据）:
//   ... 同参数 -write=false
//   # 规模对比: 同一命令换 -bucket/-prefix 指向不同规模桶，对比 CSV 的 background 列
//   # 背景负载强度对比: 同一桶在调谐高峰/低谷时段各跑一次（比加大 rounds 更有意义）
//   # 冷启动数据: -warmup 0
//
// 输出 CSV 到 stdout: driver,background,workers,op,count,avg_ms,p50_ms,p95_ms,p99_ms
// op=round 行 = 每成员轮次（默认 12 GET + 3 PUT + 2 STAT + 1 LIST）墙钟耗时，
// 直接对标真实 config 阶段。
//
// 注意: 对生产桶跑 bench 会向生产存储打真实负载，建议低 rounds 或非高峰时段。
package main

import (
	"bytes"
	"context"
	"encoding/csv"
	"flag"
	"fmt"
	"io"
	"math/rand"
	"os"
	"os/exec"
	"path/filepath"
	"sort"
	"strconv"
	"strings"
	"sync"
	"time"

	"github.com/minio/minio-go/v7"
	"github.com/minio/minio-go/v7/pkg/credentials"
)

// 默认轮次配比: 对齐 DeployWorkerConfig 每成员每轮的真实操作配比（GET 主导）
const (
	poolSize   = 64 // 真实 key 采样池上限
	probeWrite = 16 // 写空间 key 数（bench-probe/ 前缀）
)

var (
	flagEndpoint = flag.String("endpoint", "http://127.0.0.1:9000", "S3 endpoint（scheme 决定 TLS，换 endpoint 即换引擎/网络实验）")
	flagAK       = flag.String("ak", "minioadmin", "access key")
	flagSK       = flag.String("sk", "minioadmin", "secret key")
	flagBucket   = flag.String("bucket", "hiclaw", "被测桶（场景真实桶，读空间不修改任何数据）")
	flagPrefix   = flag.String("prefix", "agents/", "被测真实 key 前缀（读空间采样范围）")
	flagOps      = flag.String("ops", "stat,get,put,list", "操作类型集合: stat,get,put,list")
	flagWrite    = flag.Bool("write", true, "写路径测试（put 写到 bench-probe/ 前缀，跑完清理）")
	flagWriteSz  = flag.Int("write-size", 8192, "写空间对象大小字节（E3 对象大小实验用）")
	flagMC       = flag.String("mc", "mc", "mc 二进制路径")
	flagAlias    = flag.String("alias", "bench", "mc alias 名")
	flagDrivers  = flag.String("drivers", "mc,sdk", "驱动（逗号分隔）: mc=子进程现状, sdk=minio-go 连接池")
	flagWorkers  = flag.String("workers", "1,5", "并发 worker 数（逗号分隔）；小样本即可，环境自身调谐就是负载")
	flagRounds   = flag.Int("rounds", 10, "每 worker 正式计时的轮次数（小样本，背景负载由其他 team 调谐提供）")
	flagWarmup   = flag.Int("warmup", 2, "正式计时前的预热轮次（抹平服务端缓存顺序偏差）")
	flagClean    = flag.Bool("clean", true, "run 结束时清理写空间 bench-probe/（防残留污染）")
	flagProbeN   = flag.Int("probe-n", 64, "真实 key 采样池大小")
	flagCount    = flag.Bool("count", false, "全桶统计对象数（云 S3 上 LIST 分页贵，默认关，用控制台对象数）")
)

type rec struct {
	driver     string
	background int // 桶内真实对象总数（CSV background 列）
	workers    int
	op         string // get/put/stat/list/round
	ms         []float64
}

func main() {
	flag.Parse()
	ws := parseInts(*flagWorkers)
	drvs := strings.Split(*flagDrivers, ",")
	ops := parseOps(*flagOps)

	ctx := context.Background()
	sdk := newSDK()
	ensureBucket(ctx, sdk)
	setupMC() // mc alias set（仅一次，对齐生产静态凭据模式）

	// 背景规模: 默认不统计（云 S3 全桶 LIST 分页贵且耗 QPS），-count 开启；
	// 推荐直接用 OSS 控制台的对象数
	background := -1
	if *flagCount {
		background = countObjects(ctx, sdk)
	}
	fmt.Fprintf(os.Stderr, "bucket %q background=%d (-count 开启全桶统计；云 S3 建议用控制台对象数)\n", *flagBucket, background)

	// 读空间: 从真实前缀采样 key 池（不修改任何数据）
	readKeys := sampleKeys(ctx, sdk, *flagPrefix, *flagProbeN)
	if len(readKeys) == 0 {
		fmt.Fprintf(os.Stderr, "fatal: prefix %q 下没有对象\n", *flagPrefix)
		os.Exit(1)
	}

	// 写空间 key（bench-probe/ 前缀，跑完清理）
	writeKeys := make([]string, probeWrite)
	for i := range writeKeys {
		writeKeys[i] = fmt.Sprintf("bench-probe/%03d", i)
	}
	writeBlob := make([]byte, *flagWriteSz)
	rand.New(rand.NewSource(42)).Read(writeBlob)

	tmpDir, err := os.MkdirTemp("", "bench-s3-")
	must(err)
	defer os.RemoveAll(tmpDir)

	var all []*rec
	for _, w := range ws {
		for _, d := range drvs {
			for _, r := range benchOnce(ctx, d, sdk, w, ops, readKeys, writeKeys, writeBlob, tmpDir) {
				r.background = background
				r.workers = w
				all = append(all, r)
			}
		}
	}
	writeCSV(all)
	writeSummary(all) // 人眼可读的操作类型对比（主目标输出）

	if *flagClean {
		// 只清写空间，不动真实数据
		cleanPrefix(ctx, sdk, "bench-probe/")
	}
}

// parseOps 把 "stat,get,put,list" 解析为集合；put 受 -write 开关控制
func parseOps(s string) map[string]bool {
	m := map[string]bool{}
	for _, o := range strings.Split(s, ",") {
		o = strings.TrimSpace(o)
		if o != "" && (o == "stat" || o == "get" || o == "put" || o == "list") {
			m[o] = true
		}
	}
	if !*flagWrite {
		delete(m, "put")
	}
	return m
}

// benchOnce 以 W 个并发 worker 各跑 rounds 轮，返回各 op 与 round 的耗时样本
func benchOnce(ctx context.Context, drv string, sdk *minio.Client, workers int, ops map[string]bool, readKeys, writeKeys []string, blob []byte, tmpDir string) []*rec {
	// 每轮配比（对齐生产 config 阶段: GET 主导，12 GET + 3 PUT + 2 STAT + 1 LIST）
	var mix []string
	if ops["get"] {
		for i := 0; i < 12; i++ {
			mix = append(mix, "get")
		}
	}
	if ops["put"] {
		for i := 0; i < 3; i++ {
			mix = append(mix, "put")
		}
	}
	if ops["stat"] {
		for i := 0; i < 2; i++ {
			mix = append(mix, "stat")
		}
	}
	if ops["list"] {
		mix = append(mix, "list")
	}

	var mu sync.Mutex
	times := map[string][]float64{}
	var roundMs []float64

	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func(w int) {
			defer wg.Done()
			rng := rand.New(rand.NewSource(int64(w)))
			tmp := filepath.Join(tmpDir, fmt.Sprintf("put-%d.tmp", w))
			for r := 0; r < *flagWarmup+*flagRounds; r++ {
				rstart := time.Now()
				for _, op := range mix {
					start := time.Now()
					var err error
					switch op {
					case "get":
						err = opGet(ctx, drv, sdk, readKeys[rng.Intn(len(readKeys))])
					case "put":
						err = opPut(ctx, drv, sdk, writeKeys[rng.Intn(len(writeKeys))], blob, tmp)
					case "stat":
						err = opStat(ctx, drv, sdk, readKeys[rng.Intn(len(readKeys))])
					case "list":
						err = opList(ctx, drv, sdk)
					}
					if err != nil {
						fmt.Fprintf(os.Stderr, "fatal: %s %s: %v\n", drv, op, err)
						os.Exit(1)
					}
					elapsed := time.Since(start).Seconds() * 1000
					if r < *flagWarmup {
						continue // 预热轮: 不记录，让服务端缓存进入热态
					}
					mu.Lock()
					times[op] = append(times[op], elapsed)
					mu.Unlock()
				}
				if r < *flagWarmup {
					continue
				}
				mu.Lock()
				roundMs = append(roundMs, time.Since(rstart).Seconds()*1000)
				mu.Unlock()
			}
		}(w)
	}
	wg.Wait()

	var out []*rec
	for _, op := range []string{"get", "put", "stat", "list"} {
		if len(times[op]) > 0 {
			out = append(out, &rec{driver: drv, op: op, ms: times[op]})
		}
	}
	out = append(out, &rec{driver: drv, op: "round", ms: roundMs})
	return out
}

// ---- 两个驱动：mc 子进程（现状）与 minio-go SDK（优化方案）----

func mcRun(ctx context.Context, args ...string) error {
	cmd := exec.CommandContext(ctx, *flagMC, args...)
	var stderr bytes.Buffer
	cmd.Stderr = &stderr
	if err := cmd.Run(); err != nil {
		return fmt.Errorf("mc %s: %w (%s)", strings.Join(args, " "), err, strings.TrimSpace(stderr.String()))
	}
	return nil
}

func opGet(ctx context.Context, drv string, sdk *minio.Client, key string) error {
	switch drv {
	case "mc":
		return mcRun(ctx, "cat", *flagAlias+"/"+*flagBucket+"/"+key)
	case "sdk":
		obj, err := sdk.GetObject(ctx, *flagBucket, key, minio.GetObjectOptions{})
		if err != nil {
			return err
		}
		defer obj.Close()
		_, err = io.Copy(io.Discard, obj)
		return err
	}
	return fmt.Errorf("unknown driver %q", drv)
}

func opPut(ctx context.Context, drv string, sdk *minio.Client, key string, blob []byte, tmp string) error {
	switch drv {
	case "mc":
		// 镜像 minio.go PutObject: 先写临时文件再 mc cp
		if err := os.WriteFile(tmp, blob, 0o644); err != nil {
			return err
		}
		return mcRun(ctx, "cp", tmp, *flagAlias+"/"+*flagBucket+"/"+key)
	case "sdk":
		_, err := sdk.PutObject(ctx, *flagBucket, key, bytes.NewReader(blob), int64(len(blob)), minio.PutObjectOptions{})
		return err
	}
	return fmt.Errorf("unknown driver %q", drv)
}

func opStat(ctx context.Context, drv string, sdk *minio.Client, key string) error {
	switch drv {
	case "mc":
		return mcRun(ctx, "stat", *flagAlias+"/"+*flagBucket+"/"+key)
	case "sdk":
		_, err := sdk.StatObject(ctx, *flagBucket, key, minio.StatObjectOptions{})
		return err
	}
	return fmt.Errorf("unknown driver %q", drv)
}

func opList(ctx context.Context, drv string, sdk *minio.Client) error {
	switch drv {
	case "mc":
		return mcRun(ctx, "ls", *flagAlias+"/"+*flagBucket+"/"+*flagPrefix)
	case "sdk":
		// 取首响应即返回: LIST 延迟 = 服务端返回首个条目耗时（非全量遍历）
		for obj := range sdk.ListObjects(ctx, *flagBucket, minio.ListObjectsOptions{Prefix: *flagPrefix}) {
			if obj.Err != nil {
				return obj.Err
			}
			break
		}
		return nil
	}
	return fmt.Errorf("unknown driver %q", drv)
}

// ---- 初始化与工具 ----

func newSDK() *minio.Client {
	secure := strings.HasPrefix(*flagEndpoint, "https://")
	host := strings.TrimPrefix(*flagEndpoint, "https://")
	host = strings.TrimPrefix(host, "http://")
	c, err := minio.New(host, &minio.Options{
		Creds:  credentials.NewStaticV4(*flagAK, *flagSK, ""),
		Secure: secure,
	})
	must(err)
	return c
}

func ensureBucket(ctx context.Context, sdk *minio.Client) {
	exists, err := sdk.BucketExists(ctx, *flagBucket)
	must(err)
	if !exists {
		fmt.Fprintf(os.Stderr, "fatal: bucket %q 不存在（bench 不创建桶，只复用场景已有桶）\n", *flagBucket)
		os.Exit(1)
	}
}

func setupMC() {
	if err := mcRun(context.Background(), "alias", "set", *flagAlias, *flagEndpoint, *flagAK, *flagSK); err != nil {
		fmt.Fprintln(os.Stderr, "fatal:", err)
		os.Exit(1)
	}
}

// countObjects 全桶统计对象数（一次 LIST 递归遍历，服务端负担远小于预置负载）
func countObjects(ctx context.Context, sdk *minio.Client) int {
	n := 0
	for obj := range sdk.ListObjects(ctx, *flagBucket, minio.ListObjectsOptions{Recursive: true}) {
		if obj.Err != nil {
			fmt.Fprintf(os.Stderr, "list warning: %v\n", obj.Err)
			break
		}
		n++
	}
	return n
}

// sampleKeys 从真实前缀采样至多 n 个 key 作为读空间（只读，不修改任何数据）
func sampleKeys(ctx context.Context, sdk *minio.Client, prefix string, n int) []string {
	var keys []string
	for obj := range sdk.ListObjects(ctx, *flagBucket, minio.ListObjectsOptions{Prefix: prefix, Recursive: true}) {
		if obj.Err != nil {
			fmt.Fprintf(os.Stderr, "list warning: %v\n", obj.Err)
			break
		}
		keys = append(keys, obj.Key)
		if len(keys) >= n {
			break
		}
	}
	return keys
}

// cleanPrefix 删除指定前缀下全部对象（prefix="" 即清空整个 bucket）。
// 用 SDK 批量删除（LIST + 批量 DELETE），不用 mc，避免 bench 依赖清理路径。
func cleanPrefix(ctx context.Context, sdk *minio.Client, prefix string) {
	objectsCh := sdk.ListObjects(ctx, *flagBucket, minio.ListObjectsOptions{Prefix: prefix, Recursive: true})
	removeCh := sdk.RemoveObjects(ctx, *flagBucket, objectsCh, minio.RemoveObjectsOptions{})
	for e := range removeCh {
		if e.Err != nil {
			fmt.Fprintf(os.Stderr, "cleanup warning: %v\n", e.Err)
		}
	}
}

func writeCSV(recs []*rec) {
	w := csv.NewWriter(os.Stdout)
	_ = w.Write([]string{"driver", "background", "workers", "op", "count", "avg_ms", "p50_ms", "p95_ms", "p99_ms"})
	for _, r := range recs {
		if len(r.ms) == 0 {
			continue
		}
		sorted := append([]float64(nil), r.ms...)
		sort.Float64s(sorted)
		var sum float64
		for _, v := range sorted {
			sum += v
		}
		_ = w.Write([]string{
			r.driver, strconv.Itoa(r.background), strconv.Itoa(r.workers), r.op,
			strconv.Itoa(len(sorted)),
			strconv.FormatFloat(sum/float64(len(sorted)), 'f', 1, 64),
			strconv.FormatFloat(pct(sorted, 0.50), 'f', 1, 64),
			strconv.FormatFloat(pct(sorted, 0.95), 'f', 1, 64),
			strconv.FormatFloat(pct(sorted, 0.99), 'f', 1, 64),
		})
	}
	w.Flush()
}

// writeSummary 输出人眼可读的操作类型对比（主目标: 背景压力下哪种负载最耗时）
func writeSummary(recs []*rec) {
	type agg struct {
		avg, p50, p95 float64
	}
	by := map[string]map[string]agg{} // driver -> op -> agg
	for _, r := range recs {
		if len(r.ms) == 0 {
			continue
		}
		sorted := append([]float64(nil), r.ms...)
		sort.Float64s(sorted)
		var sum float64
		for _, v := range sorted {
			sum += v
		}
		if by[r.driver] == nil {
			by[r.driver] = map[string]agg{}
		}
		by[r.driver][r.op] = agg{sum / float64(len(sorted)), pct(sorted, 0.50), pct(sorted, 0.95)}
	}
	fmt.Fprintln(os.Stderr, "\n== 操作类型对比（背景压力下单次调用延迟 ms；round=一成员轮次墙钟）==")
	for d, ops := range by {
		fmt.Fprintf(os.Stderr, "driver=%s\n", d)
		fmt.Fprintf(os.Stderr, "  %-6s %9s %9s %9s\n", "op", "avg", "p50", "p95")
		for _, op := range []string{"stat", "get", "put", "list", "round"} {
			if a, ok := ops[op]; ok {
				fmt.Fprintf(os.Stderr, "  %-6s %9.1f %9.1f %9.1f\n", op, a.avg, a.p50, a.p95)
			}
		}
	}
}

func pct(sorted []float64, p float64) float64 {
	if len(sorted) == 0 {
		return 0
	}
	idx := int(p * float64(len(sorted)))
	if idx >= len(sorted) {
		idx = len(sorted) - 1
	}
	return sorted[idx]
}

func parseInts(s string) []int {
	var out []int
	for _, p := range strings.Split(s, ",") {
		p = strings.TrimSpace(p)
		if p == "" {
			continue
		}
		v, err := strconv.Atoi(p)
		must(err)
		out = append(out, v)
	}
	return out
}

func must(err error) {
	if err != nil {
		fmt.Fprintln(os.Stderr, "fatal:", err)
		os.Exit(1)
	}
}
```

### 6.3 实验卫生：防"隐形增量/缓存"污染

复用真实桶后，污染面比预置负载版小得多，但仍需处理：

| 污染源 | 表现 | 处理 |
|--------|------|------|
| **写空间残留** | 上次 run 的 `bench-probe/` 对象残留，PUT 覆盖写导致对象数/大小漂移 | `-clean`（默认 true）run 结束时清理 `bench-probe/` 前缀；读空间从不写 |
| **缓存热效应** | 先跑的驱动/实验把对象喂进服务端缓存（OS page cache / MinIO 缓存），后跑者"沾光"，mc vs sdk 差异被夸大 | `-warmup`（默认 2 轮）正式计时前先跑预热轮不记录，两个驱动都在热态对比 |
| **时段漂移** | 不同时段背景调谐负载强度不同（周期调谐错峰），跨时段对比失真 | 对比实验（如 mc vs sdk、不同桶规模）尽量在同一时段连续跑；需要负载强度数据时，高峰/低谷各跑一次并记录时间 |

说明：
- 预热测的是**稳态性能**，贴近生产真实场景——生产对象本来就被每 5min 周期调谐反复访问，属热对象；如需"首次建队"的冷启动数据，加 `-warmup 0`；
- 清理只针对 `bench-probe/` 前缀（SDK 批量删除），**从不触碰真实数据**；
- 对生产桶跑 bench 会向生产存储叠加少量真实负载（小样本下可忽略），建议低 rounds 或非高峰时段。

### 6.4 实验矩阵与判读

> 顺序即推荐执行顺序。**E7 是主实验**：一次运行即输出全部操作类型在背景压力下的延迟分布，直接回答"哪种类型负载最耗时"；其余实验按需追加定位。

| 实验 | 做法 | 回答的问题 |
|------|------|-----------|
| **E7 ★主实验** | 默认参数跑一次（`-drivers mc` 起步），读 stderr 的 writeSummary 输出 | **背景压力下哪种操作类型最耗时**（stat/get/put/list 分布 + round 墙钟） |
| E1 调用模式 | 同参数补跑 `sdk` 驱动，对比 mc | exec/TLS/无连接池开销占比（若 E7 显示整体都高，此实验定位其中"进程开销"部分） |
| E0 基线 | `-workers 1`，低峰时段 | 单成员轮次的最小延迟基线 |
| **E6b ★云 S3 关键** | 公网 endpoint vs OSS **VPC 内网 endpoint**（`-internal`）各跑一次 E7 | **公网路径成本**（RTT+签名），云 S3 场景最高性价比优化依据 |
| E2 对象规模 | 同一命令换 `-bucket/-prefix` 指向不同规模桶，对比 background 列（云 S3 用控制台对象数） | 规模正反馈是否成立（随 background 上升） |
| E2b 负载敏感度 | 同一桶在调谐高峰/低谷时段各跑一次 E7 | **哪种操作类型对背景压力最敏感**（延迟放大倍数最大） |
| E3 对象大小 | 写空间 `-write-size 0,1024,8192,65536` 各跑一次 | 每对象固定开销 vs 传输开销占比（0 点 = 纯协议+元数据成本） |
| E4 并发 | `-workers 1,5,10` | 服务端在真实背景负载叠加下的并发承受力（为成员并行方案探路；云 S3 关注限流拐点） |
| E8 限流观察 | 监控 429/503 错误与延迟拐点（低优先级，慎用；勿在生产高峰做） | 云 S3 QPS 限流是否触发 |
| E5 引擎对比 | 仅当评估**自建** SeaweedFS 时：同参数换 `-endpoint`（云 S3 / SeaweedFS） | 自建替换云托管是否值得（默认不做） |
| E6 网络/TLS | `http://` vs `https://`（仅自建 MinIO 有意义）、内网 vs 公网 endpoint | 网络层与 TLS 握手开销 |

### 6.5 结果解读决策矩阵

| bench 结果 | 根因 | 针对性解决（指向第 7 节） |
|-----------|------|---------------------------|
| E7：**get** 最耗时（探活/读取） | 探活走全量 GET 下载 + 传输路径 | A1（探活改 HEAD）、A2（跳过探活）、B1（网络/内网 endpoint） |
| E7：**put** 最耗时（写入） | 写路径瓶颈（元数据写/QPS） | B3（升配额）、B4（存储类型）；A 轨减少写入次数 |
| E7：**list** 最耗时 | LIST 路径瓶颈（目录遍历/registry） | B2（对象治理）、A 轨减少 list 调用 |
| E7：**stat** 也高（HEAD 慢） | 服务端基础延迟高（非操作类型问题） | E1/E0 定位：调用模式 or 网络基线 |
| **E6b：内网 endpoint 显著快于公网** | 公网路径成本（RTT+签名+限流） | **B1（切 VPC 内网 endpoint）——云 S3 场景第一优先** |
| **E8/监控：429/503 或限流事件** | 云 S3 QPS 限流 | B3（升配额/降调用频率）、A2/A3（减调用次数） |
| E1：`sdk` round 远快于 `mc`（如 5x+） | 调用模式开销（exec+TLS+无复用） | A3（SDK 替换）；**换引擎无效** |
| E2：round 随 background 显著上升 | 对象规模/元数据效应 | B2（对象治理）；自建评估时才考虑 B5 |
| E2b：某类型高峰/低谷放大倍数最大 | 该类型对背景压力最敏感（如 put 受并发写放大） | 针对该类型降频/批处理；B 轨对应项 |
| E3：大小不敏感、每 op 固定成本高 | 每对象固定开销（RTT+签名+元数据） | A3、A1；B1（内网） |
| E4：并发升高后延迟飙升 | 服务端 QPS 上限/限流 | B3（云 OSS 升配）、A 轨降调用次数 |
| E5（自建评估）：SeaweedFS 显著快 | 云 S3 单请求成本/限流不可接受 | B5 走 7.4 决策门（自建替换云托管是降级，默认不做） |
| E6：https 显著慢于 http（仅自建 MinIO 可测） | TLS 握手 | A3（连接池复用）或内网自签 |

### 6.6 服务端侧交叉验证（与 bench 对照）

**云 S3（当前部署，OSS 控制台为准）**：

```bash
# OSS 控制台监控: 请求数 QPS、平均/最大延迟、错误率、4xx/5xx 分布、限流事件
#   → 与 bench 时段对齐，看限流/延迟是否与服务端记录一致
# OSS access log（可开通日志服务）: 每条请求的 request-id、耗时、状态码
#   → 定位"具体哪个请求慢"（用日志里的 request-id 反查）

# controller 侧对照: 调谐日志里的 429/503 与慢请求
kubectl logs -n <ns> <controller-pod> | grep -E "429|503|SlowDown|RequestTimeout"

# TCP/TLS 路径（公网 vs 内网对比的量化依据）
curl -w "dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer}\n" \
  -o /dev/null -s "https://<bucket>.<region>.aliyuncs.com/<prefix>/<key>"
```

**仅自建 MinIO 时可用**（当前云 S3 部署不适用，保留备查）：

```bash
mc admin info ALIAS
mc admin perf object ALIAS --objects 64 --size 1MiB --concurrent 16
mc admin perf drive ALIAS                       # 磁盘 IOPS/带宽
mc admin prometheus metrics ALIAS | grep -E "s3_requests_total|s3_errors_total|s3_.*_duration"
iostat -x 1                                     # 宿主机磁盘
```

---

## 7. 解决方案：控制器调用方式优化（A 轨）与 S3 基座优化（B 轨）

> 排序原则：**从简单到困难**；每项给出收益与风险；优先级综合"收益 / 成本 / 风险 / 前置依赖"。
> 收益基准来自第 1.3 节拆解（单次 mc 200~400ms、每成员 10~20 次调用）。

### 7.1 总览排序表

| 优先级 | 优化项 | 轨 | 难度/工作量 | 预期收益 | 风险 | 前置依赖 |
|--------|--------|----|------------|----------|------|----------|
| P0 | ① runMC 阈值耗时日志（>300ms） | A | 半天 | 单次延迟与 op 分布真实基线，量化后续每项收益 | 无 | — |
| P0 | ② bench_s3 复现基准（第 6 节） | A+B | 半天 | 定位背景压力下耗时最高的负载类型（E7 主实验）+ 调用模式占比（E1 辅助） | 无 | ① |
| P0 | ③ 探活 GetObject → Stat（HEAD） | A | 半天 | 每成员省 5~15 次全量 GET 下载 | 低 | ①确认探活占比 |
| P1 | ④ 本地文件变更检测，跳过探活 | A | 1~2天 | 每成员调用 10~20 次 → 3~5 次（-60~70%） | 中低 | ② |
| P1 | ⑤ minio-go SDK 替换 mc 子进程 | A | 1~2天 | 单次 50~200ms → 5~20ms；消 exec/TLS 风暴 | 中 | ②量化后再改 |
| P1 | ⑥ 网络基线：**VPC 内网 endpoint**（`-internal`）+ 传输加速 | B | 半天~1天 | 公网 RTT/签名成本 → 内网（云 S3 场景第一优先，E6b 数据说话） | 低 | ②E6b |
| P2 | ⑦ 成员级并行（errgroup） | A | 2~3天 | team 内调谐 /N | 中 | ⑤（避免并发 exec 风暴）；②E4 |
| P2 | ⑧ 对象规模治理：孤儿清理 + 生命周期 | B | 1~2天 | 减缓"team 越多越慢"正反馈 | 低 | ②E2 证实规模效应 |
| P2 | ⑨ 云 OSS：QPS/带宽配额升配 + 限流适配 | B | 1天 | 提限流阈值，消 429/503 | 低（成本） | ②E8 证实限流 |
| P3 | ⑩ OSS 侧配置：存储类型检查（标准 vs 低频/归档）+ 传输加速 | B | 1天 | 降单请求成本/延迟 | 低 | ② |
| P3 | ⑪ SeaweedFS 自建迁移 | B | 2~4周 | 引擎级小对象优化（仅当限流/成本不可接受） | **极高**（自建替换云托管=降级） | ②E5 + 7.4 决策门 |

**结论倾向（云 S3 部署）**：当前证据（3m10s、云 OSS 单次 400~800ms+）指向"公网路径 + 调用模式 + QPS 限流"，而非引擎问题。**主战场是 A 轨（降调用次数/换 SDK/探活改 HEAD）+ ⑥切内网 endpoint**；SeaweedFS 自建是放弃云托管的降级动作，仅在限流/成本不可接受且 7.4 决策门全过时才考虑。

### 7.2 A 轨明细：控制器调用方式优化

**A1 探活改 HEAD（③）** — `deployer.go:479/263/362/862` 的 GetObject 仅用于判断存在性，内容直接丢弃。改为 `oss.Stat()`（`minio.go:119` 已有 `mc stat` 实现，缺失返回 `os.ErrNotExist` 语义一致）。收益：全量 GET 下载 → HEAD，省传输与 stdout 捕获。

**A2 减少调用次数（④）** — `seedLocalAgentFiles` 每轮调谐 WalkDir 全量探活。进程内缓存 `key → (本地 size, mtime)`：本地文件未变且远端已推送过 → 跳过探活；兜底：低频全量校验（每 30min 或 spec 变更时）防外部删除。

**A3 SDK 替换（⑤）** — 方案见 performance 文档 6.3.1。补充两点：
- 动态 STS 凭据模式（`buildMCHostEnv`，external-OSS 部署）：minio-go 需自定义 `credentials.Provider`（`Retrieve` + `IsExpired`），token 过期时重新 Resolve；
- 收益量化依赖 ②E1：若 sdk 与 mc 差异不大，说明瓶颈在服务端，⑤ 可降级。

**A4 成员级并行（⑦）** — 方案见 performance 文档 6.3.2（errgroup）。前置 ⑤ 的原因：并发 exec mc 会放大进程风暴，SDK 连接池下并发才安全。

**A5 变更驱动（引用）** — TeamStatus ObservedGeneration + Active 短路（performance 文档 6.2.1）落地后，稳态调谐完全不碰 OSS，是所有 A 轨优化的总开关。

### 7.3 B 轨明细：S3 基座优化（云 S3 视角）

**B1 网络基线（⑥，云 S3 场景第一优先）**
- **VPC 内网 endpoint**：OSS 同 region `-internal` endpoint（如 `oss-cn-hangzhou-internal.aliyuncs.com`）免费、免公网流量费且 RTT 低一个量级；前提是 controller 与 OSS 同 VPC/region，需打通 VPC 与 OSS 的授权关系
- 传输加速 endpoint：公网跨区场景可选
- 对照实验：E6b（公网 vs 内网各跑一次 E7）量化收益后再切，避免白改

**B2 对象规模治理（⑧）**
- 孤儿对象清理：已删除 worker/team 的残留（用 OSS 控制台/`mc ls --recursive` 统计，再定期清理任务）
- 生命周期：OSS 生命周期规则清理临时/过期对象
- 对象合并（低优先）：多小文件合并为 bundle 需改 agent 文件布局，风险高、收益中，除非 E2 证实规模是主因

**B3 限流与配额（⑨）** — OSS 控制台确认当前 QPS/带宽配额与限流事件（E8 数据）；升配或降调用频率（A2/A3 减少调用次数，比升配更持久）。限流特征：延迟拐点 + 429/503 + 控制台限流记录。

**B4 OSS 侧配置（⑩）** — 存储类型检查（标准 vs 低频/归档：低频访问读请求收费高且可能延迟高）、同 region 部署确认（bucket 与集群跨 region 是隐藏 RTT 源）、access log 开启（定位慢请求）。

### 7.4 SeaweedFS 迁移专项评估（⑪，云 S3 部署下为"自建替换云托管"）

**为什么会被考虑**：HiClaw 负载（每 worker 几十个小对象、对象数随 team 线性增长）正是 SeaweedFS 的设计目标（LOSF - lots of small files）。SeaweedFS 官方对 MinIO 小文件问题的表述（README 原文）：

> "MinIO did not have optimization for lots of small files. The files were simply stored as is to local disks. Plus the extra meta file and shards for erasure coding, it only amplifies the LOSF problem."
> "MinIO had multiple disk IO to read one file. SeaweedFS has O(1) disk reads, even for erasure coded files."

官方基准（README，1M 个 1KB 文件）：写 15,708 req/s，随机读 47,019 req/s；WARP 混合 550 obj/s。来源: https://github.com/seaweedfs/seaweedfs

**但当前底层是云 S3，迁移性质完全不同**：SeaweedFS 小对象优势是**自建引擎**层面的，换过去意味着：
- 放弃云托管（可用性 SLA、备份、容量弹性）→ 自运维 master + volume + filer；
- 数据出云迁移（`mc mirror` / rclone）+ 回滚方案；
- 云 OSS 的限流问题通过"自建"绕开，但引入了新的运维与故障面；
- **若瓶颈是公网路径/调用模式/限流（当前证据指向这些），换 SeaweedFS 无效**——先做 A 轨 + B1 内网，复测后再谈。

**兼容性风险（代码事实 + 待实测项）**：

| 面 | 当前实现 | SeaweedFS 兼容性 | 风险 |
|----|---------|-----------------|------|
| 数据面 | `mc cp/cat/stat/rm/ls/mb/mirror`（minio.go:96~199） | 对应 S3 PUT/GET/HEAD/DELETE/LIST/MakeBucket/COPY，核心 API 覆盖 | 低，但需 bench + 兼容用例实测（含 mirror 的 COPY 语义） |
| 管理面 | `mc admin user/policy create/attach/detach/remove`（minio_admin.go:50~105） | **`mc admin` 是 MinIO Admin API 专属，SeaweedFS 不实现** | **高**：用户/策略管理需整体重写 |
| 动态凭据 | `MC_HOST_<alias>` 带 STS token（`buildMCHostEnv`，阿里云 OSS STS） | SeaweedFS 无 STS 语义 | 中：凭据体系需重做（静态 AK 或自建认证） |
| 运维 | 云托管（SLA/备份/弹性） | master + volume + filer 自运维 | 高：运维模型完全变化 |
| 数据迁移 | 云 S3 | 数据出云 → 自建 | 中：迁移演练 + 回滚 |

**决策门（全部通过才迁移）**：
1. A 轨 + B1 内网落地后复测：单次调用仍慢且 ②E5（自建评估实验）显示 SeaweedFS 显著快（>2x）；
2. 限流/成本不可接受且升配无法解决（E8 + 账单数据）；
3. 兼容性实测清单通过：数据面 9 种 mc 操作逐一验证 + 管理面/凭据重写方案评审；
4. 迁移演练：数据量级确认、出云 mirror 时长、回滚路径。

**结论**：**不建议自建 SeaweedFS 替换云 S3**。当前证据（云 OSS 单次 400~800ms+）指向公网路径 + 调用模式 + 限流，A 轨 + ⑥切内网 endpoint 即可覆盖大部分收益；SeaweedFS 仅在限流/成本不可接受且决策门全过时才进入评估。

---

## 8. 后续行动建议

| 优先级 | 事项 | 工作量 |
|--------|------|--------|
| P0 | 部署带监测日志的版本（第 3 节），收集 step 4 各子阶段真实耗时数据 | 半天 |
| P0 | 用第 4.2 节 Step 2 写探针确认 CRD 字段问题归属，补齐 `reconcileAttempt`/`phaseTransitionTime` 到 helm CRD yaml | 10 分钟 |
| P0 | runMC 阈值日志（7.1 ①）确认单次 mc 延迟与 op 分布 | 半天 |
| P0 | bench_s3 跑 **E7 主实验**（7.1 ②）+ **E6b 公网 vs VPC 内网 endpoint 对照**，产出 writeSummary | 半天 |
| P1 | 依据数据决定：探活改 HEAD（A1）→ SDK 替换（A3） | 1~2 天 |
| P1 | **切 VPC 内网 endpoint**（B1，E6b 数据支持后）+ OSS 控制台限流/配额检查（E8） | 半天~1天 |
| P1 | pprof 构建开关落地（第 5 节），采样调谐热点验证 | 半天 |
| P2 | 减少调用次数（A2）、对象规模治理（B2） | 2~3 天 |
| P2 | 成员级并行（A4，前置 SDK）、`LegacyCompat.mu` 内存缓存 | 2~3 天 |
| P3 | SeaweedFS 自建评估（7.4 决策门）：仅限流/成本不可接受时启动，默认不做 | 数据驱动 |
