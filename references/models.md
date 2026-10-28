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
    VolcengineCredential,    # v2.0.8+ 火山引擎 Ark
    MiniMaxCredential,       # v2.0.9+ MiniMax（Anthropic 兼容端点）
    TypeSafeCredential,      # v2.0.8+ TypeSafe Jev 分类器（无对话模型）
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

Credential 不仅管理对话模型，还统一暴露 Embedding、TTS 与 ASR（语音转写）能力：

```python
credential = DashScopeCredential(api_key="ds-xxx")

# 对话模型
chat_cls = credential.get_chat_model_class()        # -> DashScopeChatModel
chat_models = credential.list_models()              # 对话模型卡片

# 全部对话模型类（v2.0.12+，一个 credential 可能暴露多个 API 面）
chat_classes = credential.get_chat_model_classes()  # -> list[ChatModelBase 子类]
# OpenAICredential -> [OpenAIChatModel, OpenAIResponseModel]
# （Chat Completions 与 Responses API 两个类）；其余 credential 默认返回单元素列表

# Embedding 模型（v2.0.2+）
emb_cls = credential.get_embedding_model_class()    # -> DashScopeEmbeddingModel | None
# 注意：返回单个类或 None，未支持的提供商返回 None

# TTS 模型（v2.0.2+）
tts_classes = credential.get_tts_model_classes()    # -> list[TTSModelBase 子类]
tts_cards = credential.list_tts_models()            # -> list[TTSModelCard]
# 注意：TTS 返回的是「列表」（一个提供商可能有多个模型，如普通/实时两个）

# Realtime 实时语音模型（v2.0.10+）
rt_classes = credential.get_realtime_model_classes()   # -> list[RealtimeModelBase 子类]
rt_cards = credential.list_realtime_models()           # -> list[RealtimeModelCard]

# ASR 语音转写模型（v2.0.11+）
asr_classes = credential.get_asr_model_classes()    # -> list[ASRModelBase 子类]
asr_cards = credential.list_asr_models()            # -> list[ASRModelCard]
```

> 各提供商支持的能力不同：DashScope 同时支持对话/Embedding/TTS；OpenAI/Gemini/Ollama 支持 Embedding（`get_embedding_model_class()`）；OpenAI 另提供 ASR（`get_asr_model_classes()`，v2.0.11+）；多数提供商的 TTS 列表为空。
>
> v2.0.12+ 起 `list_models()` 合并 `get_chat_model_classes()` 返回的**所有**类的模型卡，
> 每张 `ModelCard` 新增 `model_class` 字段标识所属模型类（如 `openai_chat` / `openai_response`），
> 据此可区分同名模型走哪个 API（OpenAI 的 Responses 卡 label 带 "Responses API" 后缀）。

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
    VolcengineChatModel,     # v2.0.8+ 火山引擎 Ark（doubao）
    MiniMaxChatModel,        # v2.0.9+ MiniMax（Anthropic 兼容协议）
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

> ℹ️ 模型清单由各 provider 目录下的 YAML 文件动态加载（`model/_<provider>/_models/*.yaml`），
> 没有静态 Python 常量。新增模型通常只加 YAML，不改代码。例如 v2.0.5 新增了
> `kimi-k3`（Moonshot，1048576 上下文，多模态）、DashScope 的 `qwen3.7-plus`/`deepseek-v4-pro`/`glm-5.2`，
> v2.0.6 全量刷新了各 provider 的 model card（新增 Claude opus-5/sonnet-5/fable/opus-4-8、Gemini 3.5/3.6-flash、
> GPT-5.6 系列、Kimi K2.7-code 等），v2.0.8+ Ollama 新增 `qwen3:8b`/`llama3.2:3b`/`phi4-mini`
> 三个本地模型卡，后又新增 14 个模型卡：Claude Fable 5.1（Anthropic）、`gpt-6-astra`
> （OpenAI chat/response 双格式）、`glm-5.3` / `deepseek-v4.1-flash` / `qwen3.8-flash` /
> `qwen3.8-omni-flash`（DashScope）、`deepseek-flash`（DeepSeek）、`gemini-3.7-flash` /
> `gemini-3.8-flash`（Gemini）、`grok-4.6`（xAI）、豆包 `doubao-seed-2-1-pro-260915` /
> `doubao-seed-2-1-turbo-260628`（Volcengine）、`gemma4` / `qwen3.8-27b`（Ollama），
> v2.0.9 新增 MiniMax 提供商（`MiniMax-M3` / `MiniMax-M2.7` / `MiniMax-M2.7-highspeed`
> 三张模型卡），Kimi K2.7 Code 系列模型卡启用图片与视频输入，
> 均可通过 `credential.list_models()` 查到。`kimi-k3` 在 `MoonshotChatModel` 中额外支持
> `reasoning_effort: "low"|"high"|"max"` 参数。

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

