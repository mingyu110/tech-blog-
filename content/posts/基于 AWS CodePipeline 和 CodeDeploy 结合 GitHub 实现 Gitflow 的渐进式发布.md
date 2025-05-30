---
title: "基于 AWS CodePipeline 和 CodeDeploy 结合 GitHub 实现 Gitflow 的渐进式发布"
date: 2025-05-30
tags: ["AWS", "CodePipeline", "CodeDeploy", "GitHub", "Gitflow", "DevOps", "CI/CD"]
categories: ["AWS", "DevOps", "云架构"]
description: "详细介绍如何在AWS上使用CodePipeline和CodeDeploy，结合GitHub实现基于Gitflow分支策略的渐进式发布方案，适合小型开发团队的自动化CI/CD实践"
toc: true
draft: false
---

# 基于 AWS CodePipeline 和 CodeDeploy 结合 GitHub 实现 Gitflow 的渐进式发布

### 1. 引言

#### 1.1 目的

本方案为一个小型开发团队提供一个标准的技术方案，用于在 AWS 上使用 CodePipeline 和 CodeDeploy，结合 GitHub 实现基于 Gitflow 分支策略的渐进式发布（Canary Release），并确保发布版本遵循语义化版本管理（Semantic Versioning）。方案适合小型项目，强调自动化、协作效率和低成本。

#### 1.2 背景

渐进式发布通过逐步向部分用户推出新版本，降低上线风险，适合微服务架构或高可用性需求。Gitflow 分支策略为小型团队提供清晰的协作框架，语义化版本管理确保版本号（如 v1.2.3）清晰反映变更类型。AWS CodePipeline 和 CodeDeploy 提供自动化 CI/CD 流水线，结合 GitHub 实现高效代码托管和触发。

#### 1.3 范围

- 代码托管：GitHub，用于版本控制和协作。
- 分支策略：Gitflow，包含 `main`、`develop`、`feature/`*、`release/*` 和 `hotfix/*` 分支。
- CI/CD 工具：AWS CodePipeline（流水线自动化）、CodeDeploy（渐进式部署）。
- 部署目标：Amazon ECS Fargate，运行简单的 Node.js 微服务。
- 版本管理：语义化版本（SemVer），通过 Git 标签触发发布。
- 团队规模：小型团队，需简单、可扩展的流程。

### 2. 系统架构

#### 2.1 架构概述

- GitHub：托管代码，管理 Gitflow 分支和语义化版本标签。
- AWS CodePipeline：协调源代码拉取、构建和部署阶段，监控 GitHub 标签（如 V1.X.X）。
- AWS CodeBuild：编译代码，生成 Docker 镜像并推送至 Amazon ECR。
- AWS CodeDeploy：执行渐进式部署到 ECS Fargate，使用 Canary 配置（如 10% 流量 5 分钟后全切换）。
- Amazon ECS Fargate：运行 Node.js 微服务，无需管理服务器。
- Application Load Balancer (ALB)：分发流量，支持渐进式发布。
- Amazon ECR：存储 Docker 镜像，提供容器镜像仓库服务。
- Amazon S3：存储 CodePipeline 构建工件（如 appspec.yml、taskdef.json 等配置文件）。
- Amazon CloudWatch：监控部署健康状态。

#### 2.2 方案架构图

<img src="/Users/jinxunliu/my-microservice/enhanced_aws_architecture.png" alt="enhanced_aws_architecture" style="zoom:67%;" />

### 3.Gitflow 分支策略

采用简化的 Gitflow 分支策略，适合小型团队协作：

