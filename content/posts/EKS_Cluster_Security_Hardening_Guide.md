---
date: 2025-10-05
author: mingyu110
title: "Amazon EKS 集群安全：常见攻击与生产环境加固实践"
keywords:
  - EKS
  - Kubernetes
  - Security
  - IAM
  - IRSA
  - IMDS
  - 安全加固
category: "AWS 安全"
description: "分析常见的EKS攻击，如IMDS凭证窃取和IRSA滥用，并提供生产环境下的核心加固建议与实践。"
---

# Amazon EKS 集群安全：常见攻击与生产环境加固实践

## 摘要

Amazon EKS (Elastic Kubernetes Service) 极大地简化了在 AWS 上部署和管理 Kubernetes 的复杂性。然而，根据 [AWS 的责任共担模型](https://aws.amazon.com/cn/compliance/shared-responsibility-model/)，虽然 AWS 负责管控控制平面的安全，但客户依然需要负责保护数据平面（工作节点）以及配置 IAM 和 Kubernetes 权限的安全性。配置不当可能导致严重的安全漏洞，允许攻击者提升权限并访问敏感数据。

本文档基于对常见 EKS 攻击模式的分析，深度解析了三种典型的权限提升攻击向量，并为在生产环境中加固 EKS 集群提供了具体、可操作的最佳实践和建议。

---

## 一、 常见 EKS 攻击向量深度解析

攻击者在获得对集群中某个 Pod 的初始访问权限后，通常会尝试利用环境中的配置弱点来横向移动或垂直提升权限。以下是三种常见的攻击路径。

### 1.1 风险一：通过 IMDS 窃取节点凭证

*   **漏洞描述**: 在默认配置下，运行在 EC2 工作节点上的 Pod 可以访问该节点的实例元数据服务（IMDS）。IMDSv1 尤其容易受到攻击，因为它无需特殊的头部信息即可访问。攻击者可以利用此通道，从 Pod 内部窃取附加到该 EC2 节点的 IAM 角色所拥有的临时凭证。
*   **攻击步骤**:
    1.  攻击者在 Pod 内执行 `curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME` 命令。
    2.  IMDS 返回包含 `AccessKeyId`, `SecretAccessKey`, 和 `SessionToken` 的 JSON 对象。
    3.  攻击者使用这些窃取到的凭证，通过 AWS CLI/SDK 执行该节点 IAM 角色所允许的任何 AWS API 操作，例如访问 ECR、S3 或其他服务。

### 1.2 风险二：滥用宽松的 IRSA 信任策略

*   **漏洞描述**: IAM Roles for Service Accounts (IRSA) 是将 IAM 角色关联到 Kubernetes 服务账户（Service Account）的首选方式。其安全性依赖于 IAM 角色信任策略的正确配置。如果信任策略过于宽松，例如没有严格校验请求令牌的来源服务账户，攻击者就可以利用一个低权限的服务账户的令牌，去代入一个高权限的 IAM 角色。
*   **攻击步骤**:
    1.  攻击者在集群中发现一个高权限的 IAM 角色（例如 `S3-Admin-Role`）被配置用于 IRSA。
    2.  攻击者检查其信任策略，发现该策略只校验了 OIDC 提供者，但没有对具体的服务账户（`sub` 声明）进行条件限制。
    3.  攻击者利用自己所控制的、另一个不相关的服务账户（例如 `debug-sa`）来创建一个 Web Identity Token。
    4.  攻击者使用这个 `debug-sa` 的令牌成功调用 `sts:AssumeRoleWithWebIdentity` 操作，代入了高权限的 `S3-Admin-Role`，从而获得了对 S3 的管理权限。

### 1.3 风险三：通过 `imagePullSecrets` 泄露镜像仓库凭证

*   **漏洞描述**: 当 Kubernetes 需要从私有镜像仓库拉取镜像时，通常会使用一个名为 `imagePullSecrets` 的 Secret 来存储认证凭证。如果一个 Pod 的服务账户被错误地授予了读取或列出 Secret 的 RBAC 权限，攻击者就可以轻易地获取这些凭证。
*   **攻击步骤**:
    1.  攻击者在 Pod 内使用 `kubectl get pods -o yaml` 或 `kubectl describe pod POD_NAME` 来发现 Pod 定义中引用的 `imagePullSecrets` 的名称。
    2.  如果 RBAC 权限允许，攻击者执行 `kubectl get secret SECRET_NAME -o yaml` 来读取该 Secret 的内容。
    3.  Secret 中的凭证通常是 Base64 编码的，攻击者解码后即可获得私有镜像仓库的用户名和密码/令牌，从而获得对该仓库的访问权限，可以拉取任意镜像进行离线分析，寻找更多漏洞。

---

## 二、 生产环境核心加固建议

针对上述攻击向量，应在生产环境中实施纵深防御策略。

### 2.1 阻断 Pod 对 IMDS 的访问

*   **首选方案**: 应用 Kubernetes 网络策略（Network Policy），明确禁止从业务 Pod 到 IMDS 端点 `169.254.169.254` 的出站（Egress）流量。这是最直接有效的隔离方法。
*   **补充方案**: 在节点的启动模板中，强制要求使用 IMDSv2，并将 `http-put-response-hop-limit` 设置为 `1`。这使得只有直接在节点上运行的进程（而不是容器内的进程）才能访问元数据服务，极大地增加了从 Pod 内部访问的难度。

### 2.2 强化 IRSA 信任策略的条件

*   **强制要求**: 在为 IRSA 配置的 IAM 角色信任策略中，**必须**添加一个 `Condition` 块，以严格校验 OIDC 令牌的 `sub` (subject) 声明。这确保了只有指定命名空间下的指定服务账户才能代入该角色。
*   **安全策略示例**:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID"
                },
                "Action": "sts:AssumeRoleWithWebIdentity",
                "Condition": {
                    "StringEquals": {
                        "oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID:sub": "system:serviceaccount:NAMESPACE:SERVICE_ACCOUNT_NAME"
                    }
                }
            }
        ]
    }
    ```

### 2.3 遵循 Kubernetes RBAC 的最小权限原则

*   **严格限制**: 默认情况下，不应为服务账户授予任何不必要的权限。特别是 `list` 和 `get` secrets, `create serviceaccounts/token` 等高风险权限，只应在绝对必要时才授予。
*   **定期审计**: 使用 `kubectl auth can-i` 等工具或开源的 RBAC 可视化工具，定期审计服务账户和用户的权限，确保其符合最小权限原则。

### 2.4 使用网络策略实现微隔离

*   **默认拒绝**: 在集群中实施默认“全部拒绝”（deny-all）的网络策略，然后根据业务需要，以白名单的方式明确允许必要的 Pod 间通信和出站访问。这可以限制攻击者在集群内部的横向移动能力。

### 2.5 启用 Kubernetes Secret 静态加密

*   **增强防御**: 在 EKS 集群上启用“信封加密”（Envelope Encryption）功能，使用 AWS KMS 客户主密钥（CMK）来加密存储在 etcd 中的 Kubernetes Secret。这为敏感数据提供了额外的静态加密保护层，即使攻击者直接访问了 etcd 也无法解密 Secret 内容。

---

## 三、 常见问题与解答 (Q&A)

*   **问：我们已经在使用私有镜像仓库，为什么还需要担心 `imagePullSecrets` 的安全？**

    **答：** 因为攻击的起点往往是集群内部的一个已被攻破的 Pod。即使仓库是私有的，如果攻击者从 Pod 内部窃取了 `imagePullSecrets`，他们就获得了访问该私有仓库的钥匙。这允许他们拉取您的所有应用程序镜像进行离线静态分析，寻找代码漏洞、硬编码的密钥或其他敏感信息，为下一步攻击做准备。

*   **问：在节点上强制使用 IMDSv2 是否足以防止凭证被窃取？**

    **答：** 强制使用 IMDSv2 是一个巨大的进步，因为它能有效防御 SSRF 等攻击。但是，如果其 `hop-limit` 设置大于1（例如，默认值为2），容器内的 Pod 仍然可能访问到它。因此，最安全的做法是“纵深防御”：既强制使用 IMDSv2 并将 `hop-limit` 设为1，**同时**也使用网络策略从网络层面彻底阻断 Pod 对 IMDS 端点的访问。这样即使一种机制失效，另一层防御依然有效。

*   **问：如果我们不使用 IRSA，是否就能避免相关的安全风险？**

    **答：** 您会避免 IRSA 特有的信任策略滥用风险，但这通常意味着您会采用一种更不安全的方式来为 Pod 提供 AWS 权限，例如将长期的 IAM 用户凭证作为环境变量或文件挂载到 Pod 中。这种方式的风险要大得多，因为这些凭证是静态的、长期的，一旦泄露，危害更大。IRSA 的设计初衷正是为了通过提供自动轮换的、短期的、与特定服务账户绑定的凭证来提升安全性。因此，正确的做法不是弃用 IRSA，而是**安全地使用它**，即强制执行严格的信任策略条件。
