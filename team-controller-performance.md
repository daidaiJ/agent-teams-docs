# Team Controller 性能与扩展性问题定位及优化方案

> 文档版本: v1.0  
> 日期: 2026-08-04  
> 状态: Draft

---

## 1. 问题现象

### 1.1 调谐耗时异常

| 场景 | 期望耗时 | 实际耗时 |
|------|----------|----------|
| 已有 Team（成员已就绪）周期调谐 | <1s | 5min |
| 新建 Team CR 首次收敛 | <30s | 10min |
| 镜像更新后所有 Team 调谐 | 分钟级 | 小时级 |

### 1.2 调谐风暴

控制器镜像更新后：
- 所有 Team CR 同时触发调谐
- 每个 Team 的所有成员容器被 Delete + Create
- Pod phase 变化（Running → Terminating → Pending → Running）再次触发入队
- 新建的 Team CR 被阻塞在队列中，长时间无 Phase

### 1.3 Failed Team 抢占队列

- Failed Team 每 30s 重试，无退避
- 持续抢占 workqueue 位置
- 正常 Team 和新建 Team 被挤到后面

---

## 2. 根因分析

### 2.1 并发模型：单线程瓶颈

**文件**: `team_controller.go:1139`

```go
func (r *TeamReconciler) SetupWithManager(mgr ctrl.Manager) error {
    bldr := ctrl.NewControllerManagedBy(mgr).For(&v1beta1.Team{})
    // ... 没有设置 MaxConcurrentReconciles
    return bldr.Complete(r)
}
```

controller-runtime 默认 `MaxConcurrentReconciles = 1`，所有 Team CR 串行调谐。一个 Team 卡 10min，其余全部排队。

### 2.2 TeamStatus 缺少 ObservedGeneration

**文件**: `types.go:337`

```go
type TeamStatus struct {
    Phase          string `json:"phase,omitempty"`
    // ... 没有 ObservedGeneration 字段
}
```

对比：
- `WorkerStatus`（line 182）：有 `ObservedGeneration`
- `ManagerStatus`（line 547）：有 `ObservedGeneration`
- `TeamStatus`（line 337）：**缺失**

**后果**：控制器重启后 informer re-sync 触发全量调谐，无法区分"spec 真的变了"还是"List 入队"。

### 2.3 Active Team 无短路逻辑

**文件**: `team_controller.go:161` (`reconcileTeamNormal`)

即使 Team 已经 Active 且 spec 未变化，每次 5min 周期调谐仍执行全量链路：

```
resolveTeamAdminActor → ProvisionTeamRooms → EnsureTeamStorage
→ InjectCoordinationContext → reconcileMember × N（Infra + Config + Container）
```

没有任何"已就绪则跳过"的短路。

### 2.4 Failed/Degraded Team 无退避、无上限

**文件**: `team_controller.go:731` (`failTeam`)

```go
func (r *TeamReconciler) failTeam(...) (reconcile.Result, error) {
    t.Status.Phase = "Failed"
    return reconcile.Result{RequeueAfter: reconcileRetryDelay}, fmt.Errorf(...)
    // reconcileRetryDelay = 30s，永远 30s，无退避
}
```

两条失败路径：
- `failTeam`（4 个触发点）：validateTeamRuntimeNames / resolveTeamAdminActor / ProvisionTeamRooms / writeInlineConfigs
- `perMemberErrors > 0`（line 414）：`requeue = reconcileRetryDelay`（30s）

### 2.5 镜像更新触发全量重建风暴

**文件**: `team_controller.go:959` (`hashMemberSourceSpec`)

```go
// 对整个 TeamWorkerSpec 做 JSON hash
payload = workerInput{Worker: ws, TeamPolicy: ..., PeerMentions: ...}
buf, _ := json.Marshal(payload)
```

`TeamWorkerSpec` 包含 `Image` 字段。镜像一变 → 所有 Team 的所有 worker 成员 hash 全部变化 → 触发容器 Delete + Create。

Pod phase 变化触发 `podLifecyclePredicates`（line 376）→ 再次入队，形成级联。

### 2.6 MinIO 操作使用 mc 子进程

**文件**: `minio.go:204`

