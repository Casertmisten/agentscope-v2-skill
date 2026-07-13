# 权限与工具组 — v2

## Permission 体系

v2 引入了完整的权限系统，每个工具调用都会经过权限检查。

### 核心类

```python
from agentscope.permission import (
    PermissionContext,              # 权限上下文（存在 AgentState 中）
    PermissionDecision,             # 权限决策
    PermissionRule,                 # 权限规则
    PermissionBehavior,             # 行为枚举
    PermissionMode,                 # 权限模式
    PermissionEngine,               # 权限引擎
    AdditionalWorkingDirectory,     # 额外工作目录
)
```

## PermissionMode — 权限模式

控制权限系统如何处理工具执行请求：

| 模式 | 行为 | 适用场景 |
|---|---|---|
| `DEFAULT` | 每个操作都需确认，除非有 allow 规则匹配 | 默认模式，最安全 |
| `ACCEPT_EDITS` | 自动允许工作目录内的文件读写和文件系统命令 | 快速迭代开发 |
| `EXPLORE` | 只读模式：允许只读工具和命令，拒绝修改操作 | 探索代码库 |
| `BYPASS` | 跳过所有安全检查，仅用户 deny/ask 规则生效 | 沙箱环境 |
| `DONT_ASK` | 将所有 ASK（包括安全 ASK）转为 DENY | 无人值守执行 |

```python
from agentscope.permission import PermissionMode, PermissionContext

context = PermissionContext(
    mode=PermissionMode.ACCEPT_EDITS,
    working_directories={"/project": AdditionalWorkingDirectory(
        path="/project",
        source="session",
    )},
)
```

## PermissionBehavior — 行为枚举

```python
class PermissionBehavior(Enum):
    ALLOW = "allow"            # 允许执行
    DENY = "deny"              # 拒绝执行
    ASK = "ask"                # 需要用户确认
    PASSTHROUGH = "passthrough" # 工具委托决策给引擎
```

## PermissionDecision

`PermissionDecision` 是 **dataclass**（不是枚举），`check_permissions` 必须返回它的实例。
`behavior` 字段取上面的 `PermissionBehavior` 枚举，`message` 为必填：

```python
from agentscope.permission import PermissionDecision, PermissionBehavior

PermissionDecision(
    behavior=PermissionBehavior.ASK,   # ALLOW / DENY / ASK / PASSTHROUGH
    message="该操作会修改文件，需确认",   # 必填：人类可读的决策说明
    decision_reason=None,              # 可选：决策理由
    updated_input=None,                # 可选：修改后的入参（如净化后的路径）
    suggested_rules=None,              # 可选：建议用户应用的权限规则
    bypass_immune=False,               # 可选：ASK 时是否禁止被 allow 规则覆盖（危险操作）
)
```

- **ALLOW** — 允许执行
- **DENY** — 拒绝执行
- **ASK** — 需要用户确认（触发 `RequireUserConfirmEvent`）
- **PASSTHROUGH** — 工具委托，由引擎继续规则匹配

## PermissionRule

```python
PermissionRule(
    tool_name="bash",         # 工具名
    rule_content="git *",     # 匹配模式（None 表示匹配所有调用）
    behavior=PermissionBehavior.ALLOW,  # allow/deny/ask
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
    → PASSTHROUGH → 由 PermissionEngine 继续规则匹配
```

## ToolBase 权限方法

```python
from agentscope.permission import PermissionDecision, PermissionBehavior, PermissionRule

class MyTool(ToolBase):
    async def check_permissions(self, tool_input, context) -> PermissionDecision:
        # 自定义权限逻辑；PermissionDecision 是 dataclass，不是枚举
        if is_dangerous(tool_input):
            return PermissionDecision(
                behavior=PermissionBehavior.ASK,
                message="该操作有风险，需确认",
            )
        return PermissionDecision(
            behavior=PermissionBehavior.ALLOW,
            message="安全操作",
        )

    async def match_rule(self, rule_content, tool_input) -> bool:
        # 匹配权限规则，用于自动决策（async）
        ...

    async def generate_suggestions(self, tool_input) -> list[PermissionRule]:
        # 生成建议的权限规则（显示给用户）（async）
        ...
```

