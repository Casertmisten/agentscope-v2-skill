# agentscope-v2-skill

> Claude Code Skill：AgentScope 2.0 多智能体框架开发指南

## 简介

这是一个 [Claude Code](https://claude.ai/code) 的 Skill 插件，为 [AgentScope 2.0](https://github.com/agentscope-ai/agentscope)（`agentscope-ai`）提供全面的开发参考。安装后，Claude Code 在处理 AgentScope 相关任务时会自动加载此技能，获得 v2 API 的准确知识。

⚠️ **注意**：AgentScope 2.0 与旧版 `modelscope/agentscope`（v1.x）的 API 完全不同，不兼容。

## 涵盖内容

- **Agent 创建**：单一 `Agent` 类，`reply()` 返回异步事件流
- **Credential 体系**：OpenAI / Anthropic / DashScope / Gemini / Ollama 认证
- **Model 配置**：ChatModelBase、重试、fallback
- **Toolkit / ToolBase**：工具注册、内置工具（Bash, Read, Write 等）
- **MCPClient**：StdIO / 有状态 HTTP / 无状态 HTTP 连接
- **AgentState**：上下文、摘要、会话、权限、任务管理
- **事件系统**：`REPLY_START` → `TEXT_BLOCK_DELTA` → `TOOL_CALL_*` → `REPLY_END`
- **Permission / ToolGroup / Skill**：权限控制、工具组、技能系统

## 安装

将此 Skill 克隆到 Claude Code 的技能目录：

```bash
# 方式一：克隆到全局技能目录
git clone https://github.com/Casertmisten/agentscope-v2-skill.git \
  ~/.claude/skills/agentscope-v2-skill

# 方式二：克隆到项目级技能目录
git clone https://github.com/Casertmisten/agentscope-v2-skill.git \
  your-project/.claude/skills/agentscope-v2-skill
```

安装后，当你提到 AgentScope、多智能体、写 agent 等关键词时，Claude Code 会自动激活此技能。

## 文件结构

```
agentscope-v2-skill/
├── SKILL.md                  # 技能定义与主文档（Claude Code 自动加载）
├── references/               # 详细参考文档
│   ├── agent-events.md       # Agent 配置和事件系统
│   ├── messages.md           # 消息类型与内容块
│   ├── models.md             # 模型与 Credential 配置
│   ├── permissions.md        # 权限与工具组
│   ├── state.md              # AgentState 状态管理
│   └── tools.md              # Toolkit、ToolBase、MCPClient
└── .claude/
    └── settings.local.json   # 项目级 Claude Code 设置
```

## 快速参考

```python
import asyncio
from agentscope.agent import Agent
from agentscope.credential import OpenAICredential
from agentscope.message import UserMsg
from agentscope.tool import Toolkit

async def main():
    agent = Agent(
        name="Assistant",
        sys_prompt="你是一个有用的助手。",
        credential=OpenAICredential(api_key="sk-xxx"),
        model="gpt-4o",
        toolkit=Toolkit(),
    )

    async for event in agent.reply(UserMsg("user", "你好！")):
        print(event.type, event)

asyncio.run(main())
```

## 维护

欢迎提 Issue 和 PR 来补充或修正文档。当 AgentScope 2.0 API 更新时，同步更新 `references/` 下的参考文档。

## 许可

MIT
