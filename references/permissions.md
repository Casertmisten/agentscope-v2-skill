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
| `DEFAULT` | 默认需确认，但**只读调用自动放行**（v2.0.4.post1+，`tool.check_read_only()` 返回 True 的工具/命令，如 `ls`/`git status`/Read/Glob/Grep）；其余按 deny→ask→allow 规则匹配，未命中则 ASK | 默认模式，最安全 |
| `ACCEPT_EDITS` | 自动放行工作目录内的文件读写、文件系统命令（mkdir/rm/mv/cp 等，要求所有目标路径都在工作目录内），其余按规则 | 用户在场，快速迭代开发 |
| `EXPLORE` | 只读模式：允许只读工具和只读 bash 命令，拒绝一切修改；**用户配置的 DENY/ASK 规则优先于只读自动放行** | 探索代码库 |
| `BYPASS` | 跳过所有检查（**包括安全 ASK**，如 `rm -rf /`、写 `~/.bashrc`），仅用户 deny/ask 规则和工具自身 DENY 生效 | 完全可信的沙箱环境 |
| `DONT_ASK` | **ACCEPT_EDITS 的无人值守对应版**（v2.0.4.post1+ 行为升级）：只读调用 + 工作目录内编辑自动放行，其余本应弹窗的 ASK（含安全 ASK）一律转 DENY；**保证永不返回 ASK** | 定时任务、后台执行等用户不在场的场景 |

> ℹ️ **v2.0.4.post1 权限行为变化**：
> - 五种模式统一走同一个 `_check_read_only_fast_path`，**DEFAULT 和 DONT_ASK 现在也自动放行只读调用**（之前只有 ACCEPT_EDITS/EXPLORE 有只读快路径）。
> - **DONT_ASK 语义升级**：从「全转 DENY」变成「ACCEPT_EDITS 的无人值守版」——只读 + 工作目录内编辑会自动放行，仅其余需要人工确认的操作转 DENY。`Bash` 工具的工作目录内文件系统命令自动放行逻辑也已同时覆盖 ACCEPT_EDITS 和 DONT_ASK。
> - 对比：BYPASS 适合「完全信任、要最高自由度」的沙箱；DONT_ASK 适合「无人值守但仍要安全护栏」的定时任务。

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

### 并发批量确认豁免（v2.0.5+）

当一轮 reply 中**并发**发起多个工具调用时，相似调用不再重复弹框：如果某个非安全 ASK 的调用
已被本批次早前某条 `suggested_rules` 中的 allow rule 覆盖（通过 `tool.match_rule` 判定），
则该调用保持 `PENDING` 不再弹窗，等用户回答第一条后规则入库、下一轮 reply 重新评估。

- **安全 ASK（`bypass_immune=True`）永不去重**——allow rule 无法清除它，必须各自单独弹框
  （如 `rm -rf /`、写 `~/.bashrc`、含注入风险的命令）。
- 批次共享一个 `kept_rules` 累加器（agent 内部 `list[PermissionRule]`），不暴露在
  `PermissionContext`/`PermissionDecision` 上；复用的是既有 `suggested_rules` 与 `bypass_immune` 字段。

### on_check_permission 中间件 hook（v2.0.5+）

权限检查现在也可被 agent 级中间件拦截（洋葱模型），位于 `on_reasoning` 与 `on_acting` 之间。
用于审计、自定义策略、按租户/角色覆写决策等。详见
[middleware-workspace.md](middleware-workspace.md) 的「on_check_permission」小节。

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

- **Bash** — 命令模式匹配，只读命令（`ls`/`git status` 等）在各模式自动放行；工作目录内文件系统命令（`mkdir`/`rm`/`mv`/`cp` 等，要求所有目标路径都在工作目录内）在 `ACCEPT_EDITS` 和 `DONT_ASK` 模式自动放行；v2.0.5+ 起含注入风险（命令替换/控制流，如 `ls $(rm -rf /)`）的命令**不再判为只读**，会走 bypass-immune 的安全 ASK；`find . -delete`/`-exec`/`-ok` 等会改文件的 find 谓词也不再当只读放行
- **Read/Write/Edit** — 文件路径模式匹配，敏感文件保护，`ACCEPT_EDITS`/`DONT_ASK` 模式下自动允许工作目录操作
- **Glob/Grep** — 搜索路径匹配，只读在各模式自动放行，`EXPLORE` 模式下更是全部放行

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
