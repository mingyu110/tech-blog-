---
title: "AWS Site-to-Site VPN 预共享密钥（PSK）安全管理最佳实践"
date: 2025-09-07
draft: false
tags: ["AWS", "IPSec", "VPN","PSK"]
categories: ["Cloud Architecture"]
---

# AWS Site-to-Site VPN 预共享密钥（PSK）安全管理最佳实践

本文面向网络管理员、云架构师和安全专家，旨在深入探讨如何利用 AWS Secrets Manager 集中管理 AWS Site-to-Site VPN 的预共享密钥（Pre-Shared Key, PSK），并结合详细的配置步骤和技术原理，以显著提升企业混合云环境的安全性、合规性和运维效率。

---

### 1. 真实应用场景介绍

在现代企业 IT 架构中，混合云已成为主流模式。企业通常需要将其本地数据中心（On-Premises）或分支机构网络与 AWS 云上的虚拟私有云（VPC）进行安全、可靠的连接。AWS Site-to-Site VPN 是实现这一目标的核心服务，它通过在公共互联网上建立加密的 IPsec 隧道，将企业私有网络无缝扩展至 AWS。

**典型场景包括：**

*   **混合云网络互联：** 将本地数据中心的服务器与云上应用（如数据库、计算集群）连接，构建统一的逻辑网络。
*   **灾备与备份：** 将本地数据安全地备份或复制到 AWS S3、EBS 等存储服务。
*   **分支机构接入：** 允许企业在全球各地的分支办公室安全地访问部署在 AWS 上的内部应用和服务。

在所有这些场景中，IPsec 隧道的身份验证依赖于预共享密钥（PSK）。如何安全、高效地管理这些密钥，是确保混合云网络安全的关键一环。

---

### 2. 技术挑战和解决方案

传统的 PSK 管理方式存在诸多挑战，直接影响了企业的安全基线和运维效率。

#### 2.1 技术挑战

*   **明文存储与硬编码风险：** PSK 常常以明文形式存储在网络设备的配置文件、自动化脚本（如 Terraform, Ansible）或代码仓库中。这种方式极易导致密钥意外泄露，一旦泄露，攻击者便可尝试建立非授权的 VPN 连接。
*   **缺乏集中管控与审计：** 当企业拥有数十甚至上百个 VPN 连接时，PSK 分散在各处，无法集中查看和管理。同时，对密钥的访问、修改等操作缺乏有效的审计日志，难以满足合规性要求（如 PCI-DSS, ISO 27001）。
*   **密钥轮换困难且风险高：** 安全最佳实践要求定期轮换密钥。手动轮换过程繁琐、易出错，且需要协调本地和云端的同时变更，操作风险高，导致许多企业未能有效执行密钥轮换策略。

#### 2.2 解决方案：集成 AWS Secrets Manager

为应对上述挑战，AWS 推出了 Site-to-Site VPN 与 AWS Secrets Manager 的原生集成功能。该方案将 PSK 的管理模式从传统的“分散明文存储”转变为“集中加密托管”。

*   **集中化安全存储：** 将 PSK 作为密文存储在 Secrets Manager 中，并通过 AWS KMS 进行静态加密。VPN 连接配置中不再保存密钥明文，而是引用密钥的 ARN（Amazon Resource Name）。
*   **精细化权限控制：** 利用 AWS IAM 策略，可以精确控制哪些用户或服务（Role）有权创建、读取或修改 PSK。例如，只允许网络管理员团队访问相关密钥。
*   **全面的审计追踪：** 所有对 Secrets Manager 中密钥的访问请求（如 `GetSecretValue`）都会被 AWS CloudTrail 记录下来，提供了完整、不可篡改的审计日志。
*   **简化的密钥管理：** 虽然完全自动化的轮换仍需自定义实现，但 Secrets Manager 提供了轮换框架，大大简化了与本地设备同步更新密钥的流程。

---

### 3. 核心技术原理

#### 3.1 IPsec VPN 工作原理与 PSK 的作用

要理解此方案的价值，首先需要了解 IPsec VPN 的基本工作原理。

**IPsec (Internet Protocol Security)** 是一个协议套件，用于保护 IP 网络通信，它主要通过两种协议实现安全功能：
1.  **封装安全载荷 (ESP):** 提供数据的机密性（加密）、完整性校验和来源认证。
2.  **认证头 (AH):** 提供数据的完整性校验和来源认证，但不提供加密。

