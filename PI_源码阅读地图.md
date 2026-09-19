# 从六章学习笔记阅读 Pi：源码定位与学习路线

依据：本地六章笔记与 `../../pi-main` 代码，核对日期 2026-09-20。链接以本文件所在目录为基准：分章笔记位于 `./知识点总结/`，Pi 源码位于 `../../pi-main/`，原书配套代码位于 `../chapterN/`。行号使用 `#L数字` 标注，对应当前副本；编辑器是否支持跳转取决于 Markdown 阅读器，更新代码后优先按函数名重新定位。Pi 与原书目录不在本笔记仓库内，因此这两类相对链接需要本地保留相同目录布局，无法直接在 GitHub 网页中访问。本次是静态阅读与定位，没有运行模型、测试或修改 Pi 源码。

建议先按 **第一章 → 第五章的基础工具 → 第二章 → 第四章 → 第三章 → 第六章** 阅读。这样先看懂一次工具调用，再学习上下文、扩展、记忆与评估。表中的“§”指你的笔记小节编号。

## 先认识三个核心层

| 层 | 入口 | 解决什么问题 |
|---|---|---|
| 应用层 | [coding-agent / sdk.ts](../../pi-main/packages/coding-agent/src/core/sdk.ts#L173) | 组装模型、资源、工具、会话与扩展，得到可使用的 Coding Agent。 |
| 循环层 | [agent / agent-loop.ts](../../pi-main/packages/agent/src/agent-loop.ts#L156) | 调用模型、执行工具、回填结果、处理追加消息与停止条件。 |
| 模型接口层 | [ai / types.ts](../../pi-main/packages/ai/src/types.ts#L524) | 定义模型输入和输出，并通过各 API 实现与提供商通信。 |

第一遍暂缓阅读终端绘制、OAuth、模型目录生成及 `chord`。`packages/agent/src/harness` 也有更丰富的运行时实现，但下文先沿当前 Coding Agent SDK 实际使用的 `Agent` / `agent-loop.ts` 调用链阅读，避免把两条路径混在一起。

## 第一章：把 Python 示例迁移到真实 Agent 循环

对应笔记：[CHAPTER1](./知识点总结/CHAPTER1_知识点总结.md)。

| 笔记部分 | Pi 对应入口 | 阅读时要回答的问题 |
|---|---|---|
| §1 核心公式；§4 初始化 | [createAgentSession](../../pi-main/packages/coding-agent/src/core/sdk.ts#L173)；[Agent 构造与状态](../../pi-main/packages/agent/src/agent.ts#L173) | 模型、systemPrompt、messages、tools 分别放在哪里？SDK 为什么先创建 Agent，再交给 AgentSession 组装应用功能？ |
| §3 三个入口层次 | [最小 SDK 示例](../../pi-main/packages/coding-agent/examples/sdk/01-minimal.ts#L1)；[AgentSession.prompt](../../pi-main/packages/coding-agent/src/core/agent-session.ts#L1175)；[Agent.prompt](../../pi-main/packages/agent/src/agent.ts#L350) | 用户文本如何从应用入口传到循环？先从短示例进入，不必一开始通读整个 CLI。 |
| §2 ReAct；§5 工具链 | [runLoop](../../pi-main/packages/agent/src/agent-loop.ts#L156) → [streamAssistantResponse](../../pi-main/packages/agent/src/agent-loop.ts#L279) → [executeToolCalls](../../pi-main/packages/agent/src/agent-loop.ts#L409) | 哪一步由模型决定，哪一步由程序执行？为什么工具执行后还需要再调用模型？ |
| §6 messages 演变；§9 JSON 与 SDK 对象 | [ToolCall](../../pi-main/packages/ai/src/types.ts#L373)；[ToolResultMessage](../../pi-main/packages/ai/src/types.ts#L452)；[createToolResultMessage](../../pi-main/packages/agent/src/agent-loop.ts#L784) | Pi 内部用 `toolCall` 内容块和 `toolResult` 消息，怎样通过 `id` / `toolCallId` 配对？不要照搬 Python 示例里的字段名称。 |
| §7 循环结束 | [runLoop](../../pi-main/packages/agent/src/agent-loop.ts#L156)；[shouldTerminateToolBatch](../../pi-main/packages/agent/src/agent-loop.ts#L589) | 没有工具调用后，为什么 follow-up 队列仍能触发下一轮？`error`、`aborted`、`shouldStopAfterTurn` 和工具结果的 `terminate` 怎样影响停止？ |
| §8 三类记录；§10 可观测性 | [AgentEvent](../../pi-main/packages/agent/src/types.ts#L431)；[SessionManager.appendMessage](../../pi-main/packages/coding-agent/src/core/session-manager.ts#L1071) | 界面收到的流式事件、当前 messages、磁盘会话记录有何区别？事件并不全部变成模型消息。 |
| §10 异常；§13 Harness | [prepareToolCall](../../pi-main/packages/agent/src/agent-loop.ts#L607)；[executePreparedToolCall](../../pi-main/packages/agent/src/agent-loop.ts#L677)；[截断输出处理](../../pi-main/packages/agent/src/agent-loop.ts#L379) | 工具不存在、参数错误、执行抛错如何变成可反馈的信息？为什么输出被 token 上限截断时，即便参数看似能解析也不能执行？ |

用一个例子串起上述位置：用户说“读取 README，告诉我项目用途”。模型产生 `read` 调用，运行时校验并执行，生成带相同调用 ID 的 `toolResult`，下一次模型请求带上该结果，模型再回答。

```text
01-minimal.ts：session.prompt(...)
  → AgentSession.prompt：处理输入、资源与扩展
  → AgentSession._runAgentPrompt
  → Agent.prompt → runPromptMessages → runAgentLoop
  → runLoop
      → streamAssistantResponse
          → transformContext → convertToLlm → streamFunction
      → executeToolCalls
          → prepareToolCall → tool.execute → createToolResultMessage
      → 工具结果加入当前轨迹 → 下一轮模型请求
  → 没有更多工具和待处理消息时结束
```

**第一份推荐逐行阅读的测试：** [agent-session-prompt.test.ts，第 43 行](../../pi-main/packages/coding-agent/test/suite/agent-session-prompt.test.ts#L43)。它预设一次 `echo` 工具调用和一句最终回答，直接断言 `user → assistant → toolResult → assistant`。这比先接真实模型更容易理解机制。

区别：笔记的 `max_iterations` 不是这份 Pi 低层循环中同名的固定限制。这里重点看停止钩子、取消、队列和错误路径；不要把书中所有保护机制都假设为默认已实现。§11 的 Python 语法知识可跳过，遇到 TypeScript 的类型、Promise 和异步迭代时再补。

## 第二章：上下文到底怎样组装、转换和压缩

对应笔记：[CHAPTER2](./知识点总结/CHAPTER2_知识点总结.md)。

| 笔记部分 | Pi 对应入口 | 阅读重点 |
|---|---|---|
| §2–4 消息结构、循环、静态前缀 | [Context](../../pi-main/packages/ai/src/types.ts#L524)；[streamAssistantResponse](../../pi-main/packages/agent/src/agent-loop.ts#L279) | `systemPrompt`、`messages`、`tools` 如何成为一次模型请求；`transformContext` 在 `convertToLlm` 之前执行。 |
| §10 系统提示词；§11 工具说明 | [buildSystemPrompt](../../pi-main/packages/coding-agent/src/core/system-prompt.ts#L28)；[read 工具定义](../../pi-main/packages/coding-agent/src/core/tools/read.ts#L64) | 提示词如何组合工具简介、guidelines、项目指令、Skills 和 cwd；工具参数 schema 与自然语言说明各负责什么。 |
| §7–9 缓存 | [anthropic-messages.ts / getCacheControl](../../pi-main/packages/ai/src/api/anthropic-messages.ts#L63)，继续搜索 `cache_control`；[缓存测试](../../pi-main/packages/ai/test/cache-retention.test.ts#L1) | 客户端如何表达缓存策略，以及对 system、消息、工具的处理。这里是缓存协议的使用方，KV Cache 的张量计算在模型服务端。 |
| §13 Skills 渐进披露 | [loadSkills](../../pi-main/packages/coding-agent/src/core/skills.ts#L409) → [formatSkillsForPrompt](../../pi-main/packages/coding-agent/src/core/skills.ts#L355) → [显式 Skill 命令展开](../../pi-main/packages/coding-agent/src/core/agent-session.ts#L1362) | 目录只展示名称、描述、路径；模型可通过 read/bash 读正文，用户也可用 `/skill:name` 展开正文。读取文件到程序内存，不等于把正文全部发送给模型。 |
| §14 程序维护状态并提供给模型 | [plan-mode 的 context 钩子](../../pi-main/packages/coding-agent/examples/extensions/plan-mode/index.ts#L177)；[before_agent_start 注入剩余步骤](../../pi-main/packages/coding-agent/examples/extensions/plan-mode/index.ts#L201) | 找到状态如何成为下一次模型可见的消息；对照 [status-line](../../pi-main/packages/coding-agent/examples/extensions/status-line.ts#L1)，后者主要展示界面状态，不能直接当成笔记中的上下文状态栏。 |
| §15–17 压缩触发与保留策略 | [shouldCompact](../../pi-main/packages/coding-agent/src/core/compaction/compaction.ts#L235) → [findCutPoint](../../pi-main/packages/coding-agent/src/core/compaction/compaction.ts#L403) → [prepareCompaction](../../pi-main/packages/coding-agent/src/core/compaction/compaction.ts#L750) → [compact](../../pi-main/packages/coding-agent/src/core/compaction/compaction.ts#L858) | 何时压缩、保留多少近期内容、怎样处理被切开的轮次、怎样合并旧摘要和文件操作记录。 |
| §17 压缩后的历史完整性 | [buildContextEntries](../../pi-main/packages/coding-agent/src/core/session-manager.ts#L418)；[convertToLlm](../../pi-main/packages/coding-agent/src/core/messages.ts#L148)；[压缩集成测试](../../pi-main/packages/coding-agent/test/suite/agent-session-compaction.test.ts#L1) | 摘要条目如何与保留消息组成模型上下文？磁盘历史与本次发送的历史为何不同？ |
| §17 大工具结果预算 | [read 的截断提示](../../pi-main/packages/coding-agent/src/core/tools/read.ts#L64)；[bash 的输出处理](../../pi-main/packages/coding-agent/src/core/tools/bash.ts#L222)；[truncateHead / truncateTail](../../pi-main/packages/coding-agent/src/core/tools/truncate.ts#L78) | read 告诉模型下一次从哪个 offset 继续；bash 截断时保存完整输出路径。输出预算控制与 LLM 摘要是两种机制。 |
| §18 子 Agent 隔离 | [subagent 示例说明](../../pi-main/packages/coding-agent/examples/extensions/subagent/README.md#L1)；[runSingleAgent](../../pi-main/packages/coding-agent/examples/extensions/subagent/index.ts#L272) | 子进程接收什么任务，返回什么结果，哪些中间信息留在子上下文。它是扩展示例。 |

边界：§5 注意力的 Q/K/V 和 §6 服务端 Chat Template 的完整实现不在这套客户端里；§16 六种压缩策略是笔记中的比较框架，Pi 没有逐项提供六套对应实现。§12 的安全原则可以对照第四章的执行前钩子与容器化，但不能由提示词本身推导出权限保证。

自检：找出“原始会话仍保存，但下一次模型请求只包含摘要与部分近期消息”的具体路径；解释压缩为什么会改变请求前缀，同时减少后续输入量。

## 第三章：从持久会话和文件检索理解记忆的工程边界

对应笔记：[CHAPTER3](./知识点总结/CHAPTER3_知识点总结.md)。本章与 Pi 是**部分对应**，不要把会话保存直接当成完整的用户长期记忆系统。

| 笔记部分 | Pi 对应入口 | 阅读重点与边界 |
|---|---|---|
| §2–6 记忆、上下文、存储格式 | [SessionEntry 类型](../../pi-main/packages/coding-agent/src/core/session-manager.ts#L46)；[会话格式说明](../../pi-main/packages/coding-agent/docs/session-format.md#L1) | JSONL 保存了哪些条目？`id` / `parentId` 为什么能表达分支？这是轨迹存储，没有自动完成用户事实抽取、冲突合并和遗忘。 |
| §8 可计算状态 | [todo 扩展](../../pi-main/packages/coding-agent/examples/extensions/todo.ts#L106) | 从当前分支的 toolResult.details 重建 todos，理解“程序状态 → 可显示或可返回给模型的状态”。这是结构化任务状态的类比，不是用户画像实现。 |
| §10 记忆整理；§23 历史与当前上下文 | [buildSessionContext](../../pi-main/packages/coding-agent/src/core/session-manager.ts#L461)；[分支摘要](../../pi-main/packages/coding-agent/src/core/compaction/branch-summarization.ts#L293) | 分支选择、摘要、原始历史分别起什么作用。上下文压缩不意味着所有原始记录从磁盘删除。 |
| §20 文件系统范式；§22 Agentic RAG | [find](../../pi-main/packages/coding-agent/src/core/tools/find.ts#L70) → [grep](../../pi-main/packages/coding-agent/src/core/tools/grep.ts#L70) → [read](../../pi-main/packages/coding-agent/src/core/tools/read.ts#L64)，再回到 [runLoop](../../pi-main/packages/agent/src/agent-loop.ts#L156) | 模型先发现文件，再找内容，再读证据，不足时继续查询。它体现 Agent 主动检索模式，但不是带向量索引与重排器的完整 RAG 流水线。 |
| §23 历史检索的扩展接口 | [SessionSearchService](../../pi-main/packages/agent/src/search/index.ts#L1) | 这里只定义查询、同步、通知和删除等接口；接口存在不代表默认接入了搜索后端。 |
| §25 概览与详细资料按需加载 | [项目指令加载](../../pi-main/packages/coding-agent/src/core/resource-loader.ts#L119)；[Skills 目录格式](../../pi-main/packages/coding-agent/src/core/skills.ts#L355) | 可以借鉴“稳定概览 + 按需读文件”的方式。项目指令和 Skills 不是自动生成的用户长期记忆。 |

笔记中 §7 后台记忆更新、§9 专门记忆框架、§12–19 的分块/embedding/BM25/混合检索/重排/GraphRAG、§21 索引维护、§24 Contextual Retrieval、§26–27 深层知识提炼与多模态记忆，在本次核对的 Pi 核心调用链中没有一套可直接对应的完整实现。适合回到 [原书第三章配套目录](../chapter3)，或以后做成 Pi 自定义工具的学习项目。§4 与 §17 的检索评估也需要相应任务集和真值，不能用文件工具测试替代。

自检：假设以前聊天中提到一个用户偏好，Pi 保存会话后是否就能在所有新会话中自动、可靠地利用它？分别列出还缺的抽取、检索、更新和注入步骤。

## 第四章：工具契约、扩展事件、执行边界与并发

对应笔记：[CHAPTER4](./知识点总结/CHAPTER4_知识点总结.md)。

| 笔记部分 | Pi 对应入口 | 阅读重点 |
|---|---|---|
| §2–7 工具分类、粒度、ACI、参数 | [AgentTool](../../pi-main/packages/agent/src/types.ts#L387)；[read](../../pi-main/packages/coding-agent/src/core/tools/read.ts#L64)；[edit](../../pi-main/packages/coding-agent/src/core/tools/edit.ts#L143) | 阅读 `name / description / parameters / execute`，沿一个工具看输入、执行和结果，不只看类型定义。 |
| §12–13 感知范围与多模态 | [read.ts](../../pi-main/packages/coding-agent/src/core/tools/read.ts#L64) | 文字的 offset/limit、截断续读提示，与图片 MIME 检测、处理、图片内容块的不同路径。 |
| §14–16 执行前门控；§22 人工参与 | [prepareToolCall](../../pi-main/packages/agent/src/agent-loop.ts#L607)；[AgentSession 绑定工具钩子](../../pi-main/packages/coding-agent/src/core/agent-session.ts#L483)；[permission-gate 示例](../../pi-main/packages/coding-agent/examples/extensions/permission-gate.ts#L1) | 参数验证后如何触发工具钩子，扩展如何返回 block/reason。示例的正则只演示拦截流程，不构成完整命令安全判定。 |
| §17 执行—验证—反馈 | [工具执行与错误转换](../../pi-main/packages/agent/src/agent-loop.ts#L677)；[执行后钩子](../../pi-main/packages/agent/src/agent-loop.ts#L720)；[bash](../../pi-main/packages/coding-agent/src/core/tools/bash.ts#L222) | 工具失败如何反馈给模型；测试命令的退出码与输出怎样进入下一轮。执行后钩子本身不自动证明业务目标完成。 |
| §18 长输出 | [bash.ts](../../pi-main/packages/coding-agent/src/core/tools/bash.ts#L222)；[OutputAccumulator](../../pi-main/packages/coding-agent/src/core/tools/output-accumulator.ts#L1) | 流式输出、模型可见预览、完整输出文件、UI details 是怎样联系的。 |
| §19 执行隔离 | [containerization 文档](../../pi-main/packages/coding-agent/docs/containerization.md#L1)；[sandbox 扩展示例](../../pi-main/packages/coding-agent/examples/extensions/sandbox/index.ts#L1) | 区分代码可接入的隔离方案与默认执行环境。Pi 根 README 明确说没有内置的文件/进程/网络权限限制系统。 |
| §20 超时和取消 | [Agent.abort](../../pi-main/packages/agent/src/agent.ts#L319) → [shell 执行](../../pi-main/packages/coding-agent/src/core/tools/bash.ts#L79)；[edit 的取消检查](../../pi-main/packages/coding-agent/src/core/tools/edit.ts#L143) | AbortSignal 如何往下传？为什么取消不能撤销已经写入的文件？不要把信号传递误解为事务回滚或通用幂等保证。 |
| §21 协作与子 Agent | [subagent README](../../pi-main/packages/coding-agent/examples/extensions/subagent/README.md#L1)；[runSingleAgent](../../pi-main/packages/coding-agent/examples/extensions/subagent/index.ts#L272) | 单个、并行、链式模式，任务交接、进程退出、结果和错误回传。独立上下文不代表文件系统也隔离。 |
| §23、§25–27 事件驱动与安全点 | [Agent.steer / followUp](../../pi-main/packages/agent/src/agent.ts#L283)；[runLoop](../../pi-main/packages/agent/src/agent-loop.ts#L156)；[file-trigger 示例](../../pi-main/packages/coding-agent/examples/extensions/file-trigger.ts#L1) | steer 在轮次边界注入，followUp 在原本将停止时继续；外部事件可以经扩展触发消息。file-trigger 的 `/tmp` 是示例路径，Windows 运行前要调整。 |
| §25 并发；§28 消息回填 | [executeToolCallsSequential](../../pi-main/packages/agent/src/agent-loop.ts#L431)；[executeToolCallsParallel](../../pi-main/packages/agent/src/agent-loop.ts#L487) | 并行完成的结果怎样按调用顺序组织？工具进度事件与最终 toolResult 如何区分？这里的批量工具等待不等于笔记的“先返回任务 ID，后台结果另发事件”模型。 |
| §29 检查点恢复 | [会话加载](../../pi-main/packages/coding-agent/src/core/session-manager.ts#L514)；[SDK 恢复会话上下文](../../pi-main/packages/coding-agent/src/core/sdk.ts#L173) | 保存轨迹可供恢复会话，但不会自动复活先前的操作系统进程。 |
| §8、§31 主动工具发现 | [kimi-deferred-tools 示例](../../pi-main/packages/coding-agent/examples/extensions/kimi-deferred-tools.ts#L1)；[扩展文档中的工具发现](../../pi-main/packages/coding-agent/docs/extensions.md#L2412) | `tool_search` 查找能力后调用 `setActiveTools`。Kimi 示例中的 calculate 固定返回 `42`，只用于演示发现机制，不能当真实计算器或检索算法。 |

**MCP 对应边界：** §9–11 在这份 Pi 中没有内置客户端实现。[README 的 No MCP 说明](../../pi-main/packages/coding-agent/README.md#L499)建议通过扩展或 CLI 接入。`extensions` 是 Pi 的扩展机制，不能把它等同于 MCP 协议实现；[RPC 模式](../../pi-main/packages/coding-agent/docs/rpc.md#L1)也不是 MCP。

自检：读工具执行的并行分支，画出“模型响应结束 → 参数准备/门控 → 工具执行 → 结果整理 → 下一次模型请求”的顺序。当前循环先等 assistant 响应结束，再启动该轮工具，并非模型参数刚流出就立即执行。

## 第五章：把 Coding Agent 的基础能力逐个看透

对应笔记：[CHAPTER5](./知识点总结/CHAPTER5_知识点总结.md)。这是与 Pi 最贴近的一章。

| 笔记部分 | Pi 对应入口 | 阅读重点 |
|---|---|---|
| §1–3 Coding 能力与文件系统 | [工具集合](../../pi-main/packages/coding-agent/src/core/tools/index.ts#L1)；[SDK 工具选择](../../pi-main/packages/coding-agent/src/core/sdk.ts#L173) | 本地默认基础工具是 read、bash、edit、write；可选 grep、find、ls、powershell。笔记的 Glob 对照 find；这里没有同名的独立 Code Interpreter，可通过 shell 运行解释器。 |
| §4–5 工作流程与项目约定 | [Pi 的 AGENTS.md](../../pi-main/AGENTS.md#L1)；[loadProjectContextFiles](../../pi-main/packages/coding-agent/src/core/resource-loader.ts#L119)；[buildSystemPrompt](../../pi-main/packages/coding-agent/src/core/system-prompt.ts#L28) | 规则文件怎么发现、如何进入提示词。加载指令与强制执行权限是不同事情。 |
| §6–8 故障和恢复 | [AgentSession._runAgentPrompt](../../pi-main/packages/coding-agent/src/core/agent-session.ts#L1101)；[_prepareRetry](../../pi-main/packages/coding-agent/src/core/agent-session.ts#L2917)；[_checkCompaction](../../pi-main/packages/coding-agent/src/core/agent-session.ts#L2154) | 区分工具错误、模型请求错误、上下文溢出；找到重试上限、退避和压缩恢复，而不是把所有失败都当同一类重试。 |
| §9 并行与依赖 | [并行工具执行](../../pi-main/packages/agent/src/agent-loop.ts#L487)；[withFileMutationQueue](../../pi-main/packages/coding-agent/src/core/tools/file-mutation-queue.ts#L32) | 同进程中针对同一文件的 edit/write 如何排队，不同文件如何并行。该队列不应被理解为跨进程事务锁。 |
| §10 环境与输出 | [bash.ts](../../pi-main/packages/coding-agent/src/core/tools/bash.ts#L79)；[read.ts](../../pi-main/packages/coding-agent/src/core/tools/read.ts#L64) | cwd、进程、timeout 和输出预算各在哪层处理。不要假设所有 shell 调用共享一个持久终端。 |
| §11 代码搜索 | [find](../../pi-main/packages/coding-agent/src/core/tools/find.ts#L70)；[grep](../../pi-main/packages/coding-agent/src/core/tools/grep.ts#L70) | 路径模式和文本搜索如何转换为具体工具操作。向量语义搜索和 LSP 不是这两份实现提供的能力。 |
| §12 文件编辑 | [createEditToolDefinition](../../pi-main/packages/coding-agent/src/core/tools/edit.ts#L143) → [applyEditsToNormalizedContent](../../pi-main/packages/coding-agent/src/core/tools/edit-diff.ts#L300) | `edits[]` 都针对原文件匹配；先校验唯一性和不重叠，再应用修改并返回 diff。继续看 `fuzzyFindText` 的规范化回退，理解接口要求精确匹配与实现容错之间的区别。 |
| §13 安全边界 | [permission-gate](../../pi-main/packages/coding-agent/examples/extensions/permission-gate.ts#L1)；[containerization](../../pi-main/packages/coding-agent/docs/containerization.md#L1) | 提示词、扩展门控、操作系统隔离分别能约束什么。示例不等于默认完整安全体系。 |
| §19 系统适配器 | [ReadOperations](../../pi-main/packages/coding-agent/src/core/tools/read.ts#L33)；[BashOperations](../../pi-main/packages/coding-agent/src/core/tools/bash.ts#L58)；[SSH 扩展示例](../../pi-main/packages/coding-agent/examples/extensions/ssh.ts#L1) | 如何保留工具契约，并替换真正执行读文件/命令的后端。 |
| §20 日志转回归 | [suite/regressions](../../pi-main/packages/coding-agent/test/suite/regressions)；[截断摘要回归案例](../../pi-main/packages/coding-agent/test/suite/regressions/7048-compaction-truncated-summary.test.ts#L1) | 一个真实错误如何被固定为测试条件、操作与断言。 |
| §21–22 UI 与产物呈现 | [questionnaire 示例](../../pi-main/packages/coding-agent/examples/extensions/questionnaire.ts#L1)；[edit 的 content/details](../../pi-main/packages/coding-agent/src/core/tools/edit.ts#L143) | 模型结果和界面渲染怎样分开；只能作为交互与呈现机制的对应，不能等同于完整的生成式 UI 或远程 Artifact 平台。 |

笔记 §14–18 的代码辅助思考、业务规则、多媒体和视频，§23–24 的动态软件与自举，主要是使用这些基础能力构建上层任务的方法。Pi 的工具与扩展可承载此类应用，但本次没有定位到与原书实验逐一对应的内置子系统，继续对照 [原书第五章配套目录](../chapter5)更合适。

自检：读 edit 时刻意构造三个纸上案例：oldText 不存在、出现两次、两处修改重叠。分别找到报错分支，再解释为什么不能直接“找第一处就替换”。

## 第六章：先学可复现工程测试，再做能力评估

对应笔记：[CHAPTER6](./知识点总结/CHAPTER6_知识点总结.md)。Pi 的测试能帮助理解评估基础设施，但**确定性工程测试通过，不代表真实模型的开放任务成功率高**。

| 笔记部分 | Pi 对应入口 | 阅读重点 |
|---|---|---|
| §2–7 环境、任务、控制条件 | [suite README](../../pi-main/packages/coding-agent/test/suite/README.md#L1) → [createHarness](../../pi-main/packages/coding-agent/test/suite/harness.ts#L101) → [faux 模型实现](../../pi-main/packages/ai/src/providers/faux.ts#L436) | 固定模型回复、工具和会话状态，构造可复现的测试。faux 是可编程模拟提供商，不是有真实推理能力的用户模拟器。 |
| §8、§10 结果与轨迹 | [工具调用测试](../../pi-main/packages/coding-agent/test/suite/agent-session-prompt.test.ts#L43)；[低层循环测试](../../pi-main/packages/agent/test/agent-loop.test.ts#L1) | 同时检查工具是否执行、消息角色顺序、事件和结束状态，不能只检查最后一句回答。 |
| §8、§18 多轮成本 | [getSessionStats](../../pi-main/packages/coding-agent/src/core/agent-session.ts#L3359)；[usage-totals](../../pi-main/packages/coding-agent/src/core/usage-totals.ts#L1) | 累积 input/output/cacheRead/cacheWrite，并查看工具和压缩/分支摘要费用如何归类。模拟用量不等于真实服务账单。 |
| §21 Trace / Span | [telemetry 概念与契约](../../pi-main/packages/telemetry/README.md#L1)；[InMemoryTelemetryContext](../../pi-main/packages/telemetry/src/memory.ts#L192)；[agent harness 的 telemetry schema](../../pi-main/packages/agent/src/harness/telemetry.ts#L1) | 明确父子 span、属性、事件、状态；这里只说明仓库提供的观测基础设施，不假设当前 SDK 主链已经默认导出完整 Trace。 |
| §22–23 从失败到回归 | [regressions](../../pi-main/packages/coding-agent/test/suite/regressions)；[重试测试](../../pi-main/packages/coding-agent/test/suite/agent-session-retry-events.test.ts#L1) | 找到一个失败触发条件，确认测试断言覆盖修复目标，并区分专门回归集与代表性能力集。 |
| §24 提示词可复核 | [system-prompt 测试](../../pi-main/packages/coding-agent/test/system-prompt.test.ts#L1)；[resource-loader 测试](../../pi-main/packages/coding-agent/test/resource-loader.test.ts#L1) | 不同资源和工具配置如何改变最终提示词；配置身份与实际模型输入都需要记录。 |

§9 Pass@k / Pass^k / Best@k、§11–12 Rubric / LLM-as-a-Judge、§13–17 各类能力与选型评估、§19–20 全链路比较和统计检验、§25 训练仿真环境，不是上述单元测试自然提供的完整平台。应另外定义任务数据、真值、配置、重复次数与评判器，再使用 Pi 执行任务并采集结果。可回到 [原书第六章配套目录](../chapter6)继续学习。

## 实际阅读安排：每轮带走一个可以解释清楚的机制

| 顺序 | 阅读范围 | 完成标志 |
|---|---|---|
| 1 | 最小 SDK 示例 → echo 工具测试 → runLoop | 不看书也能画出四条消息，并说清谁调用模型、谁执行工具。 |
| 2 | read → edit → bash → 工具错误处理 | 能解释 schema、执行副作用、截断、错误反馈分别位于哪层。 |
| 3 | resource-loader → system-prompt → skills → convertToLlm | 能列出模型请求内容的来源，区分自动加载元数据与按需加载正文。 |
| 4 | compaction → session-manager → 压缩测试 | 能解释压缩前后磁盘记录与模型上下文的差异。 |
| 5 | steer/followUp → 并行工具 → permission-gate → subagent | 能区分消息队列、工具并发、子进程隔离和执行前门控。 |
| 6 | suite/harness → faux → regressions → usage/telemetry | 能为一个失败设计可复现案例，并指出还缺什么才能评估真实模型能力。 |

每读一个函数，只记四件事：**输入是什么、改了什么状态、输出/事件是什么、失败后谁处理**。遇到大的 `agent-session.ts`，只追踪表中入口及直接调用，不必一次通读数千行。

最适合现在开始的三处：[最小 SDK](../../pi-main/packages/coding-agent/examples/sdk/01-minimal.ts#L1)、[echo 工具测试](../../pi-main/packages/coding-agent/test/suite/agent-session-prompt.test.ts#L43)、[runLoop](../../pi-main/packages/agent/src/agent-loop.ts#L156)。先把一次工具调用走通，再进入下一轮。
