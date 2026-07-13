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

### 多模态能力方法（v2.0.2+）

Credential 不仅管理对话模型，还统一暴露 Embedding 和 TTS 能力：

```python
credential = DashScopeCredential(api_key="ds-xxx")

# 对话模型
chat_cls = credential.get_chat_model_class()        # -> DashScopeChatModel
chat_models = credential.list_models()              # 对话模型卡片

# Embedding 模型（v2.0.2+）
emb_cls = credential.get_embedding_model_class()    # -> DashScopeEmbeddingModel | None
# 注意：返回单个类或 None，未支持的提供商返回 None

# TTS 模型（v2.0.2+）
tts_classes = credential.get_tts_model_classes()    # -> list[TTSModelBase 子类]
tts_cards = credential.list_tts_models()            # -> list[TTSModelCard]
# 注意：TTS 返回的是「列表」（一个提供商可能有多个模型，如普通/实时两个）
```

> 各提供商支持的能力不同：DashScope 同时支持对话/Embedding/TTS；OpenAI/Gemini/Ollama 支持 Embedding（`get_embedding_model_class()`）；多数提供商的 TTS 列表为空。

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

### OpenAI Chat 模型专属参数

`OpenAIChatModel` 额外支持透传到 OpenAI 兼容 API 的字段（v2.0.3+）：

```python
from agentscope.model import OpenAIChatModel

model = OpenAIChatModel(
    credential=credential,
    model="custom-model",
    client_kwargs={"timeout": 60.0},    # 转发给 openai.AsyncClient
    extra_body={"thinking": {"type": "enabled"}},  # 透传到请求体
)
```

- `client_kwargs` — 透传给 `openai.AsyncClient`（如 `timeout`/`default_headers`/`http_client`）。
- `extra_body` — 请求体级扩展字段，适配 DeepSeek、通义、vLLM 等兼容服务商的私有参数；
  不传时不会在请求里出现 `extra_body` 键。

### Omni 模型的音频输出（v2.0.2+）

> ⚠️ 实验性 / 积极开发中：Omni 音频输出属于官方 Voice Agent 路线（roadmap）的多模态阶段，
> API 和行为可能随版本变动。

支持音频输出的 omni 风格模型（如 `gpt-audio-mini`、`qwen3.5-omni-plus`）新增 `voice` 参数。设置后框架会自动把请求的 `modalities` 设为 `["text", "audio"]`：

```python
# 在模型特定的 Parameters 中设置 voice
from agentscope.model._openai_chat._model import OpenAIChatModel

model = OpenAIChatModel(
    credential=credential,
    model="gpt-audio-mini",
    parameters=OpenAIChatModel.Parameters(voice="alloy"),
)
```

产生的音频通过 `reply_stream()` 的 `DATA_BLOCK_*` 事件流式返回；模型生成的原始音频字节**不会**被写入 `state.context`（避免对话记忆膨胀）。可用语音取决于模型卡片（`ModelCard` 的 `voice.suggestions`）——无音频输出的模型会自动隐藏 `voice` 参数。

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

## Embedding 模型（v2.0.2+，v2.0.3 重构）

> ⚠️ 实验性：`embedding` 模块随多模态路线演进，类名和接口在后续版本仍可能调整。

独立的 `embedding` 模块，导出统一的基类与四个 provider 实现：

```python
from agentscope.embedding import (
    EmbeddingModelBase,            # 基类，泛型 Generic[InputT]
    EmbeddingModelCard,
    EmbeddingResponse,
    EmbeddingUsage,
    DashScopeEmbeddingModel,       # 文本 + 多模态（一个类按模型名路由）
    OpenAIEmbeddingModel,          # 文本
    GeminiEmbeddingModel,          # 文本 + 多模态（图像/视频/音频/PDF）
    OllamaEmbeddingModel,          # 文本
    EmbeddingCacheBase,
    FileEmbeddingCache,            # 文件级缓存
)
```

### 基类设计（v2.0.3 重构）

`EmbeddingModelBase[InputT]` 是泛型基类（对齐 `ChatModelBase` 的设计）：
- 文本模型绑定 `EmbeddingModelBase[str]`
- 多模态模型绑定 `EmbeddingModelBase[str | DataBlock]`