| 分支类型  | 描述                               | 用途                             | 生命周期   |
| --------- | ---------------------------------- | -------------------------------- | ---------- |
| main      | 生产就绪代码，仅包含已发布版本     | 打标签（如 v1.2.3），触发部署    | 长生命周期 |
| develop   | 集成新功能和修复，反映下一版本状态 | 特性分支合并目标                 | 长生命周期 |
| feature/* | 特性开发分支，如 feature/add-login | 新功能开发，合并回 develop       | 短生命周期 |
| release/* | 发布准备分支，如 release/1.2.3     | 测试后合并到 main，打标签        | 短生命周期 |
| hotfix/*  | 紧急生产修复分支，如 hotfix/1.2.4  | 快速修复，合并回 main 和 develop | 短生命周期 |

#### 3.1 分支工作流

- 日常开发：
  - 开发者从 develop 创建 feature/* 分支（如 feature/add-login）。
  - 完成后提交 PR（Pull Request），经过代码审查后合并回 develop，删除分支。

- 发布准备：
  - 从 develop 创建 release/1.2.3 分支，进行测试和微调。
  - **推荐方式（开启分支保护）**：创建 PR 将 release 分支合并到 main，经过审查和CI检查后合并。
  - **备选方式（无分支保护）**：直接合并到 main分支。
  - 合并完成后打标签（如 git tag v1.2.3），推送到 GitHub。
  - 将 main 的变更合并回 develop，保持分支同步。

- 紧急修复：
  - 从 main 创建 hotfix/1.2.4 分支，修复问题。
  - 通过 PR 或直接合并到 main，打新标签（如 v1.2.4），推送到 GitHub。
  - 将修复合并回 develop，同步修复。

- 分支保护策略：
  - **main分支**：强制要求 PR 审查、CI 检查通过、分支最新
  - **develop分支**：要求 PR 审查，确保代码质量
  - **feature/release/hotfix分支**：无特殊限制，便于开发和测试

- 语义化版本：

  - 版本号遵循 MAJOR.MINOR.PATCH（如 1.2.3）：
    - MAJOR：不兼容更改。
    - MINOR：向后兼容的新功能。
    - PATCH：向后兼容的错误修复。

  - 标签（如 v1.2.3）在 main 上创建，触发 CodePipeline。

#### 3.2GitFlow 流程图

下图展示了完整的 GitFlow 分支策略与 AWS CI/CD 流水线的集成流程：

![GitFlow 与 AWS CI/CD 流程图](gitflow_cd_pipeline.png)该流程图清晰展示了：

- **分支管理**：main、develop、feature、release、hotfix 分支之间的关系和合并流程
- **CI/CD 触发**：main 和 develop 分支的合并如何触发 AWS CodePipeline
- **自动化流程**：从代码提交到容器化部署的完整自动化流水线
- **蓝绿部署**：CodeDeploy 实现的蓝绿部署策略，确保零停机时间

​       ![运行结果](/Users/jinxunliu/Desktop/运行结果.jpg)

### 4. 技术实现实践步骤

演示代码和策略等配置在GitHub： [演示项目地址](https://github.com/mingyu110/my-microservice)

#### 4.1 开发环境设置

##### 4.1.1 GitHub 仓库和 Node.js 微服务

- 创建 GitHub 仓库：

  - 在 GitHub 上创建新仓库（如 my-microservice），克隆到本地：

    ```bash
    git clone git@github.com:your-user-name/my-microservice.git
    cd my-microservice
    ```

- 初始化 Node.js 项目：

  - 初始化项目并安装 Express：

    ```bash
    npm init -y
    npm install express
    ```

  - 创建应用代码 (index.js)：

    ```javascript
    const express = require('express');
    const app = express();
    const port = process.env.PORT || 3000;
    const version = process.env.VERSION || '1.0.0';
    
    app.get('/', (req, res) => {
        res.send(`Version: ${version}`);
    });
    
    app.listen(port, () => {
        console.log(`Server running on port ${port}`);
    });
    ```

  - 创建 package.json：

    ```json
    {
        "name": "my-microservice",
        "version": "1.0.0",
        "description": "Simple microservice for CI/CD demo",
        "main": "index.js",
        "scripts": {
            "start": "node index.js"
        },
        "dependencies": {
            "express": "^4.17.1"
        }
    }
    ```

  - 创建 Dockerfile：

    ```dockerfile
    FROM alibaba-cloud-linux-3-registry.cn-hangzhou.cr.aliyuncs.com/alinux3/node:16.17.1-nslt

    USER root
    RUN mkdir -p /app && chown -R node:node /app

    WORKDIR /app
    COPY package*.json ./
    RUN chown -R node:node /app

    USER node
    RUN npm install --production

    USER root
    COPY . .
    RUN chown -R node:node /app

    USER node
    EXPOSE 3000
    CMD ["node", "index.js"]
    ```

  - 创建 .buildspec.yml（用于 CodeBuild）：

    ```yaml
    version: 0.2
    phases:
      pre_build:
        commands:
          - echo Logging in to Amazon ECR...
          - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
          - REPOSITORY_URI=$ECR_REGISTRY/my-microservice
          - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
          - IMAGE_TAG=${COMMIT_HASH:=latest}
          - echo "Repository URI: $REPOSITORY_URI"
          - echo "Image Tag: $IMAGE_TAG"
      build:
        commands:
          - echo Build started on `date`
          - echo Building the Docker image with Alibaba Cloud Linux base...
          - docker build -t $REPOSITORY_URI:latest .
          - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
      post_build:
        commands:
          - echo Build completed on `date`
          - echo Pushing the Docker images...
          - docker push $REPOSITORY_URI:latest
          - docker push $REPOSITORY_URI:$IMAGE_TAG
          - echo Writing image definitions file...
          - printf '[{"name":"my-container","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
          - echo "Contents of imagedefinitions.json:"
          - cat imagedefinitions.json
    artifacts:
      files: 
        - imagedefinitions.json
        - appspec.yaml
    ```

  - 初始化 Git 仓库：

    - 创建 main 和 develop 分支：

      ```bash
      git checkout -b main
      git push origin main
      git checkout -b develop
      git push origin develop
      ```

    - 设置分支保护规则（GitHub Settings > Branches），要求 PR 合并到 main 和 develop。

**重要：配置分支保护规则**

为了确保代码质量和发布安全，强烈建议在 GitHub 仓库中配置分支保护规则：

1. **进入仓库设置**：
   - 在 GitHub 仓库页面，点击 Settings > Branches

2. **配置 main 分支保护**：
   ```
   Branch name pattern: main
   
   ✅ Restrict pushes that create files larger than 100 MB
   ✅ Require a pull request before merging
     ✅ Require approvals (1)
     ✅ Dismiss stale PR approvals when new commits are pushed
     ✅ Require review from code owners
   ✅ Require status checks to pass before merging
     ✅ Require branches to be up to date before merging
   ✅ Require conversation resolution before merging
   ✅ Require signed commits (可选，提高安全性)
   ✅ Require linear history (可选，保持干净的提交历史)
   ✅ Include administrators (建议开启，确保一致性)
   ```

3. **配置 develop 分支保护**：
   ```
   Branch name pattern: develop
   
   ✅ Require a pull request before merging
     ✅ Require approvals (1)
   ✅ Require status checks to pass before merging
   ✅ Require conversation resolution before merging
   ```

4. **添加状态检查**（如果配置了CI）：
   - 在 "Require status checks to pass before merging" 中添加：
   - `continuous-integration/...` (你的CI系统检查)
   - `security/...` (安全扫描检查)

##### 4.1.2 本地测试

- 构建并运行 Docker 容器：

  ```bash
  docker build -t my-microservice .
  docker run -p 3000:3000 -e VERSION=1.0.0 my-microservice
  ```

#### 4.2 AWS 基础设施设置

##### 4.2.1 Amazon ECR

- 创建 ECR 仓库：

  ```bash
  aws ecr create-repository --repository-name my-microservice --region us-west-2
  ```

- 记录 ECR 仓库 URI（如 your-account-id.dkr.ecr.us-west-2.amazonaws.com/my-microservice）。

- 推送本地 Docker 镜像到 ECR：

  1. 登录 ECR 仓库：

     ```bash
     aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-west-2.amazonaws.com
     ```

  2. 给本地镜像打上 ECR tag（以 v1.0.0 为例）：

     ```bash
     docker tag my-microservice:v1.0.0 <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/my-microservice:v1.0.0
     ```

  3. 推送镜像到 ECR：

     ```bash
     docker push <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/my-microservice:v1.0.0
     ```

  4. （可选）推送 latest tag：

     ```bash
     docker tag my-microservice:v1.0.0 <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/my-microservice:latest
     docker push <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/my-microservice:latest
     ```

##### 4.2.2 Amazon ECS Fargate

- 创建 ECS 集群：

  ```bash
  aws ecs create-cluster --cluster-name my-cluster --region us-west-2
  ```

- 创建任务定义 (task-definition.json)：

  ```json
  {
      "family": "my-task-definition",
      "networkMode": "awsvpc",
      "requiresCompatibilities": ["FARGATE"],
      "cpu": "256",
      "memory": "512",
      "executionRoleArn": "arn:aws:iam::your-account-id:role/ecsTaskExecutionRole",
      "taskRoleArn": "arn:aws:iam::your-account-id:role/ecsTaskRole",
      "containerDefinitions": [
          {
              "name": "my-container",
              "image": "your-account-id.dkr.ecr.us-west-2.amazonaws.com/my-microservice:latest",
              "portMappings": [
                  {
                      "containerPort": 3000,
                      "hostPort": 3000,
                      "protocol": "tcp"
                  }
              ],
              "environment": [
                  {
                      "name": "VERSION",
                      "value": "1.0.0"
                  }
              ],
              "essential": true
          }
      ]
  }
  ```

- 注册任务定义：

  ```bash
  aws ecs register-task-definition --cli-input-json file://task-definition.json --region us-west-2
  ```

- 创建 ALB 和目标组：

  - 使用 AWS 控制台或 CLI 创建 ALB，配置监听器（HTTP 80）转发到目标组
  - 目标组绑定端口 3000，健康检查路径为 /

- 创建 ECS 服务：

  ```json
  {
      "cluster": "my-cluster",
      "serviceName": "my-service",
      "taskDefinition": "my-task-definition",
      "desiredCount": 2,
      "launchType": "FARGATE",
      "networkConfiguration": {
          "awsvpcConfiguration": {
              "subnets": ["subnet-12345678", "subnet-87654321"],
              "securityGroups": ["sg-12345678"],
              "assignPublicIp": "ENABLED"
          }
      },
      "loadBalancers": [
          {
              "targetGroupArn": "arn:aws:elasticloadbalancing:us-west-2:your-account-id:targetgroup/my-target-group/1234567890",
              "containerName": "my-container",
              "containerPort": 3000
          }
      ],
      "deploymentController": {
          "type": "CODE_DEPLOY"
      }
  }
  ```

##### 4.2.3 AWS CodeDeploy

- 创建 CodeDeploy 应用：

  ```bash
  aws deploy create-application --application-name my-ecs-app --compute-platform ECS --region us-west-2
  ```

- 创建部署组 (`deployment-group.json`)：

  ```json
  {
    "applicationName": "my-ecs-app",
    "deploymentGroupName": "my-deployment-group",
    "serviceRoleArn": "arn:aws:iam::933505494323:role/CodeDeployServiceRole",
    "deploymentConfigName": "CodeDeployDefault.ECSCanary10Percent5Minutes",
    "ecsServices": [
      {
        "serviceName": "my-service",
        "clusterName": "my-cluster"
      }
    ],
    "loadBalancerInfo": {
      "targetGroupPairInfoList": [
        {
          "targetGroups": [
            { "name": "my-target-group" },
            { "name": "my-target-group-green" }
          ],
          "prodTrafficRoute": {
            "listenerArns": [
              "arn:aws:elasticloadbalancing:us-west-2:933505494323:listener/app/my-alb/9e1a63adcbcca2c1/bfe9ec199a481387"
            ]
          }
        }
      ]
    },
    "blueGreenDeploymentConfiguration": {
      "terminateBlueInstancesOnDeploymentSuccess": {
        "action": "TERMINATE",
        "terminationWaitTimeInMinutes": 5
      },
      "deploymentReadyOption": {
        "actionOnTimeout": "CONTINUE_DEPLOYMENT",
        "waitTimeInMinutes": 0
      }
    },
    "deploymentStyle": {
      "deploymentType": "BLUE_GREEN",
      "deploymentOption": "WITH_TRAFFIC_CONTROL"
    }
  }
  ```

  **核心注意点：**
  - targetGroupPairInfoList 必须包含两个目标组（蓝绿部署），如 my-target-group 和 my-target-group-green。
  - prodTrafficRoute.listenerArns 必须为 ALB 监听器的 ARN。
  - blueGreenDeploymentConfiguration 不能包含 greenFleetProvisioningOption 字段。
  - deploymentStyle 必须为 BLUE_GREEN 和 WITH_TRAFFIC_CONTROL。
  - serviceRoleArn 必须为具备 CodeDeploy 权限的 IAM 角色。
  - 所有名称、ARN、ID 必须与实际 AWS 资源一致。

- 创建部署组：

  ```bash
  aws deploy create-deployment-group --cli-input-json file://deployment-group.json --region us-west-2
  ```

##### 4.2.4 AWS CodePipeline

使用 AWS CLI 创建完整的 CodePipeline 管道：

**1. 创建 CodePipeline 服务角色**

```bash
# 创建信任策略文件
cat > codepipeline-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codepipeline.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# 创建角色
aws iam create-role \
  --role-name CodePipelineServiceRole \
  --assume-role-policy-document file://codepipeline-trust-policy.json

# 创建权限策略（包含S3工件存储和ECR镜像访问权限）
cat > codepipeline-service-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-microservice-pipeline-artifacts-*",
        "arn:aws:s3:::my-microservice-pipeline-artifacts-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:DescribeRepositories",
        "ecr:DescribeImages"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:BatchGetBuilds",
        "codebuild:StartBuild"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateDeployment",
        "codedeploy:GetApplication",
        "codedeploy:GetApplicationRevision",
        "codedeploy:GetDeployment",
        "codedeploy:GetDeploymentConfig",
        "codedeploy:RegisterApplicationRevision"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codestar-connections:UseConnection"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# 绑定策略到角色
aws iam put-role-policy \
  --role-name CodePipelineServiceRole \
  --policy-name CodePipelineServicePolicy \
  --policy-document file://codepipeline-service-policy.json
```

**2. 创建 CodeBuild 服务角色**

```bash
# 创建 CodeBuild 信任策略
cat > codebuild-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codebuild.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# 创建角色
aws iam create-role \
  --role-name CodeBuildServiceRole \
  --assume-role-policy-document file://codebuild-trust-policy.json

# 绑定 AWS 托管策略
aws iam attach-role-policy \
  --role-name CodeBuildServiceRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

# 创建自定义策略用于 ECR 访问和S3工件访问
cat > codebuild-service-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-microservice-pipeline-artifacts-*",
        "arn:aws:s3:::my-microservice-pipeline-artifacts-*/*"
      ]
    }
  ]
}
EOF

