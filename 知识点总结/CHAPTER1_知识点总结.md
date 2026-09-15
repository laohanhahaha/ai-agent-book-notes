# Chapter 1 知识点总结：Agent 基础与 ReAct 流程

> 本笔记以 `web-search-agent/agent-原版.py` 的 Kimi Formula 实现为主要代码参照，并补充第一章的整体概念。当前 `agent.py` 已改为 DeepSeek Responses API 实现，两者的工具执行位置不同，不要混为一谈。

> 2026-09-14 复核：对照[第一章正文](../book/chapter1.md)、[原版代码](web-search-agent/agent-原版.py)、[当前代码](web-search-agent/agent.py)与[实验台账](EXPERIMENT_LEDGER.md)修订。下文的具体请求结构描述本地实现，不代表所有模型服务的统一协议；本次未重新运行 API 实验。

## 1. 第一章的核心公式

现代 Agent 可以概括为：

```text
Agent = LLM + 上下文（Context）+ 工具（Tools）
```

- **LLM 是大脑**：理解用户意图、规划步骤、选择工具、判断是否继续。
- **上下文是眼睛**：模型在当前决策点能看到的系统提示、用户问题、历史消息、工具调用和工具结果。
- **工具是手脚**：搜索网页、读取文件、运行代码或调用外部 API，使模型能够获得新信息或改变外部状态。

模型本身只负责生成决策；真正的工具执行、上下文维护、错误处理和循环控制通常由 Agent 外层的 Python 程序负责。这层工程外壳也叫 **Harness**。

工具也可能由远端服务执行；本地程序负责调度与传递结果，不一定亲自实现搜索、数据库等底层能力。

### 1.1 观察、行动与策略

- **观察空间**：模型通过上下文能获得哪些环境信息。
- **动作空间**：系统允许 Agent 发起哪些操作。
- **策略**：模型根据当前信息选择下一步的决策方式。

在模型不变时，提供缺失的业务数据或必要工具，就可能解决原来做不了的任务。但扩大能力必须同时设计权限与验证，不能只增加接口数量。

本章广义工具分为感知、执行、协作、事件触发、用户沟通五类。其中事件触发是外部事件唤起 Agent，不是模型主动生成的一次函数调用。

### 1.2 Agent 在哪里“学习”

| 路径 | 改变什么 | 持续范围 |
|---|---|---|
| 上下文适应 | 当前输入中的事实、示例、反馈 | 信息仍在当前上下文中时 |
| 外部产物更新 | 记忆、知识文档、Prompt、Skill、程序 | 保存后可跨任务使用，但仍需加载或调用 |
| 参数更新 | 模型权重 | 部署更新后的模型时 |

一次聊天变得更贴合用户，不等于模型自动训练了自己的参数。这三条路径可以结合使用。

## 2. ReAct：推理—行动—观察

ReAct 是 Reasoning and Acting 的缩写，基本循环是：

```text
用户输入
   ↓
Reasoning：分析当前信息，决定下一步
   ↓
Action：申请调用某个工具，并给出参数
   ↓
Observation：获得工具返回结果
   ↓
将结果加入上下文，再次 Reasoning
   ↓
继续调用工具，或生成最终答案
```

Web Search Agent 的关键不在于“调用一次搜索接口”，而在于模型可以根据每次搜索结果动态决定：

- 信息足够：生成最终答案；
- 信息不足：修改关键词并再次搜索；
- 达到最大轮数、发生异常或耗尽输出预算：停止并返回相应结果。

## 3. 代码中的三个入口层次

### 3.1 程序入口：`main()`

`main-原版.py` 负责：

- 解析命令行参数；
- 创建 `WebSearchAgent` 实例；
- 选择单问题模式或交互模式；
- 调用 Agent 并展示、保存结果。

```python
agent = WebSearchAgent(...)
answer = agent.search_and_answer(question)
```

创建实例时，Python 会自动执行 `WebSearchAgent.__init__()`。

注意：`main-原版.py` 中实际写的是 `from agent import ...`，不会自动导入 `agent-原版.py`。两个“原版”文件可以配对阅读，但直接运行旧入口仍会加载当前 `agent.py`，存在接口不匹配的可能。

