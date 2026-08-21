# 工具 (Tool) & MCP — v2

## ToolBase — 工具基类

v2 中工具是 `ToolBase` 的子类，不再是普通函数。每个工具有权限检查、状态注入等能力。

### ToolBase 核心属性

```python
class ToolBase(ABC):
    name: str                     # 工具名称
    description: str              # 描述
    input_schema: dict            # JSON Schema
    is_concurrency_safe: bool     # 是否并发安全
    is_read_only: bool            # 是否只读（影响权限）
    is_external_tool: bool        # 是否外部工具（不在本地执行）
    is_state_injected: bool       # 是否需要注入 AgentState
    is_mcp: bool                  # 是否 MCP 工具
```

### ToolBase 核心方法

```python
async def call(self, **kwargs) -> ToolChunk | AsyncGenerator[ToolChunk, None]
async def __call__(self, **kwargs) -> ToolChunk | AsyncGenerator[ToolChunk, None]
async def check_permissions(tool_input, context) -> PermissionDecision
async def match_rule(rule_content, tool_input) -> bool
async def generate_suggestions(tool_input) -> list[PermissionRule]
async def check_read_only(tool_input) -> bool   # EXPLORE 模式下用于判断是否只读
```

> v2.0.3 起，工具逻辑应重写 `call()`（而非 `__call__`）。`__call__` 已变为门面：它负责按
> 注册顺序套用工具级中间件（见下文 `ToolMiddlewareBase`），最终在最内层调用 `call()`。
> - 自定义工具：实现 `async def call(self, **kwargs)`。
> - 外部工具（`is_external_tool=True`）：由 agent 产生 external tool call 事件，不在本地执行，
>   因此不需要也不应实现 `call()`。

### 内置工具

```python
from agentscope.tool import Bash, Read, Write, Edit, Glob, Grep, ResetTools, PowerShell

# 这些工具都有完善的权限检查和文件缓存
toolkit = Toolkit(tools=[Bash(), Read(), Write(), Edit()])
```

### PowerShell 工具（v2.0.5+，Windows）

`PowerShell` 是与 `Bash` 平级的 `ToolBase` 子类，用于 Windows 环境。探针优先 `pwsh` 后
`powershell.exe`，用 `-EncodedCommand`（UTF-16-LE base64）规避引号转义问题，会话状态不跨命令保留。
`check_permissions` 恒返回 `ASK`（保守策略，不建议 allow 规则）。

```python
from agentscope.tool import PowerShell

toolkit = Toolkit(tools=[PowerShell(), Read(), Write(), Edit()])
```

> ℹ️ 通常无需手动构造 `PowerShell`——`LocalWorkspace.list_tools()` 在 `os.name == "nt"` 时
> 自动把 shell 工具替换为 `PowerShell`（其余 `Edit/Glob/Grep/Read/Write` 不变），非 Windows 仍用 `Bash`。
> 其他沙箱后端（Docker/E2B 等）未做此集成。

### 任务管理工具

```python
from agentscope.tool import TaskCreate, TaskGet, TaskList, TaskUpdate

toolkit = Toolkit(tools=[Bash(), TaskCreate(), TaskList()])
```

### 创建自定义工具（两种方式）

**方式 1：FunctionTool — 包装普通函数（推荐，最简单）**

把一个带类型注解和 docstring 的 Python 函数直接传给 `FunctionTool`，框架自动从签名/docstring 生成 JSON Schema：

```python
from agentscope.tool import FunctionTool

def add_numbers(a: int, b: int) -> str:
    """Add two numbers together.

    Args:
        a: The first number
        b: The second number
    """
    return str(a + b)

toolkit = Toolkit(tools=[FunctionTool(add_numbers)])
```

