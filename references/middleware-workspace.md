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

# mem0 长期记忆（需 pip install mem0）
longterm = Mem0Middleware(
    user_id="alice",
    # client=mem0.AsyncMemoryClient(...),   # 方式1: 传入已构建的 mem0 client
    # chat_model=..., embedding_model=...,  # 方式2: 让中间件自动构建 OSS AsyncMemory
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
    storage=RedisStorage(),               # 存储后端（必填，关键字参数）
    message_bus=RedisMessageBus(),         # 消息总线（必填，关键字参数）
    workspace_manager=LocalWorkspaceManager(),  # 工作区管理器（必填，关键字参数）
    extra_credentials=[MyCredential],      # 自定义 Credential 类
    extra_agent_middlewares=my_middleware_factory,  # 中间件工厂
    extra_agent_tools=my_tool_factory,     # 工具工厂
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
