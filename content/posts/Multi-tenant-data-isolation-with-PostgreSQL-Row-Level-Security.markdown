---
title: 使用 PostgreSQL 行级安全（RLS）实现多租户数据隔离
author: mingyu110
date: 2025-12-31
tags:
  - PostgreSQL
  - RLS
  - 多租户
  - 数据隔离
  - SaaS
  - 数据库安全
categories:
  - 数据库
  - 架构设计
description: 探讨如何利用 PostgreSQL 行级安全（RLS）功能，在共享数据库模型中实现多租户数据隔离，降低运营成本的同时确保数据安全。
keywords:
  - PostgreSQL Row Level Security
  - 多租户架构
  - 数据分区
  - 租户隔离
  - Pool模型
  - 数据库安全策略
lang: zh-CN
---

# 使用 PostgreSQL 行级安全（RLS）实现多租户数据隔离

对于软件即服务（SaaS）提供商而言，隔离租户数据是一项基本职责。如果某个租户获取了另一个租户的数据，不仅会失去用户信任，还可能对品牌造成永久性损害，甚至导致业务流失。

鉴于此类风险的严重性，制定有效的数据隔离方案至关重要。多租户架构通过为所有租户共享数据存储资源（而非为每个租户复制资源），实现了灵活性提升和运营成本降低。然而，由于在共享模型中难以实施隔离，很多团队可能会妥协于多租户数据模型，转而采用成本更高的 “单租户单数据库” 方案。

在共享数据库模型中，通常只能依赖软件开发人员在每条 SQL 语句中实现适当的检查逻辑。与其他安全问题类似，我们希望以更集中的方式执行租户数据隔离策略，减少对日常代码变动的依赖。

本文面向 SaaS 架构师和开发人员，探讨一种既能享受租户共享数据库的优势，又能实现集中化隔离管控的方案。

## 数据分区方案

多租户系统中常用的三种数据分区模型包括：独立数据库模型（Silo）、共享数据库独立 Schema 模型（Bridge）和共享数据库共享 Schema 模型（Pool）。每种模型在隔离性实施方面各有优劣。

- **独立数据库模型（Silo）**：为每个租户分配独立的数据库实例。这种方式隔离性最强，但基础设施成本更高，且租户接入流程更复杂 —— 因为每个新租户都需要创建和管理新的数据库实例。
- **共享数据库独立 Schema 模型（Bridge）**：多个租户共享同一数据库实例，但每个租户使用独立的 Schema。该模型通过资源共享降低成本，但维护和租户接入流程仍较复杂。
- **共享数据库共享 Schema 模型（Pool）**：所有租户共享数据库实例和命名空间。在此模型中，所有租户数据共存，但每张表或视图都包含一个分区键（通常是租户标识），用于过滤数据。

共享数据库共享 Schema 模型（Pool）能最大程度降低运营成本，减少基础设施代码和维护开销。但该模型难以实施数据访问策略，通常依赖于开发人员在每条 SQL 语句中正确添加`WHERE`子句来过滤租户数据。

