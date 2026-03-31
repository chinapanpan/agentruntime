# EKS + Kata-CLH + OpenSandbox 部署与测试指南

> **用途**: Claude Code skill — 可由 Claude Code 直接执行完成全部部署和测试
> **适用范围**: 任意 AWS Account、任意 Region (需要该 Region 支持 c5.metal)

---

## 前置要求

### 必需工具

在开始前确认以下工具已安装:

```bash
aws --version        # >= 2.x
eksctl version       # >= 0.200
kubectl version --client  # >= 1.31
helm version         # >= 3.16
jq --version         # any
git --version        # any
```

### AWS 权限

```bash
aws sts get-caller-identity
# 应返回有效身份，需要 EKS/EC2/CloudFormation/IAM 权限
```

---

## Step 0: 设置环境变量

**重要**: 部署前必须设置以下变量。请根据实际环境修改 `AWS_ACCOUNT_ID` 和 `REGION`。

```bash
# ===== 必须修改 =====
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export REGION=us-west-2          # 修改为目标 Region

# ===== 可选修改 =====
export CLUSTER_NAME=sandbox-prod
export K8S_VERSION=1.34
export NODE_TYPE=c5.metal
export NODE_COUNT=2
export OPENSANDBOX_NS=opensandbox-system

# ===== 验证 =====
echo "AWS Account: ${AWS_ACCOUNT_ID}"
echo "Region:      ${REGION}"
echo "Cluster:     ${CLUSTER_NAME}"
echo "K8s Version: ${K8S_VERSION}"
echo "Node Type:   ${NODE_TYPE}"
```

**验证输出示例**:
```
AWS Account: 340636688520
Region:      us-west-2
Cluster:     sandbox-prod
K8s Version: 1.34
Node Type:   c5.metal
```

---

## Step 1: 创建 EKS 集群

### 1.1 创建集群 (无节点组)

```bash
eksctl create cluster \
  --name ${CLUSTER_NAME} \
  --region ${REGION} \
  --version ${K8S_VERSION} \
  --without-nodegroup
```

**耗时**: ~15-20 分钟

**预期输出**:
```
[✔]  EKS cluster "sandbox-prod" in "us-west-2" region is ready
```

### 1.2 创建 c5.metal 节点组

```bash
eksctl create nodegroup \
  --cluster ${CLUSTER_NAME} \
  --region ${REGION} \
  --name kata-metal \
  --node-type ${NODE_TYPE} \
  --nodes ${NODE_COUNT} \
  --nodes-min 1 \
  --nodes-max 4 \
  --node-volume-size 100 \
  --managed
```

**耗时**: ~5-10 分钟

**预期输出**:
```
[✔]  created 1 managed nodegroup(s) in cluster "sandbox-prod"
```

### 1.3 配置 kubeconfig

```bash
aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${REGION}
```

### 1.4 验证节点就绪

```bash
kubectl get nodes -o wide
```

**预期**: 所有节点 STATUS 为 `Ready`，CONTAINER-RUNTIME 为 `containerd://2.x`。

### 1.5 验证 /dev/kvm

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kvm-check
spec:
  hostPID: true
  nodeName: $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
  tolerations:
  - operator: Exists
  containers:
  - name: check
    image: ubuntu:22.04
    securityContext:
      privileged: true
    command: ["bash", "-c"]
    args:
    - |
      nsenter -t 1 -m -u -n -i -- bash -c '
        ls -la /dev/kvm 2>&1
        lsmod | grep kvm
        uname -r
      '
  restartPolicy: Never
EOF

sleep 30
kubectl logs kvm-check
kubectl delete pod kvm-check --force
```

**预期输出**: 包含 `crw-rw-rw-` 权限的 `/dev/kvm` 设备文件，以及 `kvm_intel` 模块已加载。

**故障排除**:
- `/dev/kvm` 不存在 → 节点不是裸金属实例，确认使用了 `c5.metal`
- 节点长时间 NotReady → 检查安全组是否允许与 EKS 控制平面通信

---

## Step 2: 安装 Kata Containers

### 2.1 下载 kata-deploy Helm chart

```bash
KATA_VERSION=3.28.0
mkdir -p /tmp/kata-chart/templates

for f in Chart.yaml values.yaml; do
  curl -fsSL "https://raw.githubusercontent.com/kata-containers/kata-containers/${KATA_VERSION}/tools/packaging/kata-deploy/helm-chart/kata-deploy/$f" \
    -o /tmp/kata-chart/$f
