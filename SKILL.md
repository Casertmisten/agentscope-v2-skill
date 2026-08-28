---
name: agentscope-dev
description: |
  AgentScope 2.0 (agentscope-ai) 多智能体框架开发指南。当用户提到 AgentScope、agentscope、
  多智能体开发、基于 AgentScope 构建 agent/智能体应用时使用此 skill。即使用户只是说"写个 agent"、
  "多智能体"、"帮我用 agentscope"也应触发。注意：这是 agentscope-ai/agentscope v2 版本，
  与旧版 modelscope/agentscope 的 API 完全不同（无 memory 模块，pipeline 为 v2 全新设计）。
涵盖：Agent 创建、Credential/Model 配置、Toolkit/ToolBase 工具注册（含 ToolMiddlewareBase 工具级中间件、Windows PowerShell 工具）、
MCPClient 集成、AgentState 状态管理、Event 事件系统、Permission 权限、ToolGroup、Skill 技能系统、
Middleware 中间件（含 TTSMiddleware / ReplyBudgetControlMiddleware / TracingMiddleware / Mem0Middleware 跨会话长期记忆 /
ReMeMiddleware 内嵌 ReMe 长期记忆 / AgenticMemoryMiddleware 文件系统长期记忆 / RAGMiddleware 检索增强，及 on_check_permission
权限检查洋葱 hook、on_reply 吞 ReplyEndEvent 续跑回复循环）、Agent 结构化输出（structured_schema Pydantic 模型）、运行时状态注入（时间/任务/上下文压缩感知）、
Workspace 工作区（八种 Workspace 均支持内置工具，Backend 抽象：LocalBackend / DockerBackend / E2BBackend /
K8sBackend / OpenSandboxBackend / DaytonaBackend / AppleContainerBackend / BubblewrapBackend）、Embedding/TTS 多模态模型
（Embedding v2.0.3 重构为泛型基类，dimensions 必填 + FileEmbeddingCache 文件缓存；TTS 含 OpenAITTSModel /
DashScope CosyVoice / GeminiTTSModel 等）、RAG 知识库（agentscope.rag：
KnowledgeBase / QdrantStore / MilvusLiteStore / MongoDBStore / ElasticsearchStore / Parser-Chunker 管线 / RAGMiddleware static+agentic 双模式）、
SubAgentTemplate 子智能体模板（含团队 Leader HITL 事件投影 + AgentInvite 邀请已有 agent）、App 服务化（含 KnowledgeBaseManager /
BlobStore / 内嵌或独立索引 worker / MessageBus 消息总线：
RedisMessageBus / InMemoryMessageBus / Session Status 端点（含 list_sessions 内联 status + 裁剪 context） /
AsyncSQLAlchemyStorage 持久化 / /health 健康检查端点 / workspace 文件读取与状态端点（WorkspaceService +
DirectoryListing + WorkspaceStatus/GitStatus + session cwd 工作目录）/ extra_agent_middlewares 工厂 workspace 参数 /
Anthropic thinking_mode 推理控制 / Channel IM 频道接入钉钉、飞书与 Discord / Skill 按 agent 分区隔离）Agent 中断（UserInterruptEvent）、
回复错误上报（ErrorType 分类 + ReplyFinishedReason.ERROR）、跨用户资源共享（ResourceAccessPolicy 抽象）、
Hub 注册中心（GitHubMCPHub / ClawSkillHub，从 hub 浏览-安装-拉入 workspace）、
终端控制台 console（launch_console 交互式终端对话调试 / ConsoleRenderer 事件流渲染，内置 HITL 工具确认与 Ctrl+C 中断，agent 与 pipeline 均可传入）、
Pipeline 流水线（GoalPipeline 执行者-校验者目标达成循环，对外暴露与 Agent 相同的 reply_stream 事件流）、set_id_factory 全局 ID 工厂等。
---

# AgentScope 2.0 开发指南 (agentscope-ai)

AgentScope 2.0 是完全重构的版本，API 与 1.x (modelscope/agentscope) 不兼容。当前文档对应本地源码 `2.0.7.post1`（main 分支，v2.0.7 发布后合入的变更标注 v2.0.7.post1+）。

