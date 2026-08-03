# agentscope-v2-skill

**English** | [中文](README.md)

> Claude Code Skill: Development guide for the AgentScope 2.0 multi-agent framework

![version](https://img.shields.io/badge/AgentScope-v2.0.5-blue)
![license](https://img.shields.io/badge/license-MIT-green)

## Overview

This is a [Claude Code](https://claude.ai/code) Skill plugin that provides comprehensive development reference for [AgentScope 2.0](https://github.com/agentscope-ai/agentscope) (`agentscope-ai`). Once installed, Claude Code automatically activates this skill when handling AgentScope-related tasks, ensuring accurate v2 API knowledge. Docs are continuously synced with upstream source, currently aligned with **v2.0.5**.

⚠️ **Note**: AgentScope 2.0 has a completely different, incompatible API from the legacy `modelscope/agentscope` (v1.x).

## Coverage

- **Agent Creation**: Single `Agent` class, `reply()` returns an async event stream
- **Credential System**: OpenAI / Anthropic / DashScope / Gemini / Ollama authentication
- **Model Configuration**: ChatModelBase, retries, fallback, omni model audio output, `extra_body` passthrough (v2.0.3+)
- **Embedding / TTS** (v2.0.2+, experimental): dedicated `embedding` (refactored to generic base in v2.0.3, dimensions required + multimodal routing) / `tts` modules; Credential exposes multimodal capabilities; TTS includes `OpenAITTSModel` (v2.0.4+) / DashScope `CosyVoice` (one class for both normal and realtime) / `DashScopeRealtimeTTSModel` / `GeminiTTSModel` (v2.0.5+)
- **RAG Knowledge Base** (v2.0.3+): `agentscope.rag` module (`KnowledgeBase` handle + `QdrantStore` / `MilvusLiteStore` / `MongoDBStore` vector stores (latter two v2.0.4+) / `ElasticsearchStore` (v2.0.5+) + Parser/Chunker indexing pipeline, parsers include Text/PDF/PPT/Image + `WordParser`/`ExcelParser` (v2.0.4+)); `RAGMiddleware` offers both static (auto-inject) and agentic (tool-driven) retrieval modes
- **Toolkit / ToolBase**: Tool registration, built-in tools (Bash, Read, Write, etc.; `PowerShell` on Windows, v2.0.5+), `ToolMiddlewareBase` tool-level onion middleware (v2.0.3+)
- **MCPClient**: StdIO / stateful HTTP / stateless HTTP connections
- **AgentState**: Context, summary, session, permission, task management
- **Event System**: `REPLY_START` → `TEXT_BLOCK_DELTA` → `TOOL_CALL_*` → `REPLY_END`, incl. `DATA_BLOCK_*` audio streams, agent interruption (`UserInterruptEvent`, v2.0.4+), and reply error reporting (`ErrorType` classification + `ReplyFinishedReason.ERROR`, v2.0.5+)
- **Agent Advanced Capabilities**: structured output (`structured_schema` Pydantic model, v2.0.5+), runtime state injection (time/task/context-compression awareness via `InjectionConfig`, v2.0.5+)
- **Permission / ToolGroup / Skill**: Permission control (five `PermissionMode`s; since v2.0.4.post1 a unified read-only fast path, and `DONT_ASK` upgraded to the unattended counterpart of `ACCEPT_EDITS`; since v2.0.5+ concurrent batch confirmation deduplication and injection-risk commands no longer treated as read-only), tool groups, skill system
- **Middleware**: Intercept reply/reasoning/check_permission/acting/model_call/compress_context (`on_check_permission` permission hook added in v2.0.5+), incl. built-in `TTSMiddleware` / `ReplyBudgetControlMiddleware` / `TracingMiddleware` (OpenTelemetry tracing) / `Mem0Middleware` (mem0 cross-session long-term memory, v2.0.3+) / `ReMeMiddleware` (embedded ReMe long-term memory, v2.0.4+) / `AgenticMemoryMiddleware` (filesystem long-term memory, v2.0.4+) / `RAGMiddleware` (retrieval augmentation)
- **Workspace**: `LocalWorkspace` / `DockerWorkspace` / `E2BWorkspace` / `K8sWorkspace` / `OpenSandboxWorkspace` / `DaytonaWorkspace` (latter three v2.0.4+) / `AppleContainerWorkspace` (macOS 26+ Apple Container, v2.0.5+) / `BubblewrapWorkspace` (Linux bubblewrap sandbox, v2.0.5+), all support built-in tools (Bash/Read/Write/Edit/Grep/Glob) running in containers/cloud sandboxes/K8s Pods; Backend abstraction (`LocalBackend` / `DockerBackend` / `E2BBackend` / `K8sBackend` / `OpenSandboxBackend` / `DaytonaBackend` / `AppleContainerBackend` / `BubblewrapBackend`); `skill_paths` supports `~` expansion (v2.0.5+)
- **App**: `create_app` (REST + SSE), `SubAgentTemplate` sub-agent templates (incl. team leader HITL event projection + `AgentInvite` to borrow existing agents, v2.0.4+), `MessageBus` (`RedisMessageBus` / `InMemoryMessageBus` single-node lightweight option, v2.0.3+), Session Status unified status endpoint (v2.0.4+), `AsyncSQLAlchemyStorage` multi-DB persistence (SQLite/PostgreSQL/MySQL, v2.0.5+), Hub registry (`GitHubMCPHub` / `ClawSkillHub`, browse-install-pull MCPs/skills from hub, v2.0.5+), cross-user resource sharing (`ResourceAccessPolicy` abstraction, group/org scenarios, v2.0.4+), served knowledge base (`KnowledgeBaseManager` + `LocalBlobStore`/`S3BlobStore` + embedded or standalone index worker)
- **Global Config**: `set_id_factory()` for custom ID generation (v2.0.3+)

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
│   ├── models.md             # Model, Credential, Embedding/TTS multimodal
│   ├── permissions.md        # Permissions and tool groups
│   ├── state.md              # AgentState management
│   ├── tools.md              # Toolkit, ToolBase, MCPClient
│   ├── rag.md                # RAG knowledge base (KnowledgeBase / RAGMiddleware)
│   └── middleware-workspace.md # Middleware, workspace, app service, sub-agent templates
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
