# Agent 与事件系统 — v2

## Agent 类

v2 只有一个 `Agent` 类，不再区分 AgentBase/ReActAgentBase/ReActAgent。

### 导入

```python
from agentscope.agent import Agent, ContextConfig, ReActConfig, ModelConfig, InjectionConfig
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
    injection_config=InjectionConfig(...),  # 运行时状态注入配置（v2.0.5+，可选）
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

可选参数（v2.0.5+）：
- `structured_schema: Type[BaseModel] | None` — 传入 Pydantic 模型类触发结构化输出（见下文）。
- `yield_final_msg: bool` — `True` 时在事件流末尾额外 `yield` 最终的 `Msg`（结构化输出场景
  需要它才能拿到带 `structured_output` 字段的消息；默认 `False`）。

`inputs` 参数支持多种类型：
- `Msg | list[Msg]` — 新消息触发推理
- `UserConfirmResultEvent` — 用户确认结果（继续之前的暂停）
- `ExternalExecutionResultEvent` — 外部执行结果
- `UserInterruptEvent` — 中断一个 parked reply（见下文「Agent 中断」）
- `None` — 从当前状态继续

#### reply — 获取最终消息

```python
final_msg: Msg = await agent.reply(user_msg)
```

**返回类型**：`Msg`（消费所有事件，返回最终回复消息）

支持 `structured_schema` 参数（v2.0.5+）产出 Pydantic 结构化输出，详见下文「结构化输出」。

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
    interruption_message="I notice the interruption. How can I help you?",
    # 中断事件触发后，agent 回退并产出的替代文案（见下文「Agent 中断」）
    interruption_raise_cancelled_error=False,
    # True: 处理完中断后重新抛 CancelledError；False: 静默结束 reply
    structured_output_grace_iters=5,   # v2.0.5+: 超过 max_iters 后留给结构化输出的额外迭代数
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
    tool_result_limit=50000,   # 工具结果 token 上限，默认 50000
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
| `ReplyEndEvent` | reply 结束（含 `finished_reason`；v2.0.5+ 出错时 `finished_reason=ERROR` 且带 `error: ErrorInfo`） |
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
| `ToolCallStartEvent` | 工具调用开始（含 tool_call_id, tool_call_name） |
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
| `UserInterruptEvent` | 用户中断（用于中止一个 parked reply，v2.0.4+） |
| `CustomEvent` | 服务层自定义事件（type=`"CUSTOM"`，含 name/value，用于中间件通知前端状态/团队成员/权限变更等） |

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

### Agent 中断（v2.0.4+）

当 agent 处于 parked 状态（等待 HITL 用户确认 / 外部执行结果）时，可向它发送
`UserInterruptEvent` 来中止这次 reply：

```python
from agentscope.event import UserInterruptEvent

# reply_id 指向那个正在等待确认/外部结果的 reply
await agent.reply(UserInterruptEvent(reply_id="原reply_id"))
```

收到 `UserInterruptEvent` 后，agent 会：
1. 把所有 pending 的 tool call 关闭为 `interrupted` 结果；
2. emit 一条 fallback 助手消息（文案由 `ReActConfig.interruption_message` 控制）；
3. 以 `ReplyEndReason.INTERRUPTED` 结束 reply，**不进入** reasoning-acting 循环。

`ReplyEndEvent` 的 `finished_reason` 取值（`ReplyFinishedReason` 枚举，v2.0.5+；旧的
`ReplyEndReason` 仍保留但已 deprecated）：

| 值 | 含义 |
|---|---|
| `completed` | 正常完成 |
| `interrupted` | 被 `UserInterruptEvent` 中断 |
| `exceed_max_iters` | 超过 `ReActConfig.max_iters` |
| `error`（v2.0.5+） | reply 过程抛异常，`error: ErrorInfo` 字段填充错误分类 |

> v2.0.5+：reply 出错时不再静默崩溃——`Msg` 和 `ReplyEndEvent` 都会带上 `finished_reason=ERROR`
> 与 `error: ErrorInfo`（`type` 按 HTTP 语义分类，`message` 是通用文案不泄露密钥）。
> 见下文「回复错误上报」。

> 注意区分两类「中断」：
> - **parked reply 的中断**：用 `UserInterruptEvent`（本文所述），针对等待 HITL/外部结果的 reply。
> - **running reply 的取消**：直接取消驱动 `reply_stream` 的底层 asyncio task，agent 经
>   `CancelledError` 清理路径（`ReActConfig.interruption_raise_cancelled_error=True` 时会在
>   清理后重新抛出，便于上层捕获）。
>
> 服务化场景下，HTTP 端 `POST /sessions/{sid}/interrupt` 会派发 `UserInterruptEvent`。

## 结构化输出（v2.0.5+）

`reply` / `reply_stream` 新增 `structured_schema: Type[BaseModel] | None` 参数。传入 Pydantic
模型类后，Agent 在 ReAct 循环中装入内置的 `GenerateStructuredOutput` 工具，引导模型产出符合
schema 的结构化结果，并写入 `Msg.structured_output`：

```python
from pydantic import BaseModel

