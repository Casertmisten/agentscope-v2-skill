# 中间件与工作区 — v2

## Middleware 中间件系统

v2 新增中间件系统，允许在不修改 Agent 源码的情况下拦截和修改行为。

### MiddlewareBase

```python
from agentscope.middleware import (
    MiddlewareBase,
    TracingMiddleware,
    TTSMiddleware,
    ReplyBudgetControlMiddleware,
    Mem0Middleware,
)
```

### 内置中间件

除自定义中间件外，框架提供以下内置中间件：

| 中间件 | 作用 |
|---|---|
| `TracingMiddleware` | 记录每步执行的 tracing 信息（调试/观测用） |
| `TTSMiddleware` | 把 reasoning 文本转语音，注入 `DATA_BLOCK_*` 事件（见下文） |
| `ReplyBudgetControlMiddleware` | 按 token 权重限制单次 reply 的消耗（达到预算时给智能体 hint） |
| `Mem0Middleware` | 基于 [mem0](https://github.com/mem0ai/mem0) 的长期记忆，跨会话记忆用户偏好 |

```python
from agentscope.middleware import (
    ReplyBudgetControlMiddleware,
    Mem0Middleware,
)

# token 预算控制
budget = ReplyBudgetControlMiddleware(
    token_budget=10000,            # 单次 reply 最大 token 消耗
    input_token_weight=1,          # 输入 token 权重
    output_token_weight=1,         # 输出 token 权重
)

# mem0 长期记忆 —— 详见下文「Mem0Middleware」小节
longterm = Mem0Middleware(
    user_id="alice",                       # 必填
    chat_model=my_chat_model,              # 方式1: 内部构建 OSS AsyncMemory
    embedding_model=my_embedding_model,
    mode="both",                           # static_control / agent_control / both
)
```

中间件支持 6 个拦截点，每个都是可选的（只需实现需要的）：

### 拦截点

| 拦截点 | 模式 | 说明 |
|---|---|---|
| `on_reply` | 洋葱模型 | 拦截整个 reply 过程 |
| `on_reasoning` | 洋葱模型 | 拦截推理/模型调用阶段 |
| `on_acting` | 洋葱模型 | 拦截单个工具执行 |
| `on_model_call` | 洋葱模型 | 拦截原始模型 API 调用 |
| `on_compress_context` | 洋葱模型 | 拦截上下文压缩 |
| `on_system_prompt` | 变压器模型 | 转换系统提示字符串 |

### 洋葱模型（Onion Pattern）

before/after 逻辑，通过 `next_handler` 调用下一个中间件或原始方法：

```python
class LoggingMiddleware(MiddlewareBase):
    async def on_reasoning(
        self,
        agent,           # Agent 实例
        input_kwargs,    # 字典，含 tool_choice
        next_handler,    # Callable → AsyncGenerator
    ) -> AsyncGenerator:
        print(f"Before reasoning for {agent.name}")
        async for event in next_handler():
            yield event
        print(f"After reasoning for {agent.name}")
```

### 变压器模型（Transformer Pattern）

顺序管道，每个中间件接收上一个的输出：

```python
class PromptMiddleware(MiddlewareBase):
    async def on_system_prompt(self, agent, current_prompt: str) -> str:
        return current_prompt + "\n\n额外指令：始终用中文回复。"
```

### 各拦截点的 input_kwargs

```python
# on_reply
input_kwargs = {"inputs": msg_or_event}

# on_reasoning
input_kwargs = {"tool_choice": tool_choice}

# on_acting
input_kwargs = {"tool_call": ToolCallBlock(...)}

# on_model_call
input_kwargs = {
    "current_model": ChatModelBase,
    "messages": list[Msg],
    "tools": list[dict],
    "tool_choice": ToolChoice,
}

# on_compress_context
input_kwargs = {"context_config": ContextConfig | None}
```

### TTSMiddleware — 语音合成（v2.0.2+）

> ⚠️ 实验性：依赖的 `tts` 模块属于 Voice Agent 路线（roadmap）的进行中方向，行为可能调整。

内置的 `TTSMiddleware` 把 reasoning 阶段产生的文本块自动转成语音，并以 `DATA_BLOCK_*` 事件注入事件流。它拦截 `on_reply`，因此可以直接用于前端音频播放：

```python
from agentscope.middleware import TTSMiddleware
from agentscope.tts import DashScopeRealtimeTTSModel

tts_model = DashScopeRealtimeTTSModel(credential=credential)

agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    middlewares=[TTSMiddleware(tts_model=tts_model)],
)
```

行为细节：
- **非实时 TTS**（`tts_model.realtime=False`）：在每个 `TextBlockEndEvent` 时把累计文本送入 `synthesize`，整段合成后输出。
- **实时 TTS**（`tts_model.realtime=True`）：每个 `TextBlockDeltaEvent` 通过 `push` 推送，增量音频立即作为 `DataBlockDeltaEvent` 输出（每个 delta 携带增量 base64 PCM，按 `block_id` 串接得到完整音频）。
- 每个 `DataBlockDeltaEvent.data` 是增量 base64 PCM 块；完整音频 = 所有同 `block_id` 的 delta 解码后拼接。

### Mem0Middleware — mem0 长期记忆（v2.0.3+）

> 需额外安装：`pip install agentscope[mem0]`（或 `pip install mem0`）。底层依赖
> [mem0](https://github.com/mem0ai/mem0)，兼容 OSS `mem0.AsyncMemory` 与托管版 `mem0.AsyncMemoryClient`。

`Mem0Middleware` 给 agent 加上**跨会话的长期记忆**。它在每次 reply 前检索相关记忆、reply 后写回新事实，
或/并暴露 `search_memory`/`add_memory` 工具让 agent 自主调用。三条拦截点联动：`on_reply`（自动检索/写回）+
`on_system_prompt`（注入工具说明）+ `list_tools`（暴露记忆工具）。

**三条记忆路径**，由 `mode` 参数控制：

| `mode` | 自动检索+写回 | 暴露记忆工具 | 适用 |
|---|---|---|---|
| `"static_control"` | ✅ 每次 reply 前检索、reply 后写回 | ❌ | agent 不感知 mem0，全自动 |
| `"agent_control"` | ❌ | ✅ `search_memory`/`add_memory` | agent 按需调用，系统提示注入说明 |
| `"both"`（默认） | ✅ | ✅ | 兼具，对齐 v1 的 `long_term_memory_mode` 默认 |

**两种构造方式**（二选一，`client` 优先）：

```python
from agentscope.middleware import Mem0Middleware

# 方式 1：传 AgentScope 模型 —— 中间件内部构建 OSS AsyncMemory，
#         mem0 的抽取 LLM 和 embedding 都走你的 AgentScope 模型
mw = Mem0Middleware(
    user_id="alice",                       # 必填
    chat_model=my_chat_model,
    embedding_model=my_embedding_model,    # dimensions 须匹配 mem0 向量库（默认 Qdrant 1536）
    # mem0_config=MemoryConfig(...),       # 可选：自定义向量库/历史库/reranker
    mode="both",
)

# 方式 2：传预构建的 mem0 client —— 适合托管版 Platform 或多 agent 共享同一 mem0
from mem0 import AsyncMemoryClient
mw = Mem0Middleware(
    user_id="alice",
    client=AsyncMemoryClient(api_key="m0-..."),
    mode="both",
)

# 挂载到 agent（agent_control/both 模式需把工具加进 toolkit）
agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    toolkit=Toolkit(tools=await mw.list_tools()),   # static_control 下返回 []
    middlewares=[mw],
)
```

**参数表**：

| 参数 | 默认 | 说明 |
|---|---|---|
| `user_id` | （必填） | mem0 用户隔离维度 |
| `client` | `None` | 预构建的 `AsyncMemory`/`AsyncMemoryClient`，给定则忽略模型参数 |
| `chat_model` / `embedding_model` | `None` | 内部构建 OSS mem0 用的 AgentScope 模型（embedding 的 `dimensions` 须匹配向量库） |
| `mem0_config` | `None` | `MemoryConfig`，自定义向量库/历史库；与 `client` 互斥 |
| `mode` | `"both"` | 见上表 |
| `agent_id` | `None` | 更细的 mem0 命名空间；`None` 则只按 `user_id` 隔离 |
| `top_k` | `5` | 每次静态检索的记忆数；也是 `search_memory` 工具的默认值 |
| `threshold` | `None` | 最低相似度，`None` 让 mem0 决定 |
| `scope_search_by_agent` | `True` | `True`→检索带 `agent_id` 过滤（记忆按 agent 隔离）；`False`→同用户跨 agent 共享 |
| `await_write` | `True` | `True`→reply 后同步等写回；`False`→fire-and-forget（更快但异常只进日志） |
| `memory_section_header/intro` | 内置文案 | 注入记忆块时用的标题/引导语（`static_control`/`both`） |
| `tool_instructions` | 内置文案 | `agent_control`/`both` 下追加到系统提示的工具说明 |

`static_control`/`both` 模式下，检索到的记忆会以 `AssistantMsg(name="memory")` 形式拼进
`agent.state.context`；新对话在 reply 后写回 mem0。

### 中间件提供工具

```python
class MyMiddleware(MiddlewareBase):
    async def list_tools(self) -> list[ToolBase]:
        return [MyCustomTool()]
```

### 使用方式

```python
agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    middlewares=[
        LoggingMiddleware(),
        PromptMiddleware(),
        TracingMiddleware(),
        TTSMiddleware(tts_model=tts_model),     # v2.0.2+
        ReplyBudgetControlMiddleware(token_budget=10000),
    ],
)
```

## Workspace 工作区系统

v2 新增工作区系统，为智能体提供隔离的执行环境。

### WorkspaceBase（抽象基类）

```python
from agentscope.workspace import WorkspaceBase, Offloader
```

工作区提供：
- **工具** — 内置工具（Bash/Read/Write/Edit/Glob/Grep）
- **MCP** — 动态管理 MCP 服务器
- **Skill** — 动态加载/卸载技能
- **Offload** — 持久化压缩上下文和工具结果

### LocalWorkspace — 本地工作区

```python
from agentscope.workspace import LocalWorkspace

workspace = LocalWorkspace(
    workdir="/path/to/workspace",
    workspace_id=None,                 # 可选，默认自动生成
    default_mcps=[mcp_client],         # 初始 MCP（仅首次生效）
    skill_paths=["/path/to/skills"],   # 初始技能目录
    instructions="自定义指令模板",       # 支持 {workdir} 占位符
)
```

目录结构：

```
{workdir}/
├── .mcp          # MCP 配置（JSON 数组）
├── data/         # 卸载的多模态文件
├── skills/       # 技能子目录
└── sessions/     # 会话上下文和工具结果
```

### DockerWorkspace — Docker 工作区

```python
from agentscope.workspace import DockerWorkspace

workspace = DockerWorkspace(...)
```

### E2BWorkspace — E2B 云沙箱

```python
from agentscope.workspace import E2BWorkspace

workspace = E2BWorkspace(...)
```

### Backend 抽象（v2.0.3+）

builtin 工具（Bash/Read/Write 等）的执行被抽象到 `BackendBase` 体系，docker / e2b / local 三种后端各自实现 `exec_shell` / `read_file` / `write_file` 原语。`DockerBackend` / `E2BBackend` 已对外导出，供自定义工具/工作区的高级开发者直接构造：

```python
from agentscope.workspace import DockerBackend, E2BBackend
from agentscope.tool import BackendBase, LocalBackend, ExecResult
```

> ℹ️ 普通用 `LocalWorkspace` / `DockerWorkspace` / `E2BWorkspace` 的开发者无需手动构造 Backend——workspace 内部会自行创建。

### 三种 Workspace 都支持内置工具（v2.0.3+）

`LocalWorkspace` / `DockerWorkspace` / `E2BWorkspace` 均实现了 `list_tools()`，返回六个内置工具
（Bash/Read/Write/Edit/Grep/Glob）。区别仅在于**执行后端**：

| Workspace | 内置工具执行位置 |
|---|---|
| `LocalWorkspace` | 本地文件系统（`LocalBackend`） |
| `DockerWorkspace` | 容器内（`DockerBackend`） |
| `E2BWorkspace` | E2B 云沙箱内（`E2BBackend`，路径 `/home/user/workspace`） |

```python
# E2B/Docker workspace 必须先 initialize（或用 async with），才能取内置工具
async with E2BWorkspace(...) as ws:
    tools = await ws.list_tools()   # 6 个工具，全部跑在沙箱内
    # 此时 workspace 内部已创建 E2BBackend
```

> ⚠️ `E2BWorkspace.list_tools()` 在未 initialize 时会抛 `RuntimeError`——这是有意为之，避免
> 内置工具静默回退到 `LocalBackend` 跑在宿主机上。`DockerWorkspace` 同理。

### 生命周期管理

```python
# 方式 1：上下文管理器（推荐）
async with workspace:
    agent = Agent(
        name="Assistant",
        system_prompt="...",
        model=model,
        offloader=workspace,
    )
    # workspace 自动 initialize 和 close

# 方式 2：手动管理
await workspace.initialize()
# ... 使用 workspace ...
await workspace.close()

# 重置为空状态
await workspace.reset()
```

### 动态管理

```python
# 添加/移除 MCP
await workspace.add_mcp(new_mcp_client)
await workspace.remove_mcp("weather")

# 添加/移除 Skill
await workspace.add_skill("/path/to/skill/dir")
await workspace.remove_skill("skill-name")

# 查询
tools = await workspace.list_tools()     # 内置工具列表
mcps = await workspace.list_mcps()       # MCP 客户端列表
skills = await workspace.list_skills()   # Skill 列表
instructions = await workspace.get_instructions()  # 系统提示片段
```

### 与 Agent 集成

```python
# 作为 offloader 使用（启用上下文卸载）
agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    toolkit=Toolkit(),
    offloader=workspace,
)
```

当配置了 offloader 时：
- 压缩的上下文会持久化到 `sessions/{session_id}/context.jsonl`
- 超长工具结果会持久化到 `sessions/{session_id}/tool_result-{id}.txt`
- Agent 的 system_prompt 会自动附加 workspace 指令

## App 服务化

v2 提供 FastAPI 工厂函数，用于将 Agent 托管为 REST + SSE 服务。

### create_app

```python
from agentscope.app import create_app, SubAgentTemplate

app = create_app(
    storage=RedisStorage(),               # 存储后端（必填）
    message_bus=RedisMessageBus(),         # 消息总线（必填）
    workspace_manager=LocalWorkspaceManager(),  # 工作区管理器（必填）
    extra_credentials=[MyCredential],      # 自定义 Credential 类
    extra_middlewares=[MyFastAPIMiddleware()],   # FastAPI 中间件（如 CORS、限流）
    extra_agent_middlewares=my_middleware_factory,  # agent 中间件工厂
    extra_agent_tools=my_tool_factory,     # agent 工具工厂
    custom_subagent_templates=[...],       # 子智能体模板（v2.0.2+）
    custom_agent_cls=MyAgent,              # 自定义 Agent 子类（v2.0.2+）
)

# 独立运行
import uvicorn
uvicorn.run(app, host="0.0.0.0", port=8000)

# 挂载到已有 FastAPI
root.mount("/agentscope", app)
```

> ⚠️ v2.0.2 起 `create_app` 在第三个必填参数后加了 `*`，即 `extra_*` / `custom_*` / `title` / `version`
> 均为**仅关键字参数**；`storage` / `message_bus` / `workspace_manager` 仍可位置或关键字传递（推荐关键字）。
>
> 注意区分两类 middleware 参数：
> - `extra_middlewares` —— 加到 **FastAPI 应用**层（ASGI 中间件，处理 CORS/鉴权/限流等 HTTP 层逻辑）。
> - `extra_agent_middlewares` —— 加到**每个 Agent 实例**的 `MiddlewareBase` 中间件（拦截 reply/reasoning 等）。
>
> 另有 AG-UI 协议适配层：服务会把内部 `AgentEvent` 转换成 [AG-UI](https://docs.google.com/document/d/1F8gZV5mcrzBB_Lu6p1Tq7v5K0eJ7r5K2/) 兼容的 SSE 事件流，方便接入标准 AG-UI 客户端。

### 消息总线（MessageBus）

`create_app` 的 `message_bus` 参数决定跨 session/进程的消息传输后端（事件扇出、唤醒空闲消费者、分布式锁等）。提供两种实现：

```python
from agentscope.app.message_bus import MessageBus, RedisMessageBus, InMemoryMessageBus

# 生产 / 多进程部署：Redis
bus = RedisMessageBus(...)

# 单节点 / 本地开发：纯内存（无 Redis 依赖，仅单进程内有效）
bus = InMemoryMessageBus()
```

- `RedisMessageBus` —— 生产环境、多 worker 部署必选。
- `InMemoryMessageBus`（v2.0.3+）—— 单进程部署的轻量选项，无需 Redis，适合本地开发与测试。**不支持过期/多进程**。
- `MessageBus` —— 上述两者的抽象基类，可自行子类化实现自定义后端。

> ℹ️ MessageBus 属于服务化层基础设施，普通 agent 开发者无需直接使用——由 `create_app` 启动的 FastAPI 服务在内部持有并调用。

### 内置路由

- `agent_router` — 智能体管理
- `chat_router` — 对话（REST + SSE）
- `credential_router` — 凭据管理
- `model_router` — 模型配置
- `tts_model_router` — TTS 模型配置（v2.0.2+）
- `schedule_router` — 定时任务
- `session_router` — 会话管理
- `workspace_router` — 工作区操作

## SubAgentTemplate — 子智能体模板（v2.0.2+）

`SubAgentTemplate` 是创建子智能体（团队 worker）的可复用蓝图。在 `create_app` 注册后，内置的 `AgentCreate` 工具会暴露 `subagent_type` 参数，leader 智能体据此路由到对应模板：

```python
from agentscope.app import SubAgentTemplate
from agentscope.permission import PermissionContext, PermissionMode

app = create_app(
    # ...
    custom_subagent_templates=[
        SubAgentTemplate(
            type="explorer",
            description="只读探索智能体，用于调研代码库、收集信息，不做修改。",
            system_prompt_template=(
                "你是团队 {team_name} 的成员 {member_name}。{team_description}"
            ),  # 占位符: {team_name}/{team_description}/{member_name}/
               #          {member_description}/{leader_name}
            permission_context=PermissionContext(mode=PermissionMode.EXPLORE),
            override_leader_mode=True,   # True: worker 用模板的 mode；False: 继承 leader
        ),
    ],
)
```

字段说明：

| 字段 | 说明 |
|---|---|
| `type` | 模板类型标识（如 `"explorer"`/`"coder"`），作为 `AgentCreate` 的 `subagent_type` 枚举值 |
| `description` | 给 LLM 看的描述，出现在 `AgentCreate` 工具 schema 中 |
| `system_prompt_template` | 系统提示格式串，支持占位符 `{team_name}` `{team_description}` `{member_name}` `{member_description}` `{leader_name}` |
| `context_config` | 子智能体的上下文压缩配置（默认 `ContextConfig()`） |
| `react_config` | 子智能体的 ReAct 循环配置 |
| `permission_context` | 创建时的权限上下文（如只读/完全访问） |
| `override_leader_mode` | `True`→worker 使用模板 mode；`False`（默认）→继承 leader 当前 mode |
| `extend_leader_permission_rules` | `True`（默认）→leader 的 allow/deny/ask 规则叠加在模板规则之上（模板优先匹配） |
| `extend_leader_working_directories` | `True`（默认）→leader 工作目录合并进模板目录（模板在键冲突时优先） |
| `tasks_context` | 预定义任务上下文，用于初始化子智能体的工作流 |

- 不注册任何模板时，使用内置的 `DEFAULT_SUB_AGENT_TEMPLATE`。
- `type` 不允许重复，否则 `create_app` 抛 `ValueError`。
- 另有 `custom_agent_cls` 参数可整体替换 `Agent` 类（用于团队编排等定制场景）。

### 团队 Leader 的 HITL 事件暴露（v2.0.3+）

在多智能体团队中，worker 子智能体若需要人工确认（`RequireUserConfirmEvent`）或外部执行
（`RequireExternalExecutionEvent`），事件会被**投影到 team leader 的会话事件流**，leader 的前端无需
单独订阅每个 worker。这是服务层行为，通过内部 `EventProjector`（`app/_service/_projectors/`）实现，
普通开发者**无需新增 `create_app` 参数**——只要 worker 通过 `SubAgentTemplate` 创建并归属到同一个 team，
HITL 事件就会自动冒泡到 leader session。