### 3.2 Agent 任务入口：`search_and_answer()`

`search_and_answer()` 是一次完整搜索问答任务的总控制器。它负责：

1. 创建初始消息历史；
2. 重置轨迹和请求记录；
3. 启动 ReAct 循环；
4. 调用 `_chat()`；
5. 根据 `finish_reason` 处理工具调用、截断或最终回答；
6. 执行工具并更新历史；
7. 达到结束条件后返回答案。

### 3.3 真正的模型请求：`_chat()`

原版中实际发送模型请求的是：

```python
completion = self.client.chat.completions.create(**kwargs)
```

因此完整调用链是：

```text
main()
  └─ WebSearchAgent(...)
       └─ 自动执行 __init__()
  └─ search_and_answer(question)
       └─ _chat(conversation_history)
            └─ client.chat.completions.create(...)
```

## 4. `__init__()` 初始化了什么

常见实例属性及用途：

| 属性 | 作用 |
|---|---|
| `self.client` | 配置好的模型 API 客户端 |
| `self._api_key` | API 身份凭证 |
| `self.base_url` | API 基础地址 |
| `self.model` | 使用的模型名称 |
| `self.verbose` | 是否实时打印 ReAct 轨迹 |
| `self.conversation_history` | 当前问答任务的消息历史 |
| `self.trace` | 面向人阅读的思考、行动、观察、答案轨迹 |
| `self.api_turns` | API 请求、响应、状态码、耗时和错误记录 |
| `self.formula_uri` | Kimi 网页搜索 Formula 标识 |
| `self._formula_tools` | 工具声明缓存，初始为 `None` |
| `self._request_timeout` | 请求超时时间 |
| `self.temperature` | 生成随机性参数 |
| `self.max_tokens` | 推理和回答可使用的输出预算 |

`self.client = OpenAI(...)` 只是创建并配置客户端，并不会立即发起会话。真正的网络请求发生在调用客户端的 `create()` 方法时。

## 5. 原版 Kimi 工具调用链

### 5.1 获取工具声明

原版没有把完整工具 JSON 写死在代码里，而是由 `_get_tools()` 请求 Kimi：

```text
GET /formulas/moonshot/web-search:latest/tools
```

服务器返回的 `payload` 中包含 `tools` 列表。程序会：

1. 使用 `response.json()` 把 JSON 响应解析成 Python 对象；
2. 使用 `payload.get("tools")` 读取工具列表；
3. 检查它是否为非空列表；
4. 检查其中是否存在名为 `web_search` 的 function；
5. 将结果保存到 `self._formula_tools` 后返回。

缓存判断：

```python
if getattr(self, "using_openrouter", False):
    return []

if self._formula_tools is not None:
    return self._formula_tools
```

- 本实现的 OpenRouter 兜底分支没有接入 Kimi Formula，因此返回空列表；这不是对 OpenRouter 所有工具能力的断言；
- 已有缓存时直接返回，避免在同一次问答的多轮模型调用中重复请求工具声明。

每次新的 `search_and_answer()` 都会把 `_formula_tools` 重置为 `None`，所以这里的缓存主要作用于同一个问题内部。

### 5.2 把工具声明交给模型

`_chat()` 先构造 `kwargs`，再按条件增加工具：

```python
tools = self._get_tools()
if tools:
    kwargs["tools"] = tools
```

`kwargs` 是普通字典；调用时的 `**kwargs` 表示将字典拆成关键字参数。

模型看到工具声明后，知道：

- 有哪些工具；
- 每个工具有什么用途；
- 参数名称和参数类型；
- 应该以什么结构发起调用。

### 5.3 模型生成工具调用请求

模型应从已声明的工具中选择，并返回工具名称和参数，不是把函数实现一起生成出来。执行层仍需检查未知工具名和非法参数。下面只是结构示意，`query` 并非这里已验证的 Kimi Formula 完整参数 schema，真实定义以 `_get_tools()` 的响应为准：

```json
{
  "id": "call_123",
  "type": "function",
  "function": {
    "name": "web_search",
    "arguments": "{\"query\": \"2024 年诺贝尔物理学奖得主\"}"
  }
}
```

