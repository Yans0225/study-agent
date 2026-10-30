# OpenAI Agent 学习笔记 11（流式传输篇）

> 本文档是「系统性学习 Agent」第十一阶段的整理，聚焦 **流式传输（Streaming）**——也就是 **"怎么让 Agent 边跑边把进度/半成品吐给用户，以及这中间'碎片'和'钟声'到底谁是谁"**。
> 来源：OpenAI Agents SDK 官方中文文档「流式传输」（streaming.md）；另核对了 SDK 源码 `agents/stream_events.py`（三档事件 dataclass）、`agents/result.py`（`RunResultStreaming` / `stream_events()`）、`agents/run.py`（`Runner.run_streamed` / `AgentRunner.run_streamed`）、`agents/run_internal/run_loop.py`（raw 事件入队 / `stream_step_items_to_queue` / 字段直写 / sentinel）、`agents/run_internal/streaming.py`（RunItem → 事件名映射）、`agents/models/chatcmpl_stream_handler.py`（chunk → Responses 形状事件）。
> 学习日期：2026-10-29（理论学习 + 编写并通过编译 `streaming_demo.py`，并做「配置集中化」改造）
> 衔接：Agent 静态配置 vs Runner 运行时（笔记 02）、handoff（笔记 04）、tracing（笔记 06）、session（笔记 07）、Agent Loop / RunConfig（笔记 08）、`RunResult` 行程单（笔记 09）、模型接线（笔记 10）。
> **状态：✅ 本章已学完（2026-10-29 收尾）**（已学：三档事件 / chunk·raw·item 解剖 / "一个条目一个事件" / 交替顺序 / 同步启动 vs 异步消费 / 前后台三通道 / 中断·取消·审批恢复 / `await` vs `async` / `streaming_demo.py`）。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [这一章讲什么：一句话](#二这一章讲什么一句话)
3. [⭐ 三档事件](#三-三档事件)
4. [⭐ 解剖：chunk、raw、item 到底各是什么](#四-解剖chunkrawitem-到底各是什么)
5. [⭐ 事件顺序：一轮 raw、一轮 item，交替出现](#五-事件顺序一轮-raw一轮-item交替出现)
6. [消费模式：是"固定流程"，靠"过滤"不靠"选择"](#六消费模式是固定流程靠过滤不靠选择)
7. [⭐ 同步启动 vs 异步消费（附带 `await` / `async` 的区别）](#七-同步启动-vs-异步消费附带-await--async-的区别)
8. [⭐ `RunResultStreaming` 和后台 `asyncio.create_task` 怎么打交道](#八-runresultstreaming-和后台-asynciocreate_task-怎么打交道)
9. [中断、取消、审批恢复](#九-中断取消审批恢复)
10. [实际应用：`streaming_demo.py` + 配置集中化](#十实际应用streaming_demopy--配置集中化)
11. [易混概念辨析（汇总表）](#十一易混概念辨析汇总表)
12. [问答记录（Q&A）](#十二问答记录qa)
13. [和前面章节的对照](#十三和前面章节的对照)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 流式传输 streaming | https://openai.github.io/openai-agents-python/zh/streaming/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/streaming.md |

> ⚠️ 获取记录：本机直连 GitHub 不通，经镜像 `https://ghproxy.net/https://raw.githubusercontent.com/...` 取到全文（同 04–10 的通道），已缓存到 `E:\agent_study\docs_cache\streaming.md`。
> 源码核对：`stream_events.py:11/24/52`（三档事件 dataclass）、`result.py:660`（`RunResultStreaming`）、`result.py:1005`（`stream_events()`）、`result.py:1081`（`item = await self._event_queue.get()`）、`result.py:1089`（`QueueCompleteSentinel` → break）、`result.py:1119`（`await self._await_task_safely(self.run_loop_task)`）、`result.py:941/968`（`cancel()` / `_cancel_mode`）、`run.py:489`（`Runner.run_streamed` 公开薄封装）、`run.py:2632`（`AgentRunner.run_streamed` 实现）、`run.py:2922`（`asyncio.create_task(run_loop)`）、`run_loop.py:2652`（raw 入队）、`run_loop.py:728/732/734`（`stream_step_items_to_queue` / `is_complete=True` / sentinel）、`run_loop.py:367/391/405/1195/1637`（字段直写 + `AgentUpdatedStreamEvent`）、`streaming.py:35-55`（RunItem→事件名映射）。

---

## 二、这一章讲什么：一句话

> **Agent 在跑的时候，怎么把"进度"和"半成品"实时吐给用户；而这中间"逐字碎片（raw）"和"整段钟声（item）"各是什么、谁先谁后、代码该怎么接。**

- 前面十章讲的是"**怎么把活干完、干完长什么样**"；
- 本章讲的是"**干的过程中，怎么边干边播**"。

**四个核心问题**：
1. 流式里有哪几种"事件"？分别是多大颗粒度？
2. 一个 chunk / 一个 raw / 一个 item 到底什么关系？
3. 有工具、有交接的时候，这些事件按什么顺序冒出来？
4. `run_streamed()` 明明是同步的，为什么 `stream_events()` 要异步消费？

---

## 三、⭐ 三档事件

`StreamEvent` 是一个 **union**，共 **三档**（`stream_events.py`）：

| # | 类 | `event.type` 值 | 颗粒度 | 用来干嘛 |
|---|---|---|---|---|
| ① | **`RawResponsesStreamEvent`** | `"raw_response_event"` | **最细**（网络层碎片，逐字） | 把 LLM 的**原始事件直通**给你；想"字一生成就推给用户"就用它 |
| ② | **`RunItemStreamEvent`** | `"run_item_stream_event"` | **中**（一整条条目） | 条目**完全生成后**通知你（"消息好了""工具跑完了"）；按**语义**推进度，而不是按 token |
| ③ | **`AgentUpdatedStreamEvent`** | `"agent_updated_stream_event"` | **粗**（换人） | 当前 agent 变化时通知（例如 **handoff** 交接） |

```python
# agents/stream_events.py
class RawResponsesStreamEvent:
    data: TResponseStreamEvent              # 就是 LLM 吐的原始事件
    type: Literal["raw_response_event"] = "raw_response_event"

class RunItemStreamEvent:
    name: Literal[...11 个...]              # 语义名，见第六节
    item: RunItem                           # 完整的运行项
    type: Literal["run_item_stream_event"] = "run_item_stream_event"

class AgentUpdatedStreamEvent:
    new_agent: Agent
    type: Literal["agent_updated_stream_event"] = "agent_updated_stream_event"
```

---

## 四、⭐ 解剖：chunk、raw、item 到底各是什么

### 4.1 chunk ≠ token；chunk 是"网络层碎片"

- **chunk** = LLM API 在**网络层**一块一块推给你的**流式分片**。本用户走 Chat Completions 协议，所以拿到的是 `ChatCompletionChunk`，内容在 `.choices[0].delta.content`。
- 几个细节：
  - **第一个 chunk 通常只有 `role`**（`delta.role="assistant"`），没有文字；
  - **最后一个 chunk 带 `finish_reason`**（告诉你"话说完了"）；
  - **一个 chunk 不等于一个 token**——有的 chunk 带好几个字，有的带零个字（只是控制信号）。

### 4.2 raw event = SDK 把 chunk 原样包一层

- **raw event** 就是 `RawResponsesStreamEvent`，`data` 字段里装着"**未经处理、直接从 LLM 来**"的那条事件。"**raw = 生的、没加工**"。
- 名字叫 `RawResponses` 是因为 SDK 统一把事件转成 **Responses API 的形状**——但本用户走的是 Chat Completions 协议，所以中间有个**翻译官**负责把 Chat Completions 的 chunk 转成 Responses 形状的事件。

### 4.3 ⭐ 1 个 chunk → 0..N 个 Responses 形状事件

`chatcmpl_stream_handler.py` 里 `async for chunk in stream:`（L646）下面跟着**一串 `yield`**。也就是说：

> **一个 chunk 翻译后可能产出 0 个、1 个、或好几个 Responses 形状事件。**

例如一个带文字的 chunk，可能同时产出：
- `response.output_item.added`（"第一个输出项开始了"）
- `response.content_part.added`（"文本部分开始了"）
- `response.output_text.delta`（**真正的文字碎片**，`delta=delta.content`，L921）

所以你代码里看到的 raw 事件个数，**和你直接数 chunk 个数，对不上**。

### 4.4 两档的"手感"完全不同

| | **raw（🟦）** | **item（🟨）** |
|---|---|---|
| 形态 | **连续流**（一次几个字，几十上百次） | **离散钟声**（一整条憋完了"叮"一声） |
| 触发 | 模型**每吐一点**就发 | 条目**完全生成后**才发**一次** |
| 内容 | 只是**碎片**（要自己拼） | **完整的对象**（`item` 字段直接给你整条） |
| 给谁 | **人眼**（打字机式显示、低延迟） | **代码**（按语义做 UI / 逻辑判断） |

> **没有"段落"这一档。** 想按段落推送？只能**自己把 raw delta 攒起来**、到某个标点/换行为止再当一段——SDK **不提供**中间颗粒度。

### 4.5 ⭐ `final_output` 是"货物"，不是"完成信号"

很多人会问：raw delta 都已经把字吐完了，`final_output` 岂不是多余的？

> **不是。** 二者面向的东西根本不同：

- **delta 流**：给**人眼**看的**半成品**——连续、求快、未必是最终版本（交接/多轮下可能吐出"过程性的话"）。
- **`final_output`**：给**代码**用的**最终值**——设了 `output_type` 就是**解析好的对象**；多轮/handoff 下是**最终被选中的那条 message**。
- 关系：**delta 流 ⊇ `final_output`**（是超集）。纯文本单轮时两者**相等**；一旦结构化输出，delta 只是 JSON 碎片，`final_output` 才是解析后的对象 → **分家**。

真正的"**完成信号**"是别的东西：**`is_complete` 翻 True** + 队列里的 **哨兵**（`QueueCompleteSentinel`）。`final_output` 是"**送达的货物**"，不是"**送货完毕的铃**"。

### 4.6 ⭐ "一个条目一个事件" ≠ "一个 turn 一个事件"

- **一个 turn = 一次 LLM 调用**（笔记 08）。
- 一次 LLM 调用可以产出 **0..N 个条目**（消息、工具调用、推理项、交接调用…）→ **0..N 声钟**。
- 源码里 `stream_step_items_to_queue(model_items, streamed_result._event_queue)`（`run_loop.py:728`）收的是一个 **list**——**明摆着一次可以塞多条**。
- **简单问答（无工具、无交接）→ 恰好 1 声钟**：`message_output_created`，一次把整段答案交出来（但在此之前，🟦 raw delta 已经一个字一个字冒了几十次）。

---

## 五、⭐ 事件顺序：一轮 raw、一轮 item，交替出现

**不是**"所有 raw 在前、所有 item 在后"。真相是**按 turn 交替**：

```
        ┌── turn 1（第 1 次 LLM 调用）────────────────────┐
raw:    response.created → … → response.output_text.delta … → response.completed   （一波 raw）
item:   message_output_created / tool_called                                       （几声响）
        └───────────────────────── 工具执行 / 交接 ──────┘
        ┌── turn 2（第 2 次 LLM 调用）────────────────────┐
raw:    response.created → … → response.output_text.delta … → response.completed   （又一波 raw）
item:   tool_output / message_output_created                                        （又一串铃）
        └───────────────────────────────────────────────┘
```

**全局模式**：

> **raw₁ → item₁ → raw₂ → item₂ → … → rawₙ → itemₙ → （哨兵）结束**

**记住三条**：

1. **raw 波数 == turn 数 == LLM 调用次数**。每次 LLM 调用都会开自己的一波 raw（从 `response.created` 到 `response.completed`）。
2. **有工具/交接时，后面照样有 raw**——不是只有第一波。
3. ⚠️ **工具调用的那一轮，raw 流的是"参数碎片"不是"文字碎片"**：`ResponseFunctionCallArgumentsDeltaEvent`（`chatcmpl_stream_handler.py:1128`）。所以如果你**只处理 `ResponseTextDeltaEvent`**，那一轮你会**一个字都看不到**——这不是 bug，是那一轮本来就没在"说人话"，它在"填参数"。

---

## 六、消费模式：是"固定流程"，靠"过滤"不靠"选择"

**常见误区**：以为 raw 和 item 是"二选一 / 可混搭的两种模式"。

**真相**：

- **两者都进同一个队列** `_event_queue`（raw 在 `run_loop.py:2652` 塞，item 在 `run_loop.py:728` 塞）。
- **没有开关能单独关掉 raw**（两种事件同时存在，顺序由 SDK 固定）。
- **消费者做的事是"过滤"**：`if / elif / continue` 挑出你要的、其余 `continue` 丢掉。

```python
async for event in result.stream_events():
    if event.type == "raw_response_event":
        continue                                  # 丢掉 raw
    elif event.type == "run_item_stream_event":
        ...                                        # 只要钟声
```

> **所以不是"选择/混用两种模式"，而是"固定的一路事件流，你自己筛"。**

### 6.1 十一个固定的 item 事件名（`stream_events.py:29`）

`RunItemStreamEvent.name` 只能取这 11 个固定值：

`message_output_created`、`handoff_requested`、`handoff_occured`、`tool_called`、`tool_search_called`、`tool_search_output_created`、`tool_output`、`reasoning_item_created`、`mcp_approval_requested`、`mcp_approval_response`、`mcp_list_tools`。

来源映射（`run_internal/streaming.py:35-55`，RunItem 类型 → 事件名）：

| RunItem 类型 | 事件名 |
|---|---|
| `MessageOutputItem` | `message_output_created` |
| `HandoffCallItem` | `handoff_requested` |
| `HandoffOutputItem` | `handoff_occured` |
| `ToolCallItem` | `tool_called` |
| `ToolCallOutputItem` | `tool_output` |
| `ReasoningItem` | `reasoning_item_created` |
| `ToolSearchCallItem` / `ToolSearchOutputItem` | `tool_search_called` / `tool_search_output_created` |
| `MCPApprovalRequestItem` / `MCPApprovalResponseItem` / `MCPListToolsItem` | `mcp_approval_requested` / `mcp_approval_response` / `mcp_list_tools` |
| `ToolApprovalItem` / `CompactionItem` | **不发事件**（映射为 `None`） |

**两个坑**：
- **`handoff_occured` 是故意拼错的**（少一个 `r`），为保持向后兼容——**照着文档写，别改**。
- **交接只发 `handoff_requested`，不发 `tool_called`**（虽然 handoff 内部也像个工具调用）。同一轮里**普通的函数工具调用**仍发 `tool_called`。

---

## 七、⭐ 同步启动 vs 异步消费（附带 `await` / `async` 的区别）

### 7.1 现象

```python
result = Runner.run_streamed(agent, input="...")   # ← 同步！没有 await
async for event in result.stream_events():         # ← 这里是异步
    ...
```

**为什么一个同步、一个异步？** 原则一句话：

> **要等的才 `async`。**

- `Runner.run_streamed()` 干的事只有 3 件、**每件都是瞬间完成**：
  1. 造 `RunResultStreaming` 对象（`run.py:2810`）；
  2. `asyncio.create_task(run_loop)` —— **`create_task` 本身是同步的**（只是"把任务排进事件循环"，不等人跑）；
  3. `return streamed_result`（`run.py:2932`）。
  → **全程没有 `await`，所以它是同步函数**。
- `stream_events()` 里有 **`item = await self._event_queue.get()`**（`result.py:1081`）——**它得"等"下一个事件到达**。若写成同步，就会**阻塞整个线程**→ 后台 `run_loop` 永远没机会跑→ **死锁**。所以**必须 `async`**。这是个 **async generator**。
- 补一句：`stream_events()` **这个调用本身是同步的**（它只是"返回一个异步生成器"），真正"驱动"它的是 `async for`。

**对比 `run` 家族**：

| 入口 | 性质 |
|---|---|
| `Runner.run(...)` | `async`（要等整趟跑完） |
| `Runner.run_sync(...)` | **同步包装**（内部替你 `asyncio.run`） |
| `Runner.run_streamed(...)` | **同步启动**（点火即走，不等） |

### 7.2 顺带：`async` 和 `await` 到底啥区别（用户最后一问）

> **`async` = 挂一块牌子（声明）："我是异步的，得在 event loop 里跑。"**
> **`await` = 按一下按钮（动作）："我在这儿等一下，先让位给别人。"**

| | **`async`** | **`await`** |
|---|---|---|
| 是什么 | **修饰符**（声明/资格） | **运算符**（执行/动作） |
| 出现在哪 | `def` / `with` / `for` **前面** | 表达式里、语句前 |
| 作用 | 圈出**一块场地**："这里可以放 await" | 在场地里**真正停一下**、让出控制权 |
| 执行效果 | **调用 `async def` 不跑函数体**，只返回一个协程对象 | 真正暂停当前协程，交还给 event loop |
| 能否单独用 | 能（`async def` 里一个 `await` 都没有也合法） | **不能**（没有 async 场地就 `SyntaxError`） |

**关系是"场地 vs 动作"**：`async` 给你一块**能放 `await` 的场地**；`await` 是**场地里唯一能干的事**。**单向依赖：`await` 依赖 `async`，`async` 不依赖 `await`。**

**三条语法规则**：

1. **调用 `async def` 函数不会执行它**——只返回一个协程对象：
   ```python
   async def greet(): print("hi")
   greet()          # ❌ 什么都不打印（得到一个 <coroutine object>）
   await greet()    # ✅ 才真跑
   ```
   （Python 警告 `coroutine ... was never awaited` 就是这个来源。）
2. **`await` 只能出现在 `async` 函数里**，否则 `SyntaxError: 'await' outside async function`。
3. **只能 `await` "可等待对象"**（awaitable）：协程 / Task / Future；`await 42` → `TypeError`。

**`async` 也能修饰别的**（作用都是"这里有个需要等待的操作"）：

```python
async for event in result.stream_events():   # async for：迭代器本身要 await（异步生成器）
async with some_async_resource:              # async with：进入/退出时要 await
```

**回到 SDK**：`async def run_loop()`（声明"这是协程函数"）里的 `await start_streaming(...)`（等它）；`async def stream_events()`（声明"可能暂停"）里的 `await self._event_queue.get()`（**真正的暂停点**）。**哪里标 `async`、哪里放 `await`，完全由"这里要不要等"决定。**

---

## 八、⭐ `RunResultStreaming` 和后台 `asyncio.create_task` 怎么打交道

`run_streamed` 启动后，**后台有个 `run_loop` 协程在跑 Agent Loop**，前台是**你 `async for` 的消费者**。两者靠**三条通道**通信（**同一个 `RunResultStreaming` 对象**，通过闭包捕获 `streamed_result=streamed_result` 共享）：

| 通道 | 方向 | 内容 |
|---|---|---|
| **A. 事件队列** `_event_queue` | 后台 **put** → 前台 **await get** | raw 事件、item 事件、`AgentUpdatedStreamEvent`、**哨兵** |
| **B. 字段直写**（共享 handle 上的属性） | 后台**直接改** | `current_turn`、`new_items`、`raw_responses`、`interruptions`、`current_agent`、`final_output`、`is_complete` |
| **C. 反向通道** `_cancel_mode` | 前台 **`cancel()`** → 后台**读** | 你想中途叫停（见第九节） |

**几句话**：

- **A 是"实时推送"**：所有"要边跑边看"的东西都走队列。
- **B 是"最终账目"**：`run_loop` 跑到关键节点就顺手往共享对象上写（`run_loop.py:367/391/405/729/732`）——所以**你随时读 `result.new_items` / `result.is_complete` 都是最新的**。
- 结束时后台**往队列塞一个哨兵** `QueueCompleteSentinel`（`run_loop.py:734`），前台 `await get()` 拿到它 → **break 退出循环**（`result.py:1089`），并在 `finally` 里 **`await self.run_loop_task`（join 后台任务）**（`result.py:1119`），确保后处理（session 持久化等）真的收尾。
- **无需加锁**：都在**同一个事件循环**里（单线程），不存在真并发写。

> **一句话**：`RunResultStreaming` 是两边**共用的那块白板**——后台往上写（通道 A 贴便利贴、通道 B 记台账、通道 C 收你的叫停），前台只管读。

---

## 九、中断、取消、审批恢复

### 9.1 必须把流消费到底

> 官方强调：**持续 `stream_events()` 直到迭代器结束，流式运行才算完成**。

因为**最后一个可见 token 到达之后**，SDK 可能还在**持久化会话、确定最终审批状态、压缩历史**。你在 `async for` 里 `break` 早了，这些后处理就断了。循环真正退出时，`result.is_complete` 才反映**最终运行状态**。

### 9.2 审批（人在回路）

流式与"因工具审批而暂停"兼容。如果工具需要审批：

```python
result = Runner.run_streamed(agent, "Delete temp files if not needed.")
async for _ in result.stream_events():
    pass                                   # 先消费完

if result.interruptions:                   # 有待审批的中断
    state = result.to_state()              # 转成 RunState
    for interruption in result.interruptions:
        state.approve(interruption)        # 或 state.reject(...)
    result = Runner.run_streamed(agent, state)   # ← 从 state 续跑
    async for _ in result.stream_events():
        pass
```

（呼应笔记 09：`interruptions` → `to_state()` → `approve/reject` → 续跑。）

### 9.3 取消

`result.cancel()`（`result.py:941`）：

| 模式 | 行为 |
|---|---|
| `cancel()`（默认 `mode="immediate"`） | **立刻停** |
| `cancel(mode="after_turn")` | **让当前这一 turn 正常跑完**再停 |

### 9.4 手动续接

- 用 `result.to_input_list(mode="normalized")` + **`result.last_agent`** 手动接着跑。
- 若 `cancel(mode="after_turn")` 在**某个工具轮次之后**停住，应**用规范化输入重跑 `last_agent`**、继续"尚未完成的这个用户轮次"，**而不是**马上追加一条新用户消息。
- 若在恢复前**又来了新用户输入**：`result.to_state()` → `state.add_input(...)` → 从 state 恢复（运行器会在下一次模型调用前**立即接纳**暂存的输入）。
- 若因**工具审批**停下：**别当成新轮次**——先消费完流、查 `result.interruptions`，然后从 `to_state()` 恢复。

---

## 十、实际应用：`streaming_demo.py` + 配置集中化

**文件**：`E:\agent_study\code\streaming_demo.py`（配 `run_streaming.bat`）。三个演示：

| 演示 | 看什么 | 对应本笔记 |
|---|---|---|
| ① 纯问答 | 🟦 raw delta **一个字一个字**实时冒（打字机）+ 🟨 item 钟声**单独占一行**；跑完打统计（raw 个数 vs item 个数 vs `current_turn`）；验证 `"".join(deltas) == final_output` | 四、五 |
| ② 加一个 `@function_tool` | 钟声**变多**（`tool_called` + `tool_output` + `message_output_created`）、`turn` 变 **2** → 印证**"一个条目一个事件"≠"一个 turn 一个事件"** | 四、五 |
| ③ `final_output` vs 拼接 delta | 纯文本下**相等**；注释点破：一旦设 `output_type`，delta 只是 JSON 碎片、成品对象**只在 `final_output`** | 四 |

一个辅助点：raw 只处理 `ResponseTextDeltaEvent`（文字碎片），所以**演示二里工具那一轮不会看到 raw**（那轮流的是参数碎片）——正好实物印证第五节第 3 条。

### 10.1 ⭐ 配置集中化（2026-10-29 改造）

以前每个 `run_*.bat` 里都写了一遍三条环境变量，用户嫌"每次跑都要重设 key"。改造后：

- **`config.bat` = 唯一需要维护的配置文件**（GBK + CRLF），三行：
  ```bat
  set "MODEL_BASE_URL=https://ai-gw.united-imaging.com/v1"
  set "MODEL_API_KEY=YOUR_KEY"
  set "MODEL_NAME=deepseek-v4.1-flash"
  ```
- 所有 `run_*.bat`（`run` / `run_session` / `run_runner` / `run_results` / `run_streaming`）改为：
  ```bat
  @echo off
  call "%~dp0config.bat"
  py -3.13 "E:\agent_study\code\xxx_demo.py"
  ```
- 新增 `.gitignore`：排除 `config.bat` / `.env` / `*.key` / `__pycache__/` / `*.pyc`。

> ⚠️ **坑：`.bat` 必须存成 GBK + CRLF**。UTF-8 的中文注释在 cmd（GBK 码页）下会**乱码并把 `set` 行拆断**，症状是 `'MODEL_API_KEY' is not recognized`。用 Python `io.open(path, "w", encoding="gbk", newline="")` 写入即可。
> ⚠️ **验证别 echo key 明文**（会被安全分类器拦，且明文泄进日志）——用 `if defined MODEL_API_KEY (echo KEY_IS_SET=yes)`。另注意 `cmd /c "call config.bat && echo %VAR%"` 会把变量按**解析时机**展开成字面量，**观察不到**值，得用临时 .bat 或延迟展开。

---

## 十一、易混概念辨析（汇总表）

### 11.1 三档事件

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **`RawResponsesStreamEvent`** | **`RunItemStreamEvent`** | 前者=**连续碎片**（逐字，要自己拼）；后者=**离散钟声**（一整条，一次一发） |
| **`RunItemStreamEvent`** | **`AgentUpdatedStreamEvent`** | 前者=**条目好了**（消息/工具）；后者=**换人了**（handoff） |
| **raw 粒度** | **item 粒度** | ✅ "连续流 vs 离散敲钟"；❌ **没有"段落"这一中间档**（想分段得自己攒 delta） |

### 11.2 chunk / raw / item 三层

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **chunk** | **token** | 前者=**网络层分片**（一个 chunk 可含 0~N 个字，非 1 字）；后者=模型的最小词元 |
| **chunk** | **raw event** | 前者=LLM 吐的原片；后者=SDK **包一层**后的 `RawResponsesStreamEvent`。**1 chunk → 0..N raw 事件**（翻译官展开） |
| **raw event** | **item event** | 前者=半成品碎片；后者=成品整条 |

### 11.3 完成信号 / 货物

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **delta 流** | **`final_output`** | 前者=给**人眼**的半成品（连续、求快）；后者=给**代码**的最终值（可带 `output_type`）。**delta ⊇ final_output** |
| **`final_output`** | **`is_complete` / 哨兵** | 前者=**送达的货物**；后者=**送货完毕的铃**（完成信号） |

### 11.4 计数 / 顺序

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **"一个 turn 一个事件"** | **"一个条目一个事件"** | ❌前者错；✅后者对。**一个 turn 可产 0..N 条目 → 0..N 声钟** |
| **简单问答** | **一次 turn** | 前者=**恰好 1 声钟**（`message_output_created`）；一个 turn 却可能 N 声钟 |
| **raw 波数** | **turn 数** | **相等**（每次 LLM 调用一波 raw）。全局顺序 raw₁→item₁→raw₂→item₂ |
| **工具轮次的 raw** | **文字轮次的 raw** | 前者流**参数碎片**（`function_call_arguments` delta）；后者流**文字碎片**（`output_text.delta`） |

### 11.5 同步 / 异步

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **`async`** | **`await`** | 前者=**声明/场地**（能否暂停）；后者=**动作/暂停点**（真的让位） |
| **`Runner.run_streamed()`** | **`Runner.run()`** | 前者=**同步启动**（点火即走，不等）；后者=`async`（等整趟跑完） |
| **`run_streamed()` 同步** | **`stream_events()` 异步** | 前者 3 件事都瞬间完成、无 `await`；后者有 `await queue.get()`，必须 async（否则死锁） |
| **`stream_events()` 调用** | **`async for`** | 前者=**同步**（返回异步生成器）；后者=**驱动**它 |

### 11.6 前后台三通道

| 通道 | 方向 | 内容 |
|---|---|---|
| **A 事件队列** | 后台 put → 前台 await get | raw / item / agent-updated / 哨兵 |
| **B 字段直写** | 后台直接改共享对象 | `current_turn` / `new_items` / `final_output` / `is_complete` … |
| **C `_cancel_mode`** | 前台 cancel → 后台读 | 中途叫停 |

---

## 十二、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | "每来一个 chunk 塞一个 raw 事件，chunk 和 raw 都是啥？" | **chunk**=LLM 网络层流式分片（本站是 `ChatCompletionChunk`，内容在 `.choices[0].delta.content`；首个只带 role、末个带 `finish_reason`；**一个 chunk ≠ 一个 token**）；**raw**=`RawResponsesStreamEvent`，是 SDK 对"**未经加工、直接从 LLM 来**"事件的**原样包装**。⚠️ 还夹了一层**翻译官**：Chat Completions 的 chunk → Responses 形状事件，**1 chunk 可产出 0..N 个 raw 事件**。 |
| 2 | "前台在一个字一个字输出，最终只是给我一个'完成/回复结束'的信号？" | **半对**。raw delta 流**不是信号**，是**给眼睛看的半成品**（连续、求快、可能非最终版）。真正的**完成信号**= `is_complete` 翻 True + 队列**哨兵**；`final_output` 则是**给代码的最终值**（可带 `output_type` 的解析对象）。三者分工：delta=眼、final_output=货、is_complete/sentinel=铃。 |
| 3 | "`RunItemStreamEvent` 会输出成什么样？一段一段吗？" | ❌ **不是"段落"**。它是一声**离散的钟**——条目**完全生成后响一次**，`item` 字段直接给你**完整对象**。**SDK 没有"段落"这一中间颗粒度**；想按段推，只能自己把 raw delta 攒到标点/换行。 |
| 4 | "一次 turn 只有一次 `RunItemStreamEvent`？不调工具/不交接就直接一次把问答结果发出来？" | **后半句对，规则要拧紧**。正确规则是 **"一个条目一个事件"**（`stream_step_items_to_queue` 收的是 **list**）；一次 turn（=一次 LLM 调用）可产 **0..N 条目 → 0..N 声钟**。**简单问答**（无工具无交接）恰好 **1 声** `message_output_created`，整段答案。 |
| 5 | "raw 和 item 是选其一用、可混用、还是固定流程？" | **固定流程，靠过滤**。两者**都进同一个 `_event_queue`**，**没有开关关掉 raw**，顺序 SDK 固定；消费者用 `if/elif + continue` **筛**出要的，**不是"选择/混搭两种模式"**。 |
| 6 | "调工具/交接后还有没有 raw？是所有 raw 在前 item 在后，还是一轮 raw、一轮 item？" | ✅ **一轮 raw、一轮 item，交替**。全局：raw₁→item₁→raw₂→item₂…。**有工具/交接照样有后续 raw**。**raw 波数 == turn 数 == LLM 调用次数**。⚠️ **工具轮**流的是**参数碎片**（`function_call_arguments` delta），只处理 `ResponseTextDeltaEvent` 的话那一轮**一个字都看不到**。 |
| 7 | "`RunResultStreaming` 和后台 `asyncio.create_task` 怎么打交道？" | 靠**三条通道**、**共享同一个 handle**：**A 事件队列**（后台 put / 前台 await get）、**B 字段直写**（后台改 `new_items`/`final_output`/`is_complete`…）、**C 反向 `_cancel_mode`**（前台 cancel → 后台读）。结束时后台塞**哨兵**，前台 break 并 **join `run_loop_task`**。单线程事件循环，**无需加锁**。 |
| 8 | "为啥 `run_streamed()` 是同步的、`async for stream_events()` 是异步的？" | 原则：**要等的才 async**。`run_streamed` 只干 3 件**瞬间完成**的事（造对象 / `create_task`（同步调度）/ return），**全程无 `await`** → 同步。`stream_events` 有 **`await queue.get()`**，**必须 async**（同步版会阻塞线程→后台永不跑→**死锁**）。`stream_events()` **调用本身同步**（返回异步生成器），**`async for` 驱动**它。 |
| 9 | "`await` 和 `async` 有啥区别？" | **`async`=声明（挂品牌子，圈出场地）**；**`await`=动作（在场地里按暂停、让出控制权）**。方向单向：**`await` 依赖 `async`，`async` 不依赖 `await`**。规则：①调用 `async def` **不执行**、只返回协程对象；②`await` **只能在 async 里**（否则 `SyntaxError`）；③只能 `await` awaitable（协程/Task/Future）。`async` 还修饰 `for`/`with`。回到 SDK：`async def run_loop()` 是"声明可暂停"，`await start_streaming(...)` 是"实际暂停点"。 |

---

## 十三、和前面章节的对照

| | 前十章（零件 / 发动机 / 行程单 / 大脑） | 本章（边跑边播） |
|---|---|---|
| 角色 | 造零件 + 装上 `Runner` 转 + 读卷子 + 接大脑 | **盯着它转，把过程实时播出去** |
| 关键词 | `Agent`、工具、`handoff()`、`tripwire`、`span`、`session`、`Runner.run`、Agent Loop、`RunResult`、`ModelSettings` | `run_streamed()`、`RunResultStreaming`、`stream_events()`、`RawResponsesStreamEvent`、`RunItemStreamEvent`、`AgentUpdatedStreamEvent`、`current_turn`、`cancel()`、`is_complete` |
| Agent/Runner（笔记 02） | Agent=静态"谁是"，Runner=运行时引擎 | 流式=`Runner` 循环的**实时字幕** |
| running_agents（笔记 08） | Agent Loop 的 `while True` + 四个 `NextStep*` 出口 | 每次 LLM 调用 = **一波 raw**；每个 `NextStep` 产出的条目 = **一串 item 钟声** |
| results（笔记 09） | `RunResult` 是"跑完的交卷" | `RunResultStreaming` 是"**边跑边看的直播**"（多 `stream_events`/`current_agent`/`is_complete`/`cancel`）；`interruptions`→`to_state()` 续跑同源 |
| handoff（笔记 04） | 平级交接 | 交接时发 `AgentUpdatedStreamEvent` + `handoff_requested`（**不发 `tool_called`**） |
| models（笔记 10） | Chat Completions vs Responses 两种形状 | **流式翻译官**把 `ChatCompletionChunk` 转成 Responses 形状事件（1 chunk→0..N） |

> **一句话串起来**：前面十章教你"**怎么把活干完、干完长什么样**"；本章教你"**干的过程中怎么边干边播**——碎片（raw）给眼睛、钟声（item）给代码、换人（agent-updated）通知你，而这一切靠一条队列、一块共享白板、一个同步点火 + 异步消费的机制串起来"。

---

*流式传输篇（三档事件 / chunk·raw·item 解剖 / "一个条目一个事件" / 交替顺序 / 消费靠过滤 / 同步启动 vs 异步消费 / `await` vs `async` / 前后台三通道 / 中断·取消·审批恢复 / `streaming_demo.py` + 配置集中化）**已完成**。*

*后续待学：agent orchestration（multi_agent，多智能体编排）→ human-in-the-loop（人在回路）→ sessions 进阶 → usage（用量）→ MCP → testing（测试）。*