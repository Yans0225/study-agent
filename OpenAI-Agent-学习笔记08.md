# OpenAI Agent 学习笔记 08（运行智能体篇）

> 本文档是「系统性学习 Agent」第八阶段的整理，聚焦 **运行智能体（Running agents）**——也就是**那台把所有零件转起来的发动机 `Runner`**。前七章都在讲零件：Agent（笔记 02）、工具（笔记 03）、handoff（笔记 04）、护栏（笔记 05）、tracing（笔记 06）、context/sessions（笔记 07）。本章把它们装到 `Runner` 上，回答一个最实在的问题：**「`Runner.run` 到底一步步干了什么，我在应用里该怎么用。」**
> 来源：OpenAI Agents SDK 官方中文文档「运行智能体」（running_agents.md）；另核对了 SDK 源码 `agents/run.py`、`agents/run_config.py`、`agents/run_internal/run_steps.py`、`agents/result.py`、`agents/lifecycle.py`。
> 学习日期：2026-10-27（理论学习 + 编写并跑通 `runner_demo.py`）
> 衔接：Agent 循环（笔记 02 埋的"Runner 循环机制"）、handoff 出口（笔记 04）、护栏拉闸（笔记 05）、trace/`group_id`（笔记 06）、session 四策略里的那一格（笔记 07）。
> **状态：✅ 本章已学完（2026-10-27 收尾）**（已学：三个入口 / Agent Loop 与四种出口 / `turn` 定义 / `max_turns` / `RunConfig` / 对话状态四策略 / 两个输入钩子 / `RunHooks` 观察者 / `error_handlers` 与异常 / `runner_demo.py`）。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [这一章讲什么：一句话 + 动机](#二这一章讲什么一句话--动机)
3. [三个入口：run / run_sync / run_streamed](#三三个入口run--run_sync--run_streamed)
4. [⭐ 核心：Agent Loop（这台发动机的转法）](#四-核心agent-loop这台发动机的转法)
5. [`turn` 的定义与 `max_turns` 保险丝](#五turn-的定义与-max_turns-保险丝)
6. [Runner 内部的四种"出口"（源码 `NextStep*`）](#六runner-内部的四种出口源码-nextstep)
7. [`RunConfig`：「这一趟怎么跑」的全局设置](#七runconfig这一趟怎么跑的全局设置)
8. [⭐ 状态与对话管理：四策略表](#八-状态与对话管理四策略表)
9. [两个"改输入"的钩子](#九两个改输入的钩子)
10. [`RunHooks`：把循环"打印出来"的观察者](#十runhooks把循环打印出来的观察者)
11. [错误与恢复：`error_handlers` 与异常一览](#十一错误与恢复error_handlers-与异常一览)
12. [实际应用：`runner_demo.py` 跑出来的三件事](#十二实际应用runner_demopy-跑出来的三件事)
13. [易混概念辨析（汇总表）](#十三易混概念辨析汇总表)
14. [问答记录（Q&A）](#十四问答记录qa)
15. [和前面章节的对照](#十五和前面章节的对照)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 运行智能体 running_agents | https://openai.github.io/openai-agents-python/zh/running_agents/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/running_agents.md |

> ⚠️ 获取记录：本机直连 GitHub 不通。经镜像 `https://ghproxy.net/https://raw.githubusercontent.com/...` 取到全文（同笔记 04/05/06/07 的通道），并已缓存到本地：
> - `E:\agent_study\docs_cache\running_agents.md`
>
> 提问中涉及的行为，另核对了源码：
> - `agents/run.py`：`class Runner` 的 `run` / `run_sync` / `run_streamed` 三个 classmethod；主循环 `while True:`（约 1032 行）；`max_turns` 判定 `if max_turns is not None and current_turn > max_turns`（约 1598 行）；
> - `agents/run_config.py`：`DEFAULT_MAX_TURNS = 10`；
> - `agents/run_internal/run_steps.py`：四种出口 `NextStepHandoff` / `NextStepFinalOutput` / `NextStepRunAgain` / `NextStepInterruption`；
> - `agents/result.py`：`RunResultBase` 字段（`input` / `new_items` / `raw_responses` / `final_output` / 各 `*_guardrail_results` / `context_wrapper`）、`to_input_list()`；
> - `agents/lifecycle.py`：`RunHooksBase`（`on_llm_start` / `on_llm_end` / `on_agent_start` / `on_agent_end` / `on_handoff` / `on_tool_start` / `on_tool_end`）与 `AgentHooksBase`。

---

## 二、这一章讲什么：一句话 + 动机

> **这一章讲那个把零件转起来的引擎 `Runner`：**
> - **Agent**（笔记 02）= 静态配置（一个"我是谁、能用啥"的定义）；
> - **`Runner`**（本章）= 运行时引擎（一个 `while True` 状态机，负责一圈圈调 LLM、跑工具、交接、直至出结果）。

**为什么要有它**：前面每章都在讲"零件怎么造"，但真正在应用里跑的时候，你不会自己去写"调 LLM→看有没有工具调用→跑工具→再调 LLM"这套循环——**这套循环 SDK 替你写了，入口就是 `Runner.run(...)`**。本章就是把 `Runner.run` 内部那台"发动机"拆开看：它转几圈、每圈干什么、什么时候停、停不下怎么办、结果怎么拿、多轮怎么记。

---

## 三、三个入口：run / run_sync / run_streamed

| 方法 | 同步/异步 | 返回 | 用途 |
|---|---|---|---|
| `Runner.run()` | async | `RunResult` | **主力**，异步跑完给结果 |
| `Runner.run_sync()` | 同步 | `RunResult` | 同步外壳，底层直接跑 `run()` |
| `Runner.run_streamed()` | async | `RunResultStreaming` | 边跑边吐事件（打字机效果），用 `.stream_events()` 消费 |

```python
result = await Runner.run(agent, "写一首关于递归的打油诗")
print(result.final_output)
```

**`input` 可以是三样之一**：
- 一个字符串（当用户消息）；
- 一个 OpenAI Responses 格式的输入项列表；
- 一个 `RunState`（**恢复**暂停的运行、或因 `cancel(mode="after_turn")` 停止的运行用）。

**`Runner.run` 的常用参数**（源码签名实锤，`run.py` 约 281 行）：

```
Runner.run(starting_agent, input, *,
           context=None, max_turns=10, hooks=None, run_config=None,
           error_handlers=None, previous_response_id=None,
           auto_previous_response_id=False, conversation_id=None, session=None)
```

---

## 四、⭐ 核心：Agent Loop（这台发动机的转法）

官方开篇一句话：**"智能体会在循环中运行，直到生成最终输出。"** 源码里就是个 `while True:`（`run.py:1032`）：

```
        ┌──────────────────────────────────┐
  ┌────▶│ 1. 用当前 input 调当前 agent 的 LLM  │
  │     └───────────────┬──────────────────┘
  │                     ▼
  │           LLM 产出，Runner 给这次产出分"下一步"（四选一）：
  │                     │
  │     ┌───────────────┼───────────────┬─────────────────┐
  │     ▼               ▼               ▼                 ▼
  │  最终输出        交接           跑了工具          要审批暂停
  │  (FinalOutput)  (Handoff)      (RunAgain)       (Interruption)
  │     │               │               │                 │
  │  循环结束       换 agent+input   追加工具结果       暂停、抛中断
  │  返回结果        重跑循环         重跑循环          等 approve
  └───────────────────────┘
```

**官方三种分支的原文**（背下来）：

1. 若 Runner 把 LLM 输出归类为**最终输出** → 循环结束、返回结果。
2. 若 LLM 请求**任务转移（handoff）** → 更新当前 agent 和输入，**重新运行循环**。
3. 若 LLM 生成**工具调用** → 运行这些工具调用、追加结果，**重新运行循环**。

> ⚠️ **"最终输出"的判定规则（一句话）**：**LLM 生成了所需类型的文本输出，且不存在工具调用**。

> 🧭 **接笔记 02**：笔记 02 里你说过"一步步干、记聊到哪了，是 Runner 干的不是 Agent"——这一节就是那句的**源码级展开**。Agent 只是静态配置，转圈的是 `Runner`。

---

## 五、`turn` 的定义与 `max_turns` 保险丝

**`turn`（轮次）的定义**：**一次 LLM 调用（含其触发的所有工具调用）**。

⚠️ 注意 `turn` **不是**"用户眼里的一个对话轮次"。用户问一句，Runner 可能转好几圈（调 LLM→跑工具→再调 LLM→交接→…）才给答案；**每一圈`current_turn += 1`只在每次真正调 LLM 前 +1**。

```python
result = await Runner.run(agent, "...", max_turns=5)   # 默认 10（run_config.py:45）
```

| 参数 | 行为 |
|---|---|
| `max_turns=10` | **默认值**（源码 `DEFAULT_MAX_TURNS = 10`） |
| `max_turns=N` | 超过就抛 `MaxTurnsExceeded` |
| `max_turns=None` | **关掉限制**（慎用，可能真转不停） |

**判别口径**：`current_turn > max_turns` 才抛（`run.py:1598`）——即允许跑满 `max_turns` 圈。

---

## 六、Runner 内部的四种"出口"（源码 `NextStep*`）

`agents/run_internal/run_steps.py` 里定义了四种"下一步"数据类，就是循环的四个出口：

| 出口类 | 含义 | Runner 下一步 |
|---|---|---|
| `NextStepFinalOutput(output)` | 拿到最终输出 | 结束、返回 `RunResult` |
| `NextStepHandoff(new_agent)` | 交接给新 agent | 换 `current_agent` + input，**重跑循环** |
| `NextStepRunAgain()` | 这轮跑了工具 | 把工具结果追加进 input，**重跑循环** |
| `NextStepInterruption(interruptions)` | 工具有的调用需人工审批 | **暂停**，结果里带 `interruptions`，等你处理 |

> **这张表把前面几章串起来了**：**handoff（笔记 04）= 出口②**、**工具（笔记 03）= 出口③**、**审批暂停 = 出口④**。Runner 就是个"读下一步指令 → 执行 → 再来一圈"的状态机。

---

## 七、`RunConfig`：「这一趟怎么跑」的全局设置

`run_config=` 参数，**一次性覆盖这次运行**，但**不改 Agent 定义**。

> **心智模型**：Agent 上的设置是"**这个 agent 是谁**"；`RunConfig` 是"**这次运行怎么跑**"——同一批 agent，不同 `RunConfig` 能跑出不同行为。

类别很多，**先给目录、用到再深挖**（沿用 tracing 那章的处理方式）：

| 类别 | 关键字段 |
|---|---|
| **模型 / 会话默认** | `model`、`model_provider`、`model_settings`、`session_settings`、`session_input_callback` |
| **护栏 / handoff / 输入整形** | `input_guardrails`、`output_guardrails`、`handoff_input_filter`、`call_model_input_filter`、`nest_handoff_history`(beta) |
| **追踪（笔记 06）** | `tracing_disabled`、`workflow_name`、`trace_id`、`group_id`、`trace_metadata`、`trace_include_sensitive_data` |
| **工具执行 / 审批 / 错误** | `tool_execution`(`max_function_tool_concurrency`)、`tool_not_found_behavior`、`tool_name_collision_policy`、`tool_error_formatter` |

> 一个细节：`tool_execution.max_function_tool_concurrency` 管**SDK 端本地函数工具的并发**；`ModelSettings.parallel_tool_calls` 管**模型端是否允许一次响应里生成多个工具调用**——两者不同层。

---

## 八、⭐ 状态与对话管理：四策略表

官方给了一张表，**这是本章最值钱的一页**——把"下一轮怎么记住上一轮"的四种做法放一起对比：

| 策略 | 状态存哪 | 最适合 | 下一轮传什么 |
|---|---|---|---|
| `result.to_input_list()` | **你的应用内存** | 小型聊天、完全手动控制、任意提供商 | 上一轮列表 + 新用户消息 |
| `session` | **你的存储 + SDK** | 持久化聊天、可恢复、自定义存储 | 同一个 `session` 实例 |
| `conversation_id` | **OpenAI 服务器** | 跨进程/服务共享的具名对话 | 同一个 `conversation_id`，只传新轮次 |
| `previous_response_id` | **OpenAI 服务器** | 最轻量的服务器续接 | `result.last_response_id`，只传新轮次 |

**规则**：
- 前两个**客户端管理**，后两个 **OpenAI 管理且只对 Responses API 有效**。
- **一次运行里，`session` 不能和服务器管理的对话设置（`conversation_id` / `previous_response_id` / `auto_previous_response_id`）混用——二选一**（否则历史记两遍、上下文重复）。
- 大多数应用**每个对话只选一种持久化策略**。

### 8.1 手动 vs 自动：两条路各写一遍

```python
# (A) 手动：to_input_list() —— 自己把上一轮拼进下一轮
result = await Runner.run(agent, "金门大桥在哪个城市？")
new_input = result.to_input_list() + [{"role": "user", "content": "它在哪个州？"}]
result = await Runner.run(agent, new_input)

# (B) 自动：session —— 交给 SDK 管
session = SQLiteSession("conversation_123")
result = await Runner.run(agent, "金门大桥在哪个城市？", session=session)
result = await Runner.run(agent, "它在哪个州？", session=session)
```

> **接笔记 07**：笔记 07 讲了 `session`（四策略里的"客户端+SDK"那一条）；本章把另外三条补齐——**"谁记账"这件事，四种选一：没人 / 你的应用内存（手动）/ SDK+你的存储 / OpenAI 服务器。**

### 8.2 服务器管理的两种方式

```python
# 方式 1：conversation_id（OpenAI Conversations API，可跨服务共享具名对话）
conversation = await client.conversations.create()
result = await Runner.run(agent, user_input, conversation_id=conversation.id)

# 方式 2：previous_response_id（响应链，最轻量）
result = await Runner.run(agent, user_input,
                          previous_response_id=previous_response_id,
                          auto_previous_response_id=True)
previous_response_id = result.last_response_id
```

- `conversation_id` 和 `previous_response_id` **互斥**。
- 要"多系统共享的具名资源" → `conversation_id`；要"最轻量的逐轮续接" → `previous_response_id`。
- 运行因审批暂停后从 `RunState` 恢复时，SDK 会**保留已存的 `conversation_id`/`previous_response_id`** 设置。

---

## 九、两个"改输入"的钩子

| 钩子 | 管哪一步 | 收到 / 返回 |
|---|---|---|
| `session_input_callback` | **更早那步**："新用户输入怎么和 session 历史拼在一起" | 可同步/异步 |
| `call_model_input_filter` | **最后那步**：拼好后、送进 LLM 前，裁剪/替换/注入 | 收 `CallModelData`，**必须返回 `ModelInputData`**（其 `input` 字段必填、必须是输入项列表，否则 `UserError`） |

**顺序**：`session` 拼历史 → `call_model_input_filter` → 调 LLM。

```python
def drop_old_messages(data: CallModelData[None]) -> ModelInputData:
    trimmed = data.model_data.input[-5:]           # 只留最后 5 条
    return ModelInputData(input=trimmed, instructions=data.model_data.instructions)

result = Runner.run_sync(agent, "解释 quine", run_config=RunConfig(call_model_input_filter=drop_old_messages))
```

> Runner 传给钩子的是**输入列表的副本**，所以你可以随意裁剪/替换/重排，不会就地改到调用方的原始列表。

---

## 十、`RunHooks`：把循环"打印出来"的观察者

**这是本章"用起来"的关键工具**——前面讲的 `while True` 是抽象的，`RunHooks` 是**运行时生命周期回调**：Runner 每走一步就喊你一声，于是循环变成屏幕上一行行的字。

`Runner.run(..., hooks=...)` 传入一个 `RunHooks` 子类，按需覆盖（源码 `agents/lifecycle.py` 的 `RunHooksBase`）：

| 回调 | 签名要点 | 触发时机 |
|---|---|---|
| `on_agent_start(context, agent)` | `agent` | agent 被调用前（**每次当前 agent 变化都触发**，含交接后） |
| `on_agent_end(context, agent, output)` | 产出最终输出 | agent 产出最终输出时 |
| `on_llm_start(context, agent, system_prompt, input_items)` | 送给模型的输入 | 调 LLM 前 |
| `on_llm_end(context, agent, response)` | `response` | LLM 返回后 |
| `on_handoff(context, from_agent, to_agent)` | 双方 agent | 发生交接时 |
| `on_tool_start(context, agent, tool)` | `tool` | 本地工具调用前 |
| `on_tool_end(context, agent, tool, result)` | `result` | 本地工具调用后 |

> 另有 **`AgentHooksBase`**（挂在单个 Agent 上，回调是 `on_start` / `on_end` / `on_handoff` / `on_tool_*` / `on_llm_*`），粒度是"**这个 agent 自己的**生命周期"，而 `RunHooks` 是"**整趟运行**"的。

---

## 十一、错误与恢复：`error_handlers` 与异常一览

### 11.1 `error_handlers`（把硬报错变可控输出）

三个键，返回 `RunErrorHandlerResult(final_output=..., include_in_history=...)`：

| 键 | 何时用 |
|---|---|
| `"max_turns"` | 轮次超限时，别抛异常，给个兜底话术 |
| `"model_refusal"` | 模型拒绝时，给应用自己的兜底 |
| `"invalid_final_output"` | 模型消息过不了 `output_type` 校验时，返回符合该 `output_type` 的回退值 |

- `include_in_history` **默认 `True`**（兜底输出会记进历史和 session）；不想记就设 `False`。
- ⚠️ `"invalid_final_output"` 的处理器**不会重跑模型、也不会重放工具副作用**，只补合规回退值。

### 11.2 异常一览（`agents.exceptions`）

| 异常 | 触发场景 |
|---|---|
| `AgentsException` | **所有异常的基类** |
| `MaxTurnsExceeded` | 超过 `max_turns` |
| `ModelBehaviorError` | 模型输出意外（JSON 错、工具用法错、Responses 调用 `failed`/`incomplete`） |
| `ModelTimeoutError` / `ToolTimeoutError` | 模型 / 工具超时 |
| `UserError` | **你（写代码的人）**用错了 SDK |
| `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered` | 护栏拉闸（笔记 05） |

---

## 十二、实际应用：`runner_demo.py` 跑出来的三件事

**文件**：`E:\agent_study\code\runner_demo.py`（配 `run_runner.bat` 一键脚本）。**场景**：电商客服机器人，前台客服 ——(交接)——> 退款专员，一个场景走完全部零件。

### (1) 用 `RunHooks` 把循环现出原形（全场最直观的一击）

`LoopPrinter(RunHooks)` 覆盖 `on_agent_start` / `on_llm_start` / `on_tool_start` / `on_handoff` / `on_agent_end`，跑起来大致长这样：

```
   ▶ [Agent 开始] 前台客服
     · 调 LLM（输入 2 条）          ← 第 1 圈
     · 执行工具 [check_order_status]
     · 调 LLM（输入 4 条）          ← 第 2 圈：看了工具结果后，决定交接
   → 交接：前台客服 ➜ 退款专员
   ▶ [Agent 开始] 退款专员
     · 调 LLM（输入 5 条）          ← 第 3 圈
     · 执行工具 [issue_refund]
     · 调 LLM（输入 7 条）          ← 第 4 圈：看了退款结果，给最终输出
   ■ [Agent 结束] 退款专员 产出最终输出
```

**用户就问了一句话，Runner 转了 4 圈。** 每一行 `· 调 LLM` = 一个 turn。

### (2) 从 `RunResult` 里能拿到什么

```
  final_output    : …                      ← 循环结束时的最终答案
  last_agent      : 退款专员                ← 循环结束时停在谁身上
  new_items 条数  : …                      ← 这趟新产生的消息/工具调用
  to_input_list   : …条（可直接喂给下一轮） ← 手动记忆的原料
  ctx.refunded_orders: ['A20315']          ← 代码记的，不是模型
```

### (3) 演示二 / 三

- **`max_turns` 保险丝**：同任务 `max_turns=1` → 必被拦，抛 `MaxTurnsExceeded`（任务要 好几圈，1 圈装不下）。
- **多轮记忆两条路**：`to_input_list()` 手动拼 vs `session` 自动记——**效果一样，区别是"谁当记账本"**。

> **抽象 → 具体 对照**（回答"分开都懂、合起来不会用"）：
>
> | 抽象（讲概念时） | 具体（demo 里） |
> |---|---|
> | Agent Loop 四出口 | 屏幕上 `调LLM→跑工具→调LLM→交接→…` 那一串 |
> | `NextStepRunAgain` | "执行工具 […]" 之后又 "调 LLM" |
> | `NextStepHandoff` | "→ 交接：前台客服 ➜ 退款专员" |
> | `NextStepFinalOutput` | "■ [Agent 结束] … 产出最终输出" |
> | `turn` = 一次 LLM 调用 | 每一行 "· 调 LLM" |
> | `max_turns` 保险丝 | 演示二：`max_turns=1` → `MaxTurnsExceeded` |
> | `RunResult` 行程单 | 跑完打印的 `final_output / last_agent / new_items / to_input_list()` |
> | `RunConfig` 全局设置 | `run_config=RunConfig(workflow_name="runner_demo")` |
> | 四策略的 `to_input_list` vs `session` | 演示三：两条路 |
> | `hooks=` 参数 | `LoopPrinter()` |

---

## 十三、易混概念辨析（汇总表）

### 13.1 "有没有记忆" vs "要不要手动输入"（本章最易混）

**这是两个独立的问题**，不要混在一起：

| 做法 | 谁保存历史 | 每轮你往 `input` 里传什么 | 要你手动搬历史吗 |
|---|---|---|---|
| **不传（默认）** | **没人存** | 只有本轮新消息 | 没记忆可言 |
| **`session=`** | SDK + 你的存储 | 新消息 + `session=同一个对象` | ❌ 不用，SDK 自动 prepend |
| **`to_input_list()`** | 你的应用内存 | **全量历史 + 新消息** | ✅ **要**（唯一的"手动"） |
| **`conversation_id=`** | OpenAI 服务器 | 新消息 + `conversation_id` | ❌ 不用，服务器替你存 |
| **`previous_response_id=`** | OpenAI 服务器 | 新消息 + `previous_response_id` | ❌ 不用，服务器替你存 |

> ⚠️ **核心澄清**：**`session` 从来不是默认，默认是"什么记忆都没有"。** `session` 只是四条"记账方案"里的一条。**"没有 session" ≠ "没有记忆"**——记忆可以由另外三条担当；而且"非 session"的三条里，**只有 `to_input_list()` 需要手动输入**，`conversation_id` / `previous_response_id` 也是自动的（只是记账人换成了 OpenAI 服务器）。
>
> 类比：`session` 不是"记忆",而是"**一个自动记账本**"。想让对话记住：要么自己记账（`to_input_list`）、要么用 SDK 的记账本（`session`）、要么让 OpenAI 服务器记账（`conversation_id`/`previous_response_id`）。

### 13.2 其他易混对

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **Agent** | **Runner** | 前者=静态配置（我是谁、能用啥）；后者=运行时引擎（转圈的状态机） |
| **一个 `turn`** | **用户眼里的一个对话轮次** | 前者=**一次 LLM 调用**（含其工具调用）；用户问一句可能转好几圈 |
| `max_turns=10` | `max_turns=None` | 前者=默认保险丝（超了抛 `MaxTurnsExceeded`）；后者=关掉限制 |
| `NextStepRunAgain` | `NextStepHandoff` | 都是"重跑循环"，但前者=同一 agent 继续（跑了工具），后者=**换了 agent** |
| `RunConfig` | Agent 上的设置 | 前者="这次运行怎么跑"（覆盖单次）；后者="这个 agent 是谁" |
| `session_input_callback` | `call_model_input_filter` | 前者=更早（拼 session 历史）；后者=更晚（调 LLM 前最后一道） |
| `client-managed`（`to_input_list`/`session`） | `server-managed`（`conversation_id`/`previous_response_id`） | 前者状态在你/SDK；后者状态在 OpenAI 服务器，**同一次运行不可混用** |
| `RunHooks`（整趟） | `AgentHooks`（单个 agent） | 前者观察**整次运行**的生命周期；后者是**某个 agent 自己**的 |
| `tool_execution.max_function_tool_concurrency` | `ModelSettings.parallel_tool_calls` | 前者=**SDK 端**如何执行本地函数工具；后者=**模型端**能否一次生成多个工具调用 |
| `FinalOutput`（判定） | "有文本输出" | 必须**同时**满足"所需类型文本**且无工具调用**"才算最终输出 |

---

## 十四、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | "你讲的这几个部分分开我都能看懂，但具体应用时到底怎么用" | **用 `Runner.run(agent, input, ...)` 当应用的"主函数"**：它自己转圈到出结果，还你一张 `RunResult` 行程单；你要做的是**读行程单 + 代码里的 ctx 状态**，而不是自己写循环。用一个**场景走完全部零件**的 demo（`runner_demo.py`：前台客服→交接→退款专员）来体感；用 `RunHooks` 把循环打印出来，抽象瞬间变具体 |
| 2 | "什么时候会没有 session？用另外三种方式时就都没 session、都要手动输入吗" | **① `session` 不是默认，"不传 `session=`" 就是没有 session（默认世界=无记忆）**；② 四条是**并列的四种记账方案**，选了别的就没有 session 对象，但**不等于没记忆**；③ "要手动输入"的**只有 `to_input_list()` 一条**——`conversation_id`/`previous_response_id` 也**不用**手动输入（服务器替你存）；④ 同一次运行里 `session` 与服务器管理方式**不可混用** |

---

## 十五、和前面章节的对照

| | 前七章（零件） | 本章（发动机） |
|---|---|---|
| 角色 | 造零件：Agent / 工具 / handoff / 护栏 / trace / context+session | 装到 `Runner` 上转起来 |
| 关键词 | `Agent`、`@function_tool`、`handoff()`、`tripwire`、`span`、`RunContextWrapper`、`session` | `Runner.run`、Agent Loop、`NextStep*`、`turn`、`max_turns`、`RunConfig`、`RunHooks`、`RunResult` |
| 四出口映射 | handoff（笔记 04）= 交接出口；工具（笔记 03）= RunAgain 出口；审批暂停 = Interruption 出口；护栏拉闸（笔记 05）= 直接抛异常不走出口 | |
| 对话记忆 | 笔记 07 讲了 `session` 这一条 | 本章补齐四策略：`to_input_list` / `session` / `conversation_id` / `previous_response_id` |

> **一句话串起来**：前面七章教你"**有哪些零件、各自啥脾气**"；本章教你"**把这些零件接到 `Runner` 这台发动机上，一句话跑起来，然后从行程单里读结果**"。

---

*运行智能体篇（三个入口 / Agent Loop 与四种出口 / `turn` 定义 / `max_turns` / `RunConfig` / 对话状态四策略 / 两个输入钩子 / `RunHooks` / `error_handlers` 与异常 / `runner_demo.py`）**已完成**。*

*后续待学：results、models。*