其中：

- `name` 是模型选择的工具名称；
- `arguments` 是模型生成的 JSON 参数字符串；
- `id` 用来把工具结果与本次工具调用对应起来。

### 5.4 Python 执行工具

`search_and_answer()` 读取模型的工具请求，然后调用：

```python
self._execute_formula(name, raw_arguments)
```

`_execute_formula()` 把名称和原始参数发送给 Kimi Formula Fiber：

```text
POST /formulas/moonshot/web-search:latest/fibers
```

职责分工是：

```text
Kimi 聊天模型：决定是否搜索、搜索什么
本地 Python：维护循环、解析并转发工具调用
Kimi Formula：真正执行网页搜索并返回结果
```

原版 `_execute_formula()` 转发的是原始参数字符串；循环中的 `json.loads()` 主要用于展示轨迹。返回值可能来自 `context.output`，也可能是 `encrypted_output`，不保证总是可直接阅读的网页正文。

## 6. `messages` 如何随循环变化

初始历史：

```python
[
    {"role": "system", "content": "系统提示"},
    {"role": "user", "content": "用户问题"},
]
```

模型申请调用工具后，加入一条助手消息：

```python
{
    "role": "assistant",
    "content": "",
    "tool_calls": [...]
}
```

工具执行结束后，加入一条工具消息：

```python
{
    "role": "tool",
    "tool_call_id": "call_123",
    "content": "搜索结果"
}
```

随后完整历史再次发送给模型：

```text
system
user
assistant(tool_calls)
tool(搜索结果)
```

如果模型再次申请工具，历史会继续增加新的 `assistant(tool_calls)` 和 `tool` 消息。生成最终答案后，再加入：

```python
{
    "role": "assistant",
    "content": "最终答案"
}
```

关键认识：**本原版实现**使用显式历史，每一轮由 Python 重新发送当前完整 `messages`。有些接口支持服务端会话或通过先前响应 ID 引用历史，不要求客户端重复传输所有文本；但模型仍需获得任务所需的上下文，服务端会话也不等于无限长期记忆。

原版 `search_and_answer()` 在每个新问题开始时会重新创建 `conversation_history`，所以它保留的是“一个问题内部”的工具循环，而不是默认保留多个用户问题之间的长期对话。

## 7. ReAct 循环如何结束

`choice.finish_reason` 决定下一步：

| `finish_reason` | 含义 | 程序行为 |
|---|---|---|
| `tool_calls` | 模型申请调用工具 | 执行工具、追加结果、继续循环 |
| `stop` | 正常停止生成 | 内容非空时返回，否则走未取得答案的兜底 |
| `length` | 达到 `max_tokens` 上限 | 返回部分内容和截断提示 |
| 其他值 | 需按接口判断具体原因 | 原版放入 `else`，有文本就返回，无文本则兜底 |

原版的 `else` 是简化实现，不能据此认为“所有非工具响应都是正常最终答案”。生产系统应专门处理过滤、拒绝、异常终止和无有效内容等情况。

另外，`max_iterations` 限制的是本地循环中的**主模型调用轮数**，不是工具调用总次数。一轮返回多个 `tool_calls` 时会逐个执行。如果最后允许的一轮仍在申请工具，执行完后也可能因为没有下一轮模型预算而返回上限提示。

“模型判断信息是否足够”不是一个明确的满意度分数，而是模型根据系统提示、当前历史和工具结果，决定继续产生 `tool_calls`，还是产生最终文本。

## 8. 三类记录不要混淆

| 数据 | 主要用途 | 是否发给下一轮模型 |
|---|---|---|
| `conversation_history` | 模型上下文 | 是 |
| `trace` | 向用户展示 ReAct 过程 | 否，主要用于展示与调试 |
| `api_turns` | 保存真实 API 请求、响应、耗时和错误 | 否，主要用于调试和验收 |

这些数据默认保存在 Python 进程内存中。除非使用输出功能写入 JSON 文件，否则程序结束后不会自动持久保存。

