# 消息 (Message) — v2

## Msg 类 (Pydantic BaseModel)

```python
from agentscope.message import Msg, Usage

msg = Msg(
    name="user",               # 发送者名称
    content=[TextBlock(text="你好")],  # list[ContentBlock]
    role="user",               # "user" | "assistant" | "system"
    id="auto-generated",       # 自动生成的唯一 ID
    metadata={},               # 附加元数据
    created_at="ISO时间戳",
    finished_at="ISO时间戳",
    usage=Usage(input_tokens=0, output_tokens=0),  # token 用量
)
```

> `Usage` 在 v2.0.7+ 新增两个缓存统计字段：`cache_input_tokens`（命中提示词缓存的输入
> token 数）与 `cache_creation_input_tokens`（写入提示词缓存的输入 token 数），均默认 0。
> 二者是 `input_tokens` 的**子集**（`input_tokens` 为整个 prompt 的 token 数）；v2.0.12+ Anthropic /
> MiniMax 也对齐此口径（此前其 `input_tokens` 不含缓存部分）。

## 工厂函数（推荐用法）

```python
from agentscope.message import UserMsg, AssistantMsg, SystemMsg

# 字符串自动包装为 [TextBlock(text=...)]
msg = UserMsg("user", "你好")
msg = AssistantMsg("Jarvis", "你好！")
msg = SystemMsg("system", "系统提示")

# content 也可以传 list[ContentBlock]
msg = UserMsg("user", [TextBlock(text="你好")])
```

**约束**：
- `UserMsg` 的 content 只能包含 `TextBlock` 或 `DataBlock`
- `SystemMsg` 的 content 只能包含 `TextBlock`
- `AssistantMsg` 的 content 可以包含任意 `ContentBlock`

## ContentBlock 类型

```python
from agentscope.message import (
    TextBlock,        # 文本
    ThinkingBlock,    # 推理过程
    HintBlock,        # 给 LLM 的提示/指令
    DataBlock,        # 二进制数据（图片/音频/视频等）
    ToolCallBlock,    # 工具调用
    ToolResultBlock,  # 工具结果
)
```

> ℹ️ 所有 Block 类型（v2.0.5+）均含自动生成的 `created_at: str`（ISO 时间戳），
> 以及可选的 `finished_at: str | None`（用于 ToolCallBlock/ToolResultBlock 等需追踪完成时机的场景），
> 无需手动赋值。

### TextBlock
```python
TextBlock(text="文本内容")
```

### ThinkingBlock
```python
ThinkingBlock(thinking="模型的思考过程...")
# 支持额外字段（如 Anthropic 的 signature）via extra="allow"
```

### HintBlock（v2 新增）
```python
HintBlock(hint="给 LLM 的提示指令")
# 传给 LLM 时会被转换为 user 消息
```

> v2.0.12+：`*MultiAgentFormatter` 不再丢弃 assistant 消息组里的 HintBlock（此前只处理 Text/Data，
> 运行时注入的时间/任务/压缩预警、RAG static 等 hint 在多智能体格式化器下可能到不了模型）——
> 消息组在每个 HintBlock 处切开：hint 单独成一条与对应 ChatFormatter 输出一致的 user 消息，
> 前后历史各成一条 `<history>…</history>` user 消息（`conversation_history_prompt` 只加在
> **第一个有文本内容**的历史段——Ollama 的图片-only 历史段不消耗该一次性槽位，且图片-only
> 消息本身不再被整体丢弃）。

### DataBlock（替代 v1 的 Image/Audio/Video Block）
```python
from agentscope.message import DataBlock, Base64Source, URLSource

# Base64 数据
DataBlock(source=Base64Source(data="...", media_type="image/png"))

# URL 数据
DataBlock(source=URLSource(url="https://...", media_type="image/jpeg"))
```

字段：`id`（自动生成）、`source: Base64Source | URLSource`、可选 `name`。
- `Base64Source(data: str, media_type: str)` — 内联 base64 字节，`media_type` 形如 `"image/png"`/`"audio/pcm;rate=24000"`。
- `URLSource(url: str, media_type: str)` — 远程 URL，一次性加载；也接受 `file://` 本地文件
  URL（各 formatter 会读文件并转 base64 上传，v2.0.11+ 起百分号编码的路径——如含中文/空格——
  先解码再打开）。