> ℹ️ `OpenAIChatModel.Parameters.max_tokens` 字段名不变（v2.0.4.post1+），但发送到 OpenAI API 时
> 会被映射为请求体里的 `max_completion_tokens` 键（对齐 OpenAI 新版 Chat Completions API）。

### Anthropic Chat 模型专属参数

`AnthropicChatModel` 支持三组 Claude 推理控制参数（v2.0.6+），用于精细调节 extended thinking 与推理强度：

```python
from agentscope.model import AnthropicChatModel

model = AnthropicChatModel(
    credential=credential,
    model="claude-opus-4-7",
    parameters=AnthropicChatModel.Parameters(
        thinking_mode="adaptive",       # adaptive | enabled | disabled（v2.0.6+）
        thinking_display="summarized",  # summarized | omitted（v2.0.6+）
        reasoning_effort="high",        # low | medium | high | xhigh | max（v2.0.6+）
    ),
)
```

- `thinking_mode` — 如何思考。`adaptive` 让模型自行决定深浅，是 Claude Opus 4.7 及以上**唯一接受**的模式
  （这些模型拒绝旧的 budget-based `enabled` 模式）。优先级高于 `thinking_enable`；不设则回退到 `thinking_enable`。
- `thinking_display` — thinking 块是否带可读摘要。Opus 4.7+ 默认 `omitted`，设 `summarized` 才能看到推理过程。
  与 `thinking_mode="disabled"` 互斥（禁用时 API 不接受 display）。
- `reasoning_effort` — 整个回复（不止 thinking，含工具调用与解释）的 token 投入。作为 Anthropic 的
  `output_config.effort` 发送，API 默认 `high`。各模型支持的等级不同，Claude Sonnet 4.5 / Haiku 4.5 **不接受**此参数。

> ℹ️ 旧参数 `thinking_enable` + `thinking_budget` 仍保留（budget-based `enabled` 模式）。新的 `thinking_mode`
> 是推荐用法，自动按模式组装 `thinking` 字段（`adaptive`/`disabled` 不带 budget，`enabled` 才带 budget_tokens）。

### Volcengine Chat 模型（v2.0.8+）

`VolcengineChatModel` 接入火山引擎 Ark（doubao 豆包系列），走 OpenAI 兼容协议，
支持 `client_kwargs` 透传与思考模式参数：

```python
from agentscope.credential import VolcengineCredential
from agentscope.model import VolcengineChatModel

credential = VolcengineCredential(api_key="xxx")
model = VolcengineChatModel(
    credential=credential,
    model="doubao-seed-2-1-pro-260628",   # 带版本号的 Ark 模型 ID
    parameters=VolcengineChatModel.Parameters(
        thinking_enable=True,             # 思考模式；不设用 Ark 默认 auto
        reasoning_effort="high",          # minimal | low | medium | high（minimal 关闭推理）
    ),
    client_kwargs={"timeout": 60.0},      # 转发给 openai.AsyncClient
)
```

配套的 `VolcengineChatFormatter` / `VolcengineMultiAgentFormatter` 由模型内部默认使用，无需手动指定。

### MiniMax Chat 模型（v2.0.9+）

`MiniMaxChatModel` 接入 MiniMax 的 **Anthropic 兼容端点**（继承 `AnthropicChatModel`），
支持自适应思考参数：