`trace` 中的“思考”来自接口实际返回的可见字段，不保证代表模型完整的内部推理；没有可见思考文本也可以进行工具循环。实验 1-1 的仓库记录中，移除显式 reasoning 的分支仍能完成案例，因此不能把“打印思考”当成 ReAct 生效的必要条件。

“`api_turns` 不作为独立字段发给模型”不表示它与上下文完全无关：其中可包含请求历史和响应副本。去掉认证头只代表不直接记录该凭证，不代表用户内容已经脱敏。

## 9. JSON 与 SDK 对象

### JSON 转换

```text
json.loads()：JSON 字符串 → 对应的 Python 对象
json.dumps()：可 JSON 序列化的 Python 对象 → JSON 字符串
json.dump()：把 Python 对象直接写入 JSON 文件
```

JSON 还可以表示字符串、数字、布尔值和 null，因此 `loads()` 不一定返回字典或列表。工具参数若要求对象，解析后还要检查类型与字段。

工具参数通常以 JSON 字符串返回，所以需要：

```python
arguments = json.loads(tool_call.function.arguments or "{}")
```

工具结果如果不是字符串，则使用：

```python
json.dumps(tool_result, ensure_ascii=False)
```

`ensure_ascii=False` 可以让中文和其他非 ASCII 字符保持原样显示。

### Pydantic message

OpenAI SDK 返回的 `choice.message` 是带属性的数据模型对象，不是普通字典：

```python
choice.message.content
choice.message.tool_calls
```

原版会手动重建只包含合法字段的消息字典，避免把 SDK 对象中的额外字段原样回传给服务端。

这描述的是原版处理方式，不是“所有模型都应该丢弃额外字段”的通用规则。有些协议要求保留特定推理字段或签名，应按实际接口约定处理。

## 10. 异常、超时与可观测性

常见保护机制：

- `timeout`：避免请求长时间挂起；
- `response.raise_for_status()`：遇到 HTTP 4xx/5xx 时抛出异常；
- `try/except`：捕获请求、解析和执行错误；
- 单独的 `raise`：记录错误后继续抛出原异常；
- `time.monotonic()`：计算稳定的请求耗时；
- `api_turns`：记录请求地址、方法、状态码、响应和错误；
- `max_iterations`：防止无限工具循环；
- `max_tokens`：限制模型输出预算并识别截断。

这些代码就是 Harness 的一部分：模型负责决策，工程代码负责让决策过程可控、可查和可恢复。

## 11. 本章出现的常用 Python 知识

### 类型标注

```python
List[Dict[str, Any]]
```

表示“一个列表，其中每个元素都是字典；字典键是字符串，值可以是任意类型”。

```python
Optional[List[Dict[str, Any]]]
```

表示该值可以是工具列表，也可以是 `None`。

这些是类型标注，普通 Python 运行时不会仅因为写了标注就自动强制检查；需要类型检查工具或运行时验证逻辑。

### 常用内置函数

| 函数 | 作用 |
|---|---|
| `getattr(obj, name, default)` | 安全读取对象属性，不存在时返回默认值 |
| `isinstance(value, type)` | 判断对象是否属于指定类型 |
| `any(items)` | 判断是否至少有一个元素为真 |
| `len()` | 获取长度 |
| `str()` | 转换成字符串 |
| `type()` | 获取对象类型 |
| `round()` | 四舍五入 |
| `print()` | 输出内容 |

`typing.Any` 是类型标注中的“任意类型”，而小写 `any()` 是判断“是否至少一个为真”的内置函数。

### 常用对象方法

| 方法 | 所属对象 | 作用 |
|---|---|---|
| `.append()` | 列表 | 添加一个元素 |
| `.get()` | 字典 | 安全读取键值 |
| `.strip()` | 字符串 | 删除首尾空白 |
| `.rstrip("/")` | 字符串 | 删除结尾的 `/` |
| `.lower()` | 字符串 | 转为小写 |
| `.json()` | HTTP 响应 | 解析 JSON 响应 |
| `.raise_for_status()` | `requests.Response` | 检查 HTTP 错误状态 |

### 列表推导式

```python
[expression for item in items]
```

它会逐个处理元素并直接生成新列表，效果相当于：