done

for f in _helpers.tpl kata-deploy.yaml kata-rbac.yaml runtimeclasses.yaml custom-runtimes.yaml kata-cleanup-rbac-job.yaml; do
  curl -fsSL "https://raw.githubusercontent.com/kata-containers/kata-containers/${KATA_VERSION}/tools/packaging/kata-deploy/helm-chart/kata-deploy/templates/$f" \
    -o /tmp/kata-chart/templates/$f 2>/dev/null || echo "WARN: $f not found (optional)"
done

# 移除 NFD 子 chart 依赖
sed -i '/^dependencies:/,/condition:.*enabled$/d' /tmp/kata-chart/Chart.yaml
```

### 2.2 安装 kata-deploy

```bash
helm install kata-deploy /tmp/kata-chart \
  --namespace kube-system \
  --set k8sDistribution=k8s \
  --set node-feature-discovery.enabled=false
```

### 2.3 等待 DaemonSet 就绪

```bash
kubectl -n kube-system wait --for=condition=Ready pod -l name=kata-deploy --timeout=600s
```

**耗时**: ~3-5 分钟 (下载 Kata 运行时 ~2GB)

### 2.4 验证安装

```bash
# 检查日志
kubectl -n kube-system logs -l name=kata-deploy --tail=3

# 预期输出包含:
# Kata Containers installation completed successfully
# Install completed, daemonset mode: waiting for SIGTERM

# 检查 RuntimeClasses
kubectl get runtimeclass | grep kata-clh
# 预期: kata-clh 存在
```

### 2.5 验证 Kata-CLH 运行

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kata-clh-verify
spec:
  runtimeClassName: kata-clh
  containers:
  - name: test
    image: busybox:latest
    command: ["sh", "-c", "echo HOST_KERNEL_CHECK; uname -r; echo KATA_CLH_OK"]
  restartPolicy: Never
EOF

sleep 20
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/kata-clh-verify --timeout=60s
kubectl logs kata-clh-verify
kubectl delete pod kata-clh-verify
```

**预期**: `uname -r` 输出 Kata guest kernel 版本 (如 `6.18.x`)，与宿主机不同。

**故障排除**:
- Pod 卡在 ContainerCreating → `kubectl describe pod kata-clh-verify` 检查事件
- FailedCreatePodSandBox → /dev/kvm 不可用或 containerd 配置问题
- 无法调度 → 检查节点是否有 label `katacontainers.io/kata-runtime=true`

---

## Step 3: 安装 OpenSandbox Controller

### 3.1 克隆 OpenSandbox 仓库

```bash
rm -rf /tmp/opensandbox-repo
git clone https://github.com/alibaba/OpenSandbox.git /tmp/opensandbox-repo
```

### 3.2 修复 Helm 模板 (关键!)

OpenSandbox Controller 二进制不支持 `--kube-client-qps` 和 `--kube-client-burst` 参数，必须移除，否则 Controller 会 CrashLoopBackOff。

```bash
DEPLOY_TMPL="/tmp/opensandbox-repo/kubernetes/charts/opensandbox-controller/templates/deployment.yaml"

# 删除 kube-client 相关行
sed -i '/kubeClient.*qps/d; /kubeClient.*burst/d; /kube-client-qps/d; /kube-client-burst/d' "$DEPLOY_TMPL"

# 检查是否有孤立的 {{- end }} (删掉 if 块后可能残留)
# 正确的 args 部分应该是:
#   args:
#   {{- if .Values.controller.leaderElection.enabled }}
#   - --leader-elect
#   {{- end }}
#   - --health-probe-bind-address=:8081
#   - --zap-log-level={{ .Values.controller.logLevel }}
#   ports:

# 查看 args 段验证
grep -n -A 12 "args:" "$DEPLOY_TMPL" | head -15
```

如果 `--zap-log-level` 行之后、`ports:` 行之前有多余的 `{{- end }}`，需要手动删除这些孤立行。

### 3.3 安装 Controller

```bash
helm install opensandbox-controller \
  /tmp/opensandbox-repo/kubernetes/charts/opensandbox-controller \
  --namespace ${OPENSANDBOX_NS} \
  --create-namespace \
  --wait --timeout 120s
```

### 3.4 验证

