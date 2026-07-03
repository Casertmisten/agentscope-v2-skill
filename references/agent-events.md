# Agent 与事件系统 — v2

## Agent 类

v2 只有一个 `Agent` 类，不再区分 AgentBase/ReActAgentBase/ReActAgent。

### 导入

```python
from agentscope.agent import Agent, ContextConfig, ReActConfig, ModelConfig
```

### 构造参数

```python
agent = Agent(
    name="Jarvis",                          # 智能体名称
    system_prompt="你是一个有用的助手。",     # 系统提示（注意不是 sys_prompt）
    model=chat_model,                       # ChatModelBase 实例（必填）
    toolkit=Toolkit(...),                   # 工具模块（可选）
    middlewares=[...],                      # 中间件列表（可选）
    state=AgentState(),                     # 初始状态（可选）
    offloader=workspace,                    # Offloader/Workspace（可选）
    model_config=ModelConfig(...),          # 模型重试/fallback 配置
    context_config=ContextConfig(...),      # 上下文压缩配置
    react_config=ReActConfig(...),          # ReAct 配置
)
```

### Agent 方法

#### reply_stream — 流式事件

```python
async for event in agent.reply_stream(user_msg):
    # event 是 AgentEvent 类型
    print(event.type)
```

**返回类型**：`AsyncGenerator[AgentEvent, None]`

`inputs` 参数支持多种类型：
- `Msg | list[Msg]` — 新消息触发推理
- `UserConfirmResultEvent` — 用户确认结果（继续之前的暂停）
- `ExternalExecutionResultEvent` — 外部执行结果
- `None` — 从当前状态继续

#### reply — 获取最终消息

```python
final_msg: Msg = await agent.reply(user_msg)
```

**返回类型**：`Msg`（消费所有事件，返回最终回复消息）

#### observe — 接收观察消息

```python
await agent.observe(other_agent_msg)
# 仅保存到 context，不触发推理
```

#### compress_context — 手动压缩上下文

```python
from agentscope.message import HintBlock

await agent.compress_context()
# 可选传入自定义 ContextConfig
await agent.compress_context(context_config=ContextConfig(trigger_ratio=0.5))
# 可选传入压缩提示，指导摘要保留哪些信息
await agent.compress_context(instructions=HintBlock(hint="保留 API 迁移决策"))
```

## ReActConfig

```python
react_config = ReActConfig(
    max_iters=20,            # 推理-行动最大迭代次数
    stop_on_reject=False,    # 工具被拒绝时是否停止
)
```

## ModelConfig

```python
model_config = ModelConfig(
    max_retries=0,                        # 重试次数（不含首次调用）
    fallback_model=some_fallback_model,   # 备用模型（ChatModelBase 实例）
)
```

## ContextConfig

```python
context_config = ContextConfig(
    trigger_ratio=0.8,        # token 使用比例阈值（>0, <0.9）
    reserve_ratio=0.1,        # 压缩后保留比例（>0, <0.9）
    tool_result_limit=3000,   # 工具结果 token 上限
    compression_prompt="...", # 自定义压缩提示
    summary_template="...",   # 自定义摘要模板
    summary_schema={...},     # 自定义摘要 JSON Schema（默认使用 SummarySchema）
)
```

## 事件系统 (Event)

### 事件类型总览

| 事件类型 | 说明 |
|---|---|
| `ReplyStartEvent` | reply 开始（含 session_id, reply_id, name） |
| `ReplyEndEvent` | reply 结束 |
| `ModelCallStartEvent` | 模型调用开始（含 model_name） |
| `ModelCallEndEvent` | 模型调用结束（含 input/output tokens） |
| `TextBlockStartEvent` | 文本块开始 |
| `TextBlockDeltaEvent` | 文本增量（`delta` 字段） |
| `TextBlockEndEvent` | 文本块结束 |
| `ThinkingBlockStartEvent` | 推理块开始 |
| `ThinkingBlockDeltaEvent` | 推理增量 |
| `ThinkingBlockEndEvent` | 推理块结束 |
| `DataBlockStartEvent` | 数据块开始（含 media_type，如音频流） |
| `DataBlockDeltaEvent` | 数据增量（base64） |
| `DataBlockEndEvent` | 数据块结束 |
| `HintBlockEvent` | 提示块事件 |
| `ToolCallStartEvent` | 工具调用开始（含 tool_call_id, name） |
| `ToolCallDeltaEvent` | 工具参数增量（JSON 片段） |
| `ToolCallEndEvent` | 工具调用结束 |
| `ToolResultStartEvent` | 工具结果开始 |
| `ToolResultTextDeltaEvent` | 工具结果文本增量 |
| `ToolResultDataDeltaEvent` | 工具结果数据增量 |
| `ToolResultEndEvent` | 工具结果结束（含 state） |
| `ExceedMaxItersEvent` | 超过最大迭代次数 |
| `RequireUserConfirmEvent` | 需要用户确认（含 tool_calls 列表） |
| `RequireExternalExecutionEvent` | 需要外部执行（含 tool_calls） |
| `UserConfirmResultEvent` | 用户确认结果（用于继续回复） |
| `ExternalExecutionResultEvent` | 外部执行结果（用于继续回复） |

