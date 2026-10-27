# OpenAI Agent 学习笔记 07（上下文与会话篇）

> 本文档是「系统性学习 Agent」第七阶段的整理，聚焦 **本地上下文（context）** 与 **会话（sessions）**——它和刚学的 tracing（笔记 06）正好接得上：tracing 是「在旁边**怎么录**它干了啥」，这一章是「运行前**怎么给它递料 / 让它记事**」。两者都绕不开「把多轮串起来」这件事（一个是 trace 的 `group_id`、一个是 `session`），本章第八节专门辨析。
> 来源：OpenAI Agents SDK 官方中文文档「上下文管理」（context.md）与「会话」（sessions.md）；另核对了 SDK 源码 `agents/run_context.py`、`agents/agent.py`、`agents/function_schema.py`、`agents/memory/session.py`。
> 学习日期：2026-10-27（理论学习 + 跑通 `session_demo.py`）
> 衔接：`RunContextWrapper` 在工具篇（笔记 03）首见、`RunConfig`（笔记 02/05）、`group_id`（笔记 06）、`Agent[TContext]` 泛型（本章第三节）、`Agent.as_tool()`（工具篇）。
> **状态：✅ 本章已学完（2026-10-27 收尾）**（已学：两个"上下文"的区分 / `RunContextWrapper` 与 `ToolContext` / 泛型标记 `Agent[TContext]` / LLM 上下文四条喂料路 / session 工作原理与五种后端 / 四个内存操作 / `session` vs `group_id` / 工具三形态与"谁决定"两层）。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [这一章讲什么：一句话 + 动机](#二这一章讲什么一句话--动机)
3. [核心澄清：两个"上下文"同名不同命](#三核心澄清两个上下文同名不同命)
4. [本地上下文（local context）：给代码用的"暗格"](#四本地上下文local-context给代码用的暗格)
5. [泛型标记 `Agent[TContext]` 是做什么的](#五泛型标记-agenttcontext-是做什么的)
6. [LLM 上下文：想让模型知道，就得"写到纸上"](#六llm-上下文想让模型知道就得写到纸上)
7. [会话（sessions）：内置的对话记忆](#七会话sessions内置的对话记忆)
8. [易混辨析：`session` vs trace 的 `group_id`](#八易混辨析session-vs-trace-的-group_id)
9. [实际应用：`session_demo.py` 跑出来的三件事](#九实际应用session_demopy-跑出来的三件事)
10. [易混概念辨析（汇总表）](#十易混概念辨析汇总表)
11. [问答记录（Q&A）](#十一问答记录qa)
12. [和 tracing 的对照](#十二和-tracing-的对照)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 上下文管理 context | https://openai.github.io/openai-agents-python/zh/context/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/context.md |
| 会话 sessions | https://openai.github.io/openai-agents-python/zh/sessions/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/sessions.md |

> ⚠️ 获取记录：本机直连 GitHub 不通。经镜像 `https://ghproxy.net/https://raw.githubusercontent.com/...` 取到两篇全文（同笔记 04/05/06 的通道），并已缓存到本地：
> - `E:\agent_study\docs_cache\context.md`
> - `E:\agent_study\docs_cache\sessions.md`
>
> 提问中涉及的行为，另核对了源码：
> - `agents/run_context.py`：`RunContextWrapper(Generic[TContext])`、`TContext = TypeVar("TContext", default=Any)`、`.context` / `.usage` / `.tool_input`；
> - `agents/agent.py`：`class Agent(AgentBase, Generic[TContext])` 及其 `tools` / `handoffs` / `*_guardrails` / `hooks` 字段的 `TContext` 标注；
> - `agents/function_schema.py`：**context 参数必须放第一位、只能有一个、不进 schema（模型看不见）** 的机制；
> - `agents/memory/session.py`：`Session` 协议 / `SessionABC`（`get_items` / `add_items` / `pop_item` / `clear_session`）。

---

## 二、这一章讲什么：一句话 + 动机

> **这一章解决两个"往 Agent 里塞东西"的问题：**
> - **context（本地上下文）** = 你（代码）的**口袋** 🧰：装运行时依赖和请求级状态，工具/钩子伸手进去掏，但 **LLM 摸不到**。
> - **LLM 上下文** = 桌上的**纸** 📄：LLM 只能看到纸上的字（对话历史）。你想让它知道什么，就得**写到纸上**。

**为什么要有它**：Agent 一跑起来，代码这边需要用一堆东西（用户身份、权限、数据库连接、logger……），模型那边又需要知道另一堆东西（用户说了什么、工具返回了什么）。这两拨东西**该给谁、不该给谁**分不清，就会出两类事故：要么**敏感数据泄给模型**（它不该知道的订单号、权限，结果它知道了还编造），要么**模型该知道的没给它**（它当然答不出来）。这一章就是讲**怎么把料按"该谁看"分派到正确的格子里**。

---

## 三、核心澄清：两个"上下文"同名不同命

官方开篇第一句就点破：**context 是个多重含义的词**。必须先分清是两个东西：

| | **本地上下文（local context）** | **LLM 上下文（LLM context）** |
|---|---|---|
| 给谁用 | **你的代码**（工具函数 / 回调 / 生命周期钩子） | **LLM** |
| 载体 | `RunContextWrapper[T]` 的 `.context` | 对话历史（conversation history） |
| 会发给模型吗 | **不会**（文档用 danger 框专门强调） | 就是这个 |
| 装什么 | 用户信息、权限、db 连接、logger、辅助函数 | 每句用户话、每句模型话、工具结果 |

> ⚠️ 文档原文（danger 框）：「上下文对象**不会**发送给 LLM。它完全是一个本地对象，你可以读取、写入它以及调用其方法。」

一句记忆口诀：
- **本地上下文 = 你的口袋 🧰**：装着运行时依赖，工具能伸手掏，**LLM 摸不到**。
- **LLM 上下文 = 桌上的纸 📄**：LLM 只能看到纸上的字，**想让它知道就得写到纸上**。

> 💡 接笔记 06：tracing 是「行车记录仪——看它干了啥」；context/session 是「口袋 + 桌上的纸——怎么递料、怎么记事」。

---

## 四、本地上下文（local context）：给代码用的"暗格"

### 4.1 三步用起来

```python
from dataclasses import dataclass
from agents import Agent, RunContextWrapper, Runner, function_tool

@dataclass
class UserInfo:                       # 1) 定一个自己的对象（dataclass / Pydantic / 任意类型都行）
    name: str
    uid: int

@function_tool
async def fetch_user_age(wrapper: RunContextWrapper[UserInfo]) -> str:
    return f"The user {wrapper.context.name} is 47 years old"   # 工具从上下文读数据

async def main():
    user_info = UserInfo(name="John", uid=123)
    agent = Agent[UserInfo](           # 3) 用泛型标记：本 agent 的上下文是 UserInfo（见第五节）
        name="Assistant",
        tools=[fetch_user_age],
    )
    result = await Runner.run(
        agent,
        "What is the age of the user?",
        context=user_info,             # 2) 运行时把对象传进去
    )
    print(result.final_output)
```

**三条规矩**：
1. 上下文对象可以是**任意 Python 类型**（dataclass / Pydantic 常见，但不必）。
2. `Runner.run(..., context=...)` 把它交给整次运行；所有工具、钩子都会收到包装器 `RunContextWrapper[T]`，用 `wrapper.context` 取出你的对象。
3. ⚠️ **对于给定的一次 agent 运行，每个 agent、工具函数、生命周期等都必须使用同一种上下文类型**（文档反复强调的"最重要"事项）。泛型标记就是替你盯这事的。

### 4.2 上下文里装什么（四类典型内容）

| 类别 | 装什么 | 例子 |
|---|---|---|
| **① 请求级数据**（这一轮是谁/什么） | 用户/租户/订单身份 | `user_id`、`tenant_id`、`order_no`、`locale` |
| **② 依赖对象**（外部资源句柄） | 不该每轮重建的连接/客户端 | `db` 连接池、`http_client`、`cache`、`logger` |
| **③ 辅助函数**（对象上的方法） | 基于上面数据算出来的判断 | `can_refund()`、`is_admin()` |
| **④ 可变状态**（一轮跑下来攒的结果） | 运行期读写、跑完给你看 | `actions_taken: list`、`sources: list` |

> 它**不是**配置常量（那些直接写模块级常量即可），而是「**这次请求 / 这个进程才有的东西**」。

### 4.3 `RunContextWrapper` 暴露什么

`RunContextWrapper` 是「你的上下文对象 + SDK 运行时元数据」的包装器。**只有 `.context` 是你自己的**，其余都是 SDK 管的：

| 属性 | 是谁的 | 干嘛 |
|---|---|---|
| `wrapper.context` | **你** | 你的应用可变状态与依赖 |
| `wrapper.usage` | SDK | 本次运行汇总的请求/token 用量 |
| `wrapper.tool_input` | SDK | 当前运行在 `Agent.as_tool()` 内执行时的结构化输入 |
| `wrapper.approve_tool()` / `.reject_tool()` | SDK | 以编程方式更新审批状态 |

> ⚠️ 若之后为「人在回路 / 持久化作业」序列化 `RunState`，这些运行时元数据会**随之保存**——所以**别把机密放进 `context`**。
> 源码：`run_context.py` 里 `TContext = TypeVar("TContext", default=Any)`（默认 `Any`，所以不写泛型也能跑，只是放弃静态检查）。

### 4.4 高级档：`ToolContext`

`ToolContext` 是 `RunContextWrapper` 的**子类**，除了同样的 `.context`，额外暴露**当前这次工具调用**的元数据：

| 字段 | 含义 |
|---|---|
| `tool_name` | 正在调用的工具名 |
| `tool_call_id` | 这次工具调用的唯一 ID |
| `tool_arguments` | 传给工具的原始参数字符串 |
| `tool_namespace` | 工具调用的 Responses 命名空间（用 `tool_namespace()` 等加载时才有） |
| `qualified_tool_name` | 存在命名空间时，带命名空间的工具全名 |

**什么时候用**：当你需要「**关于这次工具调用本身**」的信息时（比如日志要记 `tool_call_id`）。**一般的 agent↔工具 共享上下文，用 `RunContextWrapper` 就够。**

### 4.5 能力可见性（点到为止）

原文有一节讲「用本地上下文控制**哪些能力对模型可见**」：`FunctionTool.is_enabled`、`Handoff.is_enabled`、MCP `tool_filter` 都能拿到当前 `RunContextWrapper`，据此动态开关工具/交接。**注意**：这只能控制「暴露哪些能力」，**不能**对模型生成的参数或资源选择做授权——真正的授权要在工具实现内部做（或配合工具输入护栏 / 审批）。

> 🪧 **属于"接线型"知识，先记个目录，用到再深挖。**

---

## 五、泛型标记 `Agent[TContext]` 是做什么的

> 一句话：**`Agent[UserInfo]` 里的 `[UserInfo]` 不是运行开关，是给"类型检查器 / IDE"看的静态标注。**

源码实锤：`class Agent(AgentBase, Generic[TContext])`，其中 `TContext` 出现在一堆字段上：

```python
class Agent(AgentBase, Generic[TContext]):
    tools:             list[Tool]                       # 工具收到 RunContextWrapper[TContext]
    handoffs:          list[Agent[Any] | Handoff[TContext, Any]]
    input_guardrails:  list[InputGuardrail[TContext]]
    output_guardrails: list[OutputGuardrail[TContext]]
    hooks:             AgentHooks[TContext] | None
```

写 `Agent[UserInfo]` = 把这些带 `TContext` 的位置**一次性锁定成 `UserInfo`**，于是类型检查器能替你验一致性。

**它替你抓什么错**：

```python
async def fetch_user_age(wrapper: RunContextWrapper[UserInfo]) -> str: ...

Agent[UserInfo](tools=[fetch_user_age])     # ✅ 对得上
Agent[OrderInfo](tools=[fetch_user_age])    # ❌ mypy/pyright 报错：上下文类型不匹配
Runner.run(agent[UserInfo], "…", context=order_info)   # ❌ 同样被扣住
```

**两个别误会**：
1. **只影响静态检查，不影响运行**。运行时 `Agent[UserInfo](name=...)` 照样正常造出 agent，不会因参数多少改变行为。
2. **可省略**。`TContext` 默认 `Any`，不写也能跑，代价是失去上面所有检查（含 IDE 的 `.context.` 自动补全）。

> 同 `list[int]` 的 `int`：你在**参数化**一个泛型类。

---

## 六、LLM 上下文：想让模型知道，就得"写到纸上"

LLM **唯一**能看到的是**对话历史**。所以想给它新数据，只有四条路：

| 方式 | 什么时候用 | 例子 |
|---|---|---|
| **instructions**（= system prompt / 开发者消息） | 信息**随时都可能有用** | 用户姓名、当前日期 |
| **input**（`Runner.run` 的 `input`） | 同上，但**指令层级更低** | 本轮临时补充 |
| **FunctionTool** | **按需**——模型自己决定要不要去拿 | 「查订单状态」工具 |
| **检索 / 网络检索** | 从文件/库/网上取，让回答有依据 | RAG |

> `instructions` 可以是**静态字符串**，也可以是**接收上下文并输出字符串的动态函数**（动态函数正是"从 context 掏料、拼进系统提示"的常见手法）。

**取舍一句话**：**"永远有用" → instructions；"只对某些问题有用" → 工具/检索。**

---

## 七、会话（sessions）：内置的对话记忆

### 7.1 为什么要有它

多轮对话，以前每轮你得**手动**把历史塞回去（`result.to_input_list()`），或自己拼 messages，烦且易错。Session = **把这活儿交给 SDK**。

### 7.2 工作原理（三步）

1. **每次运行前**：Runner 自动**取该会话历史**，prepend 到输入项之前。
2. **每次运行后**：运行期间生成的所有**新条目**（用户输入、助手响应、工具调用等）自动**存入**会话。
3. **上下文保留**：用**同一个会话**的后续运行会带上完整历史，于是模型"记得"。

```python
from agents import Agent, Runner, SQLiteSession

agent = Agent(name="Assistant", instructions="Reply very concisely.")
session = SQLiteSession("conversation_123")

r1 = await Runner.run(agent, "金门大桥在哪个城市?", session=session)   # San Francisco
r2 = await Runner.run(agent, "它在哪个州?",       session=session)   # California ← 记住了
```

### 7.3 "五种后端" = 4 种存储 + 1 层加密壳

> ⚠️ 关键框架：**SDK 只定义一个接口，这几家是同接口的不同实现。** 接口规定 **4 个动作**（见 7.4），谁实现这 4 个动作谁就能当 session 用。**SDK 不关心后端是谁。**

| 后端 | 实现的是 | 存哪 | 场景 |
|---|---|---|---|
| 不给 session（默认） | —— | —— | 单轮，无内存 |
| `SQLiteSession("id")` | 存储 | 进程内存 | 临时，进程结束就没 |
| `SQLiteSession("id", "x.db")` | 存储 | 本地文件 | 单机持久 |
| `SQLAlchemySession` | 存储 | 任意 DB（PG/MySQL/SQLite…） | 已有生产库 |
| `OpenAIConversationsSession` | 存储 | OpenAI 托管 | 已依赖 OpenAI 基础设施 |
| **`EncryptedSession`** | **包装器** | **套在任意 session 外面** | 加密 + 基于 TTL 的自动过期 |

`EncryptedSession` 特别说明：它**不是独立存储**，而是**装饰**——`EncryptedSession(session_id, underlying_session, encryption_key, ttl=600)`，在读写前后加一层透明加密（HKDF 派生密钥 + Fernet）与 TTL 过期；需要 `pip install openai-agents[encrypt]`。

### 7.4 四个内存操作

```python
session = SQLiteSession("user_123", "conversations.db")

items = await session.get_items()          # 取全部历史
await session.add_items([...])             # 加新条目
last = await session.pop_item()            # 弹掉并返回最后一条
await session.clear_session()             # 清空
```

**`pop_item` 的妙用——纠正/撤销上一轮**：用户想改上一轮提问时，先 pop 掉「助手回复」、再 pop 掉「用户问题」，然后重新问：

```python
assistant_item = await session.pop_item()   # 删掉助手的回答
user_item      = await session.pop_item()   # 删掉用户的问题
result = await Runner.run(agent, "What's 2 + 3?", session=session)   # 重新问正确的
```

### 7.5 几条实务要点

- **session_id 命名**：`"user_12345"`（基于用户）/ `"thread_abc123"`（基于线程）/ `"support_ticket_456"`（基于上下文）——**起个有意义的名字**。
- **不同 agent 可共享同一个 session**（`Support` 和 `Billing` 两 agent 看到同一份历史）。
- **自定义后端**：实现 `SessionABC`（`get_items/add_items/pop_item/clear_session`）即可，其他生产系统（Redis、Django…）同理。
- **另有两条延续对话的路**（和 session 并列）：`conversation_id`、`previous_response_id`（OpenAI Responses API 的玩法）——先记个目录，讲到 results / running_agents 时再展开。

---

## 八、易混辨析：`session` vs trace 的 `group_id`

这两个都能"把多轮串一起"，最容易混，其实**干的是两件完全不同的事**：

| | trace 的 `group_id`（笔记 06） | `session`（本章） |
|---|---|---|
| 谁在维护 | **你**（外部） | **SDK**（内部） |
| 作用 | 把多轮 trace **归到一起方便查** | 把对话历史**真存下来并喂回模型** |
| 影响模型看到啥 | **不影响** | **影响**（这就是它全部的意义） |
| 典型值 | 聊天线程 ID `"thread-001"` | `SQLiteSession("user_123")` |

> **一句话**：`group_id` 是给**你查 log** 用的；`session` 是给**模型记事**用的。

---

## 九、实际应用：`session_demo.py` 跑出来的三件事

可运行 demo：`E:\agent_study\code\session_demo.py`（配套一键脚本 `run_session.bat`）。**必须用 `py -3.13` 跑**（本机默认 python 3.8 装不了 openai-agents）。它跑出三件能"看见"的事：

**(1) session 传 vs 不传，天差地别**
```
不传 session：第2轮问"我叫什么名字？"→ 忘光了（每轮都是新的）
传  session  ：第2轮问"我叫什么名字？"→ "小明"  ← 记住了
```
再用 `await session.get_items()` **把 session 肚子里的东西摊开**——里面就是**一条条对话条目**（你一句、它一句），这就是"SDK 帮你存的历史"。

**(2) 工具 JSON Schema 暴露"暗格"**（全场最直观的一击）
```
check_order_status   模型可见参数 = ['order_no']   ← 纯参数
who_am_i             模型可见参数 = []             ← 纯 context
refund_order         模型可见参数 = ['reason']     ← 混合
```
`who_am_i` 明明从 context 读姓名，**模型可见参数却是空的**；`refund_order` 的 `wrapper` 参数在 schema 里**完全不存在**。这就是"暗格"的物理证据。

**(3) 跑完看 `ctx.refunded_orders`**
用户只说"**退我这一单**"（**没提订单号**），模型也拿不到订单号，退款照样办成 `A20315`：
```
   ['A20315']
   ↑ 订单号全程来自 context，模型既没看到也没传过它
```

---

## 十、易混概念辨析（汇总表）

### 10.1 工具签名的两类参数（本章最实用的一张表）

每个 `@function_tool` 签名的参数，本质分两类：

| 参数种类 | 注解长啥样 | 谁提供 | 模型看得见吗 |
|---|---|---|---|
| **特殊参数** | `RunContextWrapper[T]` / `ToolContext[T]` | **SDK/代码**自动注入 | ❌ 看不见（不进 schema） |
| **普通参数** | 其它任何类型（`str`、`int`…） | **模型** | ✅ 看得见（进 schema） |

**硬性规矩**（源码 `function_schema.py` 实锤）：context 参数**必须是第一个参数，且只能有一个**；放别的位置直接报错。

**三种工具形态** = 这两类参数的组合：

| 形态 | 签名 | 效果 |
|---|---|---|
| 纯 context | 只有 `RunContextWrapper[T]` | 模型可见参数为空（俗称"无参"） |
| 纯参数 | 只有普通类型参数 | 模型全权提供 |
| **混合** | 特殊 + 普通**同时有** | 两个都走（如 `refund_order(wrapper, reason)`） |

> ⚠️ **"纯 context"不是真的没参数**——它有一个参数，只是**特殊**（自动注入 + 模型看不见）。准确说：**模型可见参数为空**。

### 10.2 "决策谁来做"：拆成两层（静态 vs 动态）

| | **格子怎么划**（哪些是 context、哪些是参数） | **格子里填什么值** |
|---|---|---|
| 何时定 | **写代码时（静态）** | **运行时** |
| 谁定 | **你（开发者）** | context 格 → **你的代码**；参数格 → **模型** |
| 模型能碰吗 | **完全不能** | 参数格能碰；context 格碰不到 |

> **划格子是设计期、归你；填值是运行期——参数格归模型、context 格归代码。** 上一轮模型的结果**只能当"值"进来**，而且必须是**你的代码**亲手搬进某一格（它没法自己决定进 context）。
> 类比：**模型 = 填格子的人**（只有参数格那支笔）；**你 = 画图纸的人**（格子有几个、哪些是暗格，都是你定的）；**模型永远拿不到笔去改图纸**。

### 10.3 其他易混对

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **本地上下文** | **LLM 上下文** | 前者给代码、模型看不见（口袋）；后者给模型（桌上的纸） |
| `RunContextWrapper` | `ToolContext` | 后者是前者的子类，多了 `tool_name`/`tool_call_id`/`tool_arguments`（工具级元数据） |
| context（口袋） | 工具参数（明面） | 前者=模型不该看的、代码提供；后者=模型该给的、模型生成；**可同时存在于一个工具** |
| `.context` | `.usage`/`.tool_input` | 前者是**你的**；后两者是 SDK 管理的运行时元数据 |
| `session_id` | 后端 backend | 前者="**哪段对话**"（数据层）；后者="这段对话**存在哪**"（基建层）；同 id 换后端=历史空 |
| `SQLiteSession` | `EncryptedSession` | 前者是**存储**；后者是**包装器**，套在任意 session 外做加密+TTL |
| `session` | `group_id` | 前者=SDK 真记事（影响模型）；后者=你归 trace（不影响模型） |

---

## 十一、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | 泛型标记 `Agent[UserInfo]` 是做什么的 | **给类型检查器/IDE 看的静态标注，不改变运行时行为**。`TContext` 出现在 agent 的 tools/handoffs/guardrails/hooks 上，`Agent[X]` 把它们**一次性锁定成 X**，于是能抓"工具上下文类型与 agent 不符"等错，并让 IDE 补全 `.context.`。默认 `Any`，可省略 |
| 2 | "五种后端"是靠谁判断本次该用哪种存放 | **你，在写代码时（构造那一行）定的**，是**开发期静态选择**，无运行时自动判断。SDK 只定义一个接口（4 个动作），拿到你给的对象就照接口调，**不关心后端是谁**。想"按条件选后端"是你自己应用层的 `if/else` |
| 3 | context 里装什么 | 四类：①请求级数据（user_id/order_no）②依赖对象（db/logger）③辅助函数（can_refund）④可变状态（actions_taken）。凡「**模型不该看、但代码要用**」的都放它。**不会发给 LLM** |
| 4 | 是不是"不需要模型判断的走无参工具(context)，需要模型判断的走工具参数" | 方向对，但要把**两个维度**掰正：①context 的读者**不限于工具**（钩子/护栏/动态 instructions 也读）；②参数与 context **不互斥**，一个工具**可同时**有（`refund_order(wrapper, reason)`）；③"有参数"≠"没 context"。**决策标准是"每条数据该谁提供"**，不是"这个工具有没有参" |
| 5 | 那这个"放哪一格"的决策是谁做的 | **写代码时你定的（静态）**，模型全程不参与——它连 context 参数的存在都看不到。模型运行时只能决定**调不调工具**和**普通参数填什么值**。上一轮模型结果**只能作为"值"、且由你的代码搬运**才能进 context |
| 6 | 三种形态是不是"纯 context / 纯参数 / 混合都有" | ✅ 对。三类参数组合三种形态。补充机制真相：context 那个参数是**特殊参数**（自动注入、不进 schema、**必须放第一位且只能一个**）；"纯 context"实为"**模型可见参数为空**" |

---

## 十二、和 tracing 的对照

| | tracing（笔记 06） | context / sessions（本章） |
|---|---|---|
| 角色 | 在旁边**录**（行车记录仪） | 运行前**递料 / 记事**（口袋 + 桌上的纸） |
| 关键词 | trace / span / `group_id` | `RunContextWrapper` / `session` |
| 串多轮的字段 | `group_id`（**给你查 log**，不影响模型） | `session`（**给模型记事**，影响模型） |
| 谁在用 | 你（事后排查、分析） | 代码（工具/钩子）+ 模型（对话历史） |
| 交汇点 | 本章 demo 里的 `who_am_i` / `refund_order` 被调用时，会同时产出 `function_span`；session 喂回的历史，也都在 tracing 视野内 | |

> **一句话串起来**：tracing 管「**它怎么过去的**」，context/sessions 管「**它该看到什么、该记住什么**」。

---

*上下文与会话篇（两个"上下文"的区分 / `RunContextWrapper` 与 `ToolContext` / 泛型标记 `Agent[TContext]` / LLM 上下文四条喂料路 / session 原理与五种后端 / 四个内存操作 / `session` vs `group_id` / 工具三形态与"谁决定"两层）**已完成**。*

*后续待学：running_agents、results、models。*