`__call__` 自动**分批并发**（按 `batch_size` 切片，`asyncio.gather` 并发，每批独立重试），
子类只需实现单批 `_call_api`。各 provider 内置可重试异常（`_get_retryable_exceptions`）。

### 构造（标准化参数）

所有 provider 的构造签名一致：前三个位置参数固定为 `credential` / `model` / `dimensions`，
其中 `dimensions`（输出向量维度）是**必填契约参数**（仅 OpenAI 多一个 `pass_dimensions`）：

```python
from agentscope.credential import OpenAICredential

credential = OpenAICredential(api_key="sk-xxx")
emb_model = OpenAIEmbeddingModel(
    credential=credential,
    model="text-embedding-3-small",
    dimensions=512,             # 必填：输出向量维度（契约参数）
    parameters=None,            # 各 provider 的 Parameters 子类；None 用默认
    pass_dimensions=True,       # 仅 OpenAI：是否把 dimensions 发给 API（默认 True）
    context_size=8191,          # 单条输入 token 上限（OpenAI 默认 8191，其余 8192）
    max_retries=3,              # 每批重试次数（默认 3）
    retry_delay=1.0,            # 重试间隔（默认 1.0s）
    # batch_size 由各 provider 内部定（如 OpenAI=2048, Gemini=100, Ollama=512）
)
```

> `dimensions=None` 仅为兼容旧配置（从 `parameters` 里读）而保留，新代码应显式传入。
> 不同 embedding 模型支持的维度集合不同，选不支持的值会被服务端拒绝——可用
> `credential.list_models()` 查 ModelCard 的 `supported_dimensions`。

调用（模型是 callable，返回 `EmbeddingResponse`）：

```python
response = await emb_model(inputs=["hello", "world"])
# response.embeddings -> list[Embedding]；response.usage -> EmbeddingUsage
# 多模态：DashScopeEmbeddingModel/GeminiEmbeddingModel 的 inputs 可含 DataBlock
```

### provider 差异

| Provider | 输入类型 | 多模态 | 备注 |
|---|---|---|---|
| `OpenAIEmbeddingModel` | `str` | ❌ | `pass_dimensions=False` 适配不支持该字段的 OpenAI 兼容后端（部分 vLLM/Ollama） |
| `DashScopeEmbeddingModel` | `str \| DataBlock` | ✅ | 按模型名前缀路由：`text-embedding-*`(文本) / `qwen*-vl-*`/`multimodal-*`/`tongyi-embedding-vision-*`(多模态) |
| `GeminiEmbeddingModel` | `str \| DataBlock` | ✅ | 图像/视频/音频/PDF，有每请求元素数限制 |
| `OllamaEmbeddingModel` | `str` | ❌ | 每次调用新建 client 以避免事件循环绑定问题 |

> DashScope 多模态可配 `embedding_cache=FileEmbeddingCache()` 启用文件缓存。
> 响应解析已兼容服务端省略 `index`、或在 `embedding` 为空时回退到 `dense_embedding` 的情况。

> 通过 Credential 获取实现类：`credential.get_embedding_model_class()`（未支持的 provider 返回 `None`）。

## TTS 模型（v2.0.2+）

> ⚠️ 实验性 / 积极开发中：`tts` 模块是官方 Voice Agent 路线（roadmap）的 Phase 1，
> 后续将扩展更多 TTS 提供商并走向多模态/实时阶段，API 可能随版本变动。

新增 `tts` 模块，支持普通 TTS 与实时（流式输入）TTS：

```python
from agentscope.tts import (
    TTSModelBase,
    TTSModelCard,
    TTSResponse,
    TTSUsage,
    OpenAITTSModel,                    # OpenAI TTS（tts-1/tts-1-hd/gpt-4o-mini-tts）
    DashScopeTTSModel,                 # 普通非实时
    DashScopeRealtimeTTSModel,         # Qwen3 实时流式输入
    DashScopeCosyVoiceTTSModel,        # CosyVoice 普通/实时（v2.0.4+）
)
```

通过 Credential 获取（TTS 返回**列表**，一个提供商可对应多个类）：

