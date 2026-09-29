# OpenAI Agent 学习笔记 02（Agent 基本配置篇）

> 本文档是「系统性学习 Agent」第二阶段的概念整理，聚焦 **`Agent` 类完整的基本配置属性**。
> 来源：OpenAI Agents SDK 官方中文文档「智能体」页（agents.md），配合对话逐层澄清。
> 学习日期：2026-09-29 · 学习者：陈朝辉

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [基本配置完整字段总览（16 个）](#二基本配置完整字段总览16-个)
3. [字段分组详解](#三字段分组详解)
4. [易混概念辨析（5 组对比）](#四易混概念辨析5-组对比)
5. [问答记录（Q&A）](#五问答记录qa)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 智能体 agents | https://openai.github.io/openai-agents-python/zh/agents/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/agents.md |

> 源码 Markdown 可经 `raw.githubusercontent.com` 直连获取；本文档的字段清单据此逐条核对。
> 相关相邻指南（后续待学）：tools、handoffs、guardrails、models、context、running_agents、results、multi_agent。

---

## 二、基本配置完整字段总览（16 个）

这是官方文档「基本配置」章节列出的 **`Agent` 类全部常用属性**。同一个字段，就是你在 `Agent(...)` 里能传的参数。

| 属性 | 必需 | 说明 | 归类 |
|---|---|---|---|
| `name` | **是** | 便于人类阅读的名称（标识/日志/交接工具名） | 身份 |
| `instructions` | 否 | 系统提示词（或动态指令回调），强烈建议设置 | 身份 |
| `prompt` | 否 | Responses API 提示词配置（引用平台上的提示词模板） | 身份 |
| `handoff_description` | 否 | 作为交接目标时，给发起方模型看的简短说明 | 交接 |
| `handoffs` | 否 | 能委派给哪些专业子 agent | 交接 |
| `model` | 否 | 使用哪个 LLM | 模型 |
| `model_settings` | 否 | 模型调优参数（`temperature`/`top_p`/`tool_choice` 等） | 模型 |
| `tools` | 否 | 可调用的工具 | 工具 |
| `mcp_servers` | 否 | 提供 MCP 工具的 MCP 服务器 | 工具 |
| `mcp_config` | 否 | 微调 MCP 工具的准备方式 | 工具 |
| `input_guardrails` | 否 | 输入安全防护（拦入口） | 安全 |
| `output_guardrails` | 否 | 输出安全防护（拦出口） | 安全 |
| `output_type` | 否 | structured outputs 类型（替换纯文本输出） | 输出 |
| `hooks` | 否 | 生命周期回调（观察运行过程） | 观察 |
| `tool_use_behavior` | 否 | 工具结果送回模型继续循环，还是直接结束 | 工具 |
| `reset_tool_choice` | 否 | 工具调用后重置 `tool_choice`（默认 `True`） | 工具 |

**关键认知：**
- **真正"必需"的只有 `name` 一个**。连 `instructions`、`tools` 官方都标了"否"（虽然 `instructions` 强烈建议设）。
- 之前笔记 01 说的"三要素（name/instructions/tools）"，更准确说是**最常用、最小习惯配置**，不是"必需"。
- 日常你会主动碰的其实就 `name`、`instructions`、`model`、`tools`（偶尔加 `handoffs`）；其余多是"默认就好、特殊场景才改"的可选增强。

---

## 三、字段分组详解

### 1. 身份三件套：`name` / `instructions` / `prompt`

**`name`（必需）**：给人看的标识，不喂给模型。唯一例外：交接时自动变成交接工具名 `transfer_to_<name>`。

**`instructions`**：系统提示词，每次 Responses API 调用时作为 system prompt 喂给模型，定"人设 + 规矩"。可以是**字符串**，也可以是**返回字符串的函数（动态指令）**。

- **写死字符串** `instructions="永远用俳句回答"` → 所有用户看到的提示词都一样。
- **传函数** `instructions=dynamic_instructions` → 每次运行前现场用当前 context 生成提示词，能做到"你好张三 / 你好李四"这种个性化。

  ```python
  def dynamic_instructions(context, agent) -> str:
      return f"The user's name is {context.context.name}. Help them."

  agent = Agent[UserContext](name="Triage", instructions=dynamic_instructions)
  ```

  > 本质：字符串是"答案"，函数是"配方"——SDK 里凡是"接收字符串"的地方，几乎都能换成"接收一个返回字符串的函数"。

**`prompt`**：引用"存在 OpenAI 平台上的提示词模板"，代码里只给 `id` + `version` + `variables`，模板本体在平台 UI 里建（可版本化、带 `{{变量}}` 插值）。

---

### 2. 交接相关：`handoffs` / `handoff_description`

**`handoffs`**：一个列表，声明"这活儿干到一半可以转交给谁"，里面放的是别的 agent。

**交接的工作机制（四步）：**

1. 配交接目标：`handoffs=[booking_agent, refund_agent]`
2. SDK 自动把每个交接对象**伪装成一个交接工具** —— `transfer_to_booking_agent`、`transfer_to_refund_agent`
3. 模型在"工具清单"里看到这些交接工具，判断该转时"点单" <q>我要 `transfer_to_refund_agent`</q>
4. Runner 识破这是交接（非普通工具）→ 换 `current_agent`、把整段对话历史移交给新 agent 接着聊

**为什么要把交接伪装成工具？** 因为**模型天生只会两种表达**：回话、要工具，没有第三种"交接"。所以要交接，只能借用它已有的"要工具"这一式，把交接包装成工具，再由 Runner 在底下劫持、转译成"换人"。

> 通用哲学：**"工具调用"是模型与外界交互的唯一通用接口**。任何想让模型"触发某件事"的需求（交接 / 调用别的 agent / 护栏），都得先翻译成"工具"这唯一一种它听得懂的语言。

**`handoff_description`**：一句话说明，给**发起交接的模型**看，让它判断"这个交接对象是干嘛的、什么情况该转给它"。这段描述会嵌进交接工具（`transfer_to_<name>`）的 description 里。

**`handoffs` 里放的是什么样的 agent？** 不是"功能完全相反"的，而是"**分工不同、且当前 agent 不打算自己干的专职专家**"。判据是分工，不是对立。若功能完全一样，就用不着交接了。

---

### 3. 模型相关：`model` / `model_settings`

- `model`：用哪个 LLM。
- `model_settings`：模型调优参数——`temperature`（温度）、`top_p`、`tool_choice`（是否强制用工具）等。其中 `tool_choice` 有 `auto` / `required` / `none` / `"指定工具名"` 四档。

---

### 4. 工具相关：`tools` / `mcp_servers` / `mcp_config` / `tool_use_behavior` / `reset_tool_choice`

在深入这几个字段前，先补一张背景图：**工具从哪里来**，以及**到底需要几个 agent**。

**工具的四种来源（谁写的工具）：**

| 来源 | 谁写的 | 一句话说明 |
|---|---|---|
| 手写工具（FunctionTool） | 你自己 | 写个普通 Python 函数，包一层就变成工具 |
| 托管工具（Hosted Tools） | OpenAI 现成 | 联网搜索、文件检索、跑代码等，不用自己写 |
| MCP 工具 | 第三方/社区 | 通过 MCP 协议接第三方工具（数据库、各种服务） |
| Agent 当工具 | 复用已搭好的 agent | 把一个专业 agent 整体包装成另一个 agent 的工具 |

> 这张表正是理解下面 `tools` vs `mcp_servers` 的钥匙：`tools` 字段能装手写/托管/agent 工具，`mcp_servers` 字段则对应「MCP 工具」这一来源。

**需要几个 agent？——官方哲学：能少则少。**

- **默认**：一个 agent + 一堆工具，够干大多数事，不要一上来就拆。
- **什么时候才拆专职 agent**：两种情况才考虑拆——① 工具太多一个 agent 塞不下/记不住该用哪个；② 不同方向的规则（instructions）互相打架，需要专职 agent 各管各的人设和规矩。

> 一句话：工具大多不用自己写（能托管、能外部接、能复用现成 agent）；agent 则是「能用 1 个就别用 N 个」，够了才不动，塞不下/会打架才拆。

**`tools` vs `mcp_servers`（别搞混）：**

- `tools` = 你**直接、手动**塞的一个个工具（自己写的 FunctionTool / OpenAI 托管工具 / agent 当工具）。
- `mcp_servers` = 你**连一个 MCP 服务器**，SDK 自动把那个服务器暴露的一批工具拉进来挂给 agent。

  > MCP（Model Context Protocol）= 开放标准协议，专门让 AI 应用连接外部工具/数据源。殊途同归：不管来自 `tools` 还是 `mcp_servers`，最终都翻译成 JSON schema 塞进"喂给模型的工具清单"，模型一视同仁地点单。区别只在"这些工具你怎么拿到"——一件件放，还是接一条传送带让它自动送来。

- `mcp_config`：微调 MCP 工具的准备方式（如 schema 转严格模式等）。

**`tool_use_behavior`**：决定"模型调用了工具之后，Runner 接下来怎么办"。

| 取值 | 行为 |
|---|---|
| `"run_llm_again"`（默认） | 工具结果喂回模型，模型再生成最终回应 |
| `"stop_on_first_tool"` | 第一次工具输出直接当最终答案，不再喂回模型 |
| `StopAtTools(stop_at_tool_names=[...])` | 只有调了指定工具才停，其他仍继续 |
| `ToolsToFinalOutputFunction` | 自定义函数逐个判断"要不要当最终输出" |

> 对应笔记 01 的 Runner 循环：它改的是"工具调用后还走不走循环"这一支。默认"工具做完活，模型再回来汇报一句"；`tool_use_behavior` 让你能改成"工具结果直接就是成品，不用再汇报"。

**`reset_tool_choice`**：工具调用后自动把 `tool_choice` 重置回 `auto`，防止死循环。

- 背景：如果你设 `tool_choice="get_weather"` 强制模型必须调这个工具，则工具结果喂回后模型仍被强制再调 → 无限循环烧钱。
- 破解：`reset_tool_choice=True`（默认）在每次工具调用后自动把 `tool_choice` 复位为 `auto`，让模型恢复自由、能停下来说答案。
- 大多数情况不用碰它，默认 `True` 已把坑堵上；`False` 只用于"有意连续反复调用某工具"的特殊需求。

---

### 5. 安全相关：`input_guardrails` / `output_guardrails`

一道**独立的安全检查闸门**，护住 agent 的入口和出口——不是给模型立规矩，而是**真的拦**。

| | `input_guardrails` | `output_guardrails` |
|---|---|---|
| 位置 | 入口（用户输入刚进来） | 出口（agent 刚生成最终答案） |
| 检查对象 | 用户说的话 | agent 要说出去的话 |
| 触发动作 | 拦截，不让 agent 往下处理 | 拦截最终输出，不让它回到用户 |

**软约束 vs 硬拦截（关键区别）：**

| | 软约束（instructions 里的规矩） | 硬拦截（guardrails） |
|---|---|---|
| 怎么生效 | 叮嘱模型"尽量遵守" | 程序层面强制检查，说拦就拦 |
| 模型能绕过吗 | 可能不听话 | 绕不过去 |

> 模型在 prompt 里被叮嘱"别乱说"是靠自觉，不可靠；guardrail 是硬闸门，触发一定挡下。`guardrails` 还能与主流程**并行**运行，尽量不拖慢响应。

---

### 6. 输出相关：`output_type`

让模型的输出从"随便一段自由文本"变成"一个结构固定的对象"，代码能直接拿字段，不用从话里"抠"信息。

- 默认：`result.final_output` 是一段 `str` 文本。
- 设了 `output_type`：输出是字段明确的 Pydantic 对象（或 dataclass/list/TypedDict 等），`result.final_output.字段名` 直接用。底层走 OpenAI 的 **structured outputs**。

```python
class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

agent = Agent(name="Calendar extractor",
              instructions="Extract calendar events",
              output_type=CalendarEvent)   # result.final_output.date 直接用
```

**什么时候才需要设它？一句话判据：输出后面那个"消费者"是人还是代码？**

- 是人 → 不设，人话更好（所以日常聊天看不到字段）。
- 是代码 → 设，让代码拿到明确字段。

典型触发场景：信息提取/解析、回填表格/写库、多 agent 接力传数据、输出驱动代码走分支、后端校验审计存储。

**易错澄清：`output_type` 不是你看到"中间结构化、最终人话"的原因。**
中间循环里的"结构化"（工具调用参数、工具结果）是**工具机制自带的**，与 `output_type` 无关。`output_type` 只影响"最终那一格输出"，且是你在建 Agent 时**静态写死**的开关，Runner 不动态控制它。

---

### 7. 观察相关：`hooks`

**观察者/探针**——让你旁观 agent 运行的每个关键时刻并自动触发你写的函数，只观察、不动手、不改结果。

触发时机（回调）：

| 钩子 | 触发时机 |
|---|---|
| `on_agent_start` / `on_agent_end` | 某个 agent 开始 / 完成 |
| `on_llm_start` / `on_llm_end` | **每次**调用模型的前后 |
| `on_tool_start` / `on_tool_end` | 每次本地工具调用前后 |
| `on_handoff` | 交接发生时 |

两个作用域：`RunHooks`（整个 `Runner.run()`，含交接到的所有子 agent）vs `AgentHooks`（`agent.hooks`，只看某个 agent）。

用途：记录日志/调试、记录用量（token/费用 `context.usage`）、埋点/追踪（tracing）、预取数据。

**hooks vs guardrails 别搞混：**

| | hooks（钩子） | guardrails（护栏） |
|---|---|---|
| 目的 | 观察 / 记录 | 拦截 / 阻止 |
| 改结果吗 | 否 | 是，触发就拦 |
| 类比 | 片场花絮摄像机 | 门口安检闸门 |

---

## 四、易混概念辨析（5 组对比）

### ① `instructions` vs `prompt`

| | `instructions` | `prompt` |
|---|---|---|
| 内容在哪 | 内联在代码里 | 存 OpenAI 平台（模板库） |
| 代码写什么 | 完整提示词文本 | 只有 `id`/`version`/`variables` |
| 谁能管 | 你自己在代码里改 | 平台 UI 里改，可版本化 |

> 二者**都是喂给模型的提示词、都是约束**，区别纯粹在"内容从哪来、怎么管理"。`instructions`=把话直接写在代码里；`prompt`=去平台拿一份现成带变量的模板。

### ② `instructions` vs `handoff_description`

| | `instructions` | `handoff_description` |
|---|---|---|
| 给谁看 | 这个 agent **自己上场时**的模型 | 别的 agent 的模型**考虑要不要转给它**时 |
| 时机 | 这个 agent 正式运行时 | 交接发生前的选人阶段 |
| 长度 | 可长可详细 | "简短说明" |
| 位置 | system prompt | 交接工具的 description |

> **关键铁律**：每个 agent 的 `instructions` 只在"自己亲自运行时"才喂给它自己的模型。发起交接的模型**看不到**子 agent 的 instructions（它的上下文里只有自己的 instructions + 工具清单 + 对话历史），所以必须另用 `handoff_description` 给句门牌简介。类比：`instructions`=员工完整岗位手册；`handoff_description`=前台转接花名册上的一行字。

### ③ `tools` vs `mcp_servers`

| | `tools` | `mcp_servers` |
|---|---|---|
| 给的是 | 一个个具体工具 | 一个连接信息（接口地址） |
| 工具谁的 | 你自己/OpenAI/复用 agent | 第三方提供 |
| 怎么拿到 | 直接在列表里 | SDK 连服务器自动发现挂载 |

### ④ 软约束（instructions）vs 硬拦截（guardrails）

见上「安全相关」部分。

### ⑤ hooks（观察）vs guardrails（拦截）

见上「观察相关」部分。

---

## 五、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | 文档"基本配置"是不是 agent 的属性？为何除了三要素还有很多 | 是。基本配置= `Agent` 类的属性字段，共 16 个；三要素只是最核心的最小配置 |
| 2 | `instructions` 和 `prompt` 有什么区别，是否都约束模型 | 都是喂模型的提示词/约束；区别在"内联字符串 vs 引用平台模板" |
| 3 | `instructions` 传函数没听懂；`prompt` 是 OpenAI 提供还是自己写 | 函数=每次运行现场生成提示词的"配方"；`prompt` 是用户自己在平台建的模板，代码只引用 |
| 4 | Agent 类、Runner 类都属于 SDK 包吗 | 对，都属于 `openai-agents` 包（`from agents import Agent, Runner`） |
| 5 | `handoff_description` 和 `handoffs` 各起什么作用 | `handoffs`=能转给谁；`handoff_description`=给发起方模型看的一句话门牌 |
| 6 | 为什么把交接伪装成工具？`handoffs` 里是不是功能相反的 agent | 模型只会"回话/要工具"，故交接必须包装成工具；不是相反，是"分工不同的专职专家" |
| 7 | `handoff_description` 和 `instructions` 的区别 | 对象/时机/长度/位置都不同；交接那一刻对方看不到子 agent 的 instructions |
| 8 | `tools` 和 `mcp_servers` 的区别 | 直接列工具 vs 连 MCP 服务器自动拉工具，殊途同归 |
| 9 | `input_guardrails`/`output_guardrails` 是什么 | 独立硬拦截闸门，护入口/出口，区别于 instructions 的软约束 |
| 10 | `output_type` 是做什么的 | 把自由文本输出变成结构化对象，给代码直接用 |
| 11 | 实际用 agent 看不到结构化字段？ | 因为面向人聊天默认不设 `output_type`，人话更好 |
| 12 | output_type 由 Runner 控制开关吗 | 不是。是建 Agent 时静态写死；中间的结构化是工具机制自带的，与它无关 |
| 13 | 什么情况下才设输出格式 | 输出的下一个消费者是代码（落库/写表/接力/走分支）时 |
| 14 | `hooks` 是什么 | 生命周期回调，观察 agent 每一步，只记录不改结果 |
| 15 | `tool_use_behavior` 是什么 | 工具调用后是否喂回模型继续循环 |
| 16 | `reset_tool_choice` 是什么 | 工具调用后自动重置 tool_choice 防死循环，默认 True |

---

*本文档将持续补充。下一步待学：tools（工具）、handoffs 进阶、guardrails、tracing、context/sessions、running_agents、results。*