---
name: agentscope-dev
description: |
  AgentScope 2.0 (agentscope-ai) 多智能体框架开发指南。当用户提到 AgentScope、agentscope、
  多智能体开发、基于 AgentScope 构建 agent/智能体应用时使用此 skill。即使用户只是说"写个 agent"、
  "多智能体"、"帮我用 agentscope"也应触发。注意：这是 agentscope-ai/agentscope v2 版本，
  与旧版 modelscope/agentscope 的 API 完全不同（无 memory/pipeline/formatter 模块）。
涵盖：Agent 创建、Credential/Model 配置、Toolkit/ToolBase 工具注册（含 ToolMiddlewareBase 工具级中间件）、
MCPClient 集成、AgentState 状态管理、Event 事件系统、Permission 权限、ToolGroup、Skill 技能系统、
Middleware 中间件（含 TTSMiddleware / ReplyBudgetControlMiddleware / Mem0Middleware）、
Workspace 工作区、Embedding/TTS 多模态模型、SubAgentTemplate 子智能体模板、App 服务化、
set_id_factory 全局 ID 工厂等。
---

# AgentScope 2.0 开发指南 (agentscope-ai)

AgentScope 2.0 是完全重构的版本，API 与 1.x (modelscope/agentscope) 不兼容。当前文档对应 v2.0.3。

**核心特性**：事件驱动架构、权限系统、上下文自动压缩、工具组管理、Skill 技能系统、MCP 统一客户端、
REST+SSE 智能体服务、Docker/E2B 工作区、Middleware 中间件（含 TTSMiddleware / ReplyBudgetControlMiddleware /
Mem0Middleware）、工具级洋葱中间件（ToolMiddlewareBase）、Embedding/TTS 多模态模型、SubAgentTemplate
子智能体模板、Omni 模型音频流、可配置 ID 工厂（set_id_factory）。

**安装**：`pip install agentscope`（Python >= 3.10）

## 架构概览

```
Agent (单一类，reply_stream 返回事件流，reply 返回最终消息)
 ├── ChatModelBase     → LLM 调用（通过 credential.get_chat_model_class() 创建）
 ├── Toolkit           → 管理 ToolBase 工具 + MCPClient + Skill
 │   └── ToolBase 子类 → 每个工具可挂 ToolMiddlewareBase 洋葱中间件
 ├── AgentState        → 状态（context/summary/permission/task）
 ├── Middlewares       → 中间件链（on_reply/on_reasoning/on_acting/on_model_call/on_system_prompt）
 ├── Offloader         → 上下文卸载（可指向 Workspace）
 └── Config
      ├── ContextConfig    → 上下文压缩配置
      ├── ReActConfig      → 推理-行动循环配置
      └── ModelConfig      → 重试/fallback 模型配置
```

**一切皆异步** + **事件驱动**：`agent.reply_stream()` 返回 `AsyncGenerator[AgentEvent]`，`agent.reply()` 返回 `Msg`。

**多模态（v2.0.2+）**：独立的 `embedding` / `tts` / `formatter` 模块；Credential 统一暴露
`get_chat_model_class()` / `get_embedding_model_class()` / `get_tts_model_classes()`；Omni 模型可通过
`DATA_BLOCK_*` 事件流式输出语音，`TTSMiddleware` 可把任意文本回复转语音。
> ⚠️ TTS / Embedding / Omni 音频 / Realtime 属于官方 Voice Agent 路线（roadmap）的进行中方向，
> API 可能随版本变动，使用前请以源码为准。

## ⚠️ v2 vs v1 关键区别

| 概念 | v1 (modelscope) | v2 (agentscope-ai) |
|---|---|---|
| Agent | AgentBase/ReActAgentBase/ReActAgent | 单一 `Agent` 类 |
| 记忆 | MemoryBase/InMemoryMemory/RedisMemory | `AgentState.context` (list[Msg]) |
| 格式化器 | FormatterBase/ChatFormatter 等 | `formatter` 模块存在，但由各 model 实现内部使用，Agent 无需手动选 |
| 管道 | MsgHub/SequentialPipeline/FanoutPipeline | 无，直接用 asyncio 编排 |
| 工具 | 普通函数 + Toolkit | `ToolBase` 子类 + `Toolkit` |
| MCP | HttpStatefulClient/StatelessClient 等 | 统一 `MCPClient` |
| 消息 | Msg(name, role, content) | `UserMsg/AssistantMsg/SystemMsg` 工厂函数 |
| 认证 | 直接传 api_key | `Credential` 体系 |

## 最小示例

