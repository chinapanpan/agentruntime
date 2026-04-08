# Sandbox 镜像升级策略

> 适用于 EKS + Kata-CLH + OpenSandbox Pool 环境
> 目标: 全量升级预热池和运行中的 Sandbox，最小化服务中断

---

## 1. 升级原理

### OpenSandbox Pool Revision 机制

OpenSandbox Controller 通过 **revision hash** 追踪 Pool 模板版本:

```
spec.template 变更 → SHA256(template) → status.revision 更新
                                       → 新 Pod label: sandbox.opensandbox.io/pool-revision
```

当 `spec.template` 被修改（如更换镜像）时:

| Pod 状态 | 行为 |
|---------|------|
| **空闲 (Available)** | 自动删除旧 revision Pod，创建新 revision Pod |
| **已分配 (Allocated)** | **不受影响** — 被 BatchSandbox 保护，直到释放 |

### Kata-CLH / Firecracker 限制

- **无 live migration**: 所有 Kata VMM (CLH / QEMU / Firecracker) 都不支持在线迁移
- **Firecracker snapshot/restore 不能换镜像**: restore 恢复的是同版本 VM，无法用于镜像升级
- **重建 = 新 microVM**: 镜像变更必须重建 Pod，进程状态丢失
- **回填时间**: 新 Pod 冷启动约 2.8-3.4s

### 关键结论

> 空闲 Pod 可无缝升级（内置机制）。已分配 Pod 无法 VM 级热更新，但可以通过 **状态外置 + 任务边界切换** 实现接近零中断的升级。

---

## 2. 环境变量

所有策略通用:

```bash
export NS=opensandbox-system
export POOL_NAME=kata-clh-pool
export OLD_IMAGE=busybox:latest
export NEW_IMAGE=busybox:1.37        # 替换为实际新镜像
```

---

## 3. Strategy 1: 标准滚动更新

**适用场景**: 常规镜像更新，已分配 Sandbox 可等待自然释放后逐步切换。

**影响范围**: 仅空闲 Pool Pod，已分配 Pod 不受影响。

### 执行步骤

**Step 1: 记录当前状态**

```bash
echo "=== 当前 Pool 状态 ==="
kubectl -n $NS get pool $POOL_NAME
echo ""
echo "=== 当前 Revision ==="
kubectl -n $NS get pool $POOL_NAME -o jsonpath='{.status.revision}' && echo
echo ""
echo "=== Pod Revision 分布 ==="
kubectl -n $NS get pods -l sandbox.opensandbox.io/pool-name=$POOL_NAME \
  -o custom-columns='NAME:.metadata.name,REVISION:.metadata.labels.sandbox\.opensandbox\.io/pool-revision,PHASE:.status.phase'
```

**Step 2: 更新 Pool 镜像**

```bash
kubectl -n $NS patch pool $POOL_NAME --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "'$NEW_IMAGE'"}
]'
```

**Step 3: 监控滚动更新**

```bash
# 持续观察，直到所有空闲 Pod 都更新为新 revision
NEW_REV=$(kubectl -n $NS get pool $POOL_NAME -o jsonpath='{.status.revision}')
echo "Target revision: $NEW_REV"

for i in $(seq 1 60); do
  AVAIL=$(kubectl -n $NS get pool $POOL_NAME -o jsonpath='{.status.available}')
  OLD_COUNT=$(kubectl -n $NS get pods \
    -l sandbox.opensandbox.io/pool-name=$POOL_NAME,sandbox.opensandbox.io/pool-revision!=$NEW_REV \
    --no-headers 2>/dev/null | wc -l)
  ALLOC=$(kubectl -n $NS get pool $POOL_NAME -o jsonpath='{.status.allocated}')

  echo "[$i] available=$AVAIL, old_revision_pods=$OLD_COUNT (其中 allocated=$ALLOC)"

  if [ "$OLD_COUNT" = "0" ] || [ "$OLD_COUNT" = "$ALLOC" ]; then
    echo "滚动更新完成! 所有空闲 Pod 已更新到新 revision"
    break
  fi
  sleep 5
done
```

**Step 4: 验证**

```bash
echo "=== Pool 状态 ==="
kubectl -n $NS get pool $POOL_NAME

echo ""
echo "=== 新 Claim 验证 ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: upgrade-verify
  namespace: opensandbox-system
spec:
  replicas: 1
  poolRef: kata-clh-pool
EOF

for i in $(seq 1 30); do
  READY=$(kubectl -n $NS get batchsandbox upgrade-verify -o jsonpath='{.status.ready}' 2>/dev/null)
  [ "$READY" = "1" ] && echo "新 Claim 成功!" && break
  sleep 1
done

kubectl -n $NS delete batchsandbox upgrade-verify
```

