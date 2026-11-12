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

想要全屏终端语音界面（转写显示 + HITL 控件，音频由 transport 播放）时，用
`agentscope.tui.launch_realtime_ui(agent, transport)`（v2.0.8+ 主干，
`pip install "agentscope[tui]"`），见 [agent-events.md](agent-events.md)。

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
    delegation_agent_factory=None, # 可选：GPT-Live 委托后端工厂，v2.0.12+，见上节
)
```

**三个生命周期刻意分离**：agent 拥有模型会话与状态；transport 归创建者所有；
`reply_stream` 只是借用二者。因此客户端断开重连不丢模型会话，模型会话静默超时
也会在用户说下一句话或 `send()` 发送文本时自动重连（指数退避；重连时已发生的对话
以 transcript 形式拼进 instructions 恢复上下文，上一会话遗留的终止事件不会重放）。

### 方法

| 方法 / 属性 | 说明 |
|---|---|
| `async with agent:` / `connect()` / `close()` | 打开/关闭模型会话。`connect` 幂等，重连安全 |
| `reply_stream(transport)` | 挂接一个传输并异步迭代 `AgentEvent`，直到该 transport 的输入结束 |
| `send(inputs)` | 非音频输入的唯一入口：`str`/`Msg` 文本轮、`UserConfirmResultEvent`（HITL 应答）、`ExternalExecutionResultEvent`（外部执行结果，v2.0.12+ 委托任务路由）、`UserInterruptEvent`（打断）。文本轮会先打断进行中的回复 |
| `interrupt()` | 等价于 `send(UserInterruptEvent())`，即用户按"停止"（v2.0.8+ 事件自动带当前 `reply_id`） |
| `last_turn_metrics` | `TurnMetrics`：最近一轮的时延分解 |

`reply_stream` 的语义细节：

- transport 是**借用**：必须已 `start()`，且不会被它关闭——调用方可以保留复用；
- 流在 transport 输入结束时结束（如客户端断开），**不是**某个回复结束——一次调用里有多个回复；
- 换新 transport 再次调用即可续接同一模型会话；
- 同一 agent 同时只允许一个活跃的 `reply_stream`（再调用抛 `RuntimeError`）；
- 提前 `break` 不会立即清理，需要即时清理时用 `contextlib.aclosing` 包住生成器；
- transport 与模型的输入/输出采样率必须一致，重采样是 transport 的职责（agent 原样转发）；
- v2.0.12+ 流结束（正常结束/提前关闭/取消）时丢弃"尚未绑定 provider item id 且用户还没说完"的本地
  用户话轮，避免它们抢走下一次 `reply_stream` 的首条转写；已绑定或已说完的话轮保留，迟到的转写
  仍落到原 reply。

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

统一构造签名（各 provider 一致）：

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

模型分两种交互模式（`InteractionMode`，v2.0.12+；卡片 `interaction_mode` 声明须与适配器
一致否则加载抛 `ValueError`）：`TURN_BASED`（默认，请求/应答边界）与 `FULL_DUPLEX`
（连续时间轴，无轮次概念）。

### 内置适配器与模型

```python
from agentscope.realtime import (
    DashScopeRealtimeModel,        # Qwen-Omni 系列（WebSocket）
    DashScopeAudioRealtimeModel,   # Qwen-Audio-3.0-Realtime
    OpenAIRealtimeModel,           # GPT Realtime API
    OpenAILiveModel,               # GPT-Live（全双工 + 任务委托，v2.0.12+）
    GeminiRealtimeModel,           # Gemini Live API
    XAIRealtimeModel,              # Grok Voice
)
```

| 适配器 | credential | 模型（yaml 卡片） |
|---|---|---|
| `DashScopeRealtimeModel` | `DashScopeCredential` | `qwen-omni-turbo-realtime`、`qwen3-omni-flash-realtime`、`qwen3.5-omni-flash-realtime`、`qwen3.5-omni-plus-realtime`、`qwen3.8-omni-flash-realtime`（v2.0.10+，支持 image/jpeg 输入与工具） |
| `DashScopeAudioRealtimeModel` | `DashScopeCredential` | `qwen-audio-3.0-realtime-flash`、`qwen-audio-3.0-realtime-plus`、`qwen-audio-3.1-realtime-plus`（v2.0.10+） |
| `OpenAIRealtimeModel` | `OpenAICredential` | `gpt-realtime-1.5`、`gpt-realtime-2`、`gpt-realtime-2.1`、`gpt-realtime-2.1-mini` |
| `OpenAILiveModel`（v2.0.12+） | `OpenAICredential` | `gpt-live-1`（全双工，`delegation_backend: session_chat`，工具经会话 ChatAgent 委托执行，见下节） |
| `GeminiRealtimeModel` | `GeminiCredential` | `gemini-2.5-flash-native-audio-preview-12-2025`、`gemini-3.1-flash-live-preview`、`gemini-3.8-live`、`gemini-3.8-live-extended-thinking`（v2.0.12+，见下节） |
| `XAIRealtimeModel` | `XAICredential` | `grok-voice-latest`、`grok-voice-think-fast-2.0` |

`DashScopeAudioRealtimeModel` 是 `DashScopeRealtimeModel` 的子类，差异：
`supports_text_input=True`（可在会话中注入文本轮）、话轮检测用 `smart_turn`
端点（`turn_detection: "server_vad"|"smart_turn"|"none"`，`voiceprint_audio_urls`
声纹参考仅 `smart_turn` 使用）；请求回复时带 `modalities: ["text", "audio"]` 覆盖
（v2.0.10+，基类则发裸 `response.create`）。v2.0.10 起协议细节：音频格式统一
`input_audio_format`/`output_audio_format: "pcm"`（不再带采样率后缀）、输入转写
默认模型 `qwen3-asr-flash-realtime`（原 `gummy-realtime-v1`）。

各适配器的 `Parameters`（音色、VAD 阈值、`max_history_turns` 等）不同，
通过 `RealtimeModelCard.parameter_schema` 或 `ModelClass.list_models()` 查看字段。

### GPT-Live 与 Chat Agent 委托（v2.0.12+）

`OpenAILiveModel`（GPT-Live，服务端 WebSocket）是唯一的全双工适配器：连续会话时间轴、
无轮次边界（`push_text`/`commit_turn`/`request_response` 等轮次方法均抛
`NotImplementedError`）；转写与音频带时间轴事件（`TimelineTranscriptDeltaEvent` 带
`role`/`start_ms`/`end_ms`）。`Parameters`：`voice`（默认 `"marin"`）、`store`、
`startup_timeout_s`/`close_timeout_s`。用法上 `interrupt()` 让前端停止说话但**不取消**
后台任务；`usage_seconds`/`context_usage_ratio` 暴露用量。

**Chat Agent Delegation**——GPT-Live 服务端判断需要深度任务时下发 `target="client"`
的委托（`DelegationCreatedEvent`），由 `RealtimeAgent` 用一个普通 `Agent` 执行：

```python
from agentscope.agent import Agent, RealtimeAgent
from agentscope.model import OpenAIChatModel
from agentscope.realtime import OpenAILiveModel

