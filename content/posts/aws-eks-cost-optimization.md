---
title: "AWS EKS 节点选型与自动化弹性成本优化最佳实践"
date: 2025-05-08
draft: false
tags: ["AWS", "EKS", "Kubernetes", "成本优化", "Karpenter", "Spot实例"]
categories: ["云原生"]
---

# AWS EKS 节点选型与自动化弹性成本优化最佳实践

> 随着越来越多的应用容器化以后开始部署在云服务商提供的托管Kubernetes集群，对于成本优化与性能平衡之间的考虑就变得很重要。基于自己的实践与调研，我总结了这篇文档，以AWS EKS为例，分享在云原生环境下的成本优化最佳实践。

## 目录

1. [EKS 节点类型选择策略](#1-eks-节点类型选择策略)
2. [Karpenter 与 Spot 实例成本优化](#2-karpenter-与-spot-实例成本优化)
3. [业务场景实践案例](#3-业务场景实践案例)
4. [基础设施自动化与代码实践](#4-基础设施自动化与代码实践)
5. [Spot 实例优雅终止最佳实践](#5-spot-实例优雅终止最佳实践)
6. [总结与建议](#6-总结与建议)

## 1. EKS 节点类型选择策略

在 AWS EKS 集群中，节点类型的选择对性能、成本和运维复杂度有直接影响。本节分析三种主要节点类型的适用场景。

### 1.1 EC2 节点组

**适用场景：**

- 需要对底层环境进行自定义（内核参数、AMI、驱动等）
- 长期稳定运行的工作负载
- 需要使用 GPU 或其他特殊硬件
- 有特定的合规或安全要求

**优势：**

- 完全控制节点配置
- **支持 Reserved/Spot 实例降低成本**
- 灵活的网络和存储配置

**劣势：**
- 需要自行管理节点生命周期
- 运维复杂度较高

### 1.2 Fargate

**适用场景：**

- 无需管理底层基础设施
- 工作负载波动大、短时任务多
- 多租户环境下需要强隔离
- 开发测试环境或试验性项目

**优势：**
- 无需管理节点
- 按 Pod 计费，避免资源浪费
- 更好的安全隔离性

**劣势：**
- 成本较高
- 功能限制（如无法使用特定存储驱动）
- 启动速度较慢

### 1.3 Karpenter 动态节点

**适用场景：**

- 需要极致弹性和自动化的节点管理
- 多样化实例类型需求
- 对资源利用率和成本优化要求高
- 负载波动大的生产环境

**优势：**
- 自动选择最优性价比实例
- 支持 Spot/On-Demand 混合调度
- 无需预定义节点组
- 更高的资源利用率

**劣势：**
- 配置相对复杂
- 需要额外的权限和组件

关于Karpenter的更多介绍，可以参考Karpenter的官方文档，我推荐在AWS、Azure、阿里云上在生产环境考虑使用Karpenter。

## 2. Karpenter 与 Spot 实例成本优化

### 2.1 Spot 实例简介

Spot 实例是 AWS 提供的闲置计算资源，价格通常比 On-Demand 实例低 70-90%，但可能随时被回收（提前 2 分钟通知）。

### 2.2 Karpenter 如何优化 Spot 实例使用

Karpenter 通过以下机制优化 Spot 实例的使用：

1. **智能实例选择**：自动选择可用性高、中断风险低的 Spot 实例类型
2. **多实例类型支持**：可同时使用多种实例类型，降低整体中断风险
3. **自动替换**：当 Spot 实例收到中断通知，自动创建新节点并迁移工作负载
4. **资源利用率优化**：根据 Pod 资源请求选择最合适的实例大小

### 2.3 EventBridge 与 Lambda 的协同作用

EventBridge 结合 Lambda 可以实现：

1. **定时扩缩容**：根据时间模式自动调整集群规模
2. **事件驱动扩容**：响应特定事件（如队列积压）自动扩容
3. **成本监控与优化**：定期分析资源使用情况，自动调整配置

## 3. 业务场景实践案例

### 3.1 高并发 Web 应用

**场景描述**：
电商网站的 Web 前端服务，流量波动大，有明显的高峰低谷。

**解决方案**：
- 使用 Karpenter 动态调度 Spot 实例

- **设置 Pod 优雅终止机制，确保请求完成**（**重要**）

  - 比如Python应用代码的优雅处理函数

    ```python
    def graceful_shutdown(signum, frame):
        """优雅终止处理函数"""
        global shutdown_requested
        print(f"收到信号 {signum}，开始优雅关闭...")
        shutdown_requested = True
    # 等待活跃请求完成
    def wait_for_requests():
        while active_requests > 0:
            print(f"等待 {active_requests} 个活跃请求完成...")
            time.sleep(1)
        print("所有请求已完成，关闭应用")
        sys.exit(0)
    
    # 启动后台线程等待请求完成
    threading.Thread(target=wait_for_requests).start()
    
    # 注册信号处理器
    signal.signal(signal.SIGTERM, graceful_shutdown)
    ```


  - 示范Pod的yaml配置参考      

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web-app
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: web-app
      template:
        metadata:
          labels:
            app: web-app
        spec:
          # 设置合理的优雅终止时间
          terminationGracePeriodSeconds: 60
     # 配置 Pod 拓扑分布约束，避免单点故障
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web-app
      containers:
      - name: web
            image: your-app-image:latest
            ports:
            - containerPort: 8080
            
            # 配置健康检查
            readinessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 5
              periodSeconds: 10
              
            # 配置 PreStop 钩子
            lifecycle:
              preStop:
                exec:
                  command: ["/bin/sh", "-c", "sleep 10 && /app/shutdown.sh"]
    ```

- 使用 EventBridge + Lambda 在低谷期自动缩容

**技术架构**：

- 前端服务部署在 Karpenter 管理的 Spot 实例上
- 数据库等有状态服务部署在 On-Demand 实例上
- 使用 HPA 进行 Pod 级别的自动扩缩容

### 3.2 API 网关服务

**场景描述**：
企业内部 API 网关，需要高可用性，但预算有限。

**解决方案**：
- Karpenter 管理的 Spot/On-Demand 混合实例

- 多可用区部署，确保高可用性

- **使用 `PodDisruptionBudget` 保证服务连续性**（**重要**）

  - 示范配置参考

    ```yaml
    apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: web-app-pdb
    spec:
      minAvailable: 2  # 或使用 maxUnavailable
      selector:
        matchLabels:
          app: web-app
    ```

**技术架构**：

- 网关服务使用 Deployment 部署，设置适当的 replicas
- 配置 Karpenter 优先选择 Spot 实例，但保留部分 On-Demand 实例作为保底
- 实现健康检查和自动恢复机制

### 3.3 批处理任务

**场景描述**：
每日数据处理和报表生成任务，对时间不敏感。

**解决方案**：
- 使用 Fargate 或 Karpenter + Spot 实例
- EventBridge 定时触发任务
- 实现任务重试和断点续传机制

**技术架构**：
- 使用 Kubernetes Job 或 CronJob 资源
- 数据处理结果存储在 S3 或 EFS
- Lambda 监控任务完成状态，失败时自动重试

## 4. 基础设施自动化与Lambda代码参考

GitHub地址：[eks-cost-optimization](https://github.com/mingyu110/AWS/tree/main/eks-cost-optimization)

## 5.总结与建议

### 5.1 EKS 节点选型建议

- **混合使用策略**：核心服务使用 On-Demand 实例，弹性部分使用 Spot 实例。
- **根据业务特性选择**：有状态服务优先考虑 On-Demand，无状态服务可优先使用 Spot。
- **Fargate 适用于**：隔离性要求高、短时运行、运维资源有限的场景。
- **Karpenter 适用于**：需要极致弹性、多样化实例类型、高资源利用率的场景。

### 5.2 成本优化最佳实践

- **合理设置资源请求**：避免资源请求过高导致浪费。
- **使用 Spot 实例**：对于可中断工作负载，优先使用 Spot 实例，但也需要代码保证应用优雅终止。
- **自动扩缩容**：根据实际负载自动调整集群规模。
- **定时扩缩容**：根据业务规律，在低峰期自动缩容。
- **实例多样化**：使用多种实例类型，提高 Spot 可用性并降低成本。

### 5.3 自动化与运维建议

- **基础设施即代码**：使用 Terraform 管理所有基础设施。
- **独立管理 Lambda 代码**：便于快速迭代和调试。
- **监控与告警**：实时监控 Spot 实例状态和应用健康状况。
- **自动化测试**：定期测试优雅终止机制，确保生产环境可靠性。
- **文档与知识共享**：记录最佳实践和经验教训，促进团队知识共享。