### 回滚

```bash
# 恢复旧镜像
kubectl -n $NS patch pool $POOL_NAME --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "'$OLD_IMAGE'"}
]'
# Controller 会再次替换空闲 Pod 回旧版本
```

---

## 4. Strategy 2: Blue-Green Pool 升级

**适用场景**: 生产关键升级，需要零停机、完整验证后再切换。

**核心思路**: 新旧 Pool 并行运行，验证无误后切换流量，最后排空旧 Pool。

```
时间线:
├── T0: 创建新 Pool (v2)
├── T1: v2 预热完成，smoke test 通过
├── T2: 新 Claim 指向 v2                    ← 切换点
├── T3: 旧 Pool 上 BatchSandbox 自然释放
└── T4: 旧 Pool allocated=0，删除旧 Pool
```

### 执行步骤

**Step 1: 创建新 Pool**

```bash
export NEW_POOL=kata-clh-pool-v2

cat <<EOF | kubectl apply -f -
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: $NEW_POOL
  namespace: $NS
spec:
  template:
    spec:
      runtimeClassName: kata-clh
      containers:
      - name: sandbox
        image: $NEW_IMAGE
        command: ["sh", "-c", "while true; do sleep 3600; done"]
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
  capacitySpec:
    bufferMin: 5
    bufferMax: 10
    poolMin: 5
    poolMax: 50
EOF
```

**Step 2: 等待新 Pool 预热**

```bash
echo "等待新 Pool 预热..."
for i in $(seq 1 120); do
  AVAIL=$(kubectl -n $NS get pool $NEW_POOL -o jsonpath='{.status.available}' 2>/dev/null)
  if [ "${AVAIL:-0}" -ge 5 ] 2>/dev/null; then
    echo "新 Pool 就绪: available=$AVAIL"
    break
  fi
  echo "  [$i] available=${AVAIL:-0}"
  sleep 5
done

echo ""
echo "=== 双 Pool 状态 ==="
kubectl -n $NS get pool
```

**Step 3: Smoke Test 新 Pool**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: v2-smoke
  namespace: $NS
spec:
  replicas: 1
  poolRef: $NEW_POOL
EOF

for i in $(seq 1 30); do
  READY=$(kubectl -n $NS get batchsandbox v2-smoke -o jsonpath='{.status.ready}' 2>/dev/null)
  [ "$READY" = "1" ] && echo "Smoke test 通过!" && break
  sleep 1
done

kubectl -n $NS delete batchsandbox v2-smoke
```

**Step 4: 切换新 Claim 到新 Pool**

从此刻起，所有新建的 BatchSandbox 使用 `poolRef: kata-clh-pool-v2`。

> 具体切换方式取决于你的 AI Agent 服务如何创建 BatchSandbox:
> - ConfigMap / 环境变量中的 pool name
> - API 网关配置
> - 代码中的硬编码

**Step 5: 监控旧 Pool 排空**

```bash
echo "等待旧 Pool 排空..."
for i in $(seq 1 360); do  # 最多等 1 小时
  OLD_ALLOC=$(kubectl -n $NS get pool $POOL_NAME -o jsonpath='{.status.allocated}' 2>/dev/null)
  echo "  [${i}0s] 旧 Pool allocated=$OLD_ALLOC"

  if [ "${OLD_ALLOC:-1}" = "0" ]; then
    echo "旧 Pool 已排空!"
    break
  fi
  sleep 10
done
```

**Step 6: 删除旧 Pool**

```bash
OLD_ALLOC=$(kubectl -n $NS get pool $POOL_NAME -o jsonpath='{.status.allocated}')
if [ "$OLD_ALLOC" = "0" ]; then
  # 先缩容
  kubectl -n $NS patch pool $POOL_NAME --type merge \
    -p '{"spec":{"capacitySpec":{"bufferMin":0,"bufferMax":0,"poolMin":0}}}'
  sleep 15
  # 删除
  kubectl -n $NS delete pool $POOL_NAME
  echo "旧 Pool 已删除"

  # 可选: 重命名新 Pool 为标准名称
  # kubectl -n $NS get pool $NEW_POOL -o yaml | \
  #   sed "s/name: $NEW_POOL/name: $POOL_NAME/" | \
  #   grep -v 'resourceVersion\|uid\|creationTimestamp\|generation' | \
  #   kubectl apply -f -
  # kubectl -n $NS delete pool $NEW_POOL