# 绑定自定义策略
aws iam put-role-policy \
  --role-name CodeBuildServiceRole \
  --policy-name CodeBuildServicePolicy \
  --policy-document file://codebuild-service-policy.json
```

**3. 创建 CodeBuild 项目**

```bash
# 获取账户ID和ECR仓库URI
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.us-west-2.amazonaws.com"

# 确保 ECR 仓库存在
aws ecr describe-repositories --repository-names my-microservice --region us-west-2 || \
aws ecr create-repository --repository-name my-microservice --region us-west-2

# 创建 CodeBuild 项目配置
ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.us-west-2.amazonaws.com"

cat > codebuild-project.json << EOF
{
  "name": "my-microservice-build",
  "description": "Build project for my-microservice",
  "source": {
    "type": "CODEPIPELINE",
    "buildspec": ".buildspec.yml"
  },
  "artifacts": {
    "type": "CODEPIPELINE"
  },
  "environment": {
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/amazonlinux2-x86_64-standard:3.0",
    "computeType": "BUILD_GENERAL1_SMALL",
    "environmentVariables": [
      {
        "name": "AWS_REGION",
        "value": "us-west-2"
      },
      {
        "name": "ECR_REGISTRY",
        "value": "${ECR_REGISTRY}"
      },
      {
        "name": "IMAGE_REPO_NAME",
        "value": "my-microservice"
      }
    ],
    "privilegedMode": true
  },
  "serviceRole": "arn:aws:iam::${ACCOUNT_ID}:role/CodeBuildServiceRole"
}
EOF