```python
from agentscope.credential import DashScopeCredential

credential = DashScopeCredential(api_key="ds-xxx")
tts_classes = credential.get_tts_model_classes()  # [DashScopeTTSModel, DashScopeRealtimeTTSModel, ...]
cards = credential.list_tts_models()

# 非实时：一次性合成
tts = DashScopeTTSModel(credential=credential, model="qwen3-tts-flash")
resp: TTSResponse = await tts.synthesize(text="你好")

# Qwen3 实时：流式输入，增量输出
async with DashScopeRealtimeTTSModel(credential=credential) as rt:
    await rt.push("第一段")
    chunk = await rt.push("第二段")   # 返回已合成的增量音频
    final = await rt.synthesize()      # 排空剩余音频

# CosyVoice：同一类支持普通与实时，是否实时由 parameters.realtime 控制
cosy = DashScopeCosyVoiceTTSModel(
    credential=credential,
    model="cosyvoice-v3-flash",
    cold_start_length=10,        # 可选：冷启动预热字数
    max_retries=3,               # 可选：WebSocket 断线重连次数
    retry_delay=5.0,             # 可选：重连间隔
    parameters=DashScopeCosyVoiceTTSModel.Parameters(
        voice="longanhuan",
        realtime=True,
    ),
)
async with cosy:
    await cosy.push("第一段")
    final = await cosy.synthesize()
```

### OpenAITTSModel（v2.0.4+）

基于 OpenAI TTS API 的非实时模型（`realtime=False`），支持 `tts-1` / `tts-1-hd` / `gpt-4o-mini-tts`：

```python
from agentscope.credential import OpenAICredential
from agentscope.tts import OpenAITTSModel

credential = OpenAICredential(api_key="sk-xxx")
tts = OpenAITTSModel(
    credential=credential,
    model="gpt-4o-mini-tts",   # 或 tts-1 / tts-1-hd
    parameters=OpenAITTSModel.Parameters(
        voice="alloy",                      # 音色：alloy/echo/fable/onyx/nova/shimmer 等
        response_format="mp3",              # mp3/opus/aac/flac/wav/pcm
        instructions=None,                  # 仅 gpt-4o-mini-tts 支持：自然语言风格指令
    ),
    stream=True,               # stream=True 时 synthesize() 返回增量流
)
# stream=True：synthesize() 返回 async generator，逐块 yield TTSResponse（每块含增量音频，末块 is_last=True）
# stream=False：synthesize() 返回单个聚合的 TTSResponse
async for resp in await tts.synthesize(text="你好"):
    ...
```

> 各提供商的 TTS 能力：OpenAI 提供 `OpenAITTSModel`（非实时），DashScope 提供
> `DashScopeTTSModel`（普通）/`DashScopeRealtimeTTSModel`（Qwen3 实时）/`DashScopeCosyVoiceTTSModel`
> （CosyVoice 普通+实时）；其余提供商（Anthropic/Gemini/Ollama 等）当前不提供 TTS，`get_tts_model_classes()` 返回空列表。

TTS 的核心 API：
- `synthesize(text)` — 阻塞直到整句合成完成
- `push(text)` — （仅 realtime）追加文本，返回已就绪的增量音频
- `connect()` / `close()` — （仅 realtime）连接生命周期，也可用 `async with`
- `realtime: bool` — 是否支持流式输入模式

> 当前本地源码导出的是 `DashScopeCosyVoiceTTSModel`，不是旧名
> `DashScopeCosyVoiceRealtimeTTSModel`。它支持 `cosyvoice-v3-plus` / `cosyvoice-v3-flash` 等模型，
> 构造时可用 `stream=True/False` 控制 `synthesize()` 返回增量流还是聚合结果，用
> `Parameters(realtime=True)` 开启流式输入。CosyVoice 由服务端按句切分，`push` 在文本不足以成句时
> 可能返回空，需调 `synthesize()` 强制输出剩余文本。该模型仍处于开发中，行为细节可能调整。

> 通常无需手动调用 TTS 模型，配合 `TTSMiddleware` 即可把智能体回复自动转语音（见 middleware 文档）。
