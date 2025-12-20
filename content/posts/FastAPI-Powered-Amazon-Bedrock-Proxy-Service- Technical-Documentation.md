---
title: "FastAPI 驱动的 Amazon Bedrock 代理服务：架构设计与实现"
date: 2025-12-21
draft: false
tags: ["FastAPI", "Amazon Bedrock", "Python", "AI", "代理服务", "AWS"]
categories: ["技术分享"]
---

# FastAPI 驱动的 Amazon Bedrock 代理服务：架构设计与实现

本文将介绍一个基于 FastAPI 的轻量级代理服务，它将 Amazon Bedrock 的强大功能封装在简洁的 HTTP API 背后。我们将深入探讨 FastAPI 的核心原理与优势，并详细说明为什么选择 FastAPI 作为此项目的 Web 框架。

**项目地址**：https://github.com/mingyu110/AI/tree/main/bedrock-proxy

## 一、FastAPI 框架：现代 Python Web 开发的优选方案

### 1.1 FastAPI 的核心原理

FastAPI 是一个现代、高性能的 Python Web 框架，基于以下关键技术构建：

- **Starlette**：提供 ASGI 支持和 Web 框架基础功能
- **Pydantic**：提供数据验证和序列化
- **类型提示**：利用 Python 3.6+ 的类型注解实现自动数据验证和文档生成

FastAPI 的核心工作原理可以概括为：

```python
# FastAPI 自动处理请求-响应循环
# 1. 接收 HTTP 请求
# 2. 验证请求数据（基于 Pydantic 模型）
# 3. 执行业务逻辑
# 4. 序列化响应数据
# 5. 返回 HTTP 响应
```

### 1.2 FastAPI 的核心功能特性

#### 1.2.1 自动 API 文档生成
FastAPI 最强大的特性之一是能够自动生成 OpenAPI 规范的交互式文档：

```python
# 访问 /docs 路径即可获得 Swagger UI 界面
# 访问 /redoc 路径可获得 ReDoc 界面
```

#### 1.2.2 类型安全与自动验证
利用 Python 类型提示，FastAPI 在运行时自动验证请求和响应数据：

```python
from pydantic import BaseModel
from typing import List, Optional

class ChatRequest(BaseModel):
    mode: str = "chat"
    model: Optional[str] = None
    payload: dict
    stream: bool = False
```

#### 1.2.3 异步高性能
基于 asyncio 和 ASGI，FastAPI 能够处理大量并发请求：

```python
# 支持 async/await 语法
async def handle_request(request: Request):
    # 非阻塞 IO 操作
    result = await external_service.call()
    return result
```

#### 1.2.4 依赖注入系统
FastAPI 提供优雅的依赖注入机制，便于管理共享资源：

```python
# 共享 Bedrock 客户端实例
def get_bedrock_client() -> boto3.client:
    # 单例模式创建客户端
    pass
```

## 二、为什么选择 FastAPI 作为 Bedrock 代理框架

### 2.1 性能考量：应对 AI 推理的高延迟场景

Amazon Bedrock 调用通常存在较高延迟（100-2000ms），使用 FastAPI 的异步特性可以：

- **避免请求阻塞**：当一个请求在等待 Bedrock 响应时，服务器可以处理其他请求
- **支持高并发**：单进程可处理数百个并发连接
- **资源高效利用**：非阻塞 IO 减少线程上下文切换开销

```python
# 在传统同步框架中
# 每个请求占用一个线程，线程池耗尽后无法处理新请求

# 在 FastAPI 异步模式中
# 单个线程可处理多个并发请求，通过事件循环调度
```

### 2.2 开发效率：快速迭代 AI 应用

#### 2.2.1 自动生成的 API 文档
团队成员可以直接通过浏览器测试 API，无需额外工具：

```python
# 启动服务后访问 http://localhost:8000/docs
# 即可看到完整的 API 文档和测试界面
```

#### 2.2.2 类型安全减少错误
在构建复杂的 AI 请求负载时，类型提示能及早发现错误：

```python
# 错误示例：将字符串传入需要字典的字段
payload = "text content"  # 类型错误将在开发时被发现

# 正确示例
payload = {"messages": [{"role": "user", "content": "text"}]}
```

