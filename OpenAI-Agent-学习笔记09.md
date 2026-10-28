# OpenAI Agent 学习笔记 09（结果篇）

> 本文档是「系统性学习 Agent」第九阶段的整理，聚焦 **结果（Results）**——也就是 **`Runner.run` 转完还你的那张"行程单" `RunResult`**。上一章（笔记 08 运行智能体）拆的是"发动机怎么转"；这一章拆的是"转完交卷，卷子上每栏是什么、什么时候用哪栏"。
> 来源：OpenAI Agents SDK 官方中文文档「结果」（results.md）；另核对了 SDK 源码 `agents/result.py`（`RunResultBase` / `RunResult` / `RunResultStreaming`）、`agents/items.py`（`RunItem` 各子类、`ToolApprovalItem`）、`agents/run_state.py`（`RunState.approve/reject/get_interruptions/add_input`）、`agents/agent.py`（`Agent.as_tool()`）。
> 学习日期：2026-10-27（理论学习 + 编写并通过编译 `results_demo.py`）
> 衔接：run/run_streamed 的返回值（笔记 08）、handoff 边界（笔记 04）、工具调用项（笔记 03）、护栏结果（笔记 05）、trace 的原始响应视角（笔记 06）、`to_input_list` vs `session`（笔记 07）。
> **状态：✅ 本章已学完（2026-10-27 收尾）**（已学：两种结果类型 + 共同基类 / 选栏总表 / `final_output` / 四兄弟辨析 / `new_items` 项目类型 / 继续与恢复对话 / `agent_tool_invocation` / 流式生命周期与诊断 / `results_demo.py`）。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [这一章讲什么：一句话](#二这一章讲什么一句话)
3. [两种结果类型 + 共同基类](#三两种结果类型--共同基类)
4. [⭐ 选哪栏：官方"如果你需要…用哪个"总表](#四-选哪栏官方如果你需要用哪个总表)
5. [`final_output`](#五final_output)
6. [⭐ 四兄弟辨析：`input` / `to_input_list()` / `new_items` / `raw_responses`](#六-四兄弟辨析input--to_input_list-new_items-raw_responses)
7. [继续 / 恢复对话](#七继续--恢复对话)
8. [`agent_tool_invocation`（`Agent.as_tool` 嵌套元数据）](#八agent_tool_invocationagentas_tool-嵌套元数据)
9. [流式生命周期与诊断](#九流式生命周期与诊断)
10. [实际应用：`results_demo.py`](#十实际应用results_demopy)
11. [易混概念辨析（汇总表）](#十一易混概念辨析汇总表)
12. [问答记录（Q&A）](#十二问答记录qa)
13. [和前面章节的对照](#十三和前面章节的对照)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 结果 results | https://openai.github.io/openai-agents-python/zh/results/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/results.md |

> ⚠️ 获取记录：本机直连 GitHub 不通，经镜像 `https://ghproxy.net/https://raw.githubusercontent.com/...` 取到全文（同 04–08 的通道），已缓存到 `E:\agent_study\docs_cache\results.md`。
> 源码核对：`agents/result.py:369` `class RunResultBase`（字段 `input`/`new_items`/`raw_responses`/`final_output`/四个护栏结果/`context_wrapper`）、`result.py:496` `to_input_list`、`result.py:521` `agent_tool_invocation`、`result.py:538` `last_response_id`、`result.py:547` `class RunResult`、`result.py:660` `class RunResultStreaming`（`current_agent`/`is_complete`/`cancel`）。

---

## 二、这一章讲什么：一句话

> **`Runner.run` 转完，还你一张"行程单" `RunResult`。本章就一栏一栏讲清：这张单子上有什么、各自回答什么问题、什么时候用哪栏。**

- 上一章（笔记 08）答"**发动机怎么转**"（Agent Loop、四出口、turn、max_turns）；
- 本章答"**交卷后卷子上写了啥、怎么读**"。

---

## 三、两种结果类型 + 共同基类

| 入口 | 返回类型 |
|---|---|
| `Runner.run(...)` / `Runner.run_sync(...)` | **`RunResult`** |
| `Runner.run_streamed(...)` | **`RunResultStreaming`** |

- 两者都继承 **`RunResultBase`** → 共享接口：`final_output`、`new_items`、`last_agent`、`raw_responses`、`to_state()`、`input`、`to_input_list()`、`last_response_id`…
- **`RunResultStreaming`** 额外加流式专用控制：`stream_events()`、`current_agent`、`is_complete`、`cancel(...)`。

> **一句话**：`RunResult` = "跑完的交卷"；`RunResultStreaming` = "边跑边看的直播"。**卷子上栏目是同一套**。

---

## 四、⭐ 选哪栏：官方"如果你需要…用哪个"总表

| 如果你需要…… | 用 |
|---|---|
| 向用户显示**最终答案** | `final_output` |
| **可直接重放**的下一轮输入列表 | `to_input_list()` |
| 带 agent/工具/交接/审批**元数据**的丰富运行项 | `new_items` |
| 下一轮**该复用的 agent** | `last_agent` |
| 用 `previous_response_id` 串 Responses API | `last_response_id` |
| 待审批 + 可恢复快照 | `interruptions` + `to_state()` |
| 嵌套 `Agent.as_tool()` 调用的元数据 | `agent_tool_invocation` |
| 原始模型调用 / 护栏诊断 | `raw_responses` + 各护栏结果数组 |

---

## 五、`final_output`

类型取决于**最后完成运行的那个 agent**：

| 情况 | `final_output` 是什么 |
|---|---|
| 最后一个 agent **没定义** `output_type` | `str` |
| 定义了 `output_type` | 该类型的**对象**（Pydantic / dataclass） |
| 运行**在出最终输出前就停了**（如等审批暂停） | **`None`** |

- 它的**静态类型是 `Any`**——因为 **handoff 会改变"最终完成运行的 agent"**，SDK 无法静态确定所有可能的输出类型。
- **流式下一直是 `None`，直到流跑完**才填上。
- 便利方法 **`final_output_as(cls, raise_if_incorrect_type=False)`**：默认只给类型检查器看；设 `True` 时类型不对会抛 `TypeError`。

---

## 六、⭐ 四兄弟辨析：`input` / `to_input_list()` / `new_items` / `raw_responses`

**分别回答不同的问题**：

| 栏目 | 装的是什么 | 最适合 |
|---|---|---|
| **`input`** | 这次运行的**基础输入**（若 handoff input_filter 重写过历史，这里是**过滤后**的版本） | 审计"这次实际用了什么输入" |
| **`to_input_list()`** | **可重放的输入项列表**（默认 `mode="preserve_all"`） | 手动聊天循环、客户端管理状态、普通历史检查 |
| **`new_items`** | 富 `RunItem` **包装器**（带 agent/工具/交接/审批元数据） | 日志、UI、审计、调试 |
| **`raw_responses`** | 每次模型调用的原始 `ModelResponse` | 提供商级诊断、原始响应检查 |

**官方 5 条口诀**：

1. 要**普通输入项视图** → `to_input_list()`。
2. 交接过滤 / 嵌套交接历史**重写过**历史后，要"下一轮真正会用的规范输入" → `to_input_list(mode="normalized")`。
3. 想让 **SDK 帮你存取历史** → 用 `session=...`（笔记 07）。
4. 用**服务器管理状态**（`conversation_id` / `previous_response_id`）→ **只传新用户输入 + 复用 ID**，别再整个重发 `to_input_list()`（否则历史记两遍）。
5. 要**完整转换后的历史**（日志/UI/审计）→ 默认 `to_input_list()` 或 `new_items`。

> **`preserve_all` vs `normalized`**：默认 `preserve_all` 把 `new_items` 转成完整明文历史；`normalized` 在"交接过滤改写了模型历史"时**优先用规范的续接输入**。**普通运行两者一样。**

> **一句辨析**：`to_input_list()` = "**能喂回下一轮的原料**"；`new_items` = "**带元数据的原始账目**"；`input` = "**这次开局用的底稿**"；`raw_responses` = "**欠模型家的原始账单**"。

### 6.1 `new_items` 会出现哪些项目类型（官方清单）

- `InputItem`（恢复运行前接纳的暂存输入）、`MessageOutputItem`（助手消息）、`ReasoningItem`（推理项）
- `ToolCallItem` + `ToolCallOutputItem`（工具调用及其结果）
- `ToolApprovalItem`（**因等审批而暂停**的工具调用）
- `MCPApprovalRequestItem` / `MCPApprovalResponseItem` / `MCPListToolsItem`（托管 MCP）
- `HandoffCallItem` + `HandoffOutputItem`（**交接请求 / 已完成交接**）
- `ToolSearchCallItem` / `ToolSearchOutputItem`（工具搜索）

> **选型口诀**：需要"**归属边界**"（谁调的工具、哪段是交接、哪里要审批）→ 用 **`new_items`**，别用 `to_input_list()`。

---

## 七、继续 / 恢复对话

| 场景 | 用什么 |
|---|---|
| 下一轮**复用谁** | **`last_agent`**（交接后通常是接手的那个人）；流式下用 **`current_agent`**（随运行更新，可在流结束前观察交接） |
| 工具**要审批**而暂停 | **`result.interruptions`** → **`result.to_state()`** → `state.approve/reject(it)` → **`Runner.run(agent, state)`** 续跑 |
| 暂停后又有新用户输入 | **`RunState.add_input()`**（字符串=用户消息，多次调用保序；随 state 序列化保留） |
| **服务器续接** | **`last_response_id`** 当下一轮的 `previous_response_id` |

**恢复的标准姿势**（官方示例）：

```python
result = await Runner.run(agent, "删除不再需要的临时文件。")
if result.interruptions:
    state = result.to_state()
    for it in result.interruptions:
        state.approve(it)                        # 也可 state.reject(it)
    result = await Runner.run(agent, state)      # ← 用 state 当 input 续跑
```

- `last_response_id` **只是 `raw_responses` 最后一条的 ID**（源码就是 `raw_responses[-1].response_id`）；**只有走 `previous_response_id` 服务器链时才用得上**（详解见第十二节 Q&A）。
- 恢复的边界（**点到为止**，用到再挖）：SDK 保证"**不重复执行已完成的工具/护栏/钩子/交接**"；Session 历史对不上时出于安全**抛 `UserError`**；已接受最终输出后才失败的运行**被标记为不可恢复**。

---

## 八、`agent_tool_invocation`（`Agent.as_tool` 嵌套元数据）

### 8.1 背景：Agent 可以当成"工具"用（`Agent.as_tool()`）

除了"两个 agent **平级交接**"（handoff，笔记 04），还有另一种玩法：**把一个 agent 包成"一个工具"，塞进另一个 agent 的工具箱**：

```python
inner = Agent(name="翻译", instructions="把用户的话翻成英文。", model=model)

outer = Agent(
    name="协调员",
    instructions="用 translate 工具把用户的话翻成英文。",
    model=model,
    tools=[inner.as_tool(tool_name="translate", tool_description="把中文翻成英文")],
)   # ↑ inner 这个 agent 被包成名叫 translate 的工具
```

调用 `translate` 会**启动一次完整的嵌套运行**（内层也会转圈、能用工具、能交接），跑完**把结果当"工具返回值"还给外层**。

> ⚠️ **`as_tool` vs `handoff`**：handoff 是**平级交接**（交出去后接活的人变主控制者，笔记 04）；`as_tool` 是**下级调用**（内层跑完**把结果还回外层**，外层继续自己的循环）。

### 8.2 它是什么

**"被当成工具调起的那一次运行"**，它的 `RunResult` 上会挂 **`agent_tool_invocation`**，装**外层那次工具调用的元数据**：

| 字段 | 含义 |
|---|---|
| `tool_name` | 外层调的是哪个工具（如 `"translate"`） |
| `tool_call_id` | 这次工具调用的 ID |
| `tool_arguments` | 外层**原始**传给它的参数 |

**两条规则**：
- **凡"作为工具被 `as_tool()` 调起"的运行 → 有** `agent_tool_invocation`；
- **顶层运行（没人把它当工具）→ `None`**。
- （⚠️ 不是按"内/外层"分层：**三层结构**里，**中间层自己也是"被调起的"，它的结果上也有** `agent_tool_invocation`；唯独**最顶层永远是 `None`**。）

### 8.3 它用在哪：`custom_output_extractor` 里（官方点名）

`as_tool()` 可传 **`custom_output_extractor`**：内层跑完后，用它把内层结果**加工成"外层看到的工具返回值"**。加工时常需知道"**外层那次调用是谁、哪一次、传了什么原始参数**"——`agent_tool_invocation` 正好递上这三要素。

配套：若还要"**这次嵌套运行解析后的结构化输入**"，读 **`context_wrapper.tool_input`**（`RunState` 为嵌套工具输入做通用序列化时存的字段）。

---

## 九、流式生命周期与诊断

- `RunResultStreaming` 继承全部栏目，外加：`stream_events()` / `current_agent` / `is_complete` / `cancel(...)`。
- **必须把 `stream_events()` 消费到底**——迭代器没结束，运行就不算完成（`final_output`/`interruptions`/session 落盘可能仍在收尾）。调了 `cancel()` 也**要继续消费**以完成清理。
- **`raw_responses`**：多步骤运行会有多条（跨交接、多轮 模型/工具/模型）。每条 `ModelResponse` 还带两项诊断：
  - **`request_id`**：传输请求 ID（内置 Responses/ChatCompletions 适配器会传播 `x-request-id`；兼容端点/部分适配器可能 `None`）；
  - **`raw_usage`**：需 `ModelSettings(preserve_raw_usage=True)` 开启的提供商用量载荷快照。
- **护栏结果数组**（笔记 05）：`input_guardrail_results` / `output_guardrail_results` / `tool_input_guardrail_results` / `tool_output_guardrail_results`——**整个运行过程中累积**，适合记录决策 / 调试被拦原因。
- **`context_wrapper.usage`**：用量在这（笔记 07 的 `RunContextWrapper`；流式下可能在处理完最后数据块前延迟）。

---

## 十、实际应用：`results_demo.py`

**文件**：`E:\agent_study\code\results_demo.py`（配 `run_results.bat`）。五个演示：

| 演示 | 看什么 | 对应本笔记 |
|---|---|---|
| ① 简单运行 | 五栏一次打全（`final_output`/`last_agent`/`input`/`new_items`/`raw_responses`）+ `to_input_list()`/`last_response_id`/`final_output_as()` | 三、四、五、六 |
| ② `output_type` | `final_output` 不再 `str`，而是 `Weather` 对象 | 五 |
| ③ 交接 | `last_agent` **换人** + `new_items` 里的 `HandoffCallItem` / `HandoffOutputItem` | 六、七 |
| ④ `needs_approval=True` | `final_output` 为 `None`、`interruptions` 非空 → `to_state().approve()` → `Runner.run(agent, state)` 续跑 | 五、七 |
| ⑤ `Agent.as_tool` | 顶层 `agent_tool_invocation=None`；**嵌套 `extractor` 里**拿到 `(tool_name, tool_call_id, tool_arguments)` 与 `context_wrapper.tool_input` | 八 |

一个辅助函数 `dump_new_items()` 会把每条 `new_items` 的**包装器类名 + 底层 `raw_item.type`** 打出来，让"富包装器"这件事一眼可见。

---

## 十一、易混概念辨析（汇总表）

### 11.1 本章"四兄弟"（最核心）

| 概念 A | 概念 B | 区别 |
|---|---|---|
| `to_input_list()` | `new_items` | 前者=**可重放输入项**（喂下一轮）；后者=**带元数据的原始账目**（日志/UI/审计） |
| `input` | `to_input_list()` | 前者=**本次运行的底稿输入**（可能被 handoff 过滤）；后者=**基于 `new_items` 的可重放视图**（含本次新产生的） |
| `raw_responses` | `new_items` | 前者=**原始模型响应**（提供商级）；后者=**转换后的富运行项** |
| `to_input_list(mode="preserve_all")` | `to_input_list(mode="normalized")` | 前者=完整明文历史；后者=交接过滤改写历史后**优先用规范续接输入**（普通运行两者一样） |

### 11.2 "锚点"辨析（服务器续接的两条路）

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **`previous_response_id`** | **`conversation_id`** | 前者锚在"**某一条响应**"（**指针链**，沿链回溯取历史，可分支/回退，要链尾 ID）；后者锚在"**整个对话**"（**具名容器**，一整叠直接端出，天然可跨服务共享）。**两者互斥，一次运行不可混用** |
| `last_response_id` | `conversation_id` | 前者=**响应 ID**，只服务 `previous_response_id` 这条"指针链"；后者=**对话 ID**，是"容器名"，**不需要**响应 ID |

### 11.3 其他易混对

| 概念 A | 概念 B | 区别 |
|---|---|---|
| `RunResult` | `RunResultStreaming` | 前者=跑完的交卷；后者=边跑边看的直播（多 `stream_events`/`current_agent`/`is_complete`/`cancel`） |
| `last_agent` | `current_agent` | 前者=**结束时**停在谁（非流式）；后者=**运行中**活跃 agent（**流式**，随交接更新） |
| `final_output` | `final_output_as(cls, ...)` | 前者=原样的 `Any`；后者=按指定类型取出（可要求类型不对就抛 `TypeError`） |
| `handoff` | `Agent.as_tool()` | 前者=**平级交接**（接活者掌舵，笔记 04）；后者=**下级调用**（内层跑完把结果还回外层） |
| `agent_tool_invocation` | `context_wrapper.tool_input` | 前者=被 `as_tool` 调起的运行上**直接暴露**的外层工具元数据；后者=`RunState` 为**嵌套工具输入**的通用序列化字段 |
| **被 `as_tool` 调起的运行** | **顶层运行** | 前者**有** `agent_tool_invocation`；后者**为 `None`** |

---

## 十二、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | "`last_response_id` 只是 `raw_responses` 最后一条的 ID；如果用 `to_input_list()`/`session`/`conversation_id` 续接，通常不需要它——没看懂" | **分两层**：①字面上它=源码 `raw_responses[-1].response_id`，是**列表末项的快捷取数**；②真正原因是**两种记账哲学**——**服务端记账**（`previous_response_id`）时你只递"**上一轮响应 ID 这个页码**"，服务器自己接；**客户端记账**（`to_input_list`/`session`）时你递的是"**历史内容本身**"，页码没用；`conversation_id` 另有锚点也不用它。**两条路不能混用**，选一条就走到底 |
| 2 | "用 `conversation_id` 时也不需要 `last_response_id`——为什么？" | 因为**两个服务端机制的"锚点"不同**：`previous_response_id` 是**指针链**（锚在"某一条响应"，必须给链尾 ID 才能回溯取历史）；`conversation_id` 是**具名容器**（锚在"整个对话"，服务器把容器里一整叠端出，**无所谓上次读到哪条**）。类比：前者=**书签继续读**（夹在具体某页）；后者=**走进一个房间**（里面的东西全在，跟"上一个是谁"无关）。这也是 `conversation_id` 能**跨服务共享**、`previous_response_id` 能**分支/回退**的原因 |
| 3 | "`agent_tool_invocation` 没太看懂" | **背景**：`Agent.as_tool()` 把**一个 agent 包成"一个工具"**塞进另一个 agent（内层跑完**把结果还回外层**，与 handoff 平级交接不同）；**字段**=被 `as_tool` 调起的**那一次运行**的结果上，挂外层工具调用的**三要素**（`tool_name`/`tool_call_id`/`tool_arguments`）；**用在哪**=`custom_output_extractor` 里（加工内层结果时想知道"被谁、哪一次、什么参数调起"）；**配套**=结构化输入读 `context_wrapper.tool_input` |
| 4 | "我是不是可以理解为：agent 被嵌套当工具时，**只有内层**能看到那三个元数据，**外层没有**？" | ✅ **两层结构下完全对**。拧紧两点：①不是"用出来"的，是**该次运行的 `RunResult` 自带的属性**（你只是读它）；②真正的规则按**"是不是被 `as_tool()` 当工具调起"**——**凡是"被当工具调起的运行"都有**，**顶层运行永远 `None`**。所以"内层有、外层没有"只在"外层=顶层"时成立；**三层结构**里**中间层也有** |

---

## 十三、和前面章节的对照

| | 前八章（零件 / 发动机） | 本章（行程单） |
|---|---|---|
| 角色 | 造零件 + 装上 `Runner` 转起来 | **读转完交的那张卷子** |
| 关键词 | `Agent`、工具、`handoff()`、`tripwire`、`span`、`session`、`Runner.run`、Agent Loop、`NextStep*` | `RunResult`、`final_output`、`new_items`、`to_input_list()`、`last_agent`、`to_state()`、`agent_tool_invocation`、`last_response_id` |
| tracing（笔记 06） | span 树 = "另开的行车记录仪" | `raw_responses`/`request_id` = "**行程单上的原始账单**"，同一趟的另一个视角 |
| 护栏（笔记 05） | 护栏如何拦 | 四个 `*_guardrail_results` 数组 = "**结果在哪看**" |
| session（笔记 07） | 四策略里的一条 | 结果单上对应栏目补齐（`input`/`to_input_list`/`last_response_id`） |
| running_agents（笔记 08） | 四种 `NextStep` 出口 | `Interruption` 出口 ≈ 本章 `interruptions` + `to_state()` 的暂停恢复 |

> **一句话串起来**：上一章教你"**发动机怎么转**"；本章教你"**转完卷子上每一栏怎么读、多轮怎么接着聊、被中断了怎么恢复、被当工具调起时怎么拿到调用元数据**"。

---

*结果篇（两种结果类型 / 选栏总表 / `final_output` / 四兄弟辨析 / `new_items` 项目类型 / 继续与恢复对话 / `agent_tool_invocation` / 流式生命周期与诊断 / `results_demo.py`）**已完成**。*

*后续待学：models。*