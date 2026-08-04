# Team Controller 缺陷改进文档

> 版本: v1.0
> 日期: 2026-08-05

---

## 1. 缺陷总览

| 编号 | 缺陷 | 严重度 | 引入提交 | 作者 |
|------|------|--------|----------|------|
| D-01 | HTTP API 绕过 Reconciler 直接操作 Backend + Status | 🔴 严重 | `7abe329` | ChestonGu |
| D-02 | failTeam 双重 Requeue（Result + Error 冲突） | 🔴 严重 | `7abe329` | ChestonGu |
| D-03 | Team Status 多次 Patch（非原子） | 🔴 严重 | `7abe329` | ChestonGu |
| D-04 | 无 Kubernetes Events 记录 | 🔴 严重 | `7abe329` | ChestonGu |
| D-05 | 无 Conditions（只有 Phase） | 🟡 中等 | `7abe329` | ChestonGu |
| D-06 | Healthz 始终返回 OK | 🟡 中等 | `7abe329` | ChestonGu |
| D-07 | Finalizer 更新无 Conflict 重试 | 🟡 中等 | `7abe329` | ChestonGu |
| D-08 | Reconcile 无 Context Deadline | 🟡 中等 | `7abe329` | ChestonGu |
| D-09 | Pod 无 SecurityContext | 🟡 中等 | `7abe329` | ChestonGu |
| D-10 | Phase 语义不一致（Running vs Active） | 🟡 低 | `7abe329` | ChestonGu |

> 所有缺陷均在项目初始提交 `7abe329`（PR #3, 2026-07-30）中引入，非后续变更引入。

---

## 2. 逐项缺陷详情与修改建议

### D-01: HTTP API 绕过 Reconciler 直接操作 Backend + Status