else
  echo "WARNING: 旧 Pool 仍有 $OLD_ALLOC 个已分配 Pod，请等待或使用 Strategy 3 排空"
fi
```

### 回滚

```bash
# 切换前回滚: 直接删除新 Pool
kubectl -n $NS delete pool $NEW_POOL

# 切换后回滚: 将 poolRef 切回旧 Pool，删除新 Pool 上的 BatchSandbox
# 1. 修改应用配置，poolRef 指回旧 Pool
# 2. 删除新 Pool 上的问题 BatchSandbox
kubectl -n $NS get batchsandbox -o jsonpath='{range .items[?(@.spec.poolRef=="'$NEW_POOL'")]}{.metadata.name}{"\n"}{end}' | \
  xargs -r -I{} kubectl -n $NS delete batchsandbox {}
# 3. 删除新 Pool
kubectl -n $NS delete pool $NEW_POOL
```

---

## 5. Strategy 3: 最小中断升级（状态外置 + 任务边界切换）

**适用场景**: 需要升级运行中 Sandbox 且不能丢失用户会话状态。

**核心思路**: VM 级热更新不可能，但可以在 **应用层** 实现近零中断 — 将会话状态持久化到共享存储，在任务间隙切换到新 Sandbox。

### 架构

```
┌──────────────────────────────────────────────────────────────────┐
│  AI Agent Service                                                │
│  ┌───────────────────────────────────┐                           │
│  │ Session Manager                    │                           │
│  │  - 追踪每个 session 的状态         │                           │
│  │  - 在任务边界触发迁移              │                           │
│  │  - 强制 per-session 目录隔离       │                           │
│  └──────────┬─────────────────────────┘                           │
│             │                                                     │
│  ┌──────────▼──────────┐  ┌───────────────────┐                  │
│  │ 旧 Sandbox (v1)     │  │ 新 Sandbox (v2)   │                  │
│  │ session-1 分配       │  │ session-1 迁移到此 │                  │
│  │ ┌────────────────┐  │  │ ┌───────────────┐ │                  │
│  │ │ [Task3] 执行中 │  │  │ │ [Task4] 待接收│ │                  │
│  │ └────────────────┘  │  │ └───────────────┘ │                  │
│  │     │               │  │     ▲             │                  │
│  │     ▼ save_state    │  │     │ restore     │                  │
│  └─────┼───────────────┘  └─────┼─────────────┘                  │
│        │   仅访问 session-1     │                                 │
│  ┌─────▼────────────────────────┼──────────────────────────────┐  │
│  │  EFS PVC (/data)                                            │  │
│  │  /data/sessions/                                            │  │
│  │    ├── session-1/           ← 新旧 sandbox 共享 (迁移中)    │  │
│  │    │   ├── state.json                                       │  │
│  │    │   └── workspace/                                       │  │
│  │    ├── session-2/           ← 其他 session，互不访问        │  │
│  │    │   ├── state.json                                       │  │
│  │    │   └── workspace/                                       │  │
│  │    └── session-N/                                           │  │
│  └─────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

**中断时间**: 仅在任务边界切换，用户感知 **0 中断**（当前任务完成后无缝衔接）。

### Per-Session 目录隔离模型

OpenSandbox Pool 的所有 Pod 使用相同 PodTemplateSpec（源码: `pool_controller.go:createPoolPod()`），无法在 Kubernetes 层面实现 per-pod 卷定制。因此采用**应用层目录隔离**:

| 层面 | 隔离方式 |
|------|---------|
| **EFS 挂载** | 所有 Pool Pod 挂载同一 PVC 到 `/data` (Pool 模型要求) |
| **目录隔离** | Agent 为每个 session 创建独立子目录 `/data/sessions/{session-id}/` |
| **访问控制** | Agent 强制限定所有操作在 session 目录内，不访问其他 session 目录 |
| **升级共享** | 新旧 sandbox 通过相同 session-id 访问同一子目录，实现状态迁移 |

**目录结构**:

```
/data/sessions/
├── {session-id-1}/
│   ├── state.json          # 会话状态 (环境变量、工作目录、历史)
│   └── workspace/          # 工作文件
├── {session-id-2}/         # 其他 session，互不访问
│   ├── state.json
│   └── workspace/
└── ...
```

**生命周期**: session claim 时创建目录 → 运行期间读写 → 升级迁移时新旧共享 → session 结束后清理。