```go
func (c *MinIOClient) runMC(ctx context.Context, args ...string) (string, error) {
    cmd := exec.CommandContext(ctx, c.config.MCBinary, args...)
    // 每次调用 fork 子进程 + TLS 握手 + S3 API
}
```

每次 `mc` 调用固定开销 50~200ms。`DeployWorkerConfig` 一个成员需要 15~25 次 mc 调用。

### 2.7 Leader Election 限制水平扩展

**文件**: `app.go:654`

```go
opts := ctrl.Options{
    LeaderElection:    true,
    LeaderElectionID:  leaseID,
}
```

所有 4 个 controller 共享同一个 Manager → 共享同一个 leader lease → 只有一个 Pod 执行 reconcile。水平扩容 Pod 不能增加调谐并发。

---

## 3. 调谐耗时深度拆解

### 3.1 单次 Team Reconcile 外部调用链（3-worker Team）

| 阶段 | 外部调用 | 调用数 | 单次延迟 | 阶段耗时 |
|------|----------|--------|----------|----------|
| **resolveTeamAdminActor** | | | | |
| → Get Human CR | K8s API | 1 | ~5ms | 5ms |
| → LoginAsHuman | Matrix HTTP | 1 | ~200ms | 200ms |
| **ProvisionTeamRooms** | | | | |
| → CreateRoom (team) | Matrix HTTP | 1 | ~200ms | 200ms |
| → JoinRoom (admin) | Matrix HTTP | 1 | ~150ms | 150ms |
| → ListRoomMembers | Matrix HTTP | 1 | ~150ms | 150ms |
| → InviteToRoom × N | Matrix HTTP | 4~6 | ~150ms each | 600ms |
| → CreateRoom (leader DM) | Matrix HTTP | 1 | ~200ms | 200ms |
| → ListRoomMembers + Invite | Matrix HTTP | 2~3 | ~150ms | 350ms |
| **EnsureTeamStorage** | mc fork | 2~4 | ~80ms | 200ms |
| **InjectCoordinationContext** | mc fork | 3~4 | ~80ms | 300ms |
| **per member × 4** | | | | |
| → ReconcileMemberInfra | K8s Secret + Matrix HTTP | 2 | | 220ms |
| → EnsureServiceAccount | K8s API | 1 | ~10ms | 10ms |
| → ReconcileMemberConfig | mc fork × 15~25 | 15~25 | ~80ms | 1.5~2s |
| → ReconcileMemberContainer | K8s API + Pod 等待 | 1~3 | | 100ms + 等待 |
| → ReconcileMemberExpose | Higress HTTP | 1 | ~200ms | 200ms |
| **per member 小计** | | | | **~2~2.5s** |
| **4 members 总计** | | | | **~8~10s** |
| **Legacy registry** | mc fork × 2 | 2 | ~80ms | 160ms |
| **summarizeBackendReadiness** | K8s/wb.Status × 4 | 4 | ~10ms | 40ms |
| **总计（不含容器就绪等待）** | | | | **~10~15s** |

### 3.2 容器就绪等待（额外开销）

- 镜像拉取：10s ~ 数分钟
- Pod Readiness Probe：10~60s
- Agent 首次启动 / sync：10~30s

**已存在成员**（IsUpdate=true）：跳过容器创建，总计 ~5min（主要是 mc fork + Matrix HTTP 累积延迟 + 5min requeue 周期）

**新建成员**（IsUpdate=false）：需要容器创建 + 等待就绪，总计 ~10min

### 3.3 耗时占比

| 排名 | 瓶颈 | 单次开销 | 每 Team 总次数 | 占比 |
|------|------|----------|----------------|------|
| 1 | 容器就绪等待 | 10~120s | 4 | ~80% |
| 2 | mc 子进程 fork（MinIO） | 50~100ms | 60~100 | ~12% |
| 3 | Matrix HTTP | 100~300ms | 15~20 | ~6% |
| 4 | K8s API | 5~10ms | 15~20 | ~1% |
| 5 | Higress HTTP | 100~300ms | 4 | ~1% |

---

## 4. 滚动升级全量调谐问题

### 4.1 触发链路

