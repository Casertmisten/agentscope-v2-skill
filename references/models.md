# 模型与认证 (Model & Credential)

## Credential 体系

v2 使用 `CredentialBase` 子类封装 API 认证，替代 v1 的直接传 api_key。

### 内置 Credential 类

```python
from agentscope.credential import (
    OpenAICredential,
    AnthropicCredential,
    DashScopeCredential,
    GeminiCredential,
    OllamaCredential,
    DeepSeekCredential,
    MoonshotCredential,
    XAICredential,
)
```

### 使用方式

```python
# 每个 credential 类对应一个模型提供商
credential = OpenAICredential(api_key="sk-xxx")

# 列出可用模型
models = credential.list_models()

# 获取对应的 ChatModel 类
model_cls = credential.get_chat_model_class()
```

### CredentialFactory

用于从 dict 反序列化 credential（配合数据库存储）：

```python
from agentscope.credential import CredentialFactory

credential = CredentialFactory.from_dict({"type": "openai", "api_key": "sk-xxx"})
```

## ChatModelBase

所有模型的基类，通过 `credential.get_chat_model_class()` 获取具体实现。

### 内置 Model 类

```python
from agentscope.model import (
    OpenAIChatModel,
    OpenAIResponseModel,     # OpenAI Responses API
    AnthropicChatModel,
    DashScopeChatModel,
    DeepSeekChatModel,       # 新增
    GeminiChatModel,
    MoonshotChatModel,       # 新增
    OllamaChatModel,
    XAIChatModel,            # 新增
)
```

### 构造参数

```python
model = credential.get_chat_model_class()(
    credential=credential,
    model="gpt-4o",          # 模型名称
    parameters=SomeParameters(  # 各提供商的参数类
        temperature=0.3,
        max_tokens=1000,
    ),
    stream=True,              # 是否流式
    max_retries=3,            # 重试次数
    retry_delay=1.0,          # 重试间隔
    context_size=32768,       # 上下文大小（用于压缩）
)
```

### 调用方式

```python
from agentscope.message import Msg

# 非流式
response = await model(messages=[user_msg], tools=tool_schemas, tool_choice=None)
# response 是 ChatResponse

# 流式
async for chunk in await model(messages=[...], tools=None):
    # chunk 是 ChatResponse（累加式）
    pass
```

### 结构化输出

```python
from pydantic import BaseModel, Field

class MyOutput(BaseModel):
    name: str = Field(description="姓名")

result = await model.generate_structured_output(
    messages=[user_msg],
    structured_model=MyOutput,
)
# result.content 是 dict
```

### 相关类型

```python
from agentscope.model import (
    ChatResponse,       # 模型响应
    StructuredResponse, # 结构化输出响应
    ChatUsage,          # token 用量
    ModelCard,          # 模型卡片
)
```

## 在 Agent 中使用

Agent 需要接收一个已构建的 `ChatModelBase` 实例：

```python
credential = OpenAICredential(api_key="sk-xxx")
model = credential.get_chat_model_class()(
    credential=credential,
    model="gpt-4o",
)

agent = Agent(
    name="Assistant",
    system_prompt="你是一个有用的助手。",
    model=model,
)
```