```python
import asyncio
from agentscope.agent import Agent
from agentscope.credential import OpenAICredential
from agentscope.message import UserMsg
from agentscope.tool import Toolkit

async def main():
    credential = OpenAICredential(api_key="sk-xxx")
    model = credential.get_chat_model_class()(
        credential=credential,
        model="gpt-4o",
    )

    agent = Agent(
        name="Assistant",
        system_prompt="你是一个有用的助手。",
        model=model,
        toolkit=Toolkit(),
    )

    # 方式 1：流式事件
    msg = UserMsg("user", "你好！")
    async for event in agent.reply_stream(msg):
        print(event.type, event)

    # 方式 2：直接获取最终消息
    final_msg = await agent.reply(msg)

asyncio.run(main())
```

## 核心组件速查

| 需要做什么 | 参考文件 |
|---|---|
| 配置模型和认证 | [references/models.md](references/models.md) |
| Embedding / TTS 多模态模型 (v2.0.2+) | [references/models.md](references/models.md) |
| 创建消息和内容块 | [references/messages.md](references/messages.md) |
| 注册工具 (ToolBase) | [references/tools.md](references/tools.md) |
| 集成 MCP | [references/tools.md](references/tools.md) |
| 管理状态 (AgentState) | [references/state.md](references/state.md) |
| Agent 配置和事件 | [references/agent-events.md](references/agent-events.md) |
| 权限和工具组 | [references/permissions.md](references/permissions.md) |
| 中间件（含 TTSMiddleware）和工作区 | [references/middleware-workspace.md](references/middleware-workspace.md) |
| 服务化与子智能体模板 (SubAgentTemplate) | [references/middleware-workspace.md](references/middleware-workspace.md) |

## Credential 体系

v2 使用 Credential 对象封装 API 认证，每个提供商有自己的 Credential 类：

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

# 各提供商的 credential
credential = OpenAICredential(api_key="sk-xxx")
credential = AnthropicCredential(api_key="sk-ant-xxx")
credential = DashScopeCredential(api_key="ds-xxx")
```

Credential 不仅管理对话模型，还统一暴露多模态能力（v2.0.2+）：

```python
credential.get_chat_model_class()       # 对话模型类
credential.list_models()                # 对话模型卡片
credential.get_embedding_model_class()  # Embedding 模型类（未支持返回 None）
credential.get_tts_model_classes()      # list[TTS 模型类]（未支持返回空列表）
credential.list_tts_models()            # list[TTSModelCard]
```

详见 [references/models.md](references/models.md)。

## Agent 创建

```python
from agentscope.agent import Agent, ContextConfig, ReActConfig, ModelConfig
from agentscope.workspace import LocalWorkspace

# 创建模型实例
credential = OpenAICredential(api_key="sk-xxx")
model = credential.get_chat_model_class()(
    credential=credential,
    model="gpt-4o",
)

agent = Agent(
    name="Jarvis",
    system_prompt="你是一个有用的助手。",   # 注意：是 system_prompt 不是 sys_prompt
    model=model,                        # ChatModelBase 实例
    toolkit=toolkit,                    # 工具模块
    middlewares=[],                     # 中间件列表（可选）
    state=AgentState(),                 # 可选，传入已有状态
    offloader=None,                     # Offloader 或 Workspace（可选）
    context_config=ContextConfig(       # 上下文压缩配置
        trigger_ratio=0.8,
        reserve_ratio=0.1,
        tool_result_limit=3000,
    ),
    react_config=ReActConfig(           # ReAct 循环配置
        max_iters=20,
        stop_on_reject=False,
    ),
    model_config=ModelConfig(           # 模型重试配置
        max_retries=0,
        fallback_model=None,
    ),
)
```

## Agent 方法

```python
# 流式事件（推荐用于 UI 渲染）
async for event in agent.reply_stream(user_msg):
    print(event.type)

# 直接获取最终消息（推荐用于后端）
final_msg: Msg = await agent.reply(user_msg)

# 接收外部观察消息（不触发推理）
await agent.observe(other_agent_msg)

