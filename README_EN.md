# agentscope-v2-skill

**English** | [中文](README.md)

> Claude Code Skill: Development guide for the AgentScope 2.0 multi-agent framework

## Overview

This is a [Claude Code](https://claude.ai/code) Skill plugin that provides comprehensive development reference for [AgentScope 2.0](https://github.com/agentscope-ai/agentscope) (`agentscope-ai`). Once installed, Claude Code automatically activates this skill when handling AgentScope-related tasks, ensuring accurate v2 API knowledge.

⚠️ **Note**: AgentScope 2.0 has a completely different, incompatible API from the legacy `modelscope/agentscope` (v1.x).

## Coverage

- **Agent Creation**: Single `Agent` class, `reply()` returns an async event stream
- **Credential System**: OpenAI / Anthropic / DashScope / Gemini / Ollama authentication
- **Model Configuration**: ChatModelBase, retries, fallback, omni model audio output
- **Embedding / TTS** (v2.0.2+): dedicated `embedding` / `tts` modules; Credential exposes multimodal capabilities
- **Toolkit / ToolBase**: Tool registration, built-in tools (Bash, Read, Write, etc.)
- **MCPClient**: StdIO / stateful HTTP / stateless HTTP connections
- **AgentState**: Context, summary, session, permission, task management
- **Event System**: `REPLY_START` → `TEXT_BLOCK_DELTA` → `TOOL_CALL_*` → `REPLY_END`, incl. `DATA_BLOCK_*` audio streams
- **Permission / ToolGroup / Skill**: Permission control, tool groups, skill system
- **Middleware**: Intercept reply/reasoning/acting/model_call, incl. built-in `TTSMiddleware`
- **App**: `create_app` (REST + SSE), `SubAgentTemplate` sub-agent templates

## Installation

Clone this Skill into Claude Code's skill directory:

```bash
# Option 1: Clone to the global skill directory
git clone https://github.com/Casertmisten/agentscope-v2-skill.git \
  ~/.claude/skills/agentscope-v2-skill

# Option 2: Clone to a project-level skill directory
git clone https://github.com/Casertmisten/agentscope-v2-skill.git \
  your-project/.claude/skills/agentscope-v2-skill
```

Once installed, Claude Code will automatically activate this skill when you mention AgentScope, multi-agent, or related keywords.

## File Structure

```
agentscope-v2-skill/
├── SKILL.md                  # Skill definition and main docs (auto-loaded by Claude Code)
├── references/               # Detailed reference docs
│   ├── agent-events.md       # Agent config and event system
│   ├── messages.md           # Message types and content blocks
│   ├── models.md             # Model and Credential configuration
│   ├── permissions.md        # Permissions and tool groups
│   ├── state.md              # AgentState management
│   └── tools.md              # Toolkit, ToolBase, MCPClient
└── .claude/
    └── settings.local.json   # Project-level Claude Code settings
```

## Quick Reference

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
        system_prompt="You are a helpful assistant.",   # note: system_prompt
        model=model,                                    # a ChatModelBase instance
        toolkit=Toolkit(),
    )

    async for event in agent.reply_stream(UserMsg("user", "Hello!")):
        print(event.type, event)

asyncio.run(main())
```

## Contributing

Issues and PRs are welcome to supplement or correct the documentation. When AgentScope 2.0 API updates, sync the reference docs under `references/`.

## License

MIT
