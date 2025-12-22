---
title: "Python SDK 开发最佳实践指南：以 AWS Bedrock AgentCore Memory 为例"
date: 2025-12-21
draft: false
tags: ["Python", "SDK开发", "AWS", "最佳实践", "Bedrock"]
categories: ["技术分享"]
---

# Python SDK 开发最佳实践指南
## 以 AWS Bedrock AgentCore Memory 为例

---

## 执行摘要

本文档是一份面向专业开发团队的 **Python SDK 工程实践指南**，旨在帮助开发者构建生产级、企业级的 Python SDK。文档以 **AWS Bedrock AgentCore Memory SDK** 的真实源代码为案例，深入剖析其设计理念、实现细节和工程实践。

### 为什么选择 AgentCore Memory SDK 作为范例？

AWS Bedrock AgentCore Memory SDK 是一个**工业级 Python SDK 的优秀范例**，具有以下特点：

1. **成熟的架构设计**
   - 清晰的模块划分（控制平面 vs 数据平面）
   - 渐进式 API 设计（从简单到复杂）
   - 优雅的向后兼容性处理

2. **完整的类型系统**
   - 全面的类型注解（Type Hints）
   - Pydantic 数据验证
   - 支持 IDE 智能提示

3. **专业的工程实践**
   - 90%+ 测试覆盖率
   - 现代化工具链（ruff, mypy, pytest）
   - 详细的文档和示例

4. **真实的生产环境验证**
   - AWS 官方维护
   - 大规模用户使用
   - 持续演进和优化

### 核心要点速览

| 维度 | 关键实践 | AgentCore Memory 示例 |
|------|---------|---------------------|
| **架构设计** | 分层架构、职责分离 | `MemorySessionManager` (推荐) vs `MemoryClient` (遗留) |
| **API 设计** | 渐进式、扁平化 | 3 行代码开始使用，支持高级配置 |
| **类型安全** | 完整类型注解 | 所有公共 API 都有类型注解和 Docstring |
| **错误处理** | 细粒度异常 | 自定义异常层次 + 重试机制 |
| **版本管理** | 优雅废弃 | `warnings.warn()` + 迁移指南 |
| **文档** | 代码即文档 | Google 风格 Docstring + README |
| **测试** | 高覆盖率 | 单元测试 + 集成测试 >90% |
| **性能** | 连接复用、批量操作 | boto3 连接池 + 分页优化 |

---

## 开发 Python SDK 的十一大工程实践要点

### 1. **项目结构：清晰的组织架构**

**核心原则：** 使用标准的 `src/` 布局，清晰的公私分离

```
my-sdk/
├── src/my_sdk/              # 源代码
│   ├── __init__.py         # 公共API导出
│   ├── client.py           # 核心客户端
│   ├── models/             # 数据模型
│   └── _internal/          # 内部实现（私有）
├── tests/                   # 测试代码
├── docs/                    # 文档
└── pyproject.toml          # 项目配置
```

**AgentCore Memory 实践：**
- 清晰的模块划分（`session.py`, `controlplane.py`, `models/`）
- 公共 API 在 `__init__.py` 中统一导出
- 内部工具函数使用 `_` 前缀

### 2. **API 设计：渐进式复杂度**

**核心原则：** 简单场景简单，复杂场景灵活

```python
# Level 1: 最简单 - 3行代码开始
manager = MemorySessionManager(memory_id="mem-123")
session = manager.create_memory_session(actor_id="user-1")
session.add_turns([ConversationalMessage("Hello", MessageRole.USER)])

# Level 2: 中等 - 添加配置
memories = session.search_long_term_memories(
    query="preferences",
    top_k=5
)

# Level 3: 高级 - 完全控制
memories, response, event = session.process_turn_with_llm(
    user_input="...",
    llm_callback=my_llm,
    retrieval_config={...},
    metadata={...}
)
```

**设计要点：**
- 合理的默认值
- 可选的高级参数
- 链式调用支持

### 3. **类型安全：完整的类型系统**

**核心原则：** 使用类型注解 + 运行时验证

```python
from typing import List, Optional, Dict, Any
from pydantic import BaseModel, Field

# 1. 函数签名类型注解
def add_turns(
    self,
    actor_id: str,
    session_id: str,
    messages: List[Union[ConversationalMessage, BlobMessage]],
    metadata: Optional[Dict[str, MetadataValue]] = None,
) -> Event:
    pass

# 2. Pydantic 数据验证
class RetrievalConfig(BaseModel):
    top_k: int = Field(default=10, gt=1, le=100)
    relevance_score: float = Field(default=0.0, ge=0.0, le=1.0)
```

**AgentCore Memory 实践：**
- 所有公共方法都有完整类型注解
- 使用 Pydantic 进行数据验证
- 使用 Enum 定义常量

### 4. **代理模式：减少参数重复**

**核心原则：** 使用会话代理自动注入上下文参数

```python
# ❌ 不好：重复传递参数
manager.add_turns(actor_id="user-1", session_id="sess-1", messages=[...])
manager.search_memories(actor_id="user-1", session_id="sess-1", query="...")
manager.list_events(actor_id="user-1", session_id="sess-1")

# ✅ 好：使用会话代理
session = manager.create_memory_session(actor_id="user-1", session_id="sess-1")
session.add_turns(messages=[...])
session.search_memories(query="...")
session.list_events()
```

**实现模式：**
```python
class Session:
    def __init__(self, actor_id, session_id, manager):
        self._actor_id = actor_id
        self._session_id = session_id
        self._manager = manager
    
    def add_turns(self, messages):
        # 自动注入参数
        return self._manager.add_turns(self._actor_id, self._session_id, messages)
```

### 5. **错误处理：细粒度异常体系**

**核心原则：** 自定义异常层次 + 有用的错误信息

```python
# 异常层次
class SDKError(Exception): pass
class ValidationError(SDKError): pass
class APIError(SDKError): pass
class ResourceNotFoundError(APIError): pass
class ThrottlingError(APIError): pass

# 细粒度捕获
try:
    session.add_turns(messages)
except ValidationError as e:
    print(f"Invalid input: {e.field}")
except ResourceNotFoundError as e:
    print(f"Not found: {e.request_id}")
except ThrottlingError as e:
    print(f"Rate limited, retry after {e.retry_after}s")
```

**AgentCore Memory 实践：**
- 将 boto3 的 `ClientError` 转换为语义化异常
- 异常包含有用的上下文信息（request_id, error_code）
- 自动重试机制

### 6. **API 演进：优雅的废弃机制**

**核心原则：** 渐进式废弃，提供迁移路径

```python
import warnings

def old_method(self, ...):
    """DEPRECATED: Use new_method() instead."""
    warnings.warn(
        "old_method() is deprecated and will be removed in v2.0.0. "
        "Use new_method() instead.",
        DeprecationWarning,
        stacklevel=2
    )
    # 内部调用新方法
    return self.new_method(...)
```

**演进阶段：**
1. **v0.8**: 引入新 API（旧 API 无警告）
2. **v0.9**: 标记旧 API 为废弃（显示警告）
3. **v1.0**: 移除旧 API

### 7. **文档：代码即文档**

**核心原则：** Google 风格 Docstring + 丰富示例

```python
def process_turn_with_llm(
    self,
    user_input: str,
    llm_callback: Callable[[str, List[Dict]], str],
    retrieval_config: Optional[Dict[str, RetrievalConfig]] = None,
) -> Tuple[List[Dict], str, Dict]:
    """Complete conversation turn with LLM callback integration.
    
    This method combines memory retrieval, LLM invocation, and response
    storage in a single call using a callback pattern.
    
    Args:
        user_input: The user's message
        llm_callback: Function that takes (user_input, memories) and 
            returns agent_response
        retrieval_config: Optional dictionary mapping namespaces to 
            RetrievalConfig objects
    
    Returns:
        Tuple of (retrieved_memories, agent_response, created_event)
    
    Example:
        >>> def my_llm(user_input: str, memories: List[Dict]) -> str:
        ...     return call_bedrock(user_input, memories)
        >>> 
        >>> memories, response, event = session.process_turn_with_llm(
        ...     user_input="What did we discuss?",
        ...     llm_callback=my_llm
        ... )
    
    See Also:
        - :meth:`add_turns`: For manual conversation storage
    """
    pass
```

### 8. **测试：高覆盖率 + 分层测试**

**核心原则：** 单元测试 + 集成测试 >90% 覆盖率

```python
# 单元测试：Mock 外部依赖
@pytest.fixture
def mock_boto_client():
    with patch('boto3.client') as mock:
        yield mock.return_value

def test_add_turns(manager, mock_boto_client):
    mock_boto_client.create_event.return_value = {...}
    event = manager.add_turns(...)
    assert event.eventId == "event-123"

# 集成测试：真实 API 调用
@pytest.mark.integration
def test_full_flow(real_memory):
    session = manager.create_memory_session(...)
    event = session.add_turns(...)
    assert event.eventId is not None
```

**AgentCore Memory 实践：**
- ✅ 90%+ 测试覆盖率
- ✅ 单元测试使用 Mock
- ✅ 集成测试使用真实 AWS 资源

### 9. **性能：连接复用 + 批量操作**

**核心原则：** 优化常见操作路径

```python
# 1. 连接池配置
from botocore.config import Config

config = Config(
    max_pool_connections=50,
    connect_timeout=5,
    read_timeout=60,
    retries={'max_attempts': 3, 'mode': 'adaptive'}
)

# 2. 批量操作
def batch_delete(self, record_ids: List[str], batch_size: int = 100):
    for i in range(0, len(record_ids), batch_size):
        batch = record_ids[i:i + batch_size]
        self._client.batch_delete_records(records=batch)

# 3. 智能分页
def list_all(self, max_results: int = 1000):
    all_items = []
    next_token = None
    
    while len(all_items) < max_results:
        page_size = min(100, max_results - len(all_items))
        response = self._client.list_items(
            maxResults=page_size,
            nextToken=next_token
        )
        all_items.extend(response['items'])
        next_token = response.get('nextToken')
        if not next_token:
            break
    
    return all_items
```

### 10. **工具链：现代化开发工具**

**核心原则：** 使用现代化工具提升代码质量

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.10"
disallow_untyped_defs = true
warn_return_any = true

[tool.ruff]
line-length = 120
select = ["E", "F", "I", "B", "D"]

