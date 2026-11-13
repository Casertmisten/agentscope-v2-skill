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

# 语音复刻 client（v2.0.12+）
clone_cls = credential.get_voice_cloning_client_class()  # -> Type[VoiceCloningClientBase] | None
# OpenAI -> OpenAIVoiceCloningClient；DashScope -> DashScopeVoiceCloningClient；其余 None
```

> 各提供商支持的能力不同：DashScope 同时支持对话/Embedding/TTS/ASR（ASR 为 v2.0.12+）；OpenAI/Gemini/Ollama 支持 Embedding（`get_embedding_model_class()`）；OpenAI 另提供 ASR（`get_asr_model_classes()`，v2.0.11+）；OpenAI 与 DashScope 提供语音复刻 client（v2.0.12+）；多数提供商的 TTS 列表为空。
>
> v2.0.12+ 起 `list_models()` 合并 `get_chat_model_classes()` 返回的**所有**类的模型卡，
> 每张 `ModelCard` 新增 `model_class` 字段标识所属模型类（如 `openai_chat` / `openai_response`），
> 据此可区分同名模型走哪个 API（OpenAI 的 Responses 卡 label 带 "Responses API" 后缀）。

### CredentialFactory

用于从 dict 反序列化 credential（配合数据库存储）：

```python
from agentscope.credential import CredentialFactory

credential = CredentialFactory.from_dict({"type": "openai_credential", "api_key": "sk-xxx"})
```

- 判别字段 `type` 取各 credential 类的 `type` 字面量（如 `openai_credential` / `dashscope_credential`）；
  内置类均已预注册，自定义类用 `CredentialFactory.register_credential(cls)` 注册，
  `get_credential_class(type)` / `list_schemas()` 查询。
- v2.0.12+ `TypeSafeCredential`（`type="typesafe_credential"`）也注册进工厂（服务化可保存该凭据），
  其 `list_models()` 返回 `[]`（此前抛 `NotImplementedError`），`get_chat_model_class()` 仍抛。

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
    raise_on_refusal=False,             # v2.0.12+ 拒绝信号 → ContentBlockedError
    raise_on_content_filter=False,      # v2.0.12+ 内容过滤 → ContentBlockedError
)
```

### 内容拦截错误归一化 ContentBlockedError（v2.0.12+，opt-in）

模型构造参数 `raise_on_refusal` / `raise_on_content_filter`（默认均 `False`，完全向后兼容）
开启后，provider 的拒绝信号 / 内容过滤异常被归一为统一异常
`agentscope.exception.ContentBlockedError`（`DeveloperOrientedException` 子类），带结构化字段：
`phase`（`ContentBlockedPhase.INPUT` / `OUTPUT` / `UNKNOWN`）、`provider`、`original_code`
（provider 原始错误码/原因，保留大小写）、`request_id`、`partial_output`（流式是否已产出部分内容）、
`requires_review`（provider 要求人工审查，Agent 禁止自动重试/切 fallback，目前仅
OpenAI Responses 会置 True）。

```python
from agentscope.exception import ContentBlockedError, ContentBlockedPhase
from agentscope.model import OpenAIChatModel

model = OpenAIChatModel(
    credential=credential, model="gpt-5",
    raise_on_refusal=True, raise_on_content_filter=True,
)
try:
    res = await model(messages)
except ContentBlockedError as e:
    print(e.phase, e.provider, e.original_code, e.request_id, e.partial_output)
```

各适配器支持矩阵（未列出的开关不可用）：

| 适配器 | `raise_on_refusal` | `raise_on_content_filter` | 识别信号 |
|---|---|---|---|
| `OpenAIChatModel` | ✅ | ✅ | SDK code `content_filter`→INPUT；`finish_reason=CONTENT_FILTER`、非空 `refusal`→OUTPUT |
| `OpenAIResponseModel` | ✅ | ✅ | `incomplete_details.reason=CONTENT_FILTER`→OUTPUT；错误码 `BIO_POLICY`/`MISALIGNMENT_POLICY_VIOLATION`（置 `requires_review`）/`IMAGE_CONTENT_POLICY_VIOLATION`；refusal 内容/流式 refusal 事件 |
| `AnthropicChatModel` / `MiniMaxChatModel` | ✅ | ❌ | `stop_reason="refusal"`→OUTPUT（流式带 `partial_output`） |
| `GeminiChatModel` | ❌ | ✅ | `promptFeedback.blockReason`→INPUT；candidate `finishReason`（SAFETY/BLOCKLIST/PROHIBITED_CONTENT/SPII 等）→OUTPUT |
| `DashScopeChatModel` / `DeepSeekChatModel` / `MoonshotChatModel` | ❌ | ✅ | `finish_reason=CONTENT_FILTER`（DashScope 另识别 `DataInspectionFailed` 错误码） |
| `VolcengineChatModel` | ❌ | ✅ | 双档错误码表：`*SensitiveContentDetected*` 按输入/输出分 INPUT/OUTPUT |

