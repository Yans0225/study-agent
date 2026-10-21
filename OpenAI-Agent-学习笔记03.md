# OpenAI Agent 学习笔记 03（工具篇）

> 本文档是「系统性学习 Agent」第三阶段的概念整理，聚焦 **工具（tools）**，核心是 `FunctionTool` 的机制与手动构造。
> 来源：OpenAI Agents SDK 官方中文文档「工具」页（tools.md）。
> 学习日期：2026-10-08 

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [工具的五类总览（重要更新）](#二工具的五类总览重要更新)
3. [FunctionTool 的核心机制](#三functiontool-的核心机制)
4. [双向翻译三步走（核心）](#四双向翻译三步走核心)
5. [JSON schema 是什么](#五json-schema-是什么)
6. [手动构造 FunctionTool（四零件）](#六手动构造-functiontool四零件)
7. [参数约束（Pydantic Field）](#七参数约束pydantic-field)
8. [超时与报错处理](#八超时与报错处理)
9. [托管工具与 agents as tools](#九托管工具与-agents-as-tools)
10. [易混概念辨析](#十易混概念辨析)
11. [问答记录（Q&A）](#十一问答记录qa)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 工具 tools | https://openai.github.io/openai-agents-python/zh/tools/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/tools.md |

> 源码 Markdown 可经 `raw.githubusercontent.com` 直连获取；本文档的字段/概念据此逐条核对。

---

## 二、工具的五类总览（重要更新）

官方文档现在把工具分成**五类**。

| # | 类别 | 一句话 |
|---|---|---|
| 1 | 托管工具（Hosted Tools） | OpenAI 服务器上替你跑（联网搜索、文件检索、代码解释、图像生成等） |
| 2 | 本地 / 运行时工具 | `ComputerTool` / `ApplyPatchTool` / `ShellTool`，在你自己的环境跑 |
| 3 | `FunctionTool` 实例 | 把任意 Python 函数包装成工具 |
| 4 | Agents as tools | 把一个 agent 暴露成可调用工具 |
| 5 | 实验性 Codex 工具 | 通过工具调用跑一个限定在工作区的 Codex 任务 |

**变化的本质**：旧「四来源」表把「MCP 工具」单独算一类；新版把 MCP 拆散了——托管 MCP 归进「托管工具」，本地 MCP 归进「运行时工具」，不再单列。同时补上了 `ComputerTool`/`ShellTool` 这类「能操作电脑/改文件」的强操作工具，以及实验性的 Codex。


---

## 三、FunctionTool 的核心机制

这是工具这一章最核心的「根」——其它四类工具落到模型眼里的形态**都一样**，都是一份「工具清单条目」。看懂 FunctionTool，就等于看懂了工具的通用机制。

### 一个衔接前文的心智模型

模型只会「回话 / 要工具」两种表达，工具是它对外交互的唯一接口。而 FunctionTool 是看懂「要工具到底怎么落地」的最佳切片。

### `@tool` 装饰器自动做的四件事

```python
from agents import Agent, FunctionTool, RunContextWrapper
from agents.decorators import tool

class Location:
    lat: float
    long: float

@tool
async def fetch_weather(location: Location) -> str:
    """Fetch the weather for a given location.

    Args:
        location: The location to fetch the weather for.
    """
    return "sunny"
```

| SDK 用 `@tool` 自动做 | 从函数身上哪里读 |
|---|---|
| 工具**名字** | 函数名 `fetch_weather`（可用 `name_override=` 改） |
| 工具**描述** | 函数 docstring（可用参数覆盖） |
| 参数**schema** | 函数签名 + 类型注解（`inspect` 提取） |
| 每个参数的描述 | docstring 的 `Args:` 部分（`griffe` 解析） |

底层技术栈：`inspect` 读签名 → `griffe` 读 docstring → `pydantic` 生成 schema。

> 关键认知：**函数本体（`return "sunny"` 那部分）永远留本地，模型从头到尾看不到**。模型看到的只有「名字 + 描述 + schema」三样。这落地了 01 笔记里「`tools` 给模型看的 = 名称/参数/说明的 JSON schema，函数本体留本地」那条铁律。

### docstring 为什么重要

docstring 是**模型唯一能看到的、关于「这个工具是干嘛的 / 怎么用」的文字说明**。它贡献两处：① 顶部摘要 → 工具 description；② `Args:` 部分 → 各参数 description。模型靠它判断「该不该调、参数怎么填」，写得含糊模型就会调错或乱填。

### Location 参数为什么是 schema 而不是普通传参

因为**模型的语言边界是 JSON，不是 Python 对象**——它造不出一个真的 `Location` 对象，只能用 JSON 说「我要传这些数据」。所以参数在模型这边只能以「schema（说明书）」的形式存在。

---

## 四、双向翻译三步走（核心）

同一个 `location` 参数，在三个阶段长三张不同的「脸」：

| 时刻 | 它长什么样 | 属于谁的世界 |
|---|---|---|
| 你写代码时 | `location: Location`（一个类） | Python |
| 模型看说明书时 | `{"type":"object","properties":{lat,long}}`（schema） | JSON |
| 模型点单时 | `{"location":{"lat":39.9,"long":116.4}}`（值） | JSON |
| 函数执行时 | `Location(lat=39.9, long=116.4)`（对象） | Python |

**「双向」是两个箭头**：

```
Python 类型  ──翻出去──▶  JSON schema（模型看）    ← 第一向（pydantic 生成 schema）
JSON 值     ──翻回来──▶  Python 对象（函数跑）    ← 第二向（解析回对象）
```

### 学习者自己总结出的闭环（一句顶三句）

> SDK 根据我写的 Python 函数翻译出一份 JSON schema 作说明书 → 模型根据这份说明书填写数据 → SDK 再把这份数据转为参数执行函数。

**深层好处**：模型从头到尾没碰过你的 Python 代码，只跟 JSON 打交道，函数本体安全留在本地。

### 点菜类比

| 角色 | 对应 |
|---|---|
| 后厨大厨（只会中文） | Python 开发者 |
| 外国客人（只会英文菜单） | 模型（只会 JSON） |
| 翻译官 / 服务员 | SDK |
| 英文菜单 | JSON schema |
| 客人勾好的点单 | JSON 数据 |

---

## 五、JSON schema 是什么

**一句话**：JSON schema = 用 JSON 写的一份「说明书」，规定另一个 JSON 数据应该长什么样。

| | 是什么 |
|---|---|
| JSON | 数据本身（一段有值的内容） |
| JSON schema | 对数据的「规格描述」（有哪些字段、各是什么类型、哪些必填） |

三个关键字：

| 关键字 | 意思 |
|---|---|
| `"type": "object"` | 整体是个对象 |
| `properties` | 里面有哪些字段、各是什么类型 |
| `required` | 哪些字段必填 |

> 类比：菜单（列了菜名/辣度）是 schema，勾好的点单（「鱼香肉丝，中辣」）是数据。规矩（schema）在前，填值（数据）在后。

---

## 六、手动构造 FunctionTool（四零件）

不靠 `@tool` 装饰器，自己亲手拼出四个零件，再用 `FunctionTool(...)` 组装。

### 官方完整例子

```python
from pydantic import BaseModel
from agents import RunContextWrapper, FunctionTool


def do_some_work(data: str) -> str:
    return "done"


class FunctionArgs(BaseModel):
    username: str
    age: int


async def run_function(ctx: RunContextWrapper[Any], args: str) -> str:
    parsed = FunctionArgs.model_validate_json(args)
    return do_some_work(data=f"{parsed.username} is {parsed.age} years old")


tool = FunctionTool(
    name="process_user",
    description="Processes extracted user data",
    params_json_schema=FunctionArgs.model_json_schema(),
    on_invoke_tool=run_function,
)
```

### 四个零件拆解

| 零件 | 它是什么 | 作用 |
|---|---|---|
| ① `FunctionArgs` | 参数模具（`BaseModel`） | 声明「本工具要收 username + age 两个参数」，相当于 `Location` 那个模具 |
| ② `do_some_work` | 真正干活的函数 | 最底层业务逻辑（官方用 `return "done"` 占位） |
| ③ `run_function` | 翻译官 | 把模型的 JSON 翻译回 Python 对象，再转发给②干活 |
| ④ `FunctionTool(...)` | 组装 | 用①生成 schema、登记③为执行函数 |

### 要填四格（对照 `@tool` 自动来源）

| 格 | 是什么 | 对应 `@tool` 自动从哪来 |
|---|---|---|
| `name` | 工具名 | 函数名 |
| `description` | 工具描述 | docstring 摘要 |
| `params_json_schema` | 参数说明书 | 类型注解（pydantic 生成） |
| `on_invoke_tool` | 执行函数 | 函数本体 |

### 最关键的差别：`on_invoke_tool` 收到的是 JSON 字符串

```python
async def run_function(ctx, args: str) -> str:   # args 是 str
    parsed = FunctionArgs.model_validate_json(args)  # 自己翻译
```

手动模式下，执行函数收到的第 2 个参数 `args` 是 **`str`（一串 JSON 文本）**，得自己 `model_validate_json` 翻译回对象。**这一步 `@tool` 装饰器是自动帮你做的**；手动构造时它原样把 `str` 甩给你。

### `@tool` vs 手动构造

| | `@tool` 装饰器 | 手动构造 `FunctionTool` |
|---|---|---|
| 填四格 | SDK 自动填 | 你亲手填 |
| 参数 schema | 从类型注解自动生成 | 你给 Pydantic 模型，`model_json_schema()` |
| 执行函数拿到什么 | 已解析好的 Python 对象 | 一串 JSON `str`，自己 `model_validate_json` |
| 什么时候用 | 绝大多数情况 | 想完全自控、或逻辑不是「套一个函数」时 |

> 一句话：`@tool` 是自动挡，手动构造是手动挡。手动挡没多出能力，只是把「读签名 / 读 docstring / 转 schema / 解析参数」全交到你手上。日常用自动挡就够。

### pydantic 三个生面孔

| 术语 | 作用 |
|---|---|
| `BaseModel` | pydantic 里「定义数据结构」的底座 |
| `.model_json_schema()` | 把结构生成为 JSON schema（说明书） |
| `.model_validate_json(json字符串)` | 把一串 JSON 文本校验并翻译成类的真对象 |

---

## 七、参数约束（Pydantic `Field`）

### 为什么要约束

前面 `Location(lat, long)` 的参数说明只有「类型」。想进一步告诉模型「取值范围 / 长度 / 格式」，就用 Pydantic 的 `Field` 加**约束 + 描述**。官方例子：

```python
from typing import Annotated
from pydantic import Field
from agents.decorators import tool

# 写法一：默认值形式
@tool
def score_a(score: int = Field(..., ge=0, le=100, description="Score from 0 to 100")) -> str:
    return f"Score recorded: {score}"

# 写法二：Annotated 形式
@tool
def score_b(score: Annotated[int, Field(..., ge=0, le=100, description="Score from 0 to 100")]) -> str:
    return f"Score recorded: {score}"
```

### 它干两件事（都很关键）

1. **写进 schema（喂给模型）**：约束被翻进 JSON schema，模型「看到」范围就**更可能填对**；
2. **运行期校验（拦住乱填）**：模型真填了 `105`，SDK 校验时**直接拦下**，不把脏数据塞进函数。

> `Field` 让「参数说明书」从「只有类型」升级成「类型 + 范围/长度/格式 + 说明」，既帮模型填对、又兜底拦错。

### 常用约束关键词

| 关键词 | 意思 |
|---|---|
| `ge` / `le` | 大于等于 / 小于等于 |
| `gt` / `lt` | 严格大于 / 严格小于 |
| `min_length` / `max_length` | 字符串最小 / 最大长度 |
| `pattern` | 正则匹配（如手机号格式） |
| `description` | 参数说明（也会进 schema） |

### 两种写法的区别

`score_a`（默认值形式）和 `score_b`（Annotated 形式）**效果完全一样**，只是语法糖：

- **默认值形式** `score: int = Field(...)`：把 `Field(...)` 当「默认值」之位来写，直观但有点「借默认值位置干别的事」的别扭；
- **Annotated 形式** `score: Annotated[int, Field(...)]`：用 `Annotated` 明确把「类型 `int`」和「约束 `Field`」打包，语义更干净，**更推荐**。

### `Field(...)` 里的「...」是什么（关键辨析）

- `...` 是 Python 的特殊对象 `Ellipsis`（省略号），填在 `Field` 的**第一个位置参数 = 默认值（default）**上。
- 传 `...` 是 pydantic 的约定暗号：**「这个字段没有默认值」**。
- 而「没默认值」在 pydantic 里就等于 **「必填（required）」**。

```
Field(...)   →  没默认值  →  必填
Field(5)     →  默认值 5  →  可选，缺省用 5
```

对比例子：

```python
class A(BaseModel):
    x: int = Field(..., ge=0)   # 必填
    y: int = Field(5, ge=0)     # 可选，缺省 5

A(x=3)        # ✅  x 传 3，y 用默认 5
A()           # ❌  x 必填但没传
A(x=3, y=9)   # ✅
```

**为什么普通参数（不写 Field）也必填**：pydantic 默认规则「没默认值 = 必填」，所以 `location: Location` 本来就必填；写 `Field(...)` 只为在「保持必填」的同时「加约束」。

---

## 八、超时与报错处理

工具跑起来，可能出两种事故：**「慢」（超时）** 和 **「崩」（抛异常）**。SDK 对两者各有一套兜底机制，核心哲学一致——**把事故翻译成一句「模型读得懂的话」，而不是让整个 run 直接 crash。**

### 1. 超时（timeout）：工具「慢」了怎么办

用 `@tool(timeout=...)` 设一个时限。到点工具还没跑完，SDK 按你指定的「应对方式」处理。共有**三种**应对方式（`timeout_behavior`）：

| 方式 | 后果 | 谁兜底 | 典型场景 |
|---|---|---|---|
| `error_as_result`（**默认**） | 不报错，把「超时了」当工具结果喂给模型 | SDK + 模型 | 日常服务，慢一点算了 |
| `raise_exception` | 抛 `ToolTimeoutError`，代码 `try/except` 接 | 你的代码 | 慢就是事故，要告警/重试/降级 |
| `timeout_error_function` | 调你写的函数，自定义「超时话术」给模型 | 你写的函数 | 想控制模型收到的超时说法 |

**方式一 `error_as_result`（默认）：**

```python
@tool(timeout=2.0)   # 不写 timeout_behavior，默认就是 error_as_result
async def slow_lookup(query: str) -> str:
    await asyncio.sleep(10)   # 故意慢，一定超时
    return f"Result for {query}"
```

超时后 SDK 不抛异常，而是生成一句结果塞给模型（如 `Tool 'slow_lookup' timed out after 2 seconds.`），模型看到就自己圆场回用户。体验不断线。

**方式二 `raise_exception`：**

```python
from agents import Runner, ToolTimeoutError

@tool(timeout=1.5, timeout_behavior="raise_exception")
async def slow_lookup(query: str) -> str:
    await asyncio.sleep(5)
    return "done"

try:
    await Runner.run(agent, "查天气")
except ToolTimeoutError as e:
    print(f"{e.tool_name} 超时了，等了 {e.timeout_seconds} 秒")
```

异常自带 `e.tool_name`（哪个工具超时）、`e.timeout_seconds`（阈值多少），代码据此告警/记日志/标失败。

**方式三 `timeout_error_function`（自定义话术）：**

```python
async def my_timeout_handler(context, agent, tool) -> str:
    return "⚠️ 你的查询超时了，请告诉用户稍后再试。"

@tool(timeout=2.0, timeout_error_function=my_timeout_handler)
async def slow_lookup(query: str) -> str:
    await asyncio.sleep(10)
    return "done"
```

区别只在：模型收到的「超时话术」是你写的那句，而不是 SDK 写死的英文。签名固定 `(context, agent, tool) -> str`。

**一句话**：三者只在「超时这事谁来接、怎么接」——方式一 SDK 悄悄转成一句话、方式二当场抛异常让代码接管、方式三让你自己写这句「话」。

---

### 2. 报错（失败）处理：工具「崩」了怎么办

工具自己 `raise` 异常时，用 `failure_error_function` 指定「把异常翻译成什么话给模型」。同样有三种取值：

| 取值 | 后果 |
|---|---|
| `default_tool_error_function`（**默认**） | SDK 把异常写进工具结果，模型看到「工具失败了」并附原因 |
| `failure_error_function=自定义函数` | 你写的函数收到 `(context, agent, tool, exception)`，自定义话术 |
| `failure_error_function=None` | 不兜底，异常直接往上冒泡 |

```python
@tool(failure_error_function=my_custom_error_function)
def get_user_profile(user_id: str) -> str:
    if user_id == "user_123":
        return "User profile retrieved."
    else:
        raise ValueError("API returned an error.")
```

> 与超时对称：超时是「慢」，报错是「崩」，两者都是「工具出事故时，把事故转成一句模型读得懂的话」。`failure_error_function` 与 `timeout_error_function` 是两个**互相独立**的钩子，各自管一种事故。

**关键区别：报错「不限同步异步」。** 见下节。

---

### 3. 为什么「超时」只能异步，「报错」折腾不限

| | 同步 `def` | 异步 `async def` |
|---|---|---|
| 超时 `timeout` | ❌ 只支持异步 | ✅ |
| 报错 `failure_error_function` | ✅ 都支持 | ✅ |

**超时要「中途强停」一个正在跑的函数，只有异步做得到**：

- 同步函数一进去就占住线程（如 `time.sleep(10)`），从头到尾不让位，SDK 喊「停」也插不进手。
- 异步函数沿途有 `await`，每遇到一个就「挂起、把控制权交还给事件循环」，SDK 借 `asyncio.wait_for(timeout=...)` 在时间表上登记一个「取消闹钟」，到点即可取消这个任务。

**报错不需要「打断」**：异常是函数**自己冒出来**的，不管同步异步都会往上冒，SDK 在调用处套个 `try/except` 就能接住，与同步/异步无关。

---

### 4. 异步函数到底怎么跑（深挖）

**三样东西配套出现**：

| 角色 | 是什么 |
|---|---|
| 事件循环（Event Loop） | 一个永不停歇的「调度员」，管理谁该跑、谁该等 |
| 协程（Coroutine） | 就是 `async def` 函数造出来的任务 |
| `await` | 协程里「暂停 + 让位」的信号 |

**一次运行的完整过程**（以两个任务 A、B 为例）：

1. A 先跑，撞到 `await` → 停住，标记「我在等 N 秒」，控制权交还事件循环。
2. 事件循环立刻切到排队的 B。
3. 两个都挂起时，事件循环基本在「干等，检查谁的闹钟响了」。
4. 谁的闹钟先响，就把谁从寄存处捞回来，从它 `await` 的**下一行**继续，跑完标「完成」。

> 全程**只有一个线程**，是同一根线上 A、B 交替「让位」前进，没有谁「同时」做。异步的快，来自「把等待的空档拿去干别的活」，不是「同时做」。

**关键：协作式，而非抢占式。**

| | 谁来打断谁 |
|---|---|
| 抢占式（操作系统调度线程） | 调度器随时强行暂停正在跑的任务 |
| 协作式（asyncio） | 只能被「自己」打断——主动 `await` 让位，别人才有机会 |

因此衍生的经典问题：**A 的闹钟响了，但 B 还在埋头算怎么办？**

- B 此刻**在等（已挂起）** → A 立刻醒来，无冲突；
- B 此刻**在埋头算（一大坨 CPU 循环，全程没 `await`）** → B 不让位，A 被堵住，只能等 B 算完。

> 类比：抢占式＝交警随时拦车；协作式＝只有让行手势，司机自己不让，谁也拿他没办法。同步函数就是「永远不让行的司机」，事件循环想取消它都没下手处。

---

## 九、托管工具与 agents as tools

这一大步讲「工具清单里另外两类」，核心心智不变——它们落到模型眼里，**依然是「名称 + 描述 + schema」一份工具清单条目**，变的只是「点单之后**谁去干活、跑在哪**」。

### 1. 托管工具（Hosted Tools）

**一句话抓本质**：`FunctionTool` 的「函数本体」留在你的本地，模型点单后**你的服务器跑你的 Python 代码**；托管工具反过来——**实现完全在 OpenAI 服务器上**，你一行实现都不写，只「声明我要用它」，点单后 **OpenAI 替你跑、把结果直接送回对话**。

> 类比：`FunctionTool`＝自己家厨房做菜；托管工具＝点外卖，你只下单，菜在别人餐厅做好送上门。

**官方 7 种托管工具：**

| 工具 | 干什么 |
|---|---|
| `WebSearchTool` | 联网搜索（可要文字，也能要图片） |
| `FileSearchTool` | 从你的 OpenAI 向量存储里检索文件信息 |
| `CodeInterpreterTool` | 让 LLM 在沙盒环境里执行代码 |
| `HostedMCPTool` | 把远程 MCP 服务器的工具暴露给模型 |
| `ImageGenerationTool` | 根据提示词生成图像 |
| `ToolSearchTool` | 模型按需加载「延迟加载」的工具 |
| `ProgrammaticToolCallingTool` | 模型通过生成 JS 代码来编排调用工具 |

> 注意：托管工具**只能用 `OpenAIResponsesModel`**（Responses API 模型）——这是它和 FunctionTool 的另一个区别。FunctionTool 是通用机制，什么模型都能配；托管工具是 OpenAI 服务器侧的「特权」。

**典型例子（联网 + 文件检索）：**

```python
from agents import Agent, FileSearchTool, Runner, WebSearchTool

agent = Agent(
    name="Assistant",
    tools=[
        WebSearchTool(
            search_content_types=["image", "text"],
            image_settings={"max_results": 3, "caption": True},
        ),
        FileSearchTool(
            max_num_results=3,
            vector_store_ids=["VECTOR_STORE_ID"],
        ),
    ],
)

result = await Runner.run(agent, "帮我找金门大桥日落的最新图片和文字")
print(result.final_output)
```

**「声明 vs 实现」对比**：你只传了「要几张图、要什么类型、指向哪个向量库」这些**配置**，完全没写 `def 搜索(...)` 这种实现。真正的搜索/检索逻辑在 OpenAI 那边，你既看不到也改不了。

---

### 2. agents as tools（把一个 agent 变成工具）

**一句话抓本质**：01 笔记里子 agent 要**交接（handoff）**才能被别的 agent 用；`as_tool` 给了另一种用法——**把一个现成的 agent 整体打包成一个「工具」，塞进另一个 agent 的 `tools` 列表**。另一个 agent 点它像点普通工具，只是「这个工具」背后是一整个 agent 在跑。

**经典例子（翻译编排器）：**

```python
spanish_agent = Agent(name="spanish_agent", instructions="把内容翻译成西班牙语")
french_agent  = Agent(name="french_agent",  instructions="把内容翻译成法语")

orchestrator_agent = Agent(
    name="orchestrator_agent",
    instructions="你是翻译编排员，用给你的工具翻译。要多种翻译就调用多个工具。",
    tools=[
        spanish_agent.as_tool(
            tool_name="translate_to_spanish",
            tool_description="翻译成西班牙语",
        ),
        french_agent.as_tool(
            tool_name="translate_to_french",
            tool_description="翻译成法语",
        ),
    ],
)

result = await Runner.run(orchestrator_agent, input="用西班牙语说'你好，你好吗？'")
print(result.final_output)
```

**核心**：`spanish_agent` / `french_agent` 原本是独立 agent，被 `as_tool()` 变成编排器手里的两个「工具」。编排器点它，SDK 底下会**真的跑起那个子 agent**，把它的最终输出当工具结果送回。

**两个可调项：**

1. **默认入参是一句话**：`as_tool()` 默认要模型传 `{"input": "要翻译的句子"}`（就一个 `input` 字段）。
2. **结构化入参用 `parameters`**：

```python
from pydantic import BaseModel, Field

class TranslationInput(BaseModel):
    text: str = Field(description="要翻译的文本")
    source: str = Field(description="源语言")
    target: str = Field(description="目标语言")

translator_tool = translator_agent.as_tool(
    tool_name="translate_text",
    tool_description="在语言之间翻译文本",
    parameters=TranslationInput,   # 用 Pydantic 模型定义结构化 schema
    include_input_schema=True,
)
```

> 这里前几章的「双向翻译 + Pydantic schema」全串起来了：`TranslationInput` 变成 schema 给模型看，模型填好 JSON，SDK 再翻回 `TranslationInput` 对象传给内部子 agent。

---

### 3. 三类工具「谁干活、跑在哪」对比

| | 实现是谁写的 | 跑在哪 | 你写不写实现 |
|---|---|---|---|
| `FunctionTool` | 你 | 你的服务器 | 写（`def` 函数体） |
| 托管工具（WebSearch 等） | OpenAI | OpenAI 服务器 | 不写，只声明 |
| `agent.as_tool()` | 你（复用已搭好的 agent） | 你的服务器（内部跑的是个 agent） | 不写新逻辑，复用 agent |

**一句话收束**：工具的「统一接口」（名称 + 描述 + schema）从没变过，变的只是「点单之后谁去干活」——可以是你的函数、OpenAI 的现成服务、或另一个 agent。

---

## 十、易混概念辨析

### ① JSON 文本为什么算 str

JSON 全称 **JavaScript Object Notation**，重点在 **Notation（记法/书写格式）**，不是 Object——它本质是「一套用来写数据的文字写法」，写出来的东西就是**一段文字**，在 Python 里就是 `str`。

```python
'{"username":"张三","age":42}'   # 有引号包着 → str（文字）
{"username":"张三","age":42}     # 没有引号 → dict（对象）
```

- 传输（模型吐字、网络传、写文件）只能搬「字符流」，搬不了活对象 → 所以从外界进来的数据，到你手里都已经是 `str`。
- 要取值必须先「读懂」它，`model_validate_json` / `json.loads` 就是干这事的。

### ② 「对象」 vs 「字典」

Python 哲学「一切皆对象」——对象是**统称**，字典只是其中一种。

| 值 | 是对象吗 |
|---|---|
| 整数 `42`、字符串 `"张三"`、列表、字典、自定义类实例 | ✅ 都是 |

- `json.loads(...)` → 得到 **dict（字典）**，用方括号 `d["username"]` 取值；
- `FunctionArgs.model_validate_json(...)` → 得到 **FunctionArgs 实例**，用点号 `p.username` 取值；
- 两者都是「对象」，但不是同一种。

> 在本教程语境里：手动构造里 `model_validate_json` 翻出来的对象是 `FunctionArgs` 类的实例，不是字典，所以用 `parsed.username`（点号）。

### ③ 超时 vs 报错（别搞混）

| | 超时（timeout） | 报错（failure） |
|---|---|---|
| 事故类型 | 工具「慢」了（规定时间内没跑完） | 工具「崩」了（自己抛异常） |
| 配置参数 | `timeout=` + `timeout_behavior=` | `failure_error_function=` |
| 抛什么 | 超时到点，SDK 主动放弃 | 函数自己 `raise`，异常冒上来 |
| 是否限异步 | **只支持异步** | 同步异步都支持 |
| 兜底钩子 | `timeout_error_function` | `failure_error_function`（同名对应） |

### ④ 服务器端一条龙 vs 本地 agent 循环

**判据：本地代码跟服务器「打几次交道」。**

| | 服务器端一条龙（OpenAI 托管工具） | 本地 agent 循环（Claude Code WebSearch 等） |
|---|---|---|
| 本地代码跟服务器 | **一次**——请求进、最终答案出 | **多次**——每小步一次往返 |
| 中间过程 | 服务器内部黑盒自动完成 | 本地程序一步步驱动 |
| 循环谁驱动 | 服务器（模型原生工具调用） | 本地程序（Runner 循环在本地） |

- **服务器端一条龙**：模型在「生成回答」的过程中原生调用工具，服务器自己调、自己继续生成，本地代码只在原地等最终答案。＝把 Runner 循环搬进了服务器内部。
- **本地 agent 循环**：模型每说一次「我要工具」，就结束这一轮、把「意图」返回；本地程序去执行、再当「工具结果」喂回模型。＝Runner 循环在本地转。

> 关键：这两种方式的「工具实现 + 执行动作」**都在服务器端**，真正差别在**「调→执行→喂回」这个循环谁驱动**。这也是 01 笔记「Runner 循环」放哪的区别。

### ⑤ 「托管工具」只有 OpenAI 有吗

「托管工具」这个词是 **OpenAI 专属叫法**，但「官方提供现成工具能力」是行业共识：

| 平台 | 对应物 |
|---|---|
| OpenAI | `WebSearchTool`/`FileSearchTool`/`CodeInterpreterTool` 等（托管工具） |
| Anthropic | server tools：`web_search`/`code_execution`（概念≈同款） |
| Claude Code | 内置工具是**混合体**：联网类（WebSearch/WebFetch）在服务器端、文件命令类（Bash/Read/Write）在本地 |
| Google Gemini | Google Search grounding、代码执行等 |
| 通用生态 | MCP 服务器（第三方写好，你用协议接进来） |

> 判「是不是托管/服务器端能力」，看**工具干活的代码在哪**，不看平台牌子。Claude Code 不是「整体本地跑」，而是「一部分工具服务器端、一部分工具本地」的混合体。

---

## 十一、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | `@tool` 装饰器是干嘛的 | 把普通函数包装成工具：自动填四格（名字/描述/schema/参数描述），不改函数行为 |
| 2 | docstring 为什么重要 | 它是模型唯一能看到的「工具是干嘛的/怎么用」的文字说明，贡献 description + 各参数 description |
| 3 | Location 参数为什么是 schema 而不是普通传参 | 模型语言边界是 JSON、不是 Python 对象；schema 是给模型看的参数说明书，运行期 JSON↔Python 双向翻译 |
| 4 | 双向翻译三过程怎么理解 | 举例：Python 类型→JSON schema→JSON 值→Python 对象，两个箭头（出去翻、回来翻） |
| 5 | JSON schema 是什么 | 用 JSON 写的一份「说明书」，规定数据长什么样；JSON＝值，schema＝形状 |
| 6 | 手动构造官方案例怎么理解 | 四零件：FunctionArgs(参数模具)＋do_some_work(干活)＋run_function(翻译官)＋FunctionTool(组装) |
| 7 | args 传 str，为什么 JSON 文本算 str | JSON 是「文本格式」，天生以字符串存在/传输；传输只能传字符不能传对象 |
| 8 | 「对象」指的就是字典吗 | 不是。Python 一切皆对象，字典只是其中一种；`json.loads`→dict，`model_validate_json`→类实例 |
| 9 | `Field(...)` 里的「...」为什么表示必填 | `...` 是 Python 的 Ellipsis，填在 Field 第一个参数（默认值）上；传 `...`＝没默认值，而 pydantic 里「没默认值＝必填」，所以等价于必填 |
| 10 | 为什么普通参数不写 Field 也必填 | pydantic 默认规则「没默认值＝必填」，`location: Location` 本来就必填；写 `Field(...)` 只为在「保持必填」的同时「加约束」 |
| 11 | 超时的三种应对方式分别是什么 | ① `error_as_result`（默认）：超时消息当工具结果喂给模型，不报错；② `raise_exception`：抛 `ToolTimeoutError`，代码 `try/except` 接（可取 `e.tool_name`/`e.timeout_seconds`）；③ `timeout_error_function`：自定义超时话术函数 |
| 12 | `failure_error_function` 有几种取值 | 三种：默认 `default_tool_error_function`（把异常写进结果）；自定义函数（收到 `(context, agent, tool, exception)`）；`None`（不兜底，异常直接冒泡） |
| 13 | 为什么「超时」只能异步，「报错」不限 | 超时要「中途强停」正在跑的函数，只有异步沿途有 `await` 让位、事件循环才能取消它；报错是函数自己 `raise` 冒上来的，与同步/异步无关 |
| 14 | 事件循环是抢占式还是协作式 | 协作式——任务只能被「自己」的 `await` 打断；所以当 A 到点但 B 还在埋头算（全程无 `await`）时，A 会被堵住，只能等 B 算完 |
| 15 | 托管工具是什么，和 FunctionTool 有何区别 | 托管工具实现全在 OpenAI 服务器上，你只「声明」不写实现，点单后 OpenAI 替你跑；FunctionTool 函数本体留本地、你的服务器跑。且托管工具只能用 `OpenAIResponsesModel` |
| 16 | agents as tools 是什么 | 用 `agent.as_tool(...)` 把现成 agent 打包成一个工具，塞进另一个 agent 的 `tools` 列表；编排器点它，SDK 底下真的跑起那个子 agent，把最终输出当工具结果送回 |
| 17 | 服务器端一条龙 vs 本地 agent 循环怎么区分 | 看本地代码跟服务器「打几次交道」：一条龙＝一次（调→执行→喂回循环在服务器内部）；本地循环＝多次往返（循环在本地程序）。两者工具实现/执行都在服务器，差别只在循环谁驱动 |
| 18 | 托管工具只有 OpenAI 有吗 | 「托管工具」这个词是 OpenAI 专属，但「官方提供现成能力」是行业共识：Anthropic server tools（web_search/code_execution）≈同款；Claude Code 是混合体（联网类在服务器端、文件命令类在本地）。判据看「工具干活的代码在哪」 |

---

*工具篇（FunctionTool / 超时报错 / 托管工具 / agents as tools）已学完。*

*遗留待办：① 下一步待学（衔接主题）：handoffs 进阶、guardrails、models、context/sessions、running_agents、results、tracing。*