有关多租户数据分区的更多信息，请参阅[AWS SaaS Factory 白皮书](https://d1.awsstatic.com/whitepapers/Multi_Tenant_SaaS_Storage_Strategies.pdf)。

## 行级安全（RLS）

通过在数据库层面集中执行关系型数据库管理系统（RDBMS）的隔离策略，可以减轻软件开发人员的负担。这使得我们既能享受共享数据库模型（Pool）的优势，又能降低跨租户数据访问的风险。

PostgreSQL 9.5 及更高版本提供了行级安全（Row Level Security，RLS）功能。当为表定义安全策略后，这些策略会限制`SELECT`查询返回的行，以及`INSERT`、`UPDATE`、`DELETE`命令影响的行。[Amazon 关系型数据库服务（RDS）](https://aws.amazon.com/rds/)的 Amazon Aurora for PostgreSQL 和 RDS for PostgreSQL 引擎均支持 RLS。更多信息请参见 PostgreSQL 官网的[行安全策略](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)文档。

RLS 策略具有名称，可通过`ALTER`语句应用于表或从表中移除。策略通过`USING`子句定义，该子句返回布尔值，用于判断是否处理表中的某一行。一张表可以同时应用多个策略，以实现复杂的安全控制。此外，策略可覆盖所有语句类型（`SELECT`、`INSERT`、`UPDATE`、`DELETE`），也可针对读取和修改操作设置不同策略。如果为`SELECT`和修改操作设置不同策略，则**必须**在策略定义中包含`WITH CHECK`子句。

可以将 RLS 策略理解为数据库引擎自动管理的`WHERE`子句。

## 代码示例

### 创建带 RLS 策略的表

以下代码展示了如何在表定义中创建 RLS 策略：

```sql
-- 创建租户表，对主键和租户名称创建索引
CREATE TABLE tenant (
    tenant_id UUID DEFAULT uuid_generate_v4() PRIMARY KEY, -- 租户唯一标识，默认使用UUID生成器生成
    name VARCHAR(255) UNIQUE, -- 租户名称，唯一约束（确保租户名称不重复）
    status VARCHAR(64) CHECK (status IN ('active', 'suspended', 'disabled')), -- 租户状态，仅允许"active"（活跃）、"suspended"（暂停）、"disabled"（禁用）
    tier VARCHAR(64) CHECK (tier IN ('gold', 'silver', 'bronze')) -- 租户等级，仅允许"gold"（金牌）、"silver"（银牌）、"bronze"（铜牌）
);

-- 创建租户用户表
CREATE TABLE tenant_user (
    user_id UUID DEFAULT uuid_generate_v4() PRIMARY KEY, -- 用户唯一标识，默认使用UUID生成器生成
    tenant_id UUID NOT NULL REFERENCES tenant (tenant_id) ON DELETE RESTRICT, -- 关联的租户ID，非空且引用tenant表的tenant_id，删除租户时限制删除（防止误删）
    email VARCHAR(255) NOT NULL UNIQUE, -- 用户邮箱，非空且唯一（确保邮箱不重复）
    given_name VARCHAR(255) NOT NULL CHECK (given_name <> ''), -- 名，非空且不允许空字符串
    family_name VARCHAR(255) NOT NULL CHECK (family_name <> '') -- 姓，非空且不允许空字符串
);

-- 启用租户表的行级安全（RLS）
ALTER TABLE tenant ENABLE ROW LEVEL SECURITY;

-- 限制读写操作，使租户只能看到自己的数据行
-- 将tenant_id的UUID类型转换为与current_user返回值匹配的文本类型
-- 此策略隐含与USING子句匹配的WITH CHECK条件
CREATE POLICY tenant_isolation_policy ON tenant
USING (tenant_id::TEXT = current_user); -- 仅允许当前数据库用户与tenant_id匹配的行被访问

-- 对租户用户表执行相同配置
ALTER TABLE tenant_user ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_user_isolation_policy ON tenant_user
USING (tenant_id::TEXT = current_user); -- 仅允许当前数据库用户与tenant_id匹配的行被访问
```

对于大型数据集，建议使用数字序列作为主键，而非随机 UUID，以提高可扩展性和性能。更多信息请参见 PostgreSQL 官网的[UUID-OSSP](https://www.postgresql.org/docs/current/uuid-ossp.html)文档。

## RLS 的使用方式

基于上述表和策略定义，假设存在一个共享的系统级角色（用户），供 SaaS 提供商用于数据库初始化和租户接入。以下示例使用 PostgreSQL 的命令行客户端[psql](https://www.postgresql.org/docs/current/app-psql.html)演示。

### 系统级角色的访问权限

如果以 SaaS 提供商的系统级角色（租户接入角色）登录，可查看所有租户记录。这是因为默认情况下，表所有者不受安全策略限制，除非通过`FORCE ROW LEVEL SECURITY`修改表配置。示例如下：

```powershell
rls_multi_tenant=> SELECT * FROM tenant;
              tenant_id               |    name  | status | tier 
--------------------------------------+----------+--------+------
 1cf1cc14-dd34-4a7b-b87d-adf79b2c255c | Tenant 1 | active | gold
 69ad9212-f5ef-456d-a724-dd8ea3c80d61 | Tenant 2 | active | gold
(2 行记录)
```

同时也能查看所有租户用户：

```powershell
rls_multi_tenant=> SELECT tenant_id, user_id, given_name || ' ' || family_name AS name FROM tenant_user;
              tenant_id               |               user_id                |      name       
--------------------------------------+--------------------------------------+-----------------
 1cf1cc14-dd34-4a7b-b87d-adf79b2c255c | d9f7d636-69a0-40d4-96d9-d429d1e1cee3 | User 1 Tenant 1
 69ad9212-f5ef-456d-a724-dd8ea3c80d61 | eb7a503a-a7c6-44c0-9916-8df68dd96815 | User 1 Tenant 2
(2 行记录)
```

### 租户角色的访问限制

如果以非系统用户（如租户 1 的角色）登录数据库，可观察行级安全策略的效果。首先确认当前登录用户为租户 1：

```powershell
rls_multi_tenant=> SELECT current_user;
             current_user             
--------------------------------------
 1cf1cc14-dd34-4a7b-b87d-adf79b2c255c
(1 行记录)
```

`SELECT`语句的安全策略执行时不会返回错误或提示信息，不匹配策略`USING`条件的行将直接从结果集中排除。示例如下：

```powershell
rls_multi_tenant=> SELECT * FROM tenant;
              tenant_id               |    name  | status | tier 
--------------------------------------+----------+--------+------
 1cf1cc14-dd34-4a7b-b87d-adf79b2c255c | Tenant 1 | active | gold
(1 行记录)
```

即使尝试暴力访问其他租户的信息，策略也会进行保护：

```powershell
rls_multi_tenant=> SELECT * FROM tenant WHERE tenant_id = '69ad9212-f5ef-456d-a724-dd8ea3c80d61'::UUID;
 tenant_id | name | status | tier 
-----------+------+--------+------
(0 行记录)
```

`UPDATE`和`DELETE`语句的策略执行方式类似，不匹配策略的行不会被操作：

```powershell
rls_multi_tenant=> UPDATE tenant_user SET given_name = 'Cross Tenant Access' WHERE user_id = 'eb7a503a-a7c6-44c0-9916-8df68dd96815'::UUID;
UPDATE 0 -- 未更新任何行（目标行属于其他租户）

rls_multi_tenant=> DELETE FROM tenant WHERE tenant_id = '69ad9212-f5ef-456d-a724-dd8ea3c80d61'::UUID;
DELETE 0 -- 未删除任何行（目标行属于其他租户）
```

但`INSERT`语句若违反安全策略，会返回错误：

```powershell
rls_multi_tenant=> INSERT INTO tenant (name) VALUES ('Tenant 3');
ERROR:  new row violates row-level security policy for table "tenant"
```

作为租户 1，无法插入新记录，因为`tenant_id`列的值（此处为自动生成）与当前用户身份不匹配。如果在插入时指定自己的身份 ID，会触发唯一键冲突错误。

## 使用 RLS 的注意事项

PostgreSQL 超级用户以及任何具有`BYPASSRLS`属性的角色**不受**表策略限制。此外，默认情况下，表所有者会绕过 RLS 策略，除非通过`FORCE ROW LEVEL SECURITY`修改表配置。这也是前文示例中，创建`tenant`和`tenant_user`表的系统级角色能访问所有行的原因。

如果应用程序代码使用与表所有者相同的 PostgreSQL 角色（通常是执行`CREATE TABLE`语句的用户，除非后续修改）连接数据库，安全策略**默认不生效**。

出于安全和监控考虑，应用程序应使用**非数据库对象所有者**的用户连接数据库。

### 策略`USING`子句的设计

前文示例中使用`tenant_id = current_user`，这意味着当前连接的 PostgreSQL 角色名称必须与行的`tenant_id`列值匹配才能访问。若采用此机制，需为**每个租户**创建 PostgreSQL 角色，这会增加维护成本且难以扩展。

## 替代方案

如果不想为每个租户创建和维护 PostgreSQL 用户，可使用共享的 PostgreSQL 登录账号供应用程序使用。但需定义一个运行时参数来存储应用程序的当前租户上下文。**确保登录账号不是表所有者，且未定义`BYPASSRLS`属性**。这种更具可扩展性的方案如下：

```sql
-- 租户隔离策略：通过会话变量匹配租户ID
CREATE POLICY tenant_isolation_policy ON tenant
USING (tenant_id = current_setting('app.current_tenant')::UUID); -- 租户ID需与会话变量app.current_tenant的值匹配（转换为UUID类型）
```

不再将当前登录的 PostgreSQL 用户与`tenant_id`列比较，而是使用内置函数`current_setting`读取名为`app.current_tenant`的配置变量的值（并转换为 UUID 类型，因为`tenant_id`列定义为 UUID）。变量必须采用 “前缀。变量名” 格式，无前缀的变量定义在`postgresql.conf`文件中，而 RDS 实例中无法访问该文件。

### 运行时参数的设置方式

可通过内置函数`set_config`或[SQL 命令`SET`](https://www.postgresql.org/docs/current/sql-set.html)定义`app.current_tenant`的值，声明作用于当前数据库连接会话的运行时参数。应用程序代码应在创建数据库连接或从连接池获取现有连接时执行此声明。由于 PostgreSQL 将这些变量作用于当前会话，因此在多连接应用中使用是安全的 —— 每个连接都有独立的变量副本，无法访问或修改其他连接的运行时参数。

注意：会话变量可能与服务器端连接池（如`pgBouncer`）不兼容。请务必评估连接池策略的影响，并测试其是否共享会话状态。

## 示例实现

以下代码示例展示了一种设置运行时参数的方式。示例使用 Java 语言，但核心逻辑在其他语言中类似。

在 Java 数据库连接（JDBC）中，代码可通过`javax.sql.DataSource`实例并重写`getConnection()`方法，确保每次应用程序（或连接池库）获取数据库连接时，都能设置正确的租户上下文，使表的 RLS 策略生效。示例如下：

```java
// 每次应用程序从数据源获取连接时，
// 设置PostgreSQL会话变量为当前租户，以强制数据隔离。
@Override
public Connection getConnection() throws SQLException {
    // 调用父类方法获取连接
    Connection connection = super.getConnection();
    // 使用try-with-resources自动关闭Statement
    try (Statement sql = connection.createStatement()) {
        // 执行SQL设置会话变量，值从TenantContext获取当前租户ID
        sql.execute("SET app.current_tenant = '" + TenantContext.getTenant() + "'");
    }
    return connection;
}
```

与 psql 命令行客户端类似，PostgreSQL 的 JDBC 驱动不会将 RLS 策略的触发视为异常。如果查询不满足策略的`USING`条件，会如同表中不存在这些行一样，返回空结果集。

这种数据库层面的安全保护意味着，开发人员编写的每条 SQL 语句都可以保持一致（无需关注租户上下文），由 PostgreSQL 自动执行隔离。开发人员只需针对业务场景编写正确的`WHERE`子句，无需担心在共享多租户数据库中的操作风险。

请务必彻底测试函数、存储过程、视图和复杂嵌套查询，确保策略定义不会导致意外的限制或权限问题。

## 结论

通过利用 PostgreSQL 的行级安全（RLS）功能，可构建采用共享数据库模型（Pool）的 SaaS 应用，既能共享数据库资源，又能降低隔离策略的实施风险和成本。RLS 将隔离控制集中到 PostgreSQL 后端，摆脱了对开发人员日常编码的依赖。

共享数据库模型（Pool）避免了为每个租户重复资源的高成本，以及设置和维护这些资源所需的专用基础设施代码。由于资源更少且所有租户数据集中存储，更容易实现平台的统一运营视图，同时简化数据库备份和恢复流程（因组件更少）。

如果当前正在使用独立数据库模型（Silo）或共享数据库独立 Schema 模型（Bridge）实现租户数据隔离，不妨考虑 RLS，以获得更灵活、更经济的方案。