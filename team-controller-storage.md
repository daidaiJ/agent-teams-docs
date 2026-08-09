# Team Controller 沙箱存储选型：emptyDir vs Generic Ephemeral Volume

> 文档版本: v1.0
> 日期: 2026-08-09
> 状态: Draft

---

## 1. 背景与问题

### 1.1 场景

池化预热（Pool + BatchSandbox）模式下，Pod 被频繁创建与删除：

- Pool 预创建预热 Pod，BatchSandbox 按需分配
- 释放后 Pod 回池（或删除重建），下一个用户复用
- 诉求：**分配期间写入的业务数据，释放后不能留给下一个用户**

### 1.2 三个关键事实

| 事实 | 含义 |
|------|------|
| 删除 Pod **不会**回收 PVC | PVC 是独立资源，Pod 删除后数据原样保留 → **池模板挂共享 PVC 是反模式** |
| server 管理的 `pvc` 卷（`opensandbox.io/volume-managed-by=server`）生命周期跟随 sandbox | 天然隔离，但仅限 server API 创建路径，**不适用于池模板** |
| 容器可写层（overlayfs）随 Pod 删除清除；容器重启（Restart 回收）**不**清除 | 存储隔离只对 Delete 回收策略成立 |

### 1.3 结论先行

数据隔离的正解是**让卷的生命周期跟随 Pod**，而不是在回收路径上"补清理"：

- 共享 PVC + 清理 Hook / 清理 Job：不推荐（清理失败 = 数据泄漏，清理误伤 = 丢数据）
- **emptyDir / generic ephemeral volume：生命周期绑定 Pod，删除即自动回收，零运维**

---

## 2. 选型对比

| 维度 | emptyDir | Generic Ephemeral Volume | 共享 PVC + 清理 Hook | server `pvc` 卷模型 |
|------|----------|--------------------------|----------------------|---------------------|
| 生命周期 | 随 Pod，删除即清 | 随 Pod，GC 级联删 PVC | 独立于 Pod，靠 Hook 清 | 随 sandbox（server 托管） |
| 容量 | 无规格，`sizeLimit` 兜底 | 显式声明，StorageClass 保证，计入 PVC 配额 | 声明在 PVC | 声明在 PVC |
| 介质/性能 | 节点本地盘 / tmpfs | 可选 StorageClass（本地、云盘、NFS…） | 同左 | 同左 |
| 跨节点 | 数据绑节点 | 网络盘可跨节点（但卷随 Pod 删除） | 网络盘可跨节点 | 网络盘可跨节点 |
| 快照/备份 | 无 | 支持（StorageClass 支持时） | 支持 | 支持 |
| 可观测性 | 隐形目录 | 独立 PVC 对象，kubectl/配额/监控可见 | 独立 PVC 对象 | 独立 PVC 对象 |
| 创建延迟 | 毫秒级（mkdir） | 秒级（一次 PV 供给） | 秒级 | 秒级 |
| 存储成本 | 仅有数据时占空间 | 每 Pod 一块卷（预热 buffer × 卷大小） | 同左 | 每 sandbox 一块 |
| 数据抢救 | 无 | `reclaimPolicy=Retain` 可抢救孤儿 PV | 依赖 Hook 正确性 | 依赖 server 回收 |
| 运维复杂度 | 零 | 需动态供给 StorageClass | 高（Hook/Job 基建 + 失败策略） | 已有实现，走 server API |

---

## 3. 选型决策规则

```
只想要"Pod 删除自动清"，零依赖           → emptyDir
要容量限制 + 隔离，不想碰分布式存储       → local-path StorageClass + ephemeral
要快照/备份/跨节点/配额                  → 云盘/NFS 类 StorageClass + ephemeral
需要持久用户数据（跨 sandbox 存活）      → server `pvc` 卷模型（不是池模板该做的事）
```

**前提约束**：

- `Restart` 回收策略下，两者都**不清数据**（Pod 还在，可写层与卷都保留）——存储隔离只对 `Delete` 回收策略成立
- `Noop` 回收策略同理，且连进程态都不重置，仅适合业务自管理的场景

---

## 4. 配置样板

> 两种方式都直接写在 Pool 的 `spec.template` 中。CRD 的 Template 字段为 Schemaless 透传，**无需任何 CRD 改动**。

### 4.1 emptyDir（最简）

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: example-pool
spec:
  template:
    spec:
      containers:
      - name: sandbox-container
        image: ubuntu:latest
        command: ["sleep", "3600"]
        volumeMounts:
        - name: scratch
          mountPath: /scratch
      volumes:
      - name: scratch
        emptyDir: {}
  capacitySpec:
    bufferMax: 10
    bufferMin: 2
    poolMax: 20
    poolMin: 5