> **安全说明**: Pool Pod 在文件系统层面技术上可访问 `/data/sessions/` 下所有目录。隔离由 Agent 服务层强制执行。未来 OpenSandbox Pool CRD 可能支持 per-pod volumeClaimTemplates (类似 StatefulSet)，届时可实现文件系统级隔离。

### 方案 A: Per-Session EFS 隔离 + 任务边界切换

最实用的方案。利用 EFS（NFS）作为共享存储，Agent 通过 per-session 目录隔离确保 sandbox 间数据不共享，升级时新旧 sandbox 通过 session-id 定向共享。

**Step 1: 创建 EFS 存储 (一次性)**

```bash
# 创建静态 PV (推荐，无需额外 IAM 配置)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-sandbox-pv
spec:
  capacity:
    storage: 10Gi
  accessModes: [ReadWriteMany]
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: <YOUR_EFS_ID>
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sandbox-shared-state
  namespace: opensandbox-system
spec:
  accessModes: [ReadWriteMany]
  storageClassName: efs-sc
  resources:
    requests:
      storage: 10Gi
EOF
```

**Step 2: Pool 模板中挂载 EFS PVC**

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: kata-clh-pool
  namespace: opensandbox-system
spec:
  template:
    spec:
      runtimeClassName: kata-clh
      containers:
      - name: sandbox
        image: busybox:1.37
        command: ["sh", "-c", "while true; do sleep 3600; done"]
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
        - name: shared-state
          mountPath: /data      # 所有 Pod 挂载同一路径
      volumes:
      - name: shared-state
        persistentVolumeClaim:
          claimName: sandbox-shared-state
  capacitySpec:
    bufferMin: 5
    bufferMax: 10
    poolMin: 5
    poolMax: 50
# 注意: 所有 Pod 共享 /data 挂载点，per-session 隔离由 Agent 层强制
```

**Step 3: AI Agent 侧 — Per-Session 隔离 + 任务边界迁移**

```python
# AI Agent Service 伪代码
import json, os, hashlib

class SandboxSession:
    def __init__(self, session_id, sandbox_client):
        self.session_id = session_id
        self.sandbox = sandbox_client
        self.session_dir = f"/data/sessions/{session_id}"
        self.state_path = f"{self.session_dir}/state.json"

    def setup_session_directory(self):
        """claim sandbox 后，创建 session 专属目录"""
        self.sandbox.exec(f"mkdir -p {self.session_dir}/workspace")
        self.sandbox.exec(f"chmod 700 {self.session_dir}")

    def scoped_exec(self, cmd):
        """所有命令限定在 session 目录内执行"""
        return self.sandbox.exec(f"cd {self.session_dir}/workspace && {cmd}")

    def save_state(self):
        """将会话状态写入 session 专属目录"""
        state = {
            "session_id": self.session_id,
            "env_vars": self.get_env_vars(),
            "working_dir": self.get_cwd(),
            "history": self.get_command_history(),
        }
        self.sandbox.exec(
            f"cat > {self.state_path} << 'STATEEOF'\n"
            f"{json.dumps(state)}\n"
            f"STATEEOF"
        )

    def restore_state(self):
        """从 session 专属目录恢复状态"""
        raw = self.sandbox.exec(f"cat {self.state_path}")
        state = json.loads(raw)
        for k, v in state["env_vars"].items():
            self.sandbox.exec(f"export {k}={v}")
        self.sandbox.exec(f"cd {state['working_dir']}")

    def migrate_to_new_sandbox(self, new_sandbox):
        """任务完成后，迁移到新 Sandbox (新旧共享同一 session 目录)"""
        self.save_state()
        self.sandbox = new_sandbox
        # 新 sandbox 挂载同一 EFS，通过 session-id 访问同一目录
        self.restore_state()

    def cleanup_session_directory(self):
        """session 结束后清理 (非迁移场景)"""
        self.sandbox.exec(f"rm -rf {self.session_dir}")
```

**Step 4: 执行升级流程**

```bash
# 1. 用 Strategy 2 (Blue-Green) 创建新 Pool
export NEW_POOL=kata-clh-pool-v2
# ... (同 Strategy 2 Step 1-3)

# 2. 对每个活跃 session，在当前任务完成后:
#    a) Agent 调用 save_state() 保存到 /data/sessions/{session-id}/
#    b) 从新 Pool claim 新 Sandbox (自动挂载同一 EFS PVC)
#    c) 新 Sandbox 通过 session-id 访问同一目录
#    d) Agent 调用 restore_state() 恢复会话
#    e) 删除旧 BatchSandbox