# 手动触发上下文压缩
await agent.compress_context()
```

## 事件系统

`reply_stream()` 返回类型化的事件流，覆盖智能体执行的每一步：

```python
async for event in agent.reply_stream(user_msg):
    match event.type:
        case "REPLY_START": ...
        case "MODEL_CALL_START": ...      # 模型调用开始
        case "TEXT_BLOCK_START": ...
        case "TEXT_BLOCK_DELTA": print(event.delta, end="")  # 流式文本
        case "TEXT_BLOCK_END": ...
        case "THINKING_BLOCK_START": ...  # 推理块开始
        case "THINKING_BLOCK_DELTA": ...  # 推理过程
        case "THINKING_BLOCK_END": ...
        case "DATA_BLOCK_START": ...      # 数据流开始（如 omni 模型音频、TTS 音频）
        case "DATA_BLOCK_DELTA": ...      # 数据增量（base64，按 block_id 串接）
        case "DATA_BLOCK_END": ...
        case "TOOL_CALL_START": ...       # 工具调用开始
        case "TOOL_CALL_DELTA": ...       # 工具参数增量
        case "TOOL_CALL_END": ...
        case "TOOL_RESULT_START": ...     # 工具执行开始
        case "TOOL_RESULT_TEXT_DELTA": ...# 工具结果增量
        case "TOOL_RESULT_DATA_DELTA": ...# 工具结果数据增量
        case "TOOL_RESULT_END": ...       # 工具执行结束
        case "MODEL_CALL_END": ...        # 模型调用结束（含 token 用量）
        case "EXCEED_MAX_ITERS": ...      # 超过最大迭代次数
        case "REQUIRE_USER_CONFIRM": ...  # 需要用户确认
        case "REQUIRE_EXTERNAL_EXECUTION": ...  # 需要外部执行
        case "REPLY_END": ...
```

## 消息创建

```python
from agentscope.message import UserMsg, AssistantMsg, SystemMsg, Msg

# 工厂函数（推荐）
msg = UserMsg("user", "你好")
msg = AssistantMsg("Jarvis", "你好！我能帮你什么？")
msg = SystemMsg("system", "系统提示")

# content 自动包装：字符串 → [TextBlock(text=...)]
```

## Toolkit 和 ToolBase

```python
from agentscope.tool import Toolkit, ToolGroup
from agentscope.tool import Bash, Read, Write, Edit, Glob, Grep  # 内置工具
from agentscope.tool import TaskCreate, TaskGet, TaskList, TaskUpdate  # 任务工具

# 基本用法
toolkit = Toolkit(tools=[Bash(), Read(), Write()])

# 带 MCP
toolkit = Toolkit(
    tools=[Bash()],
    mcps=[mcp_client],
)

# 带工具组
toolkit = Toolkit(
    tools=[Bash()],  # basic 组
    tool_groups=[
        ToolGroup(
            name="browser",
            description="网页浏览工具",
            instructions="先 navigate 再操作",
            tools=[Navigate(), Click()],
        ),
    ],
)
```

## MCPClient

```python
from agentscope.mcp import MCPClient
from agentscope.mcp import StdioMCPConfig, HttpMCPConfig

# 有状态 HTTP 连接
client = MCPClient(
    name="weather",
    is_stateful=True,
    mcp_config=HttpMCPConfig(url="https://api.weather.com/mcp"),
)
await client.connect()
tools = await client.list_tools()
# ...
await client.close()

# 无状态 HTTP 连接（无需 connect/close）
client = MCPClient(
    name="search",
    is_stateful=False,
    mcp_config=HttpMCPConfig(url="https://search.example.com/mcp"),
)
tools = await client.list_tools()

# StdIO 连接（必须有状态）
client = MCPClient(
    name="filesystem",
    is_stateful=True,
    mcp_config=StdioMCPConfig(command="mcp-server-filesystem"),
)
```

## AgentState

```python
from agentscope.state import AgentState, Task, TaskContext

state = AgentState()
state.context      # list[Msg] — 对话上下文
state.summary      # str | list[TextBlock|DataBlock] — 压缩摘要
state.session_id   # 会话 ID
state.tool_context # ToolContext — 工具缓存和激活的工具组
state.tasks_context # TaskContext — 任务列表
state.permission_context # PermissionContext — 权限上下文
state.middle_context # dict[str, Any] — 中间件跨 reply 存取数据
```

## Workspace（工作区）

```python
from agentscope.workspace import LocalWorkspace, DockerWorkspace, E2BWorkspace

# 本地工作区
workspace = LocalWorkspace(
    workdir="/path/to/workspace",
    default_mcps=[mcp_client],
    skill_paths=["/path/to/skills"],
)

async with workspace:
    # workspace 自动 initialize
    agent = Agent(
        name="Assistant",
        system_prompt="...",
        model=model,
        toolkit=Toolkit(),
        offloader=workspace,  # 启用上下文卸载
    )
```

## 全局配置

### set_id_factory — 自定义 ID 生成策略（v2.0.3+）

所有 AgentScope 实体（Agent / Session / Credential / Msg 等）的 ID 默认用 `uuid.uuid4().hex`。
顶层 `agentscope.set_id_factory()` 可在启动时替换为任意无参 callable，用于接入 UUIDv7、
雪花 ID、带前缀的业务 ID 等：

```python
import agentscope

# 换成 UUIDv7（需自行安装/实现 uuid7）
agentscope.set_id_factory(lambda: uuid7().hex)
```

注意：
- **安全相关 token 不受影响**（如 gateway token、Redis 锁 token），始终用 `uuid.uuid4().hex`。
- 只需在程序启动时调用一次，全局生效。