# 创建 CodeBuild 项目
aws codebuild create-project \
  --cli-input-json file://codebuild-project.json \
  --region us-west-2
```

**4. 创建 GitHub 连接**

```bash
# 创建 GitHub 连接
aws codestar-connections create-connection \
  --provider-type GitHub \
  --connection-name my-github-connection \
  --region us-west-2

# 完成连接授权（需要在AWS控制台完成）：
# 1. 前往 CodePipeline > Settings > Connections
# 2. 找到连接并点击 "Update pending connection"
# 3. 授权 AWS 访问 GitHub 账户
```

**5. 创建 CodePipeline 管道**

```bash
# 设置变量（请替换为实际值）
GITHUB_CONNECTION_ARN="arn:aws:codestar-connections:us-west-2:${ACCOUNT_ID}:connection/your-connection-id"
GITHUB_REPO="your-username/my-microservice"

# 创建S3存储桶用于Pipeline工件存储
BUCKET_NAME="my-microservice-pipeline-artifacts-${ACCOUNT_ID}-us-west-2"
aws s3 mb s3://${BUCKET_NAME} --region us-west-2

# 创建 Pipeline 配置（Docker镜像存储在ECR，构建工件存储在S3）
cat > pipeline-config.json << EOF
{
  "pipeline": {
    "name": "my-microservice-pipeline",
    "roleArn": "arn:aws:iam::${ACCOUNT_ID}:role/CodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "${BUCKET_NAME}"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "SourceAction",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeStarSourceConnection",
              "version": "1"
            },
            "configuration": {
              "ConnectionArn": "${GITHUB_CONNECTION_ARN}",
              "FullRepositoryId": "${GITHUB_REPO}",
              "BranchName": "main",
              "OutputArtifactFormat": "CODE_ZIP"
            },
            "outputArtifacts": [
              {
                "name": "SourceOutput"
              }
            ]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "BuildAction",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "my-microservice-build"
            },
            "inputArtifacts": [
              {
                "name": "SourceOutput"
              }
            ],
            "outputArtifacts": [
              {
                "name": "BuildOutput"
              }
            ]
          }
        ]
      },
      {
        "name": "Deploy",
        "actions": [
          {
            "name": "DeployAction",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeploy",
              "version": "1"
            },
            "configuration": {
              "ApplicationName": "my-ecs-app",
              "DeploymentGroupName": "my-deployment-group"
            },
            "inputArtifacts": [
              {
                "name": "BuildOutput"
              }
            ]
          }
        ]
      }
    ]
  }
}
EOF