class WeatherReport(BaseModel):
    city: str
    temperature: float

# 非流式
res = await agent.reply(
    UserMsg("user", "杭州天气如何？"),
    structured_schema=WeatherReport,
)
print(res.structured_output)   # {"city": "Hangzhou", "temperature": 25.0}

# 流式：需 yield_final_msg=True 才能拿到带 structured_output 的最终消息
async for item in agent.reply_stream(
    UserMsg("user", "杭州天气如何？"),
    structured_schema=WeatherReport,
    yield_final_msg=True,
):
    if isinstance(item, Msg):
        print(item.structured_output)
```

要点：
- 是 `reply` / `reply_stream` 的参数，**不是** `Agent(...)` 构造参数。
- `ReActConfig.structured_output_grace_iters`（默认 5）：超过 `max_iters` 后额外留给结构化输出的迭代数。
- `structured_output` 是 dict（Pydantic 模型 `.model_dump()` 的结果），存于 `Msg` 与 `ReplyContext`。
- 内置工具 `GenerateStructuredOutput` 的权限始终 ALLOW（只读、并发安全），模型校验失败会重试。

## 运行时状态注入 InjectionConfig（v2.0.5+）

Agent 默认在每个 reasoning 步骤前，把**当前墙钟时间、(plan) 任务状态、上下文压缩临近预警**
作为 `HintBlock` 追加到 `state.context`（不改 system prompt，避免破坏 prompt caching）。
由 `InjectionConfig` 控制，从 `agentscope.agent` 导出：

```python
from agentscope.agent import Agent, InjectionConfig

agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    injection_config=InjectionConfig(
        inject_runtime_state=True,    # 默认 True；False 完全关闭注入
        timezone="UTC",               # 时区名；无效/缺 tzdata 时回退 UTC
        time_format="%Y-%m-%dT%H:%M:%S",
        time_interval=0.5,            # 小时；距上次注入超过该值再注入时间
        context_buffer_ratio=0.2,     # 进入压缩阈值前 buffer 区时注入预警
        template="...{runtime_state}...",  # 必须含 {runtime_state} 占位符
        task_tool_names=["TaskCreate", "TaskGet", "TaskList", "TaskUpdate"],
        emit_hint_event=True,         # 是否发 HintBlockEvent 供前端展示
        extra_fields={},              # 额外注入字段
    ),
)
```

三个注入维度（各自独立判断是否触发）：
- **时间**：context 中无记录时间，或距上次注入超过 `time_interval` 小时。
- **任务**：存在 pending/in-progress 任务，且 agent 尚未感知（context 里无相关工具调用/注入记录）。
- **上下文**：reply 第一轮且 token 数进入压缩阈值前的 buffer 区时，提醒 agent 压缩临近。

约束：`context_buffer_ratio` 必须 < `ContextConfig.trigger_ratio`，否则 `Agent.__init__` 报错。
HintBlock 的 source 固定为 `{"label": "System", "sublabel": "Runtime State"}`。

## 回复错误上报（v2.0.5+）

reply 抛异常时，框架不再让错误静默丢失，而是把错误分类写进 `Msg` 和 `ReplyEndEvent`，
供前端/上层统一处理。类型从 `agentscope.types` 导出：

```python
from agentscope.types import ReplyFinishedReason, ErrorType, ErrorInfo

msg = await agent.reply(user_msg)
if msg.finished_reason == ReplyFinishedReason.ERROR:
    err: ErrorInfo = msg.error
    print(err.type, err.message)
    # e.g. ErrorType.RATE_LIMIT, "Rate limit or quota exceeded — try again later."
```

`ErrorType`（按 HTTP 语义分类，provider 无关）：

| `ErrorType` | HTTP 语义 | 典型场景 |
|---|---|---|
| `AUTHENTICATION` | 401 | API key 无效/过期 |
| `PERMISSION` | 403 | 无访问权限 |
| `RATE_LIMIT` | 429 | 限流/配额用尽 |
| `INVALID_REQUEST` | 400/422 | 请求格式错误 |
| `UPSTREAM` | 5xx | 服务端错误 |
| `CONNECTION` | — | 网络中断/超时 |
| `INTERNAL` | — | 框架内部错误 |
| `UNKNOWN` | — | 未能分类 |

要点：
- `ErrorInfo.message` 是通用文案，**不**包含原始异常字符串/密钥/内部细节。
- 服务化（`create_app`）的 SSE 流：reply 在发出 `ReplyEndEvent` 前崩溃时，会合成一个
  `finished_reason=ERROR`、`error=_classify_error(e)` 的 `ReplyEndEvent` 并 publish 到 SSE，
  同时 append 到持久化的 reply 消息。
- `CancelledError`（用户取消）不受影响，走原有中断路径。