```python
result = []
for item in items:
    result.append(expression)
```

列表推导式内部逐个处理 `tool_call`，最后将包含全部工具调用的列表作为一条助手消息加入 `conversation_history`。

### 条件表达式

```python
value_if_true if condition else value_if_false
```

例如：

```python
tool_content = (
    tool_result
    if isinstance(tool_result, str)
    else json.dumps(tool_result, ensure_ascii=False)
)
```

### 命令行参数

`argparse` 用来定义命令行配置：

- `nargs="*"`：接收零个或多个参数；
- `choices=[...]`：限制可选值；
- `default=...`：设置默认值；
- `type=int`：转换为整数；
- `action="store_true"`：参数出现时设置为 `True`；
- `help=...`：生成帮助说明。

```python
question = " ".join(args.query).strip()
```

Shell 通常按未被引号保护的空白划分参数，`nargs="*"` 再把零个或多个位置参数收集成列表；上式用空格重新连接并去掉首尾空白。带引号的问题可以作为单个参数，不会被 `argparse` 再按内部空格拆开；无输入时得到空字符串，供入口决定进入交互模式。这不会原样恢复用户输入的全部空白格式。

## 12. 工作流与自主 Agent

- **工作流**：编排结构由代码预先规定，可包含分支、循环、重试和并行，适合明确的业务流程；结果仍需验证，不会因路径固定就自动可靠。
- **自主 Agent**：模型根据环境反馈动态决定下一步，灵活但成本和不确定性更高。
- **混合模式**：关键、高风险步骤使用固定工作流，开放式搜索或分析部分允许模型自主决策。

Web Search Agent 属于较简单的自主 Agent：Python 固定循环框架和停止条件，模型在循环内部动态决定搜索关键词以及是否继续。

## 13. Harness 工程的五个功能

围绕模型构建生产级 Agent 时，重点不只是提示词，还包括：

1. 上下文管理：模型每一步能看到什么；
2. 工具接口：模型能够做什么、参数是否明确；
3. 权限与护栏：哪些操作允许自动执行；
4. 验证：检查输入、工具结果和交付产物是否满足要求；
5. 纠正：发现偏差后重试、调整、回滚或转人工。

这对应书中的“上下文、工具、约束、验证、纠正”。**可观测性**贯穿这五项，通过轨迹、耗时、请求和错误记录帮助定位问题，不是替代“纠正”的第五项。

安全护栏通常分为输入侧、执行侧和输出侧。高风险、不可逆操作不能只依赖模型判断，应增加参数验证、权限限制、预览或人工确认。

实现时优先保持简单、执行透明、接口清晰。模型选型要在自己的任务上比较准确性、工具能力、成本、延迟、数据边界与使用限制，而不是只看排行榜。

提示工程关注指令，上下文工程管理输入信息，Harness 工程组织执行与验证；再将评估过的运行反馈用于更新知识、指令或程序，形成改进闭环。这是关注范围的扩展，不意味着后者使前者失效。

## 14. 当前版与原版的区别

### `agent-原版.py`

```text
主模型返回 function tool_call
    ↓
Python 调用 Kimi Formula Fiber
    ↓
Python 把工具结果追加到 messages
    ↓
再次调用主模型
```

这是你本次逐行理解的客户端 ReAct 工具循环。

### 当前 `agent.py`

当前文件使用 DeepSeek Responses API。主模型先调用本地声明的 `web_search` function，Python 再发起一个独立 Responses 请求，强制使用 DeepSeek 服务端托管的 `web_search`，最后通过 `function_call_output` 把结果交还给主模型。

两种实现的接口结构不同，但核心范式不变：

```text
模型决策 → 工具行动 → 获得观察 → 更新上下文 → 再次决策
```

## 15. 学完本章后应能回答的问题

> 参考答案补充于 2026-09-15。下文按原问题逐题作答；“原文依据”是短摘录，答案是结合上下文的归纳，代码细节另列来源。链接定位到本地原文或代码的具体行，行号对应本次核对版本。

### 15.1 Agent 与普通单次 LLM 问答有什么区别？

