---
title: "Istio实践指南 之 最佳实践（一） Istio服务网格中实现优雅终止的技术指南"
date: 2025-04-30T10:30:00+08:00
draft: false
tags: ["云原生", "Istio", "服务网格"]
categories: ["云原生"]
---

### Istio实践指南 之 最佳实践（一） Istio服务网格中实现优雅终止的技术指南

#### 1. 概述

在微服务架构中，优雅终止（Graceful Shutdown）是确保服务质量和系统稳定性的关键机制。当使用 Istio 服务网格时，由于引入了 Envoy 代理作为 sidecar，优雅终止的实现变得更加复杂，需要特别关注。本文档详细介绍 Istio 环境中优雅终止的工作原理、常见问题及最佳实践解决方案。

#### 2. Istio中的终止流程

##### 2.1 标准终止流程

当 Kubernetes 需要终止一个 Pod 时，会按照以下顺序执行操作：

1. Pod 被标记为 `Terminating` 状态

2. Pod 从服务的 Endpoints 列表中移除，不再接收新的流量

3. Pod 内的容器接收 `SIGTERM` 信号

4. 容器进入宽限期（默认 30 秒）

5. 若宽限期结束后容器仍未退出，发送 `SIGKILL` 信号强制终止

##### 2.2 Istio Sidecar 特有的终止行为

在 Istio 环境中，终止流程存在以下特殊情况：

1. 当 Pod 开始停止时，Kubernetes 将其从服务的 Endpoints 中移除
2. Envoy sidecar 同时接收到 `SIGTERM` 信号
3. Envoy 立即停止接受新的 `inbound` 连接，但会继续处理现有连接
4. `Outbound` 方向的流量仍然可以正常发起

**关键问题**：默认情况下，Istio 会在 Pod 停止开始的 5 秒后强制终止 Envoy，无论其是否完成了所有连接的处理

#### 3. 技术原因分析

#### 3.1 Envoy 被强制终止导致的异常

Envoy 被强制终止会引发几种典型问题：

1. 长耗时请求中断：当服务处理的请求本身需要较长时间（如媒体转换、大数据处理等），现有的 `inbound` 请求可能在完成前被中断

2. 依赖服务调用失败：如果服务在终止过程中需要调用其他服务（如状态同步、资源清理等），`outbound` 请求可能因 Envoy 被终止而失败

3. 连接泄漏：客户端可能无法及时感知连接已断开，导致连接资源泄漏

#### 3.2 技术原因分析

这些问题的根本原因在于：

- Istio 默认配置了 terminationDrainDuration 为 5 秒

- 该设置会使 istio-agent 在收到终止信号 5 秒后强制终止 Envoy

- 此时间可能不足以完成所有在途请求的处理

### 4. 解决方案

#### 4.1 启用 EXIT_ON_ZERO_ACTIVE_CONNECTIONS

自 **Istio 1.12 版本**开始，引入了 EXIT_ON_ZERO_ACTIVE_CONNECTIONS 环境变量来解决此问题：

- 当启用此选项后，Envoy 将等待所有活跃连接处理完毕后才会退出

- 同时会通知客户端关闭长连接：

- 对 HTTP/1.1 请求，返回 Connection: close 头

- 对 HTTP/2 请求，发送 GOAWAY 帧

#### 4.2 全局配置方式

修改 Istio 的全局配置 ConfigMap（通常是 `istio-sidecar-injector` 或 `istio`），在 defaultConfig.proxyMetadata 下添加：

```yaml
defaultConfig:
  proxyMetadata:
    EXIT_ON_ZERO_ACTIVE_CONNECTIONS: "true"
```

#### 4.3 针对特定工作负载配置

为特定 Pod 添加注解：

```yaml
annotations:
  proxy.istio.io/config: '{ "proxyMetadata": { "EXIT_ON_ZERO_ACTIVE_CONNECTIONS": "true" } }'
```

#### 4.4 调整 terminationDrainDuration

除了启用 `EXIT_ON_ZERO_ACTIVE_CONNECTIONS`，还可以考虑调整 `terminationDrainDuration` 的值：

```yaml
defaultConfig:

 terminationDrainDuration: 20s *# 默认为 5s*
```

### 5. 验证与测试

#### 5.1 验证配置是否生效

1. 检查 Envoy sidecar 的环境变量

   ```bash
   kubectl exec -it <pod-name> -c istio-proxy -- env | grep EXIT_ON_ZERO
   ```

   

2. 检查Envoy 配置：

```bash
   kubectl exec -it <pod-name> -c istio-proxy -- pilot-agent request GET config_dump
```

#### 5.2 测试优雅终止效果

创建模拟长耗时请求的测试用例，并在请求处理过程中触发 Pod 终止，观察请求是否能够正常完成。

### 6. 最佳实践建议

1. 启用 EXIT_ON_ZERO_ACTIVE_CONNECTIONS：对于所有生产环境的 Istio 部署，推荐启用此特性

2. 设置合理的超时时间：为业务应用配置合理的处理超时，避免请求无限期挂起

3. 实现应用层优雅终止：确保应用本身能够正确处理 SIGTERM 信号，停止接收新请求并完成现有请求

4. 配置合适的就绪探针：使用就绪探针确保流量在应用准备好前不会被路由到 Pod

5. 调整 terminationGracePeriodSeconds：根据业务处理时间特点，合理设置 Pod 的终止宽限期



  **注**：如果是使用了腾讯云的商用服务网格产品TCM，可以在控制台上直接启用

<img src="https://image-host-1251893006.cos.ap-chengdu.myqcloud.com/20230104110821.png" alt="img" style="zoom:80%;" />

### 7. 参考资料

- [Istio 官方文档：流量管理最佳实践](https://istio.io/latest/docs/ops/best-practices/traffic-management/)

- [Kubernetes 文档：Pod 的终止](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)

- [Envoy 文档：优雅终止](https://www.envoyproxy.io/docs/envoy/latest/operations/drain)

通过以上配置和最佳实践，可以有效解决 Istio 环境下 Envoy 被强制终止导致的流量异常问题，确保系统在扩缩容、滚动更新等场景下的稳定性。

 