```
控制器 Pod 滚动重启
  → controller-manager 启动
  → informer 从 apiserver List 全部 Team CR
  → 对每个 CR 触发 Add 事件 → 入队
  → Reconcile 被调用（informer 机制决定，不可阻止）
  → 没有 ObservedGeneration → 无法判断 spec 是否变化
  → 无条件执行全量链路
  → 100 个 Active Team × 5min/个 = 8.3h（并发=1）
```

### 4.2 ObservedGeneration 的作用

`Generation` 由 apiserver 自动管理，`ObservedGeneration` 由控制器在成功 reconcile 后写入：

```
用户修改 spec → apiserver 递增 Generation(2→3)
控制器成功 reconcile → 写入 ObservedGeneration=3
控制器重启 → Generation(3) == ObservedGeneration(3) → spec 没变，跳过
```

**本质**：区分"spec 真的被修改了"还是"informer re-sync 导致的入队"。

**当前状态**：`WorkerStatus` 和 `ManagerStatus` 已有此字段，`TeamStatus` 缺失。

---

## 5. Failed Team 抢占队列量化

假设 5 个 Team 因 admin 配置错误持续 Failed：

```
5 teams × (60s / 30s requeue) = 10 次/分钟 入队
每个 failTeam 调谐耗时 ~2-5s
→ 每分钟占用队列 20-50s

队列时间分配:
  Failed:  ~83%
  Active:  ~17%
  新建:    0%（永远排不上）
```

---

## 6. 优化方案

> **排序原则**：按实施优先级排序，优先选择**低风险、低复杂度、长期维护收益高**的改动。
> - 风险：是否改 CRD API？是否改 reconcile 核心路径？是否引入新依赖？
> - 维护收益：改动是否建立长期 pattern？是否减少未来同类问题的概率？

### 6.1 第一批：零风险，立即可做（改常量/配置，不改 API）

#### 6.1.1 提升并发（预期收益：吞吐量 5x）

**改动**: `team_controller.go` — `SetupWithManager`

```go
func (r *TeamReconciler) SetupWithManager(mgr ctrl.Manager) error {
    c, err := controller.New("team-controller", mgr, controller.Options{
        Reconciler:              r,
        MaxConcurrentReconciles: 5,
    })
    if err != nil {
        return err
    }
    // 手动注册 watches ...
}
```

| 维度 | 评估 |
|------|------|
| 风险 | 低。不改 CRD API，不改 reconcile 逻辑，只改并发度 |
| 维护收益 | 高。为所有后续优化建立并发基础，未来加更多 team 不再线性增长 |
| 并发安全 | 已确认：LegacyCompat 有 sync.Mutex，WriteInlineConfigs 写独立子目录 |
| 回滚方案 | 改回1即可 |

#### 6.1.2 Active Team requeue 延长到 15~30min（预期收益：减少 60~80% 无意义调谐）

**改动**: `worker_controller.go:27` + `team_controller.go:412`

```go
const (
    reconcileInterval       = 15 * time.Minute  // 5min → 15min
    reconcileRetryDelay     = 30 * time.Second
    reconcileActiveInterval = 30 * time.Minute  // 新增：Active Team 专用
)
```

```go
// team_controller.go:412 - 成功路径
requeue := reconcileInterval
if t.Status.Phase == "Active" && t.Generation == t.Status.ObservedGeneration {
    requeue = reconcileActiveInterval  // Active + 无变更 → 30min
}
if len(perMemberErrors) > 0 {
    requeue = reconcileRetryDelay
}
```

| 维度 | 评估 |
|------|------|
| 风险 | 极低。只改 requeue 时间，不改 reconcile 逻辑 |
| 维护收益 | 中。减少队列负载，但不解决根因（ObservedGeneration 短路才是根因） |
| 回滚方案 | 改回原值 |

#### 6.1.3 Manager 同步加并发（保持一致性）

**改动**: `manager_controller.go` — `SetupWithManager`

```go
// 同 TeamReconciler 一样设置 MaxConcurrentReconciles: 3
```

| 维度 | 评估 |
|------|------|
| 风险 | 低。ManagerReconciler 已有 Generation 短路，并发安全 |
| 维护收益 | 中。保持 Manager 和 Team 一致的并发模式 |

---

### 6.2 第二批：低风险，改 CRD API（只加字段，不改现有字段）