**参考答案：** 普通单次问答主要是“输入问题 → 模型生成回答”。本章的 Agent 则能把任务拆成行动，调用外部工具，读取反馈，再决定继续执行还是结束。关键是模型、工具和环境反馈组成了闭环。一次任务可以包含多次模型请求和工具执行，Harness 负责维护状态、限制风险与验证结果。

**原文依据：** “再观察工具返回的结果并继续思考下一步。”见[第一章·ReAct 循环](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L157)。

### 15.2 LLM、上下文和工具分别承担什么职责？

**参考答案：** LLM 负责理解意图、规划和决策；上下文提供本次决策可见的规则、用户请求、历史和证据；工具把决策转化为读取信息、执行操作或协作。Harness 把三者连接起来：准备上下文、发起模型请求、调度工具，再将结果交给下一轮模型。

**原文依据：** “大脑负责思考和决策，眼睛提供思考所需的全部信息，手脚将决策转化为对现实世界的改变。”见[第一章·现代 Agent 的组成](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L19)。

### 15.3 模型如何知道有哪些工具可以使用？

**参考答案：** Harness 向模型提供工具声明，其中包含名称、用途、参数结构和约束。模型据此选择工具并生成参数。原版代码先由 `_get_tools()` 向 Kimi 获取声明，再在 `_chat()` 中加入请求的 `tools` 字段。大规模工具集还可以先提供发现入口，再按需加载完整定义。

**原文依据：** “声明 Agent 可用工具的名称、功能描述和参数格式。”见[第一章·上下文中的工具定义](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L134)。实现见[`_get_tools()`](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L150)及[`_chat()` 中的工具装配](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L318)。

### 15.4 模型生成工具调用后，究竟是谁执行工具？

**参考答案：** 模型产生的是调用请求；真实执行由模型之外的程序或服务完成。这个项目中，本地 Python 读取工具名与参数，通过 `_execute_formula()` 请求 Kimi Formula，服务端执行搜索，再由 Python 将返回值加入历史。换成读取本地文件时，执行者可能就是本机程序。

**原文依据：** “而工具本身及其执行则由 Agent 框架（或 API 内置工具）提供”。见[第一章·实验 1-2 的职责区分](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L216)。代码见[`_execute_formula()`](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L213)。

### 15.5 为什么工具参数经常是 JSON 字符串？

**参考答案：** JSON 能明确表达参数名、字符串、数字、数组和嵌套对象，便于按照工具 schema 解析与校验。在原版使用的接口中，`function.arguments` 的返回类型是字符串，字符串内部装着 JSON，因此用 `json.loads()` 得到 Python 对象。这是接口约定，不是所有工具都必须如此；原文也介绍了接收原始文本的自由格式工具。

**代码补充：** 原版解析后的对象用于可读轨迹，发送给 Formula 的仍是原始参数字符串。

**原文依据：** “传统方式中，模型调用工具时必须把所有参数打包成严格的 JSON 格式”。见[第一章·实验 1-3 的参数格式对比](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L224)。解析与转发分别见[代码第 417 行](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L417)和[第 434 行](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L434)。

### 15.6 `assistant(tool_calls)` 和 `tool` 消息为什么都要加入历史？

**参考答案：** 前者记录“模型请求了什么”，后者记录“执行后得到了什么”。两者通过调用 ID 配对，才能完整表达一次行动及其观察。一条助手消息可以请求多个工具，每个结果都要对应其调用。保留它们既满足接口协议，也让下一轮模型知道已经完成的步骤与待处理问题。

