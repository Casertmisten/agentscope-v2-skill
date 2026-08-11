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
    ReMeMiddleware,
    AgenticMemoryMiddleware,
    RAGMiddleware,
)
```

### 内置中间件

除自定义中间件外，框架提供以下内置中间件：

| 中间件 | 作用 |
|---|---|
| `TracingMiddleware` | OpenTelemetry 追踪，在 reply/model/tool 三层创建 span（见下文） |
| `TTSMiddleware` | 把 reasoning 文本转语音，注入 `DATA_BLOCK_*` 事件（见下文） |
| `ReplyBudgetControlMiddleware` | 按 token 权重限制单次 reply 的消耗（达到预算时给智能体 hint） |
| `Mem0Middleware` | 基于 [mem0](https://github.com/mem0ai/mem0) 的长期记忆，跨会话记忆用户偏好 |
| `ReMeMiddleware` | 内嵌 [ReMe](https://github.com/agentscope-ai/ReMe) 应用的长期记忆，自动写回并可工具检索 |
| `AgenticMemoryMiddleware`（v2.0.4+） | 基于文件系统（Markdown）的长期记忆，由 Agent 自主读写记忆文件 |
| `RAGMiddleware` | 检索增强生成中间件，详见 [rag.md](rag.md) |

```python
from agentscope.middleware import (
    ReplyBudgetControlMiddleware,
    TracingMiddleware,
    Mem0Middleware,
    ReMeMiddleware,
    AgenticMemoryMiddleware,
)

# token 预算控制
budget = ReplyBudgetControlMiddleware(
    token_budget=10000,            # 单次 reply 最大 token 消耗
    input_token_weight=1,          # 输入 token 权重
    output_token_weight=1,         # 输出 token 权重
)

# OpenTelemetry tracing
tracing = TracingMiddleware()

# mem0 长期记忆 —— 详见下文「Mem0Middleware」小节
longterm = Mem0Middleware(
    user_id="alice",                       # 必填
    chat_model=my_chat_model,              # 方式1: 内部构建 OSS AsyncMemory
    embedding_model=my_embedding_model,
    mode="both",                           # static_control / agent_control / both
)

# ReMe 长期记忆 —— 详见下文「ReMeMiddleware」小节
reme_memory = ReMeMiddleware(
    workspace_dir=".reme",
    parameters=ReMeMiddleware.Parameters(
        chat_model=my_chat_model,
        embedding_model=my_embedding_model,
        mode="both",
    ),
)

# 文件系统长期记忆 —— 详见下文「AgenticMemoryMiddleware」小节
agentic = AgenticMemoryMiddleware(workdir="./workspace")
```

中间件支持 7 个拦截点，每个都是可选的（只需实现需要的）：

### 拦截点

| 拦截点 | 模式 | 说明 |
|---|---|---|
| `on_reply` | 洋葱模型 | 拦截整个 reply 过程 |
| `on_reasoning` | 洋葱模型 | 拦截推理/模型调用阶段 |
| `on_check_permission`（v2.0.5+） | 洋葱模型 | 拦截单次工具调用的权限检查 |
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

# on_check_permission（v2.0.5+）
input_kwargs = {"tool_call": ToolCallBlock, "tool": ToolBase, "tool_input": dict}

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
input_kwargs = {"context_config": ContextConfig | None, "instructions": HintBlock | None}
```

### on_check_permission — 权限检查洋葱 hook（v2.0.5+）

介于 `on_reasoning` 与 `on_acting` 之间的切入点，包裹**单次工具调用的权限检查**。
`input_kwargs` 含已校验的 `tool_call`（`ToolCallBlock`）、解析后的 `tool`（`ToolBase`）、
schema 校验后的 `tool_input`（dict）。Agent 在调用链前对它们做了 `deepcopy`，中间件改副本
不会污染后续真实调用。返回 `PermissionDecision`（ALLOW/DENY/ASK/PASSTHROUGH）。

```python
from agentscope.middleware import MiddlewareBase
from agentscope.permission import PermissionDecision

class AuditPermissionMiddleware(MiddlewareBase):
    async def on_check_permission(self, agent, input_kwargs, next_handler):
        tool = input_kwargs["tool"]
        # 委派给内置权限引擎拿到默认决策
        decision = await next_handler(**input_kwargs)
        agent.logger.info(f"{tool.name} -> {decision.behavior.value}")
        return decision   # 可在此替换/拦截决策
```

> 若不调 `next_handler`，则跳过该调用的内置权限解析——可用于自定义完全独立的权限策略。
> 已被用户确认放行的调用（`tool_call.state == ALLOWED`）仍走完整中间件链，但内置引擎会短路为 ALLOW。

### TracingMiddleware — OpenTelemetry 追踪

`TracingMiddleware` 在 `on_reply` / `on_model_call` / `on_acting` 三层创建 OpenTelemetry span：
reply span 记录 agent/session/reply、HITL/外部执行等待状态和最终回复；model span 记录模型调用请求与响应；
tool span 记录工具调用与结果。未配置 OpenTelemetry SDK 时会快速透传，开销很低。

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry import trace
from agentscope.middleware import TracingMiddleware

trace.set_tracer_provider(TracerProvider())

agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    middlewares=[TracingMiddleware()],
)
```

需要导出到 OTLP/Jaeger/控制台时，按标准 OpenTelemetry Python SDK 给 `TracerProvider` 添加 exporter/processor。

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

### ReMeMiddleware — ReMe 长期记忆（v2.0.4+）

> 需额外安装：`pip install agentscope[reme]`（或 `pip install reme-ai`）。底层依赖
> [ReMe](https://github.com/agentscope-ai/ReMe)，但中间件会在进程内嵌入 ReMe 应用，**不需要单独启动服务**。

`ReMeMiddleware` 通过 `on_reply` 监听完整对话增量，并在每轮 reply 结束后调用 ReMe 的 `auto_memory`
自动写回；agent 不会手动写记忆，也没有 `add_memory` 工具。`mode` 只控制**检索**方式：

| `mode` | 自动检索 | 暴露工具 | 写回 |
|---|---|---|---|
| `"static_control"` | ✅ reply 开始时后台搜索，推理阶段尽力注入 | ❌ | ✅ 每轮自动 |
| `"agent_control"` | ❌ | ✅ `memory_search` | ✅ 每轮自动 |
| `"both"`（默认） | ✅ | ✅ `memory_search` | ✅ 每轮自动 |

```python
from agentscope.middleware import ReMeMiddleware
from agentscope.tool import Toolkit

reme_mw = ReMeMiddleware(
    workspace_dir=".reme",          # ReMe vault / workspace
    config="default",               # ReMe 配置名或路径
    parameters=ReMeMiddleware.Parameters(
        chat_model=my_chat_model,       # 可选：注入给 ReMe 的 LLM 组件
        embedding_model=my_embedding_model,  # 可选：启用/注入向量检索
        mode="both",
        top_k=5,
    ),
)

agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    toolkit=Toolkit(tools=await reme_mw.list_tools()),
    middlewares=[reme_mw],
)

# AgentScope 不托管 middleware 生命周期；应用退出时可显式关闭
await reme_mw.close()
```

关键点：
- ReMe 的 `session_id` 从 `agent.state.session_id` 读取，不在 middleware 构造时传入；恢复会话时应恢复
  `AgentState(session_id=...)`。
- `chat_model` / `embedding_model` 固定在 middleware 构造参数中，适合一个 ReMe app 共享给多个 agent。
- 静态检索是后台任务，单次非常短的 reply 可能先结束；这种情况下本轮不注入，写回仍会执行。
- `embedding_model` 为 `None` 时，ReMe 使用自身配置；提供 embedding 时会启用 ReMe 的向量存储接线。

### AgenticMemoryMiddleware — 文件系统长期记忆（v2.0.4+）

与 `Mem0Middleware`（依赖外部 mem0）不同，`AgenticMemoryMiddleware` 把长期记忆存为工作目录下的
**Markdown 文件**（含 `MEMORY.md` 索引），由 Agent 自主读写，**零外部依赖**。它给系统提示注入一段
记忆管理指令（指导 Agent 何时、如何保存 user/feedback/project/reference 四类记忆），并在每次 reply
异步检索相关主题文件作为 `HintBlock` 注入。

> 对标 Claude Code 的 Auto Memory 思路：LLM 自行决定何时用 `Write` 工具落盘记忆文件，
> `Read` 读取已有记忆；中间件负责把 `MEMORY.md` 索引和检索结果送入上下文。

```python
from agentscope.middleware import AgenticMemoryMiddleware

