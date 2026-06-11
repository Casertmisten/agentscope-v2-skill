---
name: agentscope-dev
description: |
  AgentScope 2.0 (agentscope-ai) 多智能体框架开发指南。当用户提到 AgentScope、agentscope、
  多智能体开发、基于 AgentScope 构建 agent/智能体应用时使用此 skill。即使用户只是说"写个 agent"、
  "多智能体"、"帮我用 agentscope"也应触发。注意：这是 agentscope-ai/agentscope v2 版本，
  与旧版 modelscope/agentscope 的 API 完全不同（无 memory/pipeline/formatter 模块）。
  涵盖：Agent 创建、Credential/Model 配置、Toolkit/ToolBase 工具注册、MCPClient 集成、
  AgentState 状态管理、Event 事件系统、Permission 权限、ToolGroup、Skill 技能系统等。
---

# AgentScope 2.0 开发指南 (agentscope-ai)

AgentScope 2.0 是完全重构的版本，API 与 1.x (modelscope/agentscope) 不兼容。

**核心特性**：事件驱动架构、权限系统、上下文自动压缩、工具组管理、Skill 技能系统、MCP 统一客户端、
REST+SSE 智能体服务、Docker/E2B 工作区。

**安装**：`pip install agentscope`（Python >= 3.10）

## 架构概览

```
Agent (单一类，reply 返回事件流)
 ├── Credential        → API 认证（OpenAI/Anthropic/DashScope/Gemini/Ollama...）
 ├── Model (ChatModelBase) → LLM 调用
 ├── Toolkit           → 管理 ToolBase 工具 + MCPClient + Skill
 ├── AgentState        → 状态（context/summary/permission/task）
 └── Config
      ├── ContextConfig    → 上下文压缩配置
      ├── ReActConfig      → 推理-行动循环配置
      └── ModelConfig      → 重试/fallback 模型配置
```

**一切皆异步** + **事件驱动**：`agent.reply()` 返回 `AsyncGenerator[AgentEvent]`。

## ⚠️ v2 vs v1 关键区别

| 概念 | v1 (modelscope) | v2 (agentscope-ai) |
|---|---|---|
| Agent | AgentBase/ReActAgentBase/ReActAgent | 单一 `Agent` 类 |
| 记忆 | MemoryBase/InMemoryMemory/RedisMemory | `AgentState.context` (list[Msg]) |
| 格式化器 | FormatterBase/ChatFormatter 等 | 内嵌在 model 实现，无需手动选 |
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
from agentscope.model import OpenAIChatModel
from agentscope.message import UserMsg
from agentscope.tool import Toolkit
from agentscope.state import AgentState

async def main():
    agent = Agent(
        name="Assistant",
        sys_prompt="你是一个有用的助手。",
        credential=OpenAICredential(api_key="sk-xxx"),
        model="gpt-4o",
        toolkit=Toolkit(),
    )

    # reply 返回事件流
    msg = UserMsg("user", "你好！")
    async for event in agent.reply(msg):
        print(event.type, event)

asyncio.run(main())
```

## 核心组件速查

| 需要做什么 | 参考文件 |
|---|---|
| 配置模型和认证 | [references/models.md](references/models.md) |
| 创建消息和内容块 | [references/messages.md](references/messages.md) |
| 注册工具 (ToolBase) | [references/tools.md](references/tools.md) |
| 集成 MCP | [references/tools.md](references/tools.md) |
| 管理状态 (AgentState) | [references/state.md](references/state.md) |
| Agent 配置和事件 | [references/agent-events.md](references/agent-events.md) |
| 权限和工具组 | [references/permissions.md](references/permissions.md) |

## Credential 体系

v2 使用 Credential 对象封装 API 认证，每个提供商有自己的 Credential 类：

```python
from agentscope.credential import (
    OpenAICredential,
    AnthropicCredential,
    DashScopeCredential,
    GeminiCredential,
    OllamaCredential,
)

# 各提供商的 credential
credential = OpenAICredential(api_key="sk-xxx")
credential = AnthropicCredential(api_key="sk-ant-xxx")
credential = DashScopeCredential(api_key="ds-xxx")
```

## Agent 创建

```python
from agentscope.agent import Agent, ContextConfig, ReActConfig, ModelConfig

agent = Agent(
    name="Jarvis",
    sys_prompt="你是一个有用的助手。",
    credential=credential,
    model="gpt-4o",                    # 模型名称
    toolkit=toolkit,                   # 工具模块
    state=AgentState(),                # 可选，传入已有状态
    context_config=ContextConfig(      # 上下文压缩配置
        trigger_ratio=0.8,
        reserve_ratio=0.1,
        tool_result_limit=3000,
    ),
    react_config=ReActConfig(          # ReAct 循环配置
        max_iters=20,
        stop_on_reject=False,
    ),
    model_config=ModelConfig(          # 模型重试配置
        max_retries=0,
        fallback_model=None,
    ),
)
```

## 事件系统

`reply()` 返回类型化的事件流，覆盖智能体执行的每一步：

```python
async for event in agent.reply(user_msg):
    match event.type:
        case "REPLY_START": ...
        case "TEXT_BLOCK_START": ...
        case "TEXT_BLOCK_DELTA": print(event.delta, end="")  # 流式文本
        case "TEXT_BLOCK_END": ...
        case "THINKING_BLOCK_DELTA": ...   # 推理过程
        case "TOOL_CALL_START": ...        # 工具调用开始
        case "TOOL_CALL_DELTA": ...        # 工具参数增量
        case "TOOL_RESULT_START": ...      # 工具执行开始
        case "TOOL_RESULT_TEXT_DELTA": ... # 工具结果增量
        case "TOOL_RESULT_END": ...        # 工具执行结束
        case "REQUIRE_USER_CONFIRM": ...   # 需要用户确认
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
from agentscope.state import AgentState

state = AgentState()
state.context      # list[Msg] — 对话上下文
state.summary      # str | list[TextBlock|DataBlock] — 压缩摘要
state.session_id   # 会话 ID
state.tool_context # 工具缓存和激活的工具组
state.tasks_context # 任务列表
state.permission_context # 权限上下文
```