# 创建 Pipeline
aws codepipeline create-pipeline \
  --cli-input-json file://pipeline-config.json \
  --region us-west-2
```

**6. 创建 appspec.yml（用于 CodeDeploy）**

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "<TASK_DEFINITION_ARN>"
        LoadBalancerInfo:
          ContainerName: "my-container"
          ContainerPort: 3000
```

**7. 验证管道**

```bash
# 查看管道状态
aws codepipeline get-pipeline-state \
  --name my-microservice-pipeline \
  --region us-west-2

# 手动启动管道（测试用）
aws codepipeline start-pipeline-execution \
  --name my-microservice-pipeline \
  --region us-west-2
```

#### 4.3 发布流程：

- 开发新功能：
  - 从 `develop` 创建 `feature/add-login` 分支，开发功能。
  - 提交 PR，合并回 `develop`。

- 准备发布：

  - 从 `develop` 创建 `release/1.2.3` 分支，更新 `package.json` 版本号为 `1.2.3`。

  - **如果main分支没有保护规则（直接合并方式）**：

    ```bash
    git checkout main
    git merge release/1.2.3
    git tag v1.2.3
    git push origin main v1.2.3
    ```

  - **如果main分支开启了保护规则（推荐的PR方式）**：

    1. 推送release分支到远程：
    
       ```bash
       git push origin release/1.2.3
       ```

    2. 在GitHub上创建Pull Request：
       - 源分支：`release/1.2.3`
       - 目标分支：`main`
       - 标题：`Release v1.2.3`
       - 描述：包含本次发布的功能和修复说明

    3. 经过代码审查和CI检查后，合并PR

    4. 合并后在main分支上打标签：
    
       ```bash
       git checkout main
       git pull origin main
       git tag v1.2.3
       git push origin v1.2.3
       ```

    5. 将变更合并回develop分支：
    
       ```bash
       git checkout develop
       git merge main
       git push origin develop
       ```