### 2.3 生态系统集成

FastAPI 与现代 Python 工具链无缝集成：

- **boto3**：AWS SDK 的异步兼容
- **Pydantic**：与 Bedrock 的 JSON 请求/响应完美匹配
- **pytest**：支持异步测试
- **Docker**：轻量化容器部署

### 2.4 项目长期可维护性

#### 2.4.1 清晰的代码结构
```
api/          # HTTP 层：路由、请求处理
├── main.py          # FastAPI 应用实例
├── chat_handler.py  # 请求路由和编排
└── response.py      # 响应辅助函数

core/         # 核心层：配置、客户端
├── settings.py      # 集中配置管理
└── bedrock_client.py # 共享 Bedrock 客户端

services/     # 服务层：业务逻辑
├── chat_service.py      # 聊天服务
└── embedding_service.py # 嵌入服务
```

#### 2.4.2 配置集中管理
所有配置集中在 `core/settings.py`，便于管理和修改：

```python
@dataclass
class AppSettings:
    aws_region: str = "us-east-1"
    bedrock_model_id: str = "anthropic.claude-3-sonnet-20240229-v1:0"
    # 修改一处，全局生效
```

## 三、Bedrock 代理服务核心架构

### 3.1 整体架构概览

```
┌─────────────────┐     HTTP     ┌─────────────────┐    AWS SDK     ┌─────────────────┐
│   您的应用      │ ───────────→ │  FastAPI 代理   │ ───────────→ │ Amazon Bedrock  │
│ (Web/移动/API)  │   /chat      │  (本服务)       │ boto3        │  (AI 服务)      │
└─────────────────┘              └─────────────────┘              └─────────────────┘
        ↑                                ↑
        │ 无需 AWS SDK                    │ 统一错误处理
        │ 无需管理凭据                    │ 连接池复用
        │ 简单 HTTP                       │ 自动重试
```

### 3.2 FastAPI 应用核心结构

#### 3.2.1 应用入口 (api/main.py)

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(
    title="Amazon Bedrock 代理服务",
    description="统一的 HTTP API 接口，用于访问 Bedrock 的聊天和嵌入功能",
    version="1.0.0"
)

# CORS 配置，支持前端跨域调用
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 生产环境应配置具体域名
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.post("/chat")
async def chat(request: Request):
    """统一的聊天和嵌入接口"""
    body = await request.json()
    return await handle_chat_request(body)

@app.get("/health")
async def health():
    """健康检查端点"""
    return {"status": "ok", "service": "bedrock-proxy"}
```

#### 3.2.2 请求处理核心 (api/chat_handler.py)

```python
async def handle_chat_request(body: Dict[str, Any]) -> JSONResponse | StreamingResponse:
    """
    主要路由逻辑：
    1. 判断模式：chat（聊天）还是 embed（嵌入）
    2. 判断响应方式：流式还是非流式
    3. 调用相应的服务
    4. 返回格式化的响应
    """
    mode = body.get("mode", "chat")

    if mode == "embed":
        # 处理嵌入请求
        return await handle_embedding(body)
    else:
        # 处理聊天请求
        stream = body.get("stream", False)
        if stream:
            return await handle_streaming_chat(body)
        else:
            return await handle_sync_chat(body)
```

### 3.3 Bedrock 客户端管理 (core/bedrock_client.py)

```python
def get_bedrock_client() -> boto3.client:
    """
    创建并复用 Bedrock 客户端实例

    关键配置：
    - max_pool_connections: 连接池大小（默认 50）
    - retries: 自适应重试策略
    - region: AWS 区域
    """
    global _bedrock_client

    if _bedrock_client is None:
        settings = AppSettings.instance()
        config = Config(
            region_name=settings.aws_region,
            max_pool_connections=settings.max_pool_connections,
            retries={
                "max_attempts": settings.max_retry_attempts,
                "mode": "adaptive"
            }
        )
        _bedrock_client = boto3.client(
            "bedrock-runtime",
            config=config
        )

    return _bedrock_client
