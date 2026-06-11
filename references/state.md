# 状态管理 (AgentState) — v2

## AgentState (Pydantic BaseModel)

v2 取消了独立的 memory/session 模块，改用 `AgentState` 统一管理：

```python
from agentscope.state import AgentState

state = AgentState()
```

### 字段说明

```python
class AgentState(BaseModel):
    session_id: str                      # 会话 ID（自动生成）
    summary: str | list[TextBlock|DataBlock]  # 压缩后的上下文摘要
    context: list[Msg]                   # 对话上下文（替代 v1 的 Memory）
    reply_id: str                        # 当前 reply 的 ID
    cur_iter: int                        # 当前推理-行动迭代次数
    permission_context: PermissionContext  # 权限上下文
    tool_context: ToolContext            # 工具缓存和激活的工具组
    tasks_context: TaskContext           # 任务列表
```

### context — 对话上下文

替代 v1 的 Memory 模块。`context` 是 `list[Msg]`，直接存储对话消息：

```python
# context 由 Agent 内部自动管理
# 用户消息、助手回复、工具调用/结果都存在 context 中

state.context    # -> [UserMsg, AssistantMsg, ...]
```

### summary — 压缩摘要

当 token 超过阈值时，Agent 自动压缩旧的 context 并生成 summary：

```python
# summary 会拼接到 context 之前传给 LLM
state.summary    # -> "压缩后的摘要文本"
```

### 序列化/反序列化

AgentState 是 Pydantic BaseModel，直接序列化：

```python
# 导出
state_dict = state.model_dump()
json_str = state.model_dump_json()

# 恢复
state = AgentState.model_validate(state_dict)
state = AgentState.model_validate_json(json_str)
```

## ToolContext

```python
class ToolContext(BaseModel):
    max_cache_files: int = 100         # 最大缓存文件数
    max_cache_bytes: float = 25000     # 最大缓存字节数
    read_file_cache: list[ReadCacheEntry]  # 文件读取缓存
    activated_groups: list[str]         # 激活的工具组名
```

## Task 和 TaskContext

```python
from agentscope.state import Task

task = Task(
    subject="任务标题",
    description="任务描述",
    metadata={},
    state="pending",           # pending/in_progress/completed
    owner=None,                # 所有者
    blocks=[],                 # 被此任务阻塞的任务 ID
    blocked_by=[],             # 阻塞此任务的任务 ID
)

class TaskContext(BaseModel):
    tasks: list[Task]          # 任务列表
```

## 在 Agent 中使用

```python
# 创建新 Agent 时传入空状态
agent = Agent(name="...", state=AgentState())

# 或恢复已有状态
saved_state = AgentState.model_validate_json(saved_json)
agent = Agent(name="...", state=saved_state)

# 获取当前状态
current_state = agent.state

# 持久化
import json
json.dump(current_state.model_dump(), open("state.json", "w"))
```

## 上下文压缩配置

通过 `ContextConfig` 控制自动压缩：

```python
from agentscope.agent import ContextConfig

config = ContextConfig(
    trigger_ratio=0.8,      # token 使用超过 80% 时触发压缩
    reserve_ratio=0.1,      # 压缩后保留 10% 的空间
    tool_result_limit=3000, # 工具结果最大 token 数
    compression_prompt="...",  # 压缩提示词
    summary_template="...",    # 摘要模板
    summary_schema={...},      # 摘要 JSON Schema
)
```