#### 6.2.1 TeamStatus 加 ObservedGeneration + Active 短路（预期收益：重启恢复 8.3h → 2s）

**改动 1**: `types.go` — TeamStatus 新增 3 个字段

```go
type TeamStatus struct {
    // ... 现有字段不变 ...

    // ObservedGeneration is the most recent generation observed by the
    // controller. Used to detect spec changes and short-circuit
    // expensive reconcile passes for unchanged Active teams.
    // Mirrors WorkerStatus.ObservedGeneration and ManagerStatus.ObservedGeneration.
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // ConsecutiveFailures tracks consecutive reconcile failures for
    // exponential backoff. Reset to 0 on any successful pass.
    ConsecutiveFailures int `json:"consecutiveFailures,omitempty"`

    // MaxRetriesReached stops automatic requeuing after maxTeamRetries.
    // Reset when user sets annotation hiclaw.io/retry on the Team CR.
    MaxRetriesReached bool `json:"maxRetriesReached,omitempty"`
}
```

**改动 2**: `team_controller.go` — reconcileTeamNormal 入口加短路

```go
func (r *TeamReconciler) reconcileTeamNormal(ctx context.Context, t *v1beta1.Team) (reconcile.Result, error) {
    logger := log.FromContext(ctx)
    passStart := time.Now()
    patchBase := client.MergeFrom(t.DeepCopy())

    // --- Active + spec 未变化 → 快速路径 ---
    if t.Status.Phase == "Active" && t.Generation == t.Status.ObservedGeneration {
        desiredMembers := buildDesiredMembers(t, r.ControllerName)
        leaderReady, readyWorkers := r.summarizeBackendReadiness(ctx, t, desiredMembers)

        if leaderReady && readyWorkers == t.Status.TotalWorkers {
            logger.Info("team healthy, skipping full reconcile",
                "team", t.Name, "passDuration", time.Since(passStart))
            return reconcile.Result{RequeueAfter: reconcileActiveInterval}, nil
        }
        // 有成员不健康 → 走全量 reconcile 恢复
    }

    // --- 原有逻辑 ---
    // ...

    // 成功路径末尾写入
    t.Status.ObservedGeneration = t.Generation
}
```

| 维度 | 评估 |
|------|------|
| 风险 | 低。只加新字段（`omitempty`），不影响现有 CR；短路逻辑仅在 `Phase==Active && Generation==ObservedGeneration` 时生效 |
| 维护收益 | 极高。建立 Team 和 Worker/Manager 一致的 spec-change 检测 pattern，消除滚动升级全量调谐问题 |
| 向后兼容 | 新字段 `omitempty`，旧版本控制器忽略未知字段；新字段只有在控制器成功 reconcile 后才被写入 |
| 回滚方案 | 删除 ObservedGeneration 赋值逻辑，新字段自然不被写入 |

#### 6.2.2 Failed 指数退避 + maxRetry（预期收益：Failed 不再抢占队列）

**改动**: `team_controller.go` — failTeam + Reconcile 入口

```go
const maxTeamRetries = 20

func (r *TeamReconciler) failTeam(ctx context.Context, t *v1beta1.Team,
    patchBase client.Patch, msg string) (reconcile.Result, error) {

    t.Status.Phase = "Failed"
    t.Status.Message = msg
    t.Status.ConsecutiveFailures++
    now := metav1.Now()
    t.Status.PhaseTransitionTime = &now

    if t.Status.ConsecutiveFailures > maxTeamRetries {
        t.Status.MaxRetriesReached = true
        r.Status().Patch(ctx, t, patchBase)
        return reconcile.Result{}, nil
    }

    delay := reconcileRetryDelay * time.Duration(1<<(t.Status.ConsecutiveFailures-1))
    if delay > 10*time.Minute {
        delay = 10 * time.Minute
    }

    r.Status().Patch(ctx, t, patchBase)
    return reconcile.Result{RequeueAfter: delay}, fmt.Errorf("%s", msg)
}
```

Reconcile 入口守卫：