# 3. 旧 Pool 自然排空后删除
```

### 方案 B: OSEP-0008 Pause/Resume (未来)

OpenSandbox 正在实现 **rootfs snapshot** 机制 (OSEP-0008):

```
Pause:  运行中 Sandbox → commit rootfs → 推送到 OCI Registry → 释放资源
Resume: 从 snapshot 镜像 → 创建新 Sandbox → 文件系统完整恢复
```

- **保留**: 文件系统状态（安装的包、生成的文件、配置变更）
- **不保留**: 进程状态、内存、网络连接
- **优势**: 不需要额外的 PVC，rootfs 本身就是状态载体
- **状态**: 开发中 (Phase 1: PV 持久化, Phase 2: rootfs snapshot, Phase 3: 进程 checkpoint)

---

## 6. 策略对比与推荐

| | S1: 滚动更新 | S2: Blue-Green | S3: 状态外置+任务边界 |
|--|-------------|---------------|-------------------|
| **范围** | 仅空闲 Pod | 全量 (新 Pool) | 已分配 Pod |
| **会话状态** | 不影响 | 不影响 | **保留** ✅ |
| **用户感知中断** | 无 | 无 | **零中断** ✅ |
| **Session 间隔离** | N/A | N/A | Per-Session 目录隔离 |
| **升级期间新 Claim** | 正常 | 正常 | 正常 |
| **资源开销** | 1x Pool | 2x Pool | 2x Pool + EFS PVC |
| **复杂度** | 低 | 中 | 高 (需改 Agent) |
| **回滚速度** | 快 | 快 | 快 (切回旧 Pool) |
| **前提条件** | 无 | 无 | EFS + Agent 隔离实现 |

### 推荐组合

| 场景 | 推荐方案 |
|------|---------|
| **日常镜像更新** (安全补丁等) | S1 — 旧 sandbox 自然释放后逐步切换 |
| **生产重大升级** (API 变更等) | S2 — Blue-Green 切换，旧 sandbox 自然释放 |
| **零中断升级** (用户会话不丢失) | **S2 + S3** — Blue-Green 新 Pool + Per-Session EFS 隔离 + 任务边界迁移 |

---

## 7. 监控命令

### 实时 Dashboard

```bash
watch -n 5 '
echo "========== Pool 状态 =========="
kubectl -n opensandbox-system get pool -o wide
echo ""
echo "========== BatchSandbox 状态 =========="
kubectl -n opensandbox-system get batchsandbox -o wide
echo ""
echo "========== Pod Revision 分布 =========="
kubectl -n opensandbox-system get pods -l sandbox.opensandbox.io/pool-name \
  -o custom-columns="NAME:.metadata.name,POOL:.metadata.labels.sandbox\.opensandbox\.io/pool-name,REV:.metadata.labels.sandbox\.opensandbox\.io/pool-revision,PHASE:.status.phase" \
  --sort-by=.metadata.labels.sandbox\.opensandbox\.io/pool-revision
'
```

### Controller 滚动更新日志

```bash
kubectl -n $NS logs -l app.kubernetes.io/component=controller-manager --tail=100 | \
  grep -E "Rolling update detected|Scale pool|Created pool pod|Deleting pool pod"
```

### Revision 一致性检查

```bash
POOL_REV=$(kubectl -n $NS get pool $POOL_NAME -o jsonpath='{.status.revision}')
OLD_PODS=$(kubectl -n $NS get pods \
  -l sandbox.opensandbox.io/pool-name=$POOL_NAME,sandbox.opensandbox.io/pool-revision!=$POOL_REV \
  --no-headers 2>/dev/null | wc -l)
ALLOC=$(kubectl -n $NS get pool $POOL_NAME -o jsonpath='{.status.allocated}')

echo "Pool revision: $POOL_REV"
echo "Old-revision pods: $OLD_PODS (其中 allocated=$ALLOC)"

if [ "$OLD_PODS" = "0" ]; then
  echo "PASS: 所有 Pod 已更新"
elif [ "$OLD_PODS" = "$ALLOC" ]; then
  echo "INFO: 仅已分配 Pod 仍在旧版本 (预期行为)"
else
  echo "WARN: 存在未更新的空闲 Pod，Controller 可能仍在滚动中"
fi
```

### Pod 镜像版本检查

```bash
echo "=== Pool Pod 镜像分布 ==="
kubectl -n $NS get pods -l sandbox.opensandbox.io/pool-name \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}' | \
  sort -k2 | uniq -f1 -c
```
