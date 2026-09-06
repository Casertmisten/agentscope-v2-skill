# Pipeline 流水线 — v2

`agentscope.pipeline` 模块（v2.0.8）：把多个 agent 按固定逻辑编排成一个整体，对外暴露
与 `Agent` 相同的事件流接口（`reply_stream`），可以传给 `launch_console` 等任何接受 agent 的地方。

```python
from agentscope.pipeline import GoalPipeline, PipelineProtocol
```

## PipelineProtocol

协议很简单：实现 `reply_stream(inputs) -> AsyncGenerator[AgentEvent | Msg, None]` 即可
（声明为普通 `def` 返回异步生成器，与 `Agent.reply_stream` 的形态一致）。满足该协议的对象
（Agent 或任意 pipeline）可以互换使用——`launch_console(agent=...)` 等接口接受的类型是
`Agent | PipelineProtocol`（v2.0.8）。

## GoalPipeline — 执行者-校验者循环

让一个 **executor**（执行者）完成目标、一个 **verifier**（校验者）验收，不通过则带着反馈
重试，直到验收通过 / 判定不可达成 / 达到尝试上限：

```python
from agentscope.pipeline import GoalPipeline

pipe = GoalPipeline(
    executor=executor,        # Agent：执行任务的智能体
    verifier=verifier,        # Agent：验收目标的智能体
    max_iters=10,             # 目标达成的最大尝试轮数（默认 10）
)

# 首次输入即目标（goal）：单条 Msg 或 list[Msg]
async for event in pipe.reply_stream(UserMsg("user", "实现并跑通 parser 模块的单元测试")):
    ...   # 透传 executor / verifier 产生的事件（含流式文本、工具调用、HITL 等）

# 也可直接交给终端控制台交互调试
from agentscope.console import launch_console
await launch_console(pipe)
```

工作流程：

1. **执行**：目标附加一条 system-reminder（要求完成后总结产出：文件路径、入口、运行方式等）
   后交给 executor，executor 以结构化输出（内置 `report` 字段）产出执行报告。
2. **验收**：verifier 收到「目标 + 执行报告」，以结构化输出产出验收结果：
   - `pass`：循环结束（正常完成）；
   - `impossible`：判定目标不可达成，记录日志后结束；
   - `fail`：`message` 字段的失败原因（含具体位置，如文件路径/行号）作为 system-reminder
     反馈给 executor 修正重试；已用轮数达到 `max_iters` 后强制结束。
3. **HITL 暂停恢复**：executor / verifier 内部发出 `RequireUserConfirmEvent` /
   `ExternalExecutionResultEvent` 相关事件时循环暂停；把确认/执行结果事件再传入
   `pipe.reply_stream(...)` 即可从暂停处继续（按 `reply_id` 路由到对应 agent）。
   `UserInterruptEvent` 中止的是正在等待的那个 agent 的 reply。迭代轮数记在实例上，
   HITL 恢复不会重置预算。

要点：
- 首次调用 `reply_stream` 传入 `Msg | list[Msg]` 视为新任务，目标就是这次输入，迭代轮数归零。
- executor / verifier 通常**共享同一个 Workspace**（`offloader=workspace` + `Toolkit(tools=await workspace.list_tools())`），
  verifier 才能读到 executor 写下的产物。
- 构造签名还有 `verifier_reset_context=True` / `max_retries=3` 两个参数，当前版本仅存储未生效（预留）。
- 结构化输出不合法时，pipeline 会以 system-reminder 提示对应 agent 重新调用
  `GenerateStructuredOutput`。
