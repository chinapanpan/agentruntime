# EKS + Karpenter (Nested Virtualization) + Kata-CLH + OpenSandbox Verification

## Summary

This document verifies that **Karpenter can auto-provision c8i instances with nested virtualization** for running Kata-CLH microVMs with OpenSandbox. This eliminates the need for bare-metal instances (c5.metal at $5.14/hr) or manually-managed node groups, achieving **automatic node scaling with ~82% cost reduction** ($0.47/hr per c8i.2xlarge).

| Item | Value |
|------|-------|
| EKS Cluster | `sandbox-karpenter-nv` (K8s 1.32) |
| Region | ap-northeast-1 (Tokyo) |
| Karpenter Version | Custom build from PR #9043 (commit bd49fd3) + bug fix |
| Instance Type | c8i.2xlarge (on-demand, $0.47/hr) |
| Nested Virtualization | Enabled via `CpuOptions.NestedVirtualization` |
| Kata Version | 3.28.0 (cloud-hypervisor) |
| Guest Kernel | 6.18.15 |
| Host Kernel | 6.1.166-197.305.amzn2023.x86_64 |

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    EKS Cluster                           │
│                                                          │
│  ┌──────────────┐   ┌─────────────────────────────────┐ │
│  │ System Nodes  │   │ Karpenter-Managed Nodes          │ │
│  │ (m5.xlarge×2) │   │ (c8i.2xlarge, auto-scaled)       │ │
│  │               │   │                                   │ │
│  │ • Karpenter   │   │ • /dev/kvm (nested virt)          │ │
│  │ • kata-deploy │   │ • kata-deploy (DaemonSet)         │ │
│  │ • OpenSandbox │   │ • Pool Pods (Kata-CLH microVMs)   │ │
│  │   Controller  │   │                                   │ │
│  └──────────────┘   └─────────────────────────────────┘ │
│                              ↑                            │
│                     Karpenter auto-provisions             │
│                     c8i.2xlarge with                      │
│                     NestedVirtualization=enabled           │
└─────────────────────────────────────────────────────────┘
```

## Prerequisites

- Custom Karpenter build from PR [aws/karpenter-provider-aws#9043](https://github.com/aws/karpenter-provider-aws/pull/9043)
- The PR adds `cpuOptions.nestedVirtualization` to EC2NodeClass
- **Critical bug fix required**: see [Bug Fix](#bug-fix-wellknownlabels-registration) section

## Custom Karpenter Build

### Source

```bash
git clone --branch feat/cpu-options-nested-virt \
  https://github.com/windsornguyen/karpenter-provider-aws.git
cd karpenter-provider-aws
```

### Bug Fix: WellKnownLabels Registration

PR #9043 introduces `LabelInstanceNestedVirtualization` (`karpenter.k8s.aws/instance-nested-virtualization`) but **does not register it in `WellKnownLabels`**. This causes the `CompatibleAvailableFilter` to reject all instance types that support nested virtualization, with error:

```
creating instance, insufficient capacity, all requested instance types were unavailable during launch
```

**Root cause**: When an instance type has `karpenter.k8s.aws/instance-nested-virtualization In ["true"]` in its requirements, but the NodeClaim doesn't include this label, `Requirements.Compatible()` returns an error because the label is not in the `WellKnownLabels` set (which allows labels to be "undefined" in one requirement set).

**Fix** (`pkg/apis/v1/labels.go`):

```go
// In init() function, add to karpv1.WellKnownLabels.Insert():
    LabelPlacementGroupID,
    LabelPlacementGroupPartition,
    LabelInstanceNestedVirtualization,  // ← ADD THIS LINE
)
```

### CRD Regeneration

The PR also requires CRD regeneration (the Go types were added but CRD YAML was not updated):

```bash
go install sigs.k8s.io/controller-tools/cmd/controller-gen@latest
controller-gen crd paths="./pkg/apis/..." output:crd:artifacts:config=pkg/apis/crds
```

### Build & Push

```bash
# Build binary
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -tags disable_vpa \
  -ldflags="-s -w" -o controller ./cmd/controller/

# Build and push Docker image
docker build -f Dockerfile.quick -t ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/karpenter/controller:nested-virt-fix .
docker push ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/karpenter/controller:nested-virt-fix
```

## Deployment

### 1. EKS Cluster + Karpenter IAM

```bash
# Karpenter IAM (CloudFormation)
aws cloudformation deploy --stack-name "Karpenter-sandbox-karpenter-nv" \
  --template-file cloudformation.yaml --capabilities CAPABILITY_NAMED_IAM