```python
from agentscope.credential import MiniMaxCredential
from agentscope.model import MiniMaxChatModel

credential = MiniMaxCredential(
    api_key="xxx",
    # base_url 默认国际站 https://api.minimax.io/anthropic，
    # 中国大陆改为 https://api.minimax.cn/anthropic
)
model = MiniMaxChatModel(
    credential=credential,
    model="MiniMax-M3",                    # 默认推荐（1M 上下文，多模态输入）
    parameters=MiniMaxChatModel.Parameters(
        thinking_enable=True,              # 自适应思考
    ),
)
```

- 模型卡：`MiniMax-M3`（1000000 上下文 / 131072 输出，输入支持图片与视频）、
  `MiniMax-M2.7` / `MiniMax-M2.7-highspeed`（204800 上下文 / 65536 输出，纯文本）。
- `MiniMaxCredential.get_chat_model_class()` 返回 `MiniMaxChatModel`；
  配套 `MiniMaxChatFormatter` 由模型内部默认使用。

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

> v2.0.10+：DashScope Omni 模型流式返回时，`delta.audio.transcript` 中的 spoken 文本会同步追加到
> 同一文本块（agent 同时收到文本与音频，对齐 OpenAI Chat 模型行为）。

> v2.0.10+：流式音频各 delta 以占位 WAV 头（chunk 尺寸 0xFFFFFFFF）开头，累加产出的最终块会把
> RIFF/data chunk 尺寸重写为真实值——流式与非流式一样，落盘即为可直接播放的合法 WAV。

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
多批次合并后的 `EmbeddingResponse` 统计语义（v2.0.11+ 修正）：`usage.time` 取**最大批次耗时**
（批次并发执行，此前取总和会高估墙钟时间）；`source` 仅当**所有**批次都命中缓存时为 `"cache"`，
否则为 `"api"`（此前多批次合并恒为 `"api"`，即使全部命中缓存）。

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
| `GeminiEmbeddingModel` | `str \| DataBlock` | ✅ | 图像/视频/音频/PDF，有每请求元素数限制；`parameters` 可配 `task_type` 等文本参数（v2.0.11+ 起真正传给 API，此前被忽略；per-call kwargs 优先）；per-call `output_dimensionality` 可覆盖构造时 `dimensions`（v2.0.11+，此前传入会因关键字冲突报错） |
| `OllamaEmbeddingModel` | `str` | ❌ | 构造时创建一次 `AsyncClient`（host 取自 credential 或 `OLLAMA_HOST`）；`embedding_cache` 按 host 隔离（v2.0.10+，解析后的 endpoint 计入缓存 key，不同端点同名模型不互相命中）；`usage.tokens` 保留服务端 `prompt_eval_count`（v2.0.11+，此前恒为 None） |

> DashScope 多模态可配 `embedding_cache=FileEmbeddingCache()` 启用文件缓存；v2.0.8+ 起
> 单个向量集文件超过 `max_cache_size` 时直接不入缓存（记 warning），不再因按最旧驱逐而清空整个缓存；
> v2.0.10 起 `max_file_number=0` 真正生效（写后不保留任何缓存文件，`None` 才是不限），且新文件
> 因超过 `max_cache_size` 被拒后，文件数上限仍独立执行。
> 响应解析已兼容服务端省略 `index`、或在 `embedding` 为空时回退到 `dense_embedding` 的情况。
> v2.0.8+：是否多模态以公开属性 `supports_multimodal` 暴露（DashScope/Gemini 均支持），
> 可在运行时据此决定输入走文本还是多模态路径。
> v2.0.11+：多模态路径同样把构造时的 `dimensions` 作为 `dimension` 传给 API
> （此前配置值被忽略、实际用服务端默认维度；文本路径一直正常传递）。

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
    GeminiTTSModel,                    # Gemini TTS（gemini-2.5-flash/pro-preview-tts，v2.0.5+）
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
# v2.0.10+：synthesize(text, **kwargs) 的调用级参数可覆盖 model/text/voice 等请求参数
# （api_key 与 stream 由内部强制设置，不可经 kwargs 覆盖）
resp = await tts.synthesize(text="你好", voice="cherry")

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

