# Progress Log

## Session: 2026-04-13 — EKS + Karpenter (嵌套虚拟化) + Kata-CLH + OpenSandbox 验证
- 从 PR #9043 构建自定义 Karpenter (添加 cpuOptions.nestedVirtualization 字段)
- 创建 EKS 集群 sandbox-karpenter-nv + Karpenter IAM (CloudFormation)
- 部署自定义 Karpenter, 应用再生成的 CRD (controller-gen)
- **发现并修复关键 bug**: PR #9043 未将 `LabelInstanceNestedVirtualization` 注册到 `WellKnownLabels`，导致 CompatibleAvailableFilter 拒绝所有支持嵌套虚拟化的实例类型
- 修复后 Karpenter 成功自动配置 c8i.2xlarge (嵌套虚拟化已启用)
- 验证: /dev/kvm 可用, vmx CPU flag 存在, guest kernel 6.18.15
- Karpenter 自动从 1 节点扩展到 5 节点 (20 replicas BatchSandbox)
- Warm Pool Claim 平均延迟: ~11.5s (受 OpenSandbox 控制器对账间隔限制)
- 输出文档: docs/karpenter-nested-virt-verification.md
- 成本: c8i.2xlarge $0.47/hr (比 c5.metal $5.14/hr 降低 91%)

## Session: 2026-04-09 — Per-Session EFS 隔离方案研究
- 用户提出需求: Pool Pod 间 EFS 数据不共享，仅升级时新旧 sandbox 定向共享
- 分析 OpenSandbox Pool 源码，确认 per-pod 卷定制不可行
- 提出三种方案: A(Agent 目录隔离)、B(容器内 NFS mount)、C(非 Pool ShardPatches)
- 用户质疑"为什么 per-session PVC 不能用预热" → 发现容器内 NFS mount 可绕过限制
- **下一步**: 研究 Kata VM 内 NFS mount 可行性，部署验证

## Session: 2026-04-06 — Blue-Green + Shared PVC 升级验证
- 部署 EKS + Kata-CLH + OpenSandbox 在 ap-northeast-1
- 创建 EFS + 静态 PV/PVC (动态 provisioning 因 IAM 权限问题改为静态)
- 验证 Blue-Green 双 Pool 并行 + 任务边界迁移
- 所有 8 项验证通过，输出 docs/upgrade-verification.md
- 精简 sandbox-upgrade.md: 移除 S3 优雅排空 + 方案 B Ingress 路由
- 加入 per-session 目录隔离模型 (应用层)
- 清理 AWS 资源

## Session: 2026-03-30 — 部署验证 + 文档
- 实际部署验证 EKS + Kata-CLH + OpenSandbox
- Pool Claim: avg 277ms, Batch 10: 241ms, Cold Start: avg 3209ms

## Session: 2026-03-29 — 5 方案 POC
- 测试 5 种 sandbox 方案，推荐 OpenSandbox + Kata-CLH

## Error Log
| Timestamp | Error | Resolution |
|-----------|-------|------------|
| 2026-04-06 | EFS dynamic provisioning Access Denied | 改用静态 PV 直接引用 EFS FileSystem ID |
| 2026-04-06 | Helm 模板孤立 {{- end }} | sed 删除 lines 55-56 |
| 2026-03-30 | eksctl delete cluster timeout | 等待 CloudFormation stack 完成后重试 |
| 2026-03-29 | OpenSandbox kube-client-qps crash | 删除 Helm 模板中不支持的参数 |

## 5-Question Reboot Check
| Question | Answer |
|----------|--------|
| Where am I? | Phase 1: 研究 per-session EFS 文件系统级隔离方案 |
| Where am I going? | 验证容器内 NFS mount + EFS Access Point 的可行性 |
| What's the goal? | 在保留 Pool 预热的前提下实现文件系统级 sandbox 数据隔离 |
| What have I learned? | Pool 不支持 per-pod 卷定制，但可通过容器内 NFS mount 绕过 |
| What have I done? | 分析源码、提出 3 种方案、用户确认方向 B |

---
*Last updated: 2026-04-09*