Agent 侧行为：`ContentBlockedError` **不消耗模型 SDK 重试预算**；INPUT/UNKNOWN 阶段不再以
同一 payload 重试（记 warning 后 break），仅 OUTPUT 阶段且无部分输出时才在
`ModelConfig.max_retries` 内重试；`partial_output` 或 `requires_review` 直接上抛。
`_normalize_exception` 是 `ChatModelBase` 的可覆盖 hook（自定义适配器归一化 SDK 异常的入口）。

> ℹ️ 无关 opt-in 的默认行为变化：`OpenAIChatModel` 在 `message.content` / `delta.content` 为空且
> 存在非空 `refusal` 字符串时，把 refusal 文本填充为普通 `TextBlock`（此前静默丢弃，流式与非流式一致）。
> `OpenAIResponseModel` 非流式响应 HTTP 200 但 `status="failed"` 时抛 `RuntimeError`（带
> response id 与错误详情；v2.0.12+，此前会静默解析部分输出）。

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
> v2.0.12+：新增 `kimi-k3.1-pro` / `kimi-k3.1-pro-highspeed` 模型卡（1048576 上下文与输出，图片 +
> 视频输入）。K3 系列（`kimi-k3` / `kimi-k3.1-pro` / `kimi-k3.1-pro-highspeed`，按名精确匹配）**始终思考**
> （`thinking_enable` 被忽略、`thinking` 字段被剥离）；`temperature` / `top_p` / `n` / `presence_penalty` /
> `frequency_penalty` 无论来自 Parameters、调用级还是 `extra_body` 都被静默剥离；`max_tokens` 改名为
> `max_completion_tokens` 发送；指名工具的 `tool_choice` 改为只发该工具 + `required`。
> Gemini 新增 `gemini-4-argon` / `gemini-4-carbon` 模型卡（1M 上下文 / 65536 输出，`thinking_level`
> 替代 budget 思考，见下节）。
> `ModelCard` 有 `status`（`active` / `deprecated` / `sunset`）与 `deprecated_at` 字段，`list_models()`
> 不按状态过滤——v2.0.12+ `claude-sonnet-4-5` 标为 `deprecated`（`deprecated_at` 2026-11-30）。

### 思考开关 thinking_enable 三态（v2.0.12+）

全部 11 个 chat 适配器的 `Parameters.thinking_enable` 统一为 `bool | None = None`：**`None`（默认）
不发送任何思考参数、交给 provider 默认**；`True` / `False` 显式开 / 关：

| 适配器 | `True` | `False` | `None`（默认） |
|---|---|---|---|
| DeepSeek / Moonshot（非 K3） | `extra_body.thinking.type="enabled"` | `"disabled"` | 不发（此前默认发 disabled） |
| DashScope | `extra_body.enable_thinking=True` | `False` | 不发（此前默认发 False） |
| Ollama | `think=True` | `think=False` | 不发（此前默认发 False） |
| Gemini | `thinking_config={include_thoughts: True, thinking_budget: budget or 1024}` | `{include_thoughts: False, thinking_budget: 0}` | 不发（此前默认强制关闭）；⚠️ Gemini 4 Argon/Carbon 上仅控制思维摘要输出（见下节） |
| Anthropic | budget-based `enabled`（`thinking_mode` 优先级更高） | `thinking={"type": "disabled"}`（此前等同不发） | 不发 |
| MiniMax | `{"type": "adaptive"}` | `{"type": "disabled"}`（M3 接受、M3.1-Flash 拒绝） | 不发 |
| OpenAI Chat / Responses | 发 `reasoning_effort`（若设） | `reasoning_effort="none"`（Responses 为 `reasoning={"effort": "none"}`） | 发 `reasoning_effort`（若设） |
| xAI | 发 `reasoning_effort`（若设） | 调用时抛 `ValueError`（SDK 不支持关闭思考） | 发 `reasoning_effort`（若设） |
| Volcengine（v2.0.8 起即三态） | `extra_body.thinking.type="enabled"` | `"disabled"` | 不发（Ark 默认 auto） |