```bash
# Controller Pod
kubectl -n ${OPENSANDBOX_NS} get pods
# 预期: opensandbox-controller-manager-xxxxx 1/1 Running

# CRDs
kubectl get crd | grep sandbox
# 预期:
#   batchsandboxes.sandbox.opensandbox.io
#   pools.sandbox.opensandbox.io
```

**故障排除**:
- CrashLoopBackOff + 日志含 `flag provided but not defined: -kube-client-qps` → Step 3.2 修复未生效，重新执行并 `helm upgrade`
- 镜像拉取失败 → 镜像仓库 `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com` 可能需要网络可达

---

## Step 4: 创建 Kata-CLH Pool

### 4.1 创建 Pool

```bash
cat <<'EOF' | kubectl apply -f -
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
        image: busybox:latest
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

### 4.2 等待 Pool 就绪

```bash
echo "等待 Pool 预热..."
for i in $(seq 1 60); do
  AVAIL=$(kubectl -n ${OPENSANDBOX_NS} get pool kata-clh-pool -o jsonpath='{.status.available}' 2>/dev/null)
  if [ "${AVAIL}" -ge 5 ] 2>/dev/null; then
    echo "Pool 就绪: available=${AVAIL}"
    break
  fi
  echo "  [${i}] available=${AVAIL:-0}"
  sleep 5
done

kubectl -n ${OPENSANDBOX_NS} get pool kata-clh-pool
kubectl -n ${OPENSANDBOX_NS} get pods | grep kata-clh-pool
```

**预期**: `AVAILABLE >= 5`，至少 5 个 Pod 处于 Running 状态。

### 4.3 快速验证 BatchSandbox

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: quick-verify
  namespace: opensandbox-system
spec:
  replicas: 1
  poolRef: kata-clh-pool
EOF

for i in $(seq 1 30); do
  READY=$(kubectl -n ${OPENSANDBOX_NS} get batchsandbox quick-verify -o jsonpath='{.status.ready}' 2>/dev/null)
  [ "$READY" = "1" ] && echo "BatchSandbox ready!" && break
  sleep 1
done

kubectl -n ${OPENSANDBOX_NS} get batchsandbox quick-verify
kubectl -n ${OPENSANDBOX_NS} delete batchsandbox quick-verify
```

**预期**: `READY=1`，然后清理。

---

## Step 5: 性能测试

### 5.0 部署集群内测试 Pod

所有延迟测试必须从集群内部执行。外部 kubectl 会引入 ~20s 网络延迟。

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sandbox-tester
  namespace: opensandbox-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sandbox-tester-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: sandbox-tester
  namespace: opensandbox-system
---
apiVersion: v1
kind: Pod
metadata:
  name: kubectl-tester
  namespace: opensandbox-system
spec:
  serviceAccountName: sandbox-tester
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sh", "-c", "while true; do sleep 3600; done"]
  restartPolicy: Never
EOF

kubectl -n opensandbox-system wait --for=condition=Ready pod/kubectl-tester --timeout=60s
kubectl -n opensandbox-system exec kubectl-tester -- kubectl get nodes
```

### 5.1 Test: Warm Pool Claim 延迟

从预热 Pool 中 claim sandbox 的延迟 — 这是生产环境最关键的指标。

```bash
kubectl -n opensandbox-system exec kubectl-tester -- sh -c '
echo "============================================"
echo "Test: Warm Pool Claim 延迟 (5 runs)"
echo "============================================"

RESULTS=""
for i in 1 2 3 4 5; do
  START_MS=$(date +%s%3N)

  cat <<BSEOF | kubectl apply -f - 2>/dev/null
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: warm-test-${i}
  namespace: opensandbox-system
spec:
  replicas: 1
  poolRef: kata-clh-pool
BSEOF

  for attempt in $(seq 1 300); do
    VAL=$(kubectl -n opensandbox-system get batchsandbox warm-test-${i} -o jsonpath="{.status.ready}" 2>/dev/null)
    if [ "$VAL" = "1" ]; then
      END_MS=$(date +%s%3N)
      LATENCY=$((END_MS - START_MS))
      echo "[Run $i] Warm Pool Claim: ${LATENCY}ms"
      RESULTS="${RESULTS} ${LATENCY}"
      break
    fi
    sleep 0.05
  done
  sleep 3
done

echo ""
echo "============================================"
echo "Warm Pool Results: ${RESULTS}"
echo "============================================"