# v2.0.10+：synthesize(text, **kwargs) 的调用级参数可覆盖 Parameters 默认值
# （如 voice/response_format/instructions），返回音频的 media_type 按覆盖后实际生效的 response_format 标注
await tts.synthesize(text="你好", instructions="用兴奋的语气说")
```

### GeminiTTSModel（v2.0.5+）

基于 Google Gemini TTS API 的非实时模型（`realtime=False`），复用已有的 `GeminiCredential`
（未新增独立 credential；`GeminiCredential.get_tts_model_classes()` 现返回 `[GeminiTTSModel]`）。
依赖 `model-gemini` extra（`google-genai`）。

```python
from agentscope.credential import GeminiCredential
from agentscope.tts import GeminiTTSModel

credential = GeminiCredential(api_key="xxx")
tts = GeminiTTSModel(
    credential=credential,
    model="gemini-2.5-flash-preview-tts",   # 或 gemini-2.5-pro-preview-tts
    parameters=GeminiTTSModel.Parameters(
        voice="Kore",         # Gemini 预置音色名
    ),
    stream=False,
)
resp = await tts.synthesize(text="Hello")
```

> 各提供商的 TTS 能力：OpenAI 提供 `OpenAITTSModel`（非实时），DashScope 提供
> `DashScopeTTSModel`（普通）/`DashScopeRealtimeTTSModel`（Qwen3 实时）/`DashScopeCosyVoiceTTSModel`
> （CosyVoice 普通+实时），Gemini 提供 `GeminiTTSModel`（v2.0.5+，非实时）；
> 其余提供商（Anthropic/Ollama 等）当前不提供 TTS，`get_tts_model_classes()` 返回空列表。

TTS 的核心 API：
- `synthesize(text)` — 阻塞直到整句合成完成
- `push(text)` — （仅 realtime）追加文本，返回已就绪的增量音频
- `connect()` / `close()` — （仅 realtime）连接生命周期，也可用 `async with`
- `realtime: bool` — 是否支持流式输入模式

> v2.0.10+：`TTSModelBase` 是真正的抽象基类（继承 `abc.ABC`），自定义 TTS 模型未实现
> 抽象方法时会在实例化阶段直接报错，而非留到调用时才失败。

> v2.0.10+：DashScopeTTSModel 合成出错（status_code 非 200）时显式抛 `RuntimeError`，
> 流式/非流式均不再静默吞错返回空音频。同期起 `synthesize(**kwargs)` 支持调用级参数
> 覆盖 model/text/voice 等（api_key 与 stream 内部强制设置，覆盖无效）。

> 当前本地源码导出的是 `DashScopeCosyVoiceTTSModel`，不是旧名
> `DashScopeCosyVoiceRealtimeTTSModel`。它支持 `cosyvoice-v3-plus` / `cosyvoice-v3-flash` 等模型，
> 构造时可用 `stream=True/False` 控制 `synthesize()` 返回增量流还是聚合结果，用
> `Parameters(realtime=True)` 开启流式输入。CosyVoice 由服务端按句切分，`push` 在文本不足以成句时
> 可能返回空，需调 `synthesize()` 强制输出剩余文本。该模型仍处于开发中，行为细节可能调整。

> 通常无需手动调用 TTS 模型，配合 `TTSMiddleware` 即可把智能体回复自动转语音（见 middleware 文档）。

## ASR 语音转写模型（v2.0.11+）

> ⚠️ 实验性 / 积极开发中：`asr` 模块属于官方 Voice Agent 路线，当前只有 OpenAI 批式实现，
> API 可能随版本变动。

`agentscope.asr` 提供语音转文字（transcription）的统一接口，与 TTS 镜像对称：
`ASRModelBase` 同时定义**批式**（一次性转写完整音频）与**实时**（增量推送音频流）两种协议，
由实现的 `realtime` 属性区分：

```python
from agentscope.asr import (
    ASRModelBase,      # 抽象基类（批式 + 实时双协议）
    ASRModelCard,      # 模型卡（name/label/input_types/realtime/parameter_schema）
    ASRResponse,       # 转写响应
    OpenAIASRModel,    # OpenAI Audio Transcriptions API（批式）
)
```

### 批式转写（OpenAIASRModel）

```python
from agentscope.credential import OpenAICredential
from agentscope.asr import OpenAIASRModel

