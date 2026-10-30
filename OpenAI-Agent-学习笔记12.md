# OpenAI Agent 学习笔记 12（智能体编排篇 multi_agent）

> 本文档是「系统性学习 Agent」第十二阶段的整理，聚焦 **智能体编排（Agent Orchestration）**——也就是 **"多个 agent 之间，谁先跑、谁后跑、谁决定下一步？"**
> 来源：OpenAI Agents SDK 官方中文文档「智能体编排」（multi_agent.md）；另核对了 SDK 源码 `agents/agent.py`（`as_tool()` / `tool_use_behavior` / 嵌套 run / usage 共享）、`agents/run_internal/turn_resolution.py`（工具跑完谁收尾）、`agents/run.py`（`run` / `run_sync` / SDK 内部并发）、`agents/run_internal/guardrails.py`（多护栏并行）、`agents/run_config.py`（`DEFAULT_MAX_TURNS`）。
> 学习日期：2026-10-29（2026-10-30 整理成笔记）
> 衔接：Agent 静态配置 vs Runner 运行时（笔记 02）、handoff（笔记 04）、guardrails 并行（笔记 05）、RunConfig / Agent Loop（笔记 08）、`RunResult` 行程单里的 `interruptions`（笔记 09）、模型接线（笔记 10）。
> **状态：✅ 本章已学完（2026-10-30 收尾）**（已学：编排=控制流 / LLM 决策 vs 代码决策 / as_tool vs handoff / `tool_use_be5 条 LLM 编排策略khavior` 四种 / 四种代码编排 / 五条 LLM 策略 / asyncio 深讲 / 两个可运行 demo）。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [这一章讲什么：一句话](#二这一章讲什么一句话)
3. [⭐ 编排 = 控制流](#三-编排--控制流)
4. [⭐ 两条路线：LLM 决策 vs 代码决策](#四-两条路线llm-决策-vs-代码决策)
5. [⭐ 两种 SDK 模式：agents as tools vs handoffs](#五-两种-sdk-模式agents-as-tools-vs-handoffs)
6. [⭐ `as_tool()` 全参数与三个细节](#六-as_tool-全参数与三个细节)
7. [⭐ `tool_use_behavior`：Agent 级字段，不是工具参数](#七-tool_use_behavioragent-级字段不是工具参数)
8. [⭐ 四种代码编排模式](#八-四种代码编排模式)
9. [五条 LLM 编排策略](#九五条-llm-编排策略)
10. [⭐ asyncio 深讲：`await` 到底在等谁](#十-asyncio-深讲await-到底在等谁)
11. [实际应用：两个可运行 demo](#十一实际应用两个可运行-demo)
12. [易混概念辨析（汇总表）](#十二易混概念辨析汇总表)
13. [问答记录（Q&A）](#十三问答记录qa)
14. [和前面章节的对照](#十四和前面章节的对照)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 智能体编排 multi_agent | https://openai.github.io/openai-agents-python/zh/multi_agent/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/multi_agent.md |

> ⚠️ 获取记录：同 04–11，GitHub 直连不通，经镜像取到全文，已缓存到 `E:\agent_study\docs_cache\multi_agent.md`。
> 源码核对（SDK 0.24.0）：
> - `agent.py:606`（`as_tool()` 完整签名）、`:629`（返回 `FunctionTool`）、`:630-636`（与 handoff 的两条本质区别）、`:764`（`ToolContext(usage=context.usage)`，usage 共享）、`:1044`（嵌套 `Runner.run`）、`:1074`（无条件 `await custom_output_extractor(...)`）、`:1076-1093`（默认提取顺序）。
> - `agent.py:396`（`tool_use_behavior` 字段定义）、`:414`（托管工具永远由 LLM 处理）。
> - `run_internal/turn_resolution.py:246`（每轮问一次 `check_for_final_output_from_tools`）、`:776-808`（四分支实现）。
> - `run.py:281`（`async def run`）、`:386`（`def run_sync`）、`:1800-1841`（输入护栏与首次模型调用、多护栏的 `asyncio.create_task` / `gather`）。
> - `run_internal/guardrails.py:67-84`（每个护栏一个 task + `gather(..., return_exceptions=True)`）。
> - `run_config.py:45`（`DEFAULT_MAX_TURNS = 10`）。

---

## 二、这一章讲什么：一句话

> **一个应用里有多个 agent 时，「谁先跑、谁后跑、谁决定下一步」这个问题，可以由 LLM 回答，也可以由 Python 回答。**

- 前面十一章讲的是"**一个 agent 怎么跑**"（配置、工具、交接、护栏、记忆、流式……）；
- 本章讲的是"**多个 agent 怎么组织成一个流程**"。

**四个核心问题**：
1. 编排到底在编排什么？（答：控制流，不是模型）
2. LLM 决定 vs 代码决定，各自的代价是什么？
3. `as_tool()` 和 `handoffs` 明明都能"调用专家"，到底差在哪、什么时候必须选哪个？
4. 工具跑完之后，是"再问模型一次"还是"直接收工"？这个开关在哪一级？

---

## 三、⭐ 编排 = 控制流

"编排（orchestration）"这个词听着玄，拆开就三件事：

| 问题 | 白话 | 例子 |
|---|---|---|
| **哪些 agent 参与** | 工种清单 | 分类器、订票专员、退款专员、文案、评审 |
| **按什么顺序跑** | 执行图 | 先分类再路由；先写再评；两个并行跑 |
| **下一步谁定** | 决策权归属 | 模型自己挑工具 / 你的 `if` 挑 agent |

> **关键认知：编排是"流程"层面的事，和"模型能力"是两回事。**
> 同一个模型，你把它放进 `while` 循环里，它就得反复改稿；你把它放进 `asyncio.gather` 里，它就得和别人同时跑。**流程由你搭，模型只在流程给你的格子里发挥。**

---

## 四、⭐ 两条路线：LLM 决策 vs 代码决策

官方文档把编排分成两大类（multi_agent.md:7-11）：

| | **LLM 决策**（Orchestrating via LLM） | **代码决策**（Orchestrating via Code） |
|---|---|---|
| 谁定下一步 | **模型**在运行时自己挑（调哪个工具 / 交接给谁） | **Python** 写死（`if` / `for` / `while` / `await`） |
| 适合 | **开放式任务**：步骤事先说不清，需要模型随机应变（研究、排障、多轮客服） | **流程明确的任务**：步骤能画出来（分类→路由、写→评→改、并行调研） |
| 优点 | 灵活、能处理没预设过的情况 | 确定、可预测、便宜、快、好测试 |
| 缺点 | 不可预测、难评估、贵（每步都要模型判断）、可能跑偏 | 不灵活，遇到没写过的分支就没办法 |
| 调试方式 | 看 trace、改提示词、加评估 | 看代码、断点、单测 |

**可以混搭**——这也是实践中常见的形态：

```
你的代码（外层编排）
  ├─ if 用户问的是退款:            ← 代码决策
  │    退款专员（内部自己挑工具）    ← LLM 决策
  └─ else:
       通用客服
```

**一句话取舍**：

> **流程你能提前画出来的部分，交给代码；画不出来的部分，留给模型。**

---

## 五、⭐ 两种 SDK 模式：agents as tools vs handoffs

LLM 决策路线在 SDK 里落地成两种写法（multi_agent.md:26-33）：

| | **Agents as tools**（`Agent.as_tool()`） | **任务转移**（`handoffs`） |
|---|---|---|
| 谁对用户说话 | **管理器（外层 agent）始终对用户说话** | **专家接手，成为当前 agent** |
| 专家看到什么 | **生成的输入**（管理器写的一段任务描述） | **完整对话历史** |
| 专家跑完 | 结果作为**工具输出**还给管理器，管理器再组织回答 | 专家**继续**这一回合，不再回到原 agent |
| `last_agent` | 始终是管理器 | 变成接手的那个人 |
| 外层 `new_items` | 只有一次工具调用（嵌套 run 的 items **不在里面**） | 有 `HandoffCallItem` / `HandoffOutputItem` 这一对 |
| usage | **共享**（嵌套调用的 token 算进同一份 `context_wrapper.usage`） | 同一个 run，自然累计 |
| 控制感 | 强：所有输出都过管理器一道 | 弱：专家直接面向用户 |

### 5.1 什么时候用 `as_tool()`（管理器模式）

- 你希望**由一个 agent 负责最终回答**，把多个专家的输出整合成一份；
- 需要在**一个位置**统一实施安全策略 / 输出风格 / 审核；
- 专家只是**范围明确的子任务**（"查一下这个订单状态"），不该接管对话；
- 想对专家结果做**二次加工**（`custom_output_extractor`）。

### 5.2 什么时候用 `handoffs`（交接模式）

- 你希望**专家直接响应用户**，保持提示词聚焦（客服→退款专员，退款专员用退款话术直接答）；
- **路由本身就是工作流的一部分**（分流前台就是干这个的）；
- 想通过换 agent **切换当前生效的指令集 / 工具集**；
- 不想让管理器"转述"（转述会丢细节、多一次模型调用）。

### 5.3 有没有"必须"的场景？

严格说两者在功能上大多可以互换，但有几条**强烈倾向**：

| 场景 | 选 | 原因 |
|---|---|---|
| 要合并多个专家结果写一份报告 | **as_tool** | 只有管理器能"汇总" |
| 要在所有专家输出上加统一审核 | **as_tool** | 管理器是唯一出口 |
| 专家要直接跟用户多轮对话 | **handoff** | 交接后专家持有完整历史和发言权 |
| 专家必须看到完整对话历史才能干活 | **handoff** | as_tool 只给"生成的输入"，不给历史 |
| 嵌套深处还要审批 | **as_tool**（外层 RunState 处理） | 中断会浮到外层 run，见下一章 |

> SDK 源码里 `as_tool()` 的 docstring 自己就把两条区别写死了（`agent.py:632-636`）：**handoff 给历史 + 接管对话；as_tool 给生成的输入 + 原 agent 继续。**

---

## 六、⭐ `as_tool()` 全参数与三个细节

### 6.1 完整签名（SDK 0.24.0，`agent.py:606-629`）

```python
def as_tool(
    self,
    tool_name: str | None,
    tool_description: str | None,
    custom_output_extractor: Callable[[RunResult | RunResultStreaming], Awaitable[str]] | None = None,
    is_enabled: bool | Callable[[RunContextWrapper, AgentBase], MaybeAwaitable[bool]] = True,
    on_stream: Callable[[AgentToolStreamEvent], MaybeAwaitable[None]] | None = None,
    run_config: RunConfig | dict | None = None,
    max_turns: int | None = None,
    hooks: RunHooks | None = None,
    previous_response_id: str | None = None,
    conversation_id: str | None = None,
    session: Session | None = None,
    failure_error_function: ToolErrorFunction | None = default_tool_error_function,
    needs_approval: bool | Callable[[RunContextWrapper, dict, str], Awaitable[bool]] = False,
    parameters: type | None = None,
    input_builder: StructuredToolInputBuilder | None = None,
    include_input_schema: bool = False,
    on_stream_max_pending_events: int | None = 1024,
) -> FunctionTool
```

分组记：

| 组 | 参数 | 作用 |
|---|---|---|
| **门面** | `tool_name` / `tool_description` | 模型看到的名字和说明（不给名字就用 agent 名转 snake_case） |
| **结果加工** | `custom_output_extractor` | 嵌套 run 的结果 → 返回给管理器的字符串 |
| **开关** | `is_enabled` | bool 或回调；False 时**对模型隐藏**这个工具 |
| **流式** | `on_stream` / `on_stream_max_pending_events` | 把嵌套 agent 的流式事件转发出来（会切到流式执行） |
| **怎么跑** | `run_config` / `max_turns` / `hooks` | 嵌套 run 自己的运行配置、轮数上限、生命周期钩子 |
| **记忆** | `previous_response_id` / `conversation_id` / `session` | 嵌套 run 的对话状态策略 |
| **出错** | `failure_error_function` | 嵌套失败时给模型的错误文案；None=直接抛 |
| **审批** | `needs_approval` | 这个 agent 工具本身要不要审批（下一章） |
| **结构化入参** | `parameters` / `input_builder` / `include_input_schema` | 让模型按 Pydantic/dataclass 填参数，再由 `input_builder` 组装成嵌套输入 |

### 6.2 细节一：嵌套 run 是**完整的 `Runner.run`**

`as_tool()` 内部不是"简化调用"，而是货真价实地跑一次完整 agent 循环（`agent.py:1044`）：

```python
result = await Runner.run(
    self,
    input=...,
    max_turns=self._max_turns or DEFAULT_MAX_TURNS,   # 默认 10
    hooks=...,
    run_config=...,
    session=...,
)
```

所以：
- 嵌套 agent **自己也会跑多轮、调工具、交接**；
- 嵌套 run 的 `max_turns` 默认也是 **10**（`run_config.py:45`）——专家内部也可能撞上限；
- 嵌套 run 的 items **不会**出现在外层的 `result.new_items` 里，外层只看到"一次工具调用 + 一次工具输出"。

### 6.3 细节二：usage 是**共享**，不是相加

`agent.py:764` 构造 `ToolContext(usage=context.usage)`——**传的是同一个 usage 对象**，不是拷贝。所以：

```python
print(result.context_wrapper.usage.requests)
# 已经包含了嵌套专家内部的所有模型调用
```

**别再把嵌套 usage 加一遍**，也别指望在两份结果里各看各的。这是"本趟 run 总共花了多少"的单一账本。

### 6.4 细节三：`custom_output_extractor` **必须 async**

源码 `agent.py:1074` 是无条件 `await custom_output_extractor(run_result)`：

```python
return await custom_output_extractor(run_result)   # 不管你传的是同步还是异步，都 await
```

传同步函数 → `TypeError: object str can't be used in 'await' expression`。

**正确写法**：

```python
async def inspect_nested(nested: RunResult) -> str:
    print(nested.last_agent.name, len(nested.new_items))
    return str(nested.final_output)

refund.as_tool(
    tool_name="refund_expert",
    tool_description="处理退款相关的问题。",
    custom_output_extractor=inspect_nested,
)
```

**默认提取顺序**（`agent.py:1076-1093`）：`final_output` 优先；若为空且有输出护栏，保留空值；否则回退到嵌套 run 里最后一条非空消息或工具输出。

---

## 七、⭐ `tool_use_behavior`：Agent 级字段，不是工具参数

> 这是本章最容易放错位置的知识点。**它挂在 `Agent(...)` 上，不在 `function_tool` / `as_tool` 的参数表里。**

源码位置：`agent.py:396-398`

```python
tool_use_behavior: (
    Literal["run_llm_again", "stop_on_first_tool"] | StopAtTools | ToolsToFinalOutputFunction
) = "run_llm_again"
```

语义实现：`run_internal/turn_resolution.py:246` 每轮工具跑完后调用 `check_for_final_output_from_tools`，真正分支在 `:776-808`。

### 7.1 四种取值

| 取值 | 工具跑完后 | 模型调用次数 | 最终输出 |
|---|---|---|---|
| **`"run_llm_again"`**（默认） | 把工具结果送回模型 | 多一次 | 模型组织后的回答 |
| **`"stop_on_first_tool"`** | 直接收工 | 省一次 | **第一个工具的输出原文** |
| **`StopAtTools(stop_at_tool_names=[...])`** | 命中名单里的工具就收工，其余照常 | 视情况 | 命中工具的输出 |
| **自定义可调用对象**（可 async） | 你自己判断 | 视情况 | 必须返回 `ToolsToFinalOutputResult(is_final_output, final_output)` |

```python
from agents import Agent, StopAtTools, ToolsToFinalOutputResult

agent = Agent(
    name="工具直接收尾",
    tools=[get_weather],
    tool_use_behavior="stop_on_first_tool",
)
```

### 7.2 三个容易误解的点

1. **它是 Agent 级**——所以嵌套时有两层：
   - 专家自己的 `tool_use_behavior` 决定专家**内部**工具怎么收尾；
   - 管理器的 `tool_use_behavior` 决定"专家这个工具"跑完怎么收尾。
2. **`stop_on_first_tool` 只管"第一个"**——一次模型回复里若有多个工具调用，取 `tool_results[0].output`。想按名字挑，用 `StopAtTools`。
3. **托管工具不归它管**（`agent.py:414` docstring）：file search / web search 这类托管工具**永远由 LLM 处理**，不走这个逻辑。

### 7.3 什么时候用

- **`stop_on_first_tool`**：工具输出就是最终答案（查天气、查库存、返回固定格式），省一次模型调用 = 省钱 + 省延迟；
- **`StopAtTools`**：混合场景——大部分工具要模型收尾，但某个"终结型工具"（如提交订单）一调就该结束；
- **自定义函数**：需要在代码里做判断（"结果为空就继续问模型，非空就收工"）。

---

## 八、⭐ 四种代码编排模式

官方文档列了四种（multi_agent.md:49-54），每一种都配 demo。

### 8.1 模式 A：结构化分类 → 路由

**做法**：先让一个分类 agent 用 `output_type` 吐结构化结果，再由代码按类别选下一个 agent。

```python
class Route(BaseModel):
    category: Literal["booking", "refund", "other"]
    reason: str

classifier = Agent(name="分类器", output_type=Route, model=model)
classified = await Runner.run(classifier, question)
route: Route = classified.final_output        # 已经是对象，不是字符串
picked = specialists[route.category]          # 代码路由
answer = await Runner.run(picked, question)
```

**要点**：
- `output_type` 让模型必须吐合法 JSON，SDK 帮你解析成对象；
- 路由表用 `dict`，**不要用一长串 `if/elif`**——加类别时只改表；
- 分类器和执行者**分开**：分类器不需要工具，提示词可以很短、很便宜。

**适合**：工单分流、意图识别、多租户路由。

### 8.2 模式 B：串联（把上一个的工件传给下一个）

**做法**：把 agent 的**输出**当作下一个 agent 的**输入**，而不是共享上下文。

```
研究 agent → 大纲 agent → 写作 agent → 评审 agent → 修订 agent
```

**关键**：传的是**工件（artifact）**，不是对话历史。每个 agent 只看到自己该看的那一版，提示词因此可以保持聚焦。

**坑**：工件太长会撑爆上下文；要么做摘要，要么只传结构化字段。

### 8.3 模式 C：`while` 循环「生成器 + 评估器」

**做法**：

```python
for i in range(1, 4):                 # ⚠️ 硬性上限，防死循环
    prompt = f"主题：{topic}"
    if draft:
        prompt += f"\n上一版：{draft}\n评审意见：{feedback}\n请改进。"
    draft = (await Runner.run(writer, prompt)).final_output
    verdict: Verdict = (await Runner.run(judge, f"标语：{draft}")).final_output
    if verdict.ok:
        break
```

**三个必须**：
1. **必须有硬上限**（`range(1, 4)`），否则"评估器永远不满意"就是无限循环 + 无限账单；
2. **评审用 `output_type`**（`Verdict(ok, score, feedback)`），代码才能判断 `if verdict.ok`；
3. **反馈要显式回填**到下一轮 prompt，模型不会自己记得上一版。

**适合**：文案打磨、代码生成 + 测试、报告修订。

### 8.4 模式 D：`asyncio.gather` 并行

**做法**：

```python
pro_result, con_result = await asyncio.gather(
    Runner.run(pro, "分析远程办公"),
    Runner.run(con, "分析远程办公"),
)
```

**四个注意**：
1. **结果顺序 = 入参顺序**，不是完成顺序；
2. **一个失败默认全炸**——要逐条容错加 `return_exceptions=True`，再检查每项是不是异常；
3. **usage 各自独立**（两次独立 `Runner.run`），要自己加总；
4. **注意限流**——一次 `gather` 100 个请求，网关/模型会给你颜色看；批量场景用 `Semaphore`（见第十节的"开发场景"）。

**适合**：多视角调研、多个候选并行生成、批量数据处理。

---

## 九、五条 LLM 编排策略

官方给出的五条（multi_agent.md:39-43），本质是**一个闭环**：

```
① 优质提示词 → ② 监控 + 迭代 → ③ 自省 / 改进 → ⑤ 评估 → 回到 ①
                        ④ 单一职责专家（贯穿始终）
```

| # | 策略 | 怎么落地 | 常见坑 | 何时值得 |
|---|---|---|---|---|
| **①** | **投入精力设计优质提示词** | 明确写出有哪些工具、怎么用、必须遵守什么约束；用祈使句、给例子 | 工具说明含糊 → 模型乱调；约束没写 → 模型自由发挥 | 永远值得，ROI 最高 |
| **②** | **监控应用并持续迭代** | 用 tracing 看每趟 run 的 span；把失败案例收集成数据集 | 只看最终答案不看轨迹；改完不回归 | 上线后持续做 |
| **③** | **允许智能体自省和改进** | 循环 + 评估器（模式 C）；或把错误信息回填给模型让它自己改 | 没有硬上限；反馈太笼统（"再改改"） | 输出质量要求高的创作类任务 |
| **④** | **单一职责的专家** | 一个 agent 只管一件事：订票的就别管退款；工具集尽量小 | 一个"万能 agent" 提示词越写越长、工具越挂越多 | 任务一复杂就该拆 |
| **⑤** | **评估（evals）** | 建一批固定输入 + 期望输出的数据集，每次改提示词/换模型都跑一遍 | 凭感觉判断"好像变好了"；数据集太小 | 任何要认真维护的 agent |

> **一句话**：**提示词是地基，评估是尺子，监控是眼睛，自省是修复，单一职责是结构。** 少了评估，其余四条都在盲改。

---

## 十、⭐ asyncio 深讲：`await` 到底在等谁

这一节是本章的"地基"，不理解它，模式 D 就只是抄代码。

### 10.1 先从一个错误直觉说起

很多人（包括我）的第一反应：

> "`a = await Runner.run(pro, q)` 发出请求后，进程应该挂起，趁等网络的时候去发下一个 `con` 请求吧？"

**错。** 这么写就是**串行**，两个请求不会重叠。

### 10.2 三个对象：函数、协程、任务

```python
async def fake_llm(name, seconds):     # ① 函数对象（async def 定义的东西）
    await asyncio.sleep(seconds)
    return f"{name}-result"

coro = fake_llm("pro", 2)              # ② 调用 async def → 得到「协程对象」，函数体一行都没跑
task = asyncio.create_task(coro)       # ③ 把协程提交给事件循环 → 得到一个「任务」，此刻才排队
result = await task                    # ④ 等任务跑完，拿结果
```

| 对象 | 创建方式 | 有没有开始跑 | 能不能取消 | 能 await 几次 |
|---|---|---|---|---|
| **函数对象** | `async def` 定义处 | — | — | — |
| **协程对象** | 调用 `async def` | ❌ **惰性，没跑** | ❌ 没有 `cancel()` | **只能一次**（第二次 `RuntimeError: cannot reuse already awaited coroutine`） |
| **任务 Task** | `create_task` / `gather` / `asyncio.run` | ✅ 已登记，等调度 | ✅ `cancel()` | **多次**（结果缓存） |

### 10.3 `await` 的两种身份

| 写法 | 含义 | 会不会创建任务 |
|---|---|---|
| `await coro` | **在当前任务里就地驱动这个协程** | ❌ 不创建，只是"往下跑" |
| `await task` | **等一个已经登记的任务跑完** | ❌ 任务已经存在，这里只是等它 |

> **核心结论：`await` 从不创建任务。它只是"当前任务在这里停一下"。**
> 并发的前提是**事件循环里同时有 ≥2 个已登记的任务**。

### 10.4 实测：4.0s vs 2.0s

`E:\agent_study\code\asyncio_await_demo.py` 用 `asyncio.sleep(2)` 模拟两次网络等待，实测结果：

| 演示 | 写法 | 总耗时 | 说明 |
|---|---|---|---|
| 一 | `a = await fake_llm("pro", 2)`；`b = await fake_llm("con", 2)` | **4.0s** | 串行 |
| 二 | 先 `coro_b = fake_llm("con", 2)`，再两个 await | **4.0s** | **预先创建协程也没用**——协程对象不跑 |
| 三 | `create_task` ×2 → `await gather(ta, tb)` | **2.0s** | 并发 |
| 四 | `await asyncio.gather(fake_llm(...), fake_llm(...))` | **2.0s** | 并发（gather 帮你建任务） |

演示三里终端会看到：

```
[pro] 请求发出
[con] 请求发出        ← 两个"发出"相邻，然后才一起"返回"
```

**这就是"交叠"的证据**——不是靠进程挂起，而是靠**第一个任务在 `await sleep` 处让出控制权，事件循环转去跑第二个任务**。

### 10.5 串行 / 并发 / 并行

| 词 | 含义 | asyncio 属于？ |
|---|---|---|
| **串行** | 一件做完再做下一件 | 单任务时是 |
| **并发** | 多件事**交替推进**（单线程即可） | ✅ **asyncio 的本质** |
| **并行** | 多件事**同一时刻真正同时执行**（多核/多线程） | ❌ 单线程 asyncio 不是并行 |

> **asyncio 是"单线程并发"**：CPU 计算仍是串行的，但 **I/O 等待可以重叠**。
> 对 LLM 调用这种"绝大部分时间在等网络"的任务，并发收益接近并行；对 CPU 密集任务（本地大计算），收益≈0。

### 10.6 真实的并发从哪里来？三个来源

你说过一句很准的话：**"没有 gather，只是单纯用 async/await，根本算不上并发。"** 完全正确。并发必须有多个任务，任务来源只有三种：

| 来源 | 谁建的 Task | 例子 |
|---|---|---|
| **① 你自己的代码** | 你写 `create_task` / `gather` / `TaskGroup` | demo 里的 `asyncio.gather(Runner.run(pro), Runner.run(con))` |
| **② 框架** | Web 框架每个请求一个任务 | FastAPI 同时处理 100 个用户请求 |
| **③ SDK 内部** | SDK 自己建 | 输入护栏与首次模型调用并行（`run.py:1800-1841`）；多个护栏各一个 task（`guardrails.py:67-84`） |

也就是说：**你写的是串行代码，不代表整个系统是串行的**——框架和 SDK 可能在你看不见的地方已经并发起来了。

### 10.7 开发 agent 时，哪些场景真的需要并行？

| 场景 | 为什么要并行 | 注意 |
|---|---|---|
| **多视角调研** | 正反方 / 多个专家同时分析同一问题 | 结果要合并，注意顺序 |
| **多个候选生成** | 一次生成 3 个方案再选优 | 成本 ×3，要评估值不值 |
| **批量处理** | 100 条工单分类 | **必须限流**（`Semaphore`）、`return_exceptions=True` |
| **独立子任务** | 查天气 + 查汇率 + 查库存，互不依赖 | 只要有一个结果影响另一个，就不能并行 |
| **尽量别并行** | 有先后依赖（先分类再路由）、要共享同一 session 的多轮 | 并行会打乱顺序 / 产生竞态 |

---

## 十一、实际应用：两个可运行 demo

### 11.1 `multi_agent_demo.py`（本章主 demo，配 `run_multi_agent.bat`）

五个演示，一一对应本章知识点：

| # | 演示 | 看什么 |
|---|---|---|
| 1 | 管理器（`as_tool`）vs 任务转移（`handoffs`） | `last_agent` 是否换人、`new_items` 里有没有 `HandoffCallItem`、嵌套 items 是否缺席、usage 是否含嵌套调用 |
| 2 | `tool_use_behavior` | `run_llm_again` vs `stop_on_first_tool` 的 `final_output` 差异 + `len(raw_responses)` |
| 3 | 分类 → 路由 | `output_type=Route` 的 `final_output` 是对象，`route.category` 直接用于查表 |
| 4 | 生成器 + 评估器 | `Verdict(ok, score, feedback)`；`for i in range(1, 4)` 硬上限 |
| 5 | `asyncio.gather` | 两个 `Runner.run` 并行，`time.perf_counter()` 对比耗时 |

运行（必须先填 `config.bat` 里的 key）：

```bat
run_multi_agent.bat
```

### 11.2 `asyncio_await_demo.py`（纯本地，不需要 key）

```bat
py -3.13 E:\agent_study\code\asyncio_await_demo.py
```

四个演示证明第十节的结论：串行 4.0s / 预建协程 4.0s / `create_task` 2.0s / `gather` 2.0s。

> ⚠️ Bash 捕获中文可能按 GBK 乱码，加 `PYTHONIOENCODING=utf-8` 重跑即可。

---

## 十二、易混概念辨析（汇总表）

| 易混对 | 区别一句话 |
|---|---|
| **编排 vs Agent 配置** | 编排是"流程怎么走"，Agent 配置是"单个 agent 是谁、会什么" |
| **LLM 决策 vs 代码决策** | 下一步由模型挑 vs 由 `if/for/await` 定 |
| **`as_tool` vs `handoff`** | 专家当工具（管理器继续说话、只给生成的输入）vs 专家接管（拿完整历史、直接对用户说话） |
| **`as_tool` vs 普通工具** | 前者内部跑**完整 Runner.run**（多轮、可再调工具），后者只是执行一个函数 |
| **`tool_use_behavior` 在哪级** | **Agent 级字段**，不是 `function_tool` / `as_tool` 参数 |
| **`stop_on_first_tool` vs `StopAtTools`** | 前者"第一个工具"就收工；后者**按名字**匹配才收工 |
| **嵌套 usage vs 外层 usage** | 同一份账本（共享），**不要相加** |
| **嵌套 `new_items` vs 外层 `new_items`** | 嵌套 items 只存在于嵌套 `RunResult`；外层只看到工具调用/输出两条 |
| **协程 vs Task** | 协程=惰性、未登记、只能 await 一次、不可取消；Task=已登记、可多次 await、可取消 |
| **`await 协程` vs `await Task`** | 前者就地驱动（不建任务），后者等一个已存在任务 |
| **串行 vs 并发 vs 并行** | 一件件做 / 交替推进 / 真正同时执行；asyncio=并发非并行 |
| **并发三来源** | 你的代码 / 框架 / SDK 内部 |
| **`gather` 结果顺序** | 按**入参顺序**，不是完成顺序 |
| **`gather` 报错** | 默认第一个异常就整体抛出；`return_exceptions=True` 才逐条容错 |

---

## 十三、问答记录（Q&A）

**Q1：`tool_use_behavior` 是啥？怎么没在 `as_tool()` 参数里？**
A：它是 **Agent 级字段**（`agent.py:396`），控制"工具跑完后谁来收尾"。`as_tool()` 是"把一个 agent 变成工具"，属于工具级；两者不在同一层。上一节我把它们排得太近，造成了误导——实际使用时写在 `Agent(...)` 构造参数里。嵌套时有两层（专家内部 / 管理器这一层）。

**Q2：`asyncio.gather` 又是啥？前因后果是什么？**
A：官方文档 multi_agent.md:54 说"用 `asyncio.gather` 等 Python 基础组件并行跑多个 agent"。前因：前面所有 `await` 都是一件件等（串行）；`gather` 是"把多个协程一起登记成任务，然后等它们全部完成"。后果：总耗时≈最慢那个，而不是全部相加。它的本质是**建任务 + 等待**——`create_task` 是你手动建，`gather` 是它帮你建。

**Q3：`a = await Runner.run(pro, q)` 发出请求后，不是应该挂起进程去发下一个吗？**
A：不会。`await` 是"当前任务在这里停一下"，事件循环只会去跑**已经登记的任务**。你只写了一个 `await`，循环里就只有一个任务 → 它只能干等 → 串行。想交叠，必须先用 `create_task` / `gather` 把第二个也登记进去。

**Q4：协程和任务有什么区别？啥时候创建任务？**
A：调用 `async def` 只创建**惰性协程对象**（函数体没跑）；`create_task` / `gather` / `asyncio.run` 才创建**任务**。你说的"第一步创建协程、第二步 await 启动并驱动"——**前半句对，后半句要改**：`await` 不创建任务，它是**就地驱动**这个协程。协程只能 await 一次；任务可以多次 await、可以 `cancel()`。

**Q5：所以 asyncio 叫异步，做的还是串行的事？**
A：准确说是"**并发但不是并行**"。单任务时它是串行的；一旦有 ≥2 个任务，它就是**交替推进**（I/O 并发）。CPU 计算始终是单线程串行的，所以 CPU 密集任务用 asyncio 没收益。

**Q6：没有 `gather`，只用 `async/await` 根本不算并发吧？**
A：**完全正确。** 并发需要多个已登记任务；不 `create_task` / 不 `gather`，就只有当前一个任务。补充：并发也可能来自框架（Web 服务器）或 SDK 内部（护栏并行、`run.py:1800-1841`），所以"你的代码串行"≠"整个系统串行"。

**Q7：开发 agent 时哪些场景会用到并行？**
A：多视角调研、多候选生成、独立子任务、批量处理（必须限流）。反例：有先后依赖（先分类再路由）、共享同一 session 的多轮、需要严格顺序的流程——这些别并行。

**Q8：五条 LLM 编排策略具体怎么用？**
A：见第九节。核心是闭环：提示词（地基）→ 监控（眼睛）→ 自省（修复）→ 评估（尺子）→ 回到提示词；单一职责是贯穿始终的结构原则。没有评估，其余四条都是盲改。

---

## 十四、和前面章节的对照

| 本笔记概念 | 前面哪里出现过 | 本章新在哪 |
|---|---|---|
| **handoffs** | 笔记 04 专章 | 与 `as_tool` 放在同一张取舍表里 |
| **`interruptions` / `to_state()`** | 笔记 09 只作为 `RunResult` 的一栏提过；笔记 11 提过流式审批 | **下一章正式展开 HITL**；本章只强调嵌套 `as_tool` 的审批会浮到外层 RunState |
| **护栏并行** | 笔记 05 的 `run_in_parallel` | 作为"并发三来源之 SDK 内部"的证据 |
| **Agent Loop / turn** | 笔记 08 | 嵌套 run 也是完整 Loop，`max_turns` 默认同为 10 |
| **usage** | 笔记 09 `context_wrapper.usage` | 本章明确：`as_tool` 嵌套 usage **共享不叠加** |
| **`RunResult`** | 笔记 09 行程单 | 本章用 `last_agent` / `new_items` / `final_output` 区分两种编排 |
| **`RunConfig`** | 笔记 08 | `as_tool(run_config=...)` 可给嵌套 run 单独配置 |
| **结构化输出 `output_type`** | 笔记 03 / 09 | 作为"分类器 / 评估器"的核心工具 |
| **asyncio** | 笔记 11 只讲了 `await` vs `async` 的语法层面 | **本章讲到协程 / Task / gather / 并发≠并行** |

---

## 附：本章"坑"清单

1. `custom_output_extractor` **必须 async**（`agent.py:1074` 无条件 await）；
2. `tool_use_behavior` 写在 **`Agent(...)`** 里，不在工具参数里；
3. `stop_on_first_tool` 取的是 `tool_results[0]`，不是"你想要的最后一个"；
4. `as_tool` 嵌套 run 的 `max_turns` 默认也是 **10**；
5. 嵌套 usage **共享**，别重复加总；
6. `gather` 默认**一个失败全炸**，批量场景记得 `return_exceptions=True`；
7. `gather` 结果顺序是**入参顺序**，不是完成顺序；
8. `while` 生成-评估循环**必须有硬上限**；
9. 预建协程对象**不会**让代码并发，必须 `create_task` / `gather`；
10. 批量 `gather` 要**限流**（`Semaphore`），否则网关/模型先扛不住；
11. 有先后依赖的子任务**不要**并行；
12. 托管工具（file search / web search）**永远由 LLM 处理**，不受 `tool_use_behavior` 影响。