for i in 1 2 3 4 5; do
  kubectl -n opensandbox-system delete batchsandbox warm-test-${i} 2>/dev/null &
done
wait
echo "Cleanup done"
'
```

**通过标准**: 平均延迟 < 500ms (基准: ~227-277ms)

### 5.2 Test: 批量 Warm Pool Claim (10 replicas)

```bash
# 先确保 Pool 有足够 Pod
kubectl -n opensandbox-system patch pool kata-clh-pool --type merge \
  -p '{"spec":{"capacitySpec":{"bufferMin":12,"bufferMax":15,"poolMin":12}}}'
echo "Scaling pool, waiting..."
for i in $(seq 1 60); do
  AVAIL=$(kubectl -n opensandbox-system get pool kata-clh-pool -o jsonpath='{.status.available}' 2>/dev/null)
  [ "${AVAIL}" -ge 10 ] 2>/dev/null && echo "Pool ready: available=${AVAIL}" && break
  sleep 5
done

kubectl -n opensandbox-system exec kubectl-tester -- sh -c '
echo "============================================"
echo "Test: Batch Warm Pool Claim (10 replicas)"
echo "============================================"

AVAIL=$(kubectl -n opensandbox-system get pool kata-clh-pool -o jsonpath="{.status.available}")
echo "Pool available before: ${AVAIL}"

START_MS=$(date +%s%3N)
cat <<BSEOF | kubectl apply -f - 2>/dev/null
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: batch-warm-10
  namespace: opensandbox-system
spec:
  replicas: 10
  poolRef: kata-clh-pool
BSEOF

for attempt in $(seq 1 600); do
  VAL=$(kubectl -n opensandbox-system get batchsandbox batch-warm-10 -o jsonpath="{.status.ready}" 2>/dev/null)
  if [ "$VAL" = "10" ]; then
    END_MS=$(date +%s%3N)
    echo "10 sandboxes ready: $((END_MS - START_MS))ms"
    break
  fi
  sleep 0.05
done

kubectl -n opensandbox-system get batchsandbox batch-warm-10
kubectl -n opensandbox-system delete batchsandbox batch-warm-10
echo "Cleanup done"
'

# 恢复 Pool 设置
kubectl -n opensandbox-system patch pool kata-clh-pool --type merge \
  -p '{"spec":{"capacitySpec":{"bufferMin":5,"bufferMax":10,"poolMin":5}}}'
```

**通过标准**: Pool 充足时 < 500ms (基准: ~191-241ms)

### 5.3 Test: Cold Start 延迟 (无 Warm Pool)

不使用 Pool 时的 Kata-CLH 冷启动延迟，也是 Pool 回填时的单个 Pod 启动耗时。

```bash
kubectl -n opensandbox-system exec kubectl-tester -- sh -c '
echo "============================================"
echo "Test: Cold Start 延迟 (无 Pool, 5 runs)"
echo "============================================"

RESULTS=""
for i in 1 2 3 4 5; do
  START_MS=$(date +%s%3N)

  cat <<BSEOF | kubectl apply -f - 2>/dev/null
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: cold-test-${i}
  namespace: opensandbox-system
spec:
  replicas: 1
  template:
    spec:
      runtimeClassName: kata-clh
      containers:
      - name: sandbox
        image: busybox:latest
        command: ["sh", "-c", "while true; do sleep 3600; done"]
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
BSEOF

  for attempt in $(seq 1 300); do
    VAL=$(kubectl -n opensandbox-system get batchsandbox cold-test-${i} -o jsonpath="{.status.ready}" 2>/dev/null)
    if [ "$VAL" = "1" ]; then
      END_MS=$(date +%s%3N)
      LATENCY=$((END_MS - START_MS))
      echo "[Run $i] Cold Start: ${LATENCY}ms"
      RESULTS="${RESULTS} ${LATENCY}"
      break
    fi
    sleep 0.1
  done

  kubectl -n opensandbox-system delete batchsandbox cold-test-${i} 2>/dev/null &
  sleep 2
done
wait

echo ""
echo "============================================"
echo "Cold Start Results: ${RESULTS}"
echo "============================================"
'
```

**通过标准**: 平均延迟 < 5,000ms (基准: ~3,200-3,400ms)

### 5.4 Test: Pool 回填验证

```bash
kubectl -n opensandbox-system exec kubectl-tester -- sh -c '
echo "============================================"
echo "Test: Pool 回填验证"
echo "============================================"