```go
func (r *TeamReconciler) Reconcile(ctx context.Context, req reconcile.Request) (...) {
    // ... Get team ...

    if team.Status.MaxRetriesReached {
        if team.Annotations["hiclaw.io/retry"] == "" {
            return reconcile.Result{}, nil
        }
        delete(team.Annotations, "hiclaw.io/retry")
        team.Status.MaxRetriesReached = false
        team.Status.ConsecutiveFailures = 0
        r.Update(ctx, &team)
    }
    // ...
}
```

退避时间表：

| 失败次数 | 退避间隔 | 累计时间 |
|----------|----------|----------|
| 1 | 30s | 30s |
| 2 | 60s | 1.5min |
| 3 | 2min | 3.5min |
| 4 | 4min | 7.5min |
| 5 | 8min | 15.5min |
| 6~20 | 10min (cap) | 最多 ~2.5h |
| >20 | 停止重试 | 等用户 `kubectl annotate team X hiclaw.io/retry=""` |

| 维度 | 评估 |
|------|------|
| 风险 | 低。只改 requeue 策略和计数器，不改 reconcile 核心路径 |
| 维护收益 | 高。建立 Failed CR 管理 pattern，防止类似问题再次发生 |
| 向后兼容 | ConsecutiveFailures/MaxRetriesReached 为 `omitempty`，旧版本忽略 |
| 回滚方案 | 重置 ConsecutiveFailures=0, MaxRetriesReached=false，恢复原 requeue 行为 |

---

### 6.3 第三批：中风险，需要改 reconcile 核心路径

#### 6.3.1 MinIO mc → SDK（预期收益：per member 省 1~2s）

**现状**: 每次 `mc` 调用 fork 子进程，固定开销 50~200ms，per member 15~25 次

**方案**: 使用 MinIO Go SDK (`github.com/minio/minio-go/v7`) 替代 `exec.CommandContext`

```go
// oss/minio_sdk.go - 新文件，替代 minio.go
type MinIOSDKClient struct {
    client *minio.Client
    bucket string
    prefix string
}

func (c *MinIOSDKClient) PutObject(ctx context.Context, key string, data []byte) error {
    reader := bytes.NewReader(data)
    _, err := c.client.PutObject(ctx, c.bucket, c.fullPath(key), reader,
        int64(len(data)), minio.PutObjectOptions{})
    return err
}
```

| 维度 | 评估 |
|------|------|
| 风险 | 中。需要替换 OSS 接口实现，涉及 mc alias 配置 → SDK client 初始化 |
| 维护收益 | 高。消除 subprocess fork 瓶颈，减少容器镜像对 mc 二进制的依赖 |
| 新依赖 | `github.com/minio/minio-go/v7`（标准 MinIO SDK，稳定） |
| 回滚方案 | 保留原 minio.go 实现，通过 build tag 切换 |

#### 6.3.2 Team 内成员并行化（预期收益：Team 内调谐时间 /N）

```go
// team_controller.go - Step 4
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(ctx)

// leader 串行
if err := r.reconcileMember(ctx, deps, leader, leaderMS); err != nil {
    perMemberErrors = append(perMemberErrors, ...)
}

// workers 并行
for i := range workerMembers {
    i := i
    g.Go(func() error {
        ms := memberStatus(&t.Status, workerMembers[i].Name, workerMembers[i].Role)
        if err := r.reconcileMember(ctx, deps, workerMembers[i], ms); err != nil {
            return fmt.Errorf("%s: %w", workerMembers[i].Name, err)
        }
        return nil
    })
}
g.Wait()
```

| 维度 | 评估 |
|------|------|
| 风险 | 中。需要处理 errgroup 并发错误收集、memberStatus 指针并发安全 |
| 维护收益 | 中。收益取决于 team 内 worker 数量，worker 少的 team 收益有限 |
| 注意 | `memberStatus` 返回的指针在并行场景下需使用独立 slice |

---

### 6.4 第四批：高风险，需要架构变更

#### 6.4.1 双 Controller 分离新建与存量（预期收益：新建不被 Active 阻塞）