credential = OpenAICredential(api_key="sk-xxx")
asr = OpenAIASRModel(
    credential=credential,
    model="gpt-4o-mini-transcribe",   # 默认；另带 gpt-4o-transcribe 模型卡
    parameters=OpenAIASRModel.Parameters(
        language="zh",     # 可选：ISO-639-1 语言代码
        prompt=None,       # 可选：转写引导（术语/风格提示）
    ),
)

with open("meeting.wav", "rb") as f:
    audio = f.read()

resp = await asr.transcribe(audio, filename="meeting.wav")
resp.content        # TextBlock(text=...)；转写文本
resp.is_last        # 批式结果恒为 True
```

- `filename` 仅用于 multipart API 识别内存音频的格式（扩展名），默认 `"audio.wav"`。
- per-call `**kwargs` 与构造 `parameters` 均透传给 Transcriptions API，优先级
  kwargs > `parameters` > 内置默认（v2.0.11+ 起可覆盖 `response_format` 等
  内置默认——此前传入会因关键字重复直接报错；非 JSON 响应格式返回纯文本也能解析）。
- 内置模型卡：`gpt-4o-mini-transcribe` / `gpt-4o-transcribe`，输入
  flac / mpeg / mp4 / ogg / wav / webm。

### 实时协议（自定义实现）

基类为实时实现预留了流式协议（当前无内置实现）：`async with` 进入时自动 `connect()`、
退出时 `close()`；每个音频块调 `push(audio)` 返回**已就绪的增量文本**（`ASRResponse`，
无新文本时 `content=None`）；最后 `transcribe(None)` 收尾（可再带最后一个音频块），
最终响应 `is_last=True`：

```python
async with MyRealtimeASR(credential=..., model=...) as asr:
    for chunk in audio_stream:
        partial = await asr.push(chunk)    # 增量结果，content 可能为 None
        ...
    final = await asr.transcribe()         # audio=None 收尾，is_last=True
