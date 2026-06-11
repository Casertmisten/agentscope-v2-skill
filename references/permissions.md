# 权限与工具组 — v2

## Permission 体系

v2 引入了完整的权限系统，每个工具调用都会经过权限检查。

### 核心类

```python
from agentscope.permission import (
    PermissionContext,     # 权限上下文（存在 AgentState 中）
    PermissionDecision,   # 权限决策
    PermissionRule,       # 权限规则
    PermissionBehavior,   # 行为枚举
)
```

### PermissionDecision

工具调用前的权限检查结果：

- **ALLOW** — 允许执行
- **DENY** — 拒绝执行
- **ASK** — 需要用户确认（触发 `RequireUserConfirmEvent`）

### PermissionRule

```python
PermissionRule(
    tool_name="bash",         # 工具名
    rule_content="git *",     # 匹配模式（None 表示匹配所有调用）
    behavior=PermissionBehavior.ALLOW,  # allow/deny
    source="suggested",       # 来源：suggested/user/system
)
```

### 权限流程

```
工具调用 → ToolBase.check_permissions()
    → ALLOW → 直接执行
    → DENY → 返回 ToolResultBlock(state="denied")
    → ASK  → 触发 RequireUserConfirmEvent
              → 用户确认 → UserConfirmResultEvent → 执行
              → 用户拒绝 → ToolResultBlock(state="denied")
```

## ToolBase 权限方法

```python
class MyTool(ToolBase):
    async def check_permissions(self, tool_input, context) -> PermissionDecision:
        # 自定义权限逻辑
        if is_dangerous(tool_input):
            return PermissionDecision.ASK  # 需要用户确认
        return PermissionDecision.ALLOW

    def match_rule(self, rule_content, tool_input) -> bool:
        # 匹配权限规则，用于自动决策
        ...

    def generate_suggestions(self, tool_input) -> list[PermissionRule]:
        # 生成建议的权限规则（显示给用户）
        ...
```

## 内置工具的权限

内置工具（Bash、Read、Write、Edit、Glob、Grep）都有完善的权限检查：

- **Bash** — 命令模式匹配
- **Read/Write/Edit** — 文件路径模式匹配，敏感文件保护
- **Glob/Grep** — 搜索路径匹配

### 敏感文件保护

```python
# ToolBase 内置危险路径检查
dangerous_files = [".bashrc", ".gitconfig", ...]
dangerous_directories = [".git", ".ssh", ...]

def _is_dangerous_path(self, file_path: str) -> bool:
    # 检查是否是敏感文件/目录
    ...
```

## ToolGroup 管理

工具组允许将相关工具打包，智能体通过元工具（Meta Tool）按需激活：

### 创建工具组

```python
from agentscope.tool import ToolGroup

toolkit = Toolkit(
    tools=[Bash()],  # basic 组，始终激活
    tool_groups=[
        ToolGroup(
            name="browser",
            description="网页浏览工具",
            instructions="1. 先 navigate\n2. 需认证时询问用户",
            tools=[Navigate(), Click(), Type()],
        ),
        ToolGroup(
            name="database",
            description="数据库操作工具",
            tools=[QueryTool()],
            mcps=[db_mcp],
        ),
    ],
)
```

### 激活/停用

智能体通过内置的 `ResetTools` 元工具自动管理：

```python
# Agent 内部自动调用元工具
# 开发者无需手动操作
# ResetTools 接受各工具组的 bool 参数
```

### 查询 Schema

```python
# 只返回 basic 组 + 指定激活组的工具
schemas = await toolkit.get_tool_schemas()

# 指定组
schemas = await toolkit.get_tool_schemas(groups=["browser"])
```

## Skill 系统

v2 新增了 Skill 系统，允许为智能体加载技能（一组指令+脚本+资源）：

```python
from agentscope.tool import Toolkit

# 通过目录加载
toolkit = Toolkit(
    skills_or_loaders=["/path/to/skills/dir"],
)

# 获取 skill 提示（可拼接到 sys_prompt）
instructions = await toolkit.get_skill_instructions()
```

Skill 不是工具——智能体需要先通过 `SkillViewer` 工具阅读 skill 的完整说明，再按照说明使用工具。