```go
// app.go - initReconcilers
// fast controller: Phase="" 和 Pending
ctrl.NewControllerManagedBy(a.mgr).
    For(&v1beta1.Team{}, builder.WithPredicates(predicate.NewPredicateFuncs(
        func(obj client.Object) bool {
            t := obj.(*v1beta1.Team)
            return t.Status.Phase == "" || t.Status.Phase == "Pending"
        },
    ))).
    WithOptions(controller.Options{MaxConcurrentReconciles: 3}).
    Complete(fastReconciler)

// steady controller: Active + Degraded + Failed
ctrl.NewControllerManagedBy(a.mgr).
    For(&v1beta1.Team{}, builder.WithPredicates(predicate.NewPredicateFuncs(
        func(obj client.Object) bool {
            t := obj.(*v1beta1.Team)
            return t.Status.Phase != "" && t.Status.Phase != "Pending"
        },
    ))).
    WithOptions(controller.Options{MaxConcurrentReconciles: 2}).
    Complete(steadyReconciler)
```

| 维度 | 评估 |
|------|------|
| 风险 | 高。两个 controller 共享同一个 Reconciler 实例，需要确保无状态；Phase 在 reconcile 中间可能变化导致两个 controller 争抢 |
| 维护收益 | 中。引入架构复杂度，两个 controller 的生命周期管理、配置同步是长期负担 |
| 替代方案 | 如果 P0（并发 + ObservedGeneration）已实施，新建 Team 的等待时间已大幅缩短，双 controller 的边际收益递减 |

#### 6.4.2 镜像变更分离（预期收益：镜像更新不触发全量 Infra/Config 重跑）

```go
// 区分 "容器级变更" 和 "配置级变更"
func memberConfigChanged(t *v1beta1.Team, role MemberRole, name string) bool {
    // 只对比 config 相关字段（排除 Image/Runtime）
}
```

| 维度 | 评估 |
|------|------|
| 风险 | 高。需要修改 hashMemberSourceSpec 逻辑，影响所有现有 Team 的 hash 计算 |
| 维护收益 | 中。镜像更新频率通常不高，实际收益有限 |
| 建议 | 如果 P1（MinIO SDK + 成员并行）已实施，镜像更新耗时已大幅缩短，此项优先级可进一步降低 |

---

## 7. 优化效果预估

### 7.1 滚动升级场景（100 个 Active Team）

| 阶段 | 方案 | 耗时 | 相对收益 |
|------|------|------|----------|
| 当前 | — | ~8.3h | 1x |
| + 第一批 | 并发=5 + requeue 延长 | ~1.7h | 5x |
| + 第二批 | ObservedGeneration 短路 | ~2s | 15000x |

### 7.2 新建 Team 场景

| 阶段 | 方案 | 耗时 | 相对收益 |
|------|------|------|----------|
| 当前 | — | 无限等待 | — |
| + 第一批 | 并发=5 | <5min | — |
| + 第二批 | Failed 退避 + maxRetry | <1min | — |
| + 第四批 | 双 controller 分离 | <30s | — |

### 7.3 镜像更新场景（10 Team, 3 workers each）

| 阶段 | 方案 | 耗时 | 相对收益 |
|------|------|------|----------|
| 当前 | — | ~50min | 1x |
| + 第一批 | 并发=5 | ~10min | 5x |
| + 第三批 | 成员并行 + MinIO SDK | ~1min | 50x |
| + 第四批 | 镜像分离 | ~45s | 66x |

---

## 8. 实施路线图

```
第一批（零风险，1天内可完成）
├── MaxConcurrentReconciles = 5          ← 改配置
├── Active Team requeue 延长到 30min      ← 改常量
└── Manager 并发对齐                      ← 改配置

第二批（低风险，2-3天，需 CRD 迁移）
├── TeamStatus 加 ObservedGeneration     ← 加字段 + 短路逻辑
├── Failed 指数退避 + maxRetry            ← 加字段 + 退避逻辑
└── CRD manifests 更新 + 升级验证

第三批（中风险，1周，需 OSS 层重构）
├── MinIO mc → SDK                       ← 替换 OSS 实现
└── Team 内成员并行化                     ← 改 reconcile 核心路径

第四批（高风险，需评估收益后决定）
├── 双 Controller 分离                    ← 架构变更
└── 镜像变更分离                          ← hash 逻辑变更
```

**建议**：先完成第一批 + 第二批，覆盖 90% 的性能问题。第三批和第四批根据实际效果决定是否需要。