## 内置工具的权限

内置工具（Bash、Read、Write、Edit、Glob、Grep）都有完善的权限检查：

- **Bash** — 命令模式匹配，自动允许只读命令（ls/git status）
- **Read/Write/Edit** — 文件路径模式匹配，敏感文件保护，ACCEPT_EDITS 模式下自动允许工作目录操作
- **Glob/Grep** — 搜索路径匹配，EXPLORE 模式下自动允许

### 敏感文件保护

```python
# ToolBase 内置危险路径检查
dangerous_files = [".bashrc", ".gitconfig", ...]
dangerous_directories = [".git", ".ssh", ...]
```

## ToolGroup 管理

工具组允许将相关工具打包，智能体通过元工具（ResetTools）按需激活：

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
        ToolGroup(
            name="research",
            description="调研工具",
            tools=[],
            skills_or_loaders=["/path/to/skills"],  # 也可直接绑技能（见下文 Skill 系统）
        ),
    ],
)
```

> `ToolGroup` 还接受 `skills_or_loaders`（字符串路径 / `Skill` / `SkillLoaderBase`），
> 把技能直接绑定到工具组，组激活时一并加载。

### 激活/停用

智能体通过内置的 `ResetTools` 元工具自动管理：

```python
from agentscope.tool import ResetTools

# Agent 内部自动调用 ResetTools
# 开发者通常无需手动操作
```

### 查询 Schema

```python
# 只返回 basic 组 + 指定激活组的工具
schemas = await toolkit.get_tool_schemas()

# 指定组
schemas = await toolkit.get_tool_schemas(groups=["browser"])
```

## Skill 系统

v2 新增了 Skill 系统，允许为智能体加载技能（一组指令+脚本+资源）。

### Skill 目录格式

每个 skill 是一个目录，根目录下必须有 `SKILL.md`，文件头部是 YAML frontmatter（必须含 `name` 和 `description`），之后是 Markdown 正文（智能体阅读的说明）：

```markdown
---
name: weather-agent
description: 查询天气并给出穿衣建议的技能
---

# 天气技能

当用户问天气时，先调用 get_weather 获取数据，再结合温度给出建议……
```

### skill 模块

```python
from agentscope.skill import Skill, SkillLoaderBase, LocalSkillLoader

# Skill 数据类（由 loader 自动构建）
# Skill(name, description, dir, markdown, updated_at)

# 从本地目录加载（directory 下的 SKILL.md）
loader = LocalSkillLoader(directory="/path/to/skills", scan_subdir=False)
skills = await loader.list_skills()   # -> list[Skill]

# 自定义 loader：继承 SkillLoaderBase，实现 async list_skills() -> list[Skill]
```

### 在 Toolkit 中注册

`skills_or_loaders` 接受三种形式——目录路径字符串、`Skill` 对象、或 `SkillLoaderBase` 实例：

```python
from agentscope.tool import Toolkit
from agentscope.skill import LocalSkillLoader

toolkit = Toolkit(
    # 三种形式可混用
    skills_or_loaders=[
        "/path/to/skills/dir",                      # 字符串：当作 LocalSkillLoader(directory=...)
        LocalSkillLoader(directory="/other/skills"),# 显式 loader（可设 scan_subdir=True）
    ],
)

# 获取 skill 提示（可拼接到 system_prompt）
instructions = await toolkit.get_skill_instructions()
# 也可限定激活的组：注意参数名是 activated_groups（与 get_tool_schemas 的 groups 不同）
instructions = await toolkit.get_skill_instructions(activated_groups=["basic"])
```

> Skill 不是工具——智能体需要先阅读 skill 的完整说明，再按照说明使用工具。Workspace 也可通过
> `add_skill()`/`remove_skill()` 动态管理 skill（见 middleware-workspace 文档）。