```

### 3.4 聊天服务实现 (services/chat_service.py)

#### 3.4.1 同步聊天模式
适用于不需要实时响应的场景：

```python
def chat_sync(self, body: dict) -> dict:
    """
    同步聊天流程：
    1. 准备请求负载
    2. 调用 invoke_model
    3. 解析响应
    4. 返回完整结果
    """
    response = self.client.invoke_model(**invoke_kwargs)
    response_body = json.loads(response["body"].read())

    return {
        "answer": self._extract_text(response_body),
        "usage": response_body.get("usage", {}),
        "model": model_id
    }
```

#### 3.4.2 流式聊天模式
适用于需要实时显示生成内容的场景：

```python
def chat_stream(self, prepared: tuple):
    """
    流式聊天流程：
    1. 准备请求负载
    2. 调用 invoke_model_with_response_stream
    3. 生成器逐块返回内容
    4. 客户端实时接收并显示
    """
    response = self.client.invoke_model_with_response_stream(**invoke_kwargs)

    def generate():
        for event in response["body"]:
            chunk = json.loads(event["chunk"]["bytes"])
            yield {
                "text": chunk.get("text", ""),
                "is_final": False
            }

    return generate()
```

### 3.5 嵌入服务实现 (services/embedding_service.py)

```python
def embed(self, text: str, model_id: Optional[str] = None) -> dict:
    """
    文本嵌入流程：
    1. 构建请求负载（支持 Titan 和 Cohere）
    2. 调用 invoke_model
    3. 解析嵌入向量
    4. 返回标准化格式
    """
    model = model_id or self.settings.bedrock_embed_model_id
    payload = self._build_payload(text, model)

    response = self.client.invoke_model(
        modelId=model,
        body=json.dumps(payload),
        contentType="application/json",
        accept="application/json"
    )

    body = json.loads(response["body"].read())
    embedding = self._parse_embedding(body, model)

    return {
        "embedding": embedding,
        "dimensions": len(embedding),
        "model": model
    }
```

## 四、配置管理最佳实践

### 4.1 环境变量驱动的配置

```python
# core/settings.py
@dataclass
class AppSettings:
    # AWS 配置
    aws_region: str = os.getenv("AWS_REGION", "us-east-1")
    bedrock_model_id: str = os.getenv(
        "BEDROCK_MODEL_ID",
        "anthropic.claude-3-sonnet-20240229-v1:0"
    )

    # Guardrails 配置
    guardrail_id: Optional[str] = os.getenv("BEDROCK_GUARDRAIL_ID")
    guardrail_version: str = os.getenv("BEDROCK_GUARDRAIL_VERSION", "1")

    # HTTP 调优
    max_pool_connections: int = int(os.getenv("MAX_POOL_CONNECTIONS", "50"))
    max_retry_attempts: int = int(os.getenv("MAX_RETRY_ATTEMPTS", "5"))

# 单例模式，全局复用
_settings = None

@classmethod
def instance(cls) -> "AppSettings":
    if cls._settings is None:
        cls._settings = cls()
    return cls._settings
```

### 4.2 .env 文件示例

```bash
# AWS 配置
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key

# Bedrock 模型配置
BEDROCK_MODEL_ID=anthropic.claude-3-sonnet-20240229-v1:0
BEDROCK_EMBED_MODEL_ID=amazon.titan-embed-text-v2:0

# Guardrails（可选）
BEDROCK_GUARDRAIL_ID=gr-xxxxxxxx
BEDROCK_GUARDRAIL_VERSION=1

# CORS 配置
ALLOWED_ORIGINS=http://localhost:3000,https://your-app.com

# 性能调优
MAX_POOL_CONNECTIONS=50
MAX_RETRY_ATTEMPTS=5
```

## 五、部署与使用

### 5.1 本地开发环境

```bash
# 1. 克隆项目
git clone https://github.com/loaynaser3/bedrock-proxy.git
cd bedrock-proxy

# 2. 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/Mac
# 或 venv\Scripts\activate  # Windows

# 3. 安装依赖
pip install -r requirements.txt

# 4. 配置 AWS 凭据
# 方式 1: 使用 .env 文件
cp .env.example .env
# 编辑 .env 文件填写 AWS 凭据

# 方式 2: 使用 AWS CLI
aws configure

