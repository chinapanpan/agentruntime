# AI Agent Sandbox 架构文档

> EKS + Kata Containers (Cloud Hypervisor) + OpenSandbox Pool
> 目标: sandbox 创建延迟 < 1 秒，完整 VM 级内核隔离

---

## 1. 部署架构

### 1.1 整体部署架构

```mermaid
graph TB
    subgraph "AWS Cloud"
        subgraph "Amazon EKS (Managed Control Plane)"
            API["K8s API Server"]
            ETCD["etcd"]
        end

        subgraph "VPC"
            subgraph "Private Subnets"
                subgraph "c5.metal Node 1"
                    KVM1["/dev/kvm"]
                    CD1["containerd + kata-clh shim"]
                    subgraph "Pool Pods (Node 1)"
                        P1["microVM Pod 1 ✓"]
                        P2["microVM Pod 2 ✓"]
                        P3["microVM Pod 3 ✓"]
                    end
                end
                subgraph "c5.metal Node 2"
                    KVM2["/dev/kvm"]
                    CD2["containerd + kata-clh shim"]
                    subgraph "Pool Pods (Node 2)"
                        P4["microVM Pod 4 ✓"]
                        P5["microVM Pod 5 ✓"]
                    end
                end
            end
            NAT["NAT Gateway"]
        end

        subgraph "OpenSandbox System (namespace)"
            CTRL["OpenSandbox Controller"]
            POOL["Pool CRD: kata-clh-pool"]
            BATCH["BatchSandbox CRD"]
        end

        subgraph "kube-system (namespace)"
            KATA["kata-deploy DaemonSet"]
        end
    end

    Agent["AI Agent Service"] -->|"① 创建 BatchSandbox"| API
    API --> CTRL
    CTRL -->|"② Claim from Pool"| POOL
    POOL -->|"③ Assign Pod"| P1
    Agent -->|"④ kubectl exec"| P1

    KATA -->|"安装 kata-clh shim"| CD1
    KATA -->|"安装 kata-clh shim"| CD2
    CD1 --> KVM1
    CD2 --> KVM2
```

### 1.2 组件清单

| 组件 | 部署方式 | 说明 |
|------|---------|------|
| **EKS Control Plane** | AWS 托管 | K8s API Server + etcd |
| **c5.metal Worker Nodes** | Managed Nodegroup | 裸金属实例，提供 /dev/kvm |
| **kata-deploy** | DaemonSet (kube-system) | 在每个节点安装 Kata 运行时 + containerd shim |
| **OpenSandbox Controller** | Deployment (opensandbox-system) | 管理 Pool 和 BatchSandbox |
| **Pool CRD** | Custom Resource | 定义预热 Pod 池 |
| **BatchSandbox CRD** | Custom Resource | 请求 sandbox 实例 |

### 1.3 microVM 隔离模型

```
┌─────────────────────────────────────────────┐
│  c5.metal (Host: 96 vCPU, 192GB RAM)       │
│  Host Kernel: 6.12.x (Amazon Linux 2023)   │
│                                              │
│  ┌──────────────────┐ ┌──────────────────┐  │
│  │ Kata-CLH microVM │ │ Kata-CLH microVM │  │
│  │ Guest Kernel:    │ │ Guest Kernel:    │  │
│  │   6.18.x         │ │   6.18.x         │  │
│  │ ┌──────────────┐ │ │ ┌──────────────┐ │  │
│  │ │ Container    │ │ │ │ Container    │ │  │
│  │ │ (用户代码)   │ │ │ │ (用户代码)   │ │  │
│  │ └──────────────┘ │ │ └──────────────┘ │  │
│  │ Cloud Hypervisor │ │ Cloud Hypervisor │  │
│  └────────┬─────────┘ └────────┬─────────┘  │
│           │   virtio-net        │            │
│           └────────┬────────────┘            │
│                    │                         │
│            KVM (Hardware Virtualization)     │
└─────────────────────────────────────────────┘
```

每个 sandbox 运行在独立的 microVM 中:
- **独立内核**: Guest kernel 与 Host kernel 完全隔离
- **独立文件系统**: 用户代码无法访问其他 VM 或宿主机
- **硬件级隔离**: 通过 KVM + Cloud Hypervisor 实现 CPU/内存/IO 隔离
- **标准 Pod 网络**: 通过 virtio-net + VPC CNI 获得 Pod IP

---

## 2. OpenSandbox 架构

### 2.1 系统架构

```mermaid
graph TB
    subgraph "OpenSandbox Controller"
        RC["Reconciler"]
        PM["Pool Manager"]
        BM["BatchSandbox Manager"]
        WQ["Work Queue"]
    end

    subgraph "Kubernetes API"
        PoolCR["Pool CR"]
        BatchCR["BatchSandbox CR"]
        Pods["Pods"]
    end

    subgraph "Pool: kata-clh-pool"
        direction LR
        Available["Available Pods<br/>🟢🟢🟢🟢🟢"]
        Claimed["Claimed Pods<br/>🔵🔵"]
    end

    RC -->|"Watch"| PoolCR
    RC -->|"Watch"| BatchCR
    RC -->|"Watch"| Pods

    PM -->|"维持 bufferMin"| Available
    BM -->|"Claim"| Available
    Available -->|"转移所有权"| Claimed
    PM -->|"回填"| Pods
```

