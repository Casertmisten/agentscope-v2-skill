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
    sys_prompt="你是一个有用的助手。",        # 系统提示
    credential=OpenAICredential(...),        # API 认证
    model="gpt-4o",                         # 模型名称
    toolkit=Toolkit(...),                   # 工具模块
    state=AgentState(),                     # 初始状态（可选）
    context_config=ContextConfig(...),       # 上下文配置
    react_config=ReActConfig(...),           # ReAct 配置
    model_config=ModelConfig(...),           # 模型重试/fallback 配置
)
```

### reply 方法

核心方法，接收用户消息，返回事件流：

```python
async for event in agent.reply(user_msg):
    # event 是 AgentEvent 类型
    print(event.type)
```

**返回类型**：`AsyncGenerator[AgentEvent, None]`

可以使用 `user_input` 参数传入用户确认结果等控制信号。

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
    max_retries=0,                        # 重试次数
    fallback_model=some_fallback_model,   # 备用模型
)
```

## ContextConfig

```python
context_config = ContextConfig(
    trigger_ratio=0.8,        # token 使用比例阈值
    reserve_ratio=0.1,        # 压缩后保留比例
    tool_result_limit=3000,   # 工具结果 token 上限
    compression_prompt="...", # 自定义压缩提示
    summary_template="...",   # 自定义摘要模板
    summary_schema={...},     # 自定义摘要 Schema
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
| `DataBlockStartEvent` | 数据块开始（含 media_type） |
| `DataBlockDeltaEvent` | 数据增量（base64） |
| `DataBlockEndEvent` | 数据块结束 |
| `ToolCallStartEvent` | 工具调用开始（含 tool_call_id, name） |
| `ToolCallDeltaEvent` | 工具参数增量（JSON 片段） |
| `ToolCallEndEvent` | 工具调用结束 |
| `ToolResultStartEvent` | 工具结果开始 |
| `ToolResultTextDeltaEvent` | 工具结果文本增量 |
| `ToolResultDataDeltaEvent` | 工具结果数据增量 |
| `ToolResultEndEvent` | 工具结果结束（含 state） |
| `ExceedMaxItersEvent` | 超过最大迭代次数 |
| `RequireUserConfirmEvent` | 需要用户确认（含 tool_calls 列表） |
| `RequireExternalExecutionEvent` | 需要外部执行 |
| `UserConfirmResultEvent` | 用户确认结果 |
| `ExternalExecutionResultEvent` | 外部执行结果 |

### 所有事件的公共字段

```python
event.id          # 唯一事件 ID
event.created_at  # ISO 时间戳
event.type        # 事件类型字符串
event.reply_id    # 所属 reply 的 ID
```

### 典型事件处理模式

```python
async for event in agent.reply(user_msg):
    match event.type:
        # 文本流式输出
        case "TEXT_BLOCK_DELTA":
            print(event.delta, end="", flush=True)

        # 推理过程
        case "THINKING_BLOCK_DELTA":
            print(f"[思考] {event.delta}")

        # 工具调用
        case "TOOL_CALL_START":
            print(f"调用工具: {event.tool_call_name}")
        case "TOOL_RESULT_END":
            print(f"工具结果状态: {event.state}")

        # 需要用户确认
        case "REQUIRE_USER_CONFIRM":
            for tc in event.tool_calls:
                print(f"需确认: {tc.name}({tc.input})")

        # token 用量
        case "MODEL_CALL_END":
            print(f"tokens: {event.input_tokens}+{event.output_tokens}")

        # 结束
        case "REPLY_END":
            print("\n--- 回复结束 ---")
```

### 构建最终消息

可以边消费事件边构建最终的 `Msg`：

```python
from agentscope.message import Msg

final_msg = Msg(name="assistant", content=[], role="assistant")
async for event in agent.reply(user_msg):
    final_msg.append_event(event)

# final_msg 现在包含完整的回复
print(final_msg.get_text_content())
print(final_msg.usage)
```
