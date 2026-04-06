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

### Kata-CLH 限制

- **无 live migration**: microVM 不支持在线迁移，镜像变更必须重建 Pod（新 microVM）
- **重建 = 会话丢失**: 已分配 Pod 被删除意味着用户会话终止
- **回填时间**: 新 Pod 冷启动约 2.8-3.4s

### 关键结论

> 空闲 Pod 可无缝升级（内置机制），已分配 Pod 需要 **业务层配合** 才能安全升级。

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

## 5. Strategy 3: 优雅排空运行中 Sandbox

**适用场景**: 配合 Strategy 1 或 2 使用，强制升级仍在旧版本上运行的 BatchSandbox。

**核心思路**: 通知 → 等待 → 删除旧 → 重建新。

### 执行步骤

**Step 1: 识别旧版本 BatchSandbox**

```bash
TARGET_POOL=${NEW_POOL:-$POOL_NAME}  # Blue-Green 用新 Pool，Rolling 用原 Pool
NEW_REV=$(kubectl -n $NS get pool $TARGET_POOL -o jsonpath='{.status.revision}')

echo "=== 需要升级的 BatchSandbox ==="
for BS in $(kubectl -n $NS get batchsandbox -o jsonpath='{.items[*].metadata.name}'); do
  POOL_REF=$(kubectl -n $NS get batchsandbox $BS -o jsonpath='{.spec.poolRef}' 2>/dev/null)

  # Blue-Green: 还在旧 Pool 上的
  if [ -n "$NEW_POOL" ] && [ "$POOL_REF" = "$POOL_NAME" ]; then
    REPLICAS=$(kubectl -n $NS get batchsandbox $BS -o jsonpath='{.spec.replicas}')
    echo "  $BS (pool=$POOL_REF, replicas=$REPLICAS) → 需要迁移到 $NEW_POOL"
  fi
done
```

**Step 2: 逐个优雅排空并重建**

```bash
DRAIN_TIMEOUT=300  # 秒
TARGET_POOL=${NEW_POOL:-$POOL_NAME}

for BS_NAME in $(kubectl -n $NS get batchsandbox -o jsonpath='{range .items[?(@.spec.poolRef=="'$POOL_NAME'")]}{.metadata.name}{" "}{end}'); do
  echo "========================================="
  echo "排空: $BS_NAME"
  echo "========================================="

  BS_REPLICAS=$(kubectl -n $NS get batchsandbox $BS_NAME -o jsonpath='{.spec.replicas}')

  # 1. 通知 sandbox 内的工作负载准备关闭
  ALLOC_RAW=$(kubectl -n $NS get batchsandbox $BS_NAME \
    -o jsonpath='{.metadata.annotations.sandbox\.opensandbox\.io/alloc-status}' 2>/dev/null)
  PODS=$(echo "$ALLOC_RAW" | python3.12 -c "
import sys,json
try:
    data = json.load(sys.stdin)
    for p in data.get('pods', []):
        print(p)
except: pass
" 2>/dev/null)

  for POD in $PODS; do
    echo "  通知 Pod $POD 准备关闭..."
    kubectl -n $NS exec $POD -- sh -c 'touch /tmp/shutdown-requested' 2>/dev/null || true
  done

  # 2. 等待优雅完成 (或超时)
  echo "  等待 ${DRAIN_TIMEOUT}s 优雅完成..."
  sleep $DRAIN_TIMEOUT &
  WAIT_PID=$!

  # 可在此检查业务层是否已完成 (如检查 /tmp/shutdown-complete 文件)
  ELAPSED=0
  while [ $ELAPSED -lt $DRAIN_TIMEOUT ]; do
    ALL_DONE=true
    for POD in $PODS; do
      DONE=$(kubectl -n $NS exec $POD -- sh -c 'cat /tmp/shutdown-complete 2>/dev/null' 2>/dev/null)
      if [ "$DONE" != "done" ]; then
        ALL_DONE=false
        break
      fi
    done
    if [ "$ALL_DONE" = "true" ]; then
      echo "  所有 Pod 已完成工作"
      kill $WAIT_PID 2>/dev/null
      break
    fi
    sleep 10
    ELAPSED=$((ELAPSED + 10))
  done

  # 3. 删除旧 BatchSandbox
  echo "  删除旧 BatchSandbox: $BS_NAME"
  kubectl -n $NS delete batchsandbox $BS_NAME --timeout=60s

  # 4. 在新 Pool 上重建
  echo "  重建 BatchSandbox: $BS_NAME → pool=$TARGET_POOL"
  cat <<EOF | kubectl apply -f -
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: $BS_NAME
  namespace: $NS
spec:
  replicas: ${BS_REPLICAS:-1}
  poolRef: $TARGET_POOL
EOF

  # 5. 等待新 sandbox 就绪
  for i in $(seq 1 60); do
    READY=$(kubectl -n $NS get batchsandbox $BS_NAME -o jsonpath='{.status.ready}' 2>/dev/null)
    if [ "$READY" = "${BS_REPLICAS:-1}" ]; then
      echo "  重建完成: ready=$READY"
      break
    fi
    sleep 2
  done

  echo ""
done
```

**Step 3: 验证全部升级完成**

```bash
echo "=== 最终状态 ==="
kubectl -n $NS get pool
echo ""
kubectl -n $NS get batchsandbox
echo ""
echo "=== 旧版本 Pod 数量 ==="
kubectl -n $NS get pods -l sandbox.opensandbox.io/pool-name=$POOL_NAME --no-headers 2>/dev/null | wc -l
```

### 回滚

如果重建失败，在旧 Pool（如果还存在）上重建:

```bash
kubectl -n $NS apply -f - <<EOF
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: $BS_NAME
  namespace: $NS
spec:
  replicas: ${BS_REPLICAS:-1}
  poolRef: $POOL_NAME
EOF
```

---

## 6. 策略对比与推荐

| | Strategy 1: 滚动更新 | Strategy 2: Blue-Green | Strategy 3: 优雅排空 |
|--|---------------------|----------------------|-------------------|
| **范围** | 仅空闲 Pod | 全量 (新 Pool) | 已分配 Pod |
| **服务中断** | 无 | 无 | 每个 sandbox 短暂中断 |
| **升级期间新 Claim** | 正常 (~200ms) | 正常 (~200ms) | 正常 |
| **资源开销** | 1x Pool | 2x Pool (过渡期) | 1x Pool |
| **复杂度** | 低 (一条 patch) | 中 (双 Pool 管理) | 高 (逐个排空) |
| **回滚速度** | 快 (re-patch) | 快 (切回旧 Pool) | 手动逐个 |
| **适合场景** | 常规更新 | 生产关键升级 | 强制全量升级 |

### 推荐组合

| 场景 | 推荐方案 |
|------|---------|
| **日常镜像更新** (安全补丁等) | Strategy 1 — 旧 sandbox 自然释放后逐步切换 |
| **生产重大升级** (API 变更等) | Strategy 2 + Strategy 3 — Blue-Green 切换后排空旧 sandbox |
| **紧急安全修复** | Strategy 1 + Strategy 3 — 滚动更新空闲 Pod + 立即排空已分配 sandbox |

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