### 2.2 核心 CRD

#### Pool CRD

Pool 定义一个预热的 sandbox 实例池:

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: kata-clh-pool
  namespace: opensandbox-system
spec:
  template:                      # Pod 模板 (与标准 PodSpec 相同)
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
  capacitySpec:
    bufferMin: 5                 # 最少保持 5 个空闲 Pod
    bufferMax: 10                # 最多 10 个空闲 Pod
    poolMin: 5                   # Pool 最小总量
    poolMax: 50                  # Pool 最大总量
```

**capacitySpec 策略**:

| 字段 | 说明 | 作用 |
|------|------|------|
| `bufferMin` | 最少空闲 Pod 数 | 低于此值触发回填，保证 claim 时有 Pod 可用 |
| `bufferMax` | 最多空闲 Pod 数 | 高于此值缩容，避免浪费资源 |
| `poolMin` | Pool 最小总量 | 即使全部 claimed 也不会缩到此值以下 |
| `poolMax` | Pool 最大总量 | 硬上限，防止 Pod 爆炸 |

**Status 字段**:

```yaml
status:
  total: 10        # Pod 总数
  available: 7     # 空闲 Pod (可被 claim)
  allocated: 3     # 已被 claim 的 Pod
```

#### BatchSandbox CRD

BatchSandbox 用于请求 sandbox 实例:

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: my-task
  namespace: opensandbox-system
spec:
  replicas: 1                    # 需要的 sandbox 数量
  poolRef: kata-clh-pool         # 从指定 Pool claim (可选)
```

- **有 `poolRef`**: 从 Pool 中 claim 已运行的 Pod → **~227ms** (Warm Pool)
- **无 `poolRef`**: 新建 Pod 冷启动 → **~3,200ms** (Cold Start)

**Status 字段**:
```yaml
status:
  ready: 1         # 已就绪的 sandbox 数 (注意: 不是 readyReplicas)
  allocated: 1     # 已分配的数量
```

### 2.3 Sandbox 生命周期

```mermaid
sequenceDiagram
    participant Agent as AI Agent
    participant API as K8s API
    participant Ctrl as Controller
    participant Pool as Pool
    participant Pod as microVM Pod

    Note over Ctrl,Pool: 预热 (持续运行)
    Ctrl->>Pod: 创建 Pod 至 bufferMin
    Pod-->>Pool: Pod Running → Available

    Note over Agent,Pod: Claim (~227ms)
    Agent->>API: 创建 BatchSandbox (poolRef)
    API->>Ctrl: Watch event
    Ctrl->>Pool: 选择 Available Pod
    Ctrl->>Pod: 标记为 Claimed
    Ctrl->>API: status.ready = 1

    Note over Agent,Pod: 使用
    Agent->>Pod: kubectl exec (执行代码)
    Pod-->>Agent: 返回结果

    Note over Agent,Pod: 释放
    Agent->>API: 删除 BatchSandbox
    Ctrl->>Pod: 删除 Pod

    Note over Ctrl,Pool: 回填 (~3.2s)
    Ctrl->>Pool: 检测 available < bufferMin
    Ctrl->>Pod: 创建新 Pod
    Pod-->>Pool: 新 Pod → Available
```

### 2.4 性能模型

```
Warm Pool Claim ≈ 227ms
├── K8s API 写入 BatchSandbox     ~50ms
├── Controller Watch + Reconcile   ~30ms
├── Pool 查询 + Pod Label 更新    ~100ms
└── Status 更新 + 客户端检测       ~47ms

Cold Start ≈ 3,200ms
├── K8s 调度 (scheduler)           ~800ms
├── Image Pull (缓存后)            ~200ms
├── Kata-CLH microVM 启动          ~800ms
│   ├── Cloud Hypervisor 启动       ~200ms
│   ├── Guest Kernel 启动           ~300ms
│   └── Guest Agent 就绪            ~300ms
├── 网络配置 (VPC CNI)             ~500ms
└── Pod Ready                      ~900ms
```

---

## 3. 使用 Demo

### Demo 1: 创建单个 Sandbox 并执行命令