live = OpenAILiveModel("gpt-live-1", credential)
backend = OpenAIChatModel(credential=credential, model="gpt-5.2")

def _factory(state):                       # state 是隔离副本（含会话历史，剔除 Live 转写）
    return Agent(name="Friday",            # ⚠️ 必须与 RealtimeAgent 同名，否则委托失败
                 system_prompt="结果简洁，适合口播。", model=backend, state=state)

agent = RealtimeAgent(
    name="Friday", system_prompt="简短作答，深度问题交给任务后端。",
    model=live,
    delegation_agent_factory=_factory,     # 不传则客户端委托直接失败并告知模型
)
```

机制要点：

- 委托任务**串行执行**（同时一个 active，其余排队），每个带 300s 输入超时；后端 agent 的
  工具卡片事件默认**隐藏**（缓冲），仅当产出 `RequireUserConfirmEvent` /
  `RequireExternalExecutionEvent`（HITL/外部执行）时才对事件流可见；
- 用户确认/提交经 `send()` 回传（`UserConfirmResultEvent` / `ExternalExecutionResultEvent`），
  按 tool_call id 路由到对应委托任务继续驱动；不匹配发 `external_execution_result_rejected`
  自定义事件；
- 最终结果文本经 `send_delegation_result()` 回传给 Live 口播（长文按 ≤400 字符切块）；
  工具状态投影回主 `state`；持久化检查点以 `"delegation_checkpoint"` 自定义事件发出；
- 全双工模型**不支持本地 VAD**（构造时同时传 `vad` 抛 `ValueError`）；
  `TurnAggregator` 在全双工路径不消费；完整示例见上游 `examples/realtime/openai_live_local.py`。

### Gemini Live Extended Thinking（v2.0.12+）

`gemini-3.8-live-extended-thinking` 模型卡（`supports_extended_thinking: true`）开启后台
扩展思考：推理在模型后台持续进行，工具变为非阻塞，响应完成改由 `interactionStatus == "IDLE"`
判定（`turnComplete` 不再关闭响应）。

```python
from agentscope.credential import GeminiCredential
from agentscope.realtime import GeminiRealtimeModel