- 支持同步/异步函数，返回 `ToolChunk`（非流式）或 `AsyncGenerator[ToolChunk]`/`Generator[ToolChunk]`（流式）。
- 也可以直接返回 `str`、`dict`、列表等普通对象，`FunctionTool` 会自动转换成文本 `ToolChunk`。
- `FunctionTool(func, name=..., description=..., is_concurrency_safe=True, is_read_only=False, ...)` 可覆盖默认属性。
- v2.0.7+ 支持传入 `input_schema` 覆盖自动生成的 schema：JSON schema dict 或 pydantic `BaseModel` 子类
  （经 `model_json_schema()` 转换并移除 `title` 字段）；为 `None`（默认）时仍从函数签名/docstring 生成，
  约束（枚举、取值范围等）用 `typing.Literal` 或 `typing.Annotated[int, Field(ge=0, le=10)]` 表达。

**方式 2：继承 ToolBase — 需要权限检查/状态注入等高级能力**

```python
from agentscope.tool import ToolBase, ToolChunk
from typing import Any

class MyTool(ToolBase):
    name: str = "my_tool"
    description: str = "做某件事"
    input_schema: dict = {
        "type": "object",
        "properties": {"x": {"type": "string"}},
        "required": ["x"],
    }
    is_read_only: bool = True        # 影响权限（EXPLORE 模式下自动放行）

    async def call(self, **kwargs: Any) -> ToolChunk:
        # 执行工具逻辑，返回 ToolChunk
        ...
```

> 子类化时需提供 `name`/`description`/`input_schema` 并实现 `async def call`（v2.0.3+）；
> 权限相关方法（`check_permissions`/`match_rule`/`generate_suggestions`）可选，见 permissions 文档。

## ToolMiddlewareBase — 工具级洋葱中间件（v2.0.3+）

每个 `ToolBase` 实例可在构造时挂一组 `ToolMiddlewareBase`，以洋葱方式包裹 `call()` 的执行：
最先注册的是最外层，它的前置逻辑先跑、后置逻辑最后跑。流式/非流式工具被统一成 `AsyncGenerator[ToolChunk]`，
中间件无需区分。

```python
from agentscope.tool import ToolMiddlewareBase, ToolChunk
from typing import Any, AsyncGenerator, Callable

class LoggingToolMiddleware(ToolMiddlewareBase):
    async def on_tool_call(
        self,
        tool,              # ToolBase 实例
        input_kwargs,      # dict[str, Any]，工具入参
        next_handler,      # Callable[..., AsyncGenerator[ToolChunk, None]]
    ) -> AsyncGenerator[ToolChunk, None]:
        print(f"调用 {tool.name}，入参 {input_kwargs}")
        async for chunk in next_handler(**input_kwargs):
            yield chunk
        print(f"{tool.name} 执行结束")

# 挂载方式：构造工具时传 middlewares
tool = MyTool(middlewares=[LoggingToolMiddleware()])
```

典型用途：工具级日志/trace、入参改写、结果脱敏、缓存、限流。与 agent 级的
`MiddlewareBase`（拦截 `on_acting` 等阶段）互补：工具级中间件直接作用于**单个工具的执行**，
不经过 agent 的事件循环。

## ToolChunk 和 ToolResponse

```python
from agentscope.tool import ToolChunk, ToolResponse
from agentscope.message import TextBlock, DataBlock, ToolResultState

# 工具返回增量块
ToolChunk(
    content=[TextBlock(text="部分结果")],
    state=ToolResultState.RUNNING,
    is_last=False,               # 是否最后一个块
    id="call-xxx",               # 可选：对应 ToolCallBlock.id，用于配对
    metadata={},                 # 可选：工具元数据
)

# 最终累积的完整结果
ToolResponse(
    content=[TextBlock(text="完整结果")],
    state=ToolResultState.SUCCESS,
    id="call-xxx",               # 可选
    metadata={},                 # 可选
)
```

工具函数可以返回：
- 单个 `ToolChunk`（非流式）
- `AsyncGenerator[ToolChunk]`（流式）
- `Generator[ToolChunk]`（同步流式）

> ℹ️ 流式累积时，`ToolResponse` 的状态按后到的 chunk 更新（`RUNNING` → `SUCCESS`/`INTERRUPTED`/`DENIED`/`ERROR`）。
> 但 **`ERROR` 是终态优先级最高**：一旦进入 `ERROR`，后续的 `INTERRUPTED` / `DENIED` chunk 不会再覆盖它（v2.0.6+ 修复）。

