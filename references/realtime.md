# 实时语音 (Realtime / RealtimeAgent)

v2.0.8+ 主干新增（未随 v2.0.8 发布，标注 `v2.0.8+`）。实时语音分两层：

- **`agentscope.realtime` 模块** — 模型侧：realtime 模型适配器、模型卡片、模型事件、传输层与 VAD；
- **`agentscope.agent.RealtimeAgent`** — 把实时模型、传输（麦克风/扬声器）和话轮状态机组装成语音智能体。

与 `Agent` 不同，`RealtimeAgent` 是**双向流**：音频持续流入，事件持续从 `reply_stream` 流出，
没有请求/应答边界。工具调用在 agent 内执行（走 `state.permission_context` 权限检查，
HITL 确认事件同样会出现在事件流里）。

> ⚠️ 属于官方 Voice Agent 路线的进行中方向，API 可能随版本变动，以源码为准。

## 最小示例

```python
from agentscope.agent import RealtimeAgent
from agentscope.credential import OpenAICredential
from agentscope.realtime import OpenAIRealtimeModel, LocalAudioTransport

model = OpenAIRealtimeModel(
    model="gpt-realtime-2.1",
    credential=OpenAICredential(api_key="sk-xxx"),
)
agent = RealtimeAgent("Friday", "回答简短。", model)

async with agent:                              # 模型会话（connect/close）
    async with LocalAudioTransport() as t:     # 声卡（需 pip install sounddevice）
        async for event in agent.reply_stream(t):
            print(event)                       # AgentEvent：文本增量/工具调用/确认请求...
```

`reply_stream` 产出与普通 `Agent` 相同体系的 `AgentEvent`（`TEXT_BLOCK_DELTA`、`TOOL_CALL_START`、
`REQUIRE_USER_CONFIRM` 等，见 [agent-events.md](agent-events.md)）；`agentscope.realtime`
自身的 `ModelEvent` 是**模型侧**事件，只与自定义模型适配器相关（见下文）。

## RealtimeAgent

```python
from agentscope.agent import RealtimeAgent, TurnAggregator, TurnMetrics

agent = RealtimeAgent(
    name="Friday",                 # 显示名，标注到助手消息与事件
    system_prompt="回答简短。",     # connect 时发送；toolkit 的 skill 指令自动追加
    model=model,                   # RealtimeModelBase 实例
    toolkit=None,                  # 可选：模型可调用的工具，agent 内执行
    state=None,                    # 可选：AgentState（历史/权限/工具上下文）
    vad=None,                      # 可选：本地 VAD（VADBase），见下文
    aggregator=None,               # 可选：TurnAggregator，清洗用户话轮
)
```

**三个生命周期刻意分离**：agent 拥有模型会话与状态；transport 归创建者所有；
`reply_stream` 只是借用二者。因此客户端断开重连不丢模型会话，模型会话静默超时
也会在用户说下一句话时自动重连（指数退避；重连时已发生的对话以 transcript 形式
拼进 instructions 恢复上下文）。

### 方法

| 方法 / 属性 | 说明 |
|---|---|
| `async with agent:` / `connect()` / `close()` | 打开/关闭模型会话。`connect` 幂等，重连安全 |
| `reply_stream(transport)` | 挂接一个传输并异步迭代 `AgentEvent`，直到该 transport 的输入结束 |
| `send(inputs)` | 非音频输入的唯一入口：`str`/`Msg` 文本轮、`UserConfirmResultEvent`（HITL 应答）、`UserInterruptEvent`（打断）。文本轮会先打断进行中的回复 |
| `interrupt()` | 等价于 `send(UserInterruptEvent())`，即用户按"停止" |
| `last_turn_metrics` | `TurnMetrics`：最近一轮的时延分解 |

`reply_stream` 的语义细节：

- transport 是**借用**：必须已 `start()`，且不会被它关闭——调用方可以保留复用；
- 流在 transport 输入结束时结束（如客户端断开），**不是**某个回复结束——一次调用里有多个回复；
- 换新 transport 再次调用即可续接同一模型会话；
- 同一 agent 同时只允许一个活跃的 `reply_stream`（再调用抛 `RuntimeError`）；
- 提前 `break` 不会立即清理，需要即时清理时用 `contextlib.aclosing` 包住生成器；
- transport 与模型的输入/输出采样率必须一致，重采样是 transport 的职责（agent 原样转发）。