AWS Site-to-Site VPN 主要使用 ESP 来确保数据在传输过程中的私密性和完整性。整个连接的建立过程由 **IKE (Internet Key Exchange)** 协议负责，分为两个主要阶段：

*   **IKE 阶段一 (Phase 1):**
    *   **目标：** 建立一个安全的管理通道，称为“IKE SA"(Security Association)。此通道用于保护后续阶段二的协商过程。
    *   **认证方式：** 在此阶段，通信双方（即 AWS VPN 网关和用户的本地客户网关）必须相互验证身份。**预共享密钥 (PSK) 正是在这里发挥作用**。双方各自使用本地配置的 PSK 进行计算，并交换计算结果。只有当双方的 PSK 完全一致时，身份验证才能通过。这确保了用户正在与一个可信的对端通信，而不是恶意冒充者。

*   **IKE 阶段二 (Phase 2):**
    *   **目标：** 在阶段一建立的安全通道内，协商用于实际加密和解密用户数据（例如，从用户本地网络发往 VPC 的流量）的 IPsec SA。
    *   这个阶段会确定具体的加密算法（如 AES-256）、哈希算法（如 SHA-256）以及用于加密数据的密钥。这些密钥会定期刷新，以提高安全性。

**总结来说，PSK 是 IPsec VPN 连接的“门锁钥匙”。** 它不直接加密用户数据，而是作为初始身份验证的凭证，用于打开一个安全的“会议室”（IKE Phase 1），在这个会议室里，双方才能安全地协商真正用于保护数据的密钥（IKE Phase 2）。因此，PSK 的保密性至关重要。

#### 3.2 Secrets Manager 集成原理

该集成方案的核心在于将 PSK 的生命周期管理与 VPN 连接的配置解耦。

1.  **创建阶段：**
    *   当用户通过 AWS 控制台、CLI 或 API 创建一个新的 Site-to-Site VPN 连接时，可以选择将 PSK 的存储方式指定为 `SecretsManager`。
    *   此时，Site-to-Site VPN 服务会调用 Secrets Manager 的 API，在后台自动创建一个新的 Secret。
    *   PSK 的值（无论是 AWS 自动生成还是用户手动提供）被安全地存入这个 Secret 中。
    *   最终，VPN 连接的配置信息里只保存指向该 Secret 的 ARN，而不包含任何密钥明文。

2.  **连接建立阶段：**
    *   当 VPN 服务需要建立或重新协商 IPsec 隧道时，它会利用其服务角色（Service-Linked Role）的授权，向 Secrets Manager 发起 `GetSecretValue` API 调用。
    *   Secrets Manager 验证权限后，将解密的 PSK 值安全地返回给 VPN 服务在内存中使用，用于完成 IKE 阶段一的身份验证。

3.  **迁移阶段：**
    *   对于已存在的、使用标准方式存储 PSK 的 VPN 连接，可以执行“修改隧道选项”操作。
    *   选择将 PSK 存储迁移到 Secrets Manager 后，AWS 会读取现有的 PSK，在 Secrets Manager 中创建一个新的 Secret 来存储它，然后用 Secret ARN 更新 VPN 隧道的配置。
    *   此过程会触发隧道重新协商，导致短暂的网络中断（通常在1-2分钟内）。

---

### 4. 详细配置步骤

以下将分别介绍如何通过 AWS 管理控制台和 AWS CLI 来配置和迁移 PSK 存储。

#### 4.1 通过 AWS 管理控制台操作

**A. 创建新的 VPN 连接并使用 Secrets Manager**

1.  在创建 Site-to-Site VPN 连接的工作流中，当配置隧道选项时，用户会看到 **预共享密钥存储 (Pre-shared key storage)** 的选项。
2.  选择 **Secrets Manager** 存储（新增强选项），而不是“标准存储 (Standard storage)”（原始方法）。

    [![Figure 1: PSKs storage options in the Console](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-1.png)](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-1.png)
    *图 1: 控制台中预共享密钥的存储选项*

3.  创建连接后，在 VPN 连接的详细信息页面，用户可以看到一个名为 **Secret management ARN** 的字段，其中包含了指向 Secrets Manager 中密钥的 ARN。

    [![Figure 2: Secret Management ARN in VPN Connection details page](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-2.png)](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-2.png)
    *图 2: VPN 连接详情页中的 Secret Management ARN*

4.  要查看 PSK 的值，请导航到 **Secrets Manager** 服务控制台，找到对应的 Secret，然后选择 **“检索密钥值 (Retrieve Secret Value)”**。

    [![Figure 3: PSK secret value](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-3.png)](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-3.png)
    *图 3: 在 Secrets Manager 中检索 PSK 的值*

