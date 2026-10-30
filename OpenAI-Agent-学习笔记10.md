# OpenAI Agent 学习笔记 10（模型篇）

> 本文档是「系统性学习 Agent」第十阶段的整理，聚焦 **模型（Models）**——也就是 **"agent 的大脑从哪来、怎么接线到不同的模型/供应商、哪些参数会在你不注意时被偷偷加上去"**。
> 来源：OpenAI Agents SDK 官方中文文档「模型」（models/index.md）；另核对了 SDK 源码 `agents/models/default_models.py`（`get_default_model` / `get_default_model_settings` / `gpt_5_reasoning_settings_required`）、`agents/model_settings.py`（`ModelSettings` / `resolve`）、`agents/agent.py`（`_initial_model_settings_for_model` / `model_settings` default_factory / `__post_init__` 重算）、`agents/run_internal/turn_preparation.py`（`get_model` / `get_model_settings`）、`agents/models/openai_provider.py`（`get_model` / `_get_client`）、`agents/models/_openai_shared.py`（`set_default_openai_client`）、`agents/models/openai_chatcompletions.py`（请求体装配）。
> 学习日期：2026-10-28（理论学习，**未**写配套 demo）
> 衔接：Agent 静态配置 vs Runner 运行时（笔记 02）、工具五类别（笔记 03）、多 agent 混用与 handoff（笔记 04）、session 四策略（笔记 07）、RunConfig 全局设置（笔记 08）。
> **状态：✅ 本章已学完（2026-10-28 收尾）**（已学：两种原生模型类 / 三层级模型选择 + 优先级 / `ModelSettings` + 暗默认 / Responses-only 工具功能 / 非 OpenAI 三种集成点 / 工作流级混用约束 / 三层心智模型 / 换 client 之谜）。

---

## 目录

