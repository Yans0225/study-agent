# OpenAI Agent 学习笔记 04（handoffs 进阶篇）

> 本文档是「系统性学习 Agent」第四阶段的概念整理，聚焦 **任务转移（handoffs）进阶**，核心是 `handoff()` 函数带来的三大武器：`on_handoff` / `input_type` / `input_filter`。
> 来源：OpenAI Agents SDK 官方中文文档「任务转移」页（handoffs.md）。
> 学习日期：2026-10-26 
> 衔接：笔记 02 已讲过交接的「地基」（`handoffs` 参数 / 交接伪装成工具 / `handoff_description`），本文档是其**进阶**。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [从「裸传 Agent」到「handoff() 函数」](#二从裸传-agent到handoff-函数)
3. [on_handoff：交接的「关卡」](#三on_handoff交接的关卡)
4. [input_type：交接时模型递的「纸条」](#四input_type交接时模型递的纸条)
5. [input_filter：交接的「筛子」](#五input_filter交接的筛子)
6. [推荐提示词](#六推荐提示词)
7. [易混概念辨析](#七易混概念辨析)
8. [问答记录（Q&A）](#八问答记录qa)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 任务转移 handoffs | https://openai.github.io/openai-agents-python/zh/handoffs/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/handoffs.md |

> ⚠️ 获取记录：本机直连 GitHub 不通（WebFetch 域名安全验证失败 / curl 超时 exit 28、35），最终经镜像 `https://ghproxy.net/https://raw.githubusercontent.com/...` 取到 handoffs.md 全文。此镜像可作为后续拉取官方文档的备用通道。

---

## 二、从「裸传 Agent」到「handoff() 函数」

### 先回忆地基（笔记 02 已学）

1. `handoffs=[子agent]` 声明「能转给谁」；
2. SDK 把每个交接对象**伪装成工具** `transfer_to_<name>`；
3. 模型点单 → Runner 识破是交接 → 换 `current_agent`、移交对话历史。

### 两种写法（同一套哲学）

```python
# 写法一：裸传 Agent（笔记 02 那种）
triage = Agent(name="Triage", handoffs=[billing_agent, refund_agent])

# 写法二：用 handoff() 包裹，需要定制时用
triage = Agent(name="Triage", handoffs=[
    billing_agent,                                          # 也能继续裸传
    handoff(refund_agent, on_handoff=..., input_type=...),  # 定制
])
```

> 这与笔记 03 的「`@tool` 自动挡 vs 手动构造 `FunctionTool`」是**同一套哲学**：裸传 = 自动挡（用默认 `transfer_to_<name>` + `handoff_description`）；`handoff()` = 手动挡（需要定制时用）。

### `handoff()` 函数的 8 个旋钮

| 参数 | 作用 |
|---|---|
| `agent` | 转给谁（唯一必填） |
| `tool_name_override` | 覆盖工具名（默认 `transfer_to_<name>`） |
| `tool_description_override` | 覆盖工具描述 |
| **`on_handoff`** | 交接发生那一刻自动触发的回调（关卡） |
| **`input_type`** | 让模型交接时「顺手填一张结构化纸条」 |
| **`input_filter`** | 筛选下一个 agent 能看到的历史 |
| `is_enabled` | 动态开/关这个交接（布尔或返回布尔的函数） |
| `nest_handoff_history` | 嵌套历史压缩（测试版，默认禁用） |

> 官方提醒：`handoff()` 辅助函数**总是**把控制权转移给你传入的那个 `agent`。若存在多个可能目标，应为**每个目标各注册一个交接**，让模型自己选——而**不要**自己写代码在调用时动态决定转给谁（那是自定义 `Handoff` 的领域，少用）。

---

## 三、on_handoff：交接的「关卡」

### 一句话定义

**`on_handoff` 是「交接」这个事件的生命周期钩子：模型已经决定要交接、但交接还没真正完成之前，SDK 会先调一下你写的这个函数。** 它是笔记 02 里 `hooks`（钩子）的一个特例——专门挂在「交接」这一个事件上。

### 它卡在哪个位置（时序最关键）

```
1. 模型点单  transfer_to_refund_agent   ← 决定要交接了
2. Runner 识破：这是交接，不是普通工具
3. ┌─ on_handoff 执行  ← 就卡在这里（关卡）
4. └─ 交接真正完成：换 current_agent、移交对话历史
5. 新 agent 接手开始回答
```

放在第 3 步有讲究——**前不着村后不着店**：

- 前面「交接已拍板」→ 所以此时**预取数据不会白取**（不会取完发现其实不转）；
- 后面「交接还没执行完」→ 所以此时**还能拦截**（不想转还来得及喊停）。

### 签名：固定两个位

```python
# 不带 input_type：只拿 ctx
async def on_handoff(ctx: RunContextWrapper[None]):
    ...

# 带 input_type：第二个参数是模型填的那张纸条
async def on_handoff(ctx: RunContextWrapper[None], input_data: EscalationData):
    ...
```

- 第一个参数 `ctx` —— 笔记 02 那个 `RunContextWrapper`「包装盒」，`ctx.context` 才是你塞进去的业务对象；
- 第二个参数 `input_data` —— **只有设了 `input_type` 时才有**，就是那张纸条。

### 它主要干两件事

**① 预取数据**（确定要交接后，抢在交接完成前，把下一步要用的数据拉出来）：

```python
async def on_handoff(ctx: RunContextWrapper[UserContext]):
    user_id = ctx.context.user_id
    await load_orders_into_cache(user_id)   # 异步预取，不等交接完
```

**② 授权检查 / 副作用**（交接前先看这人够不够格、顺带留档）：

```python
async def on_handoff(ctx: RunContextWrapper[UserContext]):
    user = ctx.context
    if not user.is_vip:
        raise ValueError("非 VIP 用户无权转人工")   # 抛异常 = 阻止交接
    # 正常走完（不抛）→ 放行，交接继续
```

### 特别容易踩的细节：想拦截必须「抛异常」，不是「return」

官方原文特意强调：**`on_handoff` 成功返回后，SDK 会继续执行交接。**

| 你在 on_handoff 里 | 结果 |
|---|---|
| 正常 `return`（哪怕 return 个错误信息） | SDK **照样完成交接**，拦不住 |
| `raise` 异常 | 交接被**拦下**，不转了 |

### 和 input_type 的关系

| 概念 | 是什么 |
|---|---|
| `on_handoff` | **钩子本身**（交接时的关卡），能独立使用 |
| `input_type` | **给这个钩子额外喂的数据**（模型现场填的纸条），服务于 on_handoff |

---

## 四、input_type：交接时模型递的「纸条」

### 本质

默认的交接工具是**无参数**的——模型只说「我要转给 refund_agent」，别的啥也不传。`input_type` 让模型在交接时**顺手附上一份结构化元数据**，比如转给「人工客服升级」时说明**为什么升级**：

```python
from pydantic import BaseModel
from agents import Agent, handoff, RunContextWrapper

class EscalationData(BaseModel):
    reason: str

async def on_handoff(ctx: RunContextWrapper[None], input_data: EscalationData):
    print(f"升级原因：{input_data.reason}")

handoff_obj = handoff(
    agent=escalation_agent,
    on_handoff=on_handoff,
    input_type=EscalationData,   # ← 交接工具的参数 schema
)
```

模型交接时填 `{"reason": "用户被重复扣款"}`，SDK 在本地校验后解析成 `EscalationData` 对象，塞给 `on_handoff`。

> **关键认知：这跟笔记 03 的 FunctionTool 参数 schema 是同一个「双向翻译」机制。** `input_type` 这个 Pydantic 模型 → 变成交接工具的 `parameters` 说明书 → 模型填 JSON → SDK 翻回对象。

### 三个「不要误会」（最容易踩的坑）

| 别以为 | 真相 |
|---|---|
| `input_type` 会替换下一个 agent 的主输入？ | ❌ 不会。下一个 agent 照样看到完整对话历史 |
| `input_type` 会拿 `reason` 去「分流」选目标？ | ❌ 不会。还是转给 `handoff()` 里写死的那个 agent |
| `input_type` 就是 `context` 参数？ | ❌ 两回事。`input_type` 是**模型现场决定的元数据**；`context` 是你**本地早就有的业务状态** |

### 纸条的落点：写给 `on_handoff`，不是写给下一个 agent

记住三个「写给谁」：

- ❌ **不写给下一个 agent**（下一个 agent 看不到它，还是照常看对话历史）
- ❌ **不写给用户**
- ✅ **写给 `on_handoff` 这个回调**——交接那一瞬间，你代码里的钩子

### 为什么这张纸条非要让模型来写

因为有些信息**只有模型在「决定交接的此刻」才知道，你写代码时根本猜不到**——每个用户情况都不一样：

- 升级**原因**（为什么生气、问题多严重）／**优先级**（急不急）／**摘要**（核心诉求）／**语言**

这些是模型在对话过程中**现场判断**出来的，不是你预先能写死的常量。

### 纸条拿去干什么

`on_handoff` 拿到 `input_data` 后，典型用途是**在交接那一刻留档 / 触发流程**：

```python
class EscalationData(BaseModel):
    reason: str
    priority: str = "normal"   # 模型顺带判断急不急

async def on_handoff(ctx, input_data: EscalationData):
    logger.info(f"升级到人工，原因={input_data.reason}")     # ① 记录/审计
    await ticket_system.create(reason=input_data.reason)     # ② 落库/写工单
    if input_data.priority == "high":
        await alert("有高优先级投诉升级")                     # ③ 高优先级告警
```

### 对比：没有这张纸条会怎样

| | 没有 `input_type` | 有 `input_type` |
|---|---|---|
| `on_handoff` 拿到什么 | 只有 `ctx`（你的业务状态），**不知道模型为什么交接** | `ctx` + `input_data`（模型现场填的原因/优先级） |
| 想记录「为什么升级」 | 只能事后去对话历史里翻、或再让模型总结一遍，麻烦且不结构化 | 交接瞬间就用结构化字段拿到，直接落库 |

### 「记录」这个词要精确：不止记录，还可「间接影响」

- 纸条**默认不进入下一个 agent 的输入流**，对交接后 agent 干活的**输入没有直接影响**（正确）；
- 但「记录」太窄：`on_handoff` 是个回调，**拿它想干嘛干嘛**（写日志/落工单/发告警/**预取数据**），记录只是最典型一种；
- **间接影响是自己可以主动造的**：若 `on_handoff` 故意把纸条内容写进 `ctx.context` 或数据库，而下一个 agent 又恰好读了那个 context/数据库，就会间接影响——但这是**你写的副作用**，不是 `input_type` 机制的默认行为（默认行为只有一条：纸条递到 `on_handoff` 为止）。

### 类比：前台转接电话

- 电话（对话）→ 照样转给下一个客服（agent），内容完整传过去，纸条**不改变**这个；
- 纸条（`input_data`）→ 前台在转接的同时，往**内部工单系统**填一行「客户投诉重复扣款、情绪激动、高优先级」。这张纸条**不跟电话一起给下一个人**，而是给内部系统**留档/触发流程**。

> 一句话：`input_type` 让「交接」从「只换人、不留痕」升级成「换人的同时，让模型顺手交你一份**结构化的交接单**」。这张单子喂给 `on_handoff`，你就能在交接那一刻**记录、落库、告警**。

---

## 五、input_filter：交接的「筛子」

### 默认行为（这是 input_filter 要「改」的对象）

**交接时，新 agent 如同「接管了对话」，能看到之前的完整对话历史。** 前一个 agent 跟用户聊的每一句、每一次工具调用，默认**原封不动**全塞给下一个 agent。`input_filter` 就是**改掉这个默认**：交接之前先过一道你的筛选函数。

### 核心：`HandoffInputData` 的五个字段

筛选函数收到的就是这样一个「交接输入包」，里面装着对话的**不同段落**：

| 字段 | 对应对话的哪一段 |
|---|---|
| `input_history` | `Runner.run(...)` 启动**之前**就有的历史 |
| `pre_handoff_items` | 发起交接的那个 agent **轮次之前**生成的项目 |
| `new_items` | **当前轮次**生成的项目（含交接调用本身、交接输出） |
| `input_items` | 可选——用来**代替 `new_items`** 转发给下一个 agent，让你筛选模型输入，又不动 `new_items` 本身（`new_items` 仍留作会话记录） |
| `run_context` | 交接那一刻的 `RunContextWrapper` |

顺着时间轴看，就是一段对话被切成三段，你可以在「转交」这个口子上做手脚：

```
[run 前历史] ─▶ [交接 agent 之前的轮次] ─▶ [当前轮次：含交接调用] ─▶ 下一个 agent
   input_history        pre_handoff_items           new_items
                                                     │  ← input_items 可代替它转发
```

### 现成的筛子：`remove_all_tools`

最常用场景是「交接时把一堆工具调用的中间痕迹抹掉，只留干净对话」：

```python
from agents import Agent, handoff
from agents.extensions import handoff_filters

agent = Agent(name="FAQ agent")

handoff_obj = handoff(
    agent=agent,
    input_filter=handoff_filters.remove_all_tools,   # 转过去时自动移除所有工具相关项目
)
```

### 关键概念：什么叫「结构化工具项目」（这才能看懂 remove_all_tools 删了什么）

对话历史不是一整段文字，而是**一串有序的项目（item）**，混着两种性质完全不同的东西：

| 类型 | 是什么 | 例子 |
|---|---|---|
| **自然语言消息**（message item） | 人话——用户说的、助手回的 | 「北京天气怎么样？」「今天晴，25 度」 |
| **结构化工具项目**（structured tool item） | **不是人话**，是有字段的结构记录 | 工具调用请求、工具返回结果 |

**「结构化」= 有固定字段、不是句子**（像菜单上的一行 `[菜名=鱼香肉丝, 辣度=中辣]`，而非「我要一份鱼香肉丝中辣」这种人话）。

结构化工具项目的真面目是两个东西：

1. **工具调用请求**（`function_call`）——模型说「我要调 `get_weather`，参数 `{city: 北京}`」；
2. **工具返回结果**（`function_call_output`）——工具执行完回的结果，对应 call_id，output= sunny。

### 一个具体例子（一眼看清删了什么）

| 序号 | 对话内容 | 项目类型 | `remove_all_tools` 之后 |
|---|---|---|---|
| 1 | 用户：「北京天气怎么样？」 | 自然语言消息 | ✅ 留下 |
| 2 | 模型：`get_weather(city="北京")` | **结构化工具项目** | ❌ 删掉 |
| 3 | 工具返回：`sunny, 25°C` | **结构化工具项目** | ❌ 删掉 |
| 4 | 模型：「北京今天晴，25 度」 | 自然语言消息 | ✅ 留下 |

筛选后，下一个 agent 只看到：

```
用户：北京天气怎么样？
模型：北京今天晴，25 度
```

中间「我要调 get_weather」「工具返回 sunny」这些**结构化操作痕迹被抹掉**，但对话的「人话」完整保留。

### 文档的警告：删不掉「已抄成文字的副本」

如果工具内容**被模型用自然语言复述**进了某条消息（如「我**调用天气工具**查到，北京今天晴」），那这句是**自然语言消息**——`remove_all_tools` 只认「项目类型」这个标记去删，**不认识「调用天气工具」这七个字是工具痕迹**，于是原样留下。同理，`nest_handoff_history` 生成的「对话摘要」也是自然语言，里面的工具内容同样删不掉。

> 所以文档那句提醒的意思是：**`remove_all_tools` 是按「项目类型」删的，不是按「内容」删的**。工具信息一旦被抄成文字，就逃过这张筛子。

### 类比：会议记录的两种条目

- **发言记录**（自然语言消息）
- **操作流水便签**（结构化工具项目）

`remove_all_tools` = **把所有「操作流水便签」撕掉，只留「发言记录」**。但如果有人在发言里**口头复述**了操作内容，那段是发言，撕不掉。

### 什么时候需要 input_filter

| 场景 | 为什么筛 |
|---|---|
| **隐私 / 权限隔离** | 不想把某个 agent 聊出的敏感内容传给下一个 agent |
| **简洁** | 下一个 agent 只需看结论，不用看一堆工具调用过程 |
| **职责隔离** | 每个 agent 只该看到跟自己相关的那部分 |

### 自定义筛选器（示意）

思路一句话：**收下 `HandoffInputData`，改掉你想改的字段，返回新的**：

```python
def keep_clean_history(data: HandoffInputData) -> HandoffInputData:
    # 用 input_items 覆盖转发内容：只保留纯对话，剔除工具痕迹
    data.input_items = [item for item in data.new_items if not 是工具项目(item)]
    return data
```

> 进阶（测试版、默认禁用，了解即可）：`RunConfig.nest_handoff_history` 可把历史压缩成有序的助手摘要片段（用 `<CONVERSATION HISTORY>` 包装），`RunConfig.handoff_history_mapper` 可自定义映射函数。单次交接可通过 `handoff(..., nest_handoff_history=True/False)` 覆盖。**它只改变对话记录的「表示方式」，不负责删敏**——工具参数/输出仍可能残留在摘要里，要真正删敏得用显式 `input_filter`。

---

## 六、推荐提示词

官方给了个「交接提示词前缀」`RECOMMENDED_PROMPT_PREFIX`，建议塞进每个 agent 的 `instructions` 里，帮模型更稳地学会「该转就转、该接就接」：

```python
from agents import Agent
from agents.extensions.handoff_prompt import RECOMMENDED_PROMPT_PREFIX

billing_agent = Agent(
    name="Billing agent",
    instructions=f"""{RECOMMENDED_PROMPT_PREFIX}
    <在这里填你自己的提示词>""",
)
```

---

## 七、易混概念辨析

### ① handoffs 三大武器：关卡 / 纸条 / 筛子

| 武器 | 解决什么 | 一句话 |
|---|---|---|
| `on_handoff` | 交接那一刻做点事 | 交接的**关卡**（预取数据 / 授权检查） |
| `input_type` | 交接时让模型多交代一句 | 交接的**纸条**（喂给 on_handoff） |
| `input_filter` | 控制下一个 agent 看到什么 | 交接的**筛子**（改历史） |

### ② 想干不同的「交接时的事」→ 用不同武器（别混）

| 你想做什么 | 用哪个 |
|---|---|
| 让模型交接时**顺手附一份结构化元数据**（原因/优先级） | `input_type` |
| 在交接那一刻**做点副作用**（记录/落库/预取） | `on_handoff` |
| 改变**下一个 agent 看到什么历史** | `input_filter` |

### ③ `input_type` 会「分流」选目标吗？—— 不会

`input_type` 只是给**已选定的那个目标**附一段元数据，**不参与「选哪个目标」**。想「多个目标让模型自己挑」= 为每个目标各注册一个交接；想「不换对话只给结构化输入给子 agent」= 用 `Agent.as_tool(parameters=...)`（笔记 03 讲过）。

### ④ `input_type` vs `context`（呼应笔记 02 的「context 三义」）

| | `input_type` | SDK 的 `context` |
|---|---|---|
| 是谁决定的 | **模型**交接触发的元数据 | **你**本地塞的业务状态/依赖 |
| 何时产生 | 交接那一刻，模型现场生成 | run 启动前你就放好了 |
| 去哪 | 喂给 `on_handoff` | 随整次 run 传播，供各环节代码读 |

### ⑤ 「纸条」vs「筛子」（都跟「历史」有关，但删的是不同东西）

- **纸条（`input_type`）**：**加**信息——给 `on_handoff` 补一段模型现场生成的元数据，**不碰对话历史**；
- **筛子（`input_filter`）**：**改**信息——删/改下一个 agent 能看到的对话历史。

### ⑥ 结构化工具项目 vs 自然语言消息（读懂 remove_all_tools 的钥匙）

| | 是什么 | remove_all_tools 对它的态度 |
|---|---|---|
| 自然语言消息 | 人话（用户说的、助手回的） | 留下 |
| 结构化工具项目 | 工具调用请求 / 工具结果，字段化、非句子 | 删掉 |

> 关键：`remove_all_tools` 按**项目类型**删，不按**内容**删——工具信息一旦被抄成自然语言文字，就删不掉了。

---

## 八、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | handoffs 里「裸传 Agent」和 `handoff()` 函数有什么区别 | 裸传＝自动挡（用默认 `transfer_to_<name>` + `handoff_description`）；`handoff()`＝手动挡（8 个旋钮可定制）。与笔记 03 的「`@tool` vs 手动构造 FunctionTool」同一套哲学 |
| 2 | `on_handoff` 是做什么的（详细） | 交接事件的**关卡/钩子**：模型已定要交接、但交接未完成前触发。卡在时序第 3 步（前不着村后不着店）：可**预取数据**（不会白取）、可**拦截**。签名 `(ctx, [input_data])`。**想拦必须 `raise` 异常，不能 `return`** |
| 3 | `input_type` 那张「纸条」的作用是什么 | 让模型交接时**顺手附一份结构化元数据**（原因/优先级/摘要/语言），喂给 `on_handoff`。纸条**只写给 on_handoff**，不给下一个 agent、不给用户。机制同笔记 03 的 FunctionTool 参数 schema（双向翻译） |
| 4 | 纸条是不是只起记录作用、对交接后的 agent 没影响 | 核心对：纸条默认**不进入**下一个 agent 的输入流，对交接后 agent 的**输入无直接影响**。但「记录」太窄——on_handoff 可做任何副作用（记录/落库/告警/预取）；且若 on_handoff 主动把纸条写进 context/数据库而下个 agent 又读了，会**间接影响**（那是你自己写的副作用，非机制默认） |
| 5 | `remove_all_tools` 移除的「结构化工具项目」是什么，到底删了什么 | 对话历史＝一串 item，分两类：**自然语言消息**（人话，保留）与**结构化工具项目**（`function_call` 工具调用请求 + `function_call_output` 工具结果，字段化非句子，删掉）。它是**按项目类型删、不按内容删**——工具信息若被抄成自然语言文字（如摘要），删不掉 |
| 6 | `input_type` / `input_filter` / `on_handoff` 三者怎么区分 | 关卡（on_handoff＝交接时做事）/ 纸条（input_type＝给 on_handoff 喂模型生成的元数据）/ 筛子（input_filter＝改下一个 agent 看到的历史）。想「选目标」都不该用 input_type，应各注册一个交接，或用 `as_tool(parameters=...)` |
| 7 | `input_type` 是什么，和 context 有何区别 | input_type＝模型交接触发现场生成的元数据，喂给 on_handoff；context＝run 启动前你本地塞的业务状态，随 run 传播供代码读。二者独立 |

---

*handoffs 进阶篇（handoff() 函数 / on_handoff 关卡 / input_type 纸条 / input_filter 筛子 / 推荐提示词）已学完。*

*遗留待办：① 笔记 02 旧「工具四来源」表已在本次更新为「五类工具」✅；② 下一步待学：guardrails、tracing、context/sessions、running_agents、results、models。*