# EKS cluster with eksctl (includes Pod Identity for Karpenter)
eksctl create cluster -f karpenter-cluster.yaml
```

### 2. Deploy Custom Karpenter

```bash
# Install Karpenter with custom image
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --namespace kube-system --version 1.11.1 \
  --set "settings.clusterName=sandbox-karpenter-nv" \
  --set "settings.interruptionQueue=sandbox-karpenter-nv" \
  --set controller.image.repository=${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/karpenter/controller \
  --set controller.image.tag=nested-virt-fix \
  --skip-crds

# Apply regenerated CRDs
kubectl apply --server-side --force-conflicts -f pkg/apis/crds/
```

### 3. EC2NodeClass + NodePool

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: nested-virt-kata
spec:
  role: "KarpenterNodeRole-sandbox-karpenter-nv"
  cpuOptions:
    nestedVirtualization: enabled    # ← PR #9043 new field
  amiSelectorTerms:
    - alias: al2023@latest
  subnetSelectorTerms:
    - tags: { karpenter.sh/discovery: "sandbox-karpenter-nv" }
  securityGroupSelectorTerms:
    - tags: { karpenter.sh/discovery: "sandbox-karpenter-nv" }
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs: { volumeSize: 100Gi, volumeType: gp3, deleteOnTermination: true }
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: kata-nested-virt
spec:
  template:
    metadata:
      labels:
        katacontainers.io/kata-runtime: "true"
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["c8i", "m8i", "r8i"]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["2xlarge", "4xlarge", "8xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: nested-virt-kata
      expireAfter: 720h
  limits:
    cpu: 128
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30m
```

### 4. Kata + OpenSandbox

Standard deployment as per `docs/deploy-kata-clh.md` (Steps 2-3).

## Verification Results

### Karpenter Auto-Provisioning

| Metric | Result |
|--------|--------|
| Instance type selected | c8i.2xlarge |
| CpuOptions.NestedVirtualization | `enabled` (verified via DescribeInstances) |
| /dev/kvm on node | `crw-rw-rw-` (available) |
| CPU vmx flag | Present |
| Node registration time | ~15s (from CreateFleet to node Ready) |
| Node initialization time | ~25s (from CreateFleet to fully initialized) |

### Auto-Scaling

When pool demand exceeded single-node capacity, Karpenter auto-scaled from 1 to 5 c8i.2xlarge nodes:

| Time | Nodes | Event |
|------|-------|-------|
| T+0s | 1 | 20-replica BatchSandbox created |
| T+19s | 1 | 7 pre-warmed pods claimed |
| T+43s | 2 | Second c8i.2xlarge provisioned |
| T+81s | 3 | Third node provisioned |
| T+203s | 4 | Fourth node provisioned |
| T+291s | 5 | Fifth node provisioned |

Each new node required ~2-3 minutes for kata-deploy DaemonSet to install the Kata runtime before pool pods could run.

### Kata-CLH on Karpenter Nodes

| Metric | Result |
|--------|--------|
| Guest kernel | 6.18.15 |
| Host kernel | 6.1.166-197.305.amzn2023.x86_64 |
| Runtime | kata-clh (cloud-hypervisor) |
| Pool pods per node | 7 (bufferMin=5, bufferMax=10) |

### Warm Pool Claim Latency

| Run | Latency |
|-----|---------|
| 1 | 11,691ms |
| 2 | 11,843ms |
| 3 | 11,223ms |
| 4 | 11,511ms |
| 5 | 11,458ms |
| **Average** | **11,545ms** |

Note: Latency is dominated by OpenSandbox controller reconciliation interval (~10s). Actual pool-to-pod allocation is near-instant. This is not a Kata or Karpenter bottleneck — it can be improved by tuning the controller's reconciliation frequency.

### Batch Claim

| Test | Latency |
|------|---------|
| Batch 5 claim | 11,275ms (all 5 ready simultaneously) |

## Cost Comparison

| Configuration | Instance | Cost/hr | Nested Virt | Auto-Scale | Notes |
|---------------|----------|---------|-------------|------------|-------|
| Bare Metal | c5.metal (96 vCPU) | $5.14 | N/A (HW) | No | Fixed capacity |
| Self-Managed | c8i.4xlarge (16 vCPU) | $0.94 | Yes | No | Manual ASG |
| **Karpenter** | **c8i.2xlarge (8 vCPU)** | **$0.47** | **Yes** | **Yes** | **Auto-scale, right-sized** |

Karpenter advantage: automatically selects the smallest instance that fits workload demand, scales to zero when idle.

## Known Limitations

1. **PR #9043 not merged**: Requires custom Karpenter build. Monitor the PR for upstream merge.
2. **WellKnownLabels bug**: Must apply the fix described above. Consider submitting as a follow-up PR.
3. **kata-deploy startup delay**: New Karpenter nodes need ~2-3 minutes for kata-deploy to install. Pool pods scheduled before installation will fail and be retried.
4. **Warm claim latency**: ~11s due to OpenSandbox controller reconciliation interval. Can be improved by tuning `--sync-period` on the controller.
5. **AZ selection**: Karpenter spreads across AZs (1a, 1c, 1d). Ensure subnets in all AZs are tagged with `karpenter.sh/discovery`.

## Cluster Cleanup

```bash
# Delete OpenSandbox resources
kubectl -n opensandbox-system delete pool kata-clh-pool
kubectl delete nodepool kata-nested-virt
kubectl delete ec2nodeclass nested-virt-kata

# Uninstall Helm charts
helm uninstall opensandbox -n opensandbox-system
helm uninstall kata-deploy -n kube-system
helm uninstall karpenter -n kube-system

# Delete EKS cluster
eksctl delete cluster --name sandbox-karpenter-nv --region ap-northeast-1

# Delete Karpenter CloudFormation stack
aws cloudformation delete-stack --stack-name Karpenter-sandbox-karpenter-nv --region ap-northeast-1

# Delete ECR images
aws ecr batch-delete-image --repository-name karpenter/controller \
  --image-ids imageTag=nested-virt-fix imageTag=nested-virt imageTag=nested-virt-debug \
  --region ap-northeast-1
```