[tool.pytest.ini_options]
addopts = [
    "--cov=src",
    "--cov-report=html",
    "--cov-fail-under=90"
]
```

**AgentCore Memory 工具链：**
- ✅ **mypy**: 静态类型检查
- ✅ **ruff**: 快速的 linter 和 formatter
- ✅ **pytest**: 测试框架 + 覆盖率
- ✅ **pre-commit**: Git 提交前检查

### 11. **Python 版本要求：如何确定支持的版本**

**核心原则：** 平衡新特性使用与用户兼容性

#### 11.1 决策因素

确定 Python 版本要求需要考虑以下因素：

**1. 语言特性需求**

| Python 版本 | 关键特性 | 是否必需 |
|------------|---------|---------|
| 3.10+ | `match-case` 语句、更好的错误消息、类型联合 `X \| Y` | 可选 |
| 3.9+ | 字典合并操作符 `\|`、类型提示改进 | 推荐 |
| 3.8+ | 海象操作符 `:=`、`TypedDict`、`Protocol` | 常用 |
| 3.7+ | `dataclasses`、`asyncio` 改进 | 基础 |

**2. 依赖库要求**

```python
# 检查关键依赖的 Python 版本要求
boto3 >= 1.40.0        # 需要 Python 3.8+
pydantic >= 2.0.0      # 需要 Python 3.7+
starlette >= 0.46.0    # 需要 Python 3.8+
```

**3. 用户基础分析**

```python
# Python 版本使用统计（2024年）
Python 3.12: 15%  # 最新
Python 3.11: 25%  # 新版本
Python 3.10: 30%  # 主流
Python 3.9:  20%  # 仍在使用
Python 3.8:  8%   # 逐渐淘汰
Python 3.7:  2%   # 已 EOL (2023-06-27)
```

**4. 维护成本**

- 支持更多版本 = 更多测试矩阵 = 更高维护成本
- 建议：支持 **当前版本 + 前两个版本**

#### 11.2 AgentCore Memory 的选择

```python
# pyproject.toml
[project]
name = "bedrock-agentcore"
requires-python = ">=3.10"  # 最低要求 Python 3.10

classifiers = [
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
]
```

**选择 Python 3.10+ 的原因：**

1. **类型系统增强**
   
   ```python
   # Python 3.10+ 的联合类型语法
   def process(data: str | int) -> list[str]:  # 更简洁
       pass
   
   # Python 3.9 需要
   from typing import Union, List
   def process(data: Union[str, int]) -> List[str]:
       pass
   ```
   
2. **更好的错误消息**
   
   ```python
   # Python 3.10+ 提供更精确的错误位置
   # 指向具体的表达式而不是整行
   ```
   
3. **match-case 语句**（可选使用）
   
   ```python
   # Python 3.10+
   match status:
       case "ACTIVE":
           return process_active()
       case "PENDING":
           return process_pending()
       case _:
           return handle_unknown()
   ```
   
4. **用户基础充足**
   
   - Python 3.10 发布于 2021-10
   - 截至 2024 年，已有 3 年时间
   - 覆盖 85%+ 的活跃用户

#### 11.3 版本要求实现

**1. 在 pyproject.toml 中声明**

```toml
[project]
name = "my-sdk"
version = "1.0.0"
requires-python = ">=3.10"  # 最低版本要求

