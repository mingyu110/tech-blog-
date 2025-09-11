---
title: "多租户RAG中的池化（Pool）模式实践方案指南"
date: 2025-09-12
tags: ["RAG", "GenAI", "AWS", "多租户", "SaaS"]
---

# 多租户RAG中的池化（Pool）模式实践方案指南

本文档旨在详细解析在多租户检索增强生成（RAG）应用中，如何通过池化（Pool）模式实现高效、安全且经济的架构。

## 1. 为什么需要多租户RAG？

在构建软件即服务（SaaS）平台时，通常需要为成百上千的租户（用户或企业）提供服务。如果为每个租户都部署一套独立的RAG基础设施（如独立的向量数据库、S3存储桶和知识库），将会导致以下问题：

- **成本高昂**：每个租户一套资源，即使在闲置时也会产生大量固定成本。
- **运维复杂**：管理成千上万的独立资源，进行监控、升级和维护，操作难度和工作量巨大。
- **扩展性差**：新租户的入驻流程（Onboarding）缓慢，需要手动配置和部署资源，无法实现快速扩展。

因此，采用多租户架构，通过共享基础设施来服务所有租户，成为必然选择。池化模式正是实现多租户RAG的核心策略之一。

## 2. 核心架构

池化模式的核心思想是**“物理共享，逻辑隔离”**。所有租户共享同一套基础设施，但通过**元数据（Metadata）在逻辑上严格隔离各自的数据**。

**架构流程解析:**

1.  **文档上传**: 租户（如`user123`）上传文档。应用后端不仅将文档本身存入共享的S3桶，还会额外生成一个包含租户信息的 `metadata.json` 文件，并一同上传。
2.  **知识库摄入**: 共享的Amazon Bedrock知识库被触发，开始处理S3桶中的新文件。它会同时读取文档原文和对应的 `metadata.json` 文件。
3.  **向量化与存储**: Bedrock知识库将文档分块（Chunking），调用Embedding模型生成向量，然后将向量与从 `metadata.json` 中提取的元数据（如 `userId`, `agentId`）一同存储到共享的向量数据库（如Pinecone）中。
4.  **查询与检索**: 当租户发起查询时，应用后端会向Bedrock发出检索请求，并在请求中注入一个**严格的元数据过滤器**（Filter），确保只在属于当前租户的向量中进行搜索。
5.  **生成响应**: Bedrock将检索到的相关上下文和用户问题一起提交给大语言模型（如Claude 3），生成最终响应并返回给用户。

## 3. 项目代码结构树

一个典型的基于Node.js/NestJS的后端项目结构如下，清晰地展示了各模块的职责：

```
/src
├── /agents                 # 智能体管理模块
│   ├── agents.controller.ts  # API接口
│   ├── agents.service.ts     # 业务逻辑
│   └── agent.schema.ts       # 数据模型
├── /users                  # 用户管理模块
│   ├── users.service.ts      # 用户注册等
│   └── user.schema.ts        # 数据模型
├── /rag                    # RAG核心模块
│   ├── rag.service.ts        # 编排查询、过滤等
│   └── bedrock.client.ts     # 与Bedrock交互的客户端
├── /documents              # 文档管理模块
│   ├── documents.controller.ts # 文档上传接口
│   └── documents.service.ts    # 处理文档和元数据生成
└── main.ts                 # 应用入口
```

## 4. 核心实现与代码注释

### 4.1. `meta.json`文件的生成与重要性

`meta.json` 是实现逻辑隔离的**命脉**。它由应用后端在处理文档上传时动态生成。

**生成过程**：
当用户上传一个文件（如 `report.pdf`）时，`documents.service.ts` 服务会执行以下操作：
1.  从请求的认证信息中获取 `userId` 和 `agentId`。
2.  定义原始文档在S3中的存储路径 `s3Key`。
3.  定义元数据文件在S3中的存储路径 `metadataKey`，通常是在原文件路径后附加 `.metadata.json`。
4.  创建一个JSON对象，包含所有需要用于后续过滤的元数据。
5.  将原始文档和这个JSON对象分别上传到S3对应的路径。

**代码示例** (`documents.service.ts`):
```typescript
// 定义S3存储路径
const s3Key = `users/${userId}/agents/${agentId}/documents/${documentId}/${filename}`;
const metadataKey = `${s3Key}.metadata.json`;

// 构造元数据内容
const metadata = {
  "metadataAttributes": {
    "userId": userId,
    "agentId": agentId,
    "filename": filename,
    "uploadDate": new Date().toISOString()
  }
};

// 上传元数据文件到S3
await this.s3Client.putObject({
  Bucket: this.configService.get('SHARED_S3_BUCKET'),
  Key: metadataKey,
  Body: JSON.stringify(metadata),
  ContentType: 'application/json'
});
```
**如果没有这个文件，数据隔离将彻底失效，所有租户的数据会混在一起，造成严重的数据泄露。**

