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

### DataBlock（替代 v1 的 Image/Audio/Video Block）
```python
from agentscope.message import DataBlock, Base64Source, URLSource

# Base64 数据
DataBlock(source=Base64Source(data="...", media_type="image/png"))

# URL 数据
DataBlock(source=URLSource(url="https://...", media_type="image/jpeg"))
```

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

## Msg 实用方法

```python
msg.get_text_content()                # 提取所有 TextBlock 文本（"\n" 分隔）
msg.has_content_blocks("tool_call")   # 检查是否包含指定类型
msg.has_content_blocks(["tool_call", "tool_result"])  # 支持列表
msg.get_content_blocks("text")        # 获取指定类型的 block 列表
msg.append_event(event)               # 从事件流更新消息（流式场景）
```

## Usage 类

```python
from agentscope.message import Usage

usage = Usage(input_tokens=100, output_tokens=50)
```