classifiers = [
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

dependencies = [
    "boto3>=1.40.0",
    "pydantic>=2.0.0,<3.0.0",  # 指定版本范围
]
```

**2. 运行时版本检查**

```python
# src/my_sdk/__init__.py
"""My SDK - Python 3.10+ required."""

import sys

# 版本检查
if sys.version_info < (3, 10):
    raise RuntimeError(
        f"Python 3.10 or higher is required. "
        f"You are using Python {sys.version_info.major}.{sys.version_info.minor}."
    )

__version__ = "1.0.0"
```

**3. 使用版本检查装饰器**

```python
# src/my_sdk/_internal/version_check.py
import sys
import warnings
from functools import wraps

def require_python_version(major: int, minor: int):
    """装饰器：检查 Python 版本"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if sys.version_info < (major, minor):
                raise RuntimeError(
                    f"{func.__name__} requires Python {major}.{minor}+. "
                    f"Current: {sys.version_info.major}.{sys.version_info.minor}"
                )
            return func(*args, **kwargs)
        return wrapper
    return decorator

# 使用
@require_python_version(3, 11)
def use_new_feature():
    """此功能需要 Python 3.11+"""
    pass
```

**4. 条件导入处理**

```python
# 处理不同 Python 版本的差异
import sys

if sys.version_info >= (3, 10):
    # Python 3.10+ 使用新语法
    from typing import TypeAlias
    JSONType: TypeAlias = dict[str, any]
else:
    # Python 3.9 使用旧语法
    from typing import Dict, Any
    JSONType = Dict[str, Any]
```

#### 11.4 版本支持策略

**推荐策略：N-2 原则**

支持当前版本和前两个版本：

```
当前年份: 2024
Python 3.13 (2024-10) ✅ 支持
Python 3.12 (2023-10) ✅ 支持
Python 3.11 (2022-10) ✅ 支持
Python 3.10 (2021-10) ✅ 最低版本
Python 3.9  (2020-10) ❌ 不支持
```

**版本生命周期表**

| 版本 | 发布日期 | EOL 日期 | 状态 |
|------|---------|---------|------|
| 3.13 | 2024-10 | 2029-10 | 最新 |
| 3.12 | 2023-10 | 2028-10 | 稳定 |
| 3.11 | 2022-10 | 2027-10 | 稳定 |
| 3.10 | 2021-10 | 2026-10 | 稳定 |
| 3.9  | 2020-10 | 2025-10 | 即将 EOL |
| 3.8  | 2019-10 | 2024-10 | 已 EOL |

#### 11.5 测试矩阵配置

**GitHub Actions 示例**

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12", "3.13"]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -e ".[dev]"
      
      - name: Run tests
        run: pytest
      
      - name: Type check
        run: mypy src/
```

**tox 配置**

```ini
# tox.ini
[tox]
envlist = py310,py311,py312,py313

[testenv]
deps =
    pytest
    pytest-cov
commands =
    pytest tests/
```

#### 11.6 版本升级策略

**何时提升最低版本要求？**

考虑以下情况：

1. **新特性需求强烈**
   - 需要 Python 3.11 的 `ExceptionGroup`
   - 需要 Python 3.12 的性能改进

2. **旧版本 EOL**
   - Python 3.9 将在 2025-10 EOL
   - 提前 6 个月通知用户

3. **依赖库要求**
   - 关键依赖升级最低版本要求

**升级流程：**

```python
# 阶段 1: 提前通知（v1.5.0）
import warnings

if sys.version_info < (3, 11):
    warnings.warn(
        "Python 3.10 support will be dropped in v2.0.0. "
        "Please upgrade to Python 3.11+.",
        FutureWarning,
        stacklevel=2
    )

# 阶段 2: 更新文档（v1.6.0）
# README.md, CHANGELOG.md 中明确说明

# 阶段 3: 正式升级（v2.0.0）
if sys.version_info < (3, 11):
    raise RuntimeError("Python 3.11+ is required")
```

#### 11.7 最佳实践总结

**✅ 推荐做法：**

1. **选择合理的最低版本**
   - 新项目：Python 3.10+
   - 保守项目：Python 3.9+
   - 企业项目：根据内部标准

2. **明确声明版本要求**
   ```toml
   requires-python = ">=3.10"
   ```

3. **运行时检查**
   ```python
   if sys.version_info < (3, 10):
       raise RuntimeError("Python 3.10+ required")
   ```

4. **完整的测试矩阵**
   - 测试所有声明支持的版本
   - 使用 CI/CD 自动化测试

5. **提前通知升级**
   - 至少提前一个大版本通知
   - 在文档中明确说明

**❌ 避免做法：**

1. ❌ 支持过多版本（增加维护成本）
2. ❌ 没有运行时检查（用户遇到奇怪错误）
3. ❌ 突然升级最低版本（破坏用户环境）
4. ❌ 使用未声明版本的特性（隐式依赖）

#### 11.8 AgentCore Memory 的完整配置

```toml
# pyproject.toml
[project]
name = "bedrock-agentcore"
version = "1.1.1"
requires-python = ">=3.10"

classifiers = [
    "Development Status :: 3 - Alpha",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
]

dependencies = [
    "boto3>=1.40.52",
    "botocore>=1.40.52",
    "pydantic>=2.0.0,<2.41.3",
]

[tool.mypy]
python_version = "3.10"  # 与 requires-python 一致
```

```python
# src/bedrock_agentcore/__init__.py
"""Bedrock AgentCore SDK - Python 3.10+ required."""

import sys

if sys.version_info < (3, 10):
    raise RuntimeError(
        "Python 3.10 or higher is required. "
        f"You are using Python {sys.version_info.major}.{sys.version_info.minor}."
    )

__version__ = "1.1.1"
```

---

## 关于本文档

### 文档结构

本文档分为 **11 个核心章节**，每章深入讲解一个主题，并提供 AgentCore Memory SDK 的真实代码示例：

1. **项目结构设计** - 如何组织 SDK 代码
2. **API 设计原则** - 如何设计易用的 API
3. **类型系统与验证** - 如何实现类型安全
4. **错误处理策略** - 如何优雅地处理错误
5. **API 演进与版本管理** - 如何平滑升级 API
6. **文档与可发现性** - 如何编写好文档
7. **测试策略** - 如何保证代码质量
8. **性能优化** - 如何提升 SDK 性能
9. **完整示例** - 可直接使用的代码模板
10. **总结与检查清单** - 开发检查清单
11. **Python 版本要求** - 如何确定支持的版本

### 适用对象

- **SDK 开发者**：构建客户端库的工程师
- **架构师**：设计 API 和系统架构的技术负责人
- **技术主管**：制定团队开发规范的管理者
- **Python 开发者**：希望提升代码质量的工程师

### 如何使用本文档

1. **快速入门**：阅读本执行摘要和十大要点
2. **深入学习**：按章节顺序阅读详细内容
3. **实践应用**：参考第 9 章的完整示例
4. **质量检查**：使用第 10 章的检查清单

---

## 目录

1. [项目结构设计](#1-项目结构设计)
2. [API 设计原则](#2-api-设计原则)
3. [类型系统与验证](#3-类型系统与验证)
4. [错误处理策略](#4-错误处理策略)
5. [API 演进与版本管理](#5-api-演进与版本管理)
6. [文档与可发现性](#6-文档与可发现性)
7. [测试策略](#7-测试策略)
8. [性能优化](#8-性能优化)
9. [完整示例](#9-完整示例)

---

## 1. 项目结构设计

### 1.1 标准 SDK 项目结构

```
my-sdk/
├── src/
│   └── my_sdk/
│       ├── __init__.py              # 公共API导出
│       ├── client.py                # 遗留客户端（向后兼容）
│       ├── session.py               # 推荐的新API
│       ├── constants.py             # 常量和枚举
│       ├── exceptions.py            # 自定义异常
│       ├── models/                  # 数据模型
│       │   ├── __init__.py
│       │   ├── base.py             # 基础模型类
│       │   └── requests.py         # 请求/响应模型
│       ├── _internal/               # 内部实现（私有）
│       │   ├── __init__.py
│       │   └── utils.py
│       └── integrations/            # 第三方集成
│           └── framework_x/
├── tests/
│   ├── unit/                        # 单元测试
│   └── integration/                 # 集成测试
├── docs/                            # 文档
├── examples/                        # 示例代码
├── pyproject.toml                   # 项目配置
├── README.md
└── CHANGELOG.md
```

### 1.2 AgentCore Memory 的实际结构

```python
# src/bedrock_agentcore/memory/
memory/
├── __init__.py                      # ✅ 公共API导出
├── client.py                        # ⚠️  遗留API（已废弃但保留）
├── session.py                       # ✅ 推荐的新API
├── controlplane.py                  # ✅ 控制平面客户端
├── constants.py                     # ✅ 常量定义
├── README.md                        # ✅ 模块文档
├── models/                          # ✅ 数据模型
│   ├── __init__.py
│   ├── DictWrapper.py              # 基础包装器
│   └── filters.py                  # 过滤器模型
└── integrations/                    # ✅ 第三方集成
    └── strands/
```

**设计原则：**

1. **清晰的公私分离**
   - 公共API在顶层
   - 内部实现在 `_internal/` 或以 `_` 开头

2. **模块化组织**
   - 按功能分模块（models, integrations）
   - 每个模块有独立的 `__init__.py`

3. **向后兼容性**
   - 保留旧API但标记为废弃
   - 提供迁移路径


---

## 2. API 设计原则

### 2.1 渐进式 API 设计（Progressive API）

AgentCore Memory 展示了优秀的渐进式设计：

#### Level 1: 简单场景 - 最少代码

```python
from bedrock_agentcore.memory import MemorySessionManager
from bedrock_agentcore.memory.constants import ConversationalMessage, MessageRole

# 3行代码开始使用
manager = MemorySessionManager(memory_id="mem-123")
session = manager.create_memory_session(actor_id="user-1", session_id="sess-1")
session.add_turns([
    ConversationalMessage("Hello", MessageRole.USER),
    ConversationalMessage("Hi!", MessageRole.ASSISTANT)
])
```

#### Level 2: 中等场景 - 添加配置

```python
from bedrock_agentcore.memory.constants import RetrievalConfig

# 添加记忆检索配置
memories = session.search_long_term_memories(
    query="user preferences",
    namespace_prefix="/prefs/user-1",
    top_k=5
)
```

#### Level 3: 高级场景 - 完全控制

```python
# 完整的LLM集成流程
def my_llm(user_input: str, memories: List[Dict]) -> str:
    # 自定义LLM逻辑
    return process_with_llm(user_input, memories)

memories, response, event = session.process_turn_with_llm(
    user_input="What did we discuss?",
    llm_callback=my_llm,
    retrieval_config={
        "facts/{sessionId}": RetrievalConfig(top_k=5, relevance_score=0.3),
        "prefs/{actorId}": RetrievalConfig(top_k=3, relevance_score=0.5)
    },
    metadata={"location": {"stringValue": "NYC"}},
    event_timestamp=datetime.now(timezone.utc)
)
```

**设计原则：**

```python
# ✅ 好的设计：默认值 + 可选参数
def search_memories(
    query: str,                           # 必需
    namespace_prefix: str,                # 必需
    top_k: int = 3,                      # 有合理默认值
    strategy_id: Optional[str] = None,   # 高级选项
    max_results: int = 20                # 有合理默认值
):
    pass

# ❌ 不好的设计：太多必需参数
def search_memories(
    query: str,
    namespace_prefix: str,
    top_k: int,              # 应该有默认值
    strategy_id: str,        # 应该是可选的
    max_results: int,        # 应该有默认值
    timeout: int,            # 应该有默认值
    retry_count: int         # 应该有默认值
):
    pass
```

### 2.2 API 扁平化（Flattening）

**问题：** 深层嵌套的导入路径

```python
# ❌ 不好：用户需要知道内部结构
from my_sdk.core.session.manager import SessionManager
from my_sdk.core.session.context import SessionContext
from my_sdk.models.requests.memory import MemoryRequest
from my_sdk.constants.enums.status import StatusEnum
```

**解决方案：** 在 `__init__.py` 中扁平化

```python
# src/my_sdk/__init__.py
"""My SDK - A Python SDK for XYZ service."""

# 从深层导入
from .core.session.manager import SessionManager
from .core.session.context import SessionContext
from .models.requests.memory import MemoryRequest
from .constants.enums.status import StatusEnum

# 扁平化导出
__all__ = [
    "SessionManager",
    "SessionContext", 
    "MemoryRequest",
    "StatusEnum",
]
```

**用户体验：**

```python
# ✅ 好：简洁的导入
from my_sdk import SessionManager, SessionContext, MemoryRequest, StatusEnum
```

**AgentCore Memory 的实现：**

```python
# src/bedrock_agentcore/memory/__init__.py
"""Bedrock AgentCore Memory module."""

from .client import MemoryClient
from .controlplane import MemoryControlPlaneClient
from .session import Actor, MemorySession, MemorySessionManager

__all__ = [
    "Actor",
    "MemoryClient",
    "MemorySession",
    "MemorySessionManager",
    "MemoryControlPlaneClient"
]
```


### 2.3 代理模式减少参数重复

**问题：** 重复传递相同参数

```python
# ❌ 不好：每次调用都要传递相同参数
manager = MemorySessionManager(memory_id="mem-123")

manager.add_turns(
    actor_id="user-1",      # 重复
    session_id="sess-1",    # 重复
    messages=[...]
)

manager.search_memories(
    actor_id="user-1",      # 重复
    session_id="sess-1",    # 重复
    query="..."
)

manager.list_events(
    actor_id="user-1",      # 重复
    session_id="sess-1",    # 重复
)
```

**解决方案：** 使用会话代理

```python
# ✅ 好：创建会话代理，自动注入参数
class MemorySession:
    """会话级别的代理"""
    
    def __init__(self, memory_id: str, actor_id: str, session_id: str, manager):
        self._memory_id = memory_id
        self._actor_id = actor_id
        self._session_id = session_id
        self._manager = manager
    
    def add_turns(self, messages, **kwargs):
        """自动注入 actor_id 和 session_id"""
        return self._manager.add_turns(
            self._actor_id,
            self._session_id,
            messages,
            **kwargs
        )
    
    def search_memories(self, query, **kwargs):
        """自动注入 actor_id 和 session_id"""
        return self._manager.search_long_term_memories(
            query,
            **kwargs
        )

# 用户代码
session = manager.create_memory_session(
    actor_id="user-1",
    session_id="sess-1"
)

# 不需要重复传递参数
session.add_turns(messages=[...])
session.search_memories(query="...")
session.list_events()
```

**实现模式：**

```python
class Manager:
    """管理器：处理多个会话"""
    
    def operation(self, actor_id: str, session_id: str, data):
        # 实际实现
        pass
    
    def create_session(self, actor_id: str, session_id: str):
        """工厂方法创建会话代理"""
        return Session(actor_id, session_id, manager=self)

class Session:
    """会话代理：绑定特定 actor 和 session"""
    
    def __init__(self, actor_id: str, session_id: str, manager):
        self._actor_id = actor_id
        self._session_id = session_id
        self._manager = manager
    
    def operation(self, data):
        """委托给管理器，自动注入参数"""
        return self._manager.operation(
            self._actor_id,
            self._session_id,
            data
        )
```

### 2.4 方法命名规范

**一致的命名模式：**

```python
# ✅ 好的命名模式
class MemorySessionManager:
    # 创建操作：create_*
    def create_memory_session(self, ...): pass
    
    # 添加操作：add_*
    def add_turns(self, ...): pass
    
    # 列表操作：list_*
    def list_events(self, ...): pass
    def list_actors(self, ...): pass
    def list_branches(self, ...): pass
    
    # 搜索操作：search_*
    def search_long_term_memories(self, ...): pass
    
    # 获取单个：get_*
    def get_event(self, ...): pass
    def get_memory_record(self, ...): pass
    
    # 删除操作：delete_*
    def delete_event(self, ...): pass
    def delete_memory_record(self, ...): pass
```

**动词选择指南：**

| 操作类型 | 推荐动词 | 示例 |
|---------|---------|------|
| 创建资源 | `create_*` | `create_session()` |
| 添加项目 | `add_*` | `add_turns()` |
| 获取单个 | `get_*` | `get_event()` |
| 获取列表 | `list_*` | `list_events()` |
| 搜索查询 | `search_*`, `find_*` | `search_memories()` |
| 更新资源 | `update_*` | `update_strategy()` |
| 删除资源 | `delete_*`, `remove_*` | `delete_event()` |
| 检查状态 | `is_*`, `has_*` | `is_active()` |


---

## 3. 类型系统与验证

### 3.1 完整的类型注解

**基础类型注解：**

```python
from typing import List, Dict, Optional, Union, Tuple, Any, Callable
from datetime import datetime

class MemorySessionManager:
    """完整的类型注解示例"""
    
    def __init__(
        self,
        memory_id: str,
        region_name: Optional[str] = None,
        boto3_session: Optional[boto3.Session] = None
    ) -> None:
        self._memory_id = memory_id
        self.region_name = region_name or "us-west-2"
    
    def add_turns(
        self,
        actor_id: str,
        session_id: str,
        messages: List[Union[ConversationalMessage, BlobMessage]],
        branch: Optional[Dict[str, str]] = None,
        metadata: Optional[Dict[str, MetadataValue]] = None,
        event_timestamp: Optional[datetime] = None,
    ) -> Event:
        """
        添加对话轮次。
        
        Args:
            actor_id: Actor 标识符
            session_id: Session 标识符
            messages: 消息列表
            branch: 可选的分支信息
            metadata: 可选的元数据
            event_timestamp: 可选的时间戳
            
        Returns:
            创建的事件对象
            
        Raises:
            ValueError: 如果消息列表为空
            ClientError: 如果 AWS API 调用失败
        """
        pass
    
    def process_turn_with_llm(
        self,
        actor_id: str,
        session_id: str,
        user_input: str,
        llm_callback: Callable[[str, List[Dict[str, Any]]], str],
        retrieval_config: Optional[Dict[str, RetrievalConfig]] = None,
    ) -> Tuple[List[Dict[str, Any]], str, Dict[str, Any]]:
        """返回元组类型"""
        pass
```

### 3.2 使用 Pydantic 进行数据验证

**定义验证模型：**

```python
from pydantic import BaseModel, Field, validator
from typing import Optional

class RetrievalConfig(BaseModel):
    """检索配置模型"""
    
    top_k: int = Field(
        default=10,
        gt=1,           # 大于 1
        le=100,         # 小于等于 100
        description="返回的记录数量"
    )
    
    relevance_score: float = Field(
        default=0.0,
        ge=0.0,         # 大于等于 0.0
        le=1.0,         # 小于等于 1.0
        description="相关性分数阈值"
    )
    
    strategy_id: Optional[str] = Field(
        None,
        description="可选的策略ID"
    )
    
    retrieval_query: Optional[str] = Field(
        None,
        max_length=1000,
        description="自定义检索查询"
    )
    
    @validator('top_k')
    def validate_top_k(cls, v):
        """自定义验证器"""
        if v < 1 or v > 100:
            raise ValueError('top_k must be between 1 and 100')
        return v
    
    class Config:
        """Pydantic 配置"""
        validate_assignment = True  # 赋值时也验证
        extra = 'forbid'            # 禁止额外字段

# 使用
config = RetrievalConfig(top_k=5, relevance_score=0.3)
# config = RetrievalConfig(top_k=200)  # ❌ ValidationError
```

### 3.3 使用 Dataclass 定义数据结构

```python
from dataclasses import dataclass, field
from typing import Optional
from enum import Enum

class MessageRole(Enum):
    """消息角色枚举"""
    USER = "USER"
    ASSISTANT = "ASSISTANT"
    TOOL = "TOOL"
    OTHER = "OTHER"

@dataclass
class ConversationalMessage:
    """对话消息数据类"""
    
    text: str
    role: MessageRole
    metadata: Optional[Dict[str, Any]] = field(default=None)
    
    def __post_init__(self):
        """初始化后验证"""
        if not isinstance(self.text, str):
            raise ValueError("text must be a string")
        if not isinstance(self.role, MessageRole):
            raise ValueError("role must be a MessageRole")
        if not self.text.strip():
            raise ValueError("text cannot be empty")

# 使用
msg = ConversationalMessage(
    text="Hello",
    role=MessageRole.USER
)

# msg = ConversationalMessage(text="", role=MessageRole.USER)  # ❌ ValueError
```

### 3.4 TypedDict 用于字典结构

```python
from typing import TypedDict, Optional, Literal

class EventMetadataFilter(TypedDict):
    """事件元数据过滤器"""
    
    left: 'LeftExpression'
    operator: Literal['EQUALS_TO', 'EXISTS', 'NOT_EXISTS']
    right: Optional['RightExpression']

class LeftExpression(TypedDict):
    """左表达式"""
    metadataKey: str

class RightExpression(TypedDict):
    """右表达式"""
    metadataValue: 'MetadataValue'

class StringValue(TypedDict):
    """字符串值"""
    stringValue: str

# 使用
filter_expr: EventMetadataFilter = {
    'left': {'metadataKey': 'location'},
    'operator': 'EQUALS_TO',
    'right': {'metadataValue': {'stringValue': 'NYC'}}
}
```

### 3.5 Protocol 用于结构化子类型

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class LLMProvider(Protocol):
    """LLM 提供商协议"""
    
    def generate(
        self,
        prompt: str,
        context: List[Dict[str, Any]]
    ) -> str:
        """生成响应"""
        ...
    
    def stream_generate(
        self,
        prompt: str,
        context: List[Dict[str, Any]]
    ) -> Iterator[str]:
        """流式生成"""
        ...

# 任何实现了这些方法的类都符合协议
class BedrockLLM:
    def generate(self, prompt: str, context: List[Dict]) -> str:
        return "response"
    
    def stream_generate(self, prompt: str, context: List[Dict]) -> Iterator[str]:
        yield "chunk"

# 类型检查通过
def process_with_llm(llm: LLMProvider, prompt: str):
    return llm.generate(prompt, [])

process_with_llm(BedrockLLM(), "Hello")  # ✅ 类型检查通过
```


---

## 4. 错误处理策略

### 4.1 自定义异常层次结构

```python
# exceptions.py
"""SDK 异常定义"""

class SDKError(Exception):
    """SDK 基础异常"""
    pass

class ConfigurationError(SDKError):
    """配置错误"""
    pass

class ValidationError(SDKError):
    """验证错误"""
    
    def __init__(self, message: str, field: str = None):
        super().__init__(message)
        self.field = field

class APIError(SDKError):
    """API 调用错误"""
    
    def __init__(
        self,
        message: str,
        status_code: int = None,
        error_code: str = None,
        request_id: str = None
    ):
        super().__init__(message)
        self.status_code = status_code
        self.error_code = error_code
        self.request_id = request_id

class ResourceNotFoundError(APIError):
    """资源未找到"""
    pass

class ThrottlingError(APIError):
    """请求限流"""
    
    def __init__(self, message: str, retry_after: int = None):
        super().__init__(message)
        self.retry_after = retry_after

class TimeoutError(SDKError):
    """超时错误"""
    
    def __init__(self, message: str, elapsed: float):
        super().__init__(message)
        self.elapsed = elapsed
```

### 4.2 错误处理模式

**模式1：细粒度异常捕获**

```python
from botocore.exceptions import ClientError
import logging

logger = logging.getLogger(__name__)

def add_turns(self, actor_id: str, session_id: str, messages: List):
    """带完整错误处理的方法"""
    
    # 输入验证
    if not messages:
        raise ValidationError("At least one message is required", field="messages")
    
    try:
        # API 调用
        response = self._client.create_event(
            memoryId=self._memory_id,
            actorId=actor_id,
            sessionId=session_id,
            payload=self._format_messages(messages)
        )
        
        logger.info("Successfully created event: %s", response['eventId'])
        return Event(response['event'])
        
    except ClientError as e:
        error_code = e.response['Error']['Code']
        error_msg = e.response['Error']['Message']
        request_id = e.response['ResponseMetadata'].get('RequestId')
        
        # 根据错误码分类处理
        if error_code == 'ResourceNotFoundException':
            raise ResourceNotFoundError(
                f"Memory {self._memory_id} not found",
                error_code=error_code,
                request_id=request_id
            ) from e
        
        elif error_code == 'ValidationException':
            raise ValidationError(
                f"Invalid input: {error_msg}",
            ) from e
        
        elif error_code == 'ThrottlingException':
            retry_after = int(e.response['Error'].get('RetryAfter', 5))
            raise ThrottlingError(
                "Request rate exceeded",
                retry_after=retry_after
            ) from e
        
        else:
            # 通用 API 错误
            raise APIError(
                f"API call failed: {error_msg}",
                error_code=error_code,
                request_id=request_id
            ) from e
    
    except Exception as e:
        logger.exception("Unexpected error in add_turns")
        raise SDKError(f"Unexpected error: {str(e)}") from e
```

**模式2：重试装饰器**

```python
import time
from functools import wraps
from typing import Callable, Type

def retry_on_throttle(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0
):
    """重试装饰器"""
    
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                
                except ThrottlingError as e:
                    last_exception = e
                    
                    if attempt == max_retries:
                        logger.error(
                            "Max retries (%d) exceeded for %s",
                            max_retries,
                            func.__name__
                        )
                        raise
                    
                    # 计算延迟时间
                    delay = min(
                        base_delay * (exponential_base ** attempt),
                        max_delay
                    )
                    
                    # 使用服务器建议的延迟
                    if e.retry_after:
                        delay = min(e.retry_after, max_delay)
                    
                    logger.warning(
                        "Throttled, retrying in %.2fs (attempt %d/%d)",
                        delay,
                        attempt + 1,
                        max_retries
                    )
                    
                    time.sleep(delay)
            
            raise last_exception
        
        return wrapper
    return decorator

# 使用
class MemoryClient:
    @retry_on_throttle(max_retries=3)
    def add_turns(self, ...):
        # 实现
        pass
```

### 4.3 上下文管理器用于资源清理

```python
from contextlib import contextmanager
from typing import Generator

@contextmanager
def memory_session_context(
    manager: MemorySessionManager,
    actor_id: str,
    session_id: str
) -> Generator[MemorySession, None, None]:
    """会话上下文管理器"""
    
    session = None
    try:
        # 设置
        logger.info("Creating memory session: %s/%s", actor_id, session_id)
        session = manager.create_memory_session(actor_id, session_id)
        
        yield session
        
    except Exception as e:
        # 错误处理
        logger.error("Error in memory session: %s", e)
        raise
    
    finally:
        # 清理
        if session:
            logger.info("Memory session completed: %s/%s", actor_id, session_id)
            # 执行清理操作（如果需要）

# 使用
with memory_session_context(manager, "user-1", "sess-1") as session:
    session.add_turns(messages)
    memories = session.search_memories(query="...")
# 自动清理
```

### 4.4 错误日志记录

```python
import logging
import json
from datetime import datetime

class StructuredLogger:
    """结构化日志记录器"""
    
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
    
    def log_api_call(
        self,
        method: str,
        params: Dict[str, Any],
        duration: float,
        success: bool,
        error: Exception = None
    ):
        """记录 API 调用"""
        
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "method": method,
            "duration_ms": int(duration * 1000),
            "success": success,
        }
        
        # 脱敏参数
        safe_params = self._sanitize_params(params)
        log_entry["params"] = safe_params
        
        if error:
            log_entry["error"] = {
                "type": type(error).__name__,
                "message": str(error),
            }
            
            if isinstance(error, APIError):
                log_entry["error"]["error_code"] = error.error_code
                log_entry["error"]["request_id"] = error.request_id
        
        if success:
            self.logger.info(json.dumps(log_entry))
        else:
            self.logger.error(json.dumps(log_entry))
    
    def _sanitize_params(self, params: Dict) -> Dict:
        """脱敏敏感参数"""
        sensitive_keys = {'password', 'token', 'secret', 'api_key'}
        
        sanitized = {}
        for key, value in params.items():
            if any(s in key.lower() for s in sensitive_keys):
                sanitized[key] = "***REDACTED***"
            else:
                sanitized[key] = value
        
        return sanitized
```


---

## 5. API 演进与版本管理

### 5.1 废弃 API 的优雅处理

**AgentCore Memory 的实际案例：**

```python
import warnings

class MemoryClient:
    """遗留客户端（已废弃）"""
    
    def save_turn(
        self,
        memory_id: str,
        actor_id: str,
        session_id: str,
        user_input: str,
        agent_response: str,
        event_timestamp: Optional[datetime] = None,
    ) -> Dict[str, Any]:
        """
        DEPRECATED: Use save_conversation() for more flexibility.
        
        This method will be removed in v1.0.0.
        """
        warnings.warn(
            "save_turn() is deprecated and will be removed in v1.0.0. "
            "Use save_conversation() for flexible message handling.",
            DeprecationWarning,
            stacklevel=2,  # 显示调用者的位置
        )
        
        # 内部调用新 API
        messages = [
            (user_input, "USER"),
            (agent_response, "ASSISTANT")
        ]
        
        return self.create_event(
            memory_id=memory_id,
            actor_id=actor_id,
            session_id=session_id,
            messages=messages,
            event_timestamp=event_timestamp,
        )
```

### 5.2 API 版本演进策略

**阶段1：引入新 API（v0.8.0）**

```python
# 新 API
class MemorySessionManager:
    """推荐的新 API"""
    
    def add_turns(self, actor_id, session_id, messages):
        """新的灵活 API"""
        pass

# 旧 API 仍然工作
class MemoryClient:
    """遗留 API（无警告）"""
    
    def save_turn(self, memory_id, actor_id, session_id, user_input, agent_response):
        """旧 API"""
        pass
```

**阶段2：标记废弃（v0.9.0）**

```python
class MemoryClient:
    """遗留 API（已废弃）"""
    
    def save_turn(self, ...):
        """DEPRECATED: Use MemorySessionManager.add_turns()"""
        warnings.warn(
            "save_turn() is deprecated and will be removed in v1.0.0. "
            "Use MemorySessionManager.add_turns() instead.",
            DeprecationWarning,
            stacklevel=2
        )
        # 实现...
```

**阶段3：移除旧 API（v1.0.0）**

```python
# MemoryClient 完全移除或只保留新方法
# 用户必须迁移到 MemorySessionManager
```

### 5.3 向后兼容的字段名处理

**AgentCore Memory 的实际案例：**

```python
def _normalize_memory_response(self, memory: Dict[str, Any]) -> Dict[str, Any]:
    """
    规范化响应，同时提供新旧字段名。
    
    API 返回新字段名，但 SDK 用户可能期望旧字段名。
    """
    # 确保两个版本的 ID 都存在
    if "id" in memory and "memoryId" not in memory:
        memory["memoryId"] = memory["id"]
    elif "memoryId" in memory and "id" not in memory:
        memory["id"] = memory["memoryId"]
    
    # 确保两个版本的策略字段都存在
    if "strategies" in memory and "memoryStrategies" not in memory:
        memory["memoryStrategies"] = memory["strategies"]
    elif "memoryStrategies" in memory and "strategies" not in memory:
        memory["strategies"] = memory["memoryStrategies"]
    
    return memory
```

**通用模式：**

```python
class ResponseNormalizer:
    """响应规范化器"""
    
    # 字段映射：新字段名 -> 旧字段名
    FIELD_MAPPINGS = {
        'id': 'resource_id',
        'created_at': 'createdAt',
        'updated_at': 'updatedAt',
    }
    
    @classmethod
    def normalize(cls, response: Dict[str, Any]) -> Dict[str, Any]:
        """规范化响应，提供新旧字段名"""
        normalized = response.copy()
        
        for new_field, old_field in cls.FIELD_MAPPINGS.items():
            # 新字段存在，添加旧字段
            if new_field in normalized and old_field not in normalized:
                normalized[old_field] = normalized[new_field]
            
            # 旧字段存在，添加新字段
            elif old_field in normalized and new_field not in normalized:
                normalized[new_field] = normalized[old_field]
        
        return normalized
```

### 5.4 功能标志（Feature Flags）

```python
class SDKConfig:
    """SDK 配置"""
    
    # 功能标志
    ENABLE_NEW_PAGINATION = True
    ENABLE_ASYNC_API = True
    ENABLE_EXPERIMENTAL_FEATURES = False
    
    @classmethod
    def is_feature_enabled(cls, feature: str) -> bool:
        """检查功能是否启用"""
        return getattr(cls, f"ENABLE_{feature.upper()}", False)

class MemoryClient:
    def list_events(self, ...):
        """列出事件"""
        
        if SDKConfig.is_feature_enabled('new_pagination'):
            # 使用新的分页逻辑
            return self._list_events_v2(...)
        else:
            # 使用旧的分页逻辑
            return self._list_events_v1(...)
```

### 5.5 版本检查和警告

```python
import sys
from packaging import version

class SDKVersionChecker:
    """SDK 版本检查器"""
    
    MIN_PYTHON_VERSION = "3.10"
    CURRENT_SDK_VERSION = "1.1.1"
    
    @classmethod
    def check_python_version(cls):
        """检查 Python 版本"""
        current = f"{sys.version_info.major}.{sys.version_info.minor}"
        
        if version.parse(current) < version.parse(cls.MIN_PYTHON_VERSION):
            raise RuntimeError(
                f"Python {cls.MIN_PYTHON_VERSION}+ is required. "
                f"You are using Python {current}"
            )
    
    @classmethod
    def check_dependency_version(cls, package: str, min_version: str):
        """检查依赖版本"""
        try:
            import importlib.metadata
            installed = importlib.metadata.version(package)
            
            if version.parse(installed) < version.parse(min_version):
                warnings.warn(
                    f"{package} {min_version}+ is recommended. "
                    f"You have {installed} installed.",
                    UserWarning
                )
        except Exception:
            pass

# 在 __init__.py 中执行检查
SDKVersionChecker.check_python_version()
SDKVersionChecker.check_dependency_version('boto3', '1.40.52')
```


---

## 6. 文档与可发现性

### 6.1 完整的 Docstring

**Google 风格 Docstring（推荐）：**

```python
def process_turn_with_llm(
    self,
    actor_id: str,
    session_id: str,
    user_input: str,
    llm_callback: Callable[[str, List[Dict[str, Any]]], str],
    retrieval_config: Optional[Dict[str, RetrievalConfig]] = None,
    metadata: Optional[Dict[str, MetadataValue]] = None,
    event_timestamp: Optional[datetime] = None,
) -> Tuple[List[Dict[str, Any]], str, Dict[str, Any]]:
    """Complete conversation turn with LLM callback integration.
    
    This method combines memory retrieval, LLM invocation, and response storage
    in a single call using a callback pattern.
    
    Args:
        actor_id: Actor identifier (e.g., "user-123")
        session_id: Session identifier
        user_input: The user's message
        llm_callback: Function that takes (user_input, memories) and returns agent_response.
            The callback receives the user input and retrieved memories,
            and should return the agent's response string.
        retrieval_config: Optional dictionary mapping namespaces to RetrievalConfig objects.
            Each namespace can contain template variables like {actorId}, {sessionId},
            {memoryStrategyId} that will be resolved at runtime.
        metadata: Optional custom key-value metadata to attach to an event.
        event_timestamp: Optional timestamp for the event
    
    Returns:
        Tuple of (retrieved_memories, agent_response, created_event)
    
    Raises:
        ValueError: If llm_callback doesn't return a string
        ClientError: If AWS API call fails
    
    Example:
        >>> def my_llm(user_input: str, memories: List[Dict]) -> str:
        ...     context = "\\n".join([m['content']['text'] for m in memories])
        ...     return call_bedrock(user_input, context)
        >>> 
        >>> retrieval_config = {
        ...     "facts/{sessionId}": RetrievalConfig(top_k=5, relevance_score=0.3)
        ... }
        >>> 
        >>> memories, response, event = session.process_turn_with_llm(
        ...     user_input="What did we discuss?",
        ...     llm_callback=my_llm,
        ...     retrieval_config=retrieval_config
        ... )
    
    Note:
        This method automatically handles the complete flow:
        1. Retrieves relevant memories based on retrieval_config
        2. Invokes the LLM callback with user input and memories
        3. Stores the conversation turn (user input + LLM response)
    
    See Also:
        - :meth:`add_turns`: For manual conversation storage
        - :meth:`search_long_term_memories`: For manual memory retrieval
    """
    pass
```

### 6.2 模块级文档

```python
# memory/__init__.py
"""Bedrock AgentCore Memory module for agent memory management capabilities.

This module provides high-level interfaces for managing conversational AI memory,
including both short-term (conversational events) and long-term (semantic memory)
storage capabilities.

Core Components:
    - MemorySessionManager: Primary interface for managing multiple sessions and actors
    - MemorySession: Session-scoped interface that simplifies operations
    - MemoryControlPlaneClient: Control plane operations for memory resources
    - MemoryClient: Legacy client interface (deprecated)

Quick Start:
    >>> from bedrock_agentcore.memory import MemorySessionManager
    >>> from bedrock_agentcore.memory.constants import ConversationalMessage, MessageRole
    >>> 
    >>> # Initialize the session manager
    >>> manager = MemorySessionManager(memory_id="your-memory-id")
    >>> 
    >>> # Create a session
    >>> session = manager.create_memory_session(
    ...     actor_id="user-123",
    ...     session_id="session-456"
    ... )
    >>> 
    >>> # Add conversation turns
    >>> session.add_turns([
    ...     ConversationalMessage("Hello!", MessageRole.USER),
    ...     ConversationalMessage("Hi there!", MessageRole.ASSISTANT)
    ... ])

For detailed documentation, see the README.md file in this directory.
"""

from .client import MemoryClient
from .controlplane import MemoryControlPlaneClient
from .session import Actor, MemorySession, MemorySessionManager

__all__ = [
    "Actor",
    "MemoryClient",
    "MemorySession",
    "MemorySessionManager",
    "MemoryControlPlaneClient",
]

__version__ = "1.1.1"
```

### 6.3 README 文档结构

```markdown
# Module Name

Brief description of what this module does.

## Table of Contents

- [Overview](#overview)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Concepts](#core-concepts)
- [Usage Examples](#usage-examples)
- [API Reference](#api-reference)
- [Error Handling](#error-handling)
- [Best Practices](#best-practices)
- [Migration Guide](#migration-guide)

## Overview

Detailed description...

## Installation

\```bash
pip install your-sdk
\```

## Quick Start

\```python
from your_sdk import Client

client = Client(api_key="...")
result = client.do_something()
\```

## Core Concepts

### Concept 1: Sessions

Explanation...

### Concept 2: Memory

Explanation...

## Usage Examples

### Example 1: Basic Usage

\```python
# Code example
\```

### Example 2: Advanced Usage

\```python
# Code example
\```

## API Reference

### Class: MemorySessionManager

Description...

#### Methods

##### `create_memory_session(actor_id, session_id)`

Description...

**Parameters:**
- `actor_id` (str): ...
- `session_id` (str): ...

**Returns:**
- `MemorySession`: ...

**Example:**
\```python
session = manager.create_memory_session("user-1", "sess-1")
\```

## Error Handling

### Common Exceptions

- `ResourceNotFoundError`: ...
- `ValidationError`: ...

### Example

\```python
try:
    session.add_turns(messages)
except ValidationError as e:
    print(f"Validation failed: {e}")
\```

## Best Practices

1. Always use MemorySessionManager for new projects
2. Handle exceptions appropriately
3. Use type hints

## Migration Guide

### From v0.x to v1.x

\```python
# Old way (v0.x)
client = MemoryClient()
client.save_turn(...)

# New way (v1.x)
manager = MemorySessionManager(...)
session = manager.create_memory_session(...)
session.add_turns(...)
\```
```

### 6.4 类型存根文件（.pyi）

```python
# memory.pyi
"""Type stubs for memory module."""

from typing import List, Dict, Optional, Tuple, Any, Callable
from datetime import datetime

class MemorySessionManager:
    def __init__(
        self,
        memory_id: str,
        region_name: Optional[str] = ...,
    ) -> None: ...
    
    def add_turns(
        self,
        actor_id: str,
        session_id: str,
        messages: List[ConversationalMessage | BlobMessage],
        branch: Optional[Dict[str, str]] = ...,
        metadata: Optional[Dict[str, MetadataValue]] = ...,
        event_timestamp: Optional[datetime] = ...,
    ) -> Event: ...
    
    def create_memory_session(
        self,
        actor_id: str,
        session_id: Optional[str] = ...,
    ) -> MemorySession: ...

class MemorySession:
    def add_turns(
        self,
        messages: List[ConversationalMessage | BlobMessage],
        branch: Optional[Dict[str, str]] = ...,
        metadata: Optional[Dict[str, MetadataValue]] = ...,
        event_timestamp: Optional[datetime] = ...,
    ) -> Event: ...
```


---

## 7. 测试策略

### 7.1 测试结构

```
tests/
├── unit/                           # 单元测试
│   ├── test_session_manager.py
│   ├── test_models.py
│   └── test_utils.py
├── integration/                    # 集成测试
│   ├── test_memory_operations.py
│   └── test_llm_integration.py
└── fixtures/                       # 测试数据
    ├── __init__.py
    └── sample_data.py
```

### 7.2 单元测试示例

```python
# tests/unit/test_session_manager.py
import pytest
from unittest.mock import Mock, patch, MagicMock
from datetime import datetime

from bedrock_agentcore.memory import MemorySessionManager
from bedrock_agentcore.memory.constants import ConversationalMessage, MessageRole
from bedrock_agentcore.memory.models import Event

class TestMemorySessionManager:
    """MemorySessionManager 单元测试"""
    
    @pytest.fixture
    def mock_boto_client(self):
        """Mock boto3 客户端"""
        with patch('boto3.client') as mock:
            client = MagicMock()
            mock.return_value = client
            yield client
    
    @pytest.fixture
    def manager(self, mock_boto_client):
        """创建测试用的 manager"""
        return MemorySessionManager(
            memory_id="test-memory-123",
            region_name="us-east-1"
        )
    
    def test_create_memory_session(self, manager):
        """测试创建会话"""
        session = manager.create_memory_session(
            actor_id="user-1",
            session_id="sess-1"
        )
        
        assert session._actor_id == "user-1"
        assert session._session_id == "sess-1"
        assert session._memory_id == "test-memory-123"
    
    def test_add_turns_success(self, manager, mock_boto_client):
        """测试成功添加对话"""
        # 设置 mock 返回值
        mock_boto_client.create_event.return_value = {
            'event': {
                'eventId': 'event-123',
                'actorId': 'user-1',
                'sessionId': 'sess-1',
                'eventTimestamp': datetime.utcnow().isoformat()
            }
        }
        
        # 执行测试
        event = manager.add_turns(
            actor_id="user-1",
            session_id="sess-1",
            messages=[
                ConversationalMessage("Hello", MessageRole.USER)
            ]
        )
        
        # 验证
        assert isinstance(event, Event)
        assert event.eventId == 'event-123'
        
        # 验证 API 调用
        mock_boto_client.create_event.assert_called_once()
        call_args = mock_boto_client.create_event.call_args
        assert call_args[1]['memoryId'] == 'test-memory-123'
        assert call_args[1]['actorId'] == 'user-1'
    
    def test_add_turns_empty_messages(self, manager):
        """测试空消息列表"""
        with pytest.raises(ValueError, match="At least one message is required"):
            manager.add_turns(
                actor_id="user-1",
                session_id="sess-1",
                messages=[]
            )
    
    def test_add_turns_api_error(self, manager, mock_boto_client):
        """测试 API 错误处理"""
        from botocore.exceptions import ClientError
        
        # 模拟 API 错误
        mock_boto_client.create_event.side_effect = ClientError(
            {
                'Error': {
                    'Code': 'ResourceNotFoundException',
                    'Message': 'Memory not found'
                }
            },
            'CreateEvent'
        )
        
        # 验证异常
        with pytest.raises(ClientError):
            manager.add_turns(
                actor_id="user-1",
                session_id="sess-1",
                messages=[ConversationalMessage("Hello", MessageRole.USER)]
            )

class TestConversationalMessage:
    """ConversationalMessage 单元测试"""
    
    def test_valid_message(self):
        """测试有效消息"""
        msg = ConversationalMessage("Hello", MessageRole.USER)
        assert msg.text == "Hello"
        assert msg.role == MessageRole.USER
    
    def test_invalid_text_type(self):
        """测试无效文本类型"""
        with pytest.raises(ValueError, match="text must be a string"):
            ConversationalMessage(123, MessageRole.USER)
    
    def test_invalid_role_type(self):
        """测试无效角色类型"""
        with pytest.raises(ValueError, match="role must be a MessageRole"):
            ConversationalMessage("Hello", "INVALID")
```

### 7.3 集成测试示例

```python
# tests/integration/test_memory_operations.py
import pytest
import os
from datetime import datetime

from bedrock_agentcore.memory import MemorySessionManager, MemoryControlPlaneClient
from bedrock_agentcore.memory.constants import ConversationalMessage, MessageRole

@pytest.mark.integration
class TestMemoryIntegration:
    """集成测试（需要真实 AWS 凭证）"""
    
    @pytest.fixture(scope="class")
    def control_client(self):
        """创建控制平面客户端"""
        return MemoryControlPlaneClient(region_name="us-east-1")
    
    @pytest.fixture(scope="class")
    def test_memory(self, control_client):
        """创建测试用的 memory"""
        memory = control_client.create_memory(
            name=f"test-memory-{datetime.now().timestamp()}",
            wait_for_active=True
        )
        
        yield memory
        
        # 清理
        try:
            control_client.delete_memory(memory['id'])
        except Exception as e:
            print(f"Cleanup failed: {e}")
    
    @pytest.fixture
    def manager(self, test_memory):
        """创建 session manager"""
        return MemorySessionManager(
            memory_id=test_memory['id'],
            region_name="us-east-1"
        )
    
    def test_full_conversation_flow(self, manager):
        """测试完整对话流程"""
        # 创建会话
        session = manager.create_memory_session(
            actor_id="test-user",
            session_id="test-session"
        )
        
        # 添加对话
        event = session.add_turns([
            ConversationalMessage("What's the weather?", MessageRole.USER),
            ConversationalMessage("It's sunny today!", MessageRole.ASSISTANT)
        ])
        
        assert event.eventId is not None
        
        # 列出事件
        events = session.list_events()
        assert len(events) >= 1
        
        # 获取特定事件
        retrieved_event = session.get_event(event.eventId)
        assert retrieved_event.eventId == event.eventId
```

### 7.4 Pytest 配置

```python
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

# 标记
markers = [
    "unit: Unit tests",
    "integration: Integration tests (require AWS credentials)",
    "slow: Slow running tests",
]

# 覆盖率
addopts = [
    "--cov=src/bedrock_agentcore",
    "--cov-report=html",
    "--cov-report=term-missing",
    "--cov-fail-under=90",
]

# 异步支持
asyncio_mode = "auto"
```

### 7.5 测试数据工厂

```python
# tests/fixtures/factories.py
"""测试数据工厂"""

from dataclasses import dataclass
from typing import List
from datetime import datetime, timezone

from bedrock_agentcore.memory.constants import ConversationalMessage, MessageRole

@dataclass
class TestData:
    """测试数据容器"""
    memory_id: str = "test-memory-123"
    actor_id: str = "test-user"
    session_id: str = "test-session"

class MessageFactory:
    """消息工厂"""
    
    @staticmethod
    def create_user_message(text: str = "Test message") -> ConversationalMessage:
        """创建用户消息"""
        return ConversationalMessage(text, MessageRole.USER)
    
    @staticmethod
    def create_assistant_message(text: str = "Test response") -> ConversationalMessage:
        """创建助手消息"""
        return ConversationalMessage(text, MessageRole.ASSISTANT)
    
    @staticmethod
    def create_conversation(turns: int = 3) -> List[ConversationalMessage]:
        """创建对话"""
        messages = []
        for i in range(turns):
            messages.append(
                MessageFactory.create_user_message(f"User message {i}")
            )
            messages.append(
                MessageFactory.create_assistant_message(f"Assistant response {i}")
            )
        return messages

# 使用
def test_with_factory():
    messages = MessageFactory.create_conversation(turns=5)
    assert len(messages) == 10
```


---

## 8. 性能优化

### 8.1 连接池和客户端复用

```python
import boto3
from botocore.config import Config as BotocoreConfig

class MemorySessionManager:
    """优化的会话管理器"""
    
    def __init__(
        self,
        memory_id: str,
        region_name: Optional[str] = None,
        boto_client_config: Optional[BotocoreConfig] = None,
    ):
        # 配置连接池
        if boto_client_config is None:
            boto_client_config = BotocoreConfig(
                max_pool_connections=50,      # 连接池大小
                connect_timeout=5,            # 连接超时
                read_timeout=60,              # 读取超时
                retries={
                    'max_attempts': 3,        # 最大重试次数
                    'mode': 'adaptive'        # 自适应重试
                }
            )
        
        # 复用客户端实例
        self._data_plane_client = boto3.client(
            "bedrock-agentcore",
            region_name=region_name,
            config=boto_client_config
        )
```

### 8.2 批量操作

```python
class MemorySessionManager:
    """支持批量操作"""
    
    def batch_delete_memory_records(
        self,
        record_ids: List[str],
        batch_size: int = 100
    ) -> Dict[str, Any]:
        """批量删除记录"""
        
        all_successful = []
        all_failed = []
        
        # 分批处理
        for i in range(0, len(record_ids), batch_size):
            batch = record_ids[i:i + batch_size]
            
            try:
                result = self._data_plane_client.batch_delete_memory_records(
                    memoryId=self._memory_id,
                    records=[{"memoryRecordId": rid} for rid in batch]
                )
                
                all_successful.extend(result.get("successfulRecords", []))
                all_failed.extend(result.get("failedRecords", []))
                
            except ClientError as e:
                logger.error("Batch delete failed: %s", e)
                # 将整个批次标记为失败
                all_failed.extend([{"memoryRecordId": rid} for rid in batch])
        
        return {
            "successfulRecords": all_successful,
            "failedRecords": all_failed
        }
```

### 8.3 分页优化

```python
class MemorySessionManager:
    """优化的分页处理"""
    
    def list_all_events(
        self,
        actor_id: str,
        session_id: str,
        max_results: int = 1000
    ) -> List[Event]:
        """高效的分页列表"""
        
        all_events = []
        next_token = None
        max_iterations = 100  # 防止无限循环
        
        iteration_count = 0
        while len(all_events) < max_results and iteration_count < max_iterations:
            iteration_count += 1
            
            # 动态调整每次请求的数量
            remaining = max_results - len(all_events)
            page_size = min(100, remaining)  # API 限制每次最多 100
            
            params = {
                "memoryId": self._memory_id,
                "actorId": actor_id,
                "sessionId": session_id,
                "maxResults": page_size,
            }
            
            if next_token:
                params["nextToken"] = next_token
            
            try:
                response = self._data_plane_client.list_events(**params)
                events = response.get("events", [])
                
                if not events:
                    break  # 没有更多数据
                
                all_events.extend([Event(e) for e in events])
                next_token = response.get("nextToken")
                
                if not next_token:
                    break  # 没有下一页
                    
            except ClientError as e:
                logger.error("Pagination failed: %s", e)
                raise
        
        return all_events[:max_results]
```

### 8.4 缓存策略

```python
from functools import lru_cache
from typing import Optional
import time

class CachedMemoryClient:
    """带缓存的客户端"""
    
    def __init__(self, memory_id: str):
        self._memory_id = memory_id
        self._cache = {}
        self._cache_ttl = 300  # 5分钟
    
    def get_memory_record(
        self,
        record_id: str,
        use_cache: bool = True
    ) -> Dict[str, Any]:
        """获取记录（带缓存）"""
        
        cache_key = f"record:{record_id}"
        
        # 检查缓存
        if use_cache and cache_key in self._cache:
            cached_data, cached_time = self._cache[cache_key]
            
            if time.time() - cached_time < self._cache_ttl:
                logger.debug("Cache hit for record: %s", record_id)
                return cached_data
        
        # 缓存未命中，调用 API
        logger.debug("Cache miss for record: %s", record_id)
        response = self._client.get_memory_record(
            memoryId=self._memory_id,
            memoryRecordId=record_id
        )
        
        record = response['memoryRecord']
        
        # 更新缓存
        if use_cache:
            self._cache[cache_key] = (record, time.time())
        
        return record
    
    def invalidate_cache(self, record_id: Optional[str] = None):
        """使缓存失效"""
        if record_id:
            cache_key = f"record:{record_id}"
            self._cache.pop(cache_key, None)
        else:
            self._cache.clear()
```

### 8.5 异步支持

```python
import asyncio
from typing import List

class AsyncMemorySessionManager:
    """异步会话管理器"""
    
    async def add_turns_async(
        self,
        actor_id: str,
        session_id: str,
        messages: List[ConversationalMessage]
    ) -> Event:
        """异步添加对话"""
        
        # 使用 asyncio 运行同步代码
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            None,
            self.add_turns,
            actor_id,
            session_id,
            messages
        )
    
    async def batch_operations_async(
        self,
        operations: List[Callable]
    ) -> List[Any]:
        """并发执行多个操作"""
        
        tasks = [
            asyncio.create_task(op())
            for op in operations
        ]
        
        return await asyncio.gather(*tasks, return_exceptions=True)

# 使用
async def main():
    manager = AsyncMemorySessionManager(memory_id="mem-123")
    
    # 并发执行多个操作
    results = await manager.batch_operations_async([
        lambda: manager.add_turns_async("user-1", "sess-1", messages1),
        lambda: manager.add_turns_async("user-2", "sess-2", messages2),
        lambda: manager.search_memories_async("query"),
    ])
```

### 8.6 懒加载

```python
class LazyMemoryClient:
    """懒加载客户端"""
    
    def __init__(self, memory_id: str):
        self._memory_id = memory_id
        self._client = None
        self._strategies = None
    
    @property
    def client(self):
        """懒加载 boto3 客户端"""
        if self._client is None:
            logger.debug("Initializing boto3 client")
            self._client = boto3.client("bedrock-agentcore")
        return self._client
    
    @property
    def strategies(self) -> List[Dict]:
        """懒加载策略列表"""
        if self._strategies is None:
            logger.debug("Loading strategies")
            response = self.client.get_memory(memoryId=self._memory_id)
            self._strategies = response['memory'].get('strategies', [])
        return self._strategies
```


---

## 9. 完整示例

### 9.1 完整的 SDK 模块实现

```python
# src/my_sdk/__init__.py
"""My SDK - A professional Python SDK example.

This SDK demonstrates best practices for building production-ready Python SDKs.
"""

from .client import Client
from .session import SessionManager, Session
from .constants import Status, MessageType
from .exceptions import SDKError, ValidationError, APIError
from .models import Request, Response

__version__ = "1.0.0"

__all__ = [
    # Core classes
    "Client",
    "SessionManager",
    "Session",
    
    # Constants
    "Status",
    "MessageType",
    
    # Exceptions
    "SDKError",
    "ValidationError",
    "APIError",
    
    # Models
    "Request",
    "Response",
]

# Version check
import sys
if sys.version_info < (3, 10):
    raise RuntimeError("Python 3.10+ is required")
```

```python
# src/my_sdk/session.py
"""Session management module."""

import logging
import uuid
from typing import List, Dict, Optional, Any
from datetime import datetime, timezone

import boto3
from botocore.exceptions import ClientError
from botocore.config import Config as BotocoreConfig

from .constants import MessageType, Status
from .exceptions import ValidationError, APIError, ResourceNotFoundError
from .models import Message, Event, DictWrapper

logger = logging.getLogger(__name__)


class SessionManager:
    """
    Manages multiple sessions and provides data plane operations.
    
    This is the recommended entry point for the SDK. It provides a high-level
    interface for managing sessions and performing operations.
    
    Example:
        >>> manager = SessionManager(resource_id="res-123")
        >>> session = manager.create_session(user_id="user-1")
        >>> session.add_message("Hello", MessageType.USER)
    """
    
    def __init__(
        self,
        resource_id: str,
        region_name: Optional[str] = None,
        boto3_session: Optional[boto3.Session] = None,
        boto_client_config: Optional[BotocoreConfig] = None,
    ):
        """
        Initialize SessionManager.
        
        Args:
            resource_id: The resource identifier
            region_name: AWS region name (default: us-west-2)
            boto3_session: Optional boto3 Session
            boto_client_config: Optional boto3 client configuration
        
        Raises:
            ValidationError: If resource_id is invalid
        """
        if not resource_id or not isinstance(resource_id, str):
            raise ValidationError("resource_id must be a non-empty string")
        
        self._resource_id = resource_id
        self.region_name = region_name or "us-west-2"
        
        # Configure client
        if boto_client_config is None:
            boto_client_config = BotocoreConfig(
                max_pool_connections=50,
                connect_timeout=5,
                read_timeout=60,
                retries={'max_attempts': 3, 'mode': 'adaptive'}
            )
        
        # Initialize boto3 client
        session = boto3_session or boto3.Session()
        self._client = session.client(
            "your-service",
            region_name=self.region_name,
            config=boto_client_config
        )
        
        logger.info(
            "Initialized SessionManager for resource: %s in region: %s",
            resource_id,
            self.region_name
        )
    
    def create_session(
        self,
        user_id: str,
        session_id: Optional[str] = None
    ) -> 'Session':
        """
        Create a new session.
        
        Args:
            user_id: User identifier
            session_id: Optional session ID (auto-generated if not provided)
        
        Returns:
            Session object
        
        Example:
            >>> session = manager.create_session(user_id="user-1")
            >>> print(session.session_id)
        """
        session_id = session_id or str(uuid.uuid4())
        
        logger.info(
            "Creating session for user: %s, session: %s",
            user_id,
            session_id
        )
        
        return Session(
            resource_id=self._resource_id,
            user_id=user_id,
            session_id=session_id,
            manager=self
        )
    
    def add_message(
        self,
        user_id: str,
        session_id: str,
        text: str,
        message_type: MessageType,
        metadata: Optional[Dict[str, Any]] = None,
        timestamp: Optional[datetime] = None,
    ) -> Event:
        """
        Add a message to a session.
        
        Args:
            user_id: User identifier
            session_id: Session identifier
            text: Message text
            message_type: Type of message
            metadata: Optional metadata
            timestamp: Optional timestamp
        
        Returns:
            Created event
        
        Raises:
            ValidationError: If input is invalid
            APIError: If API call fails
        """
        # Validation
        if not text or not text.strip():
            raise ValidationError("Message text cannot be empty", field="text")
        
        if not isinstance(message_type, MessageType):
            raise ValidationError("Invalid message type", field="message_type")
        
        # Prepare payload
        payload = {
            "text": text,
            "type": message_type.value,
        }
        
        if metadata:
            payload["metadata"] = metadata
        
        if timestamp is None:
            timestamp = datetime.now(timezone.utc)
        
        try:
            response = self._client.create_event(
                resourceId=self._resource_id,
                userId=user_id,
                sessionId=session_id,
                timestamp=timestamp,
                payload=payload
            )
            
            logger.info(
                "Created event: %s for user: %s, session: %s",
                response['eventId'],
                user_id,
                session_id
            )
            
            return Event(response['event'])
            
        except ClientError as e:
            self._handle_client_error(e)
    
    def list_events(
        self,
        user_id: str,
        session_id: str,
        max_results: int = 100
    ) -> List[Event]:
        """
        List events in a session.
        
        Args:
            user_id: User identifier
            session_id: Session identifier
            max_results: Maximum number of events to return
        
        Returns:
            List of events
        """
        all_events = []
        next_token = None
        
        while len(all_events) < max_results:
            params = {
                "resourceId": self._resource_id,
                "userId": user_id,
                "sessionId": session_id,
                "maxResults": min(100, max_results - len(all_events))
            }
            
            if next_token:
                params["nextToken"] = next_token
            
            try:
                response = self._client.list_events(**params)
                events = response.get("events", [])
                
                if not events:
                    break
                
                all_events.extend([Event(e) for e in events])
                next_token = response.get("nextToken")
                
                if not next_token:
                    break
                    
            except ClientError as e:
                self._handle_client_error(e)
        
        logger.info(
            "Retrieved %d events for user: %s, session: %s",
            len(all_events),
            user_id,
            session_id
        )
        
        return all_events[:max_results]
    
    def _handle_client_error(self, error: ClientError) -> None:
        """Handle boto3 ClientError."""
        error_code = error.response['Error']['Code']
        error_msg = error.response['Error']['Message']
        request_id = error.response['ResponseMetadata'].get('RequestId')
        
        if error_code == 'ResourceNotFoundException':
            raise ResourceNotFoundError(
                f"Resource not found: {error_msg}",
                error_code=error_code,
                request_id=request_id
            ) from error
        
        elif error_code == 'ValidationException':
            raise ValidationError(error_msg) from error
        
        else:
            raise APIError(
                f"API call failed: {error_msg}",
                error_code=error_code,
                request_id=request_id
            ) from error


class Session(DictWrapper):
    """
    Session-scoped proxy that simplifies operations.
    
    This class automatically injects user_id and session_id into operations,
    reducing parameter repetition.
    """
    
    def __init__(
        self,
        resource_id: str,
        user_id: str,
        session_id: str,
        manager: SessionManager
    ):
        """Initialize Session."""
        self._resource_id = resource_id
        self._user_id = user_id
        self._session_id = session_id
        self._manager = manager
        
        super().__init__(self._construct_dict())
    
    def _construct_dict(self) -> Dict[str, Any]:
        """Construct dictionary representation."""
        return {
            "resourceId": self._resource_id,
            "userId": self._user_id,
            "sessionId": self._session_id,
        }
    
    @property
    def user_id(self) -> str:
        """Get user ID."""
        return self._user_id
    
    @property
    def session_id(self) -> str:
        """Get session ID."""
        return self._session_id
    
    def add_message(
        self,
        text: str,
        message_type: MessageType,
        metadata: Optional[Dict[str, Any]] = None,
        timestamp: Optional[datetime] = None,
    ) -> Event:
        """
        Add a message (delegates to manager).
        
        Args:
            text: Message text
            message_type: Type of message
            metadata: Optional metadata
            timestamp: Optional timestamp
        
        Returns:
            Created event
        """
        return self._manager.add_message(
            self._user_id,
            self._session_id,
            text,
            message_type,
            metadata,
            timestamp
        )
    
    def list_events(self, max_results: int = 100) -> List[Event]:
        """List events (delegates to manager)."""
        return self._manager.list_events(
            self._user_id,
            self._session_id,
            max_results
        )
```


### 9.2 配置文件（pyproject.toml）

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-sdk"
version = "1.0.0"
description = "A professional Python SDK"
readme = "README.md"
requires-python = ">=3.10"
license = {text = "Apache-2.0"}
authors = [
    { name = "Your Name", email = "your.email@example.com" }
]

classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: Apache Software License",
    "Operating System :: OS Independent",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Topic :: Software Development :: Libraries :: Python Modules",
]

dependencies = [
    "boto3>=1.40.0",
    "botocore>=1.40.0",
    "pydantic>=2.0.0,<3.0.0",
    "typing-extensions>=4.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.0.0",
    "mypy>=1.8.0",
    "ruff>=0.2.0",
    "pre-commit>=3.0.0",
]

[project.urls]
Homepage = "https://github.com/yourorg/my-sdk"
Documentation = "https://my-sdk.readthedocs.io"
Repository = "https://github.com/yourorg/my-sdk"
"Bug Tracker" = "https://github.com/yourorg/my-sdk/issues"

[tool.hatch.build.targets.wheel]
packages = ["src/my_sdk"]

# MyPy 配置
[tool.mypy]
python_version = "3.10"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true

# Ruff 配置
[tool.ruff]
line-length = 120
target-version = "py310"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "B",    # flake8-bugbear
    "C4",   # flake8-comprehensions
    "UP",   # pyupgrade
    "D",    # pydocstyle
]

ignore = [
    "E501",  # line too long (handled by formatter)
    "D100",  # missing docstring in public module
    "D104",  # missing docstring in public package
]

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["D"]  # Don't require docstrings in tests

# Pytest 配置
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

markers = [
    "unit: Unit tests",
    "integration: Integration tests",
    "slow: Slow running tests",
]

addopts = [
    "--cov=src/my_sdk",
    "--cov-report=html",
    "--cov-report=term-missing",
    "--cov-fail-under=90",
    "-v",
]

asyncio_mode = "auto"

# Coverage 配置
[tool.coverage.run]
branch = true
source = ["src"]
omit = [
    "*/tests/*",
    "*/__pycache__/*",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
]
```

### 9.3 使用示例

```python
# examples/basic_usage.py
"""Basic usage example."""

from my_sdk import SessionManager
from my_sdk.constants import MessageType

def main():
    """Run basic example."""
    
    # Initialize manager
    manager = SessionManager(
        resource_id="res-123",
        region_name="us-east-1"
    )
    
    # Create session
    session = manager.create_session(user_id="user-1")
    
    # Add messages
    session.add_message("Hello!", MessageType.USER)
    session.add_message("Hi there!", MessageType.ASSISTANT)
    
    # List events
    events = session.list_events()
    print(f"Found {len(events)} events")
    
    for event in events:
        print(f"Event {event.eventId}: {event.payload}")

if __name__ == "__main__":
    main()
```

```python
# examples/advanced_usage.py
"""Advanced usage example with error handling."""

import logging
from datetime import datetime, timezone

from my_sdk import SessionManager
from my_sdk.constants import MessageType
from my_sdk.exceptions import ValidationError, APIError, ResourceNotFoundError

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)

def main():
    """Run advanced example."""
    
    try:
        # Initialize with custom configuration
        manager = SessionManager(
            resource_id="res-123",
            region_name="us-east-1"
        )
        
        # Create session
        session = manager.create_session(
            user_id="user-1",
            session_id="custom-session-id"
        )
        
        # Add message with metadata
        event = session.add_message(
            text="What's the weather?",
            message_type=MessageType.USER,
            metadata={
                "location": "New York",
                "timestamp": datetime.now(timezone.utc).isoformat()
            }
        )
        
        logger.info("Created event: %s", event.eventId)
        
        # List events with pagination
        all_events = session.list_events(max_results=100)
        logger.info("Retrieved %d events", len(all_events))
        
    except ValidationError as e:
        logger.error("Validation error: %s (field: %s)", e, e.field)
    
    except ResourceNotFoundError as e:
        logger.error("Resource not found: %s (request_id: %s)", e, e.request_id)
    
    except APIError as e:
        logger.error(
            "API error: %s (code: %s, request_id: %s)",
            e,
            e.error_code,
            e.request_id
        )
    
    except Exception as e:
        logger.exception("Unexpected error: %s", e)

if __name__ == "__main__":
    main()
```

---

## 10. 总结与检查清单

### 10.1 SDK 开发检查清单

#### 项目结构 ✓
- [ ] 使用 `src/` 布局
- [ ] 清晰的公私分离
- [ ] 模块化组织
- [ ] 完整的 `__init__.py`

#### API 设计 ✓
- [ ] 渐进式 API（简单到复杂）
- [ ] API 扁平化
- [ ] 代理模式减少参数重复
- [ ] 一致的命名规范

#### 类型系统 ✓
- [ ] 完整的类型注解
- [ ] Pydantic 数据验证
- [ ] Dataclass 用于数据结构
- [ ] TypedDict 用于字典结构

#### 错误处理 ✓
- [ ] 自定义异常层次
- [ ] 细粒度错误捕获
- [ ] 重试机制
- [ ] 结构化日志

#### 版本管理 ✓
- [ ] 优雅的废弃机制
- [ ] 向后兼容性
- [ ] 版本检查
- [ ] 迁移指南

#### 文档 ✓
- [ ] 完整的 Docstring
- [ ] 模块级文档
- [ ] README 文档
- [ ] 使用示例

#### 测试 ✓
- [ ] 单元测试
- [ ] 集成测试
- [ ] 测试覆盖率 >90%
- [ ] 测试数据工厂

#### 性能 ✓
- [ ] 连接池配置
- [ ] 批量操作
- [ ] 分页优化
- [ ] 缓存策略

#### Python 版本 ✓
- [ ] 明确声明 `requires-python`
- [ ] 运行时版本检查
- [ ] 完整的测试矩阵
- [ ] 版本升级策略

### 10.2 关键要点

1. **以用户体验为中心**
   - 简单场景要简单
   - 复杂场景要灵活
   - 提供清晰的迁移路径

2. **类型安全**
   - 使用类型注解
   - Pydantic 验证
   - 运行时检查

3. **错误处理**
   - 细粒度异常
   - 有用的错误信息
   - 自动重试

4. **向后兼容**
   - 废弃警告
   - 字段名规范化
   - 版本检查

5. **文档完善**
   - 代码即文档
   - 示例丰富
   - 迁移指南

6. **测试充分**
   - 单元测试
   - 集成测试
   - 高覆盖率

7. **性能优化**
   - 连接复用
   - 批量操作
   - 智能缓存

8. **Python 版本管理**
   - 合理的最低版本
   - 运行时检查
   - 测试矩阵覆盖

---

**参考资源：**
- [AWS Bedrock AgentCore SDK](https://github.com/aws/bedrock-agentcore-sdk-python)
- [Python Packaging Guide](https://packaging.python.org/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)