1. [官方文档来源](#一官方文档来源)
2. [这一章讲什么：一句话](#二这一章讲什么一句话)
3. [两种原生模型类](#三两种原生模型类)
4. [⭐ 三层级的模型选择 + 优先级](#四-三层级的模型选择--优先级)
5. [`ModelSettings`（模型参数盒子）](#五modelsettings模型参数盒子)
6. [⭐ "暗默认"：为什么 gpt-5 开头就自动带参数](#六-暗默认为什么-gpt-5-开头就自动带参数)
7. [仅限 Responses 的工具功能](#七仅限-responses-的工具功能)
8. [非 OpenAI 模型：三种集成点](#八非-openai-模型三种集成点)
9. [混用约束：工作流级 vs 跨提供商](#九混用约束工作流级-vs-跨提供商)
10. [⭐ 三层心智模型：协议 / SDK 类 / 厂商](#十-三层心智模型协议--sdk-类--厂商)
11. [易混概念辨析（汇总表）](#十一易混概念辨析汇总表)
12. [问答记录（Q&A）](#十二问答记录qa)
13. [和前面章节的对照](#十三和前面章节的对照)

---

## 一、官方文档来源

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 模型 models | https://openai.github.io/openai-agents-python/zh/models/ | https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/models/index.md |

> ⚠️ 获取记录：本机直连 GitHub 不通，经镜像 `https://ghproxy.net/https://raw.githubusercontent.com/...` 取到全文（同 04–09 的通道），已缓存到 `E:\agent_study\docs_cache\models.md`。
> 源码核对：`default_models.py:103` `get_default_model`（默认 `"gpt-5.6-luna"`）、`default_models.py:106` `get_default_model_settings`、`default_models.py:87` `startswith("gpt-5")`、`agent.py:172` `_initial_model_settings_for_model`、`agent.py:367` `model_settings` 的 `default_factory`、`agent.py:517` `__post_init__` 重算、`turn_preparation.py:134` `get_model`、`turn_preparation.py:162` `get_model_settings`、`model_settings.py:254` `resolve`、`_openai_shared.py:27` `set_default_openai_client`、`openai_provider.py:243` `get_model`、`openai_chatcompletions.py:706-707` 请求体装配。

---

## 二、这一章讲什么：一句话

> **Agent 的"大脑"从哪来、怎么接到不同模型/供应商、哪些参数会在你不注意时被偷偷加上去。**

- 前面九章讲的都是"**零件和发动机**"（Agent / 工具 / handoff / 护栏 / tracing / session / Runner / 结果）；
- 本章讲的是"**这些零件背后那个大脑，怎么选、怎么接线**"。

**四个核心问题**：
1. SDK 自带哪两种"接线方式"（Responses vs Chat Completions）？
2. 模型可以在哪三个层级选（全局 / 每次运行 / 每个 agent）？优先级怎样？
3. `ModelSettings` 是什么？为什么 gpt-5 名字会自动带参数？
4. 接非 OpenAI 的模型，怎么写？

---

## 三、两种原生模型类

SDK 自带**两个**原生 OpenAI 模型类，代表**两种 API 形状/格式**：

| 类 | 对接的 API | 状态 |
|---|---|---|
| **`OpenAIResponsesModel`** | Responses API | **推荐** |
| **`OpenAIChatCompletionsModel`** | Chat Completions API | 兼容老协议（**本用户走的是这个**） |

- 二者**只在外层（协议侧）不同**；到了内层（你的世界）都吐出**同样的** `ModelResponse` / `RunItem`。
- **官方建议：一个工作流只用一种形状**（因为二者支持的功能/工具集合不同）——详见第九节。

---

## 四、⭐ 三层级的模型选择 + 优先级

模型可以在**三个层级**指定：

| 层级 | 怎么写 | 作用范围 |
|---|---|---|
| **全局默认** | 环境变量 `OPENAI_DEFAULT_MODEL` / `set_default_openai_client(...)` | 所有 agent、所有 run |
| **每次运行** | `RunConfig(model=..., model_provider=...)` | 一次 `Runner.run` |
| **每个 agent** | `Agent(model=...)` | 单个 agent |

**优先级（源码 `turn_preparation.py:134` `get_model`）**：

> **`RunConfig.model` > `Agent.model` > 全局默认**

（注意：**RunConfig 比 Agent 高**——它是"这一趟运行"的总设置，一次运行里可以盖过单个 agent 的设定。）

**`ModelSettings` 的优先级同理**（`turn_preparation.py:162` `get_model_settings`）：

> **全局默认 → `Agent.model_settings` → `RunConfig.model_settings`（后者逐项 Override 前者）**

> ⚠️ **与笔记 08 呼应**：`RunConfig` 是"**这一趟怎么跑**"的全局设置；`Agent.model` 是"**这个 agent 是谁**"。所以"这趟用什么模型"这种**运行期**的事，RunConfig 说了算。

---

## 五、`ModelSettings`（模型参数盒子）

**独立的 dataclass**（`model_settings.py:88`），装"这一趟模型调用带哪些参数"。常用字段：

| 类别 | 字段 |
|---|---|
| **采样** | `temperature`、`top_p`、`frequency_penalty`、`presence_penalty`、`top_logprobs` |
| **工具** | `tool_choice`、`parallel_tool_calls` |
| **输出/推理** | `reasoning`（含 `effort`）、`verbosity`、`max_tokens`、`truncation` |
| **缓存/存储** | `store`、`prompt_cache_retention`、`prompt_cache_options`、`metadata` |
| **透传** | `extra_query`、`extra_body`、`extra_headers`、`extra_args` |
| **可靠性** | `retry`、`timeout` |
| **诊断** | `include_usage`、`preserve_raw_usage`、`response_include`、`context_management` |

### 5.1 `resolve()` 的合并规则（`model_settings.py:254`）

`base.resolve(override)`：**override 里"非 None"的字段盖掉 base；`extra_args` 是字典合并；`retry` 是深合并。**

---

## 六、⭐ "暗默认"：为什么 gpt-5 开头就自动带参数

### 6.1 现象

SDK 对**名字以 `gpt-5` 开头**的模型，会**自动塞一份默认 `ModelSettings`**；其他名字**啥都不给**。

`default_models.py:106`：

```python
def get_default_model_settings(model=None):
    _model = model if model is not None else get_default_model()
    if gpt_5_reasoning_settings_required(_model):        # ← 看名字是不是 gpt-5
        effort = _get_default_reasoning_effort(_model)   # ← 再按具体变体查表
        if effort is not None:
            return copy.deepcopy(_GPT_5_DEFAULT_MODEL_SETTINGS_BY_REASONING_EFFORT[effort])
        return copy.deepcopy(_GPT_5_TEXT_ONLY_DEFAULT_MODEL_SETTINGS)
    return ModelSettings()                               # ← 非 gpt-5：空的
```

### 6.2 触发条件是**名字字符串**，不是身份 ⭐

`default_models.py:87`：

```python
return model_name.startswith("gpt-5")     # 纯字符串匹配！
```

**它只认名字，不认身份**：
- 你把 deepseek 的模型**起名叫** `"gpt-5-deepseek"` → SDK **照样**给它套上 gpt-5 参数；
- 反过来，名字不以 `gpt-5` 开头的（哪怕是 OpenAI 家）→ **不管**。

> **结论**：是"**因为名字以 gpt-5 开头**"才特殊，**不是**"因为它是 OpenAI 的模型"。

### 6.3 这是**谁的设置**？→ SDK 在本地写死的

- **不是** OpenAI 服务器给的，**不是**模型自带的；
- 是 **SDK 作者在代码里敲进去的一张表**（`default_models.py:16` 起）：
  - `reasoning=Reasoning(effort="low")` + `verbosity="low"`
  - **为什么**：GPT-5 系列天生"话多"（推理一大段、输出啰嗦），SDK 帮开发者**降噪**（少想、少说）；
  - **为什么只照顾 gpt-5**：这是 **OpenAI 官方出的 SDK**，最懂自家模型；**别家不认识，就给空的**。

### 6.4 落到你的代码：为什么你没见过"偷偷带参数"⭐

你传的是 **Model 实例**（不是字符串）：`Agent(model=OpenAIChatCompletionsModel(...))`。链路如下：

**① `Agent` 先塞一份全局默认**（`agent.py:367`）：

```python
model_settings: ModelSettings = field(default_factory=get_default_model_settings)
```

`default_factory=get_default_model_settings` **不带参数**调用 → 用**全局默认模型名**（`OPENAI_DEFAULT_MODEL`，没有就是 `"gpt-5.6-luna"`，`default_models.py:103`）→ 因为是 gpt-5 开头 → **先塞上一份 GPT-5 设置**。

**② 紧接着被"重算"**（`agent.py:517`，关键机关）：

```python
if self.model is not None and self.model_settings == get_default_model_settings():
    self.model_settings = _initial_model_settings_for_model(self.model)
```

翻译：**你要是没自己设 `model_settings`，就按"你这个 agent 实际用的 model"重新算一份。**

**③ 三岔路口**（`agent.py:172`）：

| 你传的 `model=` | 结果 |
|---|---|
| 不传（`None`） | 按全局 `gpt-5.6-luna` → **有一套参数** |
| 字符串 `"deepseek-v4.1-flash"` | 查表 → 不是 gpt-5 → **空** |
| **Model 实例**（← **你的写法**） | **直接空** ⭐ |

```python
def _initial_model_settings_for_model(model):
    if model is None:  return get_default_model_settings()      # ①
    if isinstance(model, str):  return get_default_model_settings(model)  # ②
    return ModelSettings()                                      # ③ ← 你在这
```

> **所以你的 `deepseek-v4.1-flash` 一个默认参数都不会被加**——不是"在照顾你"，是 SDK **压根没往你这边塞**（连"按名字查表"都跳过了：**实例身上没有名字可查**）。

### 6.5 假设实验：改名叫 `gpt-5.6-luna` 会怎样

1. **SDK 端**：`gpt_5_reasoning_settings_required("gpt-5.6-luna")` → True → 查表（`default_models.py:68`）得 effort=**`"none"`** → `ModelSettings(reasoning=Reasoning(effort="none"), verbosity="low")`。
   - ⚠️ **注意是 `none` 不是 `low`**：`gpt-5.6-luna` 在表里对应 `"none"`。
2. **发请求时**：`OpenAIChatCompletionsModel` **真的会塞进 JSON**（`openai_chatcompletions.py:706-707`）：

```python
"reasoning_effort": self._non_null_or_omit(reasoning_effort),   # → "none"
"verbosity": self._non_null_or_omit(model_settings.verbosity),  # → "low"
```

3. **网关端**：你的网关（New API → deepseek）**不认识这两个字段** → 轻则忽略，重则报 400。

> **这实验证明了**：这些"默认参数"是 **SDK 单方面加的**（按**名字**触发、在**本地**执行），跟**对面网关有没有这个概念**毫不相干。

---

## 七、仅限 Responses 的工具功能

**这些只有 `OpenAIResponsesModel` 能用，Chat Completions 会拒绝**：

| 功能 | 是什么 |
|---|---|
| `ToolSearchTool` | 工具搜索（服务端**托管工具**） |
| `tool_namespace()` | 工具命名空间 |
| `@function_tool(defer_loading=True)` | 工具**延迟加载**（Responses 协议机制） |
| `ProgrammaticToolCallingTool` / `allowed_callers` / `tool_choice="programmatic_tool_calling"` | 程序化工具调用 |

### 7.1 为什么只有 Responses 能用？⭐

先把**工具按"在哪执行"分两类**：

| 类型 | 谁执行 | 例子 | 两协议都支持？ |
|---|---|---|---|
| **客户端工具** | **你的机器** | 普通 `@function_tool` | ✅ 都支持（function calling 两协议都有） |
| **服务端托管工具** | **OpenAI 服务器** | web search、file search、computer use | ❌ 只有 Responses |

**关键辨析（拨正一个因果倒置）**：

> **不是"这些工具采用了这个协议写的"，而是"这些工具/机制本身就是这个协议的一部分"。**

- `web_search` / `ToolSearch` 这些，是**在 Responses 协议里被定义、由服务端执行**的条目；
- 老的 **Chat Completions 协议里压根没有这些条目** → 你就算想传，**也没有那个位置可放**。

**反例**：普通 `@function_tool` 两种协议都能用 → 因为它是"**客户端执行 + function calling**"，而 function calling **两协议里都有**，所以它**不挑协议**。

> **一句话**：`OpenAIResponsesModel` 能用它们，不是因为它是"Responses 的翻译器"有什么特别，而是**它对面那个协议/服务端，本来就长着这些能力**；Chat Completions 那套协议里从来没长过。

> ⚠️ **对本站无意义**：本用户的网关只提供 Chat Completions 兼容端点 → 这些功能**本来就用不了**（不管选不选规范）。

---

## 八、非 OpenAI 模型：三种集成点

接非 OpenAI 模型（如本站的 deepseek），SDK 内置**三条路径**：

| # | 写法 | 作用域 | 还能用字符串名？ |
|---|---|---|---|
| ① | `set_default_openai_client(client)` | **全局**（换默认 client） | ✅ 能 |
| ② | `RunConfig(model_provider=...)` | **每次运行** | ✅ 能 |
| ③ | `Agent.model=<Model 实例>` | **单个 agent**（← **本站 demo 用这条**） | ❌ 不用名字 |

（对应文档「在一个工作流中混用模型」的三种选择方式：传名字 / 传名字+provider / 传 Model 实现。）

### 8.1 ⭐ 为什么换 client 之后就能用字符串名？

因为**"字符串名"和"client"本来就是两件独立的东西**：

| 输入 | 管什么 |
|---|---|
| 字符串名 `"gpt-4o"` | 只是个**名字**——"用哪个模型" |
| client（含 `base_url`） | 负责**地址**——"发到哪" |

**字符串名从来不携带地址。** 看源码 `openai_provider.py:243` `OpenAIProvider.get_model("gpt-4o")`：

```python
resolved_model_name = model_name or get_default_model()   # ① 名字：你给的 or 全局默认
client = self._get_client()                               # ② client：从"全局默认"取 ⭐
return OpenAIResponsesModel(model=resolved_model_name, openai_client=client)
```

而 `_get_client()`（`openai_provider.py:150`）里：

```python
default_client = (None if has_explicit_client_options
                  else _openai_shared.get_default_openai_client())   # ← 全局默认 client
```

`set_default_openai_client` 就是**改一个全局变量**（`_openai_shared.py:10,27`）：

```python
_default_openai_client: AsyncOpenAI | None = None
def set_default_openai_client(client):
    global _default_openai_client
    _default_openai_client = client
```

> **所以不是"换 client 让字符串变聪明"，而是"名字一直只负责选模型名，地址一直从全局默认取"——你换 client，换的是"地址那一半"。**

### 8.2 为什么本站 demo 偏偏传 Model 实例（第③条）？

因为要打**公司网关的 deepseek**，默认翻译官（OpenAI 官方 provider）不认识这条路 → **传字符串必然打到 `api.openai.com`**（而你没 key、还被墙）。所以必须**自己把「地址 + 协议」两样交给 SDK**：这两条正好打包进 `OpenAIChatCompletionsModel` 实例。第③条**最直白**（地址/协议/名字全在一处），教学上最清晰。

---

## 九、混用约束：工作流级 vs 跨提供商

### 9.1 "工作流"到底指什么？⭐

**SDK 里没有叫 `Workflow` 的对象**——它是**概念词**。真实边界 = **共享同一份对话历史的运行链**。

| 你的写法 | 一个"工作流"多大 |
|---|---|
| **无状态**（每次 run 独立） | **一次 `Runner.run`** |
| **有记忆**（`session` / `to_input_list()` 接着聊） | **从第一轮 run 到这条对话结束** |

**判断标准**：**这些运行之间，对话历史连不连着？**

- **"一次会话"** ≈ 一个工作流（如果它是带记忆的连续对话）；无状态则每次 run 各算一个。

**为什么边界卡在"历史连着的地方"**：历史项会在**交接时**（同一个 run 内）和**跨轮次时**（同一个 session 内）流动，两端必须**说同一种规范**才读得懂。

### 9.2 官方约束

> 官方建议**每个工作流只使用一种模型接口形态**（二者支持不同的功能和工具集合）。
> 若非要混用，**请确保用到的所有功能在二者中均可用**（= 被限制在**交集**里）。

**正确姿势**：

| 情况 | 做法 |
|---|---|
| 工作流只用普通 `@function_tool` | 两种都行，**统一选一种** |
| 工作流要用 Responses 独有功能（web_search 等） | **整个工作流用 Responses** ✅ |
| 两个独立工作流，一个要 Responses、一个不要 | **各选各的**，互不影响 |

> **粒度**：是"**工作流级**"，**不是**"agent 级随便混"——因为交接会把历史传下去。

### 9.3 跨提供商混用（功能差异）

文档「跨提供商混用模型」提醒：

- **不要**向不支持的提供商发它不认识的 `tools`；
- 调**纯文本模型前**，过滤掉多模态输入；
- 不支持结构化 JSON 输出的提供商，**偶尔会吐无效 JSON**。

---

## 十、⭐ 三层心智模型：协议 / SDK 类 / 厂商

本章最重要的"认知地基"，把三样东西彻底分开：

| 层 | 是什么 | 类比 |
|---|---|---|
| ① **协议 / 规范** | "OpenAI 兼容"= 大家**采用的那套定义** | 大家约好说"普通话" |
| ② **SDK 类**（`OpenAIChatCompletionsModel` 等） | **在你机器上**跑的**翻译代码**（客户端） | 你的**随身翻译** |
| ③ **厂商服务端** | 模型厂商**自己的实现**，**原生就说这个协议** | **原生说普通话的人** |

**几个关键点**：

- **"OpenAI"前缀是"API 格式"，不是"厂商"**。厂商（deepseek）**不用翻译**——它**天生就说这个协议**（因为它**照这个规范实现了服务端**）。
- SDK 类是**双向翻译器**，且**两个方向都是"为你（客户端）"服务的**：把你的参数转成协议格式发出去，再把协议响应转回你的世界的 `ModelResponse`/`RunItem`。
- **"厂商"= 模型/LLM 厂商**，**不是"agent 厂商"**。你开发的 agent **没有大脑**，是**租**模型厂商的 LLM。
- **handoff 里的"其他 agent"是你自己写的代码**，与厂商无关。

**你的代码有两半**：

| 半边 | 内容 | 依赖协议吗？ |
|---|---|---|
| **主体** | `Agent` / `Runner` / tools / handoffs | ❌ 协议无关 |
| **接线** | `AsyncOpenAI(base_url=…)` + 选哪个 Model 类 | ✅ 协议相关 |

---

## 十一、易混概念辨析（汇总表）

### 11.1 本章最核心的"三兄弟"（认知地基）

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **协议/规范** | **SDK 类** | 前者=大家**采用的"定义"**；后者=**你机器上的翻译代码** |
| **SDK 类** | **厂商服务端** | 前者=**对你**做双向翻译；后者=**原生说这个协议、不需翻译** |
| **`OpenAIChatCompletionsModel`** | **`OpenAIResponsesModel`** | 前者=Chat Completions 形状；后者=Responses 形状。**只在外层（协议侧）不同**，内层都吐同样的 `ModelResponse`/`RunItem` |

### 11.2 三种（选择/集成）方式

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **`set_default_openai_client`** | **`RunConfig(model_provider=…)`** | 前者=**全局**换 client；后者=**每次运行** |
| **`RunConfig(model_provider=…)`** | **`Agent.model=<实例>`** | 前者=run 级，**保留字符串名**；后者=agent 级，**直接用成品** |
| **传字符串名** | **传 Model 实例** | 前者=只是个名字，**地址从全局默认取**；后者=地址+协议+名字**焊在一起** |

### 11.3 "默认参数"相关

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **`ModelSettings`（显式）** | **"暗默认"（隐式）** | 前者=你写的；后者=SDK 按**名字**（gpt-5 开头）自动补的 |
| **触发"暗默认"的原因** | **是不是 OpenAI 的模型** | ❌ **不是身份**；✅ **是"名字以 `gpt-5` 开头"**（纯字符串匹配） |
| `Agent.model_settings` default_factory | `_initial_model_settings_for_model` | 前者=**先**按全局默认名（gpt-5.6-luna）塞；后者=**马上**按"你这个 agent 的实际 model"重算 |
| 传 `None` / 传字符串 / 传实例 | 暗默认结果 | 分别 = **有参数 / 查表 / 空** |

### 11.4 "工作流"相关

| 概念 A | 概念 B | 区别 |
|---|---|---|
| **"工作流"** | **`Workflow` 对象** | 前者=**概念词**（共享历史的运行链）；后者=**SDK 里根本没有这个类** |
| **工作流级统一规范** | **agent 级随便混** | 前者=**官方建议/正解**；后者=会因交接传历史而崩，只能留在交集里 |

---

## 十二、问答记录（Q&A）

| # | 问题 | 结论 |
|---|---|---|
| 1 | "`OpenAIResponsesModel` 和 `OpenAIChatCompletionsModel` 都是 openai 提供的吗？现在有什么 agent 用这个 API？" | 都是 **OpenAI 官方 SDK 提供**的**两个类**。它们对接的还是 **OpenAI 自家两种 API**；只是**别的厂商也照这套协议实现了服务端**，所以能拿来接别家（"OpenAI 兼容"）。 |
| 2 | "它们只是一个定义，其他厂商只是采用了这个定义？" | ⚠️ **半对**。**协议/规范**才是"大家采用的定义"；**SDK 类是你机器上的翻译代码**，**不是别家'采用'的东西**——别家采用**的是协议**，SDK 类是**你"翻译给协议"的工具**。 |
| 3 | "SDK 类负责翻译，是不是也给厂商服务端翻译？" | SDK 是**双向翻译器**，但**两个方向都是"为你（客户端）"服务的**：出=把你的调用转成协议格式；回=把协议响应转回你的世界的 `ModelResponse`/`RunItem`。**厂商服务端不用翻译**（它原生说协议）。 |
| 4 | "我是在开发 agent 吗？换协议后我的代码还能跑吗？为什么厂商天生会说普通话？" | ✅ 你在开发 agent。你的代码分两半：**主体**（Agent/Runner/tools/handoffs，**协议无关**）+ **接线**（`AsyncOpenAI(base_url)` + 选哪个 Model 类，**协议相关**）。**换协议**大多只动"接线"那一半。厂商"天生会说"是因为它**照这个规范实现了服务端**。 |
| 5 | "SDK 把方言转成普通话后，厂商接收、调用模型回答，再把答案转回方言给我？" | ✅ **基本对**，只纠一处主语：**把答案转回你的方言的是 SDK（不是厂商）**。厂商只管"原生说普通话、吐普通话"。 |
| 6 | "我不是在开发 agent 吗？咋还需要其他 agent 的厂商？" | **"厂商"= 模型/LLM 厂商**，**不是"agent 厂商"**。你的 agent **没有大脑**，是**租**模型厂商的 LLM。handoff 里的"其他 agent"（前台客服→退款专员）**都是你自己写的代码**，与厂商无关。 |
| 7 | "只有 Responses 才能用的那些工具，是 openai 提供给开发者的吗？为什么只有 Responses 能用？是因为工具用协议写的吗？" | ① 那些多是 **Responses 的服务端托管工具/协议机制**（web search、ToolSearch、defer_loading…）；② 只有 Responses 能用的原因**不是"工具挑协议"**，而是**它们本身就是定义在 Responses 协议里的东西**——老协议 Chat Completions 里**没这个概念、没这个字段、没这套服务端实现**；③ **拨正因果**：不是"工具用协议写的"，而是"**协议里定义了这些工具**"。**反例**：普通 `@function_tool` 两协议都能用（客户端执行 + function calling，两协议都有）。 |
| 8 | "我只能用一种规范吗？某工作流用到 Responses 功能就整个用 Responses？" | ✅ **对**。规范是**"按工作流"选**的，**不是锁死的**。不同工作流可各选各的。**但不能在同一工作流内混**（交接会把历史传下去，另一端读不懂）。用到 Responses 独有功能 → **整条工作流走 Responses**（官方推荐）。 |
| 9 | "工作流这个层级具体表现是什么？一次会话？" | **最接近的答案**。SDK **没有** `Workflow` 类。真实边界 = **共享同一份对话历史的运行链**：无状态→一次 `run`；带记忆→一条 session 从第一轮到结束。**判断标准：对话历史连不连着。** |
| 10 | "模型名叫 gpt-5 开头就默认带参数，为什么？因为是 openai 的模型吗？谁的设置？" | **谁的设置** → **SDK 自己在本地写死的一张表**（`default_models.py`），**不是** OpenAI 服务器、**不是**模型自带。**触发条件** → **名字字符串以 `gpt-5` 开头**（`startswith("gpt-5")`），**不是"因为它是 OpenAI 的模型"**——你若给 deepseek 起名叫 `gpt-5-xxx`，**照样**会被套上。**为什么** → GPT-5 天生话多，SDK 帮你降噪；别家不认识 → 给空的。 |
| 11 | "第五大点（落到你的代码）没太看懂，详细讲" | 三步：① `Agent` 先按**全局默认名**（gpt-5.6-luna）塞一份；② `__post_init__` 发现你没自己设 → **按你说的 model 重算**；③ `_initial_model_settings_for_model` 三岔：**None→有参数 / 字符串→查表 / 实例→空**。**你传的是实例 → 空**（连查表都跳过：实例没名字可查）。⚠️ 顺带更正前一版口误：`gpt-5.6-luna` 对应的是 **`effort="none"`**，不是 `low`。 |
| 12 | "为啥我会传 Model 实例？难道不是都传模型名字？" | **传名字只是三种方式之一**，且有**隐藏前提**：得有能翻译这个名字的**翻译官**（默认是 OpenAI 官方 provider → 指向 `api.openai.com`）。你**没官方 key + 被墙** → 传字符串必失败。所以要**自己当翻译官**，把「**地址** + **协议**」两样交给 SDK，`OpenAIChatCompletionsModel` 实例正好是装这两样的盒子。 |
| 13 | "为啥换 client 之后就能用字符串名？三种实现方式有什么区别？" | **换 client 之谜**：字符串名**从来不带地址**；名字负责"选模型"、client 负责"指地址"（`_get_client` 从**全局默认**取）。换 client = 换"地址那一半"。**三种方式区别**：不在"要不要 url+名字"（都要），而在**作用域**（全局 / run / agent）+ **谁当翻译官**。 | 
| 14 | "用户选模型是全局的，什么场景会同一工作流不同 agent 用不同模型？" | **【人选】vs【系统选】**是不同层面。per-agent 混用在【系统选】：① **成本/延迟分层**（小模型分流、大模型攻坚——最主流）；② **能力差异**（多模态/代码/embedding/语音）；③ **合规**（敏感数据走私有部署，数据不出院）；④ **专用模型**（审核/翻译/意图识别）；⑤ **灰度 A/B**；⑥ **推理强度分层**。**何时没必要**：任务同质 / 为炫技 / 会破坏"统一协议"约束。 |

---

## 十三、和前面章节的对照

| | 前九章（零件 / 发动机 / 行程单） | 本章（大脑怎么接） |
|---|---|---|
| 角色 | 造零件 + 装上 `Runner` 转起来 + 读交卷 | **选大脑、接大脑、管大脑参数** |
| 关键词 | `Agent`、工具、`handoff()`、`tripwire`、`span`、`session`、`Runner.run`、Agent Loop、`RunResult` | `OpenAIResponsesModel`/`OpenAIChatCompletionsModel`、三层级模型选择、`ModelSettings`、暗默认、`set_default_openai_client`、`ModelProvider`、工作流级混用约束 |
| Agent/Runner（笔记 02） | Agent=静态"谁是"，Runner=运行时引擎 | **`Agent.model`** 属"谁是"（静态）；**`RunConfig(model=)`** 属"这一趟怎么跑"（运行期） |
| RunConfig（笔记 08） | "这一趟怎么跑"的全局设置 | 其中 **`model` / `model_provider` / `model_settings`** 本章详解，且**优先级高于 Agent** |
| session/上下文（笔记 07） | 四策略共享对话历史 | **"工作流"的边界 = 共享历史的那条链**（thus 同工作流必须同规范） |
| handoff（笔记 04） | 平级交接 | 交接会**传历史** → 所以**同一工作流内不可混协议形状** |

> **一句话串起来**：前面九章教你"**造零件、装发动机、读卷子**"；本章教你"**这颗大脑从哪来、怎么换供应商、怎么调参数、以及哪些参数是 SDK 背着你加的**"。

---

*模型篇（两种原生模型类 / 三层级模型选择 + 优先级 / `ModelSettings` + 暗默认 / Responses-only 工具功能 / 非 OpenAI 三种集成点 / 工作流级混用约束 / 三层心智模型（协议·SDK 类·厂商）/ 换 client 之谜 / per-agent 混用场景）**已完成**。*

*后续待学：streaming（流式）。*