**provider 专属媒体参数（v2.0.12+）**：`DataBlock` 为 `extra="allow"`，可直接附带额外关键字参数
（存于 `block.model_extra`，随 `model_dump()`/持久化往返保留），由各 formatter 按目标 API 的嵌套规则放置：

```python
DataBlock(source=URLSource(url="https://example.com/a.png", media_type="image/png"), detail="high")
# OpenAI Chat → {"type": "image_url", "image_url": {"url": "...", "detail": "high"}}
DataBlock(source=URLSource(url="https://example.com/v.mp4", media_type="video/mp4"), fps=2)
# DashScope → {"type": "video_url", "video_url": {"url": "..."}, "fps": 2}
```

| Formatter | 额外参数放置位置 |
|---|---|
| OpenAI Chat（含 DeepSeek） | image 嵌入 `image_url` 对象、audio 嵌入 `input_audio` 对象；PDF 不转发 |
| OpenAI Responses | 平铺到 `input_image` 项顶层；PDF/音频不转发 |
| Anthropic（MiniMax 图片/PDF 同） | image / PDF document 内容块顶层（如 `cache_control={"type": "ephemeral"}`） |
| DashScope | image/video/audio 内容项顶层（如 `fps`、`snippet`） |
| Gemini | part 顶层，与 `inline_data` 同级（如 `video_metadata={"fps": 2}`） |
| Moonshot | image/video 嵌入 `image_url`/`video_url` 对象 |
| xAI | 作为关键字参数传给 `xai_sdk.chat.image(url, **extras)`（只能用该函数接受的参数） |
| Volcengine / Ollama / MiniMax 视频 | 不转发 |

框架不校验键名，与 formatter 自身生成的同名键会覆盖后者。

### ToolCallBlock
```python
from agentscope.message import ToolCallBlock, ToolCallState

ToolCallBlock(
    id="xxx",
    name="tool_name",
    input='{"arg": "value"}',     # 原始 JSON 字符串
    state=ToolCallState.PENDING,  # pending/asking/allowed/submitted/finished
)
```

> `id` 字段（v2.0.4+）承载 provider 的 call_id（如 OpenAI Responses API 返回的 `call_id`），
> 用于把后续 `ToolResultBlock` 与对应的工具调用配对。不再用 `extra="allow"` 容纳 provider 私有字段。

ToolCallState 状态流转：
```
pending → asking (需用户确认) → allowed (用户允许) → finished
pending → allowed (自动允许) → finished
pending → finished (被拒绝)
allowed → submitted (外部执行) → finished
```

### ToolResultBlock
```python
from agentscope.message import ToolResultBlock, ToolResultState

ToolResultBlock(
    id="xxx",
    name="tool_name",
    output="结果字符串" 或 [TextBlock(...), DataBlock(...)],
    state=ToolResultState.SUCCESS,  # success/error/interrupted/denied/running
)
```

> v2.0.12+：Anthropic formatter 对 `state=ERROR` 的工具结果在 tool_result 内容块附带
> `is_error: true`（出站方向；入站方向 `is_error` → `ToolResultState.ERROR` 早已支持），
> 转发历史时错误状态不再被抹掉。

## Msg 实用方法

```python
msg.get_text_content()                # 拼接顶层 TextBlock 文本（separator 默认 "\n"），恒返回 str——v2.0.12+ 无文本块时返回 ""（此前为 None）；区分"无文本块"与"空文本"用 has_content_blocks("text")
msg.has_content_blocks("tool_call")   # 检查是否包含指定类型
msg.has_content_blocks(["tool_call", "tool_result"])  # 支持列表
msg.get_content_blocks("text")        # 获取指定类型的 block 列表
msg.append_event(event)               # 从事件流更新消息（流式场景）
msg.append_usage(usage)               # 累计一次模型调用的 token 用量（消息持有副本，不突变传入的 Usage 对象；v2.0.8，副本语义 v2.0.10+）
```

## Usage 类

```python
from agentscope.message import Usage

usage = Usage(input_tokens=100, output_tokens=50)
# v2.0.7+ 缓存统计（默认 0）：
#   cache_input_tokens          # 命中提示词缓存的输入 token 数
#   cache_creation_input_tokens # 写入提示词缓存的输入 token 数
```