```bash
# 1. 创建 sandbox (从 warm pool claim, ~227ms)
cat <<'EOF' | kubectl apply -f -
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: demo-task
  namespace: opensandbox-system
spec:
  replicas: 1
  poolRef: kata-clh-pool
EOF

# 2. 等待就绪
kubectl -n opensandbox-system wait --for=jsonpath='{.status.ready}'=1 \
  batchsandbox/demo-task --timeout=10s

# 3. 找到 sandbox Pod
POD=$(kubectl -n opensandbox-system get batchsandbox demo-task \
  -o jsonpath='{.status.sandboxStatuses[0].podName}' 2>/dev/null)
# 如果上面返回空，用以下方式查找 claimed pod:
if [ -z "$POD" ]; then
  POD=$(kubectl -n opensandbox-system get pods \
    -l batch-sandbox-name=demo-task -o name 2>/dev/null | head -1)
fi

# 4. 在 sandbox 中执行命令
kubectl -n opensandbox-system exec ${POD} -- sh -c '
  echo "=== Sandbox Environment ==="
  echo "Kernel: $(uname -r)"
  echo "Hostname: $(hostname)"
  echo "PID 1: $(cat /proc/1/cmdline | tr "\0" " ")"
  echo ""
  echo "=== Running user code ==="
  echo "print(\"Hello from isolated sandbox!\")" > /tmp/hello.py
  # 如果有 python，执行; 否则用 sh 模拟
  if command -v python3 >/dev/null; then
    python3 /tmp/hello.py
  else
    echo "Hello from isolated sandbox!"
  fi
'

# 5. 释放 sandbox
kubectl -n opensandbox-system delete batchsandbox demo-task
```

### Demo 2: 批量创建 Sandbox

```bash
# 一次创建 5 个隔离的 sandbox
cat <<'EOF' | kubectl apply -f -
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: batch-demo
  namespace: opensandbox-system
spec:
  replicas: 5
  poolRef: kata-clh-pool
EOF

# 等待全部就绪
for i in $(seq 1 30); do
  READY=$(kubectl -n opensandbox-system get batchsandbox batch-demo \
    -o jsonpath='{.status.ready}' 2>/dev/null)
  [ "$READY" = "5" ] && echo "All 5 sandboxes ready!" && break
  sleep 1
done

# 查看状态
kubectl -n opensandbox-system get batchsandbox batch-demo

# 清理
kubectl -n opensandbox-system delete batchsandbox batch-demo
```

### Demo 3: Cold Start (无 Pool)

```bash
# 不指定 poolRef，直接创建 (冷启动 ~3.2s)
cat <<'EOF' | kubectl apply -f -
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: BatchSandbox
metadata:
  name: cold-demo
  namespace: opensandbox-system
spec:
  replicas: 1
  template:
    spec:
      runtimeClassName: kata-clh
      containers:
      - name: sandbox
        image: python:3.12-slim
        command: ["sh", "-c", "while true; do sleep 3600; done"]
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
EOF

# 等待就绪 (需要 ~3-5s)
for i in $(seq 1 60); do
  READY=$(kubectl -n opensandbox-system get batchsandbox cold-demo \
    -o jsonpath='{.status.ready}' 2>/dev/null)
  [ "$READY" = "1" ] && echo "Cold-start sandbox ready!" && break
  sleep 0.5
done

kubectl -n opensandbox-system delete batchsandbox cold-demo
```

---

## 4. 方案选型对比

### 4.1 为什么选择 Kata-CLH

| 特性 | Kata-CLH | Kata-QEMU |
|------|----------|-----------|
| 隔离级别 | 完整 VM | 完整 VM |
| EKS 兼容 | ✅ | ✅ |
| Pool Claim | 193-277ms | 226ms |
| 冷启动 | 2.8-3.4s | 3.3s |
| Pod Overhead | 130Mi | 160Mi |
| virtio-fs | ✅ 原生 | ✅ |
| 设计定位 | 云原生轻量 | 传统成熟 |

选择 CLH 的原因: 更轻量 (内存少 30Mi)、原生 virtio-fs 支持、专为云原生设计。

### 4.2 安全模型

| 攻击向量 | Kata-CLH 防御 |
|---------|--------------|
| 容器逃逸 | ✅ VM 隔离阻断，即使逃逸也在 VM 内 |
| 内核漏洞利用 | ✅ 独立 guest kernel，漏洞不影响 host |
| 资源耗尽 | ✅ VM 资源限制 + K8s cgroup |
| 网络攻击 | ✅ NetworkPolicy + VPC 安全组 |
| 数据泄露 | ✅ 独立文件系统，sandbox 间无共享 |

### 4.3 成本参考

| 实例类型 | 单价/hr | Pool 50 Pod 月成本 (2 节点) |
|---------|---------|---------------------------|
| c5.metal (96 vCPU, 192GB) | ~$5.06 | ~$7,300 |
| c5.metal Spot | ~$1.50-2.00 | ~$2,200-2,900 |

> 使用 Spot 实例可降低 60-70% 成本。

---

## 5. 容量规划

### 5.1 单节点容量 (c5.metal: 96 vCPU, 192GB)

| Sandbox 规格 | 每节点最大数 | 适用场景 |
|-------------|------------|---------|
| 1 vCPU, 256MB | ~90 | 轻量代码执行 |
| 1 vCPU, 512MB | ~80 | 一般任务 |
| 2 vCPU, 1GB | ~45 | 编译/ML推理 |
| 4 vCPU, 2GB | ~20 | 大型任务 |

### 5.2 Pool 规模建议

| 并发级别 | bufferMin | bufferMax | poolMax |
|---------|-----------|-----------|---------|
| 低 (< 10 QPS) | 5 | 10 | 20 |
| 中 (10-50 QPS) | 20 | 40 | 100 |
| 高 (> 50 QPS) | 50 | 100 | 200 |

`bufferMin` 应 >= 预期突发峰值。