echo "--- Before claim ---"
kubectl -n opensandbox-system get pool kata-clh-pool
BEFORE=$(kubectl -n opensandbox-system get pool kata-clh-pool -o jsonpath="{.status.available}")
echo "Available: ${BEFORE}"

echo ""
echo "--- Claiming 3 pods ---"
cat <<BSEOF | kubectl apply -f - 2>/dev/null
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: backfill-test
  namespace: opensandbox-system
spec:
  replicas: 3
  poolRef: kata-clh-pool
BSEOF

sleep 3
AFTER=$(kubectl -n opensandbox-system get pool kata-clh-pool -o jsonpath="{.status.available}")
echo "Available after claim: ${AFTER}"

echo ""
echo "--- Waiting for backfill ---"
for i in $(seq 1 24); do
  CURRENT=$(kubectl -n opensandbox-system get pool kata-clh-pool -o jsonpath="{.status.available}")
  TOTAL=$(kubectl -n opensandbox-system get pool kata-clh-pool -o jsonpath="{.status.total}")
  echo "  [${i}0s] available=${CURRENT}, total=${TOTAL}"
  if [ "${CURRENT}" -ge 5 ] 2>/dev/null; then
    echo ""
    echo "Pool backfilled! available=${CURRENT} >= bufferMin(5)"
    break
  fi
  sleep 10
done

echo ""
kubectl -n opensandbox-system get pool kata-clh-pool
kubectl -n opensandbox-system delete batchsandbox backfill-test
echo "Cleanup done"
'
```

**通过标准**: 120 秒内 available 恢复到 >= bufferMin (5)

### 5.5 清理测试资源

```bash
for name in warm-test-1 warm-test-2 warm-test-3 warm-test-4 warm-test-5 \
  batch-warm-10 cold-test-1 cold-test-2 cold-test-3 cold-test-4 cold-test-5 \
  backfill-test quick-verify; do
  kubectl -n opensandbox-system delete batchsandbox ${name} 2>/dev/null
done
kubectl -n opensandbox-system delete pod kubectl-tester
kubectl delete clusterrolebinding sandbox-tester-admin
kubectl -n opensandbox-system delete serviceaccount sandbox-tester
echo "Test resources cleaned"
```

---

## Step 6: 清理全部资源

```bash
# 删除 OpenSandbox 资源
kubectl -n ${OPENSANDBOX_NS} delete batchsandbox --all
kubectl -n ${OPENSANDBOX_NS} delete pool --all
sleep 10

# 卸载 Helm releases
helm uninstall opensandbox-controller -n ${OPENSANDBOX_NS}
kubectl delete namespace ${OPENSANDBOX_NS}
helm uninstall kata-deploy -n kube-system

# 删除 EKS 集群
eksctl delete cluster --name ${CLUSTER_NAME} --region ${REGION} --wait
```

**耗时**: ~10-15 分钟

**验证清理**:
```bash
aws cloudformation list-stacks --region ${REGION} \
  --stack-status-filter CREATE_COMPLETE \
  --query "StackSummaries[?starts_with(StackName, 'eksctl-${CLUSTER_NAME}')].StackName" \
  --output text
# 预期: 无输出
```

---

## 快速参考

```bash
# Pool 状态
kubectl -n opensandbox-system get pool

# 所有 BatchSandbox
kubectl -n opensandbox-system get batchsandbox

# Controller 日志
kubectl -n opensandbox-system logs -l app.kubernetes.io/component=controller-manager -f

# Kata RuntimeClasses
kubectl get runtimeclass | grep kata
```

## 版本信息

| 组件 | 版本 |
|------|------|
| EKS | 1.34 |
| Kata Containers | 3.28.0 |
| OpenSandbox Controller | 0.1.0 |
| Cloud Hypervisor | (bundled in Kata) |

## 历史测试结果

| Region | 日期 | Warm Pool Claim | Batch 10 | Cold Start | 通过 |
|--------|------|----------------|----------|------------|------|
| us-west-2 | 2026-03-31 | avg 193ms [154,134,274,157,247] | 271ms | avg 2835ms [2732,2376,3071,3064,2934] | ✅ |
| ap-northeast-1 | 2026-03-30 | avg 277ms [293,288,253,258,291] | 241ms | avg 3209ms [3077,3091,3871,3082,2923] | ✅ |
