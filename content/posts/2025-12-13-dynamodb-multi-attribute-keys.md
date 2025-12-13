---
title: "Amazon DynamoDB 全局二级索引多属性复合键技术详解"
date: 2025-12-13
draft: false
tags: ["DynamoDB", "NoSQL", "数据库设计", "AWS", "GSI", "复合键"]
categories: ["技术方案", "数据库"]
author: "mingyu110"
---

# Amazon DynamoDB 全局二级索引多属性复合键技术文档

## 概述

Amazon DynamoDB 近期发布了一项更新：全局二级索引（GSI）原生支持多属性复合键。对于架构师、无服务器开发人员和AWS工程师来说，这是一项重大突破。

在此之前，开发者必须手动将多个值连接合成为复合键（例如：`user#123#region#eu#status#active`）才能支持复杂的查询模式。这种方法虽然可行，但需要额外开销，必须在一个键中维护多个值，并且强制将所有内容转换为字符串类型。

现在，用户可以为GSI分区键定义最多4个属性，为排序键也定义最多4个属性，直接使用原生属性类型。这带来了更清晰的数据模型、更好的类型安全性，以及更灵活的查询能力。

## 实际案例：多租户CRM系统

假设正在构建一个CRM应用，用户属于不同组织，希望根据以下条件查询客户：

- 组织ID
- 团队ID
- 客户细分
- 最近活动日期

### 传统方式

```javascript
GSI1PK: `ORG#${orgId}#TEAM#${teamId}`,
GSI1SK: `SEG#${segment}#TS#${timestamp}`
```

### 新的多属性GSI方式

```javascript
PartitionKey:
  - orgId (S)
  - teamId (S)
  - segment (S)

SortKey:
  - lastInteraction (N)
```

查询示例：

```javascript
const { DynamoDBDocumentClient, QueryCommand } = require("@aws-sdk/lib-dynamodb");
const dynamo = DynamoDBDocumentClient.from(new DynamoDBClient({}));

const response = await dynamo.send(
  new QueryCommand({
    TableName: "crm",
    IndexName: "CustomerByOrgTeamSegment",
    KeyConditionExpression:
      "orgId = :org AND teamId = :team AND segment = :segment AND lastInteraction > :ts",
    ExpressionAttributeValues: {
      ":org": "org#A1",
      ":team": "marketing",
      ":segment": "enterprise",
      ":ts": 1700000000,
    },
  })
);
```

无需字符串连接、无需解析、没有歧义，数字比较现在也更加准确。

## 核心概念

### 分区键和排序键基础

**分区键**：
- 代表查询中的已知元素（如user_id、account_id等）
- 决定数据的存储位置
- 必须在高效的API操作（如Query API）中提供

**排序键**：
- 可选属性，用于将相关项目存储在一起
- 支持范围操作进行高效查询
- 使一对多关系成为可能，生成项目集合

### 传统方式的局限

在传统方式中，当需要在排序键中查询多个属性时，常见的模式是将属性连接成复合排序键。例如：

```
PK: C#1A2B3C
SK: ORDER#PENDING#2024-11-01T10:30:00Z
```

这种方式需要：
- 写入数据时进行字符串连接
- 读取数据时进行解析
- 向现有表添加GSI时需要回填合成键

## 多属性复合键特性

### 主要优势

1. **多分区键**：最多可组合4个属性作为分区键（如租户、客户、部门）
2. **多排序键**：最多可定义4个排序键属性，支持特定查询模式
3. **原生数据类型**：每个属性保持其类型（字符串、数字或二进制），无需字符串转换和连接
4. **高效查询**：可以使用越来越具体的属性组合进行查询，无需重构数据
5. **简单的分片技术**：使用两个或多个分区键来降低热分区风险

### 查询模式支持

- 查询要求对所有分区键属性使用等值条件（`=`）
- 排序键条件是可选的，最多可对4个属性使用等值（`=`）条件
- 范围条件（`<`、`>`、`<=`、`>=`、`BETWEEN`、`BEGINS_WITH`）仅在最后一个排序键属性上支持
- 不能跳过排序键查询，例如不能仅使用第一和第三个排序键进行查询
- 可以提供部分排序键

## 最佳实践

虽然最佳实践没有太大变化，但这些实践将始终决定查询的效率：

### 1. 仔细设计属性顺序

- **分区键属性**：在查询中必须全部提供
- **排序键条件**：从左到右应用，将最具选择性的排序属性放在最后

### 2. 保持属性类型的逻辑性

- **数字类型**：用于时间戳或指标
- **字符串类型**：用于实体标识（如orgId、teamId）

### 3. 避免过度索引

- 不要因为可以使用8个属性就全部使用
- 使用此功能解决具体的查询模式，而不是存储所有项目元数据

### 4. 规划向后兼容性

- 如果当前使用合成键，考虑引入额外的GSI使用多属性键，而不是立即替换

### 5. 策略性使用稀疏索引

- 仍然可以省略属性以避免索引不需要的项目

## 使用场景示例

### 场景1：订单仪表板

**访问模式**：
- 查询客户的所有订单
- 获取特定状态的订单
- 检索两个日期之间的待处理订单

**传统设计**：
```javascript
PK: customerId
SK: orderId
GSI1PK: customerId
GSI1SK: `ORDER#${status}#${orderDate}`
```

**多属性GSI设计**：
```javascript
Base table:
  PK: orderId
  SK: customerId