### 4.2. 共享知识库的摄入任务

所有租户的文档都由同一个知识库处理，大大简化了管理。

**代码示例** (`rag.service.ts`):
```typescript
// 导入Bedrock客户端命令
import { StartIngestionJobCommand } from "@aws-sdk/client-bedrock-agent";

// ...

// 触发共享知识库的摄入任务
// 无需关心是哪个租户的文档，知识库会自动处理S3桶中的所有新文件
await this.bedrockClient.send(new StartIngestionJobCommand({
  knowledgeBaseId: this.configService.get('SHARED_KNOWLEDGE_BASE_ID'),
  dataSourceId: this.configService.get('SHARED_DATA_SOURCE_ID'),
}));
```

### 4.3. 查询时的元数据过滤

查询是保护数据安全的关键环节。必须确保每个查询都带有正确的租户ID过滤器。

**代码示例** (`rag.service.ts` 中构建过滤器):
```typescript
// 为Bedrock的RetrieveAndGenerate API构建过滤器
const filter = {
  // 使用 andAll 确保所有条件都满足
  andAll: [
    {
      // 条件一：userId必须匹配当前发起请求的用户
      equals: {
        key: 'userId',
        value: userId,
      },
    },
    {
      // 条件二：agentId必须匹配当前对话的智能体
      equals: {
        key: 'agentId',
        value: agentId,
      },
    },
  ],
};

// 在调用Bedrock时传入过滤器
const response = await this.bedrockClient.send(new RetrieveAndGenerateCommand({
    // ...其他参数
    retrieveAndGenerateConfiguration: {
        type: 'KNOWLEDGE_BASE',
        knowledgeBaseConfiguration: {
            knowledgeBaseId: this.configService.get('SHARED_KNOWLEDGE_BASE_ID'),
            modelArn: selectedModelArn,
            retrievalConfiguration: {
                vectorSearchConfiguration: {
                    filter: filter // 注入过滤器
                }
            }
        }
    }
}));
```

## 5. 生产落地选型考量

将此架构投入生产时，需要考虑以下几点：

1.  **向量数据库选型**:
    *   **Pinecone**: 完全托管的服务，API简单，与Bedrock集成良好，非常适合快速启动。元数据过滤性能强大。
    *   **Amazon OpenSearch Serverless**: AWS原生方案，无服务器架构，按需扩展，与AWS生态系统无缝集成。需要对OpenSearch有一定了解。
    *   **Milvus/Weaviate**: 开源方案，提供更大的灵活性和控制力，但需要自行部署和维护，运维成本较高。
    *   **选型关键**: 无论选择哪种，**必须确保其元数据过滤（Metadata Filtering）功能高效且稳定**，这是池化模式的基石。

2.  **可扩展性 (Scalability)**:
    *   **摄入端**: 文档上传和摄入是异步过程。可以使用S3事件触发Lambda函数来启动Bedrock摄入任务，实现无服务器的弹性扩展。
    *   **查询端**: 查询API应部署在可自动扩展的服务上，如AWS Lambda或配置了Auto Scaling的ECS/EKS服务，以应对高并发查询请求。

3.  **安全性与隔离**:
    *   **IAM策略**: 最小权限原则。配置严格的IAM角色和策略，确保应用后端服务只有权限读写指定的S3路径和调用Bedrock相关API。
    *   **过滤器逻辑**: 对过滤器逻辑进行严格的单元测试和集成测试，确保在任何情况下都不会发生过滤器失效或被绕过的风险。这是防止数据泄露的最后一道防线。

4.  **成本管理**:
    *   **监控与告警**: 监控Bedrock API调用、向量数据库索引大小和查询单元、S3存储量的增长。为每个租户建立用量追踪机制。
    *   **资源优化**: 定期清理不再使用的文档及其向量。根据租户的活跃度，可以考虑将冷数据归档到成本更低的存储层。

## 6. 总结

多租户RAG的池化模式是一种优雅且实用的架构。它通过**共享基础设施**来最大化资源利用率和降低成本，同时利用**元数据过滤**这一核心机制来保证租户数据的严格逻辑隔离。该模式的成功关键在于 `meta.json` 文件的正确生成和查询时过滤器的强制应用，是构建可扩展、可维护的SaaS级AI应用的理想选择。