### 所有事件的公共字段

```python
event.id          # 唯一事件 ID
event.created_at  # ISO 时间戳
event.type        # 事件类型字符串
event.reply_id    # 所属 reply 的 ID
```

### 典型事件处理模式

```python
async for event in agent.reply_stream(user_msg):
    match event.type:
        # 文本流式输出
        case "TEXT_BLOCK_DELTA":
            print(event.delta, end="", flush=True)

        # 推理过程
        case "THINKING_BLOCK_DELTA":
            print(f"[思考] {event.delta}")

        # 数据流（如音频）
        case "DATA_BLOCK_DELTA":
            print(f"[数据] {event.media_type}: {len(event.data)} bytes")

        # 工具调用
        case "TOOL_CALL_START":
            print(f"调用工具: {event.tool_call_name}")
        case "TOOL_RESULT_END":
            print(f"工具结果状态: {event.state}")
        # 需要用户确认
        case "REQUIRE_USER_CONFIRM":
            for tc in event.tool_calls:
                print(f"需确认: {tc.name}({tc.input})")

        # 需要外部执行
        case "REQUIRE_EXTERNAL_EXECUTION":
            for tc in event.tool_calls:
                print(f"需外部执行: {tc.name}")

        # token 用量
        case "MODEL_CALL_END":
            print(f"tokens: {event.input_tokens}+{event.output_tokens}")

        # 结束
        case "REPLY_END":
            print("\n--- 回复结束 ---")
```

### 人工介入（HITL）模式

当 Agent 需要用户确认或外部执行时，会暂停并等待：

```python
# 第一次调用 - 可能返回 REQUIRE_USER_CONFIRM 事件
async for event in agent.reply_stream(user_msg):
    if event.type == "REQUIRE_USER_CONFIRM":
        # 保存事件信息，暂停处理
        saved_event = event
        break

# 用户确认后，再次调用传入确认结果
from agentscope.event import UserConfirmResultEvent, ConfirmResult
confirm_event = UserConfirmResultEvent(
    confirm_results=[ConfirmResult(confirmed=True, tool_call=tc)]
)
final_msg = await agent.reply(confirm_event)
```

> `ConfirmResult` 字段：`confirmed: bool`、`tool_call: ToolCallBlock`、可选 `rules: list[PermissionRule]`
> （确认时附带的权限规则，会沉淀进会话，避免重复询问）。两者均从 `agentscope.event` 导入。

### Omni 模型音频流（v2.0.2+）

支持音频输出的 omni 模型（如 `gpt-audio-mini`、`qwen3.5-omni-plus`）会通过 `DATA_BLOCK_*` 事件流式返回生成的语音。每个 `DataBlockDeltaEvent.data` 是增量 base64 PCM 块，按 `block_id` 串接得到完整音频：

```python
audio_buffers: dict[str, bytearray] = {}   # block_id -> bytes

async for event in agent.reply_stream(user_msg):
    match event.type:
        case "DATA_BLOCK_START":
            audio_buffers[event.block_id] = bytearray()
        case "DATA_BLOCK_DELTA":
            # event.data 是 base64 PCM 增量；event.media_type 形如 "audio/pcm"
            audio_buffers[event.block_id].extend(base64.b64decode(event.data))
        case "DATA_BLOCK_END":
            audio = bytes(audio_buffers.pop(event.block_id))
```

注意：
- 模型原生音频由 `DATA_BLOCK_*` 传递，**不会**写入 `state.context`（避免记忆膨胀）。
- 也可以用 `TTSMiddleware` 把任意文本回复转成同样的 `DATA_BLOCK_*` 音频流（见 middleware 文档），此时数据来自 TTS 模型而非对话模型。

### 构建最终消息

可以边消费事件边构建最终的 `Msg`：

```python
from agentscope.message import Msg

final_msg = Msg(name="assistant", content=[], role="assistant")
async for event in agent.reply_stream(user_msg):
    final_msg.append_event(event)

# final_msg 现在包含完整的回复
print(final_msg.get_text_content())
print(final_msg.usage)
```
