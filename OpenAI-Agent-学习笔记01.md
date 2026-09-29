# OpenAI Agent 学习笔记(概念篇)

> 本文档系统梳理了「系统性学习 Agent」第一阶段的全部概念,来源为 OpenAI 官方文档(Agents SDK 中文版)与对话中逐步澄清的心智模型。
> 学习日期:2026-09-24 起 · 学习者:陈朝辉

---

## 目录

1. [学习目标与资料入口](#一学习目标与资料入口)
2. [基础概念:SDK 是什么](#二基础概念sdk-是什么)
3. [Agent 是什么](#三agent-是什么)
4. [Runner 的循环机制](#四runner-的循环机制)
5. [Agent vs Runner 的职责边界](#五agent-vs-runner-的职责边界)
6. [Responses API 是什么](#六responses-api-是什么)
7. [一次完整提问的流转过程](#七一次完整提问的流转过程)
8. [一轮循环的精确定义](#八一抡循环的精确定义)
9. [多 Agent 设计模式与工具、任务转移](#九多-agent-设计模式与工具任务转移)
10. [问答记录(Q&A)](#十问答记录qa)

---

## 一、学习目标与资料入口

### 学习路径(4 步)

1. 读《A practical guide to building agents》→ 建立"何时该用 agent、agent 有哪些形态"的心智模型;
2. 读「Building agents」指南 → 概念术语与官方定义对齐;
3. 跑通 OpenAI Agents SDK 快速开始(`pip install openai-agents`);
4. 重点研究 handoffs 和 tracing 两章。

### 官方文档来源(能直连的中文版)

| 文档 | GitHub Pages 渲染 | 源码 Markdown |
|---|---|---|
| 智能体 agents | https://openai.github.io/openai-agents-python/zh/agents/ | .../docs/zh/agents.md |
| 工具 tools | https://openai.github.io/openai-agents-python/zh/tools/ | .../docs/zh/tools.md |
| 任务转移 handoffs | https://openai.github.io/openai-agents-python/zh/handoffs/ | .../docs/zh/handoffs.md |
| 智能体编排 multi_agent | https://openai.github.io/openai-agents-python/zh/multi_agent/ | .../docs/zh/multi_agent.md |

> 源码 Markdown 的完整前缀:`https://raw.githubusercontent.com/openai/openai-agents-python/main/docs/zh/`

### 网络限制备忘

- ❌ `platform.openai.com`(Building agents 指南)、`cdn.openai.com`(practical guide PDF 白皮书)在国内被墙,打不开。
- ✅ `github.com`、`openai.github.io`(SDK 文档 GitHub Pages)可直连。
- 官方文档有完整中文翻译(`docs/zh/` 目录),无需啃英文。

---

## 二、基础概念:SDK 是什么

**SDK = Software Development Kit,软件开发工具包。**

- **工具包(Kit)**:不是能直接运行的"软件",而是一堆别人打包好的"代码积木"。
- **开发(Development)**:拿来给你**写代码**用的。
- **软件(Software)**:这些积木是**代码库(library)**。

> 一句话:SDK 就是"官方打包好的一套代码积木,让你不用从零开始写,直接拼装"。

**Agent SDK = 帮你搭「AI 智能体」的代码积木包。** 它把"调用模型 → 拿到工具调用 → 执行工具 → 喂回模型 → 再调用"这套循环,连同交接、护栏、会话管理都预先写好了。你只需填配置:

```python
agent = Agent(name="天气助手", instructions="你是天气助手", tools=[get_weather])
result = await Runner.run(agent, "北京天气怎么样?")
```

**易混淆点**:「Agent」有两层含义——
1. **概念上的 Agent**:能自主多轮执行的 AI(方法论,与工具无关);
2. **SDK 里的 `Agent` 类**:`openai/openai-agents-python` 包里一个具体的代码类。

---

## 三、Agent 是什么

### 一句话定义

**Agent = 一个 LLM + 指令(instructions)+ 工具(tools)+ 可选行为配置(交接/护栏/输出格式)。**
它不是新模型,还是那个 LLM,只是配了"手脚"(工具)和"规则"(指令),让它能**自主多轮执行**。

### 一个最关键的分界

| | 直接调 Responses API | 用 Agent + Runner |
|---|---|---|
| 谁管"调工具→拿结果→再喂模型"这个**循环** | 你自己写 while 循环 | SDK 帮你管 |
| 适用 | 想要完全控制循环细节 | 想省事专注业务 |

**Agent 的本质价值 = 编排(orchestration)**,即自动管理"轮次、工具、护栏、交接、会话"。

---

## 四、Runner 的循环机制

### 场景:用户问「北京和上海今天哪个更热?」

#### 不用 SDK,你要亲手写的循环

模型不是一问就答,它有「要工具 → 你执行 → 再喂回」的循环:

```python
response = model(messages=[{"role":"user", "content":"北京和上海哪个更热?"}])
# → 模型回:tool_call get_weather("北京")   ← 不是答案,是要工具

result1 = get_weather("北京")   # → 30°C
response = model(messages=[...])  # 把 30°C 附进去再问
# → 模型回:tool_call get_weather("上海")   ← 还要再查一次

result2 = get_weather("上海")   # → 25°C
response = model(messages=[...])  # 再喂进去
# → 模型回:"北京更热,30°C vs 25°C"       ← 终于答完了
```

**要点**:这个循环要转好几轮,每轮 message 列表变长,你还得自己判断"模型是在要工具,还是答完了"。

#### 用 Agent SDK,一行搞定

```python
agent = Agent(name="天气助手", instructions="根据天气回答问题", tools=[get_weather])
result = await Runner.run(agent, "北京和上海哪个更热?")
print(result.final_output)   # → "北京更热,30°C vs 25°C"
```

`Runner.run()` 在内部把上述循环自动转完,直接给 `final_output`。

---

## 五、Agent vs Runner 的职责边界

> **这是最根本的分界,也是学习者自己澄清出来的关键认知。**

| | 真正的角色 | 性质 |
|---|---|---|
| `Agent` 对象 | **岗位说明书(纸面)** | 静态、被动、没有生命 |
| `Runner` | **正在上班干活的活员工** | 动态、主动,会一步步走、会记事 |

**结论:真正"活着的员工"是 `Runner`,不是 `Agent`。**

- `Agent` 只定义三样静态信息:`name` + `instructions`(怎么干)+ `tools`(能干什么)。放抽屉里啥也不干。
- **"一步步干"和"记住聊到哪"都是 `Runner` 干的**:它维护不断变长的对话记录、驱动模型一步步行动、需要时换人。

> 曾经把 `Agent` 类比成"会自己一步步干的员工"——**这是错的**,它把静态配置和动态运行时行为混在了一起。正确说法:`Agent`=岗位说明书(一页纸),`Runner`=拿着说明书干活的执行引擎。

---

## 六、Responses API 是什么

### 一句话定位

**Responses API = OpenAI 新一代对话接口,专门为 agent/工具调用设计,取代旧的 Chat Completions API。** 它是你(或 SDK)跟模型**对话的底层通道**。

### 与旧接口(Chat Completions)的区别

| | 旧 Chat Completions | 新 Responses API |
|---|---|---|
| 返回什么 | 只有文本 | 文本 **或 工具调用请求**(或其他结构化结果) |
| 一次能干多少事 | 一问一答 | 可同时声明用内置工具(联网搜索/文件检索/跑代码等) |
| 定位 | 简单对话 | 为 agent、多轮、工具调用而设计 |

关键在"或":Responses API 一次调用,模型可能不回文字,而是回一句"帮我调用 `get_weather(city='北京')`"——把"要工具的意图"作为结构化信号吐给你。

### 它只管"对话",不管"循环"

Responses API 只负责"和模型对话、拿信号",**不负责执行工具、也不负责把结果喂回去再循环**。后面这段循环得你自己写,或交给 Agent SDK 的 `Runner`。

### 技术栈全景

```
你的代码
   ├─ 用 Agent SDK 时:Agent(配置) + Runner(循环引擎)
   │        └─ 底层仍调用 ↓
   └─ Responses API  ← 真正和模型对话的接口
              ↓
        OpenAI 的模型
```

> 一句话:Responses API 是"对话管道",Agent SDK 是"架在这根管道上的自动循环机"。

---

## 七、一次完整提问的流转过程

### 演员表(5 个角色)

| 角色 | 它是啥 | 责任 |
|---|---|---|
| **Agent** | 岗位说明书(静态) | 开局提供「指令 + 工具清单」,之后不动 |
| **Runner** | 循环引擎(活的) | 维护聊天记录、驱动循环、判断结束 |
| **Responses API** | 电话线 | 每轮把"记录+指令"发模型,拿回模型的"决定" |
| **模型** | 大脑 | 看信号做决定:回文字 / 要工具 |
| **工具函数** | 手 | 真去执行查天气(北京→30°C) |

> 解开乱麻的关键:**Responses API 不是"新插进来的一层",它就是 Runner 与模型之间那根电话线。循环转几轮,就打几次电话。**

### 时间线(问「北京今天天气怎么样?」)

**准备阶段**(写代码时):`Agent` 只是写好说明书,啥也没发生。

**第 1 轮循环:**

| 序号 | 谁在动 | 干了什么 |
|---|---|---|
| ① | Runner | 打包聊天记录:`messages = ["用户:北京今天天气?"]` |
| ② | Runner | 拿起说明书,拨电话 → 调 Responses API |
| ③ | Responses API | 把「指令+工具清单+记录」发给模型 |
| ④ | 模型 | 决定:"我要调 `get_weather(北京)`" |
| ⑤ | Responses API | 把"要工具"信号递给 Runner |
| ⑥ | Runner | 执行 `get_weather("北京")` → "30°C" |
| ⑦ | Runner | 把结果记进 messages |

**第 2 轮循环:**

| 序号 | 谁在动 | 干了什么 |
|---|---|---|
| ⑧ | Runner | 再拨电话,带上更长的记录 |
| ⑨ | 模型 | "已查到 30°C" → 返回纯文本"北京今天 30°C,晴天" |
| ⑩ | Responses API | 把文本信号递给 Runner |
| ⑪ | Runner | 判定收工 → return 最终答案 |

---

## 八、一轮循环的精确定义

### 术语统一

- **一轮(turn)= 一次 Responses API 调用 = 一次"喂进去 + 拿回来"**;
- 一次 `Runner.run()` = 整个旅程,里面可能转 0~N 轮。

### 一次 Responses API 调用的两件事

**① 喂进去(三样):** 指令(instructions)+ 工具清单(tools)+ 聊天记录(messages)。

**② 拿回来(两种,这是关键):**

| 模型返回 | 含义 | Runner 处理 |
|---|---|---|
| 文本 | 这是最终答案 | 收工,return |
| 工具调用请求 | 要用 XX 工具 | 回头执行工具,结果再喂回 |

**最重要的补充:模型只"说要做",它自己不去做。**
- "我要调 `get_weather`" → 模型说的;
- 真正跑 `get_weather("北京")` 拿 30°C → Runner 回头干的。

> 一次完整调用是不对称往返:Runner 喂一堆东西 → 模型回一个字条(答案 / 要工具)→ 若是"要工具",Runner 挂电话后还得先跑工具,再打下一通电话喂回结果。

---

## 九、多 Agent 设计模式与工具、任务转移

### 两种多 Agent 模式(最重要的心智模型)

- **模式 A · 管理器(agents as tools)**:中央 agent 保留对话控制权,把专业子 agent **当作工具调用**("我调用你")。
- **模式 B · 任务转移(handoffs)**:对等 agent 把控制权**整个转交**给专业 agent,专业 agent 拿历史接管对话("把活儿整个交给你")。

> 一句话记法:"管理器"是"我调用你";"交接"是"我把活儿整个交给你"。这是设计多 agent 系统时最核心的架构选择。

### 工具(Tools)五类

新手先记最常用的两类:
- **FunctionTool**:把 Python 函数包装成工具(标 `@tool` 装饰器),90% 场景的基础;
- **托管工具**:OpenAI 服务器端现成能力——WebSearchTool(联网)、FileSearchTool(查知识库)、CodeInterpreterTool(沙盒跑代码)等。

### 任务转移(handoffs)的两个能力

- **结构化元数据(`input_type`)**:交接时让模型顺便给"原因/优先级",如 `{"reason":"duplicate_charge","priority":"high"}`;
- **输入过滤器(`input_filter`)**:交接时新 agent 默认看得到全部历史,可过滤掉工具调用记录等。

---

## 十、问答记录(Q&A)

以下是本次学习中提出的问题和解答要点(详细内容见对应章节):

| # | 问题 | 结论 |
|---|---|---|
| 1 | platform.openai.com 访问不了怎么办 | 用 GitHub 生态替代:SDK 文档走 openai.github.io 可直连,且官方有中文版 docs/zh/ |
| 2 | Agent SDK 是什么,SDK 是什么 | SDK=软件开发工具包(代码积木);Agent SDK=帮你搭 agent 的积木包 |
| 3 | 能否举例说明 runner 循环 | 天气例子:手写 while 循环 vs Runner.run 一行,见[第四章](#四runner-的循环机制) |
| 4 | agent 是不是"通用函数",runner 传"具体问题";runner 逻辑如何一轮轮跑 | Agent 更像"聘好的员工"而非一次性函数;runner 循环靠判据"文本停/工具继续/交接换人",见[第四、五章](#四runner-的循环机制) |
| 5 | "一步步干、记得聊到哪"不该是 runner 干吗,agent 只定义能干什么怎么干 | **用户直觉正确**:Agent=静态岗位说明书,Runner 才是干活的活物,见[第五章](#五agent-vs-runner-的职责边界) |
| 6 | Responses API 是什么 | 新一代对话接口,为 agent/工具调用设计,取代 Chat Completions;它是"对话管道"而非"循环",见[第六章](#六responses-api-是什么) |
| 7 | 加入 Responses API 后有点乱,各部分如何流转 | 5 角色演员表 + 时间线,见[第七章](#七一次完整提问的流转过程) |
| 8 | 一次 runner 循环是否只含一次 Responses API 调用 | 对。一轮=一次调用=喂进去+拿回来;拿回来分"文本/工具请求"两种,模型只说、Runner 执行,见[第八章](#八一抡循环的精确定义) |

---

*本文档将持续补充:下一步待学 guardrails(护栏)、tracing(追踪)、context/sessions(上下文与会话)。*