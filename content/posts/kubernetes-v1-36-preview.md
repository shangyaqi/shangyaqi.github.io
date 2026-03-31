---
title: "Kubernetes v1.36 预览：重要功能与变更解析"
date: 2026-03-31
draft: false
tags: ["Kubernetes", "容器编排", "DevOps", "云原生"]
categories: ["DevOps"]
summary: "详解即将于2026年4月发布的Kubernetes v1.36版本中的重要功能增强、废弃与移除，包括SELinux卷标记加速、动态资源分配增强和安全改进。"
ShowToc: true
TocOpen: true
---

Kubernetes v1.36 计划于 **2026年4月22日** 正式发布。作为云原生领域的基础设施标准，每个新版本都值得运维和开发人员密切关注。本文将深入分析这个版本中的主要变更，帮助您提前规划升级策略。

<!--more-->

## 版本概述

v1.36 版本包含了多项重要的功能增强、废弃和移除。该版本特别关注了以下几个方面：

1. 安全增强：移除安全风险功能，强化身份验证机制
2. 性能优化：提高启动速度和资源利用率
3. 资源管理：改进动态资源分配，特别是对高价值硬件资源的支持

## 废弃与移除

### Service.spec.externalIPs 废弃

`Service.spec.externalIPs` 字段将被废弃，这意味着您将很快失去一种将任意 externalIPs 路由到 Service 的快捷方式：