### TurnAggregator / TurnMetrics

`TurnAggregator` 把 provider 推断出的转写收拢成干净的用户话轮：合并被停顿拆开的一句话、
丢弃纯附和词。可子类化改变"什么算一个话轮"：

```python
TurnAggregator(
    merge_window_ms=800,     # 距上一话轮多久内到达的转写视为续说
    backchannels=frozenset({"嗯", "好的"}),  # 附和词（语言相关，默认空）
    min_chars=1,             # 短于此的转写丢弃
)
```

`TurnMetrics` 记录一轮的四个单调时钟时间戳（用户停说 / 话轮提交 / 后端首音频字节 /
首音频到达扬声器）及派生时延（`endpointing_delay` / `backend_ttfb` / `first_audio_delay` 等）
和 token 用量。

## Realtime 模型

统一构造签名（四个 provider 一致）：

```python
ModelClass(
    model="gpt-realtime-2.1",     # 模型名（必须有配套卡片，或显式传 model_card）
    credential=...,               # 对应 provider 的 CredentialBase 实例
    parameters=None,              # 各自的 Parameters 子类，缺省用默认值
    model_card=None,              # 不传则按 model 名在自带 yaml 卡片中查找
)
ModelClass.list_models()          # -> list[RealtimeModelCard]，含自定义目录
ModelClass.list_models(custom_yaml_dir="/path/to/yamls")
```

### 内置适配器与模型

```python
from agentscope.realtime import (
    DashScopeRealtimeModel,        # Qwen-Omni 系列（WebSocket）
    DashScopeAudioRealtimeModel,   # Qwen-Audio-3.0-Realtime
    OpenAIRealtimeModel,           # GPT Realtime API
    GeminiRealtimeModel,           # Gemini Live API
    XAIRealtimeModel,              # Grok Voice
)
```

| 适配器 | credential | 模型（yaml 卡片） |
|---|---|---|
| `DashScopeRealtimeModel` | `DashScopeCredential` | `qwen-omni-turbo-realtime`、`qwen3-omni-flash-realtime`、`qwen3.5-omni-flash-realtime`、`qwen3.5-omni-plus-realtime` |
| `DashScopeAudioRealtimeModel` | `DashScopeCredential` | `qwen-audio-3.0-realtime-flash`、`qwen-audio-3.0-realtime-plus` |
| `OpenAIRealtimeModel` | `OpenAICredential` | `gpt-realtime-1.5`、`gpt-realtime-2`、`gpt-realtime-2.1`、`gpt-realtime-2.1-mini` |
| `GeminiRealtimeModel` | `GeminiCredential` | `gemini-2.5-flash-native-audio-preview-12-2025`、`gemini-3.1-flash-live-preview` |
| `XAIRealtimeModel` | `XAICredential` | `grok-voice-latest`、`grok-voice-think-fast-2.0` |

`DashScopeAudioRealtimeModel` 是 `DashScopeRealtimeModel` 的子类，差异：
`supports_text_input=True`（可在会话中注入文本轮）、话轮检测用 `smart_turn`
端点（`turn_detection: "server_vad"|"smart_turn"|"none"`，`voiceprint_audio_urls`
声纹参考仅 `smart_turn` 使用）。

各适配器的 `Parameters`（音色、VAD 阈值、`max_history_turns` 等）不同，
通过 `RealtimeModelCard.parameter_schema` 或 `ModelClass.list_models()` 查看字段。

### RealtimeModelCard

YAML 模型卡片（`realtime/_<provider>/_models/*.yaml`，可用 `custom_yaml_dir` 扩展），
关键字段：`name` / `label` / `status`、`input_types` / `output_types`、
`input_sample_rate` / `output_sample_rate`、`supports_tools`、
`max_context_tokens` / `max_audio_turns` / `max_audio_duration_s` /
`max_session_duration_s`、`parameter_schema`（可调参数的 JSON Schema，供 UI 展示）。

### 自定义 realtime 模型

继承 `RealtimeModelBase` 并实现抽象方法：

