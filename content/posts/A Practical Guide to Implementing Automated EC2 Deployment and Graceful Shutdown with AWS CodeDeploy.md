---
author: mingyu110
date: 2025-10-12
description: 本实践指南深入探讨了如何利用AWS CodeDeploy实现Amazon EC2实例的自动化部署流程，并确保在部署过程中的服务优雅停机，从而最大限度地减少服务中断时间，提升应用发布的安全性和可靠性。
keywords:
  - AWS
  - CodeDeploy
  - EC2
  - 自动化部署
  - 蓝绿部署
  - 优雅停机
tags:
  - AWS
  - CodeDeploy
  - EC2
  - DevOps
---
# 使用 AWS CodeDeploy 实现 EC2 自动化部署与优雅停机实践指南

## 1. 应用场景与技术挑战

### 1.1. 应用场景
在现代云应用中，快速、可靠地迭代和部署软件至关重要。对于部署在Amazon EC2实例上的应用程序（如Web服务、API后端、微服务等），自动化部署是实现持续集成/持续部署（CI/CD）流程的核心环节。主要应用场景包括：
- **频繁的功能更新**：敏捷开发团队需要每天甚至每小时多次部署新功能或修复。
- **环境一致性**：确保开发、测试、生产等多个环境中的应用程序版本和配置保持一致。
- **弹性伸缩下的版本管理**：在Auto Scaling Group（ASG）中，新启动的实例必须自动部署最新且正确的应用程序版本。
- **高可用要求**：部署过程中需要最小化甚至零停机时间，以保证业务连续性。

### 1.2. 技术挑战
传统的EC2手动部署方式（如SSH登录、手动拉取代码、重启服务）面临着严峻的挑战：
- **人为错误风险高**：手动操作流程繁琐，容易因疏忽导致部署失败、配置错误或环境污染。
- **部署耗时且效率低下**：当服务实例数量增加时，手动部署变得极其耗时，无法满足快速迭代的需求。
- **停机时间长**：简单的“停机-更新-启动”模式会导致服务中断，影响用户体验。
- **回滚困难**：一旦新版本出现问题，手动回滚过程复杂、压力大，且难以保证快速恢复。
- **实例突然终止导致的问题**：在使用ASG进行缩容时，EC2实例会被突然终止。这可能导致正在处理的用户请求中断、内存中的数据丢失或后台任务未完成，从而造成数据不一致或糟糕的用户体验。

## 2. 解决方案：AWS CodeDeploy 核心能力

AWS CodeDeploy是一项完全托管的部署服务，可自动将应用程序部署到Amazon EC2实例、本地实例、AWS Lambda或Amazon ECS服务。它使发布新功能变得更加容易，有助于避免部署期间的停机时间，并简化了应用程序更新的复杂性。

### 核心组件
- **应用程序 (Application)**：CodeDeploy中的一个逻辑实体，作为部署相关组件（如部署组、修订版本）的容器。
- **部署组 (Deployment Group)**：一组用于接收部署的EC2实例集合。可以通过实例标签、ASG名称或两者结合来定义。
- **部署配置 (Deployment Configuration)**：定义部署期间的执行策略，如一次部署一个实例（OneAtATime）、一次部署一半实例（HalfAtATime）或蓝/绿部署（Blue/Green）。
- **修订版本 (Revision)**：要部署的应用程序的特定版本，通常包含源代码、配置文件、Web文件以及一个`appspec.yml`文件。修订版本通常打包成zip或tar.gz格式，并存储在Amazon S3或GitHub中。
- **AppSpec 文件 (`appspec.yml`)**：位于修订版本根目录的YAML格式文件，是CodeDeploy的“指挥中心”。它定义了部署的每个阶段（生命周期钩子）需要执行的脚本和操作。
- **CodeDeploy Agent**：一个必须安装并运行在每个目标EC2实例上的软件包。它负责轮询CodeDeploy服务以获取部署指令，并根据`appspec.yml`文件执行部署任务。

## 3. 生产落地实践考虑

