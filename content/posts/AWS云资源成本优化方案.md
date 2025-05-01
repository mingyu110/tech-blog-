---
title: "AWS云资源成本及合规自动化优化方案"
date: 2025-05-01
draft: false
tags: ["AWS", "云计算", "成本优化", "自动化", "Terraform"]
categories: ["云计算"]
description: "基于Terraform的AWS云资源成本与合规性自动化优化解决方案"
toc: true
---

## 一、功能特性

1. **资源合规监控**
   - 检测未标记的资源（EC2、S3、RDS 等）
   - 检测闲置的 EC2 实例
   - 监控安全组开放端口
   - 实时监控新创建或状态变更的资源

2. **自动化操作**
   - 自动为不合规资源添加标签
   - 自动停止闲置的 EC2 实例
   - 发送实时通知到钉钉

3. **成本异常检测**
   - 每日自动分析 AWS 成本数据
   - 检测成本异常（超过过去 7 天平均值的设定阈值）
   - 发送详细异常报告并提供处理建议

## 二、架构概述

本解决方案采用无服务器架构，主要由以下组件构成：

![架构图](/images/aws-architecture.jpg)

1. **AWS Config**：负责持续监控和记录 AWS 资源配置变更，检测不合规的资源
2. **AWS EventBridge**：处理各类事件触发器，如 Config 规则变更、EC2 状态变更
3. **AWS Lambda**：提供自动化处理逻辑，包括资源合规处理和成本异常检测
4. **AWS SNS**：用于消息通知路由
5. **钉钉通知**：通过自定义的钉钉机器人发送通知和警报

整个解决方案通过 Terraform 实现基础设施即代码（IaC），便于版本控制和持续部署。

## 三、AWS 服务说明

本解决方案使用了以下 AWS 服务：

1. **AWS Config** - 提供 AWS 资源清单、配置历史和配置变更通知的服务，使您能够进行合规性审核、安全性分析、资源变更跟踪和故障排除。

2. **AWS EventBridge** - 无服务器事件总线服务，使应用程序能够轻松地从各种来源接收事件数据并响应这些事件。在本解决方案中用于处理 Config 规则变更和 EC2 状态变更事件。

3. **AWS Lambda** - 无服务器计算服务，允许您运行代码而无需配置或管理服务器。在本解决方案中用于执行资源合规处理和成本异常检测逻辑。

4. **AWS SNS (Simple Notification Service)** - 高度可用、持久、安全、完全托管的发布/订阅消息收发服务，支持分离微服务、分布式系统和无服务器应用程序。在本解决方案中用于通知路由。

5. **AWS Cost Explorer** - 提供了查看和分析 AWS 成本和使用情况的界面，本解决方案通过 API 调用获取成本数据进行异常检测。

6. **Amazon S3 (Simple Storage Service)** - 提供行业领先的可扩展性、数据可用性、安全性和性能的对象存储服务，在本解决方案中用于存储 AWS Config 配置历史和 Terraform 状态文件。

7. **AWS IAM (Identity and Access Management)** - 安全管理 AWS 服务和资源访问的服务，本解决方案使用 IAM 角色和策略确保 Lambda 函数的最小权限原则。

## 四、代码工程结构

完整代码工程请查阅 [Github仓库](https://github.com/mingyu110/AWS/tree/main/AWS-Cost-Optimization)

```
aws-cost-optimization/
├── main.tf                    # 主 Terraform 配置文件
├── variables.tf               # 变量定义文件
├── outputs.tf                 # 输出定义文件
├── terraform.tfvars.example   # 示例变量值文件
├── modules/                   # Terraform 模块目录
│   ├── config/                # AWS Config 相关配置
│   ├── lambda/                # Lambda 函数模块
│   └── eventbridge/           # EventBridge 事件规则
└── lambda/                    # Lambda 函数源码
    ├── resource_compliance/   # 资源合规检查函数
    ├── cost_anomaly/          # 成本异常检测函数
    └── utils/                 # 共享工具函数
```

## 五、部署指南

### 1. 前提条件

1. **AWS 账户**
   - 有足够权限创建和管理 AWS 资源的 IAM 用户或角色
   - 确保 AWS CLI 已配置正确的访问凭证

2. **Terraform 环境**
   - 安装 Terraform (>= 1.0.0)
   - 如使用远程状态，需准备 S3 存储桶存储状态文件

3. **钉钉机器人**
   - 创建钉钉群组和自定义机器人
   - 获取机器人的 Webhook URL 和安全密钥（Secret）

4. **AWS 服务启用**
   - 确保已启用 AWS Config 服务
   - 确保已启用 AWS Cost Explorer API

### 2. 配置说明

1. **创建配置文件**

   将示例配置文件复制为实际配置文件：
   ```bash
   cp terraform.tfvars.example terraform.tfvars
   ```

2. **编辑配置文件**

   编辑 `terraform.tfvars` 文件，填写必要的配置参数：
   ```hcl
   # AWS 区域
   aws_region = "ap-northeast-1"  # 使用您的 AWS 区域，如：东京区域

   # 环境名称
   environment = "prod"  # 如 dev, test, prod

   # 钉钉机器人配置
   dingtalk_webhook = "https://oapi.dingtalk.com/robot/send?access_token=your_access_token"
   dingtalk_secret = "your_dingtalk_secret"

   # 成本异常阈值百分比
   threshold_percentage = 15  # 根据需要调整，默认为 10%
   ```

### 3. 部署步骤

1. **初始化 Terraform**

   ```bash
   terraform init
   ```

2. **验证配置**

   ```bash
   terraform validate
   ```

3. **预览部署计划**

   ```bash
   terraform plan
   ```

4. **执行部署**

   ```bash
   terraform apply
   ```

## 六、使用指南

### 1. 成本异常监控

- 每日自动执行，检测前一天的成本数据
- 异常检测基于过去7天平均值和设定的阈值
- 当检测到异常时，会通过钉钉发送详细报告
- 报告包含异常服务、超出金额及百分比、处理建议

### 2. 资源合规管理

- 实时监控资源创建和状态变更
- 自动检测未标记资源并添加默认标签
- 检测闲置EC2实例并可选择自动停止
- 监控安全组配置，发现潜在安全问题

### 3. 钉钉通知设置

- 支持自定义通知模板和触发条件
- 可按资源类型和严重性级别过滤通知
- 提供丰富的消息格式，包括Markdown和ActionCard
- 支持@相关负责人进行紧急处理

## 七、注意事项

1. **权限设置**
   - 确保Lambda函数拥有足够但最小必要的权限
   - Config服务需要特定权限记录AWS资源配置

2. **成本考量**
   - AWS Config服务本身会产生费用
   - Lambda调用和日志存储会产生少量费用
   - 建议设置成本预算避免意外支出

3. **故障排除**
   - 检查CloudWatch日志获取详细错误信息
   - 验证IAM角色权限是否正确配置
   - 确认钉钉Webhook URL和Secret是否有效

## 八、未来扩展

1. **多账户支持**
   - 支持AWS组织中多账户的集中管理
   - 跨账户成本分析和资源优化

2. **智能推荐**
   - 基于使用模式自动推荐Savings Plans和Reserved Instances
   - 提供自动扩缩建议和实例类型优化

3. **更多资源类型支持**
   - 扩展到更多AWS服务，如Lambda、DynamoDB等
   - 支持更复杂的合规性检查规则

4. **自定义仪表板**
   - 提供Web界面展示成本趋势和优化建议
   - 支持自定义报表和导出功能 