GSI1:
  PK: customerId
  SK: status, orderDate
```

**查询示例**：
```javascript
// 查询客户的所有订单
KeyConditionExpression: "customerId = :cid"

// 查询客户的待处理订单
KeyConditionExpression: "customerId = :cid AND status = :status"

// 查询客户在日期范围内的待处理订单
KeyConditionExpression: "customerId = :cid AND status = :status AND orderDate BETWEEN :start AND :end"
```

### 场景2：电商订单管理

**数据模型**：
- 基础表：订单ID作为分区键
- GSI：客户ID作为分区键，状态和订单日期作为多属性排序键

**优势**：
- 无需手动连接状态和时间戳
- 支持按状态筛选，然后按日期范围查询
- 保持数据类型的原生性（日期作为数字类型）

### 场景3：分层组织数据

**场景**：组织结构为区域 > 国家 > 业务部门 > 团队

**传统方式**：
```
PK: orgId
SK: `REGION#${region}#COUNTRY#${country}#BU#${bu}#TEAM#${team}`
```

**多属性GSI方式**：
```javascript
GSI PK: orgId
GSI SK: region, country, businessUnit, team
```

**优势**：
- 可以查询组织中的所有团队
- 可以查询特定区域的所有团队
- 可以查询特定区域和国家的所有团队
- 支持在任意层级进行范围查询

### 场景4：SaaS多租户

**场景**：多租户应用需要按租户、应用、环境和资源类型查询资源

```javascript
GSI PK: tenantId, appId, environment
GSI SK: resourceType, createdAt
```

**查询能力**：
- 查询租户的所有资源
- 查询租户特定应用的所有资源
- 查询租户特定应用和生产环境的所有资源
- 查询所有虚拟机（按创建时间排序）

### 场景5：时间序列数据

**场景**：物联网传感器读数，按设备、传感器类型和时间查询

```javascript
GSI PK: deviceId, sensorType
GSI SK: timestamp
```

**优势**：
- 高效查询特定设备特定传感器类型的读数
- 支持时间范围查询
- 避免字符串连接导致的时间戳精度问题

## 实施指南

### 创建支持多属性键的表

```javascript
import { DynamoDBClient, CreateTableCommand } from "@aws-sdk/client-dynamodb";

const client = new DynamoDBClient({ region: 'us-west-2' });