在生产环境中使用CodeDeploy时，需要考虑以下几点以确保部署的稳定、安全和高效：
- **高可用部署策略**：优先选择滚动更新（如`CodeDeployDefault.OneAtATime`）或蓝/绿部署策略，以避免服务中断。蓝/绿部署通过将流量从旧环境切换到新环境来实现零停机，是生产环境的最佳实践。
- **健康检查与自动回滚**：在部署组中配置健康检查。CodeDeploy可以在部署期间监控实例状态，一旦检测到新版本导致实例不健康，它能自动停止部署并回滚到上一个稳定版本。
- **IAM最小权限原则**：为CodeDeploy服务本身创建一个服务角色（Service Role），并为EC2实例创建一个实例配置文件（Instance Profile）。严格遵循最小权限原则，仅授予它们执行任务所必需的权限。
- **日志、监控与告警**：
    - **部署日志**：CodeDeploy Agent的日志（位于Linux上的`/var/log/aws/codedeploy-agent/codedeploy-agent.log`）是排查部署问题的首要入口。
    - **事件监控**：在CodeDeploy控制台可以查看详细的部署事件和生命周期钩子的执行状态。
    - **CloudWatch告警**：可以为部署失败、成功或停止等事件创建Amazon CloudWatch Events规则，并设置SNS通知或触发Lambda函数，实现自动化告警。
- **优雅停机（Graceful Termination）**：这是本文档的重点实践之一。通过配置ASG生命周期钩子与CodeDeploy的终止钩子，确保在实例缩容时，应用程序能够先完成当前任务、保存状态，然后再被终止。

## 4. 实践步骤一：实现EC2自动化部署（基础篇）

本节将指导您完成一个基础的“Hello World”自动化部署。

1.  **创建IAM角色**：需要两个角色。
    *   **CodeDeploy服务角色**：授予CodeDeploy服务调用AWS API（如查找EC2实例）的权限。
    *   **EC2实例配置文件**：授予EC2实例从S3下载修订版本和与CodeDeploy通信的权限。
2.  **启动并配置EC2实例**：
    *   启动一个Amazon Linux 2实例。
    *   为其附加第1步创建的EC2实例配置文件。
    *   给实例打上一个明确的标签，如`App:MyWebApp`，以便部署组识别。
    *   在“用户数据”中或通过SSH登录后，安装CodeDeploy Agent。
3.  **准备应用修订版本**：
    *   创建一个`index.html`文件，内容为`<h1>Hello, CodeDeploy!</h1>`。
    *   创建一个`appspec.yml`文件，内容如下：
        ```yaml
        version: 0.0
        os: linux
        files:
          - source: /index.html
            destination: /var/www/html/
        hooks:
          BeforeInstall:
            - location: scripts/install_dependencies.sh
              timeout: 300
              runas: root
        ```
    *   创建一个`scripts/install_dependencies.sh`脚本，用于安装Web服务器：
        ```bash
        #!/bin/bash
        yum update -y
        yum install -y httpd
        systemctl start httpd
        systemctl enable httpd
        ```
    *   将这些文件和目录打包成一个zip文件，如`MyWebApp_v1.zip`。
4.  **上传修订版本到S3**：创建一个S3存储桶，并将`MyWebApp_v1.zip`上传上去。
5.  **创建CodeDeploy应用和部署组**：
    *   在CodeDeploy控制台创建一个名为`MyWebApp`的应用程序（计算平台选择EC2/本地）。
    *   在该应用下创建一个部署组，配置如下：
        *   **服务角色**：选择第1步创建的CodeDeploy服务角色。
        *   **部署类型**：选择“就地部署”（In-place）。
        *   **环境配置**：选择“Amazon EC2实例”，并使用第2步设置的标签（`App:MyWebApp`）来选择实例。
        *   **部署配置**：选择`CodeDeployDefault.OneAtATime`。
6.  **执行部署**：
    *   在刚创建的部署组中，点击“创建部署”。
    *   指定修订版本的位置（S3桶和`MyWebApp_v1.zip`的路径）。
    *   启动部署并观察部署流程。成功后，访问EC2的公有IP，您应该能看到“Hello, CodeDeploy!”页面。

