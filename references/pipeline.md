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
- 构造签名还有 `verifier_reset_context=True` / `max_retries=3` 两个参数。`verifier_reset_context`
  已生效（v2.0.8+ 主干）：每轮迭代结束后清空 verifier 的 `context` 与 `summary`（工具缓存、任务状态保留），
  让每轮验收从干净上下文开始，避免上一轮验收历史污染判断；`max_retries` 为 executor/verifier
  生成合法结构化输出的重试次数（默认 3）。
- 结构化输出不合法时，pipeline 会以 system-reminder 提示对应 agent 重新调用
  `GenerateStructuredOutput`。

## SOP 标准作业程序（v2.0.8+ 主干）

`agentscope.sop` 模块：**固定顺序的里程碑序列**，每一步完成并验证通过后才进入下一步。与
`GoalPipeline` 的"单目标循环重试"不同，SOP 表达的是"一份可复用的流程定义"——定义本身
不携带任何运行状态，同一个 `SOP` 对象可以驱动任意多次运行，每次运行的进度记录在
`SOPRunState`（纯 Pydantic 数据，可持久化、跨进程存活）。

```python
from agentscope.sop import SOP, SOPStep, SOPEngine, SOPRunState
from agentscope.message import UserMsg

sop = SOP(
    name="需求交付",
    description="从需求到交付的三步流程",
    steps=[
        SOPStep(subject="写方案", description="给出技术方案与改动点", executor=writer),
        SOPStep(subject="实现", description="按方案完成编码与自测", executor=coder,
                verifier=reviewer, max_attempts=3),
        SOPStep(subject="验收", description="端到端验证交付目标", executor=tester,
                verifier=acceptor),
    ],
)

engine = SOPEngine(sop)                    # 第二个参数 state=SOPRunState(...) 可续接已有运行
async for event in engine.reply_stream(UserMsg("user", "给 parser 模块加上缓存")):
    ...    # 透传各步骤 executor/verifier 的全部事件（含 HITL）
```

### 核心概念：handover 隔离

步骤之间**只通过 handover（交接物）传递信息**：executor 以结构化输出（内置 `handover` 字段）
交出"给后续步骤看的东西"——下一步看不到上一步的文件、工具输出或对话，只有这份交接说明。
每步只判 handover 是否兑现了该步 `description` 承诺的目标，不管它是怎么做到的。

### SOPStep — 常规步骤

`SOPStep(subject, description, executor, verifier=None, max_attempts=3)`：

- **executor**（`AgentLike`，即任意满足 `reply_stream(..., structured_schema=...)` 协议的
  Agent / pipeline）：干活的智能体。一次调用即一次尝试：干活 → 产出 handover 结构化输出。
  复用同一个 executor 则各步共享其上下文；每步各给一个则互不可见。
- **verifier**（可选）：裁决者，收到「该步的目标 + 上一步的 handover 提交物」，以结构化输出
  产出裁决 `{passed, message}`。`None` 表示直接通过（适用于"只需要发生"的步骤）。
- **max_attempts**：被拒绝多少次后放弃（由 engine 执行预算控制，默认 3）。拒绝时
  `message`（要求写明具体哪里不对、怎么办）原样回传给 executor 重试。

### SOPEngine — 运行与挂起恢复

`SOPEngine` 形如一个 agent（`reply_stream`），可直接传给 `launch_console` 等接口：

- 遇到 `RequireUserConfirmEvent` / `RequireExternalExecutionEvent` 时**释放事件流并挂起**
  （步骤进入 `AWAITING`，不持有协程）；把确认/执行结果事件再传入 `engine.reply_stream(...)`
  即可从挂起处继续——答案会路由到当时挂起的那个步骤。
- `UserInterruptEvent` 放弃当前尝试（不计入尝试预算），步骤回到 `PENDING` 等待全新开始。
- 每步开始/结束时发 `CustomEvent`：`SOP_STEP_STARTED`（value 含 step / attempt）与
  `SOP_STEP_ENDED`（value 含 step / phase）。
- 步骤定义与运行状态可各自演化：`SOPEngine(sop, state)` 续接时若 state 的步数与 sop 不符
  抛 `ValueError`（说明流程定义被改过）。

### 状态模型（可持久化）

- `SOPRunState`：一次运行的全部可存档内容——`id` / `inputs`（首次输入）/ `steps` / `created_at`；
  `phase` 属性由各步推导（任一 FAILED 即 FAILED；任一 AWAITING 即 AWAITING；全部 COMPLETED
  才算 COMPLETED）。
- `SOPStepRunState`：单步记录——`phase` / `given`（本次尝试拿到的输入）/ `submission`
  （handover 提交物，`list[TextBlock|DataBlock]`）/ `verifications`（已落定的裁决列表，
  其长度即已用尝试次数）。
- `SOPPhase`：`PENDING / RUNNING / AWAITING / COMPLETED / FAILED`。
- `VerificationResult`：一条裁决——`passed` / `message`（为何被拒）/ `verifier`（谁裁的）/
  `created_at`。

### 自定义步骤

继承 `SOPStepBase`（`SOPStep` 的父类）自定义任意步骤形态，约定只有两条：**一次
`reply_stream` 调用是一次尝试**；在传入的 `state` 上记录进度（`phase` / `submission` /
`verifications`）。步骤需要记住更多东西时，子类化 `SOPStepRunState` 并在类属性
`state_type` 上声明——额外字段在持久化往返后仍保留（`model_config` 允许 extra 字段）。