**B. 迁移现有的 VPN 连接**

1.  对于使用标准存储（明文 PSK）的现有连接，其 **Secret management ARN** 字段为空。

    [![Figure 5. Empty Secret management ARN in VPN Connection Details page](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-5-1.png)](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-5-1.png)
    *图 5: 标准存储模式下，Secret management ARN 字段为空*

2.  在 VPN 连接列表中选中要迁移的连接，选择 **“操作 (Actions)”** -> **“修改 VPN 隧道选项 (Modify VPN Tunnel Options)”**。

    [![Figure 6. Modify VPN tunnel option under action menu for the selected VPN connection](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-6.png)](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-6.png)
    *图 6: 修改 VPN 隧道选项*

3.  选择要修改的隧道，然后在 **预共享密钥存储 (Pre-shared key storage)** 处，将选项从“标准存储”更改为 **“Secrets Manager”**，然后保存更改。

    [![Figure 8. Options for selecting pre-shared key storage in site-to-site VPN console](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-8.png)](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-8.png)
    *图 8: 将预共享密钥存储选项更改为 Secrets Manager*

4.  保存后，AWS 会自动将现有 PSK 迁移到新创建的 Secret 中，并在 VPN 详情页显示其 ARN。

    [![Figure 9. The ARN of the Secrets Manager for the Tunnel Migrated to Secrets Manager Storage](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-9.png)](https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/08/11/Figure-9.png)
    *图 9: 迁移后生成的 Secret ARN*

5.  当一个隧道迁移成功并恢复 UP 状态后，对另一个隧道重复此过程。

#### 4.2 通过 AWS CLI 操作

**A. 创建新的 VPN 连接**

使用 `create-vpn-connection` 命令，并添加 `--pre-shared-key-storage SecretsManager` 参数。

```bash
# 示例：创建一个连接到 VGW 的 VPN 连接，并将 PSK 存储在 Secrets Manager 中
aws ec2 create-vpn-connection \
    --customer-gateway-id cgw-06fa98e11e66c3646 \
    --type ipsec.1 \
    --vpn-gateway-id vgw-0ae7deff670a92445 \
    --pre-shared-key-storage SecretsManager
```

**B. 迁移现有 VPN 隧道的存储**

使用 `modify-vpn-tunnel-options` 命令来切换 PSK 的存储类型。

```bash
# 示例：将指定隧道的 PSK 存储迁移到 Secrets Manager
aws ec2 modify-vpn-tunnel-options \
    --vpn-connection-id vpn-01d7549f4acca1f70 \
    --vpn-tunnel-outside-ip-address 54.71.32.128 \
    --pre-shared-key-storage SecretsManager \
    --tunnel-options "{}"
```

**C. 验证配置**

使用 `describe-vpn-connections` 命令来查看 VPN 连接的详细信息。如果配置成功，输出的 JSON 中将包含 `PreSharedKeyArn` 字段。

```bash
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-01d7549f4acca1f70
```

输出结果将包含类似以下的字段：
```json
{
  "VpnConnections": [
    {
      "...": "...",
      "PreSharedKeyArn": "arn:aws:secretsmanager:us-west-2:123456789012:secret:s2svpn-preprod!vpn-0074cc1c3ce9a041e-1Skga6-pevysO",
      "VpnConnectionId": "vpn-0074cc1c3ce9a041e",
      "State": "pending",
      "...": "..."
    }
  ]
}
```

---

### 5. 生产落地考虑

在生产环境中部署此方案时，需综合考虑以下几点：

*   **IAM 权限策略（最小权限原则）：**
    *   **用户权限：** 确保只有授权的管理员 IAM 用户或角色才有 `ec2:CreateVpnConnection`, `ec2:ModifyVpnTunnelOptions` 以及访问对应 Secret 的 `secretsmanager:GetSecretValue`, `secretsmanager:DescribeSecret` 等权限。
    *   **服务角色：** 确认 `AWSServiceRoleForVPCTransitGateway` 或类似的服务相关角色拥有访问 Secrets Manager 的权限。通常这是自动配置的，但自定义权限边界时需特别注意。

*   **迁移计划与风险控制：**
    *   **选择维护窗口：** 由于迁移会造成隧道中断，务必在业务低峰期的维护窗口内进行。
    *   **逐一迁移隧道：** Site-to-Site VPN 连接通常包含两个隧道以实现高可用。应先迁移一个隧道，待其恢复稳定（状态为 UP）后，再迁移另一个。这样可以在迁移过程中保持网络连接不中断。
    *   **充分测试：** 在迁移前，确保用户有能力访问本地网络设备（Customer Gateway）的配置，以便在出现问题时进行排错。