**原文依据：** “tool 消息通过 `tool_call_id` 与对应的工具调用关联”。第一章给出轨迹概念，第二章补充了协议细节：见[第一章·轨迹](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L159)和[第二章·多轮交互的三个细节](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter2.md#L221)。

### 15.7 原版为什么下一轮重新发送完整消息历史？服务端会话接口又有什么不同？

**参考答案：** 原版采用客户端显式管理历史，单独发送当前问题不足以让模型获得上一轮的工具请求和结果，因此每次 `_chat()` 都接收当前完整历史。服务端会话接口则可以保存先前状态，客户端通过会话或响应 ID 关联续接，减少重复传输。两者改变的是历史由谁保存与组装，模型仍需要相关上下文。

**原文依据：** “第二次请求包含了第一次的全部对话历史”。见[第二章·显式历史示例](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter2.md#L219)。服务端引用的本地代码例子见[`previous_response_id` 的请求装配](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/search-codegen/agent.py#L115)。后者属于代码补充，不将原文的无状态示例推广为所有接口。

### 15.8 `finish_reason == "tool_calls"`、`"stop"`、`"length"` 分别意味着什么？

**参考答案：** 在原版接口中，`tool_calls` 表示模型请求工具，程序执行并回填结果后继续；`stop` 表示正常停止生成，有有效文本时可以作为回答；`length` 表示输出预算耗尽，可能只得到部分内容甚至没有最终正文，原版会添加截断提示。其他结束原因和空内容也应单独判断。

原版并未专写 `stop` 分支，而是在 `else` 中只要有内容就返回，这是该教学实现的简化处理。

**原文依据：** “常见的退出条件包括：调用最终输出工具、模型返回没有任何工具调用的响应，或者遇到错误、达到最大轮次数。”见[第一章·自主 Agent](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L375)。具体枚举处理以[代码第 389 行](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L389)、[第 459 行](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L459)和[第 475 行](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L475)为依据。

### 15.9 如何使用超时、最大轮数、错误记录和人工确认控制风险？

**参考答案：** 超时约束单次等待；最大轮数或预算限制持续循环；结构化错误记录帮助判断应重试、改参数还是停止；人工确认用于重要授权、不可逆动作或连续失败。原版 `max_iterations` 限制主模型调用轮数，一轮可包含多个工具调用。对于有副作用的工具，超时后还要查询执行状态，不能直接假设未执行并重发。

**原文依据：** “为 Agent 的重试次数或操作次数设置上限。”见[第一章·人工干预](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L442)，以及[执行侧风险评级](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L423)。迭代上限见[原版循环条件](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L376)；超时副作用的补充依据见[第四章·幂等性与取消语义](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter4.md#L298)。

### 15.10 换成其他模型或搜索服务时，哪些部分需要改，哪些 ReAct 主循环可以保留？

**参考答案：** 通常要适配模型地址、鉴权、模型名、采样配置、消息格式、工具声明、返回解析，以及搜索服务的执行接口。若工具改为服务端托管，部分调度循环也会移动到服务端。可保留的是概念上的“准备上下文 → 模型决策 → 工具执行 → 回填观察 → 再次决策”，以及超时、预算、日志和验证原则。

**原文依据：** “客户端的工具调用循环（检测 `tool_calls` → 执行 → 回传结果）逻辑保持不变”。这段原文讨论参数格式变化，提供了区分“循环机制”与“接口表示”的思路；跨提供商迁移还需额外检查执行位置。见[第一章·实验 1-3](https://github.com/bojieli/ai-agent-book/blob/main/book/chapter1.md#L224)。本地适配点见[`_chat()`](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L298)和[`_execute_formula()`](https://github.com/bojieli/ai-agent-book/blob/main/chapter1/web-search-agent/agent-原版.py#L213)。

如果能够脱离代码回答这些问题，就已经掌握了第一章的核心概念和 Web Search Agent 的主要处理范式。

还应能区分主模型调用轮数、工具调用次数与用户对话轮次，并说明上下文适应、外部记忆更新和模型参数训练之间的区别。

## 16. 一句话复习

> Agent 不是“会调用 API 的聊天模型”，而是一个由 LLM 决策、上下文承载状态、工具连接外部世界、Harness 控制执行与风险的持续循环系统。

## 17. 复核记录（2026-09-14）

- 校正 Harness 五个功能，补充观察/行动/策略和三类学习路径。
- 校正 API 历史管理的适用范围、原版入口实际导入目标、缓存生命周期和停止分支。
- 区分迭代轮数与工具次数，说明 Formula 加密输出与可见轨迹的边界。
- 修正 JSON 转换范围、类型标注和命令行拆分说明。
- 实验结论仅引用仓库既有记录，本次没有重跑或变更业务代码。