**核心特性**：事件驱动架构、权限系统（含 on_check_permission 中间件 hook、批量确认豁免、
DEFAULT/DONT_ASK 只读快路径）、上下文自动压缩、工具组管理、Skill 技能系统、MCP 统一客户端、
REST+SSE 智能体服务（MessageBus 消息总线，含 InMemoryMessageBus 单节点轻量选项；
AsyncSQLAlchemyStorage 多数据库持久化存储；`/health` 健康检查端点 v2.0.6+；
workspace 产物文件读取端点 v2.0.6+）、八种 Workspace 均支持
内置工具在容器/云沙箱执行（LocalBackend / DockerBackend / E2BBackend / K8sBackend / OpenSandboxBackend /
DaytonaBackend / AppleContainerBackend / BubblewrapBackend）、Middleware 中间件（含
TTSMiddleware / ReplyBudgetControlMiddleware / TracingMiddleware OpenTelemetry 追踪 /
Mem0Middleware 跨会话长期记忆 / ReMeMiddleware 内嵌 ReMe 长期记忆 / AgenticMemoryMiddleware
文件系统长期记忆（v2.0.4+）/ RAGMiddleware 检索增强）、工具级洋葱中间件（ToolMiddlewareBase）、
Embedding/TTS 多模态模型（Embedding v2.0.3 重构为泛型基类，dimensions 为必填契约参数 + 多模态路由 + FileEmbeddingCache 文件缓存；
TTS 含 OpenAITTSModel / DashScopeTTSModel / DashScopeRealtimeTTSModel / DashScopeCosyVoiceTTSModel / GeminiTTSModel）、
RAG 知识库（agentscope.rag：KnowledgeBase + QdrantStore + MilvusLiteStore（v2.0.4+）+ MongoDBStore（v2.0.4+）
+ ElasticsearchStore（v2.0.5+）+ Parser/Chunker 索引管线 + RAGMiddleware static/agentic 双模式）、
SubAgentTemplate 子智能体模板（含团队 Leader HITL 事件投影 + AgentInvite 邀请已有 agent 入队）、
服务化知识库（KnowledgeBaseManager + LocalBlobStore/S3BlobStore + 内嵌或独立索引 worker）、
Session Status 统一状态查询端点（v2.0.4+）、Agent 中断机制（UserInterruptEvent，v2.0.4+）、
回复错误上报（ErrorType 分类 + ReplyFinishedReason.ERROR，v2.0.5+）、
Agent 结构化输出（structured_schema，v2.0.5+）、运行时状态注入（InjectionConfig，v2.0.5+）、
跨用户资源共享（ResourceAccessPolicy 抽象，group/org 场景）、
Workspace 新增 AppleContainerWorkspace / BubblewrapWorkspace（v2.0.5+）、Windows PowerShell 工具（v2.0.5+）、
skill 路径支持 `~` 展开（v2.0.5+）、Hub 注册中心（GitHubMCPHub / ClawSkillHub，v2.0.5+，
从 hub 浏览 MCP/Skill → 安装到个人库 → 拉入 workspace）、
服务化 `/health` 健康检查端点（v2.0.6+，按组件就绪探测，sub-app 误挂时返回 503）、
workspace 产物文件读取与状态端点（`GET /workspace/directories` 返回 `DirectoryListing` 含解析后绝对路径 /
`GET /workspace/status` 返回 `WorkspaceStatus`（workdir + cwd + `GitStatus`） / `/workspace/files` + download-token，
背后统一收敛到 `WorkspaceService` 服务层，v2.0.6+）、
Session 工作目录（`SessionConfig.cwd` 锚点 + `GET /workspace/status` git 状态，v2.0.6+）、
`GET /sessions` 返回的 `SessionView` 内联 `status` 字段并裁剪 `context/summary/tool_context`（`is_running` deprecated，v2.0.6+）、
`PATCH /sessions/{id}` 在 run 持有 session 时返回 409（v2.0.6+）、
`extra_agent_middlewares` 工厂新增第四个 `workspace` 参数（旧三参数继续兼容，v2.0.6+）、
`cur_iter` 轮次计数修复（一轮仅在全部 tool call 拿到结果后才计数，新增 `AgentState.get_unfinished_tool_calls`，v2.0.6+）、
BackendBase 抽象对外导出 `DirEntry`（v2.0.6+）、
Anthropic 新增 `thinking_mode` / `thinking_display` / `reasoning_effort` 推理控制参数（v2.0.6+）、
Channel IM 频道（接入钉钉 / 飞书 / Discord，ChannelBase 适配器 + ChannelGateway 入站路由 +
ChannelLifecycleDispatcher 出站转发 + 确定性派生会话 + 交互卡片权限审批，v2.0.6+；钉钉适配器
与 `enable_channel_worker` 专用连接 worker 为 v2.0.7+）、
Skill 按 agent 隔离（`skills/.seed` 模板 + 每 agent 一个分区，惰性装备、原地可编辑、删除 agent 时
`purge_agent` 清理，`list/add/remove_skill` 带 `agent_id`，v2.0.6+）、
终端控制台 console（`launch_console` 交互式终端对话：自动渲染流式回复、处理工具调用 y/n 确认、Ctrl+C 中断当前 reply；
`ConsoleRenderer` 被动事件渲染器，可嵌入自定义循环消费 `reply_stream`；试运行 / 调试首选入口，无 session 与持久化；
v2.0.7.post1+ `agent` 参数接受 `Agent | PipelineProtocol`）、
Pipeline 流水线（`GoalPipeline` 执行者-校验者目标达成循环：executor 产出执行报告 → verifier 结构化验收
pass/fail/impossible → fail 带反馈重试至 `max_iters`；支持 HITL 暂停恢复，对外暴露与 Agent 相同的事件流，v2.0.7.post1+）、
达到 `max_iters` 后强制一次无工具的最终文本总结再结束（总结消息 `finished_reason=EXCEED_MAX_ITERS`，v2.0.7.post1+）、
on_reply 中间件可吞掉 `ReplyEndEvent` 续跑回复循环（v2.0.6+，最终 `Msg` 仅在事件逃出中间件链后产生；
`ExceedMaxItersEvent` 同步 deprecated，改查 `ReplyEndEvent.finished_reason`）、
MCP 有状态客户端支持 close 后重连（v2.0.6+）、
Omni 模型音频流、可配置 ID 工厂（set_id_factory）。