## 5. 实践步骤二：配置ASG优雅停机（高级实践）

本节深入探讨原始文档的核心内容：如何利用CodeDeploy实现ASG中实例的优雅停机。

### 5.1. 为什么需要优雅停机？
想象一个场景：您的电商网站正在处理一个用户的订单提交，就在事务处理的关键时刻，处理该请求的EC2实例因为ASG缩容事件而被突然终止。这会导致订单失败和极差的用户体验。优雅停机机制允许您的应用程序在实例终止前：
- **完成进行中的任务**：如完成请求处理、刷新缓存、关闭数据库连接。
- **排空连接**：将实例从负载均衡器目标组中移除，让现有连接正常结束，同时不再接收新连接。
- **执行清理工作**：如保存未持久化的数据、上传日志、从外部服务注销。
- **防止数据损坏**：确保不会因突然关机而丢失或损坏数据。

### 5.2. 工作流程解析
下图展示了优雅停机的工作流程：

![优雅停机流程图](/images/优雅停机.png)
*图解：此图展示了从ASG发起缩容事件到CodeDeploy介入执行优雅停机脚本，再到实例最终被终止的完整流程。*

1.  **缩容事件触发**：ASG因负载降低或健康检查失败，决定需要终止一个实例。
2.  **生命周期钩子介入**：ASG不会立即终止实例，而是触发`EC2_INSTANCE_TERMINATING`生命周期钩子，并将实例置于`Terminating:Wait`状态。
3.  **CodeDeploy响应**：与该ASG关联的CodeDeploy部署组会收到此事件的通知，并针对该实例启动一个特殊的“终止部署”。
4.  **`ApplicationStop`钩子执行**：作为终止部署的一部分，CodeDeploy会执行您在`appspec.yml`中定义的`ApplicationStop`生命周期钩子。您的自定义优雅停机脚本将在此阶段运行。
5.  **心跳与完成信号**：在`ApplicationStop`脚本运行时，CodeDeploy会向ASG发送心跳以延长等待时间。脚本成功执行后，CodeDeploy会通知ASG可以“继续”终止流程。
6.  **实例终止**：ASG收到“继续”信号后，才会真正终止该EC2实例。

### 5.3. 配置步骤

#### 步骤 1: 确保 `appspec.yml` 包含 `ApplicationStop` 钩子
`appspec.yml`文件是关键。`ApplicationStop`钩子是您实现自定义停机逻辑的地方。

