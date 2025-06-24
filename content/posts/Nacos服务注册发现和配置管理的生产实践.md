---
title: "Nacos服务注册发现和配置管理的生产实践"
date: 2025-06-24
draft: false
tags: ["Nacos", "服务注册", "配置管理", "微服务", "云原生", "Kubernetes"]
categories: ["云原生"]
description: "深入探讨Nacos在生产环境中的服务注册发现和配置管理最佳实践，包括版本选择、推空保护、与Kubernetes集成等关键特性"
toc: true
author: "Your Name"
---

# Nacos服务注册发现和配置管理的生产实践

## 1. Nacos介绍

- Nacos 是一个由阿里巴巴开发的动态服务发现、配置管理和服务管理平台，广泛用于微服务和云原生应用中。版本从 1.x 到 2.x，再到 3.x，经历了显著的更新和优化

<img src="https://nacos.io/img/nacosMap.jpg" alt="nacos_map" style="zoom:50%;" />

<img src="https://nacos.io/img/doc/overview/availability-structure.svg" alt="Nacos高可用架构图" style="zoom:50%;" />

- 建议限制 Nacos 版本为 2.3.0 或更高，但低于 3.0，以确保稳定性；Nacos目前比较稳定且适合用于生产环境的版本是2.2.0 - 2.5.X版本
- Nacos 3.0 引入了 API 分类、控制台认证默认启用等重大变化，可能需要额外适配工作
- Nacos 3.X版本不再兼容1.X版本的OpenAPI，同时不再兼容2.X版本的HTTP OpenAPI ，参考[Nacos OpenAPI概览](https://nacos.io/docs/latest/manual/user/overview/api-overview)、[客户端API](https://nacos.io/docs/latest/manual/user/open-api)

## 2. Nacos Controller 2.0版本发布满足的场景需求

**背景**：2025年3月26日，Nacos Controller 2.0版本发布，主要满足的用户需求：参考文档：[Nacos 3.0.0 BETA、2.5.1、Nacos Controller 2.0同时发布](https://nacos.io/blog/release-300-beta/)

- 有些用户可能同时使用了Nacos服务发现与K8s服务发现，使用Nacos服务发现的应用希望能够通过Nacos发现K8s集群的服务；
- 应用目前使用K8s的configmap和secret，很方便的通过Nacos管理configmap和secret；

### 2.1 Nacos服务与K8S服务互相发现

Nacos Controller 2.0 支持将Kubernetes集群特定命名空间下的Service同步到Nacos指定命名空间下。用户可以通过Nacos实现对Kubernetes服务的服务发现。以此实现跨K8S集群的服务发现和访问，或实现K8S集群与非K8S集群间的服务发现和访问，解决容灾备份，平滑迁移等一系列高可用，稳定性相关的高级服务发现场景。

<img src="https://nacos.io/img/blog/3_0_0-release/nacos_controller_cross_k8s_situ.svg" alt="img" style="zoom:50%;" />

<img src="https://nacos.io/img/blog/3_0_0-release/nacos_controller_non_k8s_situ.svg" alt="img" style="zoom:50%;" />

### 2.2 Nacos管理K8S configmap和secret

Nacos Controller 2.0 支持Kubernetes集群配置和Nacos 配置的双向同步，将Kubernetes集群特定命名空间下的ConfigMap以及Secret同步到Nacos指定命名空间下中。用户可以通过Nacos实现对于Kubernetes集群配置的修改和管理，以达到ConfigMap和Secret的**<u>动态修改</u>**、**<u>版本管理</u>**、灰度发布等场景。

<img src="https://nacos.io/img/blog/3_0_0-release/nacos_controller_config_gray.svg" alt="img" style="zoom:50%;" />

- Nacos Controller的文档需要参考：[Nacos Controller](https://github.com/nacos-group/nacos-controller/blob/main/README_CN.md)

## 3. Nacos的推空保护

推空保护是一种保护机制，防止服务消费者在注册中心变更（如升级、故障转移）或遇到突发情况（如网络中断）时收到空的实例列表，从而影响业务可用性。

- Nacos客户端在1.4.1以及以上版本支持推空保护设置，通过配置参数启用，例如 通过`properties.put(PropertyKeyConst.NAMING_PUSH_EMPTY_PROTECTION, "true")` 或 `spring.cloud.nacos.discovery.namingPushEmptyProtection=true` 来启用。Nacos服务端可以支持客户端对于推空保护的逻辑

- 商业的Nacos产品例如阿里云MSE Nacos服务端进行了推空保护的设置优化（例如：优化了空列表保护的触发范围，确保只在特定场景（如引擎节点升级或所有实例移除）生效；减少无效日志的生成改善系统性能等），开源的Nacos服务端没有进行相关的推空保护配置优化

  