## Toolkit

```python
from agentscope.tool import Toolkit, ToolGroup
from agentscope.mcp import MCPClient

# 基本创建
toolkit = Toolkit(
    tools=[Bash(), Read(), Write()],     # basic 组
    mcps=[mcp_client],                   # basic 组的 MCP
    tool_groups=[                         # 额外工具组
        ToolGroup(
            name="browser",
            description="网页浏览工具组",
            instructions="使用前先 activate",
            tools=[NavigateTool()],
            mcps=[browser_mcp],
        ),
    ],
)

# 获取工具 Schema
schemas = await toolkit.get_tool_schemas()            # 当前激活的工具
schemas = await toolkit.get_tool_schemas(groups=["browser"])  # 指定组

# 执行工具
async for chunk_or_response in toolkit.call_tool(tool_call_block, agent_state):
    ...

# 查询工具
tool = await toolkit.get_tool("bash")
tool = await toolkit.check_tool_available("bash", activated_groups=["basic"])

# Skill 相关
instructions = await toolkit.get_skill_instructions()  # 获取 skill 提示
instructions = await toolkit.get_skill_instructions(activated_groups=["basic"])  # 注意参数名
```

## ToolGroup

工具组允许按功能分组管理工具，默认只有 `"basic"` 组始终激活：

```python
from agentscope.tool import ToolGroup

group = ToolGroup(
    name="data_analysis",       # 组名（"basic" 保留）
    description="数据分析工具",  # 必填（basic 除外）
    instructions="使用指南...",   # 激活时返回给智能体的说明
    tools=[QueryTool()],
    mcps=[db_mcp],
)
```

## MCPClient — 统一 MCP 客户端

v2 使用单一 `MCPClient` 类，通过 `is_stateful` 和 `mcp_config` 区分：

```python
from agentscope.mcp import MCPClient, StdioMCPConfig, HttpMCPConfig

# 有状态 HTTP（需 connect/close）
http_stateful = MCPClient(
    name="weather",
    is_stateful=True,
    mcp_config=HttpMCPConfig(url="https://api.weather.com/mcp"),
)
await http_stateful.connect()
tools = await http_stateful.list_tools()
await http_stateful.close()

# 无状态 HTTP（无需 connect/close）
http_stateless = MCPClient(
    name="search",
    is_stateful=False,
    mcp_config=HttpMCPConfig(url="https://search.example.com/mcp"),
)
tools = await http_stateless.list_tools()  # 自动创建临时会话

# StdIO（必须有状态）
stdio_client = MCPClient(
    name="filesystem",
    is_stateful=True,
    mcp_config=StdioMCPConfig(command="mcp-server-filesystem"),
)
await stdio_client.connect()
```

### MCPClient 方法

```python
await client.connect()              # 连接（仅 stateful）
await client.close()                # 关闭（仅 stateful）
client.is_connected                 # 是否已连接
tools = await client.list_tools()   # 获取 ToolBase 列表
tool = await client.get_tool(name)  # 获取单个 MCPTool
raw = await client.list_raw_tools() # 获取原始 mcp.types.Tool
```

> v2.0.6+：有状态客户端支持重连——`connect() → close() → connect()` 会重建底层 transport
> （stdio / streamable HTTP / SSE 均为一次性 context manager，每次连接前重建），断线后可复用
> 同一 `MCPClient` 实例重新 `connect()`。

### 工具过滤

```python
MCPClient(
    name="weather",
    is_stateful=False,
    mcp_config=HttpMCPConfig(url="..."),
    enable_tools=["get_weather", "get_forecast"],   # 只启用这些
    disable_tools=["internal_debug"],                # 禁用这些
    execution_timeout=30.0,                          # 超时秒数
)
```

## 注册到 Toolkit

```python
# 方式 1：构造时直接传入 MCP
toolkit = Toolkit(mcps=[mcp_client])

# 方式 2：放在 ToolGroup 中
toolkit = Toolkit(
    tool_groups=[
        ToolGroup(name="maps", description="地图服务", mcps=[map_mcp]),
    ],
)
```
