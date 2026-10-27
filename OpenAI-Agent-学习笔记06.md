# OpenAI Agent 学习笔记 06（tracing 追踪篇）

> 本文档是「系统性学习 Agent」第六阶段的整理，聚焦 **追踪（tracing）**——它和你刚学的 guardrails（笔记 05）正好接得上：护栏是「**在运行中拦一道**」，tracing 是「**在旁边把整个运行录下来**」，一个管掐断、一个管留证据。
> 来源：OpenAI Agents SDK 官方中文文档「追踪」页（tracing.md），另核对了 SDK 源码 `tracing/` 目录（provider.py / config.py / span_data.py / create.py / __init__.py）与 `run.py` / `run_config.py`。
> 学习日期：2026-10-26 ~ 2026-10-27
> 衔接：`run_config` 参数（笔记 02）、`RunConfig` 字段分组（笔记 05 顺带讲过）、`handoff_span` 对应 handoffs（笔记 04）、`guardrail_span` 对应 guardrails（笔记 05）。
> **状态：✅ 本章已学完（2026-10-27 收尾）**（已学：trace/span 概念 / 默认 10 类 span / 三种关闭方式及优先级 / `include_task_and_turn_spans` 紧凑层级 / `trace()` 与 `custom_span()` 用法 / 敏感数据 / `flush_traces` / 自定义处理器与生态集成（点到为止））。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [追踪是什么：一句话 + 动机](#二追踪是什么一句话--动机)
3. [两个核心对象：trace 与 span](#三两个核心对象trace-与-span)
4. [默认追踪：SDK 自动录了什么](#四默认追踪sdk-自动录了什么)
5. [怎么关：三种方式与优先级](#五怎么关三种方式与优先级)
6. [紧凑层级：关掉 task/turn 两层](#六紧凑层级关掉-taskturn-两层)
7. [创建 trace 与 span（自己要写的那两块）](#七创建-trace-与-span自己要写的那两块)
8. [实际应用长什么样（客服 Agent 例子）](#八实际应用长什么样客服-agent-例子)
9. [敏感数据](#九敏感数据)
10. [长时间运行进程与即时导出](#十长时间运行进程与即时导出)
11. [自定义处理器与数据隐去（点到为止）](#十一自定义处理器与数据隐去点到为止)
12. [非 OpenAI 模型的追踪](#十二非-openai-模型的追踪)
13. [易混概念辨析](#十三易混概念辨析)
14. [问答记录（Q&A）](#十四问答记录qa)
15. [和 guardrails 的对照](#十五和-guardrails-的对照)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 追踪 tracing | https://openai.github.io/openai-agents-python/zh/tracing/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/tracing.md |

> ⚠️ 获取记录：本机直连 GitHub 不通。经镜像 `https://ghproxy.net/https://raw.githubusercontent.com/...` 取到 tracing.md 全文（同笔记 04/05 的通道）。提问中涉及的行为，另核对了源码：
> - `src/agents/tracing/provider.py`：关闭标志的优先级机制（`_env_disabled` / `_manual_disabled` / `_disabled`）；
> - `src/agents/tracing/config.py`：`TracingConfig` TypedDict 与 `include_task_and_turn_spans()`；
> - `src/agents/tracing/span_data.py`：各类 `*SpanData` 的字段定义；
> - `src/agents/tracing/create.py`：`trace()` / `custom_span()` 等工厂函数签名；
> - `src/agents/run.py`：运行期 `task_span` / `turn_span` / `agent_span` 的创建与生命周期；
> - `src/agents/run_config.py`：`tracing` / `tracing_disabled` / `trace_include_sensitive_data` 字段。

---

## 二、追踪是什么：一句话 + 动机

> **追踪 = 给整个 Agent 系统装的「行车记录仪」。**
> `trace` = 一趟完整的行程（一次任务，从头到尾）；`span` = 行程里的一段路（一次 LLM 调用、一次工具调用、一次交接…）。

**为什么要有它**：Agent 一次运行内部会发生很多事——多次模型轮次、多次工具调用、可能的交接、护栏检查、语音转写……线上出了问题（用户投诉、结果不对、烧钱、变慢），**光看最终输出根本不知道是哪一步坏的**。追踪把这些步骤全录成**可以点开的树**，让你能像看慢放录像一样逐帧定位。

**关键心智**：SDK **默认全自动录**，绝大多数 span 你一行代码都不用写。你手工只加两处：
- `with trace(...)`：把「一次业务任务」圈出来（起名字、贴用户 ID、串会话）；
- `custom_span(...)`：把 SDK 看不见的**你自己业务逻辑**也录进去。

**默认是开的**。文档原文：「追踪默认启用。」三条关闭方式见第五节。

---

## 三、两个核心对象：trace 与 span

### 3.1 定义

| 对象 | 一句话 | 类比 |
|---|---|---|
| **trace（追踪）** | 一次「工作流」的**端到端**操作 | 一整趟行程 |
| **span（跨度）** | 一段**有开始/结束时间**的操作 | 行程里的一段路 |

一个 trace 由**多个 span** 组成；span 之间靠 `parent_id` 形成**父子嵌套树**。

### 3.2 属性对照

**trace 的属性**：

| 属性 | 含义 |
|---|---|
| `workflow_name` | 逻辑工作流/应用的名称，如 `"code_generation"` / `"customer_service"` |
| `trace_id` | 追踪唯一 ID。不传则自动生成。**格式必须 `trace_<32位字母数字>`** |
| `group_id` | 可选。用于**关联同一场对话/进程的多个 trace**（如聊天线程 ID） |
| `disabled` | 若为 True，则不记录该 trace |
| `metadata` | 可选的**自定义元数据**字典（用来贴标签、日后筛选） |

**span 的属性**：

| 属性 | 含义 |
|---|---|
| `started_at` / `ended_at` | 开始/结束时间戳 |
| `trace_id` | 它属于哪个 trace |
| `parent_id` | 指向**父 span**（若有）——这就是树的来源 |
| `span_data` | 这段具体的信息载体。如 `AgentSpanData` 存 agent 信息、`GenerationSpanData` 存 LLM 生成……（见 `span_data.py`） |

> 💡 **对称性陷阱**：`trace_id` 同时出现在 trace 和 span 上——**不是重复**。trace 的 `trace_id` 是「**我是谁**」，span 的 `trace_id` 是「**我属于哪个 trace**」（外键）。同一个字段名，方向相反。

### 3.3 默认名字是占位的

> 默认情况下，追踪名称是**字面字符串 `Agent workflow`**。想换名字，用 `trace()` 显式命名，或用 `RunConfig` 配置。不命名的话，仪表板上所有 trace 都叫 `Agent workflow`——**多个任务长得一模一样，没法区分**。所以「起个名」是有实际意义的动作。

---

## 四、默认追踪：SDK 自动录了什么

SDK 默认**自动**把这些包装成 span（你不用写）：

| span | 对应哪一段 |
|---|---|
| `trace()` | 整个 `Runner.{run, run_sync, run_streamed}()` |
| `task_span()` | **每次 runner 调用**（一次任务） |
| `turn_span()` | **每个模型轮次** |
| `agent_span()` | **每次 agent 实际运行时** |
| `generation_span()` | LLM 生成 |
| `function_span()` | **每次函数工具调用**（各自独立） |
| `guardrail_span()` | 安全防护措施执行 |
| `handoff_span()` | 任务交接 |
| `transcription_span()` | 音频输入（语音转文字） |
| `speech_span()` | 音频输出（文字转语音） |
| `speech_group_span()` | SDK 可能把相关音频 span 归到它下面 |

**它们天然形成一棵树**（外层大、内层小）：

```
trace (整个 run)
  └─ task_span (一次 runner 调用)
      ├─ guardrail_span (进门前安检)
      └─ agent_span (某个 agent 干活)
          ├─ turn_span #1 (模型第 1 轮)
          │   ├─ generation_span (LLM 调用：输入/输出/token)
          │   └─ function_span (某个工具调用)
          ├─ turn_span #2
          │   ├─ generation_span
          │   └─ handoff_span (交接)
          └─ agent_span (交接后的新 agent …… 递归)
```

> 💡 **注意层级关系**：`agent_span` 里套 `turn_span`，`turn_span` 里套 `generation_span` + `function_span`。所以「**这轮工具调用的结果**」天然挂在「**这轮**」下面、「**这个 agent**」下面、「**这次任务**」下面——**上下文是免费的**，位置就说明了归属。

---

## 五、怎么关：三种方式与优先级

| 方式 | 粒度 | 写法 |
|---|---|---|
| ① 环境变量 | **全局**（进程级） | `OPENAI_AGENTS_DISABLE_TRACING=1` |
| ② 代码全局 | **全局**（进程级） | `set_tracing_disabled(True)` |
| ③ 单次运行 | **这一次** | `RunConfig(tracing_disabled=True)` |

> ⚠️ **ZDR 政策**：对根据零数据保留（ZDR）政策使用 OpenAI API 的组织，**追踪功能不可用**。

### 5.1 两个全局方法有什么区别（源码级）

它俩都落到同一个地方（`provider.py` 的 `DefaultTraceProvider`），但**写入的标志位不同**：

```
_env_disabled     ← 环境变量「懒读取」，第一次用的时候读一次，之后缓存
_manual_disabled  ← set_tracing_disabled() 写的
_disabled         ← 最终生效值，由上面两个算出来
```

优先级逻辑（`_refresh_disabled_flag`）：

```
若 _manual_disabled is None  → 用 _env_disabled 的值（环境说了算）
否则                          → 用 _manual_disabled 的值（手动覆盖环境）
```

**结论**：
- **`set_tracing_disabled()` 手动设置的优先于环境变量**。设了 `True`，就算环境变量没给，也是关；设了 `False`，**甚至能覆盖掉环境变量的「关」**（把手动设成 False 会强行打开）。
- 环境变量**只认 `"true"` / `"1"`**（其余值不当作开/关）。
- **`_manual_disabled` 一旦被调用过就永久不为 `None`**（它就是被 `set_disabled` 赋值的，没有复位路径）→ **一旦你调过 `set_tracing_disabled()`，环境变量这条路对你**永久失效**，之后一切都听手动的**。

### 5.2 选哪个

- 开发时懒得改代码全局关掉 → **① 环境变量**（或 ② 一次性代码）。
- 只想让**某一次特殊运行**不录 → **③ `RunConfig(tracing_disabled=True)`**（不影响别的运行）。

---

## 六、紧凑层级：关掉 task/turn 两层

文档原句：「如果希望使用更紧凑的层级结构，请**针对一次运行**禁用自动任务跨度和轮次跨度。智能体、生成、函数、安全防护措施、任务转移和自定义跨度**仍会被记录**。」

```python
from agents import RunConfig, Runner

result = await Runner.run(
    agent,
    "Hello",
    run_config=RunConfig(tracing={"include_task_and_turn_spans": False}),
)
```

### 6.1 关的是谁

只关 **`task_span`（每次 runner 调用）** 和 **`turn_span`（每个模型轮次）** 这两层**外壳**。其余（`agent_span` / `generation_span` / `function_span` / `guardrail_span` / `handoff_span` / `custom_span`）**照样录**。

树从：

```
task_span
  └─ agent_span
      └─ turn_span
          ├─ generation_span
          └─ function_span
```

变成：

```
agent_span
  ├─ generation_span
  └─ function_span
```

### 6.2 「更紧凑」是什么意思

task/turn 这两层是**纯容器**——它们本身不承载业务信息，只是把里面那一堆操作**再包一层**。层级深了，仪表板上点开要一层层剥，视觉上冗余。关掉 = **少两层壳，操作直接挂在 agent 下**，一眼能看完。

### 6.3 关掉有啥影响 / 它们有实际作用吗

**运行时零影响**。这两层纯属「记账/组织」用：

| span | `span_data` 字段（源码 `span_data.py`） | 作用 |
|---|---|---|
| `TaskSpanData` | `name` / `usage` / `metadata` | 汇总**整次任务**的 token 用量 |
| `TurnSpanData` | `turn` / `agent_name` / `usage` / `metadata` | 汇总**单个轮次**的 token 用量（第几轮、哪个 agent） |

**代价**：关掉后你就**丢了「按轮次/按任务看 token 用量」这个视图**——排查「哪一轮烧钱」就没那么直接了（还能从 `generation_span` 逐条看，只是少了个汇总层）。

> ✅ **结论**：想省 token 而关它——**没用**（它不额外调用模型）。想**图清爽**而关它——可以，代价是失去用量汇总层。属于「纯展示偏好」，不是性能开关。

### 6.4 怎么配的（源码）

`tracing/config.py`：

```python
def include_task_and_turn_spans(config: TracingConfig | None) -> bool:
    return config is None or config.get("include_task_and_turn_spans", True)
```

**不传 → 默认 True（录）**。所以只有显式给 `False` 才关。

---

## 七、创建 trace 与 span（自己要写的那两块）

### 7.1 创建 trace：`trace()`

```python
def trace(
    workflow_name: str,          # 工作流名（仪表板显示用）
    trace_id: str | None = None, # 不传自动生成；推荐 util.gen_trace_id()
    group_id: str | None = None, # 关联同一对话的多个 trace（如聊天线程 ID）
    metadata: dict | None = None,# 自定义元数据（贴标签、日后筛选）
    tracing: TracingConfig | None = None,
    disabled: bool = False,
) -> Trace:
```

**两种启动方式**：

1. **推荐**：当上下文管理器用——`with trace(...) as my_trace:`，**自动**在合适时机 start/finish。
2. 手动：`trace.start()` + `trace.finish()`。

> ⚠️ 当前 trace 用 Python 的 `contextvar` 跟踪 → **天然支持并发**。若手动启停，**必须**把 `mark_as_current` 传给 `start()`、`reset_current` 传给 `finish()`，否则「当前 trace」指针不更新。

**「更高级别的追踪」**：想让**多次 `run()` 属于同一个 trace**，就把整个代码块包进一个 `trace()`：

```python
with trace("Joke workflow"):
    first  = await Runner.run(agent, "Tell me a joke")
    second = await Runner.run(agent, f"Rate this joke: {first.final_output}")
```

两次 `run` 就成了**同一个整体 trace 的一部分**，而不是各录各的。

### 7.2 创建 span：各种 `*_span()` 与 `custom_span()`

早说过：**通常不需要手动创建 span**（SDK 自动包）。你唯一常用的是：

```python
def custom_span(
    name: str,
    data: dict | None = None,     # 你想塞进去的任意结构化数据
    span_id: str | None = None,
    parent: Trace | Span | None = None,  # 不传自动挂当前 trace/span 下
    disabled: bool = False,
) -> Span[CustomSpanData]:
```

用法：`with custom_span("风险预检", data={"rule": "refund_lock_v2"}):`

- span **自动成为当前 trace 的一部分**，并**嵌套在最近的当前 span** 下（`contextvar` 跟踪）。
- 这是你**补录自己的业务逻辑**（风控、DB 查询、外部 API）的唯一入口。

---

## 八、实际应用长什么样（客服 Agent 例子）

**场景**：电商客服 Agent——能查订单、发起退款（工具）；有输入护栏（拦骂人/威胁）；遇纠纷交接给「退款专员 agent」。

```python
from agents import Agent, Runner, trace, custom_span

async def handle_user_message(thread_id: str, user_message: str):
    with trace(                                    # ① 你加的：圈任务
        workflow_name="customer_support",
        group_id=thread_id,                        # 同一对话的多轮靠它串起来
        metadata={"user_id": "u_8842", "order_no": "A20315"},
    ):
        with custom_span("风险预检", data={"rule": "refund_lock_v2"}):  # ② 你加的
            risk = await my_risk_check(user_message)   # 你自己的代码

        result = await Runner.run(triage_agent, user_message)  # ③ 其余自动成树
        return result.final_output
```

**仪表板里渲染成的树**：

```
⏱ trace：customer_support
  └─ task_span
      ├─ custom_span：风险预检   ← 你加的
      ├─ guardrail_span：输入护栏
      ├─ agent_span：triage_agent
      │   ├─ turn_span #1
      │   │   ├─ generation_span
      │   │   └─ function_span：查订单
      │   ├─ turn_span #2
      │   │   ├─ generation_span
      │   │   └─ handoff_span     ← 交接给退款专员
      │   └─ ...
      └─ agent_span：refund_agent
          └─ turn_span #1
              ├─ generation_span
              └─ function_span：发起退款
```

### 8.1 拿它干什么（这才是重点）

| 线上问题 | 点开哪个 span | 看到什么 |
|---|---|---|
| 「凭什么拒我退款？」 | `guardrail_span` | 输入护栏是否 `tripwire_triggered=True`（被误拦） |
| 「这轮怎么这么贵/慢？」 | `turn_span` + `generation_span` | 每轮 token `usage` + 耗时，定位烧钱那一轮 |
| 「工具是不是报错了？」 | `function_span` | 工具**输入/输出原文**（所以它是敏感数据） |
| 「为什么转给退款专员？」 | `handoff_span` | 交接发生在第几轮、从谁到谁 |
| 「我的风控跑了多久、命中哪条规则？」 | `custom_span` | 你塞的 `data` + 耗时 |

**排查动作**：用户投诉 → 按 `metadata`（`user_id`/`order_no`）或 `trace_id` 捞录像 → 沿异常分支点进去 → 看原文 → 定位。**不用复现、不用猜。**

### 8.2 `trace_id` / `group_id` / `metadata` 的分工

| 字段 | 角色 |
|---|---|
| `trace_id` | **这一趟行程的唯一编号**——贴给同事「你看这个 trace」 |
| `group_id` | 把**同一场对话的多次 trace** 串成一条线（同一 `thread_id`） |
| `metadata` | 贴在录像带上的**自定义标签**，用来**筛选/检索** |

---

## 九、敏感数据

某些 span 会捕获**潜在敏感数据**：

- `generation_span()` 存 **LLM 生成的输入和输出**；
- `function_span()` 存 **函数调用的输入和输出**。

**总开关**：`RunConfig.trace_include_sensitive_data`（默认 **True**）。

```python
runner.run(..., run_config=RunConfig(trace_include_sensitive_data=False))
```

**环境变量设置默认值**：`OPENAI_AGENTS_TRACE_INCLUDE_SENSITIVE_DATA`（`true/1` 或 `false/0`），不用改代码。

**关掉后（=False）会发生什么**：

| 位置 | =False 时的行为 |
|---|---|
| Responses 模型 span | **省略**请求输入和响应输出；对**官方端点**仍保留 `response_id`（关联元数据）；对**自定义端点**连这个 `response_id` 也省掉 |
| 需审批的工具（被暂停） | 不把 SDK 内部结果包装器存为工具输出；若 App 用自定义拒绝消息拒批，**仅当 =True 时**才把它存为输出/错误文本；=False 时 span **省略输出**，用通用错误文本 `Tool execution rejected` |

**音频**：默认音频 span 会存**输入/输出音频的 base64 编码 PCM**。关它用 **`VoicePipelineConfig.trace_include_sensitive_audio_data`**（注意：是和 `trace_include_sensitive_data` **不同的**配置项）。

> ⚠️ **对称性陷阱**：`trace_include_sensitive_data` 管的是**文本**（generation/function span）；**音频**有它自己的开关 `trace_include_sensitive_audio_data`。两个名字像、作用域不同，别只关一个以为就干净了。

---

## 十、长时间运行进程与即时导出

**默认导出器 `BatchTraceProcessor`**：每隔几秒在后台导出 trace；内存队列到达大小阈值时**提前导出**；进程退出时**最后一次刷新**。

- **长跑 worker 场景**（Celery / RQ / Dramatiq / FastAPI 后台任务）：因为进程一直活着，trace **通常无需额外代码就自动导出**——但**每个作业结束后不会立刻**出现在仪表板（要等后台批次/进程退出）。

**要「工作单元结束就立刻送达」** → 在退出 `trace()` 上下文后调用 `flush_traces()`：

```python
from agents import Runner, flush_traces, trace

@celery_app.task
def run_agent_task(prompt: str):
    try:
        with trace("celery_task"):
            result = Runner.run_sync(agent, prompt)
        return result.final_output
    finally:
        flush_traces()                 # 阻塞直到缓冲全部导出
```

**要点**：

- `flush_traces()` **会阻塞**，直到当前缓冲的 trace/span 全部导出。
- ⚠️ **必须在 `trace()` 关闭之后调**——否则会去刷一个**还没构建完**的 trace。
- **禁用追踪后 `flush_traces()` 仍会继续刷**已缓冲的数据：禁用只阻止**新的** trace/span 产生，**不丢弃**处理器里已有的缓冲。

---

## 十一、自定义处理器与数据隐去（点到为止）

**默认架构**：

- 初始化时创建全局 `TraceProvider`（负责创建 trace）；
- 用 `BatchTraceProcessor` 配置它 → 批量把 trace/span 发给 `BackendSpanExporter` → 导出到 OpenAI 后端。

**两种自定义方式**：

| 函数 | 行为 |
|---|---|
| `add_trace_processor()` | **额外加**一个处理器（**保留**默认 OpenAI 导出器）——你还能顺便做自己的处理 |
| `set_trace_processors()` | **替换**默认处理器——除非你自己的处理器里包含导出，否则**不再发往 OpenAI** |

**隐去的关键坑（文档重点）**：处理器是**各自独立**的观察者，默认提供程序会**吞掉某处理器的回调异常**继续叫别人。因此：**即使你在导出器之前注册了隐去处理器，隐去失败也拦不住导出器拿到数据**。

→ 正确做法：**把「隐去」和「交付」放进同一个由 App 拥有的导出器**——用 `set_trace_processors()` 替换默认，导出器**复制**payload → 对副本隐去 → **只**把隐去后的结果送达；序列化/复制/隐去任一步失败就**丢弃整批**，并记一条**不含 payload/异常/回溯**的固定失败消息；失败的批丢弃、后续批继续。

> 📌 这一节属**合规/安全工程**范畴，日常用不到。记住「**隐去必须和交付同处一个导出器，否则拦不住**」这一句即可，真做隐私工程时再回来看实现。（官方示例：`examples/basic/trace_redaction.py`。）

---

## 十二、非 OpenAI 模型的追踪

用**非 OpenAI 模型**时，可以给追踪导出器提供一个 **OpenAI API 密钥**，从而**不禁用追踪**的前提下，在 OpenAI 追踪仪表板里**免费用追踪**：

```python
import os
from agents import set_tracing_export_api_key, Agent

tracing_api_key = os.environ["OPENAI_API_KEY"]
set_tracing_export_api_key(tracing_api_key)   # 全局设置导出用 key
```

**单次运行换 key**（不改全局）：

```python
from agents import Runner, RunConfig

await Runner.run(
    agent,
    input="Hello",
    run_config=RunConfig(tracing={"api_key": "sk-tracing-123"}),
)
```

> 与第五节对照：`RunConfig(tracing={...})` 这个键不只放 `api_key`，第六节的 `include_task_and_turn_spans` 也放这里——**「单次运行的追踪设置」统一从这个口子进**。

---

## 十三、易混概念辨析

| 概念 A | 概念 B | 区别 |
|---|---|---|
| `trace` | `span` | trace=整趟行程（端到端 workflow）；span=其中一段操作 |
| `trace_id`（在 trace 上） | `trace_id`（在 span 上） | 前者=「我是谁」；后者=「我属于哪个 trace」（外键） |
| `trace()` | `RunConfig(tracing=...)` | 前者=**代码里显式建 trace**（可套多 run）；后者=**单次运行**的追踪配置（api_key、include_task_and_turn_spans） |
| `trace_include_sensitive_data` | `trace_include_sensitive_audio_data` | 前者管**文本**（gen/function span）；后者管**音频**（voice pipeline） |
| `add_trace_processor()` | `set_trace_processors()` | 前者**追加**（保留默认导出）；后者**替换**（丢掉默认，除非自己加导出） |
| env `_env_disabled` | `set_tracing_disabled()` 的 `_manual_disabled` | 手动**优先且永久覆盖**环境；一旦手动设过，环境失效 |
| 关 tracing | 关 task/turn span | 前者=**啥都不录**；后者=**只少两层壳**，其它照录 |

---

## 十四、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | `run_config` 这个 Runner 参数是干啥的 | 传**整个运行的配置**（dataclass）。追踪相关的三个字段：`tracing_disabled`、`tracing`（`api_key`/`include_task_and_turn_spans`）、`trace_include_sensitive_data`。是 `Runner.run/run_sync/run_streamed` 的可选参数（`RunConfig \| dict \| None`） |
| 2 | 「更紧凑的层级、单次运行关掉它们」没看懂 | 关的是**两层的容器外壳** `task_span`（每次 runner 调用）和 `turn_span`（每个模型轮次）。它们**不承载业务信息，只负责组织/汇总用量**。关掉=仪表板上少两层壳，`generation_span`/`function_span` 等**仍录**。写法 `RunConfig(tracing={"include_task_and_turn_spans": False})` |
| 3 | 关掉这俩有啥影响？有实际作用吗 | **运行时零影响**（不额外调模型，省 token 无从谈起）。它们的**唯一作用**是 `TaskSpanData`(`name/usage/metadata`) 与 `TurnSpanData`(`turn/agent_name/usage/metadata`) 提供**按任务/按轮次的 token 用量汇总**。关掉=丢这个汇总视图，属**纯展示偏好** |
| 4 | 两个**全局**关闭方法有何区别 | 都走 `DefaultTraceProvider`，但**写的标志位不同**：env→`_env_disabled`（懒读缓存）；`set_tracing_disabled()`→`_manual_disabled`。优先级：**手动 `is not None` 时手动赢**，否则看环境。env **只认 `"true"`/`"1"`** |
| 5 | 调过 `set_tracing_disabled()` 后 `_manual_disabled` 会一直不为 None 吗？ | **✅ 对**。它就是靠 `set_disabled` 赋值，**没有复位路径**——一旦调过，`_manual_disabled` 永久不为 None → **环境变量那条路对你永久失效**，从此一切都听手动的（设 `False` 甚至能反向覆盖环境里的「关」） |
| 6 | `trace`/`span` 在实际应用中怎么用 | 见第八节：SDK 全自动录（task/turn/gen/function/guardrail/handoff 都不用管）；你手工加 `with trace(...)`（圈任务 + `group_id` + `metadata`）和 `custom_span()`（补自己业务逻辑）；出事靠 `trace_id`/`group_id`/`metadata` 捞录像，沿树定位到出问题的 span |

---

## 十五、和 guardrails 的对照

| | guardrails（笔记 05） | tracing（本章） |
|---|---|---|
| 角色 | **在运行中拦一道**（安检门） | **在旁边录下来**（行车记录仪） |
| 时机 | 进门/出门/每次工具调用**前**，`tripwire_triggered=True` → **掐断运行** | **全程旁观**，不干预流程 |
| 可见性 | 只对**触发点**起作用（效果=拦/不拦） | 对**每一步**都留痕（效果=可回溯/可观察） |
| 关闭/停用 | 不挂 / 函数内短路 / 动态构建 | `tracing_disabled` / 环境变量 / `RunConfig` |
| 关系 | 护栏跑的时候，它的执行**也被 tracing 录成 `guardrail_span`** | tracing 是**上位**观察者，把护栏、工具、交接、生成全录进去 |

> **一句话串起来**：护栏负责「**该不该让它过去**」，tracing 负责「**它到底怎么过去的、哪一步出的岔子**」。

---

*追踪 tracing 篇（trace/span 概念 / 默认 10 类 span / 三种关闭方式及优先级 / include_task_and_turn_spans 紧凑层级 / trace() 与 custom_span() 用法 / 敏感数据 / flush_traces / 自定义处理器与生态集成）**已完成**。*

*后续待学：context/sessions、running_agents、results、models。*