# AWS 第 8 代实例嵌套虚拟化验证报告

> **目标**: 验证 AWS C8i/M8i 实例的嵌套虚拟化特性是否支持 Kata-CLH + OpenSandbox，替代昂贵的 c5.metal 裸金属实例
> **验证环境**: AWS ap-northeast-1 (Tokyo)
> **验证日期**: 2026-04-09

---

## 1. 背景

### 现有方案: c5.metal 裸金属

Kata Containers 需要 `/dev/kvm` 硬件虚拟化支持。此前只能使用裸金属实例（如 c5.metal），成本高昂:

- c5.metal: **$5.14/hr** (96 vCPU, 192 GiB)
- 单个 Kata Pod 仅需 0.1 CPU + 128MB，c5.metal 资源严重浪费

### AWS 第 8 代嵌套虚拟化

2026年2月，AWS 宣布 **C8i、M8i、R8i** 实例支持嵌套虚拟化:

- 通过 CPU 选项 `NestedVirtualization=enabled` 启用
- Nitro 系统将 Intel VT-x 指令传递给虚拟机实例
- 支持 KVM 和 Hyper-V 作为 L1 hypervisor
- 架构: 物理主机 (L0 Nitro) → EC2 实例 (L1 KVM) → Kata microVM (L2)

### 成本对比

| 实例类型 | 类别 | vCPU | 内存 | 价格 (Tokyo) | vs c5.metal |
|---------|------|------|------|-------------|-------------|
| **c5.metal** | 裸金属 | 96 | 192 GiB | $5.14/hr | 基准 |
| **c8i.4xlarge** | 虚拟机 | 16 | 32 GiB | $0.94/hr | **-82%** |
| **c8i.2xlarge** | 虚拟机 | 8 | 16 GiB | $0.47/hr | **-91%** |
| **m8i.4xlarge** | 虚拟机 | 16 | 64 GiB | $1.09/hr | **-79%** |
| **m8i.2xlarge** | 虚拟机 | 8 | 32 GiB | $0.55/hr | **-89%** |

---

## 2. 验证环境

| 项目 | 值 |
|------|-----|
| **AWS Region** | ap-northeast-1 (Tokyo) |
| **EKS Cluster** | sandbox-nested-virt |
| **Kubernetes** | 1.31 |
| **测试节点 1** | c8i.4xlarge (16 vCPU, 32 GiB, Intel Xeon 6975P-C) |
| **测试节点 2** | m8i.4xlarge (16 vCPU, 64 GiB, Intel Xeon 6975P-C) |
| **嵌套虚拟化** | `NestedVirtualization=enabled` |
| **Kata Containers** | 3.28.0 |
| **OpenSandbox Controller** | 0.1.0 |

---

## 3. 部署过程

### 3.1 EKS 集群创建

```bash
eksctl create cluster --name sandbox-nested-virt \
  --region ap-northeast-1 --version 1.31 --without-nodegroup
```

### 3.2 启用嵌套虚拟化的节点

**关键发现**: EKS 托管节点组（managed nodegroup）目前**不支持**嵌套虚拟化 CPU 选项（GitHub issue: `aws/containers-roadmap#2784`）。需要通过 `aws ec2 run-instances` 直接启动实例。

**前提**: AWS CLI 版本 >= 2.34（旧版本不支持 `NestedVirtualization` 参数）。

```bash
# 创建节点 IAM 角色
aws iam create-role --role-name eksNodeRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{"Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"}]
  }'

# 附加必需策略
for POLICY in AmazonEKSWorkerNodePolicy AmazonEKS_CNI_Policy \
              AmazonEC2ContainerRegistryReadOnly AmazonSSMManagedInstanceCore; do
  aws iam attach-role-policy --role-name eksNodeRole \
    --policy-arn arn:aws:iam::aws:policy/$POLICY
done

# 创建 Instance Profile
aws iam create-instance-profile --instance-profile-name eksNodeRole
aws iam add-role-to-instance-profile \
  --instance-profile-name eksNodeRole --role-name eksNodeRole

# 获取 EKS optimized AMI
AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.31/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --region ap-northeast-1 --query "Parameter.Value" --output text)

# 构造 NodeConfig userdata
USERDATA=$(cat <<UDEOF | base64 -w 0
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="BOUNDARY"

--BOUNDARY
Content-Type: application/node.eks.aws

---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: sandbox-nested-virt
    apiServerEndpoint: <ENDPOINT>
    certificateAuthority: <CA_DATA>
    cidr: <SERVICE_CIDR>

--BOUNDARY--
UDEOF
)

# 启动 c8i.4xlarge (关键: --cpu-options NestedVirtualization=enabled)
aws ec2 run-instances \
  --region ap-northeast-1 \
  --image-id $AMI_ID \
  --instance-type c8i.4xlarge \
  --cpu-options "NestedVirtualization=enabled" \
  --subnet-id $SUBNET_ID \
  --security-group-ids $CLUSTER_SG \
  --iam-instance-profile Name=eksNodeRole \
  --user-data "$USERDATA" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=kubernetes.io/cluster/sandbox-nested-virt,Value=owned}]'

# 同样启动 m8i.4xlarge
aws ec2 run-instances \
  --instance-type m8i.4xlarge \
  --cpu-options "NestedVirtualization=enabled" \
  ...  # 其他参数同上
```

### 3.3 EKS Access Entry

新版 EKS 使用 API 认证模式，需要显式授权节点角色:

```bash
aws eks create-access-entry \
  --cluster-name sandbox-nested-virt \
  --region ap-northeast-1 \
  --principal-arn arn:aws:iam::340636688520:role/eksNodeRole \
  --type EC2_LINUX
```

### 3.4 /dev/kvm 验证

```
=== c8i.4xlarge ===
crw-rw-rw-. 1 root kvm 10, 232 /dev/kvm    ✅
kvm_intel             393216  0
kvm                  1155072  1 kvm_intel
CPU: Intel(R) Xeon(R) 6975P-C  (vmx flag present)

=== m8i.4xlarge ===
crw-rw-rw-. 1 root kvm 10, 232 /dev/kvm    ✅
kvm_intel             393216  0
kvm                  1155072  1 kvm_intel
CPU: Intel(R) Xeon(R) 6975P-C  (vmx flag present)
```

### 3.5 Kata-CLH 验证

```
=== c8i.4xlarge: kata-clh Pod ===
Guest kernel: 6.18.15  (Host: 6.1.166)  ✅  KATA_CLH_OK

=== m8i.4xlarge: kata-clh Pod ===
Guest kernel: 6.18.15  (Host: 6.1.166)  ✅  KATA_CLH_OK
```

Kata-CLH microVM 在嵌套虚拟化环境中完全正常运行，Guest kernel 与 Host kernel 隔离。

---

## 4. 性能测试结果

### 4.1 Warm Pool Claim 延迟

| Run | 延迟 |
|-----|------|
| 1 | 268ms |
| 2 | 208ms |
| 3 | 222ms |
| 4 | 286ms |
| 5 | 203ms |
| **平均** | **237ms** |

### 4.2 批量 Warm Pool Claim (10 replicas)

| 指标 | 值 |
|------|-----|
| 10 sandboxes ready | **215ms** |

### 4.3 Cold Start 延迟 (无 Pool)

| Run | 延迟 |
|-----|------|
| 1 | 3400ms |
| 2 | 2968ms |
| 3 | 2979ms |
| 4 | 3084ms |
| 5 | 2930ms |
| **平均** | **3072ms** |

---

## 5. 与 c5.metal 裸金属对比

| 指标 | c5.metal (裸金属) | C8i/M8i (嵌套虚拟化) | 差异 |
|------|-----------------|-------------------|------|
| **Warm Pool Claim** | avg 277ms | avg **237ms** | **-14%** (更快) |
| **Batch 10** | 241ms | **215ms** | **-11%** (更快) |
| **Cold Start** | avg 3209ms | avg **3072ms** | **-4%** (更快) |
| **价格 (4xlarge)** | $5.14/hr | **$0.94/hr** | **-82%** |
| **/dev/kvm** | 原生 | 嵌套虚拟化 | 功能等价 |
| **Kata Guest Kernel** | 6.18.15 | 6.18.15 | 相同 |
| **EKS Managed Nodegroup** | ✅ 支持 | ❌ 需自管理节点 | 运维复杂度略高 |

**关键结论**: 嵌套虚拟化性能与裸金属**持平甚至略优** (可能因为 Intel Xeon 6975P-C 是更新的处理器)，成本降低 **82%**。

---

## 6. 注意事项与建议

### EKS 托管节点组限制

截至 2026-04-09，EKS managed nodegroup **不支持** `NestedVirtualization` CPU 选项（`aws/containers-roadmap#2784`）。目前需要:

1. 通过 `aws ec2 run-instances --cpu-options "NestedVirtualization=enabled"` 手动启动实例
2. 使用 EKS Access Entry (`type: EC2_LINUX`) 授权节点加入集群
3. 或使用 Karpenter 等节点自动管理方案配合 Launch Template

**预期**: AWS 后续会在 EKS managed nodegroup 中原生支持此选项。

### AWS CLI 版本要求

`NestedVirtualization` 参数需要 AWS CLI **>= 2.34**。旧版本（如 2.33.x）会报 `Unknown parameter` 错误。

### 推荐实例选择

| 场景 | 推荐实例 | 理由 |
|------|---------|------|
| **成本敏感** | c8i.2xlarge ($0.47/hr) | 8 vCPU 可运行 ~40 个 Kata Pod |
| **均衡** | c8i.4xlarge ($0.94/hr) | 16 vCPU，足够一般生产负载 |
| **内存密集** | m8i.4xlarge ($1.09/hr) | 64 GiB 内存，适合大型 sandbox |
| **大规模** | c8i.24xlarge ($5.66/hr) | 96 vCPU，接近 c5.metal 但更便宜 |

### 安全考量

AWS 文档指出:
- 嵌套虚拟化的安全性由用户负责（"security in the cloud"）
- AWS 建议对延迟敏感的生产工作负载评估裸金属实例
- 本次测试显示嵌套虚拟化延迟与裸金属持平，但建议持续监控

---

## 7. 版本信息

| 组件 | 版本 |
|------|------|
| EKS | 1.31 |
| EKS Optimized AMI | ami-0238edc5a6a3efea9 (AL2023) |
| Host Kernel | 6.1.166-197.305.amzn2023.x86_64 |
| Kata Containers | 3.28.0 |
| Kata Guest Kernel | 6.18.15 |
| OpenSandbox Controller | 0.1.0 |
| AWS CLI | 2.34.27 (必须 >= 2.34) |
| CPU | Intel Xeon 6975P-C |