# 5. 启动服务
uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload

# 6. 访问文档
# 浏览器打开: http://localhost:8000/docs
```

### 5.2 Docker 部署

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
# 构建镜像
docker build -t bedrock-proxy .

# 运行容器
docker run -d \
  --name bedrock-proxy \
  -p 8000:8000 \
  --env-file .env \
  bedrock-proxy
```

### 5.3 Kubernetes 部署示例

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bedrock-proxy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bedrock-proxy
  template:
    metadata:
      labels:
        app: bedrock-proxy
    spec:
      containers:
      - name: proxy
        image: your-registry/bedrock-proxy:latest
        ports:
        - containerPort: 8000
        env:
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_MODEL_ID
          value: "anthropic.claude-3-sonnet-20240229-v1:0"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: bedrock-proxy-service
spec:
  selector:
    app: bedrock-proxy
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

## 六、API 使用示例

### 6.1 聊天补全（同步模式）

```python
import requests

response = requests.post(
    "http://localhost:8000/chat",
    json={
        "mode": "chat",
        "model": "anthropic.claude-3-sonnet-20240229-v1:0",
        "payload": {
            "anthropic_version": "bedrock-2023-05-31",
            "messages": [
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": "解释 DevOps 概念"}
                    ]
                }
            ],
            "max_tokens": 256
        }
    }
)

result = response.json()
print(f"回答: {result['answer']}")
print(f"用量: {result['usage']}")
```

### 6.2 流式聊天（实时响应）

```python
import requests
import json

response = requests.post(
    "http://localhost:8000/chat",
    json={
        "mode": "chat",
        "stream": True,
        "model": "anthropic.claude-3-sonnet-20240229-v1:0",
        "payload": {
            "anthropic_version": "bedrock-2023-05-31",
            "messages": [
                {
                    "role": "user",
                    "content": "写一段关于机器学习的介绍"
                }
            ],
            "max_tokens": 512
        }
    },
    stream=True
)

# 逐块接收并显示
for line in response.iter_lines():
    if line:
        chunk = json.loads(line)
        if "text" in chunk:
            print(chunk["text"], end="", flush=True)
```

### 6.3 文本嵌入

```python
import requests

response = requests.post(
    "http://localhost:8000/chat",
    json={
        "mode": "embed",
        "model": "amazon.titan-embed-text-v2:0",
        "text": "这是一个测试句子"
    }
)

result = response.json()
print(f"嵌入维度: {result['dimensions']}")
print(f"前5个值: {result['embedding'][:5]}")
```

## 七、产品级增强建议

### 7.1 认证与授权

```python
# 添加 API Key 认证
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def verify_api_key(credentials: HTTPAuthorizationCredentials = Depends(security)):
    if credentials.credentials not in VALID_API_KEYS:
        raise HTTPException(status_code=401, detail="Invalid API Key")
    return credentials.credentials

@app.post("/chat")
async def chat(request: Request, api_key: str = Depends(verify_api_key)):
    # 业务逻辑
    pass
```

### 7.2 速率限制

```python
from fastapi_limiter import FastAPILimiter
from fastapi_limiter.depends import RateLimiter

@app.post("/chat", dependencies=[Depends(RateLimiter(times=10, seconds=60))])
async def chat(request: Request):
    # 每个用户每分钟最多 10 次请求
    pass
```

### 7.3 请求日志与监控

```python
import logging
from datetime import datetime

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = datetime.now()
    response = await call_next(request)
    duration = (datetime.now() - start_time).total_seconds()

    logger.info(
        f"{request.method} {request.url.path} "
        f"Status: {response.status_code} "
        f"Duration: {duration:.3f}s"
    )
    return response
```

### 7.4 缓存策略

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def get_embedding_cached(text: str, model: str):
    # 缓存频繁请求的嵌入结果
    return embedding_service.embed(text, model)
