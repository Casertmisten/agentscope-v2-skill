# RAG 检索增强 (v2.0.3+)

> ⚠️ `agentscope.rag` 是 v2.0.3 发布后新增的模块,API 可能随版本调整,使用前请以源码为准。

RAG 让 Agent 能基于外部知识库回答问题。模块组成:

- **`KnowledgeBase`** —— 运行时句柄,绑定 (embedding 模型 + 向量库 + collection),暴露 search/insert/delete/list 四个操作
- **`QdrantStore`** —— 内置向量库后端(基于 Qdrant),`VectorStoreBase` 可子类化扩展其它后端
- **Parser / Chunker** —— 文档解析与切分管线(`bytes → Section[] → Chunk[]`)
- **`RAGMiddleware`** —— 把知识库接入 Agent,两种模式:static(自动注入)/ agentic(工具驱动)

## 安装

```bash
# RAG 依赖(含 qdrant-client)
uv pip install "agentscope[rag]"

# 若 RAGMiddleware 与 Mem0Middleware 混用,另装
uv pip install "agentscope[mem0]"
```

## 索引管线

数据流:

```
文件 bytes ── Parser.parse ──► Section[] ── Chunker.chunk ──► Chunk[]
                                                                │
                                                                ▼
                                            kb.insert_document(chunks)
                                                                │
                                                                ▼
                                                  embed → 存入向量库 collection
```

检索镜像同一条链路:把 query 交给 `kb.search(...)`,返回 `list[VectorSearchResult]`。

### Parser —— 按格式解析为 Section

`ParserBase` 子类各管一种文件格式,**不做切分**,只保留文档的自然结构边界(一页 PDF、一张幻灯片、一个 Markdown 标题段)。4 个内置实现:

| Parser | 支持格式 | Section 粒度 |
|---|---|---|
| `TextParser` | text/markdown 等 | 整个文件为 1 个 Section(或按 Markdown 标题分段) |
| `PDFParser` | `application/pdf` | 每页 1 个 Section,内嵌图片单列 |
| `PPTParser` | PPTX | 每张幻灯片 1 个 Section |
| `ImageParser` | 图像 | 整个文件为 1 个 Section(多模态) |

```python
from agentscope.rag import TextParser

parser = TextParser()
# file 可传 bytes(原始内容),也可传 str(路径;TextParser 则按存在性区分路径/纯文本)
sections = await parser.parse(file=b"# 标题\n正文...", filename="policy.md")
```

- 子类需声明类属性 `supported_media_types: list[str]`(IANA media type),服务端据此路由上传文件。
- `parse(file, filename)` 的 `file` 对二进制解析器(PDF/PPT/Image)可传**文件路径 str**(解析器自己读);`TextParser` 的 str 路径若指向已存在文件则按文件解码,否则视为纯文本。

### Section 与 Chunk(数据结构)

两个管线中间产物,**不落库**,只在内存中流转:

```python
from agentscope.rag import Section, Chunk
```

**`Section`** —— parser 输出的自然边界单元:
- `content: TextBlock | DataBlock`(文本用 TextBlock,多模态用 DataBlock)
- `source: str`(源文件名,带到最后用于引用)
- `metadata: dict`(格式特定,如 PDF `{"page": 3}`、PPT `{"slide": 2}`)

**`Chunk`** —— chunker 输出的最终索引单元(每条对应向量库一条记录):
- `content: TextBlock | DataBlock`
- `source: str`(继承自父 Section)
- `chunk_index: int` / `total_chunks: int`(文档内序号/总数,用于检索时「展开命中点上下文」)
- `metadata: dict`(继承自父 Section)

> 关键约束:Chunk **永不跨 Section**——Section 是硬边界,防止格式结构泄露到下游切分。

### ApproxTokenChunker —— 近似 token 切分