![ApplicationStop工作流](https://miro.medium.com/v2/resize:fit:864/1*qMmIYQ_Ttsf9yj1IMUvXiQ.png)
*图解：`ApplicationStop`是部署生命周期中的最后一个钩子，在实例从负载均衡器取消注册后执行，是执行清理工作的理想时机。*

一个示例`appspec.yml`可能如下所示：
```yaml
version: 0.0
os: linux

file_exists_behavior: OVERWRITE

hooks:
  AfterInstall:
    - location: startUp.sh
      timeout: 300
      runas: root
  ApplicationStop:
    - location: stop_application.sh
      timeout: 300 # 确保超时时间足够长，以完成所有清理工作
      runas: root
```
**关键点**：`ApplicationStop`的`timeout`至关重要。它定义了CodeDeploy等待您的脚本完成的最长时间。

#### 步骤 2: 编写优雅停机脚本 `stop_application.sh`
这个脚本包含了实际的停机逻辑。以下是一个健壮的模板，它能判断当前部署是否由ASG终止事件触发：

```bash
#!/bin/bash

# 从CodeDeploy环境变量中获取部署ID
# 参考: https://aws.amazon.com/blogs/devops/using-codedeploy-environment-variables/
DEPLOYMENT_ID=$DEPLOYMENT_ID
# 从CodeDeploy环境变量中获取部署组ID
DEPLOYMENT_GROUP_ID=$DEPLOYMENT_GROUP_ID
# 从EC2元数据中获取区域信息
AWS_REGION=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)


# 获取部署的触发者（creator）
DeploymentType=$(aws deploy get-deployment --deployment-id "${DEPLOYMENT_ID}" --query 'deploymentInfo.creator' --output text --region ${AWS_REGION})

echo "CodeDeploy Deployment Type: ${DeploymentType}"

# 仅当部署由ASG终止事件触发时，才执行停止服务的逻辑
if [ "${DeploymentType}" = "autoscalingTermination" ]; then
  echo "正在为ASG终止事件启动优雅停机..."
  
  # --- 在此处替换为您应用的实际停机命令 ---
  
  # 示例：停止一个以systemd方式运行的Java应用
  # sudo systemctl stop my-java-app
  
  # 示例：停止一个使用PM2管理的Node.js应用
  # pm2 stop my-node-app
  # pm2 save
  
  # 在此处添加其他清理或数据刷盘命令
  echo "应用优雅停机完成。"
else
  echo "此部署非ASG终止事件触发，跳过优雅停机逻辑。"
fi

exit 0 # 务必以状态码0退出，表示成功
```
**脚本要点**:
- **`DeploymentType`判断**：通过`aws deploy get-deployment`命令获取部署的`creator`。当部署由ASG终止钩子触发时，其值会是`autoscalingTermination`。这是区分正常部署和终止部署的关键。
- **替换占位符**：您**必须**将脚本中的示例命令替换为能真正优雅关闭您应用程序的命令。
- **IAM权限**：确保EC2实例的IAM角色有权限执行`deploy:GetDeployment`操作。

#### 步骤 3: 在部署组中启用终止钩子
这是将ASG终止事件与CodeDeploy连接起来的关键一步。

**使用AWS管理控制台:**
1.  导航至 **AWS CodeDeploy** 控制台。
2.  选择您的应用程序，然后选择您的**部署组**。
3.  点击**编辑**。
4.  向下滚动到**高级**部分。
5.  找到并勾选**“启用终止钩子”**复选框。

![启用终止钩子](https://miro.medium.com/v2/resize:fit:1120/1*JzIvGtcCAPbVvq7n8eMqTg.png)
*图解：在部署组的高级设置中，只需勾选此选框，CodeDeploy就会自动处理与关联ASG的终止生命周期钩子。*

6.  保存对部署组的更改。

### 5.4. 测试优雅停机
1.  **手动缩容**：在ASG控制台，将“所需容量”（Desired capacity）减少1。
2.  **监控部署**：在CodeDeploy控制台，您应该会看到一个由`autoscalingTermination`创建的新部署任务。
3.  **检查实例日志**：在ASG的“实例管理”选项卡中，您会看到一个实例处于`Terminating:Wait`状态。此时SSH登录到该实例，检查CodeDeploy Agent日志（`/var/log/aws/codedeploy-agent/codedeploy-agent.log`）和您的应用日志，确认`stop_application.sh`脚本是否按预期执行。

## 6. 参考文档
为了更深入地理解和实践，以下是官方的核心参考文档：
- [AppSpec 'hooks' section](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html#appspec-hooks-server)
- [Using CodeDeploy environment variables](https://aws.amazon.com/blogs/devops/using-codedeploy-environment-variables/)

- [AWS CodeDeploy User Guide](https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html): CodeDeploy最全面的官方指南。
- [AppSpec File reference](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file.html): `appspec.yml`所有部分的详细参考。
- [Lifecycle event hooks](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html): 对所有生命周期钩子执行顺序和用途的权威解释。
- [Tutorial: Deploy an application to an Auto Scaling group](https://docs.aws.amazon.com/codedeploy/latest/userguide/tutorials-auto-scaling.html): 一个将CodeDeploy与ASG集成的官方教程。
- [CodeDeploy with blue/green deployments](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployments-blue-green.html): 关于如何实现零停机蓝/绿部署的详细指南。
- [Lifecycle Hooks for Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html): ASG生命周期钩子的官方文档。