*   **监控与告警：**
    *   **CloudTrail 监控：** 在 CloudTrail 中创建跟踪，并配置 CloudWatch Alarms，以监控对 VPN 密钥的 `GetSecretValue` 和 `PutSecretValue` 等敏感 API 调用。对非预期来源的调用进行告警。
    *   **VPN 隧道状态监控：** 配置 CloudWatch Alarms 来监控 VPN 隧道的健康状态（`TunnelState` 指标），在隧道断开时及时获得通知。

*   **密钥轮换策略：**
    *   Secrets Manager 的自动轮换功能需要一个 Lambda 函数来执行。对于 VPN PSK，此 Lambda 函数的逻辑会更复杂，因为它不仅要更新 Secret 中的值，还必须：
        1.  调用 `ec2:ModifyVpnTunnelOptions` API 更新云端的隧道 PSK。
        2.  **（关键步骤）** 通过自动化脚本（如 Ansible）或手动方式，同步更新本地网络设备上的 PSK。
    *   由于步骤2的复杂性，多数企业选择“计划性手动轮换”而非全自动轮换。即利用 Secrets Manager 的框架，在维护窗口期，由管理员触发轮换流程。

*   **成本考量：**
    *   通过 VPN 服务集成创建的 Secret **本身不收取存储费用**。
    *   API 调用费用遵循 Secrets Manager 的标准定价。正常的 VPN 服务使用（隧道建立/重协商）所产生的 API 调用通常成本极低。只有在频繁手动读取或触发轮换时，才需要考虑 API 调用成本。

---

### 6. 专业问答 (Q&A)

**Q1: 为什么不应该将 PSK 硬编码或存储在 Git 等代码仓库中？**
**A:** 将 PSK 这类敏感凭证硬编码是严重的安全隐患。代码仓库的访问权限通常较广，一旦仓库被攻破或凭证被无意中提交到公共仓库，将直接暴露用户的网络边界，带来巨大安全风险。此外，这也违反了几乎所有主流的安全与合规标准。

**Q2: 将现有 VPN 连接的 PSK 迁移到 Secrets Manager 会导致业务中断吗？**
**A:** 是的，会。迁移操作会更新隧道配置，这会强制 IPsec 隧道重新进行协商，从而导致该隧道短暂中断。为避免业务影响，强烈建议在维护窗口期操作，并利用双隧道的冗余性，逐一迁移，确保总有一条隧道是可用的。

**Q3: 使用 Secrets Manager 管理 PSK 后，如何实现密钥的自动轮换？**
**A:** 这需要一个自定义的解决方案。用户需要创建一个 Lambda 轮换函数，并将其与 Secret关联。该函数的核心逻辑应包括：1) 生成一个新的高强度随机 PSK；2) 调用 Secrets Manager API 更新 Secret 中的密钥版本；3) 调用 EC2 API (`ModifyVpnTunnelOptions`) 将新密钥应用到 VPN 隧道；4) **（最复杂的部分）** 触发一个流程（如通过 SSM Run Command, Ansible Tower API, 或人工工单）来更新用户本地网络设备上的 PSK。由于需要与本地设备交互，全自动化方案需要精心的设计和测试。

**Q4: 除了安全优势，使用 Secrets Manager 统一管理 PSK 还有其他好处吗？**
**A:** 有的。主要好处在于 **标准化** 和 **合规性**。它将所有网络凭证的管理方式统一起来，降低了运维复杂性。对于需要满足如 SOC 2, PCI-DSS, HIPAA 等合规框架的企业，能够提供清晰的密钥访问控制证据和审计日志，是证明其安全实践符合标准的关键一环。

---

### 7. 参考文档

*   [**AWS Site-to-Site VPN Documentation**](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html)
*   [**Securing AWS Site-to-Site VPN pre-shared keys using AWS Secrets Manager**](https://aws.amazon.com/blogs/networking-and-content-delivery/securing-aws-site-to-site-vpn-pre-shared-keys-using-aws-secrets-manager/)
*   [**AWS Secrets Manager Developer Guide**](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
*   [**modify-vpn-tunnel-options (AWS CLI Command Reference**)](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/modify-vpn-tunnel-options.html)
*   [**IAM policies for AWS Secrets Manager**](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access-control.html)