**引入提交**: `7abe329` (PR #3 from ChestonGu/feat/team-reconcile-timing-logs)
**文件**: `internal/server/lifecycle_handler.go:35`

**问题代码**:
```go
func (h *LifecycleHandler) Wake(w http.ResponseWriter, r *http.Request) {
    // 1. 直接改 Spec
    worker.Spec.State = &running
    h.k8s.Update(ctx, &worker)

    // 2. 直接操作 Docker/K8s backend
    b := h.registry.DetectWorkerBackend(ctx)
    b.Start(ctx, name)

    // 3. 直接写 Status
    worker.Status.Phase = "Running"
    h.k8s.Status().Update(ctx, &worker)
}
```

**问题**:
- Reconciler 是 K8s Controller 的唯一真相源，HTTP API 绕过它直接操作会导致状态不一致
- 并发写入：HTTP 写 Status + Reconciler 写 Status → 竞态条件
- Controller 重启后 Reconciler 看到的是 stale 的 Spec 变更，但 Status 已被 HTTP API 提前写入

**修改建议**:
```go
func (h *LifecycleHandler) Wake(w http.ResponseWriter, r *http.Request) {
    // 只改 Spec，由 Reconciler 负责 Status 和 Backend 操作
    running := "Running"
    worker.Spec.State = &running
    if err := h.k8s.Update(ctx, &worker); err != nil {
        writeK8sError(w, "update worker spec.state", err)
        return
    }
    // 返回 202 Accepted，由 Reconciler 异步处理
    httputil.WriteJSON(w, http.StatusAccepted, WorkerLifecycleResponse{
        Name:  name,
        Phase: worker.Status.Phase, // 返回当前 Status，不是预测值
    })
}
```

**影响范围**: `Wake`, `Sleep`, `EnsureReady` 三个端点均需修改。

---

### D-02: failTeam 双重 Requeue（Result + Error 冲突）

**引入提交**: `7abe329`
**文件**: `internal/controller/team_controller.go:745`

**问题代码**:
```go
func (r *TeamReconciler) failTeam(...) (reconcile.Result, error) {
    t.Status.Phase = "Failed"
    return reconcile.Result{RequeueAfter: reconcileRetryDelay}, fmt.Errorf("%s", msg)
    //                                         ^^^^^^^^^^^^^^^^     ^^^^^^^^^^^^^^^^^^^^
    //                                         Result.RequeueAfter  返回 Error 也会触发 requeue
}
```

**问题**: controller-runtime 同时应用 `RequeueAfter` 和 rate limiter 指数退避，预期的 30s 重试变成了 5ms → 10s → 100s → ... → 1000s 的不可预测行为。

**修改建议**:
```go
// 方案 A: 延迟重试，不触发 rate limiter
return reconcile.Result{RequeueAfter: reconcileRetryDelay}, nil

// 方案 B: 触发 rate limiter，不设固定间隔
return reconcile.Result{}, fmt.Errorf("%s", msg)
```

建议选方案 A，保留显式的退避控制。

---

### D-03: Team Status 多次 Patch（非原子）

**引入提交**: `7abe329`
**文件**: `internal/controller/team_controller.go:161`

**问题代码**:
```go
func (r *TeamReconciler) reconcileTeamNormal(...) {
    patchBase := client.MergeFrom(t.DeepCopy())

    if t.Status.Phase == "" {
        t.Status.Phase = "Pending"
        r.Status().Patch(ctx, t, patchBase)  // 第一次 patch
        patchBase = client.MergeFrom(t.DeepCopy())  // 重置!
    }
    // ... 100 行 ...
    r.Status().Patch(ctx, t, patchBase)  // 第二次 patch
    // ... 又 100 行 ...
    r.Status().Patch(ctx, t, patchBase)  // 第三次 patch
}
```

**问题**: 多次 patch 之间 `patchBase` 被重置，可能导致 patch 基线不一致；中间 patch 失败时后续 patch 基于 stale 数据。

**修改建议**: 统一用 defer 单次 patch，与 Manager/Human 保持一致：
```go
func (r *TeamReconciler) Reconcile(ctx context.Context, req reconcile.Request) (...) {
    // ...
    defer func() {
        if !team.DeletionTimestamp.IsZero() {
            return
        }
        t.Status.Phase = computeTeamPhase(&team, reterr)
        if err := r.Status().Patch(ctx, &team, patchBase); err != nil {
            logger.Error(err, "failed to patch team status")
        }
    }()
    // reconcileTeamNormal 只修改内存中的 status，不调用 Patch
}
```

---

### D-04: 无 Kubernetes Events 记录

**引入提交**: `7abe329`

**问题**: 整个 controller 未使用 `EventRecorder`。`kubectl describe team X` 不会显示任何事件，运维无法通过 K8s 原生工具排查问题。

**修改建议**:
```go
// team_controller.go - Reconciler 结构体增加
type TeamReconciler struct {
    // ... existing fields ...
    Recorder record.EventRecorder
}

// app.go - 初始化时注入
recorder := mgr.GetEventRecorderFor("hiclaw-team-controller")

// 关键节点记录事件
r.Recorder.Eventf(&team, corev1.EventTypeNormal, "PhaseTransition",
    "Team transitioned from %s to %s", prevPhase, t.Status.Phase)
r.Recorder.Eventf(&team, corev1.EventTypeWarning, "ReconcileFailed",
    "Reconcile failed: %s", msg)
```

---

### D-05: 无 Conditions

**引入提交**: `7abe329`
**文件**: `api/v1beta1/types.go:337`

**问题**: Phase 是单值字段，无法表达"Matrix 已就绪但容器未启动"等复合状态。

**修改建议**:
```go
type TeamStatus struct {
    // 保留 Phase 用于快速判断，增加 Conditions 用于详细状态
    Phase      string              `json:"phase,omitempty"`
    Conditions []metav1.Condition  `json:"conditions,omitempty"`
}

// Condition Types:
// - InfraReady: Matrix account + Gateway consumer provisioned
// - ConfigReady: OSS config (openclaw.json, SOUL.md) deployed
// - ContainerReady: All member pods Running
// - LeaderReady: Leader pod Running and healthy
```

---

### D-06: Healthz 始终返回 OK

**引入提交**: `7abe329`
**文件**: `internal/server/status_handler.go:23`

**问题代码**:
```go
func (h *StatusHandler) Healthz(w http.ResponseWriter, _ *http.Request) {
    w.WriteHeader(http.StatusOK)
    fmt.Fprint(w, "ok")
}
```

**问题**: K8s liveness probe 永远通过，即使控制器已卡死。

**修改建议**:
```go
func (h *StatusHandler) Healthz(w http.ResponseWriter, r *http.Request) {
    // 检查 leader election 状态、workqueue 深度等
    // 如果控制器卡死（如 rate limiter 退避到 1000s），返回 503
    if time.Since(h.lastReconcileTime) > 10*time.Minute {
        w.WriteHeader(http.StatusServiceUnavailable)
        fmt.Fprint(w, "stale: no reconcile in 10m")
        return
    }
    w.WriteHeader(http.StatusOK)
    fmt.Fprint(w, "ok")
}
```

---

### D-07: Finalizer 更新无 Conflict 重试

**引入提交**: `7abe329`
**文件**: `internal/controller/team_controller.go:109`

**问题代码**:
```go
if !controllerutil.ContainsFinalizer(&team, finalizerName) {
    controllerutil.AddFinalizer(&team, finalizerName)
    if err := r.Update(ctx, &team); err != nil {
        return reconcile.Result{}, err  // Conflict → 整个 reconcile 重来
    }
}
```

**修改建议**: 使用 `controllerutil.AddFinalizer`（内部处理 Conflict）:
```go
if err := controllerutil.AddFinalizer(ctx, &team, finalizerName); err != nil {
    return reconcile.Result{}, err
}
```

---

### D-08: Reconcile 无 Context Deadline

**引入提交**: `7abe329`
**文件**: `internal/controller/team_controller.go:84`

**问题**: 整个 reconcile 可能运行 10+ 分钟，无超时保护。Matrix API 慢响应会导致无限阻塞。

**修改建议**:
```go
func (r *TeamReconciler) Reconcile(ctx context.Context, req reconcile.Request) (...) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Minute)
    defer cancel()
    // ...
}
```

---

### D-09: Pod 无 SecurityContext

**引入提交**: `7abe329`
**文件**: `internal/backend/kubernetes.go:148`

**问题**: 创建的 Pod 无 SecurityContext，容器以 root 运行，不符合 Pod Security Standards。

**修改建议**:
```go
pod.Spec.SecurityContext = &corev1.PodSecurityContext{
    RunAsNonRoot: boolPtr(true),
    RunAsUser:    int64Ptr(1000),
    FSGroup:      int64Ptr(1000),
}
agentContainer.SecurityContext = &corev1.SecurityContext{
    ReadOnlyRootFilesystem: boolPtr(true),
    AllowPrivilegeEscalation: boolPtr(false),
    Capabilities: &corev1.Capabilities{
        Drop: []corev1.Capability{"ALL"},
    },
}
```

---

### D-10: Phase 语义不一致

**引入提交**: `7abe329`
**文件**: `api/v1beta1/types.go`

**问题**:
| CRD | Phase 值 |
|-----|----------|
| Worker | Pending/**Running**/Sleeping/Stopped/Failed |
| Team | Pending/**Active**/Degraded/Failed |
| Manager | Pending/**Active**/Failed |

同一项目内 Running vs Active 语义不一致。

**修改建议**: 统一为 `Running`，与 Worker 和 K8s 原生 Pod Phase 对齐。

---

## 3. 修复优先级

| 优先级 | 缺陷 | 工作量 |
|--------|------|--------|
| P0 立即修 | D-02 failTeam 双重 requeue | 1行 |
| P0 立即修 | D-07 Finalizer Conflict 重试 | 3行 |
| P1 尽快修 | D-04 注入 EventRecorder | 20行 |
| P1 尽快修 | D-06 Healthz 真实检查 | 10行 |
| P1 尽快修 | D-08 Context Deadline | 2行 |
| P2 计划修 | D-01 HTTP API 去绕过 | 50行 |
| P2 计划修 | D-03 Status 单次 Patch | 30行 |
| P2 计划修 | D-09 SecurityContext | 15行 |
| P3 评估后修 | D-05 Conditions | 100行 |
| P3 评估后修 | D-10 Phase 统一 | 需 API 迁移 |