model = GeminiRealtimeModel(
    "gemini-3.8-live-extended-thinking",
    GeminiCredential(api_key="xxx"),
    parameters=GeminiRealtimeModel.Parameters(
        thinking_level="high",   # low | medium | high（后台推理深度；不设则不发 thinkingConfig）
    ),
)
```

行为要点：

- `Parameters.thinking_level` 仅支持 `supports_extended_thinking=True` 的模型——
  其他 Gemini Live 模型设置时构造即抛 `ValueError`（"thinking_level requires Gemini Live
  Extended Thinking."）；
- 所有工具声明自动加 `"behavior": "NON_BLOCKING"`：`ToolCallEvent.non_blocking=True` 时
  工具立即执行且不自动 `request_response`，回复要等**后台工具全部完成 + 模型 IDLE** 才结束；
- 用户输入的部分转写以新事件 `InputTranscriptionDeltaEvent`（`item_id`/`delta`/`response_id`）
  流出，定稿（`InputTranscriptionEvent`）时替换 `state.context` 中的占位 `UserMsg`、
  被丢弃时移除（checkpoint 快照剔除未定稿的部分转写消息）；
- 本地 VAD 检测到新用户话轮时调用 `model.start_turn()`；回复被打断后陈旧工具结果经
  `model.push_tool_result_silently()` 送回（Gemini wire 上加 `scheduling: "SILENT"`，
  不打断进行中的生成）；
- 内部走 `v1alpha` WebSocket 端点（普通 Live 模型仍走 `v1beta`）；带 `thought` 标记的
  思维摘要 part 不当作语音转录处理。

### RealtimeModelCard

YAML 模型卡片（`realtime/_<provider>/_models/*.yaml`，可用 `custom_yaml_dir` 扩展），
关键字段：`name` / `label` / `status`、`input_types` / `output_types`、
`input_sample_rate` / `output_sample_rate`、`supports_tools`、
`interaction_mode`（`turn_based`（默认）| `full_duplex`，v2.0.12+，须与适配器一致）、
`delegation_backend`（`none`（默认）| `session_chat`，v2.0.12+，后者表示工具经会话
ChatAgent 委托执行）、
`max_context_tokens` / `max_audio_turns` / `max_audio_duration_s` /
`max_session_duration_s`、`parameter_schema`（可调参数的 JSON Schema，供 UI 展示）、
`release_date`（模型发布日期，v2.0.12+）、
`supports_extended_thinking`（是否支持后台扩展思考 + 非阻塞工具 + interactionStatus 完成
判定，v2.0.12+，Gemini 3.8 Live Extended Thinking）。

### 自定义 realtime 模型

继承 `RealtimeModelBase` 并实现抽象方法：

```python
class MyRealtimeModel(RealtimeModelBase):
    type = "my_realtime"                          # 适配器标识，写进每张卡片
    interaction_mode = InteractionMode.TURN_BASED # TURN_BASED / FULL_DUPLEX（v2.0.12+）
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
    def interrupt_speech(self): ...                # 可选（v2.0.12+）：全双工打断语音，
                                                   # 默认调 cancel_response
    async def send_delegation_result(              # 可选（v2.0.12+）：回传委托任务结果，
        self, delegation_id, content): ...         # 默认抛 NotImplementedError
    async def push_tool_result_silently(           # 可选（v2.0.12+）：不打断当前生成的
        self, block: ToolResultBlock): ...         # 迟到工具结果，默认委托 push_tool_result
    async def start_turn(self): ...                # 可选（v2.0.12+）：本地 VAD 检测到新
                                                   # 用户话轮时通知模型，默认 no-op
```

`TruncationSupport` 描述被打断话轮的纠正方式：`SERVER`（provider 自纠，`truncate` 空操作）、
`EXPLICIT`（接受显式 truncate 帧）、`NONE`。
`ModelDisconnectedError`（`ConnectionError` 子类）表示 provider 关闭了会话——
下一句用户音频会自动重连。

`connect` 的 `**kwargs`：`turn_detection_disabled=True` 要求 provider 关闭自身话轮检测
（调用方自跑 VAD 时使用）；部分 provider 还接受 resumption 句柄。

### 模型事件（ModelEvent）

`agentscope.realtime` 的模型侧事件，供自定义适配器/传输层使用（agent 已内置消费）：

`ModelEvent` 基类及 `SessionStartedEvent` / `SessionClosedEvent`（v2.0.12+）、
`SessionUsageEvent`（累计秒数 + `context_usage_ratio`，v2.0.12+）、
`SessionEndedEvent`、`SpeechStartedEvent`、`SpeechEndedEvent`、
`InputTranscriptionEvent`（用户转写）、`InputTranscriptionDeltaEvent`（未定稿的输入转写增量，
`item_id`/`delta`/`response_id`，v2.0.12+）、`ResponseCreatedEvent`、`AudioDeltaEvent`（生成的音频块）、
`TranscriptDeltaEvent`（助手转写增量）、`TimelineAudioDeltaEvent` /
`TimelineTranscriptDeltaEvent`（全双工带时间轴的音频块/字幕，v2.0.12+）、
`DelegationCreatedEvent`（GPT-Live 委托开始：`delegation_id`/`target`/`response_id`/`offset_ms`，v2.0.12+）、
`DelegatedResponseEvent`（GPT-Live 委托的嵌套 Responses 事件，v2.0.12+）、
`ToolCallEvent`（v2.0.12+ 新增 `non_blocking: bool`——provider 在继续当前响应的同时立即
执行该工具，如 Gemini Extended Thinking 的 NON_BLOCKING 工具）、
`ResponseDoneEvent`、`ModelErrorEvent`。

## 传输层 (Transport)

```python
from agentscope.realtime import (
    TransportBase,          # 抽象基类：start/close/incoming/send_audio/clear_audio/playout
    LocalAudioTransport,    # 本地声卡（sounddevice）
    AudioFrame,             # PCM16 单声道音频帧（上游用户音频）
    ControlFrame,           # 控制帧：type + data
    ControlFrameType,       # TEXT / USER_CONFIRM / INTERRUPT / EXTERNAL_EXECUTION_RESULT（v2.0.12+）
    TransportFrame,         # AudioFrame | ControlFrame 联合
    PlayoutPosition,        # 播放进度（item_id + 已播放毫秒/文本前缀）
)
```

`TransportBase` 的完整接口（v2.0.10 Frame 化）：

- `input_sample_rate` / `output_sample_rate` — 声明两侧采样率，须与模型一致；
- `start()` / `close()`（也可 `async with`）— 生命周期归 transport 的创建者；
- `incoming()` — `AsyncIterator[TransportFrame]`：上游帧（用户音频 + 控制信令）
  直到 transport 关闭；
- `send_audio(pcm, item_id)` — 播放助手音频（归属到 `item_id` 便于播放进度归因）；
- `clear_audio()` — 清空待播队列，返回打断前的 `PlayoutPosition`；
- `playout()` — 不打断地查当前播放位置。

`ControlFrameType` 各成员与 `RealtimeAgent` 的公共方法一一对应——远端客户端
（浏览器）通过 datachannel 发控制帧与 Python 调用方直接调方法走同一条语义路径：
`TEXT`（打字的用户轮 → `send()`）、`USER_CONFIRM`（HITL 权限确认 →
`UserConfirmResultEvent`）、`INTERRUPT`（按停止键 → `interrupt()`；区别于说话
打断，后者由 provider/VAD 话轮检测触发）、`EXTERNAL_EXECUTION_RESULT`（外部执行
结果 → `ExternalExecutionResultEvent`，v2.0.12+ 委托任务路由）。

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

v2.0.8+：transport 在同一进程内重启（stop 后再次 start，如重连新会话）时会重置
输入队列与播放游标，上一会话的哨兵和未播完音频不会泄漏进新会话。

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
全双工模型（如 GPT-Live）无轮次边界，**不支持本地 VAD**，同时传入抛 `ValueError`（v2.0.12+）。

## 已知限制（v2.0.8+）

- 工具 schema 与 instructions 仅在 `connect()` 时发送一次；会话中途激活工具组 / 安装
  skill（ResetTools、meta tool）要等**下一次 connect**（重连）才生效。
- GPT-Live（v2.0.12+）：`supports_tools: false`——connect 时不发送工具 schema，
  工具全部经委托后端的 ChatAgent 执行；全双工不支持文本轮（`send()` 文本轮抛
  `NotImplementedError`）。

## 服务化实时语音（v2.0.10+ 主干）

服务化（`create_app`）后浏览器可直接与持久化 session 语音对话，WebRTC 协商端点在
`/realtime` 路由（详见 [middleware-workspace.md](middleware-workspace.md)）：

- `GET /realtime/models?provider=openai_credential` — 列该 provider 的 realtime 模型卡
  （`credential.list_realtime_models()`）；
- `GET /realtime/config` — 返回部署提供的 ICE servers（浏览器 RTCPeerConnection 配置；
  经 `create_app(realtime_ice_servers=[...])` 或环境变量 `AGENTSCOPE_REALTIME_ICE_SERVERS`
  以 JSON 数组配置，未配置时为空列表，浏览器走默认 STUN）；
- `POST /realtime/sessions/{session_id}/offer` — body `{agent_id, sdp, type: "offer"}`，
  服务端经 `RealtimeService` 把持久化 session 组装成带完整 toolkit 的 `RealtimeAgent`
  （复用 session 锁，与文本对话互斥），aiortc 协商后返回 `{sdp, type: "answer"}`。

前提：session 已通过 `PATCH /sessions/{id}` 配置 `realtime_model_config`
（`type`/`credential_id`/`model`/`parameters`，引用已存的 credential）；未配置或
`turn_detection="none"`（浏览器语音模式不支持）返回 409；同一 session 重复 offer
会先关闭旧连接。WebRTC 栈（aiortc）随 `agentscope[service]` extra 安装。

GPT-Live（v2.0.12+）：`openai_credential` 的 `/realtime/models` 会列出 `gpt-live-1`。
选它做 `realtime_model_config` 时服务端自动用该 session 当前配置的 Chat 模型、工具与
中间件组装委托后端（Web UI 语音会话自动复用 session 的 Chat 能力）；offer 前置两个
409：session 存在未完成的 Chat 确认/外部执行工具调用，或未配置 `chat_model_config`。