mem_mw = AgenticMemoryMiddleware(
    workdir="./demo_workspace",   # 必填：工作目录，记忆文件存在其下的 memory_dir
    memory_dir="Memory",          # 记忆子目录，默认 "Memory"（含 MEMORY.md 索引）
    parameters=None,              # AgenticMemoryMiddleware.Parameters，None 用默认
    backend=None,                 # BackendBase；None 用本地文件系统（LocalBackend）
)

# 必须配合 Read/Write 工具，Agent 才能读写记忆文件
agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    toolkit=Toolkit(tools=[Read(), Write()]),
    middlewares=[mem_mw],
)
```

**记忆类型**（保存在 Markdown frontmatter 的 `type` 字段）：`user`（用户画像）/ `feedback`
（用户反馈与偏好）/ `project`（项目动态）/ `reference`（外部系统指引）。中间件指令会明确告诉
Agent **不该存什么**（代码结构、git 历史、临时任务状态等——这些可从项目本身派生）。

| 参数 | 默认 | 说明 |
|---|---|---|
| `workdir` | （必填） | 工作目录根路径 |
| `memory_dir` | `"Memory"` | 记忆文件子目录（`MEMORY.md` 索引也在其中） |
| `parameters` | `None` | `Parameters` 配置（检索 top_k、注入上限等），`None` 用默认 |
| `backend` | `None` | `BackendBase`；`None` 用 `LocalBackend`，也可传 docker/e2b 后端实现远程记忆存储 |

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
        ReMeMiddleware(workspace_dir=".reme"),
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
- **MCP** — 动态管理 MCP 服务器（v2.0.6+ 起按 agent+session 隔离，见下文「MCP 隔离与生命周期」）
- **Skill** — 动态加载/卸载技能
- **Offload** — 持久化压缩上下文和工具结果

### LocalWorkspace — 本地工作区

```python
from agentscope.workspace import LocalWorkspace

workspace = LocalWorkspace(
    workdir="/path/to/workspace",
    workspace_id=None,                 # 可选，默认自动生成
    default_mcps=[mcp_client],         # 初始 MCP（仅首次生效）
    skill_paths=["/path/to/skills"],   # 初始技能目录（v2.0.5+ 支持 ~/path 展开）
    max_live_stateful_mcps=None,       # 可选，并发 live stateful MCP 上限（v2.0.6+）
    instructions="自定义指令模板",       # 支持 {workdir} 占位符
)
```

> ℹ️ v2.0.5+ 起，`skill_paths`、`LocalSkillLoader(directory=...)`、`workspace.add_skill(path)`
> 中的路径都会先 `expanduser` 再 `abspath`，因此可直接传 `~/my-skills/foo`，
> 不再被误解析为相对当前工作目录的字面量。

目录结构：

```
{workdir}/
├── .mcp          # MCP 配置（v2.0.6+ 起 per agent/session，schema 见下）
├── data/         # 卸载的多模态文件
├── skills/       # 技能子目录
└── sessions/     # 会话上下文和工具结果（按 session_id 分目录）
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

builtin 工具（Bash/Read/Write 等）的执行被抽象到 `BackendBase` 体系，local / docker / e2b / k8s /
opensandbox / daytona / applecontainer / bubblewrap 八种后端各自实现 `exec_shell` / `read_file` /
`write_file` 原语。各 `*Backend` 均已对外导出，供自定义工具/工作区的高级开发者直接构造：

```python
from agentscope.workspace import (
    DockerBackend, E2BBackend, K8sBackend, OpenSandboxBackend, DaytonaBackend,
    AppleContainerBackend, BubblewrapBackend,  # 后两者 v2.0.5+
)
from agentscope.tool import BackendBase, LocalBackend, ExecResult, DirEntry  # DirEntry v2.0.6+
```

> ℹ️ 普通用 `LocalWorkspace` / `DockerWorkspace` / `E2BWorkspace` 的开发者无需手动构造 Backend——workspace 内部会自行创建。
> `K8sBackend` / `OpenSandboxBackend` / `DaytonaBackend` / `AppleContainerBackend` / `BubblewrapBackend`
> 也已对外导出，对应的沙箱 workspace 同理。

### K8sWorkspace — K8s Pod 工作区（v2.0.4+）

基于 `kubernetes_asyncio` 管理 Pod + PVC 生命周期，适合在自有 K8s 集群中为每个智能体分配
带持久化卷的隔离执行环境。文件 I/O 走 **tar-over-exec**（因 `kubernetes_asyncio` 的 WebSocket
对原始二进制流不可靠）:

```python
from agentscope.workspace import K8sWorkspace

workspace = K8sWorkspace(
    workspace_id=None,                # None 时自动生成;给定则按 label 复用已有 Pod
    kubeconfig=None,                  # None 用集群内 ServiceAccount;本地开发传 ~/.kube/config 路径
    namespace="agentscope",           # 目标命名空间
    image="python:3.11-slim",         # Pod 镜像
    image_pull_policy="IfNotPresent",
    image_pull_secrets=None,          # 私有仓库的 Secret 名列表
    resources=None,                   # 资源请求/限制,透传给 Pod spec
    node_selector=None, tolerations=None, service_account=None,
    gateway_port=DEFAULT_GATEWAY_PORT,
    extra_pip=None,                   # bootstrap 时额外 pip install 的包
    # PVC 持久化(Pod 重启后 /workspace 仍在)
    storage_class=None,               # None 用集群默认 StorageClass
    storage_size="1Gi",
    delete_pvc_on_close=False,        # True: close 时连 PVC 一起删;False: 只删 Pod,数据保留
    env=None,                         # Pod 环境变量
    instructions=_DEFAULT_INSTRUCTIONS,
    default_mcps=None, skill_paths=None,
)
# 必须先 initialize(或用 async with)
```

关键点:
- PVC 名固定为 `as-ws-{workspace_id}`,挂载到 `/workspace`,Pod 被删/重建后数据仍在。
- `initialize()` 按 label `agentscope.workspace.id` 查找 Pod:Running 则复用,Failed/Unknown 则重建。
- `close()` 删 Pod;`delete_pvc_on_close=True` 才会连 PVC 一起删。
- bootstrap 阶段会在 Pod 内安装 `ripgrep`/`uv`/`curl` 并建 venv,首次启动较慢。

### OpenSandboxWorkspace — OpenSandbox 工作区（v2.0.4+）