```python
from agentscope.rag import ApproxTokenChunker

chunker = ApproxTokenChunker(
    chunk_size=512,   # 每块最大近似 token 数(默认 512)
    overlap=50,       # 相邻块重叠的近似 token 数(默认 50)
)
chunks = await chunker.chunk(sections)
```

- token 数近似为 `len(text.encode("utf-8")) // 4`——**无需任何 tokenizer 依赖**,数量级对大多数 LLM tokenizer 足够。
- 携带 `DataBlock` 的 Section(图片/视频等)原样作为单条 Chunk 透传。
- 约束:`chunk_size > 0`,且 `0 <= overlap < chunk_size`,否则抛 `ValueError`。

## QdrantStore —— 向量库后端

`QdrantStore` 是目前唯一内置实现(基于 [Qdrant](https://qdrant.tech));自定义后端继承 `VectorStoreBase` 实现抽象方法即可。

```python
from agentscope.rag import QdrantStore

# 三种部署形态(三选一)
store = QdrantStore(location=":memory:")          # 内存(测试/原型)
store = QdrantStore(url="http://localhost:6333")  # 远程 Qdrant 服务
store = QdrantStore(path="./qdrant_data")         # 本地磁盘持久化
# QdrantStore(url=..., api_key="...",             # Qdrant Cloud / 受保护服务
#             distance="Cosine")                   # 默认 Cosine,可选 Dot/Euclid/Manhattan
```

> ⚠️ **必须用 `async with` 进出**:`__aenter__` 打开 client 连接,`__aexit__` 关闭。collection 在首次操作时惰性创建(`ensure_collection` 幂等且记忆化),无需手动建。

## KnowledgeBase —— 运行时句柄

`KnowledgeBase` 把 (embedding 模型, 向量库, collection) 三元组绑在一起,免去每次重复接线:

```python
from agentscope.rag import KnowledgeBase

kb = KnowledgeBase(
    name="company-handbook",        # 面向 Agent/前端的名称
    description="公司 HR 与制度文档",  # 面向 Agent 的说明:LLM 据此判断是否检索
    embedding_model=embedding_model,
    vector_store=store,
    collection="handbook",          # 物理 collection 名(惰性创建)
    # metadata_filter={"tenant_id": "acme"},  # 可选:多租户隔离(见下文)
)
```

四个方法:

| 方法 | 作用 |
|---|---|
| `search(queries, top_k=5, score_threshold=None)` | 批量 embed 查询 → 并发检索 → 去重(按 `document_id, chunk_index`)→ 按 score 降序截 top_k |
| `insert_document(chunks, document_id=None, document_metadata=None)` | 批量 embed 并插入为同一文档,返回 `document_id` |
| `delete_document(document_id)` | 按文档 id 删除其全部记录 |
| `list_documents()` | 列出当前知识库内所有文档摘要 |

关键点:
- **索引与检索必须用同一个 embedding 模型**,否则向量不可比。
- `insert_document` 时 chunk 元数据合并优先级(高者覆盖低者):`metadata_filter` 键(安全边界) > chunk 自身 metadata(解析器写入) > `document_metadata`(文档级,如 filename)。
- `metadata_filter`(多租户隔离):检索/列出**强制**按该过滤;插入**强制**写入这些键——防止解析器 bug 或恶意内容把记录塞进别的租户作用域。`None`(默认)表示该知识库独占 collection。
- 检索返回的 `VectorSearchResult`:`score`(相似度,余弦/点积越高越相似,L2 越低越相似)+ `document_id` + `chunk`。

### 完整索引 + 检索示例

```python
import asyncio
from agentscope.credential import DashScopeCredential
from agentscope.embedding import DashScopeEmbeddingModel
from agentscope.rag import (
    ApproxTokenChunker,
    KnowledgeBase,
    QdrantStore,
    TextParser,
)


async def main():
    credential = DashScopeCredential(api_key="sk-xxx")
    embedding_model = DashScopeEmbeddingModel(
        credential=credential,
        model="text-embedding-v4",
        dimensions=1024,
    )

    store = QdrantStore(location=":memory:")
    async with store:  # 必须:开/关 client 连接
        kb = KnowledgeBase(
            name="demo-kb",
            description="演示语料。",
            embedding_model=embedding_model,
            vector_store=store,
            collection="demo-kb",
        )

        # 1. 索引:bytes → parse → chunk → insert
        parser = TextParser()
        chunker = ApproxTokenChunker(chunk_size=256, overlap=32)
        for filename, content in {"policy.md": b"...", "faq.md": b"..."}.items():
            sections = await parser.parse(file=content, filename=filename)
            chunks = await chunker.chunk(sections)
            doc_id = await kb.insert_document(
                chunks,
                document_metadata={"filename": filename},
            )

        # 2. 检索
        results = await kb.search(queries=["年假多少天?"], top_k=3)
        for r in results:
            print(r.score, r.chunk.source, r.chunk.content)


asyncio.run(main())
```

## RAGMiddleware —— 接入 Agent

`RAGMiddleware` 接收一个或多个 `KnowledgeBase` 句柄(可绑定**不同** embedding 模型),挂到 Agent 的 `middlewares` 列表上即可。两种模式:

| `mode` | 行为 | 暴露工具 | 适用 |
|---|---|---|---|
| `"agentic"`(默认) | 模型自主决定何时搜、搜什么 | `search_knowledge` 工具 | 通用,模型按需检索 |
| `"static"` | 每轮 reply 首次推理时自动检索,命中片段作为**一次性** `HintBlock` 注入上下文 | 无 | 强制每轮都带检索上下文 |

### 构造与参数

```python
from agentscope.middleware import RAGMiddleware

mw = RAGMiddleware(
    knowledge_bases=[kb1, kb2],
    parameters=RAGMiddleware.Parameters(  # None 用默认(agentic, top_k=5)
        mode="agentic",            # "agentic" | "static"
        top_k=5,                   # 1-50,跨所有 KB 合并后的最大返回数
        score_threshold=None,      # 最低相似度,None 不过滤
        emit_hint_event=True,      # static 模式:发 HintBlockEvent 供前端展示
        persist_hint=False,        # static 模式:True 则注入的 hint 保留在上下文中
        # hint_template="...{context}...",  # 必须含且仅含一个 {context}(默认有内置模板)
    ),
)

agent = Agent(
    name="Assistant",
    system_prompt="...",
    model=model,
    toolkit=Toolkit(),
    middlewares=[mw],
)
```

| Parameters 字段 | 默认 | 说明 |
|---|---|---|
| `mode` | `"agentic"` | 检索触发方式 |
| `top_k` | `5` | 每次检索返回的最大 chunk 数(跨所有 KB 合并后),范围 1-50 |
| `score_threshold` | `None` | 最低相似度,仅对余弦/点积有意义 |
| `emit_hint_event` | `True` | static 模式下发 `HintBlockEvent`,前端可展示命中片段 |
| `persist_hint` | `False` | static 模式下是否保留注入的 hint(默认模型调用后即移除,避免污染下一轮) |
| `hint_template` | 内置 | 包裹检索结果的模板,须含**且仅含一个** `{context}` 占位符,否则校验报错 |

### static 模式工作流

- `on_reply`:缓存本轮输入(因 `on_reasoning` 运行时输入可能已不在 `state.context`)
- `on_reasoning`:仅当 `agent.state.cur_iter == 0`(每轮首次推理)触发检索 → 命中片段包装成 `HintBlock` 注入 `state.context` → 可选发 `HintBlockEvent`
- 默认 `persist_hint=False`:该 hint 仅参与本次推理,模型调用后按 block id 移除(其它中间件可往同一载体消息追加自己的 block,互不干扰)

> 「一次性」设计避免了每轮迭代都重新 embed/注入,也避免 hint 堆积污染上下文。

### agentic 模式工具

`list_tools()` 在 agentic 模式下返回单个 `search_knowledge` 工具:
- **单工具 fan-out 多个 KB**:并发检索所有绑定的知识库,结果按 score 合并截 top_k。
- 模型可通过参数 `knowledge_bases=["kb1"]` 按**名称**缩小检索范围(schema 的 enum 被收窄为已装备的 KB 名,LLM 无法杜撰未知名称)。
- 只读操作,`check_permissions` 自动返回 ALLOW。
- 工具描述会列出每个 KB 的 `name: description`,LLM 据此决定是否调用。

### 多 KB(不同 embedding 模型)注意事项

一个 `RAGMiddleware` 可绑定用不同 embedding 模型的 KB,各 KB 独立 embed、独立检索。但**不同 embedding 模型的分数不可严格比较**——当前实现按原始 score 排序合并。若部署中这种差异很重要,建议改用 rank-based 融合(如 RRF);每个 `kb.search` 本身已返回有序结果。

### 接入 Agent 完整示例(static + agentic)

```python
from agentscope.agent import Agent
from agentscope.credential import DashScopeCredential
from agentscope.embedding import DashScopeEmbeddingModel
from agentscope.message import UserMsg
from agentscope.middleware import RAGMiddleware
from agentscope.model import DashScopeChatModel
from agentscope.rag import KnowledgeBase, QdrantStore
from agentscope.tool import Toolkit


def build_agent(name, model, rag_mw):
    """RAG 中间件只是 middlewares 列表的一项,可与其它中间件组合。"""
    return Agent(
        name=name,
        system_prompt="你是一个简洁的助手。有检索上下文时优先使用;不知道就说不知道。",
        model=model,
        toolkit=Toolkit(),
        middlewares=[rag_mw],
    )


async def demo(kb, model):
    # static:首轮推理自动检索并注入 HintBlock
    static_mw = RAGMiddleware(
        knowledge_bases=[kb],
        parameters=RAGMiddleware.Parameters(
            mode="static", top_k=3, emit_hint_event=False,
        ),
    )
    static_agent = build_agent("rag-static", model, static_mw)
    reply = await static_agent.reply(UserMsg("user", "公司每周允许几天远程办公?"))
    print(reply.get_text_content())

    # agentic(默认):暴露 search_knowledge 工具,模型自主调用
    agentic_mw = RAGMiddleware(
        knowledge_bases=[kb],
        parameters=RAGMiddleware.Parameters(mode="agentic", top_k=3),
    )
    agentic_agent = build_agent("rag-agentic", model, agentic_mw)
    reply = await agentic_agent.reply(UserMsg("user", "总结一下发版说明。"))
    print(reply.get_text_content())


async def main():
    credential = DashScopeCredential(api_key="sk-xxx")
    model = DashScopeChatModel(credential=credential, model="qwen-plus", stream=False)
    embedding_model = DashScopeEmbeddingModel(
        credential=credential, model="text-embedding-v4", dimensions=1024,
    )

    store = QdrantStore(location=":memory:")
    async with store:
        kb = KnowledgeBase(
            name="handbook",
            description="公司制度与发版说明。",
            embedding_model=embedding_model,
            vector_store=store,
            collection="handbook",
        )
        # ...此处省略索引步骤(parse → chunk → insert_document)
        await demo(kb, model)
```

## 服务模式(简述)

上述都是**库模式**——你在单进程里自己驱动管线。完整服务模式由 `create_app(knowledge_base_manager=...)` 托管:提供知识库 CRUD、文档上传、后台索引 worker、检索等 REST 接口,并通过消息总线分发索引任务(支持嵌入式 worker 或独立 worker 部署)。

服务端细节见 `examples/agent_service`(后端)与 `examples/web_ui`(聊天式 UI,含知识库管理面板)。本 skill 聚焦库模式 API;若走服务模式,库模式里的 `KnowledgeBase`/Parser/Chunker/embedding 概念完全一致,只是改由服务层调度。