const response = await client.send(new CreateTableCommand({
    TableName: 'TournamentMatches',

    // 基础表：简单分区键
    KeySchema: [
        { AttributeName: 'matchId', KeyType: 'HASH' }
    ],

    AttributeDefinitions: [
        { AttributeName: 'matchId', AttributeType: 'S' },
        { AttributeName: 'tournamentId', AttributeType: 'S' },
        { AttributeName: 'region', AttributeType: 'S' },
        { AttributeName: 'round', AttributeType: 'S' },
        { AttributeName: 'bracket', AttributeType: 'S' },
        { AttributeName: 'player1Id', AttributeType: 'S' },
        { AttributeName: 'matchDate', AttributeType: 'S' }
    ],

    // 支持多属性键的GSI
    GlobalSecondaryIndexes: [
        {
            IndexName: 'TournamentRegionIndex',
            KeySchema: [
                { AttributeName: 'tournamentId', KeyType: 'HASH' },    // GSI PK属性1
                { AttributeName: 'region', KeyType: 'HASH' },          // GSI PK属性2
                { AttributeName: 'round', KeyType: 'RANGE' },          // GSI SK属性1
                { AttributeName: 'bracket', KeyType: 'RANGE' },        // GSI SK属性2
                { AttributeName: 'matchId', KeyType: 'RANGE' }         // GSI SK属性3
            ],
            Projection: { ProjectionType: 'ALL' }
        },
        {
            IndexName: 'PlayerMatchHistoryIndex',
            KeySchema: [
                { AttributeName: 'player1Id', KeyType: 'HASH' },       // GSI PK
                { AttributeName: 'matchDate', KeyType: 'RANGE' },      // GSI SK属性1
                { AttributeName: 'round', KeyType: 'RANGE' }           // GSI SK属性2
            ],
            Projection: { ProjectionType: 'ALL' }
        }
    ],

    BillingMode: 'PAY_PER_REQUEST'
}));
```

### 查询多属性GSI

```javascript
import { DynamoDBDocumentClient, QueryCommand } from "@aws-sdk/lib-dynamodb";

// 查询1：使用所有分区键属性
const response1 = await docClient.send(new QueryCommand({
    TableName: 'TournamentMatches',
    IndexName: 'TournamentRegionIndex',
    KeyConditionExpression: 'tournamentId = :tournament AND region = :region',
    ExpressionAttributeValues: {
        ':tournament': 'WINTER2024',
        ':region': 'NA-EAST'
    }
}));

// 查询2：分区键 + 第一个排序键属性
const response2 = await docClient.send(new QueryCommand({
    TableName: 'TournamentMatches',
    IndexName: 'TournamentRegionIndex',
    KeyConditionExpression: 'tournamentId = :tournament AND region = :region AND round = :round',
    ExpressionAttributeValues: {
        ':tournament': 'WINTER2024',
        ':region': 'NA-EAST',
        ':round': 'SEMIFINALS'
    }
}));

// 查询3：分区键 + 所有排序键属性
const response3 = await docClient.send(new QueryCommand({
    TableName: 'TournamentMatches',
    IndexName: 'TournamentRegionIndex',
    KeyConditionExpression: 'tournamentId = :tournament AND region = :region AND round = :round AND bracket = :bracket AND matchId = :matchId',
    ExpressionAttributeValues: {
        ':tournament': 'WINTER2024',
        ':region': 'NA-EAST',
        ':round': 'SEMIFINALS',
        ':bracket': 'UPPER',
        ':matchId': 'match-002'
    }
}));

// 查询4：在最后一个排序键属性上使用范围条件
const response4 = await docClient.send(new QueryCommand({
    TableName: 'TournamentMatches',
    IndexName: 'PlayerMatchHistoryIndex',
    KeyConditionExpression: 'player1Id = :player AND matchDate > :startDate',
    ExpressionAttributeValues: {
        ':player': '101',
        ':startDate': '2024-01-01'
    }
}));
```

## 关键要点

1. **简化数据建模**：无需手动连接属性，直接使用自然域属性
2. **更好的类型安全**：每个属性保持其原生数据类型
3. **灵活的查询能力**：支持多维查询，无需重构数据
4. **无需回填**：向现有表添加新的多属性GSI时，DynamoDB自动索引所有现有项目
5. **成本效益**：该功能在所有支持DynamoDB的AWS区域均不额外收费

## 结论

DynamoDB多属性复合键功能代表了NoSQL数据建模的重大进步。通过消除手动属性连接的需求，它使数据建模更加直观，减少了代码复杂性，并提高了类型安全性。无论是构建多租户SaaS应用、电商系统还是物联网平台，此功能都能帮助您创建更高效、更易维护的数据模型。

---

**来源**：
- AWS官方公告：https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-dynamodb-multi-attribute-composite-keys-global-secondary-indexes/
- DynamoDB开发者指南：https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html
- AWS数据库博客：https://aws.amazon.com/blogs/database/multi-key-support-for-global-secondary-index-in-amazon-dynamodb/
- 多属性键模式文档：https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.DesignPattern.MultiAttributeKeys.html