> ⚠️ 迁移：DeepSeek / DashScope / Ollama / Moonshot（非 K3）/ Gemini 此前默认**强制关闭**思考，
> 现在默认思考的模型会开始输出 ThinkingBlock、消耗更多 token——要保持旧行为须显式
> `thinking_enable=False`。OpenAI / xAI 的 `reasoning_effort` 不再要求同时 `thinking_enable=True`
> （单独设即生效）；对非推理模型或 OpenAI 兼容端点显式设 `False` 会发出 `reasoning_effort="none"`，
> 可能被服务端拒绝。gpt-6-luna / gpt-6-sol / gpt-6.1-sol 模型卡不再隐藏 `reasoning_effort`。
> 调用级覆盖（如 DeepSeek `extra_body={"thinking": ...}`、Ollama `think=`、Gemini `thinking_config=`）
> 只在直接调用 `model(...)` 时有效，Agent 内部调用不传这些参数，需用 `Parameters` 配置。

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

> ℹ️ v2.0.12+：`OpenAIChatFormatter` 发送的消息 `name` 自动规范化——每段连续的空白或 `<` `|` `\` `/` `>`
> 替换为一个 `_`（如 `"Research Assistant"` → `"Research_Assistant"`），空名省略 `name` 键（API 要求
> `^[^\s<|\\/>]+$`，否则 400）。只影响发给 API 的字段，`Msg.name` 本身与 tool 消息的工具名不变。

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
>
> ℹ️ v2.0.12+ 有效模式优先级：`thinking_mode` > `thinking_enable=False`（→ `disabled`，此前等同不发）>
> `thinking_enable=True`（→ `enabled`）> `None`（不发 thinking）。有效 `max_tokens` 取
> `extra_body["max_tokens"]` > 调用级 `max_tokens=` > `Parameters.max_tokens` > 默认 8192。`enabled` 模式
> 未设 `thinking_budget` 时 budget 取 `max(1024, 有效max_tokens // 2)`（此前忽略调用级 max_tokens），
> 有效 max_tokens ≤ 1024 时发请求前抛 `ValueError`；显式 `thinking_budget` 原样发送由 provider 校验，
> 仅当各层都没给 max_tokens 且 budget ≥ 8192 时自动把 max_tokens 设为 budget + 1024（显式上限不再被改写）。
> 调用级 `thinking=` 或 `extra_body["thinking"]` 存在时完全跳过自动构建，如
> `model(msgs, max_tokens=1000, thinking={"type": "disabled"})`。

### Gemini Chat 模型专属参数（v2.0.12+）

`gemini-4-argon` / `gemini-4-carbon` 使用 `thinking_level` 取代 budget 式思考：

```python
from agentscope.credential import GeminiCredential
from agentscope.model import GeminiChatModel

model = GeminiChatModel(
    credential=GeminiCredential(api_key="xxx"),
    model="gemini-4-argon",
    parameters=GeminiChatModel.Parameters(
        thinking_level="high",   # low | medium | high；不设默认 medium（wire 上大写）
        thinking_enable=True,    # Gemini 4 上仅控制思维摘要（summaries）输出，不控制推理本身
    ),
)
```

Gemini 4（按模型名 `gemini-4-argon` / `gemini-4-carbon` 精确匹配）的专属规则：

- `thinking_budget` 禁用——设置即 `ValueError`（"require thinking_level instead of thinking_budget"）；
- `temperature` / `top_p` / `top_k` 被静默丢弃并告警（模型不支持，参数不发）；
- `max_output_tokens > 65536` 抛 `ValueError`（不支持 long decode）；
- 构造时自动把 `client_kwargs["http_options"]` 的 `timeout` 默认设为 900_000ms（15 分钟，
  显式传 0 保留）。
- 反向限制：**非** Gemini 4 模型设置 `thinking_level` 抛 `ValueError`。

> ℹ️ 调用级 kwargs（`model(..., **kwargs)`）新增 camelCase 别名：`thinkingConfig→thinking_config`、
> `maxOutputTokens→max_output_tokens`、`topP→top_p`、`topK→top_k`；snake/camel 同时提供抛
> `ValueError`，per-call 值优先于 Parameters 默认。

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

配套的 `VolcengineChatFormatter` 由模型内部默认使用；多智能体场景的 `VolcengineMultiAgentFormatter`
需经 `formatter=` 显式传入。

v2.0.12+ 支持**音频输入**：`VolcengineChatFormatter` 默认 `input_types` 追加 `audio/mpeg`、`audio/mp3`、
`audio/wav`、`audio/aac`、`audio/mp4`，编码为 Ark 的 `{"type": "input_audio", "input_audio": {"format": ...}}`
（mpeg/mp3→`mp3`、wav→`wav`、aac→`aac`、mp4→`m4a`）：`Base64Source` 放原始 base64（不带 `data:` 前缀），
`file://` 本地文件读取后转 base64，http(s) URL 原样放进 `url` 字段透传。默认列表外的音频类型（如 ogg）
告警跳过；把 `input_types` 放宽到映射外的类型会抛 `TypeError`。内置模型卡中只有
`doubao-seed-2-1-lite-260915` 声明了音频输入（服务化按模型卡覆盖 formatter 的 `input_types`）；
示例见上游 `scripts/model_examples/volcengine_multimodal.py`。

### MiniMax Chat 模型（v2.0.9+）

`MiniMaxChatModel` 接入 MiniMax 的 **Anthropic 兼容端点**（继承 `AnthropicChatModel`），
支持自适应思考参数（v2.0.12+ `thinking_enable=False` 发 `disabled`，取决于模型是否接受；
`None` 默认不发）：

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
        thinking_enable=True,              # 自适应思考（False 发 disabled，None 默认不发）
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
> v2.0.12+：非流式下经 `extra_body` 覆盖 `audio.format` 时，返回 `DataBlock` 的 `media_type`
> 按**实际生效**的格式标注（优先级 per-call `extra_body` > 实例 `extra_body` > 直接 `audio`
> 参数；此前 mp3 会被错标为 `audio/wav`）。

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

> 失败时按 `forced`（强制 tool_choice）→ `auto` → `no_think`（关思考 + 强制 tool_choice，仅当适配器
> 实现了 `_get_disable_thinking_kwargs()`）→ `none` 逐级降级重试；v2.0.12+ Ollama 也参与
> `no_think` 一级（`think=False`），Moonshot K3 系列因始终思考而跳过该级。

### 相关类型

```python
from agentscope.model import (
    ChatResponse,       # 模型响应
    StructuredResponse, # 结构化输出响应
    ChatUsage,          # token 用量
    ModelCard,          # 模型卡片
)
```

> `ChatUsage.input_tokens` 是整个 prompt 的输入 token 数，**包含** `cache_input_tokens`（缓存读）与
> `cache_creation_input_tokens`（缓存写）。v2.0.12+ Anthropic（及继承它的 MiniMax）对齐此口径——
> `input_tokens = input + cache_creation + cache_read`，流式取 `message_delta` 的累计值；开启 prompt
> caching 时 `ModelCallEndEvent.input_tokens`、`Msg.usage`、tracing 与 `ReplyBudgetControlMiddleware`
> 的输入计费数值随之变大。

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
某批最终失败时框架先取消并等待其余批次收尾，再把原始异常抛给调用者（v2.0.12+，
此前兄弟批次任务在后台泄漏）；调用方取消同样先完成清理再抛出。
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
    parameters=None,            # EmbeddingModelBase.Parameters（extra="allow"，额外字段透传，如 DashScope 文本
                                # text_type/instruct、Gemini 文本 task_type）；None 用默认
    pass_dimensions=True,       # 仅 OpenAI：是否把 dimensions 发给 API（默认 True）
    embedding_cache=None,       # EmbeddingCacheBase（如 FileEmbeddingCache()），四个 provider 均支持
    context_size=8191,          # 单条输入 token 上限（OpenAI 默认 8191，其余 8192）
    max_retries=3,              # 每批重试次数（默认 3）
    retry_delay=1.0,            # 重试间隔（默认 1.0s）
    # batch_size 由各 provider 内部定（如 OpenAI=2048, Gemini=100, Ollama=512）；
    # 基类要求 >0，自定义 embedding 子类传 <=0 构造即抛 ValueError（v2.0.12+）
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

> 调用级参数（v2.0.12+ 各 provider 对齐）：`await emb_model(inputs, **kwargs)` 的 kwargs 优先于
> 构造 `parameters`，再优先于内置默认——OpenAI per-call `dimensions` 可覆盖实例值（此前被静默覆盖；
> `pass_dimensions=False` 时实例值不发，per-call 值照常透传）；DashScope 文本路径合并 `parameters`
> （此前被忽略）且 per-call `api_key` 可用（此前报关键字重复 TypeError）。DashScope/Gemini 的
> **多模态路径只合并 per-call kwargs、不合并 `parameters`**。调用参数计入缓存 key（`api_key` 除外，
> 换 key 仍可命中旧缓存）；OpenAI 的 `base_url` 同样计入缓存 key（v2.0.12+，官方 API 与
> vLLM/Ollama 兼容端点的同名模型不再互相命中对方缓存，旧缓存条目一次性失效）。

### provider 差异

| Provider | 输入类型 | 多模态 | 备注 |
|---|---|---|---|
| `OpenAIEmbeddingModel` | `str` | ❌ | `pass_dimensions=False` 适配不支持该字段的 OpenAI 兼容后端（部分 vLLM/Ollama） |
| `DashScopeEmbeddingModel` | `str \| DataBlock` | ✅ | 按模型名前缀路由：`text-embedding-*`(文本) / `qwen*-vl-*`/`multimodal-*`/`tongyi-embedding-vision-*`(多模态) |
| `GeminiEmbeddingModel` | `str \| DataBlock` | ✅ | 图像/视频/音频/PDF，有每请求元素数限制；`parameters` 可配 `task_type` 等文本参数（v2.0.11+ 起真正传给 API，此前被忽略；per-call kwargs 优先）；per-call `output_dimensionality` 可覆盖构造时 `dimensions`（v2.0.11+，此前传入会因关键字冲突报错） |
| `OllamaEmbeddingModel` | `str` | ❌ | 构造时创建一次 `AsyncClient`（host 取自 credential 或 `OLLAMA_HOST`）；`embedding_cache` 按 host 隔离（v2.0.10+，解析后的 endpoint 计入缓存 key，不同端点同名模型不互相命中）；`usage.tokens` 保留服务端 `prompt_eval_count`（v2.0.11+，此前恒为 None） |

> 四个 provider 均可配 `embedding_cache=FileEmbeddingCache()` 启用文件缓存（签名
> `FileEmbeddingCache(cache_dir="./.cache/embeddings", max_file_number=None, max_cache_size=None)`，
> `max_cache_size` 单位 MB；v2.0.12+ 负值构造即抛 `ValueError`，此前静默等同 0）；v2.0.8+ 起
> 单个向量集文件超过 `max_cache_size` 时直接不入缓存（记 warning），不再因按最旧驱逐而清空整个缓存；
> v2.0.10 起 `max_file_number=0` 真正生效（写后不保留任何缓存文件，`None` 才是不限），且新文件
> 因超过 `max_cache_size` 被拒后，文件数上限仍独立执行。v2.0.12+：Gemini 多模态路径同样读写缓存
> （此前完全不走缓存）；DashScope 多模态缓存 key 改用深拷贝（SDK 上传本地/base64 图片时就地改写
> `input`，导致写入与查询的 key 不一致、多模态缓存从不命中）；缓存文件名哈希对 dict 键序不敏感
> （`sort_keys=True`，语义相同的调用共享缓存，旧缓存文件因哈希规则变化一次性失配）。
> 响应解析：OpenAI 兼容服务端省略 `index`、或在 `embedding` 为空时回退到 `dense_embedding`；
> v2.0.12+ DashScope 文本按 `text_index`、多模态按 `index` 还原输入顺序（缺失时按原位置，
> 此前服务端乱序返回会让向量与输入错配）。
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
    VoiceCloningClientBase,            # v2.0.12+ 语音复刻 client 基类
    OpenAIVoiceCloningClient,          # v2.0.12+
    DashScopeVoiceCloningClient,       # v2.0.12+
    VoiceCloningError,                 # v2.0.12+ provider 侧复刻失败（RuntimeError 子类）
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
        voice="alloy",                      # 音色：alloy/echo/fable/onyx/nova/shimmer 等；v2.0.12+ 类型为
                                            # str | dict，克隆音色传 {"id": voice_id}（仅 gpt-4o-mini-tts）
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

v2.0.12+ 新增 `gemini-3.8-flash-tts` / `gemini-3.8-flash-lite-tts` 模型卡（结构化语音）：

- `Parameters.style`（默认 `""`）— 3.8 模型专属的**交付风格指令**（如 `"warm and friendly"`，
  不朗读出来），随请求以 `speech_metadata.style` 发送；非 3.8 模型设置非空 `style` 抛
  `ValueError`（"Structured style requires a Gemini 3.8 TTS model."）；
- `stream` 默认值由 `False` 改为 `True`（`synthesize()` 默认返回增量流，要聚合结果显式传
  `stream=False`）；流式仍要求 PCM（24kHz 单声道），对外统一封装为 `media_type="audio/wav"`
  的 `DataBlock`（3.8 非流式直接返回 WAV）；
- 3.8 模型请求体用 `speechConfig.voice_config: {"voice": ...}`（2.5 保持
  `{"prebuilt_voice_config": {"voice_name": ...}}`）；Developer API 默认走 `v1beta`
  （`client_kwargs["http_options"]["api_version"]` 可覆盖）。

> v2.0.12+：`TTSModelBase` 新增 classmethod 钩子 `validate_parameters(model, parameters)`，
> 构造时自动调用（默认无约束；子类可抛 `ValueError` 拒绝不支持的模型/参数组合，不得做 I/O）——
> 服务化保存 TTS 配置时同样先校验，非法组合当场报错。

> 各提供商的 TTS 能力：OpenAI 提供 `OpenAITTSModel`（非实时），DashScope 提供
> `DashScopeTTSModel`（普通）/`DashScopeRealtimeTTSModel`（Qwen3 实时）/`DashScopeCosyVoiceTTSModel`
> （CosyVoice 普通+实时），Gemini 提供 `GeminiTTSModel`（v2.0.5+，非实时；v2.0.12+ 多个音频候选只取
> 第一个——候选是互为替代的录音，此前被拼接在一起；同批次新增 3.8 TTS 结构化语音与 `style`）；
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
> v2.0.12+：同一类还承载 `cosyvoice-v3.5-plus/flash`（**无预置音色**，只能用复刻音色）与
> `qwen-audio-3.0/3.1-tts-flash/plus` 模型卡；实时 `push` 冷启动阶段发送失败返回空响应并保留缓冲
> （下次 push 整段重试），冷启动之后的发送失败**直接抛出原异常**（此前静默返回空、该段文本丢失），
> 抛出前回滚累计文本，调用方重试不会重复计数。

> v2.0.12+ DashScope 端点路由：`DashScopeCredential` 新增 `websocket_url`（`ws(s)://`，CosyVoice /
> Qwen Audio 走网关时显式指定，优先级最高）；未设时，非默认的官方端点（国际/美区等）与带标准 API
> 后缀的网关由 `base_url` 推导 `wss://…/api-ws/v1/inference`，默认端点与其他网关形态仍沿用 SDK
> 环境变量/全局配置（HTTP 与 WebSocket 路由分离的网关请两个 URL 都显式设置）。`DashScopeTTSModel` 在显式设置
> `base_url` 时把其 API 后缀换成 `/api/v1` 作为 `base_address`（此前 `base_url` 对它不生效），
> 调用级 `base_address=` 可覆盖。依赖要求 `dashscope>=1.20.0`。

> 通常无需手动调用 TTS 模型，配合 `TTSMiddleware` 即可把智能体回复自动转语音（见 middleware 文档）。

### 语音复刻 VoiceCloningClient（v2.0.12+）

用一段音频样本在 provider 侧注册自定义音色，返回 **provider 的 voice id**，再作为对应 TTS
模型的 `voice` 参数使用。`VoiceCloningClientBase` 是独立于 `TTSModelBase` 的类层次（自定义
TTS 不受影响），经 `credential.get_voice_cloning_client_class()` 获取：

```python
from agentscope.credential import OpenAICredential, DashScopeCredential
from agentscope.tts import (
    OpenAITTSModel, OpenAIVoiceCloningClient,
    DashScopeCosyVoiceTTSModel, DashScopeVoiceCloningClient,
)

# OpenAI：先上传说话人授权录音（须朗读 provider 规定的声明原文），再创建音色
client = OpenAIVoiceCloningClient(OpenAICredential(api_key="sk-xxx"))
consent_id = await client.create_consent(consent_wav, name="Alice consent",
                                         language="zh", filename="consent.wav")
voice_id = await client.create_voice(sample_mp3, filename="sample.mp3",
    parameters=client.Parameters(name="Alice", consent=consent_id))
tts = OpenAITTSModel(credential=..., model="gpt-4o-mini-tts",
    parameters=OpenAITTSModel.Parameters(voice={"id": voice_id}))   # 自定义音色传 dict

# DashScope：CosyVoice 只收公网 audio_url；Qwen-TTS / Qwen Audio 只收音频字节
ds = DashScopeVoiceCloningClient(DashScopeCredential(api_key="ds-xxx"))
voice_id = await ds.create_voice(audio_url="https://example.com/sample.mp3",
    parameters=ds.Parameters(target_model="cosyvoice-v3-plus"))
cosy = DashScopeCosyVoiceTTSModel(credential=..., model="cosyvoice-v3-plus",
    parameters=DashScopeCosyVoiceTTSModel.Parameters(voice=voice_id))
```

- 契约：`create_voice(audio=None, *, filename="voice.wav", audio_url=None, parameters=None) -> str`，
  `audio` 与 `audio_url` 二选一；输入非法抛 `ValueError`，provider 失败抛 `VoiceCloningError`
  （`RuntimeError` 子类）。创建非幂等，只发一次请求、不重试（超时 60s）。
- `OpenAIVoiceCloningClient.Parameters(name="agentscope", consent=必填)`；不支持 `audio_url`。
- `DashScopeVoiceCloningClient.Parameters(target_model=必填, prefix="agentscope", text=None)`：
  `target_model` 决定注册协议（`cosyvoice-*`/`qwen-audio-*` → `voice-enrollment`，其余 `qwen*` →
  `qwen-voice-enrollment`，`get_enrollment_model(model)` 可查）；`text` 仅用于 Qwen-TTS 注册。
  复刻出的音色须用**同一 credential、同一 target_model** 合成。
- `TTSModelCard.voice_cloning: bool`（YAML 键 `voice_cloning`）标记支持复刻的模型：
  `gpt-4o-mini-tts`、`cosyvoice-v3-flash/plus`、`cosyvoice-v3.5-flash/plus`（无预置音色，只能用
  复刻音色）、`qwen-audio-3.0/3.1-tts-flash/plus`。同期 OpenAI 预置音色调整：`tts-1`/`tts-1-hd`
  去掉 `ballad`/`verse`，`gpt-4o-mini-tts` 新增 `marin`/`cedar`。
- 服务化有 `/voice-profile` 音色档案端点，见 [middleware-workspace.md](middleware-workspace.md)。

## ASR 语音转写模型（v2.0.11+）

> ⚠️ 实验性 / 积极开发中：`asr` 模块属于官方 Voice Agent 路线，v2.0.12+ 起有 OpenAI 批式与
> DashScope 批式/实时实现，API 可能随版本变动。

`agentscope.asr` 提供语音转文字（transcription）的统一接口，与 TTS 镜像对称：
`ASRModelBase` 同时定义**批式**（一次性转写完整音频）与**实时**（增量推送音频流）两种协议，
由实现的 `realtime` 属性区分：

```python
from agentscope.asr import (
    ASRModelBase,      # 抽象基类（批式 + 实时双协议）
    ASRModelCard,      # 模型卡（name/label/input_types/output_types/realtime/parameter_schema/parameters_overrides）
    ASRResponse,       # 转写响应
    OpenAIASRModel,    # OpenAI Audio Transcriptions API（批式）
    DashScopeASRModel,           # DashScope 批式（v2.0.12+）
    DashScopeRealtimeASRModel,   # DashScope 实时（v2.0.12+）
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

### DashScope 转写（v2.0.12+）

`DashScopeCredential.get_asr_model_classes()` 返回 `[DashScopeASRModel, DashScopeRealtimeASRModel]`：

```python
from agentscope.credential import DashScopeCredential
from agentscope.asr import DashScopeASRModel, DashScopeRealtimeASRModel

credential = DashScopeCredential(api_key="ds-xxx")

# 批式：默认 qwen-audio-3.1-asr-flash（另有 qwen-audio-3.0-asr-flash）
asr = DashScopeASRModel(
    credential=credential,
    parameters=DashScopeASRModel.Parameters(
        language_hints=["zh"],              # 最多 4 个
        keep_dialect=False,
        speaker_diarization_enabled=False,
        vocabulary=None,                    # dict[str, int] 热词
    ),
)
resp = await asr.transcribe(audio_bytes, filename="meeting.mp3")   # 按扩展名定格式

# 实时：默认 qwen-audio-3.1-asr-flash-streaming，输入 PCM16（需 agentscope[realtime] 的 websockets）
rt = DashScopeRealtimeASRModel(
    credential=credential,
    parameters=DashScopeRealtimeASRModel.Parameters(sample_rate=16000, language_hints=["zh"]),
)
async with rt:
    partial = await rt.push(pcm16_chunk)   # ⚠️ 返回累计全文（已定稿句 + 当前句），不是增量
    final = await rt.transcribe()          # 收尾，is_last=True
```

- 批式支持 aac/amr/flac/m4a/mp3/mp4/mpeg/ogg/opus/wav/webm，空音频或扩展名不支持抛 `ValueError`；
  参数优先级 kwargs > `parameters`；`qwen-audio-3.0-asr-flash` 自动去掉它不支持的
  `keep_dialect` / `speaker_diarization_enabled`；API 非 200 或无文本抛 `RuntimeError`。
- 实时 `Parameters` 另有 `vad_model`（`near_meeting_16k` / `far_field_meeting_16k`，默认后者）、
  `vocabulary_id`、`semantic_punctuation_enabled=True`；连接 `wss://{base_url 主机}/api-ws/v1/inference`；
  启动/收尾超时、provider task-failed、未连接均抛 `RuntimeError`。
- 服务化有浏览器语音输入端点 `/asr/*`，见 [middleware-workspace.md](middleware-workspace.md)。

### 实时协议（自定义实现）

基类为实时实现预留了流式协议（内置实现见上文 `DashScopeRealtimeASRModel`）：`async with` 进入时自动 `connect()`、
退出时 `close()`；每个音频块调 `push(audio)` 返回**已就绪的文本**（`ASRResponse`，
无新文本时 `content=None`；是增量还是累计全文由实现决定，DashScope 返回累计全文）；最后 `transcribe(None)` 收尾（可再带最后一个音频块），
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
加载模型卡（`realtime` 与实现类不一致的卡被过滤，坏卡告警跳过）。v2.0.12+ YAML 可写
`parameter_overrides`：`{参数名: {hidden: true}}`（或值为 null）从 `parameter_schema` 中删除该参数
（同步移出 `required`），其他键与原属性 schema 合并——用于按模型裁剪前端可见参数，结果存于
`ASRModelCard.parameters_overrides`。

ASR 的核心 API：
- `transcribe(audio, filename)` — 批式整段转写 / 实时收尾（最终结果 `is_last=True`）
- `push(audio)` — （仅 realtime）推送音频块，返回已就绪的增量 `ASRResponse`
- `connect()` / `close()` — （仅 realtime）连接生命周期，也可用 `async with`
- `realtime: bool` — 是否支持流式输入模式

> 各提供商的 ASR 能力：OpenAI 提供 `OpenAIASRModel`（批式）；DashScope 提供
> `DashScopeASRModel`（批式）/ `DashScopeRealtimeASRModel`（实时，v2.0.12+）；其余提供商
> `get_asr_model_classes()` 当前返回空列表。注意 RealtimeAgent 场景的输入转写
> （如 DashScope 的 `qwen3-asr-flash-realtime`）由 `agentscope.realtime` 模块的实时模型自己完成，
> 不使用 `ASRModelBase`，与 `asr` 模块是两条独立路径。

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
  `get_chat_model_class()` 会抛 `NotImplementedError`（v2.0.12+ `list_models()` 返回 `[]`、
  已注册进 `CredentialFactory`）。

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