**安装**：`pip install agentscope`（Python >= 3.11）

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
| 管道 | MsgHub/SequentialPipeline/FanoutPipeline | `pipeline` 模块全新设计：`GoalPipeline` 执行者-校验者循环（v2.0.7.post1+），其余编排直接用 asyncio |
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
| Embedding / TTS 多模态模型（含 GeminiTTSModel、CosyVoice、OpenAITTSModel） | [references/models.md](references/models.md) |
| 创建消息和内容块 | [references/messages.md](references/messages.md) |
| 注册工具 (ToolBase，含 Windows PowerShell) | [references/tools.md](references/tools.md) |
| 集成 MCP | [references/tools.md](references/tools.md) |
| 管理状态 (AgentState) | [references/state.md](references/state.md) |
| Agent 配置和事件（含结构化输出 / 运行时状态注入 / 错误上报） | [references/agent-events.md](references/agent-events.md) |
| Pipeline 流水线（GoalPipeline 执行者-校验者循环） | [references/pipeline.md](references/pipeline.md) |
| 终端控制台（launch_console 交互调试 / ConsoleRenderer 事件渲染） | [references/agent-events.md](references/agent-events.md) |
| 权限和工具组（含 on_check_permission hook） | [references/permissions.md](references/permissions.md) |
| RAG 知识库（KnowledgeBase / ElasticsearchStore / RAGMiddleware） | [references/rag.md](references/rag.md) |
| 中间件（含 TTS / Tracing / Mem0 / ReMe / AgenticMemory / on_check_permission）和工作区 | [references/middleware-workspace.md](references/middleware-workspace.md) |
| 服务化、RAG 服务层与子智能体模板 (SubAgentTemplate) | [references/middleware-workspace.md](references/middleware-workspace.md) |
| Workspace 全部后端（含 AppleContainer / Bubblewrap）+ 跨用户资源共享 | [references/middleware-workspace.md](references/middleware-workspace.md) |
| Hub 注册中心（GitHubMCPHub / ClawSkillHub，浏览-安装-拉入 MCP/Skill） | [references/middleware-workspace.md](references/middleware-workspace.md) |
| Channel IM 频道（钉钉 / 飞书 / Discord，接入即时通讯平台驱动 agent 会话） | [references/middleware-workspace.md](references/middleware-workspace.md) |

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
        tool_result_limit=50000,        # 工具结果 token 上限，默认 50000
    ),
    react_config=ReActConfig(           # ReAct 循环配置
        max_iters=50,                   # 最大迭代次数（v2.0.7+ 默认 50，此前为 20）
        # v2.0.7.post1+：达到上限且尚无最终消息时，先注入提示并禁用工具强制一次
        # 最终文本总结，再以 finished_reason=EXCEED_MAX_ITERS 结束
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
await agent.compress_context(instructions=HintBlock(hint="压缩时保留关键决策依据"))
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
        case "EXCEED_MAX_ITERS": ...      # 超过最大迭代次数（v2.0.6+ deprecated，改查 REPLY_END 的 finished_reason）
        case "REQUIRE_USER_CONFIRM": ...  # 需要用户确认
        case "REQUIRE_EXTERNAL_EXECUTION": ...  # 需要外部执行
        case "REPLY_END": ...             # 结束（finished_reason 可能为 ERROR，含 error: ErrorInfo）
```

## 终端控制台 Console

`agentscope.console` 提供两个入口，方便在终端里试运行 / 调试 agent，无需自己写事件渲染逻辑：

```python
from agentscope.console import launch_console, ConsoleRenderer
```

**`launch_console`** — 一行启动交互式对话：从 stdin 读输入，自动渲染流式回复，agent 请求确认时提示 `y/n`，`Ctrl+C` 中断当前 reply（再 `Ctrl+C` 或输入 `exit` 退出）。无 session 管理、无持久化，对话仅存于 `agent.state`，随进程结束。`agent` 参数也可传 pipeline（`Agent | PipelineProtocol`，v2.0.7.post1+）：

```python
agent = Agent(name="Friday", system_prompt="...", model=model, toolkit=toolkit)
await launch_console(agent)          # 默认 verbosity="default"
# 参数：user_name（输入提示符）/ verbosity（"quiet"|"default"|"debug"）/ max_tool_result_lines（默认 20）
```

**`ConsoleRenderer`** — 被动事件渲染器，适合嵌入自己的循环（脚本、测试、自定义 UI）。把 `reply_stream` 的事件一行式输出，并通过 `last_msg` 暴露累积的回复消息：

```python
renderer = ConsoleRenderer(verbosity="default", max_tool_result_lines=20)
async for event in agent.reply_stream(user_msg):
    renderer.render(event)
print(renderer.last_msg)             # 累积出的 Msg
```

`verbosity`：`"quiet"` 仅回复文本与错误；`"default"` 额外含思考、工具调用/结果、提示块、token 用量、HITL 通知；`"debug"` 再加生命周期等默认不可见事件。详情见
[references/agent-events.md](references/agent-events.md)。

## Pipeline 流水线（v2.0.7.post1+）

`agentscope.pipeline` 把多个 agent 按固定逻辑编排成整体，对外暴露与 `Agent` 相同的
`reply_stream` 事件流，可直接传给 `launch_console` 等接受 agent 的接口：

```python
from agentscope.pipeline import GoalPipeline

# executor 执行目标 → verifier 结构化验收（pass/fail/impossible）
# fail 带反馈重试，直到通过 / 判定不可达成 / 达到 max_iters
pipe = GoalPipeline(executor=executor, verifier=verifier, max_iters=10)

async for event in pipe.reply_stream(UserMsg("user", "跑通 parser 模块的单元测试")):
    ...   # 透传两个 agent 的全部事件（含 HITL，支持暂停恢复）

await launch_console(pipe)    # 交互式调试整个流水线
```

executor / verifier 通常共享同一个 Workspace（verifier 才能读到 executor 的产物）。详情见
[references/pipeline.md](references/pipeline.md)。

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
from agentscope.workspace import (
    LocalWorkspace, DockerWorkspace, E2BWorkspace,
    K8sWorkspace, OpenSandboxWorkspace, DaytonaWorkspace,  # 后三者 v2.0.4+
    AppleContainerWorkspace, BubblewrapWorkspace,          # 后两者 v2.0.5+
)

# 八种 workspace 都支持内置工具（Bash/Read/Write/Edit/Grep/Glob），只是执行后端不同：
# LocalWorkspace -> 本地；DockerWorkspace -> 容器；E2BWorkspace -> 云沙箱；
# K8sWorkspace -> K8s Pod + PVC 持久化（v2.0.4+）；OpenSandboxWorkspace -> OpenSandbox SDK（v2.0.4+）；
# DaytonaWorkspace -> Daytona 沙箱（v2.0.4+，支持自托管）；
# AppleContainerWorkspace -> macOS 26+ Apple Container CLI（v2.0.5+，Apple Silicon 原生沙箱）；
# BubblewrapWorkspace -> Linux bubblewrap（bwrap）无 daemon 沙箱（v2.0.5+）

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

## Agent 结构化输出（v2.0.5+）

`reply` / `reply_stream` 新增 `structured_schema` 参数，传一个 **Pydantic 模型类**，
Agent 会在 ReAct 循环中调用内置的 `GenerateStructuredOutput` 工具产出符合 schema 的结构化结果：

```python
from pydantic import BaseModel

class WeatherReport(BaseModel):
    city: str
    temperature: float

res = await agent.reply(
    UserMsg("user", "杭州天气如何？"),
    structured_schema=WeatherReport,
)
# res.structured_output == {"city": "Hangzhou", "temperature": 25.0}
```

- 不是 `Agent(...)` 构造参数，而是 `reply` / `reply_stream` 的参数。
- 流式获取最终结构化消息需 `reply_stream(..., structured_schema=..., yield_final_msg=True)`。
- `ReActConfig.structured_output_grace_iters`（默认 5）控制超过 `max_iters` 后留给结构化输出的额外迭代数。
- 详情见 [references/agent-events.md](references/agent-events.md)。

## 运行时状态注入（v2.0.5+）

Agent 默认在每轮推理前把**当前墙钟时间、任务状态、上下文压缩临近预警**作为 `HintBlock` 注入
`state.context`（不改 system prompt，避免破坏 prompt caching）。由 `InjectionConfig` 控制：

```python
from agentscope.agent import Agent, InjectionConfig

agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    injection_config=InjectionConfig(
        inject_runtime_state=True,   # 默认开启
        timezone="Asia/Shanghai",
        time_interval=0.5,           # 距上次注入超过 0.5 小时再注入时间
        context_buffer_ratio=0.2,    # 进入压缩阈值前 20% 区间时预警
    ),
)
# 关闭：InjectionConfig(inject_runtime_state=False)
```

约束：`context_buffer_ratio` 必须 < `ContextConfig.trigger_ratio`，否则构造报错。

## 回复错误上报（v2.0.5+）

reply 出错时不再静默崩溃，而是把错误分类后写进 `Msg` 与 `ReplyEndEvent`：

```python
from agentscope.types import ReplyFinishedReason, ErrorType, ErrorInfo

msg = await agent.reply(user_msg)
if msg.finished_reason == ReplyFinishedReason.ERROR:
    print(msg.error.type, msg.error.message)
    # e.g. "rate_limit", "Rate limit or quota exceeded — try again later."
```

- `ReplyFinishedReason` 新增 `ERROR`（原 `COMPLETED/INTERRUPTED/EXCEED_MAX_ITERS`）。
- `ErrorType` 按 HTTP 语义分类：`authentication(401)` / `permission(403)` / `rate_limit(429)` /
  `invalid_request(400/422)` / `upstream(5xx)` / `connection` / `internal` / `unknown`。
- `ErrorInfo.message` 是通用文案，不泄露原始异常/密钥。
- 服务化（`create_app`）的 SSE 流也会合成 `finished_reason=ERROR` 的 `ReplyEndEvent`。

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
