# agentscope-v2-skill

[English](README_EN.md) | **中文**

> Claude Code Skill：AgentScope 2.0 多智能体框架开发指南

![version](https://img.shields.io/badge/AgentScope-v2.0.6-blue)
![license](https://img.shields.io/badge/license-MIT-green)

## 简介

这是一个 [Claude Code](https://claude.ai/code) 的 Skill 插件，为 [AgentScope 2.0](https://github.com/agentscope-ai/agentscope)（`agentscope-ai`）提供全面的开发参考。安装后，Claude Code 在处理 AgentScope 相关任务时会自动加载此技能，获得 v2 API 的准确知识。文档持续跟随上游源码同步，当前对应 **v2.0.6**。

⚠️ **注意**：AgentScope 2.0 与旧版 `modelscope/agentscope`（v1.x）的 API 完全不同，不兼容。

## 涵盖内容

- **Agent 创建**：单一 `Agent` 类，`reply()` 返回异步事件流
- **Credential 体系**：OpenAI / Anthropic / DashScope / Gemini / Ollama 认证
- **Model 配置**：ChatModelBase、重试、fallback、Omni 模型音频输出、`extra_body` 透传（v2.0.3+）、Anthropic 推理控制（`thinking_mode` / `thinking_display` / `reasoning_effort`，v2.0.6+）
- **Embedding / TTS**（v2.0.2+，实验性）：独立的 `embedding`（v2.0.3 重构为泛型基类，dimensions 必填 + 多模态路由） / `tts` 模块，Credential 统一暴露多模态能力；TTS 含 `OpenAITTSModel`（v2.0.4+）/ DashScope `CosyVoice`（同一类支持普通/实时）/ `DashScopeRealtimeTTSModel` / `GeminiTTSModel`（v2.0.5+）
- **RAG 知识库**（v2.0.3+）：`agentscope.rag` 模块（`KnowledgeBase` 句柄 + `QdrantStore` / `MilvusLiteStore` / `MongoDBStore`（均为 v2.0.4+）/ `ElasticsearchStore`（v2.0.5+）向量库 + Parser/Chunker 索引管线，Parser 含 Text/PDF/PPT/Image + `WordParser`/`ExcelParser`（v2.0.4+）），`RAGMiddleware` 提供 static（自动注入）与 agentic（工具驱动）两种检索模式
- **Toolkit / ToolBase**：工具注册、内置工具（Bash, Read, Write 等；Windows 下 `PowerShell`，v2.0.5+）、`ToolMiddlewareBase` 工具级洋葱中间件（v2.0.3+）
- **MCPClient**：StdIO / 有状态 HTTP / 无状态 HTTP 连接
- **AgentState**：上下文、摘要、会话、权限、任务管理
- **事件系统**：`REPLY_START` → `TEXT_BLOCK_DELTA` → `TOOL_CALL_*` → `REPLY_END`，含 `DATA_BLOCK_*` 音频流、Agent 中断（`UserInterruptEvent`，v2.0.4+）、回复错误上报（`ErrorType` 分类 + `ReplyFinishedReason.ERROR`，v2.0.5+）
- **Agent 高级能力**：结构化输出（`structured_schema` Pydantic 模型，v2.0.5+）、运行时状态注入（时间/任务/上下文压缩感知，`InjectionConfig`，v2.0.5+）
- **Permission / ToolGroup / Skill**：权限控制（五种 `PermissionMode`，v2.0.4.post1 起统一只读快路径，`DONT_ASK` 升级为 `ACCEPT_EDITS` 的无人值守版；v2.0.5+ 起并发批量确认豁免、含注入风险的命令不再当只读）、工具组、技能系统
- **Middleware**：拦截 reply/reasoning/check_permission/acting/model_call/compress_context（`on_check_permission` 权限检查 hook 为 v2.0.5+），含内置 `TTSMiddleware` / `ReplyBudgetControlMiddleware` / `TracingMiddleware`（OpenTelemetry 追踪）/ `Mem0Middleware`（mem0 跨会话长期记忆，v2.0.3+）/ `ReMeMiddleware`（内嵌 ReMe 长期记忆，v2.0.4+）/ `AgenticMemoryMiddleware`（文件系统长期记忆，v2.0.4+）/ `RAGMiddleware`（检索增强）
- **Workspace**：`LocalWorkspace` / `DockerWorkspace` / `E2BWorkspace` / `K8sWorkspace` / `OpenSandboxWorkspace` / `DaytonaWorkspace`（后三者 v2.0.4+）/ `AppleContainerWorkspace`（macOS 26+ Apple Container，v2.0.5+）/ `BubblewrapWorkspace`（Linux bubblewrap 沙箱，v2.0.5+），均支持内置工具（Bash/Read/Write/Edit/Grep/Glob）在容器/云沙箱/K8s Pod 执行，Backend 抽象（`LocalBackend` / `DockerBackend` / `E2BBackend` / `K8sBackend` / `OpenSandboxBackend` / `DaytonaBackend` / `AppleContainerBackend` / `BubblewrapBackend`）；`skill_paths` 路径支持 `~` 展开（v2.0.5+）；MCP 按 agent+session 隔离（共享 workspace 下延迟实例化 + 私有实例、`.mcp` v2 schema、`purge_session` 会话清理、`max_live_stateful_mcps` LRU 容量管理，v2.0.6+）
- **服务化**：`create_app`（REST + SSE）、`SubAgentTemplate` 子智能体模板（含团队 Leader HITL 事件投影 + `AgentInvite` 邀请已有 agent 入队，v2.0.4+）、`MessageBus` 消息总线（`RedisMessageBus` / `InMemoryMessageBus` 单节点轻量选项，v2.0.3+）、Session Status 统一状态查询端点（v2.0.4+；v2.0.6+ 起 `GET /sessions` 内联 `status` 字段并裁剪 `context/summary/tool_context`，`is_running` deprecated）、`AsyncSQLAlchemyStorage` 多数据库持久化存储（SQLite/PostgreSQL/MySQL，v2.0.5+）、`/health` 健康检查端点（按组件就绪探测，v2.0.6+）、workspace 产物文件读取与状态端点（`WorkspaceService` 服务层统一收敛解析/下载令牌/skill 上传/git 汇总；`GET /workspace/directories` 返回 `DirectoryListing` 含解析后绝对路径、`GET /workspace/status` 返回 `WorkspaceStatus`（workdir + cwd + `GitStatus`）、`/workspace/files` + download-token，v2.0.6+）、Session 工作目录锚点（`SessionConfig.cwd` + `PATCH /sessions/{id}` 在 run 持有时返回 409，v2.0.6+）、`extra_agent_middlewares` 工厂新增 `workspace` 第四参数（旧三参数继续兼容，v2.0.6+）、Hub 注册中心（`GitHubMCPHub` / `ClawSkillHub`，从 hub 浏览-安装-拉入 MCP/Skill，v2.0.5+）、Channel IM 频道（接入飞书 / Discord，`ChannelBase` 适配器 + `ChannelGateway` 入站路由 + `ChannelLifecycleDispatcher` 出站转发 + 确定性派生会话 + 交互卡片权限审批，v2.0.6+）、跨用户资源共享（`ResourceAccessPolicy` 抽象，group/org 场景，v2.0.4+）、服务化知识库（`KnowledgeBaseManager` + `LocalBlobStore`/`S3BlobStore` + 内嵌或独立索引 worker）
- **Agent 执行**：`cur_iter` 轮次计数修复（一轮仅在全部 tool call 拿到结果后才计数，新增 `AgentState.get_unfinished_tool_calls`，v2.0.6+）
- **全局配置**：`set_id_factory()` 自定义 ID 生成策略（v2.0.3+）

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
│   ├── models.md             # 模型、Credential、Embedding/TTS 多模态
│   ├── permissions.md        # 权限与工具组
│   ├── state.md              # AgentState 状态管理
│   ├── tools.md              # Toolkit、ToolBase、MCPClient
│   ├── rag.md                # RAG 知识库（KnowledgeBase / RAGMiddleware）
│   └── middleware-workspace.md # 中间件、工作区、服务化、子智能体模板
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
    credential = OpenAICredential(api_key="sk-xxx")
    model = credential.get_chat_model_class()(
        credential=credential,
        model="gpt-4o",
    )
    agent = Agent(
        name="Assistant",
        system_prompt="你是一个有用的助手。",   # 注意：是 system_prompt
        model=model,                          # ChatModelBase 实例
        toolkit=Toolkit(),
    )

    async for event in agent.reply_stream(UserMsg("user", "你好！")):
        print(event.type, event)

asyncio.run(main())
```

## 维护

欢迎提 Issue 和 PR 来补充或修正文档。当 AgentScope 2.0 API 更新时，同步更新 `references/` 下的参考文档。

## 许可

MIT