```python
class MyRealtimeModel(RealtimeModelBase):
    type = "my_realtime"                          # 适配器标识，写进每张卡片
    truncation = TruncationSupport.EXPLICIT       # NONE / SERVER / EXPLICIT
    supports_text_input = False                   # 能否 mid-session 注入文本轮

    async def connect(self, instructions, tools=None, **kwargs): ...   # 开会话
    async def close(self): ...                                         # 关会话
    async def events(self): ...                    # AsyncIterator[ModelEvent]
    async def push_audio(self, pcm: bytes): ...    # 用户音频：PCM16 单声道
    async def push_text(self, text: str): ...      # 文本用户轮
    async def push_tool_result(self, block: ToolResultBlock): ...
    async def commit_turn(self): ...               # 仅调用方自持端点检测时调用
    async def request_response(self): ...          # 请求开始生成回复
    async def cancel_response(self): ...           # 停止当前回复但不关会话
    async def truncate(self, item_id, played_ms, played_text): ...
```

`TruncationSupport` 描述被打断话轮的纠正方式：`SERVER`（provider 自纠，`truncate` 空操作）、
`EXPLICIT`（接受显式 truncate 帧）、`NONE`。
`ModelDisconnectedError`（`ConnectionError` 子类）表示 provider 关闭了会话——
下一句用户音频会自动重连。

`connect` 的 `**kwargs`：`turn_detection_disabled=True` 要求 provider 关闭自身话轮检测
（调用方自跑 VAD 时使用）；部分 provider 还接受 resumption 句柄。

### 模型事件（ModelEvent）

`agentscope.realtime` 的模型侧事件，供自定义适配器/传输层使用（agent 已内置消费）：

`ModelEvent` 基类及 `SessionEndedEvent`、`SpeechStartedEvent`、`SpeechEndedEvent`、
`InputTranscriptionEvent`（用户转写）、`ResponseCreatedEvent`、`AudioDeltaEvent`（生成的音频块）、
`TranscriptDeltaEvent`（助手转写增量）、`ToolCallEvent`、`ResponseDoneEvent`、`ModelErrorEvent`。

## 传输层 (Transport)

```python
from agentscope.realtime import (
    TransportBase,          # 抽象基类：start/incoming/outgoing/__aenter__
    LocalAudioTransport,    # 本地声卡（sounddevice）
    AudioFrame,             # PCM16 单声道音频帧
    ControlFrame,           # 控制帧（打字/暂停等信令）
    ControlFrameType,
    TransportFrame,         # AudioFrame | ControlFrame 联合
    PlayoutPosition,        # 播放进度（item_id + 已播放毫秒/文本前缀）
)
```

`LocalAudioTransport` 基于 `sounddevice`（PortAudio），麦克风采集回调在音频线程上运行、
经 `call_soon_threadsafe` 交还事件循环；播放进度统计与浏览器 AudioWorklet 同一语义：

```python
LocalAudioTransport(
    input_sample_rate=16000,    # 应与模型 input_sample_rate 一致
    output_sample_rate=24000,   # 应与模型 output_sample_rate 一致
    input_device=None,          # sounddevice 设备号/名，None 用默认
    output_device=None,
    chunk_ms=100,               # 采集块长；内部缓冲约 10s，超出丢最旧
    fade_ms=30,                 # clear_audio 淡出防爆音
)
```

自定义 transport：继承 `TransportBase`，实现 `start()` / `incoming()`（异步产出
`TransportFrame`）/ 播放接口。浏览器端控制帧应调用 `RealtimeAgent` 的公共方法
（`send` / `interrupt`），与 Python 调用方走同一条语义路径。

## VAD（语音活动检测）

```python
from agentscope.realtime import VADBase, SpeechTransition
```

给 `RealtimeAgent` 传 `vad=SomeVAD()` 时：provider 自身的话轮检测被关闭
（`turn_detection_disabled=True`），由本地 VAD 唯一决定话轮边界——
`STARTED` 打断用户插话压过的回复，`ENDED` 提交用户话轮交给模型。
不传 `vad` 时由 provider 决定，agent 只响应其上报。

## 已知限制（v2.0.8+）

- 工具 schema 与 instructions 仅在 `connect()` 时发送一次；会话中途激活工具组 / 安装
  skill（ResetTools、meta tool）要等**下一次 connect**（重连）才生效。