```

可选：`sizeLimit` 限制单卷上限（防单 Pod 撑爆节点盘），`medium: Memory` 用 tmpfs（删除即清、更快，占节点内存）：

```yaml
      volumes:
      - name: scratch
        emptyDir:
          sizeLimit: 10Gi
          # medium: Memory   # 内存盘模式
```

### 4.2 Generic Ephemeral Volume（推荐，容量可控）

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: example-pool
spec:
  template:
    spec:
      containers:
      - name: sandbox-container
        image: ubuntu:latest
        command: ["sleep", "3600"]
        volumeMounts:
        - name: scratch
          mountPath: /scratch
      volumes:
      - name: scratch
        ephemeral:
          volumeClaimTemplate:
            metadata:
              labels:
                app.kubernetes.io/component: pool-scratch
            spec:
              accessModes: [ReadWriteOnce]
              storageClassName: standard
              resources:
                requests:
                  storage: 10Gi
  capacitySpec:
    bufferMax: 10
    bufferMin: 2
    poolMax: 20
    poolMin: 5
```

字段说明：

| 字段 | 说明 |
|------|------|
| `storageClassName` | 省略则用集群默认 StorageClass；**必须支持动态供给**。`reclaimPolicy: Delete` = 删 PVC 连数据清（推荐）；`Retain` = 数据保留在 PV 可抢救（会产生孤儿 PV，需人工清理） |
| `accessModes: ReadWriteOnce` | 单副本 Pod 够用；跨节点共享读写才用 `ReadWriteMany`（需网络盘支持） |
| `resources.requests.storage` | 容量硬约束，超限供给失败；计入 PVC 配额 |
| `metadata.labels` | 建议打标签，方便观测池存储用量 |
| 不可设置 | PVC 的 `name`（自动生成）、ownerReference（自动指向 Pod） |

**Retain 抢救模式**（数据误删可恢复，代价是回收后 PV 变孤儿，需人工清理）：

```yaml
      volumes:
      - name: scratch
        ephemeral:
          volumeClaimTemplate:
            spec:
              storageClassName: scratch-retain   # StorageClass: reclaimPolicy: Retain
              resources:
                requests:
                  storage: 10Gi
```

**配额管控**（防池内单卷超规格）：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pool-storage-quota
spec:
  hard:
    requests.storage: 500Gi
    persistentvolumeclaims: "50"
```

### 4.3 自动产生的资源与验证

每个 Pod 创建时自动生成独立 PVC，命名 `<pod名>-<卷名>`，ownerReference 指向该 Pod；Pod 删除 → GC 级联删 PVC → `reclaimPolicy=Delete` 时底层数据一并清除。**回收链全自动，控制器零参与**。

```bash
# 确认每 Pod 一卷
kubectl get pvc -l app.kubernetes.io/component=pool-scratch

# 验证自动回收：删一个 Pool Pod
kubectl delete pod <pool-pod>
kubectl get pvc -l app.kubernetes.io/component=pool-scratch   # 对应 PVC 应消失
```

---

## 5. 注意事项

1. **StorageClass 的 `reclaimPolicy` 决定数据去向**：`Delete` 自动清数据；`Retain` 数据保留在孤儿 PV（可抢救，需人工回收）
2. **存储成本**：ephemeral 模式下预热 buffer 个 Pod 就占 buffer 块卷容量（PoolMax × 单卷大小），可用独立 "scratch" StorageClass 或小容量卷控制
3. **补位延迟**：ephemeral 补位多一步 PV 供给（秒级），emptyDir 无此步骤
4. **Restart 回收策略不适用**：任何"随 Pod 消失"的存储假设（可写层、emptyDir、ephemeral）在 Restart 下都不成立，需要额外清理机制
5. **节点盘容量**：emptyDir 受节点本地盘约束，多 Pod 挤同一节点盘；kubelet 清理目录是异步的，频繁建删时空间短暂占用

---

## 6. 结论

- **emptyDir**：零声明、零依赖、零供给延迟，是高频场景下"免费"的数据隔离方案；容量与观测能力弱
- **Generic Ephemeral Volume**：本质是"内嵌 PVC 声明 + 生命周期语法糖"（走 StorageClass/CSI 动态供给，非新卷类型），用固定 8 行声明换回容量可控、配额、快照、可观测性
- **两者都自动回收，控制器无需任何清理逻辑**；复杂度全部来自所选存储后端，而非声明本身
- 共享 PVC + 清理 Hook / 边车 / Job 均为补救方案，只在无法改变模板时考虑