- 触发 CI/CD：
  - GitHub 标签 v1.2.3 触发 CodePipeline。
  - CodeBuild 构建 Docker 镜像，推送至 ECR。
  - CodeDeploy 执行 Canary 部署（10% 流量 5 分钟后全切换）。
- 验证：
  - 访问 ALB DNS（如 my-alb-1234567890.us-west-2.elb.amazonaws.com）。
  - 初始 10% 请求返回 Version: 1.2.3，5 分钟后全切换。
  - 使用 CloudWatch 监控部署健康。

#### 4.4 紧急修复

- 从 `main` 创建 `hotfix/1.2.4` 分支，修复问题。
- 更新 `package.json` 版本号为 `1.2.4`。
- 合并到 `main`，打标签 `v1.2.4`，推送到 GitHub。
- 合并回 `develop`，触发 `CodePipeline` 和 `CodeDeploy`。

### 5. 优化与监控及注意事项

- 性能优化：

  - 使用 CodeBuild 的缓存（如 Docker 层缓存）加速构建。

  - 配置 ECS Fargate 的 CPU 和内存（256 CPU，512 MB 内存）以匹配负载。

  - 设置 ALB 健康检查，确保快速检测故障。

- 监控：

  - 使用 CloudWatch 监控 ECS 任务健康、ALB 请求延迟和 CodeDeploy 部署状态。

  - 设置警报（如部署失败或任务失败）通知团队。
  - CodeDeploy 自动回滚失败部署，需监控 CloudWatch 日志。

- 成本优化：

  - 配置 ECR 仓库的生命周期策略，自动清理旧版本镜像。
  - 优化镜像大小，使用 multi-stage builds 和 .dockerignore。
  - 定期清理未使用的 ECR 镜像，降低存储成本。

- 权限策略：

  - 确保 IAM 角色（如 ecsTaskExecutionRole、CodeDeployServiceRole）配置正确。

    - **创建 ecsTaskExecutionRole 并绑定自定义策略**：

      1. 创建自定义策略（保存为 ecs-task-execution-policy.json）：

         ```json
         {
             "Version": "2012-10-17",
             "Statement": [
                 {
                     "Effect": "Allow",
                     "Action": [
                         "ecr:GetAuthorizationToken",
                         "ecr:BatchCheckLayerAvailability",
                         "ecr:GetDownloadUrlForLayer",
                         "ecr:BatchGetImage",
                         "logs:CreateLogStream",
                         "logs:PutLogEvents"
                     ],
                     "Resource": "*"
                 }
             ]
         }
         ```

      2. 创建角色并绑定信任策略：

         ```bash
         aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document '{
           "Version": "2012-10-17",
           "Statement": [
             {
               "Effect": "Allow",
               "Principal": {
                 "Service": "ecs-tasks.amazonaws.com"
               },
               "Action": "sts:AssumeRole"
             }
           ]
         }'
         ```

      3. 创建并绑定自定义策略：

         ```bash
         aws iam create-policy --policy-name MyECSTaskExecutionPolicy --policy-document file://ecs-task-execution-policy.json
         aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::<your-account-id>:policy/MyECSTaskExecutionPolicy
         ```

    - **ecsTaskRole 只需信任策略**：
      仅允许 ECS 任务以该角色身份运行，不授予任何 AWS 资源访问权限，符合最小权限原则。如后续业务需要再补充权限策略。