```

### 自定义 ASR 模型

继承 `ASRModelBase` 并实现 `transcribe()`（抽象方法，必须）；实时实现还需覆盖
`connect()` / `close()` / `push()` 并置 `realtime = True`。`Parameters`（pydantic BaseModel）
声明 provider 特有参数，`list_models(custom_yaml_dir=None)` 从实现类旁的 `_models/*.yaml`
加载模型卡（`realtime` 与实现类不一致的卡被过滤，坏卡告警跳过）。

ASR 的核心 API：
- `transcribe(audio, filename)` — 批式整段转写 / 实时收尾（最终结果 `is_last=True`）
- `push(audio)` — （仅 realtime）推送音频块，返回已就绪的增量 `ASRResponse`
- `connect()` / `close()` — （仅 realtime）连接生命周期，也可用 `async with`
- `realtime: bool` — 是否支持流式输入模式

> 各提供商的 ASR 能力：OpenAI 提供 `OpenAIASRModel`（批式）；其余提供商
> `get_asr_model_classes()` 当前返回空列表。注意 RealtimeAgent 场景的输入转写
> （如 DashScope 的 `qwen3-asr-flash-realtime`）走 `agentscope.realtime` 模块的
> 实时模型，与此批式 `asr` 模块是两条独立路径。

## 分类器模型 Classifier（v2.0.8+）

独立的 `classifier` 模块，用于**类型化概率判定**：对一段输入（文本或 JSON）一次性回答多个
问题，返回概率化答案（而非生成文本）。两路实现：TypeSafe 的 Jev System One API（v2.0.8+）
与 OpenAI Decisions API（v2.0.10+）：

```python
from agentscope.classifier import (
    ClassifierModelBase,     # 基类
    JevClassifierModel,      # TypeSafe Jev 实现
    OpenAIClassifierModel,   # v2.0.10+ OpenAI Decisions API 实现
    BinaryQuestion,          # 二分类问题 → 概率
    ChoiceQuestion,          # 多选一问题 → 选项 + 概率分布
    ScoreQuestion,           # 有序评分标准问题 → 期望得分 + 概率分布
    BinaryCriteria,
    ClassifierResponse, ClassifierUsage,
    RefusalAnswer,           # v2.0.10+ 单题拒答（不影响其他答案）
)
```

> 需额外安装：`pip install "agentscope[classifier-jev]"`（`typesafe-sdk>=0.7`）。

### 调用方式

```python
from agentscope.credential import TypeSafeCredential
from agentscope.classifier import JevClassifierModel, ChoiceQuestion, ScoreQuestion

credential = TypeSafeCredential(api_key="ts-xxx")
classifier = JevClassifierModel(
    credential=credential,
    model="jev-latest",     # 默认
    timeout=30.0, max_retries=2, retry_delay=0.5,
)

response = await classifier(
    state="这段共享输入文本/JSON",          # 所有问题共享同一份输入
    questions={
        "topic": ChoiceQuestion(            # 命名问题，一次可问多个
            criteria={"tech": "技术问题", "chat": "闲聊"},
        ),
        "severity": ScoreQuestion(
            criteria=["无关紧要", "一般", "紧急"],   # 从 0 分起的有序标准
        ),
    },
)
response.content["topic"]    # ChoiceAnswer: choice/confidence/probabilities
response.content["severity"] # ScoreAnswer: score/confidence/legend/probabilities
response.usage               # ClassifierUsage: time/input_tokens/output_tokens
```

### 问题与答案类型

| 问题类型 | 判别字段 | 答案类型 | 答案字段 |
|---|---|---|---|
| `BinaryQuestion` | `type="binary"` | `BinaryAnswer` | `probability`（正类概率 0–1） |
| `ChoiceQuestion` | `type="choice"` | `ChoiceAnswer` | `choice` / `confidence` / `probabilities`（各选项概率） |
| `ScoreQuestion` | `type="score"` | `ScoreAnswer` | `score`（概率加权期望分）/ `confidence` / `legend` / `probabilities`（各整数分位概率） |
| —（provider 拒答） | — | `RefusalAnswer`（v2.0.10+） | 无字段；仅拒答的那一题返回它，同请求其他答案照常 |

- 问题都可选 `instructions`（要判定什么）；`BinaryQuestion.criteria` 用
  `BinaryCriteria(true=..., false=...)` 描述两个结果的含义，`ChoiceQuestion.criteria`
  是 `{选项名: 描述}`，`ScoreQuestion.criteria` 是从 0 分起的有序描述列表。
- 调用级 `model=` 可覆盖 `JevClassifierModel` 的实例默认模型（v2.0.12+ 真正到达
  Jev system_one 调用，仅本次调用生效、不会被记忆到后续调用）。
- `ClassifierResponse.model` 返回实际使用的模型名，`metadata` 携带 provider 无关的附加信息。
- `TypeSafeCredential`（api_key + 可选 base_url）**只服务分类器**，
  `get_chat_model_class()` 会抛 `NotImplementedError`。

### OpenAIClassifierModel — OpenAI Decisions API（v2.0.10+）

```python
from agentscope.credential import OpenAICredential
from agentscope.classifier import OpenAIClassifierModel, BinaryQuestion

classifier = OpenAIClassifierModel(
    credential=OpenAICredential(api_key="sk-xxx"),
    model="gpt-6-luna",     # 默认
    timeout=30.0, max_retries=2,
)

# state 额外支持 list[TextBlock | DataBlock]（图片须 Base64Source，URL 源不支持）
response = await classifier(
    state="共享输入文本 / dict（序列化为 JSON）/ 文本+图片块列表",
    questions={"urgent": BinaryQuestion(criteria=BinaryCriteria(
        true="需要立即处理", false="可以等待",
    ))},
)
response.content["urgent"]  # BinaryAnswer 或 RefusalAnswer
```

- 需 `openai>=3.26.0`（低版本构造时抛 `ImportError` 指明升级命令）。
- 问题映射：Binary → `predicate`（criteria 的 true/false 描述拼进 instructions）、
  Choice → `choice`、Score → `score`；`instructions` 缺省用问题名。
- `**kwargs` 透传 `client.decisions.create`（`safety_identifier` / `extra_headers` 等）；
  v2.0.11+ 起 `model` / `input` 也可经 kwargs 覆盖实例默认值（仅本次调用生效）。

### 与 ModelRouterMiddleware 配合

分类器最常见的用法是给 `ModelRouterMiddleware` 当路由模型——按用户输入为每条回复
挑选合适的 chat model，见 [middleware-workspace.md](middleware-workspace.md)。