- **安全原因**：该字段多年来一直是安全隐患，可能导致集群流量的中间人攻击（[CVE-2020-8554](https://github.com/kubernetes/kubernetes/issues/970760)）
- **时间表**：从 v1.36 开始显示废弃警告，计划在 v1.43 完全移除
- **替代方案**：
  - 对于云托管入口，使用 LoadBalancer 类型的 Service
  - 对于简单端口暴露，使用 NodePort
  - 对于更灵活和安全的外部流量处理，使用 Gateway API

### gitRepo 卷驱动完全移除

`gitRepo` 卷类型自 v1.11 以来一直处于废弃状态，从 Kubernetes v1.36 开始，该卷插件将被永久禁用且无法重新启用：

- **安全问题**：使用 gitRepo 可能允许攻击者以 root 身份在节点上执行代码
- **影响**：尽管 gitRepo 已被废弃多年且推荐了更好的替代方案，但在之前的版本中仍然可以技术上使用它
- **替代方案**：
  - 使用 init 容器
  - 使用外部 git-sync 风格的工具

### Ingress NGINX 退役

为优先考虑生态系统的安全性和安全性，Kubernetes SIG Network 和安全响应委员会已于 2026 年 3 月 24 日退役 Ingress NGINX：

- **影响**：不再有新版本、错误修复或安全漏洞更新
- **现状**：现有 Ingress NGINX 部署将继续运行，安装工件如 Helm charts 和容器镜像将保持可用
- **行动建议**：评估替代入口控制器，选择符合当前安全和维护最佳实践的解决方案

## 重要功能增强

### SELinux 卷标记加速 (GA)

Kubernetes v1.36 将 SELinux 卷挂载改进功能提升为正式发布（GA）状态：

- **技术变更**：用 `mount -o context=XYZ` 选项替代递归文件重标记，在挂载时为整个卷应用正确的 SELinux 标签
- **性能提升**：带来更一致的性能表现，减少 SELinux 强制系统上的 Pod 启动延迟
- **发展历程**：
  - v1.28 引入测试版，适用于 ReadWriteOncePod 卷
  - v1.32 增加了指标和选择退出选项（`securityContext.seLinuxChangePolicy: Recursive`）以帮助捕获冲突
  - v1.36 达到稳定版，默认适用于所有卷，Pod 或 CSIDrivers 通过 `spec.SELinuxMount` 选择加入
- **风险提示**：由于潜在的特权和非特权 Pod 混合使用问题，我们预计此功能在未来 Kubernetes 版本中可能会带来破坏性变更

```yaml
# 示例：在 Pod 中设置 SELinux 策略
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"
    seLinuxChangePolicy: AtMount  # 使用新的挂载时标记方式
  containers:
  - name: app
    image: secure-app:1.0
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

### ServiceAccount 令牌外部签名 (GA)

作为测试版功能，Kubernetes 已经支持 ServiceAccount 令牌的外部签名。这允许集群与外部密钥管理系统或签名服务集成，而不仅仅依赖于内部管理的密钥：

- **功能描述**：kube-apiserver 可以将令牌签名委托给外部系统，如云密钥管理服务或硬件安全模块
- **优势**：
  - 提高安全性
  - 简化依赖集中式签名基础设施的集群的密钥管理服务
- **状态**：预计在 Kubernetes v1.36 中升级为稳定版（GA）

## 动态资源分配 (DRA) 增强

### 设备污点和容忍度支持 (Beta)

Kubernetes v1.33 引入了通过[动态资源分配 (DRA)](/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) 管理的物理设备的污点和容忍度支持：

- **功能描述**：
  - 允许 DRA 驱动程序将设备标记为污点，确保不会用于调度
  - 集群管理员可以创建 DeviceTaintRule 来标记匹配特定选择标准的设备
- **优势**：改进调度控制，确保专用硬件资源仅被显式请求它们的工作负载使用
- **状态**：在 Kubernetes v1.36 中升级为 Beta 版本，默认可用，无需特性标志

```yaml
# 示例：为 GPU 设备创建污点规则
apiVersion: resource.k8s.io/v1alpha2
kind: DeviceTaintRule
metadata:
  name: gpu-reserved-for-ml
spec:
  deviceSelector:
    driverName: gpu.example.com
    model: "A100"
  taints:
  - key: "ml.example.com/dedicated"
    value: "true"
    effect: "NoSchedule"
```

### 可分区设备支持

Kubernetes v1.36 扩展了动态资源分配 (DRA)，引入了对可分区设备的支持：

- **功能描述**：允许将单个硬件加速器分割为多个逻辑单元，可在工作负载之间共享
- **适用场景**：特别适用于 GPU 等高成本资源，避免将整个设备专用于单个工作负载导致的利用率不足
- **优势**：
  - 平台团队可以通过仅为每个工作负载分配设备所需部分，提高集群整体效率
  - 在保持隔离和控制的同时，支持在同一硬件上运行多个工作负载
  - 帮助组织从基础设施中获得更多价值

```yaml
# 示例：请求 GPU 分区
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  containers:
  - name: training
    image: ml-framework:latest
    resources:
      claims:
      - name: gpu-partition
  resourceClaims:
  - name: gpu-partition
    spec:
      resourceClassName: gpu.nvidia.com
      parametersRef:
        apiGroup: gpu.resource.example.com
        kind: GpuClaimParameters
        name: gpu-partition-params
---
apiVersion: gpu.resource.example.com/v1
kind: GpuClaimParameters
metadata:
  name: gpu-partition-params
spec:
  memoryGB: 16    # 请求 16GB GPU 内存
  computeUnits: 4 # 请求 4 个计算单元
```

## 对企业的影响

### 安全加固

- **移除不安全功能**：externalIPs 和 gitRepo 的变更表明 Kubernetes 持续关注安全性
- **身份验证增强**：ServiceAccount 令牌外部签名提供了与企业密钥管理系统集成的更好方式

### 性能优化

- **启动加速**：SELinux 卷标记改进将显著减少 Pod 启动延迟，特别是在大规模部署中
- **资源效率**：DRA 的可分区设备支持将提高 GPU 等昂贵资源的利用率，降低基础设施成本

### 迁移需求

- **代码审查**：使用 externalIPs 和 gitRepo 卷的工作负载需要迁移
- **入口控制器**：由于 Ingress NGINX 退役，组织应评估替代方案

## 行动建议

1. **审计现有工作负载**：检查是否有使用即将废弃或移除的功能的工作负载
2. **测试兼容性**：在非生产环境中测试应用程序与 v1.36 的兼容性
3. **规划迁移**：为依赖 gitRepo 卷、externalIPs 或 Ingress NGINX 的应用制定迁移计划
4. **探索 DRA 增强功能**：评估可分区设备支持如何优化 GPU 工作负载部署
5. **更新安全策略**：考虑利用 ServiceAccount 令牌外部签名增强集群安全态势

## 总结

Kubernetes v1.36 的这些变更反映了项目向更安全、更高效和更灵活的容器编排平台演进的持续努力。特别是在资源管理、安全性和性能方面的改进，将为企业级部署提供显著价值。

随着发布日期的临近，建议密切关注官方发布说明和文档，以获取最终版本中可能出现的任何变更。

## 参考链接

- [Kubernetes v1.36 Sneak Peek](https://kubernetes.io/blog/2026/03/30/kubernetes-v1-36-sneak-peek/)
- [KEP-1710: Speed up recursive SELinux label change](https://kep.k8s.io/1710)
- [KEP-740: Support external signing of service account tokens](https://kep.k8s.io/740)
- [KEP-5055: DRA: device taints and tolerations](https://kep.k8s.io/5055)
- [KEP-4815: DRA Partitionable Devices](https://kep.k8s.io/4815)
- [Kubernetes v1.36 CHANGELOG](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md)