- 版本一致性：

  - 严格遵循 SemVer，确保 `package.json` 和 Git 标签一致。

### 6. 示例工作流

#### 6.1 基础工作流（无分支保护）

```bash
# 开发新功能
git checkout develop
git checkout -b feature/add-login
# 开发代码
git add .
git commit -m "Add login feature"
git push origin feature/add-login
# 提交 PR，合并到 develop

# 准备发布
git checkout develop
git pull origin develop
git checkout -b release/1.2.3
# 更新 package.json 版本
git add .
git commit -m "Prepare release 1.2.3"
git push origin release/1.2.3

# 测试通过后，直接合并到main
git checkout main
git pull origin main
git merge release/1.2.3
git tag v1.2.3
git push origin main v1.2.3

# 合并回 develop
git checkout develop
git merge release/1.2.3
git push origin develop

# 清理release分支
git branch -d release/1.2.3
git push origin --delete release/1.2.3
```

#### 6.2 推荐工作流（开启分支保护）

```bash
# 开发新功能
git checkout develop
git pull origin develop
git checkout -b feature/add-login
# 开发代码
git add .
git commit -m "Add login feature"
git push origin feature/add-login
# 在GitHub上创建PR: feature/add-login → develop
# 经过代码审查后合并PR

# 准备发布
git checkout develop
git pull origin develop  # 拉取最新的develop
git checkout -b release/1.2.3
# 更新 package.json 版本号为 1.2.3
vim package.json  # 或使用其他编辑器
git add package.json
git commit -m "Bump version to 1.2.3"
git push origin release/1.2.3

# 在GitHub上创建PR: release/1.2.3 → main
# PR标题: "Release v1.2.3"
# PR描述: 详细说明本次发布的功能和修复
# 经过代码审查和CI检查后合并PR

# PR合并后，在本地打标签
git checkout main
git pull origin main
git tag v1.2.3
git push origin v1.2.3

# 创建PR将main的变更合并回develop
# 在GitHub上创建PR: main → develop
# 或者在本地合并（如果develop没有保护规则）
git checkout develop
git pull origin develop
git merge main
git push origin develop

# 清理release分支
git branch -d release/1.2.3
git push origin --delete release/1.2.3
```

#### 6.3 GitHub CLI 自动化示例（可选）

如果安装了GitHub CLI，可以自动化PR创建过程：

```bash
# 安装GitHub CLI: https://cli.github.com/

# 创建release到main的PR
gh pr create \
  --base main \
  --head release/1.2.3 \
  --title "Release v1.2.3" \
  --body "Release version 1.2.3 with new features and bug fixes"

# 查看PR状态
gh pr status

# 合并PR（需要满足所有保护规则）
gh pr merge --squash --delete-branch

# 创建main到develop的PR
gh pr create \
  --base develop \
  --head main \
  --title "Merge release v1.2.3 back to develop" \
  --body "Sync main branch changes back to develop"
```

#### 6.4 分支保护规则推荐配置

在GitHub仓库Settings > Branches中，为main和develop分支设置保护规则：

**main分支保护规则：**
- ✅ Require pull request reviews before merging
- ✅ Require status checks to pass before merging
- ✅ Require branches to be up to date before merging
- ✅ Require signed commits (可选)
- ✅ Include administrators

**develop分支保护规则：**
- ✅ Require pull request reviews before merging
- ✅ Require status checks to pass before merging
- ✅ Require branches to be up to date before merging

---

## 常见问题解答 (FAQ)

### 部署和配置问题

#### Q1: 出现 `head: |: No such file or directory` 错误怎么办？

**问题**: 在执行AWS CLI命令时出现终端显示错误
**原因**: PAGER环境变量配置错误 (`head -n 10000 | cat`)
**解决方案**: 
```bash
unset PAGER
export AWS_PAGER=""
echo 'export AWS_PAGER=""' >> ~/.zshrc
```

#### Q2: AppSpec配置错误 `INVALID_REVISION: Could not parse or validate one of the resources`？

**问题**: CodeDeploy无法解析AppSpec文件
**原因**: TaskDefinition使用占位符 `<TASK_DEFINITION>` 而不是完整ARN
**解决方案**: 更新appspec.yml使用完整ARN
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:us-west-2:933505494323:task-definition/my-task-definition:2"
        LoadBalancerInfo:
          ContainerName: "my-container"
          ContainerPort: 3000