基于 [OpenSandbox](https://github.com/herenchn/open-sandbox) SDK,适合用托管沙箱服务
替代自建 Docker/K8s。文件 I/O 走 SDK 原生 `files.read_bytes` / `files.write_files`,
exec 走 `commands.run`:

```python
from agentscope.workspace import OpenSandboxWorkspace

workspace = OpenSandboxWorkspace(
    workspace_id=None,                # None 自动生成;给定则按 sandbox metadata 复用
    image=DEFAULT_IMAGE,
    api_key="",                       # OpenSandbox 服务 API key
    domain="", protocol="http",
    request_timeout_seconds=DEFAULT_REQUEST_TIMEOUT,
    timeout_seconds=DEFAULT_TIMEOUT,
    gateway_port=DEFAULT_GATEWAY_PORT,
    env=None, sandbox_metadata=None, resource=None,
    entrypoint=None, network_policy=None, extra_pip=None,
    instructions=_DEFAULT_INSTRUCTIONS,
    default_mcps=None, skill_paths=None,
)
```

关键点:
- 通过 `Sandbox.create/connect/resume` 管理 sandbox;AgentScope 把 workspace_id 存进
  sandbox 的 `agentscope.workspace.id` metadata 键,用于后续查找并重新挂载。
- `close()` 调用 `sandbox.pause()`——**只暂停不销毁**,文件系统保留,下次可 resume,适合
  频繁启停的场景。
- bootstrap 阶段在 sandbox 内安装 `ripgrep`/`uv`/`curl` 并建 venv。

### DaytonaWorkspace — Daytona 沙箱工作区（v2.0.4+）

基于 [Daytona](https://www.daytona.io/) SDK,适合用托管或自托管的 Daytona 沙箱服务。与其他
沙箱 workspace 不同,Daytona 的 `workdir` / `_gateway_home` **不在构造时写死**,而是在
`initialize()` 阶段从 SDK 的 `get_work_dir()` / `get_user_home_dir()` 派生(因为 snapshot 可能
以非 root 用户运行)。文件 I/O 走 `sandbox.fs.upload_file` / `download_file`,exec 走
`sandbox.process.exec`:

```python
from agentscope.workspace import DaytonaWorkspace

workspace = DaytonaWorkspace(
    workspace_id=None,                # None 自动生成;给定则按 label 复用已有沙箱
    api_key="",                       # "" 让 Daytona SDK 从环境读取凭据
    api_url="",                       # 自托管部署 URL;"" 用 SDK 默认
    target="",                        # Daytona 区域/target;"" 用 SDK 默认
    timeout_seconds=DEFAULT_TIMEOUT,  # create/start/recover/stop 超时
    gateway_port=DEFAULT_GATEWAY_PORT,
    env=None,                         # 沙箱环境变量
    sandbox_metadata=None,            # 额外 label,与 workspace-id label 合并
    extra_pip=None,                   # bootstrap 时额外 pip install 的包
    os_user=None,                     # None 让 Daytona/snapshot 决定运行用户
    instructions=_DEFAULT_INSTRUCTIONS,
    default_mcps=None, skill_paths=None,
)
# 必须先 initialize(或用 async with)
```

关键点:
- **唯一运行时派生路径的 workspace**:`workdir`、`_gateway_home`(= `{user_home}/.agentscope`)、
  uv 二进制(`{user_home}/.local/bin/uv`)都在 `_provision_backend` 时从 SDK 读取,而非类常量。
- **沙箱状态机最复杂**:`_ensure_existing_sandbox_ready()` 处理 `started/stopped/paused/
  pausing/stopping/starting/resuming/error(可恢复)` 共 8 种状态,按 label
  `agentscope.workspace.id` 查找并重连。
- `close()` 调用 `sandbox.stop(force=False)`——**非 pause(E2B/OpenSandbox 用 pause)、非 delete
  (Docker/K8s 删容器/Pod)**,文件系统保留以便下次 label 重连。
- exec 命令被 `shlex.quote` 折叠成单行并追加 `2>&1`(Daytona SDK 无独立 stderr 字段)。
- 依赖 `daytona` 包,延迟导入,未安装不影响其他 workspace。
- 每个 workspace 拥有独立的 `AsyncDaytona` 客户端。

### AppleContainerWorkspace — Apple Container 工作区（v2.0.5+）

基于 macOS 26+（Apple Silicon）的 `container` CLI（Apple Container），提供 Apple 原生的轻量
沙箱。架构上镜像 Docker/E2B，只是把 SDK 换成系统级 `container` 命令。**无 Python 包依赖**，
也**没有对应的 pip extra**——只需系统装好 `container` CLI（macOS 26+ 自带，需先 `container system start`）。

```python
from agentscope.workspace import AppleContainerWorkspace

workspace = AppleContainerWorkspace(
    workspace_id=None,                 # None 自动生成；给定则按容器名 as_ws_<id> 复用
    base_image="python:3.11-slim",     # 默认 python:3.11-slim
    gateway_port=5600,                 # MCP gateway 端口
    cpus=2,                            # CPU 核数
    memory="2G",                       # 内存上限
    env=None,                          # 容器环境变量
    extra_pip=None,                    # bootstrap 时额外 pip install 的包
    instructions=_DEFAULT_INSTRUCTIONS,
    default_mcps=None, skill_paths=None,
)
# 必须先 initialize（或用 async with）
```

关键点：
- `_provision_backend` 执行 `container run -d --name as_ws_<id> --cpus .. --memory .. <image> sleep infinity`，
  并绑定 `AppleContainerBackend(container_id=..., workdir="/workspace")`。
- 文件 I/O 走 `container exec` / `container cp`（临时文件进容器）；MCP gateway 机制与 Docker/E2B 一致。
- `close()` 执行 `container stop` 后 `container rm -f`——**容器文件系统即持久层，停止即丢失**
  （无 host workdir 概念，与 Docker volume / K8s PVC 的持久化语义不同）。

### BubblewrapWorkspace — bubblewrap 沙箱工作区（v2.0.5+）

基于 Linux [bubblewrap](https://github.com/containers/bubblewrap)（`bwrap`）的无 daemon 沙箱，
适合在不依赖 Docker/容器运行时的 Linux 环境里做命名空间隔离。**无 Python 包依赖**，也没有对应
pip extra——只需系统装好 `bwrap` 二进制（多数 Linux 发行版原生可用）。**仅 Linux**：
`_provision_backend` 会校验 `sys.platform.startswith("linux")` 和 `shutil.which("bwrap")`。

```python
from agentscope.workspace import BubblewrapWorkspace

workspace = BubblewrapWorkspace(
    workspace_id=None,                 # None 自动生成
    host_workdir=None,                 # None 创建临时目录并在 close 时删除；给定则用该宿主目录
    host_cache_dir=None,               # None 按 blake2b(host_workdir) 派生私有缓存目录
    gateway_port=None,                 # MCP gateway 端口；None 自动选取
    share_net=True,                    # 必须为 True（MCP gateway 需跨多次 bwrap 调用经 loopback 互通）
    env=None,                          # 沙箱环境变量
    extra_pip=None,                    # bootstrap 时额外 pip install 的包
    instructions=_DEFAULT_INSTRUCTIONS,
    default_mcps=None, skill_paths=None,
)
# 必须先 initialize（或用 async with）
```

关键点：
- `BubblewrapBackend.exec_shell` **每次调用都新起一个 `bwrap` 进程**：`bwrap --die-with-parent
  --new-session --unshare-all --bind <host_workdir> /workspace --bind <tmp> /tmp --share-net ...`，
  只读挂载 `/usr`/`/bin`/`/lib` 等及网络/证书配置文件，`--clearenv` 后注入受控环境。
- `read_file`/`write_file` 用沙箱内 `sh -c` + `realpath` 限制只允许写 `/workspace` 和 `/tmp`，
  并拒绝符号链接逃逸；`_validate_mount_sources` 每次执行前用 `(st_dev, st_ino)` 校验挂载源未被替换。
- bootstrap 阶段在沙箱内装 `uv`、建 venv、下载并校验 `ripgrep` 14.1.0（SHA256）。
- `share_net=False` 会直接抛 `ValueError`。

### WorkspaceManager（服务化）

服务化部署（`create_app`）通过 `WorkspaceManager` 按用户/agent/session 隔离策略派发 workspace
实例,并带 TTL 缓存 + 后台 sweeper 自动清理空闲 workspace。八种实现对应八种后端:

```python
from agentscope.app.workspace_manager import (
    LocalWorkspaceManager, DockerWorkspaceManager, E2BWorkspaceManager,
    K8sWorkspaceManager, OpenSandboxWorkspaceManager, DaytonaWorkspaceManager,  # 后三者 v2.0.4+
    AppleContainerWorkspaceManager, BubblewrapWorkspaceManager,                # 后两者 v2.0.5+
    IsolationPolicy,  # PER_AGENT / PER_USER / PER_SESSION
)

# K8s 集群部署示例
ws_manager = K8sWorkspaceManager(
    isolation=IsolationPolicy.PER_AGENT,
    namespace="agentscope",
    image="python:3.11-slim",
    storage_size="1Gi",
    ttl=3600.0,           # 空闲 workspace 存活时间(秒)
    sweep_interval=300.0, # sweeper 扫描间隔
    delete_pvc_on_close=False,
    default_mcps=None, skill_paths=None,
)
app = create_app(..., workspace_manager=ws_manager)
```

- 所有 manager 共享同一套 API:`get_workspace(user_id, agent_id, session_id, workspace_id=None)` /
  `close(workspace_id)` / `close_all()`,调用方无需关心后端类型。
- `__aenter__` 启动后台 sweeper,`__aexit__` 停止 sweeper 并 `close_all()`。
- `PER_AGENT` 下同一 agent 的多个 session 复用同一 workspace 实例，但 MCP 仍按 session 隔离
  （见「MCP 隔离与生命周期」）；session 删除时由 `SessionService` 调 `purge_session` 清理其状态，
  不影响其他 session。

Daytona 部署示例(托管或自托管,`api_url` 留空用 SDK 默认):

```python
from agentscope.app.workspace_manager import DaytonaWorkspaceManager

ws_manager = DaytonaWorkspaceManager(
    isolation=IsolationPolicy.PER_AGENT,
    api_key="",                        # "" 让 Daytona SDK 从环境读取
    api_url="",                        # 自托管 URL;"" 用 SDK 默认
    target="",
    ttl=3600.0,
    sweep_interval=300.0,
    default_mcps=None, skill_paths=None,
)
app = create_app(..., workspace_manager=ws_manager)
```

### 八种 Workspace 都支持内置工具（v2.0.3+）

`LocalWorkspace` / `DockerWorkspace` / `E2BWorkspace` / `K8sWorkspace` / `OpenSandboxWorkspace` /
`DaytonaWorkspace` / `AppleContainerWorkspace` / `BubblewrapWorkspace` 均实现了 `list_tools()`，
返回六个内置工具（Bash/Read/Write/Edit/Grep/Glob）。区别仅在于**执行后端**：

| Workspace | 内置工具执行位置 |
|---|---|
| `LocalWorkspace` | 本地文件系统（`LocalBackend`）；Windows 上 shell 工具自动用 `PowerShell`（v2.0.5+），非 Windows 用 `Bash` |
| `DockerWorkspace` | 容器内（`DockerBackend`） |
| `E2BWorkspace` | E2B 云沙箱内（`E2BBackend`，路径 `/home/user/workspace`） |
| `K8sWorkspace`（v2.0.4+） | K8s Pod 内（`K8sBackend`，tar-over-exec 文件 I/O） |
| `OpenSandboxWorkspace`（v2.0.4+） | OpenSandbox 内（`OpenSandboxBackend`，SDK 原生 files API） |
| `DaytonaWorkspace`（v2.0.4+） | Daytona 沙箱内（`DaytonaBackend`，`sandbox.fs` + `sandbox.process.exec`） |
| `AppleContainerWorkspace`（v2.0.5+） | Apple Container 内（`AppleContainerBackend`，`container exec`/`container cp`） |
| `BubblewrapWorkspace`（v2.0.5+） | bubblewrap 沙箱内（`BubblewrapBackend`，每次 `bwrap` 新起隔离进程） |

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
# 添加/移除 MCP（agent_id/session_id 为仅关键字参数，默认 "" 即旧版遗留 id）
await workspace.add_mcp(new_mcp_client, agent_id=agent_id, session_id=session_id)
await workspace.remove_mcp("weather", agent_id=agent_id, session_id=session_id)

# 添加/移除 Skill
await workspace.add_skill("/path/to/skill/dir")
await workspace.remove_skill("skill-name")

# 查询
tools = await workspace.list_tools()     # 内置工具列表
mcps = await workspace.list_mcps(        # 该 agent/session 的 MCP 客户端列表
    agent_id=agent_id, session_id=session_id,
)
skills = await workspace.list_skills()   # Skill 列表
instructions = await workspace.get_instructions()  # 系统提示片段
```

### MCP 隔离与生命周期（v2.0.6+）

同一 workspace 可被多个 agent/session 共享（尤其 `PER_AGENT` 隔离下，同一 agent 的多个 session
共用一个 workspace 实例），但 **MCP 的声明和 live 实例按 `(agent_id, session_id)` 隔离**，
互不串扰：

- **延迟实例化 + 私有实例**：MCP 只在某 session 第一次调用 `list_mcps` 时才连接，
  每个 session 拿到独立实例——有状态 MCP（浏览器 cookie、登录态）不会跨 agent/session 泄漏。
  实例被 LRU 淘汰后，下次 `list_mcps` 会按声明顺序重建。
- **声明继承**：未自行增删过 MCP 的 session 会从 `default_mcps` 拷贝一份副本；
  显式清空的 session 记为 `[]`（这是有意义的状态，会持久化进 `.mcp`）。
- **`agent_id` / `session_id` 仅关键字参数**：因两者都是普通字符串，强制关键字传参以防
  位置参数误命中另一个 session。`None` 等价于遗留的空串 `""`。

`.mcp` 文件 schema 升到 **v2**：

```jsonc
{
  "version": 2,
  "mcps": {
    "<agent_id>": { "<session_id>": [ /* MCPClient 配置 */ ] }
  }
}
```

- v1 文件（裸 JSON 数组）启动时自动迁移到 `("", "")` 遗留 id 下，下次写入即升级为 v2。
- 文件损坏或不可读时回退到 `default_mcps`，不阻断启动。

**LRU 容量管理**：`max_live_stateful_mcps`（构造参数，默认 `max(40, 2 * <stateful defaults>)`）
限制并发存活的 **stateful** MCP 实例总数（stateless 不占名额，无连接可回收）。超出时按
`_mcp_last_used` 时间戳 LRU 淘汰**其他** agent/session 的实例；发起方自身的集合永不被淘汰，
因此总能拿到完整默认集。

**`purge_session`**（基类方法）：删除 session 时由 `SessionService.delete_session` 调用，
关闭该 session 的 MCP 实例、清除其 `.mcp` 声明、删除其 `sessions/{session_id}/` offload 目录：

```python
await workspace.purge_session(agent_id=agent_id, session_id=session_id)
```

- best-effort——任何失败只记日志、不抛异常，因此 workspace 后端不可达也不会回滚已提交的删除。
- 逐个移除 MCP 不等价：清空的 session 会持久化为 `[]`，会永远留在 `.mcp` 里。

> 服务化路由的 MCP 端点（`/workspace/mcp/...`）已自动带 `agent_id`/`session_id`；
> `add_mcp` 重名时返回 **HTTP 409 Conflict**。

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
from agentscope.app.rag import CollectionPerKbManager, LocalBlobStore, S3BlobStore
from agentscope.app.storage import RedisStorage, AsyncSQLAlchemyStorage
from agentscope.app.message_bus import RedisMessageBus
from agentscope.app.workspace_manager import LocalWorkspaceManager

app = create_app(
    storage=RedisStorage(),               # 存储后端（必填）；也可用 AsyncSQLAlchemyStorage（v2.0.5+）
    message_bus=RedisMessageBus(),         # 消息总线（必填）
    workspace_manager=LocalWorkspaceManager(basedir="./workspaces"),  # 工作区管理器（必填）
    knowledge_base_manager=None,           # 可选：启用知识库服务层
    knowledge_parsers=None,                # 可选：Parser 列表或 media_type -> Parser dict
    knowledge_chunker=None,                # 可选：默认 ApproxTokenChunker()
    blob_store=None,                       # 可选：默认 LocalBlobStore("./blobs")
    enable_index_worker=True,              # True=API 进程内嵌索引 worker
    extra_credentials=[MyCredential],      # 自定义 Credential 类
    extra_middlewares=[MyFastAPIMiddleware()],   # FastAPI 中间件（如 CORS、限流）
    extra_agent_middlewares=my_middleware_factory,  # agent 中间件工厂
    extra_agent_tools=my_tool_factory,     # agent 工具工厂
    custom_subagent_templates=[...],       # 子智能体模板（v2.0.2+）
    custom_agent_cls=MyAgent,              # 自定义 Agent 子类（v2.0.2+）
    resource_access_policy=None,           # 跨用户资源共享策略（v2.0.4+）
    mcp_hubs=None,                         # MCP hub 列表（v2.0.5+）
    skill_hubs=None,                       # Skill hub 列表（v2.0.5+）
    channels=None,                         # Channel 适配器类列表（v2.0.6+，如 [FeishuChannel, DiscordChannel]）
)

# 独立运行
import uvicorn
uvicorn.run(app, host="0.0.0.0", port=8000)

# 挂载到已有 FastAPI
root.mount("/agentscope", app)
```

> ⚠️ 当前签名中 `storage` / `message_bus` / `workspace_manager` / `knowledge_base_manager` /
> `knowledge_parsers` / `knowledge_chunker` / `blob_store` / `enable_index_worker` 位于 `*` 之前；
> `extra_*` / `custom_*` / `title` / `version` 均为**仅关键字参数**。推荐全部用关键字传参。
>
> 注意区分两类 middleware 参数：
> - `extra_middlewares` —— 加到 **FastAPI 应用**层（ASGI 中间件，处理 CORS/鉴权/限流等 HTTP 层逻辑）。
> - `extra_agent_middlewares` —— 加到**每个 Agent 实例**的 `MiddlewareBase` 中间件工厂（拦截 reply/reasoning 等）。
>
> `extra_agent_middlewares` 是一个异步工厂，每个 chat turn 装配 agent 时调用一次，可返回
> user/session 专属中间件（鉴权、审计、租户隔离等）。签名有两种，服务端用 `inspect.signature` 探测
> 后按兼容形式调用——**旧的三参数工厂继续可用**（v2.0.6+ 新增第四个参数）：
> ```python
> # 旧形式（仍兼容）：(user_id, agent_id, session_id) -> Awaitable[list[MiddlewareBase]]
> # 新形式（v2.0.6+）：(user_id, agent_id, session_id, workspace) -> Awaitable[list[MiddlewareBase]]
> async def longterm_memory_factory(user_id, agent_id, session_id, workspace):
>     # workspace 是该 session 解析出的 WorkspaceBase，暴露 workdir / get_backend()，
>     # 便于挂文件系统级中间件（如 AgenticMemoryMiddleware）。
>     from agentscope.middleware import AgenticMemoryMiddleware
>     return [AgenticMemoryMiddleware(workdir=workspace.workdir, backend=workspace.get_backend())]
> ```
> 默认 PER_AGENT workspace 隔离下，同一 agent 跨 session 复用同一 workspace，故这样挂的长期记忆会跨 session 保留。
>
> 另有 AG-UI 协议适配层：服务会把内部 `AgentEvent` 转换成 [AG-UI](https://docs.google.com/document/d/1F8gZV5mcrzBB_Lu6p1Tq7v5K0eJ7r5K2/) 兼容的 SSE 事件流，方便接入标准 AG-UI 客户端。

### 服务化 RAG 参数

`knowledge_base_manager` 不为 `None` 时，知识库路由、文档上传、后台索引和 session 级 RAG 注入会启用。
服务层常用导入集中在 `agentscope.app.rag`：

```python
from agentscope.app.rag import (
    CollectionPerKbManager,
    LocalBlobStore,
    S3BlobStore,
    run_worker,
)
from agentscope.rag import QdrantStore, TextParser, PDFParser, ApproxTokenChunker

vector_store = QdrantStore(url="http://localhost:6333")
kb_manager = CollectionPerKbManager(storage=storage, vector_store=vector_store)

app = create_app(
    storage=storage,
    message_bus=RedisMessageBus(),
    workspace_manager=LocalWorkspaceManager(basedir="./workspaces"),
    knowledge_base_manager=kb_manager,
    knowledge_parsers=[TextParser(), PDFParser()],
    knowledge_chunker=ApproxTokenChunker(chunk_size=512, overlap=50),
    blob_store=LocalBlobStore(root_dir="./blobs"),
    enable_index_worker=True,      # 单进程/开发：API lifespan 内启动 IndexWorker + sweeper
)
```

BlobStore 负责保存上传文档原始字节，直到索引 worker 读取并入库：
- `LocalBlobStore(root_dir="./blobs")`：本地开发/单节点，URI 形如 `local://...`。
- `S3BlobStore(bucket=..., endpoint_url=...)`：S3 兼容对象存储，支持 AWS S3、MinIO、R2、OSS 等，URI 形如 `s3://bucket/key`。

`enable_index_worker=False` 用于 API 与索引 worker 分离的部署。独立 worker 从 `agentscope.app.rag`
导入 `run_worker`，复用同一组 storage/message_bus/knowledge_base_manager/blob_store/parser/chunker 配置。

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

### 存储后端（Storage）

`create_app` 的 `storage` 参数决定服务化的持久化后端（credentials / agents / sessions / 会话消息 /
schedules / teams / knowledge bases / knowledge documents 共 8 类记录）。两种实现：

```python
from agentscope.app.storage import RedisStorage, AsyncSQLAlchemyStorage

# Redis（默认，生产多进程部署常用）
storage = RedisStorage(...)

# SQLAlchemy（v2.0.5+）—— 支持 SQLite / PostgreSQL / MySQL 等 SQLAlchemy async 方言
storage = AsyncSQLAlchemyStorage(
    url="sqlite+aiosqlite:///./agentscope.db",   # 或 postgresql+asyncpg://... / mysql+aiomysql://...
    create_tables=True,   # True: __aenter__ 时 CREATE TABLE（开发/首次启动）
    auto_migrate=False,   # True: 走 Alembic upgrade head（生产迁移）
    engine=None,          # 可传入预构建的 AsyncEngine
    engine_kwargs=None,   # 透传给 create_async_engine
)
```

- `AsyncSQLAlchemyStorage` 依赖 `storage-sql` extra（`sqlalchemy[asyncio]>=2.0` + `alembic>=1.13`），
  具体 async 驱动（`aiosqlite`/`asyncpg`/`aiomysql` 等）**不在 extra 内**，由用户自装。
- 通过 `app.storage.__init__` 惰性导入（`__getattr__`），未装 sql extra 不会触发 SQLAlchemy 导入。
- 表定义刻意只用跨方言子集（plain `JSON`、无 generated column、upsert 按方言分发），兼容主流数据库。
- 附带 Alembic 迁移脚手架（`app/storage/_sql/_alembic/`），`auto_migrate=True` 时自动 `alembic upgrade head`。

> ℹ️ `StorageBase` 是两者的抽象基类，可自行子类化。普通 agent 开发者无需直接操作 storage——
> 由 `create_app` 启动的 FastAPI 服务在内部持有。

### 内置路由

- `agent_router` — 智能体管理
- `chat_router` — 对话（REST + SSE）
- `credential_router` — 凭据管理
- `model_router` — 模型配置
- `embedding_model_router` — 列出某 credential 类型下可用 embedding 模型（`GET /embedding-model/`，v2.0.5+）
- `tts_model_router` — TTS 模型配置（v2.0.2+）
- `schedule_router` — 定时任务
- `session_router` — 会话管理
- `workspace_router` — 工作区操作（含产物文件读取，v2.0.6+）
- `hub_router` — Hub 浏览与安装（v2.0.5+）
- `mcp_router` — 用户 MCP 库（v2.0.5+）
- `skill_router` — 用户 Skill 库（v2.0.5+）
- `channel_router` — IM 频道管理（`/channels`，v2.0.6+）
- `health_router` — `/health` 健康检查（v2.0.6+）

### /health 健康检查端点（v2.0.6+）

`GET /health` 报告服务存活性和**按组件的就绪状态**，不做任何 I/O——只检查 `app.state` 上挂载的对象，
所以探测它不会触碰 Redis、数据库或 workspace 后端：

```json
{
  "status": "ok",            // 或 "not_ready"
  "version": "2.0.6",
  "components": {
    "storage": "ok",
    "message_bus": "ok",
    "workspace_manager": "ok",
    "background_task_manager": "ok",
    "chat_run_registry": "ok",
    "scheduler_manager": "ok",
    "resource_access_service": "ok",
    "chat_service": "ok",
    "session_service": "ok",
    "mcp_hubs": "ok",        // 未配置则为 "disabled"
    "skill_hubs": "ok",      // 未配置则为 "disabled"
    "knowledge_base": "disabled"  // 未启用 KB 时为 disabled；启用但 worker 缺失为 not_ready
  }
}
```

- 组件状态取值：`ok` / `not_ready`（应存在但缺失）/ `disabled`（可选组件未配置）。
- 任一组件 `not_ready` 时整体 `status="not_ready"` 并返回 **HTTP 503**。
- **核心用途**：捕获 sub-app 误挂（`root.mount("/agentscope", app)`）。Starlette 不运行被挂载子 app 的
  lifespan，导致所有 lifespan 组件缺失、业务端点全部失效；此端点会把这种静默误配置变成一个明确的 503 信号。
- 需要 `X-User-ID` 鉴权（与其它业务端点一致）。

### workspace 产物文件读取与状态端点（v2.0.6+）

服务化后，前端可读取某 session 对应 workspace 内的产物文件（图表、报告等），并查询 session
当前聚焦的目录及其 git 状态。背后的解析、下载令牌签发、skill 上传、git 汇总统一收敛到
`WorkspaceService`（`app.state.workspace_service`，lifespan 内装配）：

```python
# 1. 列目录（返回 DirectoryListing，含解析后的绝对路径）
# GET /workspace/directories?agent_id=...&session_id=...&path=./
# -> {path: "/abs/resolved", entries: [{name, is_dir, size_bytes, updated_at}, ...]}
# path 可绝对可相对（相对于 workspace 根），不限定在根目录内（沙箱后端的可达 FS 就是沙箱）。
# 回传解析后的绝对 path，是因为相对调用者无法自行算出根（LocalBackend 是宿主目录，容器后端是沙箱内固定路径）。

# 2. session 工作目录状态（v2.0.6+）
# GET /workspace/status?agent_id=...&session_id=...
# -> WorkspaceStatus{workdir, cwd, git: GitStatus | None}
# cwd 取自 session 自身的 cwd（见 SessionConfig.cwd），而非查询参数——它是 session 的锚点。
# git 是尽力而为：非仓库/git 缺失/命令超时/后端不可达都收敛为 git=null，其余字段照常返回。

# 3. 申请一次性下载 token（浏览器原生下载无法带自定义 header，故用 URL 里的能力令牌）
# POST /workspace/files/download-token
#      {agent_id, session_id, path}  ->  {token, expires_at}
# token 默认 60s 有效，HMAC 签名绑定 (expires_at, user_id, path) 三元组，仅限该 path。

# 4. 下载文件
# GET /workspace/files?token=...   （带 token 时无需 X-User-ID header）
# 或 GET /workspace/files  + X-User-ID header（直接鉴权）
```

- `DirectoryListing`（v2.0.6+，`path` + `entries`）取代原先裸 `DirectoryEntry` 列表，`entries` 中的
  `DirectoryEntry`（`name` / `is_dir` / `size_bytes` / `updated_at`）由 workspace 后端的
  `BackendBase.scandir` 产生，目录恒返回 `size_bytes=null`。
- `GitStatus` 汇总分支 / HEAD SHA / ahead-behind / 增删行 / staged/unstaged/untracked/conflicted 文件数。
  通过 `git --no-optional-locks status --porcelain=v2 -z` + `git diff --shortstat HEAD` 收集，
  `--no-optional-locks` 避免与 agent 自身 Bash 抢 `.git/index.lock`；命令设 5s 超时。
- download-token 是**能力令牌而非身份**：只授权"该用户读取该路径一小段时间"，便于浏览器 `<a download>`
  或导航式下载把凭证放在 URL 里。换用 OAuth/JWT 时只改令牌签发来源，校验逻辑不变。

## Channel — 接入 IM 平台（v2.0.6+）

Channel（频道）把 AgentScope agent 接入即时通讯平台（飞书 / Discord），让 IM 群/私聊里的消息
直接驱动 agent 会话。这是**纯服务化层**功能——只在 `create_app` 启动的 FastAPI 服务里生效，
SDK 层的 `Agent` / `reply_stream` 本身不变。

### 启用

通过 `create_app` 的 `channels` 参数传入适配器**类**（不是实例）。传入至少一个类即开启整个
channel 子系统（路由、生命周期调度、回复转发）；为 `None`（默认）则该功能完全关闭：

```python
from agentscope.app import create_app
from agentscope.app.channel import DiscordChannel, FeishuChannel

app = create_app(
    storage=storage,
    message_bus=message_bus,
    workspace_manager=ws_manager,
    channels=[DiscordChannel, FeishuChannel],   # v2.0.6+
)
```

内置两个适配器：

| 类 | `channel_type` | 认证字段 | 连接方式 |
|---|---|---|---|
| `FeishuChannel` | `"feishu"` | `app_id` + `app_secret` | lark-oapi WebSocket 长连接（独立线程 + 自有 loop） |
| `DiscordChannel` | `"discord"` | `bot_token` + `application_id` | discord.py gateway WebSocket（跑在 app loop） |

### 数据模型

频道实例本身是持久化记录 `ChannelRecord`，由 REST CRUD 管理；运行中的适配器实例是它的投影。
核心字段：

```python
from agentscope.app.storage import (
    ChannelRecord, ChannelBinding, RoutingConfig,
    SessionSettings, SessionScope,
)

# 一条路由规则：匹配某事件 → 决定由哪个 agent 处理 + 如何分组会话
ChannelBinding(
    match_key="chat_id",        # 匹配 event.chat_id；也可是 "user_id" 或 event.metadata 里的 key
    match_value="oc_xxx",       # 精确匹配；"*" 表示 catch-all（必须且只能在最后一条）
    agent_id="agent-uuid",      # 命中后由哪个 agent 处理
    session_scope=SessionScope.PER_CHAT,   # 会话分组粒度
)

# 路由配置：有序规则列表，首条命中生效；末条必须是 catch-all
RoutingConfig(bindings=[..., ChannelBinding(match_value="*", agent_id="默认agent")])

# 派生会话设置
SessionSettings(
    chat_model_config={...},              # 必填，频道创建的会话无全局默认模型
    fallback_chat_model_config=None,      # 可选 fallback 模型
    permission_mode="default",            # 默认 default——工具审批走交互卡片而非自动拒绝
)
```

- `SessionScope.PER_CHAT`：每个群一个会话（私聊天然 per-user）。
- `SessionScope.PER_CHAT_USER`：群内每个成员隔离成独立会话。
- `(agent_id, session_id)` **由纯函数从事件确定性派生**（`uuid5` of `channel:agent:scope_key`），
  无持久化的 channel→session 映射表——所以多节点零协调得到同一结果，会话创建幂等。

### 架构：入站数据面 vs 出站转发

Channel 子系统分两层，避免双向耦合：

```
IM 平台 ──消息──▶ Channel(适配器) ──emit──▶ ChannelGateway ──▶ 派生 (agent,session) ──▶ run
                  (维持长连接、归一化事件)     (薄编排：路由/会话/媒体聚合)                 │
                  ◀──send_response─── ChannelLifecycleDispatcher ◀──订阅 run 事件流─────┘
                                       (出站转发：把回复折叠后送回平台)
```

- **`ChannelGateway`**（入站）：`process(event)` 是唯一入口。消息 → 路由到 `(agent_id, session_id)`
  并入队为一个 run（空闲时是新的 user turn；运行中则注入为 `HintBlock` inbox 提示）。卡片点击 →
  从 session state 读权威的待审批工具调用并 resume run。**不收集也不发送回复**。
- **`ChannelLifecycleDispatcher`**（出站 + 生命周期）：每节点一个，把本节点的活跃适配器集合
  与 storage 对账（lifecycle 通知 + 定期 sweep 自愈）。它还订阅每个频道相关 run 的事件流，
  无缝地（先订阅、再回放日志、按 `entry_id` 去重）喂给本地 channel 的 `send_response`——
  channel 负责折叠渲染（流式卡片/确认卡片），dispatcher 只喂数据。
- **`ChannelService`**：无状态 CRUD（校验、写记录、发 lifecycle 通知），不持有任何运行实例。

> 这种拆分让**定时/后台 run** 的输出也能送达频道——不只是入站消息触发的回复。

### 权限审批走交互卡片

频道会话默认 `permission_mode="default"`。当一个工具调用需要用户确认（`RequireUserConfirmEvent`）时：

1. channel 在平台上渲染一张**确认卡片**（飞书 interactive card / Discord button），卡片上**钉住**
   当时的 `(agent_id, session_id, tool_call_id)`。
2. 用户点击 → channel 发出 `ChannelConfirmationResultEvent`，经同一 gateway 入口。
3. gateway 优先用卡片上钉定的目标（而非重新路由），从 session state 读**权威的**待审批工具调用，
   resume 那个 run。过期的/重复的点击找不到 ASKING 状态的工具调用，自然幂等忽略。

### channel 暴露给 agent 的工具

每个 channel 可通过 `list_tools(workspace)` 暴露平台工具给 agent（例如「给当前对话以外的某个
群/用户发消息」）。飞书内置 5 个工具，组成发现 → 发送的闭环：

| 工具 | 作用 |
|---|---|
| `ListChats` / `ListChatMembers` | 发现：返回 `receive_id` + `receive_id_type` |
| `SendMessage` / `SendFile` / `SendImage` | 发送：消费 discovery 给出的 id 定向送达 |

> ℹ️ 这些工具由频道实例按需注入到该会话的 workspace，普通 SDK agent 开发者无需手动注册。

### 自定义平台

继承 `ChannelBase` 即可接入新平台——子类在**类自身**上声明平台元数据，service 据此自动渲染
前端表单、做 bot 唯一性校验、按记录构造实例：

```python
from agentscope.app.channel import ChannelBase, ChannelCapability

class MyChannel(ChannelBase):
    channel_type = "myplatform"          # 唯一类型 id
    display_name = "My Platform"
    platform_bot_id_field = "bot_id"     # 用于拒绝同一 bot 绑两个频道

    class Credentials(BaseModel):        # 凭据字段；password 格式会脱敏
        bot_id: str
        token: str = Field(json_schema_extra={"format": "password"})

    class Config(BaseModel):             # 非密的行为开关
        only_at_reply: bool = True

    capabilities = ChannelCapability(text=True, markdown=False, interactive=True)

    def __init__(self, channel_id, credentials, config): ...
    async def start_listening(self, emit): ...   # 维持长连接，归一化事件后 await self._emit(event)
    async def send_response(self, event, events): ...  # 消费 run 事件流并送达平台
```

### REST API

| 方法 | 路径 | 作用 |
|---|---|---|
| GET | `/channels/types` | 列出已注册的平台类型 + JSON Schema（前端渲染表单用） |
| GET | `/channels/` | 列出当前用户的频道（凭据脱敏） |
| POST | `/channels/` | 创建频道（拒绝同一 bot 重复绑定，返回 409） |
| GET/PATCH/DELETE | `/channels/{id}` | 详情 / 更新路由·会话·配置（凭据与类型不可变）/ 删除 |
| POST | `/channels/{id}/enable` · `/disable` | 启停 |
| GET | `/channels/{id}/status` | 实时连接状态（`stopped/connecting/connected/retrying/failed`） |
| GET | `/channels/{id}/sessions` | 该频道派生的会话 |
| GET | `/channels/{id}/chat_ids` | 路由配置可用的群（平台列表 ∪ 被动见过 inbound 的） |

> ⚠️ Channel 功能属于服务化部署层，依赖 Redis 或 SQLAlchemy storage + MessageBus。
> 普通单 agent SDK 开发无需关心。多节点部署时每个节点都会 host 所有启用频道（无分片）。



Hub 让 agent service 从外部注册中心**浏览 → 安装到个人库（library）→ 拉入 workspace**，
无需手写 `MCPClient` 配置或手动复制 skill 文件。内置两个 hub：

| Hub | 注册内容 | 默认地址 | 认证 |
|---|---|---|---|
| `GitHubMCPHub` | MCP server（GitHub 官方 MCP Registry） | `https://api.mcp.github.com` | 免认证，token 仅提升限流 |
| `ClawSkillHub` | Skill（[ClawHub](https://clawhub.ai) 公共 registry） | `https://clawhub.ai` | 免认证，token 仅提升限流 |

Hub 通过 `create_app` 注入（不在配置文件中）：

```python
import os
from agentscope.app import create_app
from agentscope.app.hub import GitHubMCPHub, ClawSkillHub

app = create_app(
    storage=...,
    message_bus=...,
    workspace_manager=...,
    mcp_hubs=[GitHubMCPHub()],                              # 默认连 GitHub MCP Registry
    skill_hubs=[ClawSkillHub(api_token=os.getenv("CLAWHUB_API_TOKEN"))],
)
```

- `hub_id` 必须匹配 `[a-zA-Z0-9_-]+`（用作 URL path segment），同种 hub id 重复会抛 `ValueError`。
- hub 的 HTTP client 在进程生命周期内只建一次（lifespan `__aenter__`），跨请求复用。

### Card 概念

Hub 上的一个条目叫 **card（卡片）**，是元数据模板，不是可直接使用的客户端：

- `MCPCard` 的 `config_template` 带 `${...}` 占位符（API key 等），由用户在安装时填值，
  占位符定义在 `inputs_schema`（JSON Schema，密钥字段标 `writeOnly:true` + `format:password`）。
- `SkillCard` 无配置项——安装 skill 只是把文件复制进 workspace，没有输入表单。
- card 有两个**故意分离**的标识：`id`（hub 本地寻址，可任意不透明字符串）与
  `name`（安装后的客户端/skill 名，必须匹配 `[a-zA-Z0-9_-]+`，会组成工具名 `mcp__{name}__{tool}`）。
- card 在不同 hub 间不合并，全局由 `(hub_id, card_id)` 二元组标识。

### 直接用 Hub 客户端（Python 异步 API）

```python
from agentscope.app.hub import GitHubMCPHub, ClawSkillHub, MCPCard, SkillCard

mcp_hub = GitHubMCPHub()
page = await mcp_hub.list_mcps(user_id="u1", q="github", limit=20)  # MCPHubPage
card = await mcp_hub.get_mcp(user_id="u1", card_id="github-mcp")    # MCPCard

skill_hub = ClawSkillHub()
page = await skill_hub.list_skills(user_id="u1", q="browser")        # SkillHubPage
card = await skill_hub.get_skill(user_id="u1", card_id="runware/music")  # SkillCard
archive = await skill_hub.download(user_id="u1", card_id="runware/music")  # SkillArchive
```

抽象方法（所有 hub 子类须实现）：
- `MCPHubBase.list_mcps(user_id, q=None, cursor=None, limit=20) -> MCPHubPage`
- `MCPHubBase.get_mcp(user_id, card_id) -> MCPCard`
- `SkillHubBase.list_skills(user_id, q=None, cursor=None, limit=20) -> SkillHubPage`
- `SkillHubBase.get_skill(user_id, card_id) -> SkillCard`
- `SkillHubBase.download(user_id, card_id, version=None) -> SkillArchive`

> ℹ️ `MCPHubBase` / `SkillHubBase` 是抽象基类，可自行子类化对接私有 registry。
> `SkillArchive` 是 `NamedTuple(format, stream)`，`format` 为 `"zip"|"tar"|"tar.gz"`，
> `stream` 是字节流，可流式写入 workspace 而不必整体缓存。

### HTTP 路由（推荐用法）

启动 service 后调用，`user_id` 通常由鉴权中间件注入：

| 方法 | 路径 | 作用 |
|---|---|---|
| GET | `/hub/mcp` | 列出所有已注册 MCP hubs |
| GET | `/hub/mcp/{hub_id}/cards` | 浏览/搜索（query: `q`/`cursor`/`limit` 1-200） |
| GET | `/hub/mcp/{hub_id}/cards/{card_id}` | 取单个 MCP card（含需用户填的 inputs） |
| POST | `/hub/mcp/{hub_id}/cards/{card_id}/install` | 安装 MCP 到个人库 |
| GET | `/hub/skill` | 列出所有已注册 skill hubs |
| GET | `/hub/skill/{hub_id}/cards` | 浏览/搜索 skill |
| GET | `/hub/skill/{hub_id}/cards/{card_id}` | 取单个 skill card（含 `SKILL.md` 正文） |
| POST | `/hub/skill/{hub_id}/cards/{card_id}/install` | 安装 skill 到个人库 |

**MCP 安装**（`POST .../install`，body 为 `InstallMCPRequest`）：

```json
{
  "name": "my-github-mcp",      // 可选，默认 card.name；冲突时换名，同名返回 409
  "values": {"GITHUB_TOKEN": "ghp_xxx"}   // 对 card.inputs_schema 的填值
}
```

安装时用 `render_mcp(card, values, name)` 把模板渲染成可连接的 `MCPClient`，
快照入库（`MCPRecord` 含 `hub_id`/`card_id`/`version`/`values`）。
**安装时不测连接**——错的 key 在首次使用时才暴露。

**Skill 安装**（`POST .../install?name=...`）只记录 card 元数据（`SkillRecord` 含 `SKILL.md` 正文快照），
**不在此下载归档**——文件在拉入 workspace 时才按需从 hub 重新获取。

### 拉入 workspace

library 里的 MCP/skill 需拉入会话 workspace 才生效：

| 方法 | 路径 | 作用 |
|---|---|---|
| POST | `/workspace/mcp/from-library` | 把库里的 MCP 拉入会话 workspace |
| POST | `/workspace/skill/from-library` | 把库里的 skill 拉入会话 workspace |
| POST | `/workspace/skill/upload` | 浏览器文件夹上传安装 skill（不经 hub，限单文件 50 MiB / 总 500 MiB / 100 文件） |

### ClawHub 的 card id 按 owner 作用域

ClawHub 上 **slug（skill 名）并不唯一**——多个 publisher 可能用同一个 slug（如 `music`）。
裸 slug 有歧义时，ClawHub API 会返回 `409 AMBIGUOUS_SKILL_SLUG`。

为此，从 search/detail 端点取到的 card（含 owner 信息）其 `id` 形如 `"owner/slug"`（如 `runware/music`），
下载时带 `ownerHandle` 参数精确定位；`name` 仍是裸 slug（成为 workspace 目录名，不能带分隔符）。
catalog 浏览端点不返回 owner，歧义在 install/拉取时由 409 兜底提示。

> ℹ️ v2.0.6 起，`409 AMBIGUOUS_SKILL_SLUG` 的响应体会被解析成可读提示（列出所有同名 publisher 的 ref，
> 引导用户按名搜索后选装），而非原始 JSON。

> ℹ️ Hub 功能完全通过 FastAPI HTTP 路由暴露，**无 CLI 命令**。
> 前端 WebUI 有完整的 hub 浏览/安装界面。普通 agent 开发者（非服务化部署）一般无需直接用 hub 类。

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

### AgentInvite — 邀请已有 agent 入队（v2.0.4+）

`AgentCreate` 是从 `SubAgentTemplate` **新建** worker；`AgentInvite`（leader 内置工具）则是**借用**一个
**已存在的、用户拥有的 agent** 入队：被邀请的 agent 保留自己的 system_prompt / model / workspace / MCP /
skills / 权限，只为团队新建一个 session 来承载并行的团队对话。团队解散或 leader 被删除时，**只清理借用的
session，原 agent 记录不受影响**，仍可独立使用。

哪些 agent 能被邀请，由其 `InviteConfig` 控制（在 `create_app` 的 agent CRUD 接口里配置）：

| `InviteConfig` 字段 | 说明 |
|---|---|
| `invitable: bool` | 是否允许被 `AgentInvite` 借用，默认 `False` |
| `invite_description: str \| None` | 给 leader LLM 看的简介，`invitable=True` 时**必须非空** |

```python
# 通过 REST 创建 agent 时设置（也可在 UpdateAgentRequest 中更新）
POST /agents
{
  "name": "DBA助手",
  "system_prompt": "...",
  "invite_config": {
    "invitable": true,
    "invite_description": "精通 PostgreSQL/MySQL 的 DBA，可处理建表、索引、慢查询优化"
  }
}
```

- leader 调用 `AgentInvite` 时，`target` 参数是形如 `"AgentName@9f3c1a20"`（名字 + agent_id 前缀）的句柄。
- 邀请产生 `role="invited"` 的 team member，与 `AgentCreate` 产生 `role="team"` 的成员区分。
- HITL 事件投影对 invited 成员同样生效。

### Session Status 端点（v2.0.4+）

`GET /sessions/{session_id}/status?agent_id=...` 返回 session 的统一状态（合并"集群活跃度"和"持久化挂起态"
两路信号为单一枚举），供前端轮询一个 session 当前处于什么阶段：

| `SessionStatus` | 含义 |
|---|---|
| `running` | 有 worker 正在运行该 session |
| `awaiting_permission` | 无 worker 运行，末尾有工具调用等待用户确认（优先级高于 external） |
| `awaiting_external_result` | 无 worker 运行，末尾有工具调用等待外部执行结果 |
| `idle` | 无 worker 运行，且无挂起工具调用 |

v2.0.6+ 起，`GET /sessions`（列表）返回的每个 `SessionView` 也直接内联了 `status` 字段，
前端渲染侧边栏时无需再逐个轮询状态端点。同时该端点的 `is_running` 被标记为 **deprecated**——
它仅在 worker 持有 run lease 时为 `True`，而 session 停在确认提示上时读作 `False`
（明明在等用户）；应改用 `status`：

```json
{
  "session": { ... },            // 已裁剪：context/summary/tool_context 被清空（见下）
  "is_running": false,           // deprecated，用 status
  "status": "awaiting_permission",
  "team": null
}
```

> ⚠️ `GET /sessions` 为减少侧边栏渲染负载，对返回的 `session` 记录做了**裁剪**（v2.0.6+）：
> `state.context`（模型看到的对话）、`state.summary`（压缩摘要）、`state.tool_context`
> （缓存了它读过的每个文件）一律清空——否则列出 20 个 session 就要传输 20 份完整记录。
> `permission_context` / `tasks_context` 因体量小而保留，前端面板可据此初始化。
> 读取某 session 的消息请走 `GET /sessions/{id}/messages`。

### Session 工作目录（cwd，v2.0.6+）

`SessionConfig.cwd` 记录 session 当前聚焦的目录（绝对路径或相对于 workspace 根的相对路径，
`None` 表示根）。通过 `PATCH /sessions/{session_id}` 更新（body 的 `cwd` 字段，传 `null` 重置到根、
省略则保持不变）。

- 它**纯粹是查看锚点**：不改变 `Bash` / `Glob` / `Grep` 的实际执行位置。
- 写入时不做校验——这个值只是"一个要去查看的地点命名"，路径消失会在真正读取时报错而非阻断修改。
- 相对值刻意保持相对：workspace 根是后端相关的（LocalBackend 是宿主目录，容器后端是沙箱内固定路径），
> 只能异步解析，在这里去规范化反而会在 session 跨后端迁移时变陈旧。使用点用
> `backend.abspath(cwd, cwd=workspace.workdir)` 解析（两种形式都能处理）。
- 配合 `GET /workspace/status` 读取该 cwd 的 git 状态（见 workspace 端点一节）。

> ⚠️ `PATCH /sessions/{session_id}` 在有 chat run 持有 session 时返回 **409 Conflict**：
> agent 在 run 开始时一次性快照配置，运行中修改当前 reply 会被忽略，且写入会与 run 自身的状态持久化竞争。

### 跨用户资源共享（ResourceAccessPolicy，v2.0.4+）

服务层把"谁能看/改谁的资源"抽成**可注入的策略基类**——这是服务端（REST 层）的扩展点,不是 SDK 层 API。
AgentScope **自身不存储** group/org/share 记录,group/org 的成员表、共享规则由部署方自行实现
（从配置、LDAP、外部 IAM 等读取）。默认实现 `DenyAllResourceAccessPolicy` 拒绝一切跨用户访问,
保持历史 owner 隔离行为。

**可共享的三种资源**:`credential` / `agent` / `knowledge_base`。

```python
from agentscope.app.access import (
    ResourceAccessPolicyBase,   # 抽象基类,部署方继承它
    ResourceKind,               # CREDENTIAL / AGENT / KNOWLEDGE_BASE
    ResourcePermission,         # READ / EDIT(EDIT 蕴含 READ)
    ResourceRef,                # 策略返回的跨 owner 资源引用
)

class OrgPolicy(ResourceAccessPolicyBase):
    """示例:按组织成员表共享资源。group/org 成员关系存在你自己的存储里。"""
    async def list_accessible(
        self, viewer_id: str, kind: ResourceKind, storage,
    ) -> list[ResourceRef]:
        # 查你自己的成员表,返回该 viewer 能看到的(其他 owner 的)资源引用
        return [
            ResourceRef(
                kind=kind,
                owner_id="alice",           # 资源所有者的 user_id
                resource_id="cred-openai",  # 目标资源 id
                permission=ResourcePermission.READ,
            ),
            ...
        ]

    # 可选:更细粒度的写权限判断(默认按 list_accessible 返回的 permission)
    async def can_edit(self, viewer_id, kind, owner_id, resource_id, storage) -> bool:
        ...

app = create_app(
    ...,
    resource_access_policy=OrgPolicy(),  # 不传则用 DenyAllResourceAccessPolicy
)
```

行为细节:
- 注入策略后,**既有 CRUD 端点自动透明支持跨 owner 资源**——无需新增 `/groups`、`/share` 端点。
  `GET /credential` / `GET /agent` / `GET /knowledge_bases` 会把"自有(editable=True)"和
  "策略授予的跨 owner 项"合并返回(共享 credential 的 `data` 字段对非所有者脱敏,只剩 `type`/`name`)。
- `PATCH` / `DELETE` 先经 `resolve_for_edit` 拿到 `owner_id`,再通过所有者的 storage key 回写——
  即「shared editor writes back through the owning user's storage key」。
- runtime 路径(chat / embedding / model / tts_model / knowledge_base / toolkit router)改用
  `resolve_credential/resolve_agent/resolve_knowledge_base`,拿到未脱敏的原始记录正常跑 agent。
- `source == "team"` 的 agent(团队 worker)被显式排除在共享之外。
- 共享关系完全由策略实现决定,不落进 AgentScope 自己的存储 schema。`TeamMember.owner_id` 字段
  就是为此预留,文档原话:"a future admin-share layer can slot in without a schema migration"。
