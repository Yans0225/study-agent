# OpenAI Agent 学习笔记 05（guardrails 护栏篇）

> 本文档是「系统性学习 Agent」第五阶段的整理，聚焦 **安全防护措施（guardrails）**——它和你刚学的 handoffs（笔记 04）正好接得上：`input_filter` 是「交接时筛历史」，guardrails 是「进出门安检、不达标就拉闸」。
> 来源：OpenAI Agents SDK 官方中文文档「安全防护措施」页（guardrails.md），另核对了 SDK 源码 `run.py` / `agent.py` / `guardrail.py` / `run_config.py`。
> 学习日期：2026-10-26
> 衔接：前言中的 `hooks`（笔记 02）、交接过滤器 `input_filter`（笔记 04）、FunctionTool 参数 schema（笔记 03）。
> **状态：✅ 本章已学完（2026-10-27 收尾）**（已学：概念 / 三类护栏 / 位置规则 / `tripwire_triggered` / `run_in_parallel` 执行模式 / 实现 / 工具护栏边界 / 输出触发器会话持久化）。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [护栏是什么：一句话 + 动机](#二护栏是什么一句话--动机)
3. [三类护栏与「工作流边界」](#三三类护栏与工作流边界)
4. [护栏怎么跑：三步 + tripwire_triggered](#四护栏怎么跑三步--tripwire_triggered)
5. [执行模式：并行 vs 阻塞](#五执行模式并行-vs-阻塞)
6. [触发器（Tripwires）与异常](#六触发器tripwires与异常)
7. [怎么写护栏（实现）](#七怎么写护栏实现)
8. [工具护栏覆盖不到的地方](#八工具护栏覆盖不到的地方)
9. [进阶细节（点到为止）](#九进阶细节点到为止)
10. [易混概念辨析](#十易混概念辨析)
11. [问答记录（Q&A）](#十一问答记录qa)
12. [和 handoffs 的对照](#十二和-handoffs-的对照)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 安全防护措施 guardrails | https://openai.github.io/openai-agents-python/zh/guardrails/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/guardrails.md |

> ⚠️ 获取记录：本机直连 GitHub 不通。经镜像 `https://ghproxy.net/https://raw.githubusercontent.com/...` 取到 guardrails.md 全文（同笔记 04 的通道）。提问中涉及的行为，另核对了源码：
> - `src/agents/agent.py`：`input_guardrails` / `output_guardrails` 字段定义与默认值；
> - `src/agents/run.py`：运行时如何决定「跑哪些护栏」；
> - `src/agents/guardrail.py`：`InputGuardrail` / `OutputGuardrail` / `GuardrailFunctionOutput` 定义；
> - `src/agents/run_config.py`：`RunConfig` 的 `input_guardrails` / `output_guardrails` 字段。

---

## 二、护栏是什么：一句话 + 动机

> **护栏 = 卡在「进」和「出」上的安检门**，用便宜/快的检查去保护贵/慢的模型，不合格就立刻「拉闸」停掉流程。

官方开篇的例子最能说明动机：你有一个用**很聪明但很慢很贵**的模型做客服。不想让恶意用户拿它写数学作业。于是用**又便宜又快**的模型跑一道护栏去查——查到「这是数学作业」就**立刻抛错中断，贵模型根本没启动**，省时省钱。

**灵魂：用小成本拦截大成本。**（呼应：护栏本身通常就是「再跑一个小 agent 来判定」，见第七节。）

---

## 三、三类护栏与「工作流边界」

护栏分三类，挂点、检查对象、运行时机各不同：

| 类别 | 挂在哪 | 检查什么 | 什么时候跑 |
|---|---|---|---|
| **输入护栏** | agent 的 `input_guardrails` | 用户**初始输入** | 仅当该 agent 是链里**第一个**时 |
| **输出护栏** | agent 的 `output_guardrails` | agent 的**最终输出** | 仅当该 agent 是链里**最后一个**（产出最终输出）时 |
| **工具护栏** | 具体的 `FunctionTool` 上 | 某次**函数工具调用**的前 / 后 | 每次调用这个工具都跑 |

### 位置规则（最容易误会的一点）

输入/输出护栏**挂在 agent 身上，但不是每个 agent 都会跑**：

- **输入护栏**只认「最前面那个」agent——因为它的语义就是「针对（用户的）输入」；
- **输出护栏**只认「最后那个」agent——因为它针对「最终输出」；
- 一串 handoff（前台 → 专家 → 后台）里，**中间那些 agent 的输入/输出护栏一个都不跑**。

**「第一个 / 最后一个」不是 agent 的固有属性，而是「这次运行里它站的位置」**——由 **Runner 在运行时自动判定**，你不会收到任何提示，位置不对就是静默不跑。

> 推论：工作流含管理器 / 任务转移 / 委派专用 agent 时，想要「每次函数调用前后都检查」，要用**工具护栏**，别指望 agent 级输入/输出护栏（工具护栏绑在工具上，与 agent 站位无关）。

---

## 四、护栏怎么跑：三步 + tripwire_triggered

输入护栏、输出护栏都走这三步：

```
① 收到检查对象
     输入护栏 → 收到跟 agent 一样的那份输入
     输出护栏 → 收到 agent 生成的输出

② 跑你的护栏函数 → 产出 GuardrailFunctionOutput
     （对象里装着两个字段：tripwire_triggered + output_info）

③ Runner 看 tripwire_triggered：
     == True  → 抛异常（InputGuardrailTripwireTriggered /
                          OutputGuardrailTripwireTriggered），整个运行立刻停
     == False → 放行，什么都不做，运行继续
```

### 关键字段：`tripwire_triggered`（一个开关，两个位置）

- 名字来源：**tripwire = 绊线/报警线**，一被绊到就触发警报。
  - `tripwire_triggered = True` → **绊线被绊到** → 拉闸（抛异常、停运行）；
  - `tripwire_triggered = False` → **没绊到** → 放行。
- 它是**一个布尔字段**（不是两个开关），值由**你的护栏函数自己算出来**填进去。
- ⚠️ **这是护栏唯一能「叫停运行」的通道。**

### 同对象里的另一个字段：`output_info`

`GuardrailFunctionOutput` 就这两个字段，分工完全不同：

| 字段 | 作用是 | 影响运行吗 |
|---|---|---|
| `tripwire_triggered` | **控制**：拉闸 or 放行 | ✅ 就是它决定 |
| `output_info` | **记录**：塞点额外信息（检查了什么、判定理由），可在 `try/except` 里读出来看 | ❌ 纯信息，不影响流程 |

> 类比：护栏函数交一份**体检报告单**（`GuardrailFunctionOutput`）。「**是否异常**」栏 = `tripwire_triggered`（医院只看这栏决定叫停）；「**详细说明**」栏 = `output_info`（给人看）。
> 与 `input_type`「纸条」同理——**一个对象里，有的字段管"控制"，有的字段管"记录"，别把整份对象的职责压在一个字段上。**

### 和 `on_handoff` 的对比（务必记牢）

| | 怎么「拦截」 |
|---|---|
| `on_handoff`（笔记 04） | 得自己 **`raise` 异常**；正常 `return` 反而放行 |
| 护栏（本章） | 你**不用 raise**，只要把 `tripwire_triggered` **设成 True**，**Runner 替你抛异常** |

护栏是「你按按钮（设 True），系统自动拉闸」；on_handoff 是「你得自己动手扔炸弹」。

---

## 五、执行模式：并行 vs 阻塞（`run_in_parallel`）

这是**输入护栏独有**的旋钮。它管的是「**这一条护栏** 和 **那个又贵又慢的模型**」之间的**时序关系（谁先谁后）**——**不是**管多条护栏之间互相排队。

### 它是个什么字段

```python
class InputGuardrail:
    run_in_parallel: bool = True   # 默认 True（并行）
```

定义护栏时烘焙进去，用装饰器设：

```python
@input_guardrail(name="检查是否越界", run_in_parallel=False)
async def my_check(ctx, agent, input): ...
```

### Runner 怎么用它：把护栏劈成两堆

运行时 SDK 按这个字段把输入护栏**分桶**（源码 `run.py`）：

```python
all_input_guardrails  = starting_agent.input_guardrails + run_config.input_guardrails
sequential_guardrails = [g for g in all_input_guardrails if not g.run_in_parallel]  # 阻塞堆
parallel_guardrails   = [g for g in all_input_guardrails if g.run_in_parallel]      # 并行堆
```

调度时（仅首轮、仅第一个 agent）：

```python
if sequential_guardrails:
    await run_input_guardrails(...)                    # ① 阻塞堆先跑完，模型还没影儿
model_task = asyncio.create_task(run_single_turn(...)) # ② 模型现在才启动
if parallel_guardrails:
    guardrail_task = asyncio.create_task(...)          # ③ 并行堆和模型同时起跑
guardrail_model_future = asyncio.gather(guardrail_task, model_task)
```

| 模式 | 参数 | 与模型的关系 | 拉闸时 | 代价 |
|---|---|---|---|---|
| **并行执行**（默认） | `run_in_parallel=True` | 护栏与模型**同时起跑** | 模型**早跑上了**，只能 cancel 掉——**已烧的 token / 已跑的工具有副作用** | 延迟最低 |
| **阻塞执行** | `run_in_parallel=False` | 护栏**先跑完、通过**才启动模型 | 模型**从没启动** → **零 token、零副作用** | 慢（护栏 + 模型串行相加） |

### ⚠️ 最反直觉的一点：默认值并不省钱

官方给护栏的动机是「**用低成本检查保护高成本模型**」。但默认 `run_in_parallel=True` 意味着：**护栏和模型同时开跑**——等护栏发现违规、触发 tripwire 时，那个贵模型**早就跑起来、可能已经花了 token / 调过工具**了，此时 cancel 也撤不回来。

所以：

- **想真正「护住贵模型 / 省钱 / 防副作用」→ 必须显式写 `run_in_parallel=False`。**
- **默认 `True` 换来的只是「最低延迟」**，代价是拉闸时你照样为模型买了单。

> 一句话记忆：**默认是「快」的那档，不是「省」的那档。**

### 几个容易踩的点

1. **逐条生效，可混用**：字段挂在**每条**护栏上，不是全局开关。同一个 agent 上可以挂多条——一条 `False` 当硬闸门（如敏感词拦截）、一条默认 `True` 图快。Runner 自动分堆：先跑完阻塞堆，再一起跑并行堆 + 模型。
2. **阻塞堆内部仍是并发的**：这里的「顺序」只相对**模型**而言；堆内多条护栏彼此照样用 `asyncio.create_task` 并发跑（`run_input_guardrails_with_queue`）。准确说法是该堆**整体**在模型之前跑完，而非堆内彼此排队。
3. **`OutputGuardrail` 没有这个字段**：输出护栏**必然发生在 agent 完成之后**（要先拿到输出），谈不上「和模型抢跑」，所以 SDK 干脆不给——传了也无效（参数根本不存在）。
4. **定义期属性，运行期不可翻**：写在 `@input_guardrail(...)` 或构造时定死；`RunConfig` **没有**对应覆盖项。想按这次 run 换模式，得换一个护栏对象（正好呼应第七节「护栏没有 `is_enabled`」的绕法）。
5. **只在首轮生效**：和输入护栏本身一样，只发生在链的第一个 agent、第一回合（源码 `current_turn <= 1`）。中间 agent 的输入护栏本就不跑，这字段自然也无从谈起。

---

## 六、触发器（Tripwires）与异常

护栏不过 → Runner 抛异常、停止执行。四种：

| 场景 | 抛出的异常 |
|---|---|
| 输入护栏触发 | `InputGuardrailTripwireTriggered` |
| 输出护栏触发 | `OutputGuardrailTripwireTriggered` |
| 工具输入护栏触发 | `ToolInputGuardrailTripwireTriggered` |
| 工具输出护栏触发 | `ToolOutputGuardrailTripwireTriggered` |

在 `try/except` 里接住它，就能恰当地响应用户 / 处理异常。异常里能拿到：**是哪个护栏触发的**（`guardrail_result`），以及**这次运行已完成的所有护栏结果**（`run_data.input_guardrail_results` / `run_data.output_guardrail_results` 等）。

---

## 七、怎么写护栏（实现）

核心规律：**写一个函数，返回 `GuardrailFunctionOutput`**。最常见手法是**在护栏里再跑一个小 agent 来判定**（呼应第二节「用便宜模型查」）。

### 输入护栏（`@input_guardrail`）

```python
from pydantic import BaseModel
from agents import (Agent, GuardrailFunctionOutput,
                    InputGuardrailTripwireTriggered, RunContextWrapper, Runner)
from agents.decorators import input_guardrail

class MathHomeworkOutput(BaseModel):
    is_math_homework: bool
    reasoning: str

guardrail_agent = Agent(                      # 廉价「安检员」agent
    name="Guardrail check",
    instructions="Check if the user is asking you to do their math homework.",
    output_type=MathHomeworkOutput,
)

@input_guardrail
async def math_guardrail(ctx: RunContextWrapper[None], agent: Agent,
                         input: str | list[TResponseInputItem]) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, input, context=ctx.context)
    return GuardrailFunctionOutput(
        output_info=result.final_output,                          # 附注（可读，不影响流程）
        tripwire_triggered=result.final_output.is_math_homework,  # ← 结论开关
    )

agent = Agent(
    name="Customer support agent",
    instructions="You are a customer support agent...",
    input_guardrails=[math_guardrail],            # ← 挂上去
)

async def main():
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")
    except InputGuardrailTripwireTriggered:
        print("Math homework guardrail tripped")
```

### 输出护栏

结构一模一样，只是：装饰器换 `@output_guardrail`，函数接的是 `output`（agent 的最终输出），挂到 `output_guardrails=[...]`，异常换 `OutputGuardrailTripwireTriggered`。

### 工具护栏（`@tool_input_guardrail` / `@tool_output_guardrail`）

```python
import json
from agents import ToolGuardrailFunctionOutput
from agents.decorators import tool, tool_input_guardrail, tool_output_guardrail

@tool_input_guardrail
def block_secrets(data):
    args = json.loads(data.context.tool_arguments or "{}")
    if "sk-" in json.dumps(args):
        return ToolGuardrailFunctionOutput.reject_content("Remove secrets before calling this tool.")
    return ToolGuardrailFunctionOutput.allow()

@tool_output_guardrail
def redact_output(data):
    text = str(data.output or "")
    if "sk-" in text:
        return ToolGuardrailFunctionOutput.reject_content("Output contained sensitive data.")
    return ToolGuardrailFunctionOutput.allow()

@tool(tool_input_guardrails=[block_secrets],
      tool_output_guardrails=[redact_output])
def classify_text(text: str) -> str:
    """Classify text for internal routing."""
    return f"length:{len(text)}"

agent = Agent(name="Classifier", tools=[classify_text])
```

> **工具护栏不玩 `tripwire_triggered` 那套**——用的是 `ToolGuardrailFunctionOutput.allow()`（允许）/ `.reject_content(消息)`（拒绝并给条消息）。语义更直白，还多两个本事：**跳过调用**、**替换输出**。

### 运行时「跳过」某条护栏（护栏没有 `is_enabled`）

护栏**没有** `is_enabled` 旋钮，想在运行时「这次不跑」，用护栏函数内**短路放行**：

```python
@input_guardrail
async def math_guardrail(ctx: RunContextWrapper[None], agent: Agent, input) -> GuardrailFunctionOutput:
    if ctx.context.get("skip_guardrails"):            # 运行时开关
        return GuardrailFunctionOutput(output_info=None, tripwire_triggered=False)  # 直接放行
    ... # 正常检查
```

（`GuardrailFunctionOutput` 的 `output_info` 可传 `None`，所以「空跑一条放行」合法。）

---

## 八、工具护栏覆盖不到的地方

文档明确列的「坑」（免得以后踩）：

- **handoff 调用本身**不走工具护栏管线（它走 handoff 管线）；
- **托管工具**（`WebSearchTool` / `FileSearchTool` / `HostedMCPTool` / `CodeInterpreterTool` / `ImageGenerationTool`）和**内置执行工具**（`ComputerTool` / `ShellTool` / `ApplyPatchTool` / `LocalShellTool`）都**不走**工具护栏管线；
- `Agent.as_tool()` 目前也**不直接**暴露工具护栏选项；
- **本地 MCP 服务器**上可设 `tool_input_guardrails` / `tool_output_guardrails`（附到该服务器每个工具上）。

---

## 九、进阶细节（点到为止）

文档约 line 58–73 讲**输出触发器触发时的会话持久化行为**（先不深挖，留待 sessions / running_agents 章回看）：

- 触发时会**保留已完成的工具调用/输出 + 重放所需的推理上下文**，但**排除被拒绝的那个候选最终输出**；
- 被拒工具输出的载荷会被替换为占位文本 `"Output withheld by an output guardrail."`；
- 可用 `RunConfig.output_guardrail_blocked_message` 自定义那句占位文本（可传字符串或**同步**格式化函数；函数拿不到被拒内容 / `output_info`）。

现在只需记住：**输出护栏拒绝输出时，SDK 会小心保留「可重放」的上下文，同时把被拒的输出内容清洗掉**，不把敏感/被拒载荷原样留在会话里。

---

## 十、易混概念辨析

### ① 三类护栏挂点 / 时机速查

| 类别 | 挂点 | 时机 |
|---|---|---|
| 输入护栏 | `agent.input_guardrails` | 仅第一个 agent（首轮） |
| 输出护栏 | `agent.output_guardrails` | 仅产出最终输出的 agent |
| 工具护栏 | 具体 `FunctionTool` 上 | 每次调用该工具的前 / 后 |

### ② 护栏 vs `input_filter`（都跟 handoff 有关，但干的不是一回事）

| | `input_filter`（笔记 04） | Guardrails（本章） |
|---|---|---|
| 干什么 | 交接时**筛历史**，决定下一个 agent 看到什么 | **硬拦截**：不达标就抛异常掐断 |
| 触发不了怎么办 | 只是「少看一些」，流程照走 | 直接**停**，`try/except` 接住 |
| 拦是「谁」拦 | 你写函数**返回**新列表 | 你设 `tripwire_triggered=True`，**Runner 替你抛** |
| 典型场景 | 隐私内容不给下游看 | 恶意输入、越狱、敏感数据出入 |

> 一句话：**input_filter 是「交接时过滤」，guardrails 是「进出门安检」。**

### ③ 护栏有没有 `is_enabled`？（对称性陷阱）

| 对象 | 有 `is_enabled` 吗？ | 证据 |
|---|---|---|
| **工具** `FunctionTool` | ✅ 有（bool 或可调用） | 笔记 03；源码 `tool.py` |
| **handoff()** | ✅ 有 | 笔记 04 的 8 旋钮之一 |
| **护栏** `InputGuardrail` / `OutputGuardrail` | ❌ **没有** | 源码 `guardrail.py` 中 `is_enabled` 出现 0 次 |

**所以别想当然**：工具、handoff 挂一个 `is_enabled` 就能运行时开关，**护栏没有这个口子**，只能走「不挂 / 函数内短路 / 动态构建 agent」。官方不给的意图：护栏是安全兜底，随手能关掉就形同虚设——**开关由"构建时挂不挂"决定（静态），而非"运行时能不能翻"（动态）**。

### ④ `tripwire_triggered` vs 护栏函数的「结果」

- 护栏函数的**结果** = 一整个对象 `GuardrailFunctionOutput`（含 `tripwire_triggered` + `output_info` 两个字段）；
- `tripwire_triggered` 是其中**「结论」那一栏**——**控制意义上**它"就是结果的关键"，**结构意义上**它只是结果的一个字段。

### ⑤ Agent 级护栏 vs RunConfig 级护栏

`RunConfig` 里也有 `input_guardrails` / `output_guardrails`，是**运行级（每次调用临时附加）**的护栏，区别于 **agent 级（构建时写死）** 的护栏。关键差别：**只能加、不能减**（详见 Q4/Q5）。

---

## 十一、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | 是不是每个 agent 都有输入/输出护栏，但只有它是链中第一个/最后一个时才被调用？这个过程是自动检测的吗 | **基本对**。`input_guardrails` / `output_guardrails` 字段长在 `Agent` 上，谁都能填。「第一个/最后一个」**不是 agent 固有属性，而是这次运行里它站的位置**，由 **Runner 运行时自动判定**，无提示、位置不对就静默不跑。同一 agent 在别的运行里当中间环节时，护栏就不跑 |
| 2 | 护栏是默认「绝对执行」的吗？我不想让它执行有办法吗 | **默认绝对执行，没有内置跳过开关**。三条路：① **干脆不挂**（构建时不放进列表）；② **护栏函数内短路**（读 `ctx.context` 开关，立即 `return tripwire_triggered=False` 放行）；③ **动态构建 agent**（按条件造带/不带护栏的版本）。⚠️ `RunConfig` 只能加不能减，不是关闭手段 |
| 3 | `tripwire_triggered == True / False` 这两个「开关」是做什么的 | **不是两个开关，是一个布尔字段的两个位置**。`tripwire = 绊线`：`True` = 绊到 = **抛异常、拉闸、停运行**；`False` = 没绊到 = **放行**。值由你的护栏函数算出来填进去，是护栏**唯一**的「叫停通道」。别和同对象的 `output_info`（纯记录、不影响流程）混淆 |
| 4 | 我可以理解这个字段就是护栏函数的结果吗？ | **方向对，精确一点**：护栏函数的**结果**是一个**对象** `GuardrailFunctionOutput`；`tripwire_triggered` 是这个对象里的**一个字段**（"结论"栏），不是整个结果。从「**什么决定 Runner 拦不拦**」看，可以说"就是它"；从结构看，它只是结果的一个字段 |
| 5 | 「starting_agent 的护栏永远在里面，RunConfig 只能再往上加，减不掉」是什么意思（我举的 RunConfig 例子没看懂） | 有**两个同名列表、不是一个**：`Agent.input_guardrails`（构建时写死）+ `RunConfig.input_guardrails`（每次运行额外附加）。运行时 SDK 用 `+` **拼接**：`starting_agent.input_guardrails + (run_config.input_guardrails or [])`。因为是拼接**没有减法**，agent 自带的那些**永远都跑**；RunConfig **只能让你多跑几条，不能少跑** |
| 6a | 可以理解为 RunConfig 类似「函数（运行）级护栏」吗 | **方向对**：`RunConfig` 传给 `Runner.run(...)`，是**运行级/调用级**护栏，区别于 agent 级。但精确两点：① 它是**在 agent 护栏基础上"再补几条"**（只加不减）；② **照样受位置规则约束**（不是无差别作用每一环） |
| 6b | 挂到「中间的 agent」上也能起作用吗 | **❌ 不能**。RunConfig 的**输入护栏**被拼进**起始 agent** 那组、`current_turn==0` 才跑；**输出护栏**跟**产出最终输出的 agent** 跑。中间 agent（非首非尾）**agent 级和 RunConfig 级的输入/输出护栏都不作用**→ 正因此才有**工具护栏**补这个洞。（源码另有一条 resume+pending_input 的特殊路径用 `current_agent`，正常流程碰不到） |
| 6c | agent 允许不写输入/输出护栏吗 | **✅ 允许，且是默认状态**。源码：`input_guardrails` / `output_guardrails` 默认值都是**空列表**（`field(default_factory=list)`）。不写 = 空 = 没护栏 |
| 6d | 我的 agent 没有输入护栏、但排在链第一位，会怎样 | **什么都不发生，正常运行、不报错**。关键心智：**「是第一个 agent」只是「触发条件」，不是「义务」**——它说「**若**第一个 agent 有输入护栏，现在就跑」，**没说**「第一个 agent **必须**有护栏」。有护栏→跑它；没护栏→这一关本来就没装门，直接进。**位置 ≠ 义务** |

---

## 十二、和 handoffs 的对照

| | `input_filter`（笔记 04） | Guardrails（本章） |
|---|---|---|
| 阶段 | 交接时 | 进门 / 出门 / 每次工具调用前后 |
| 手段 | 改「下一个 agent 看到的历史」 | 设 `tripwire_triggered=True` → Runner 抛异常 |
| 强制力 | 弱（只筛历史，流程照走） | 强（直接掐断） |
| 位置规则 | 交接时才生效 | 输入护栏＝第一个、输出护栏＝最后一个、工具护栏＝每次 |

---

*guardrails 护栏篇（三类护栏 / 位置规则 / tripwire_triggered / run_in_parallel 执行模式 / 实现 / 与 input_filter 对照）**已完成**。*

*后续待学：tracing（下一章）、context/sessions、running_agents、results、models。*