```

#### Q3: S3权限错误 `IAM_ROLE_PERMISSIONS: CodeDeployServiceRole does not give permission to perform operations in Amazon S3`？

**问题**: CodeDeploy服务角色缺少S3访问权限
**原因**: CodePipeline使用S3存储构建工件（如appspec.yml、taskdef.json），CodeDeploy需要从S3获取这些部署工件
**解决方案**: 添加S3权限到CodeDeployServiceRole
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetBucketLocation",
    "s3:GetObject", 
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::my-microservice-pipeline-artifacts-933505494323-us-west-2",
    "arn:aws:s3:::my-microservice-pipeline-artifacts-933505494323-us-west-2/*"
  ]
}
```

**重要说明**: 
- **Docker镜像**: 存储在ECR中，提供高效的容器镜像管理
- **构建工件**: 存储在S3中，包括appspec.yml、taskdef.json等部署配置文件
- **工件传递**: CodePipeline的各阶段通过S3传递构建工件，这是AWS CodePipeline的标准工作方式

#### Q4: ElasticLoadBalancing权限错误？

**问题**: `does not give permission to perform operations in Amazon ElasticLoadBalancing`
**解决方案**: 添加ELB权限到 `aws-permissions/codedeploy-policy.json`
```json
"elasticloadbalancing:DescribeLoadBalancers",
"elasticloadbalancing:DescribeListeners", 
"elasticloadbalancing:DescribeRules",
"elasticloadbalancing:ModifyListener",
"elasticloadbalancing:CreateTargetGroup",
"elasticloadbalancing:DeleteTargetGroup",
"elasticloadbalancing:RegisterTargets",
"elasticloadbalancing:DeregisterTargets"
```

#### Q5: ECS TaskSet权限错误 `User is not authorized to perform: ecs:CreateTaskSet`？

**问题**: ECS Blue/Green部署需要TaskSet操作权限
**解决方案**: 添加ECS TaskSet权限到 `aws-permissions/codedeploy-policy.json`
```json
"ecs:CreateTaskSet",
"ecs:UpdateTaskSet", 
"ecs:DeleteTaskSet",
"ecs:DescribeTaskSets",
"ecs:UpdateServicePrimaryTaskSet"
```

### 配置匹配问题

#### Q6: 如何确保服务名称配置匹配？

**关键点**: 以下配置必须保持一致
- `service-definition.json`: `"serviceName": "my-service"`
- CodeDeploy部署组: `"serviceName": "my-service"`
- 集群名称: `"cluster": "my-cluster"` / `"clusterName": "my-cluster"`
- 容器配置: `"my-container:3000"`
- 目标组: `my-target-group` / `my-target-group-green`

#### Q7: Pipeline阶段状态含义？

**Pipeline阶段说明:**
- ✅ **Source**: GitHub源代码拉取成功
- ✅ **Build**: Docker构建和ECR推送成功  
- 🔄 **Deploy**: ECS Blue/Green部署进行中
- ❌ **Failed**: 检查具体错误消息

### 故障排除

#### Q8: 如何监控Pipeline执行？

```bash
# 检查Pipeline状态
aws codepipeline get-pipeline-state --name my-microservice-pipeline --region us-west-2 --query 'stageStates[*].[stageName,latestExecution.status]' --output table

# 检查CodeDeploy部署状态
aws deploy list-deployments --application-name my-ecs-app --deployment-group-name my-deployment-group --include-only-statuses InProgress --region us-west-2

# 获取部署详情
aws deploy get-deployment --deployment-id <deployment-id> --region us-west-2
```

#### Q9: 如何手动触发Pipeline？

```bash
aws codepipeline start-pipeline-execution --name my-microservice-pipeline --region us-west-2
```

### 最佳实践

#### Q11: IAM权限配置策略？

1. **渐进式权限**: 根据错误逐步添加权限，避免过度授权
2. **权限分离**: 不同服务使用专门的服务角色
3. **资源限制**: 尽可能指定具体资源ARN而不是使用 `*`
4. **定期审计**: 定期检查和清理不必要的权限

#### Q12: Blue/Green部署注意事项？

1. **健康检查**: 确保容器健康检查配置正确
2. **目标组配置**: 生产和测试目标组必须存在
3. **流量切换**: 合理配置流量切换时间和策略
4. **回滚准备**: 确保能够快速回滚到上一版本

---

## 总结

本方案实现了基于AWS CodePipeline和CodeDeploy的完整GitFlow CI/CD流程，通过渐进式解决权限和配置问题，最终实现了从源代码到生产环境的自动化部署。关键成功因素包括：

1. **严格的配置匹配** - 确保所有服务名称、集群、容器配置一致
2. **渐进式权限配置** - 根据实际错误逐步完善IAM权限
3. **完整的AppSpec配置** - 使用正确的TaskDefinition ARN格式
4. **系统化的故障排除** - 建立完整的监控和诊断流程

通过本方案，小型团队可以快速建立现代化的CI/CD流程，提高开发效率和部署可靠性。