```

### 7.5 负载均衡与高可用

在生产环境建议：

1. **部署多个实例**：使用 Kubernetes 或 ECS 部署 3+ 实例
2. **配置健康检查**：定期探测 /health 端点
3. **自动扩缩容**：根据 CPU/内存指标自动调整实例数
4. **使用 ALB**：AWS Application Load Balancer 分发流量

## 八、为什么选择代理架构

### 8.1 安全性优势

**传统直接访问模式：**
```
前端/移动端 → 暴露 AWS 凭据 → 直接调用 Bedrock
❌ 凭据泄露风险
❌ 无法控制访问权限
❌ 难以审计使用量
```

**代理模式：**
```
前端/移动端 → HTTP 调用 → FastAPI 代理 → 安全凭据 → Bedrock
✅ 凭据不暴露
✅ 可添加自定义认证
✅ 完整使用日志
✅ 可实施速率限制
```

### 8.2 开发效率提升

| 对比项 | 直接调用 Bedrock | 使用代理服务 |
|--------|----------------|------------|
| 前端代码复杂度 | 需要 AWS SDK，处理认证 | 简单 HTTP 请求 |
| 多应用维护 | 各应用独立配置 | 统一配置 |
| 模型切换 | 每个应用修改代码 | 修改代理配置 |
| 错误处理 | 各自实现 | 统一处理 |
| 连接池管理 | 各自实现 | 集中优化 |

### 8.3 可扩展性设计

代理架构为 future enhancement 提供了扩展点：

```python
# 示例：添加自定义工具调用
class CustomToolService:
    def __init__(self, bedrock_client):
        self.client = bedrock_client

    def register_tool(self, name: str, function: Callable):
        # 注册自定义工具
        pass

    def execute_tool_call(self, tool_call: dict):
        # 执行工具调用
        pass

# 集成到 ChatService
chat_service = ChatService(bedrock_client, settings)
chat_service.tool_executor = CustomToolService(bedrock_client)
```

```python
# 示例：添加请求拦截器
class RequestInterceptor:
    async def before_request(self, request: dict):
        # 日志记录
        # 内容审核
        # 格式转换
        pass

    async def after_response(self, response: dict):
        # 结果缓存
        # 后处理
        pass
```

## 九、性能优化技巧

### 9.1 连接池调优

```python
# core/bedrock_client.py
config = Config(
    region_name=settings.aws_region,
    max_pool_connections=100,  # 根据并发量调整
    retries={
        "max_attempts": 3,
        "mode": "adaptive"  # 自适应重试策略
    }
)
```

### 9.2 流式响应优化

```python
# 设置合适的分块大小
@app.post("/chat")
async def chat(request: Request):
    response = await chat_service.chat_stream(prepared)

    # 使用 StreamingResponse 减少内存占用
    return StreamingResponse(
        response,
        media_type="application/x-ndjson",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no"  # 禁用代理缓冲
        }
    )
```

### 9.3 模型预热

```python
# 应用启动时预热连接
@app.on_event("startup")
async def startup_event():
    # 初始化 Bedrock 客户端
    _ = get_bedrock_client()

    # 可选：发送一个测试请求预热模型
    logger.info("Bedrock 代理服务启动完成")
```

## 十、总结

FastAPI 作为现代化 Python Web 框架，为构建 Amazon Bedrock 代理服务提供了理想的 foundation：

**技术优势：**
- **高性能**：异步 IO 支持高并发，适合 AI 推理场景
- **自动文档**：Swagger UI 降低团队协作成本
- **类型安全**：Pydantic 模型确保数据完整性
- **易于扩展**：中间件和依赖注入支持业务需求演变
- **轻量级**：最小化运行时开销

**业务价值：**
- **安全性**：AWS 凭据不再暴露给前端
- **一致性**：统一模型配置和 Guardrails 策略
- **可观测性**：集中日志和监控
- **简化开发**：客户端无需 AWS SDK 知识

通过本文介绍的架构，开发团队可以：
1. **快速集成**：任何支持 HTTP 的语言/框架均可调用
2. **专注业务**：屏蔽 AWS SDK 实现细节
3. **灵活演进**：在代理层添加新功能不影响客户端
4. **生产就绪**：易于扩展认证、监控、缓存等企业级特性

无论您是构建 AI 驱动的 Web 应用、移动应用还是微服务架构，这个 FastAPI + Amazon Bedrock 的代理模式都能显著提升开发效率和系统可维护性。

---

