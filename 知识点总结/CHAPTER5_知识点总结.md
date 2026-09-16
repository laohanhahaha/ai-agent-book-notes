# Chapter 5 知识点总结：Coding Agent 与通用 Agent

> 依据：第五章正文、配套项目、实验台账与相关代码。整理日期：2026-09-16。本文总结当前本地书稿；实验数字引用仓库已有记录，本次没有重新运行实验。各节提供来源，章末包含自检题答案和原书思考题的参考分析。公开版引用固定到原书提交 `d39b1d74702c7fec9e6767acf6f98aff901725ae`（与本地第五章正文和台账一致）；各引用行已逐处核对。

## 1. 本章主线：用 Coding 能力把前四章组合起来

前四章依次回答了“怎样决策”“提供什么上下文”“怎样记住和检索”“怎样执行与协作”。第五章把这些能力组合成一个能产出文件、操作系统、解决开放任务的运行框架。

本章强调的架构是：**Coding Agent + 文件系统 + 验证反馈**。模型可以临时编写脚本、调用已有程序、读写中间产物，再依据运行结果修改方案。

```text
用户目标 → 理解任务与环境 → 生成代码或工具调用
                              ↓
                         受控执行与产物
                              ↓
                   测试 / 数据核对 / 渲染检查
                              ↓
                满足验收则交付，否则带着反馈继续
```

这仍是第一章的 ReAct 循环；变化在于行动可以是生成并执行程序，观察可以是测试结果、截图、数据库结果与真实文件。

原文把 Coding 作为核心架构的判断限定在开放任务型 Agent。固定业务流程的客服等系统，仍可能以领域工具与对话策略为中心。

来源：[本章架构](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L5)、[适用边界](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L80)。

## 2. 七个核心工具：少量基础操作形成丰富能力

| 工具 | 解决的问题 | 常见用途 |
|---|---|---|
| Code Interpreter | 运行计算代码 | 数值计算、数据处理、绘图 |
| Bash / Shell | 调用命令行程序 | 测试、构建、格式转换 |
| Read | 读取文件 | 查看代码、配置、日志 |
| Write | 创建或整体写入文件 | 生成报告、新模块 |
| Edit | 局部修改文件 | 修复函数、调整配置 |
| Glob | 根据路径模式找文件 | 定位测试文件、扫描目录 |
| Grep | 根据内容模式找位置 | 查函数名、报错、TODO |

例如“统计各模块 TODO 数量并画图”，可以组合内容搜索、代码执行和文件写入。工具列表不必为每种图表、每类统计都增加一个专用接口。

七工具是本章给出的基础设计，不是所有产品必须恰好提供七个工具。它与第四章的五类工具分类角度不同：这里主要覆盖感知和执行，协作、事件、沟通还需要其他能力。

来源：[核心工具清单](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L21)、[分类区别](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L33)。

## 3. 文件系统为什么成为工作中枢

文件系统可以同时承载源代码、任务计划、原始数据、完整日志、长期记忆和最终产物。文件路径让不同步骤、不同 Agent 能引用同一个对象，也让任务在上下文压缩或下一次会话后继续推进。

要区分三种状态：

| 状态 | 保存在哪里 | 下一次模型调用怎样使用 |
|---|---|---|
| 当前上下文 | 本次请求的消息及相关输入 | 直接参与本轮生成 |
| 持久文件 | 工作区或存储系统 | 通过读取、搜索或摘要加载 |
| 模型参数 | 模型权重 | 不会因为写入文件而自动改变 |

把经验写入 `MEMORY.md`，只是增加了一份外部记录。记录能否成为可靠知识，还要核查来源、适用条件和后续表现。Markdown 便于人工审阅与版本管理，但不自动解决冲突、过期、权限或大规模检索。

来源：[文件系统的作用](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L74)、[记忆与经验记录](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L76)、[记录后的验证](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L78)。

## 4. Coding Agent 的工程流程

推荐流程可以归纳为：理解项目 → 明确需求 → 必要的设计 → 实现 → 验证与修复 → 审查 → 文档与交付。

- 理解项目：阅读目录、关键实现、调用方、测试与项目约定。
- 明确需求：确认目标、影响范围、约束和完成标准。
- 必要的设计：对复杂或跨模块任务解释方案和取舍。
- 实现与验证：根据反馈修复，直到满足对应验收条件。
- 交付：说明结果、验证情况及仍未解决的问题，同步受影响的文档。

原文明确说这是一套可裁剪的推荐流程。简单修改无需每次生成完整设计文档，也不需要把每一步都变成人工审批。流程的厚度应随任务复杂度、影响范围和既有授权变化。

“什么时候停止阅读并开始修改”也受模型学到的策略影响，不能把模型之间的全部行为差异都归因于 Harness。

来源：[流程适用范围](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L88)、[模型行为策略](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L90)、[验证与交付](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L116)。

## 5. 项目文档与项目指令文件

README、架构文档、任务描述和项目指令共同提供 Agent 能消费的知识：构建与测试命令、模块职责、接口约束、代码风格、禁止修改的范围。

README 主要解释项目是什么、怎样使用；项目指令文件着重约定 Agent 在项目里怎样工作。`AGENTS.md` 等文件如何发现、加载和确定作用范围，取决于具体运行时，文件名本身不会产生权限。

“对远程新人友好”可以作为文档质量的检查方式：一个看不到口头约定的人，能否只靠仓库与任务说明开始工作？隐含在会议、聊天和个人经验里的决策，需要整理为可查证的记录。

来源：[项目文档化](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L96)、[项目指令](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L98)、[远程协作](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L100)。

## 6. Harness：验收、边界、反馈、恢复

| 工程组件 | 核心问题 | 具体机制 |
|---|---|---|
| 验收基线 | 什么算完成？ | 功能测试、验收样例、CI、审查 |
| 执行边界 | 允许怎样完成？ | 权限、目录范围、依赖规则、操作限制 |
| 反馈信号 | 哪里出了问题？ | 类型错误、测试失败、结构化诊断 |
| 回退手段 | 出错后怎样恢复？ | 分支、补丁、快照、状态备份 |

目标明确且结果可自动验证时，Agent 更容易形成高效闭环。有自动检查但目标不清楚，可能只是更快地优化了错误指标。

例如，删掉失败测试后得到“全部通过”，并没有完成修复。除了检查最终结果，还要约束修改过程，避免删除数据、绕过校验等捷径。

工程补充：测试通过只能说明检查覆盖的行为满足要求；Git 能恢复被跟踪的代码版本，不能自动撤回已发出的邮件或外部数据库操作。原文中关于可验证性与回退的强表述，应按这些实际范围理解。

来源：[四个组件](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L132)、[四象限](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L139)、[过程性错误](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L167)。

## 7. 四层故障：错误发生在哪一层

| 层次 | 典型故障 | 首要处理方向 |
|---|---|---|
| API | 限流、断流、超时、输出截断 | 判断可重试性、限次重试或调整请求 |
| 工具 | 工具不存在、参数非法、执行异常 | 返回具体错误，改变输入或策略 |
| 上下文 | 溢出、压缩失败、消息配对损坏 | 管理预算、恢复真实状态和消息结构 |
| 控制流 | 无进展循环、恢复逻辑递归失败 | 熔断、停止辅助调用、升级处理 |

检测不仅要看单个异常，还要识别重复模式：工具名与参数的指纹、连续失败次数、是否有新证据、任务状态是否推进。

重复调用不一定都是错误，例如按约定轮询后台任务；应结合时间间隔、预期变化和预算判断。长连接还需要空闲超时或活性监控，不能把“连接仍在”当作“任务仍在推进”。

来源：[故障分类](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L177)、[检测](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L184)、[活性与完整性](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L188)。

## 8. 恢复循环也必须有终点

可重试的网络错误可以使用指数退避、随机抖动和服务端等待提示；权限不足、参数错误或工具不存在，需要先改变条件。主链路和辅助功能应有不同预算，避免自动生成标题等后台操作耗尽请求配额。

恢复可以逐级升级：限次重试 → 调整请求或降级 → 向用户报告最终失败与已尝试的方法。具体上限应根据运行数据、费用与任务时限设定。

死亡螺旋是“处理错误的逻辑自身再次失败”：例如上下文溢出后，停止钩子又调用同一个模型生成提交说明，继而再次溢出。应让错误路径尽量依赖确定性逻辑，并限制递归深度和总预算。

两处需要特别注意：

- 修补缺失的工具结果时，要明确记录结果缺失或执行失败，不能虚构成功结果。
- 参数修复只能处理含义明确且可审计的格式问题；改变字段含义或猜测关键值，应返回错误让模型重新提供。

这些是结合第四章参数保真性得到的工程补充。对有副作用的调用，重试前还要核查操作状态和幂等性。

来源：[恢复分级](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L190)、[工具错误反馈](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L196)、[终止与死亡螺旋](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L200)。

## 9. 并行与流式执行：以依赖关系划分故障范围

模型输出某个工具调用的完整参数后，运行时可以先校验并启动该调用，与后续生成重叠。存在审批要求时，还必须经过相应门控。

独立读取三个文件可以并行；“读取结果 → 根据内容修改 → 运行测试”则存在依赖，不能随意同时启动。某个文件不存在，应保留另外两个独立读取的结果，只取消依赖失败结果的后续操作。

并行写同一文件、共享可变工作目录或共用会话状态可能产生冲突。判断能否并发，要看读写对象、依赖和副作用，不能仅凭工具名称。

来源：[流式启动](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L214)、[并发声明与级联中止](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L216)。

## 10. 上下文管理与执行环境管理

本章将第二章的原则落到编程任务：按范围读取大文件、附上真实行号、截断长输出并保存完整日志、在上下文末尾提供动态环境状态。

状态信息包括当前目录、分支、近期提交和已有改动。稳定项目约定放在稳定前缀；经常变化的信息按需追加，减少缓存失效。

持久终端便于保留目录、环境变量和后台服务，但也会累积隐藏状态。另一种实现是每次显式传入工作目录、解释器路径和环境变量。并行任务应隔离相互冲突的状态，并管理后台进程的生命周期。

写文件后立即提供语法或 lint 反馈能缩短纠错周期；这些检查只覆盖部分错误，仍需与功能验证结合。

来源：[范围读取与长日志](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L222)、[动态状态](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L228)、[持久终端](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L239)、[即时反馈](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L245)。

## 11. 四种代码搜索方式

| 方法 | 适合的问题 | 局限与成本 |
|---|---|---|
| Glob | 文件可能在哪里？ | 只匹配路径，不理解内容 |
| Grep / 正则 | 哪些地方出现这个名称或模式？ | 文本命中不等于语义相关 |
| 语义检索 | 哪个模块实现了这个概念？ | 索引更新、召回质量和数据处理成本 |
| 符号搜索 / LSP | 这个定义被谁调用？ | 依赖语言分析能力，动态行为可能难以覆盖 |

不知道实现名称时可先做语义定位；已经知道函数或报错时，精确搜索通常更直接；重构则需要追踪定义、调用和类型关系。

语义检索可以采用结构感知分块、关键词与向量融合、候选重排。是否建索引，要比较收益和维护成本。正文中的产品案例用来说明路线差异，不应当作永久不变的产品能力清单。

来源：[搜索工具](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L255)、[语义检索](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L259)、[符号定位](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L268)。

## 12. 五种编辑方式：怎样准确表达修改

| 方式 | 优势 | 主要风险 |
|---|---|---|
| 差异描述 + Apply Model | 主模型专注修改意图 | 二次合并可能误解位置或内容 |
| 旧字符串 → 新字符串 | 匹配过程明确、容易审计 | 原文必须准确，相同片段需要消歧 |
| 行号范围 → 新字符串 | 大段替换表达简短 | 文件修改后行号漂移 |
| 类 Vim 命令 | 移动、重排等操作灵活 | 强依赖中间状态，批量规划困难 |
| 起止字符串 → 新字符串 | 缩短大块替换的输入 | 边界匹配仍需唯一且顺序正确 |

共同要求是：基于当前文件状态定位、明确歧义、检查修改结果。工具显示“已写入”只证明操作执行了，还要检查 diff 和相关行为。

这些方案是接口设计选择；具体 Coding 产品采用的方式可能变化，不能由一段产品举例推断所有版本都使用相同编辑工具。

来源：[文件编辑工具](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L274)。

## 13. Coding Agent 的安全：数据、输入、输出与记忆

“致命三要素”形成一条攻击路径：不可信内容影响模型 → 读取私有数据 → 经外部通道传出。持久记忆会让影响跨会话延续，是放大器，不是攻击必须具备的第四个条件。

执行环境需要明确四类边界：

- 文件：哪些目录可读、可写，凭证是否进入执行环境。
- 网络：可以访问哪些目的地、采用何种身份与范围。
- 资源：CPU、内存、磁盘、时间和进程数量。
- 记忆：哪些内容可以持久化，来源和规则变更怎样审查。

命令安全不能只搜索危险关键词，还要理解参数、重定向、管道、子命令和实际目标。语义解析有自身覆盖范围，需要与隔离、权限和独立门控配合。

多方委托下，Agent 还要保持用户授权的任务边界：外部文档、交易对手和工具返回值可以提供事实，不能自行改变任务目标或扩大授权。持有执行能力也不等于已获得操作许可。

来源：[三要素与记忆](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L290)、[隔离策略](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L308)、[命令解析](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L316)、[多方委托](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L320)。

## 14. 代码作为元能力的六个方向

元能力是“能创造其他能力”的能力。模型可以现场生成程序，扩展当前任务可用的操作和表达形式。

| 方向 | 生成什么 | 交给谁处理 |
|---|---|---|
| 思考工具 | 方程、约束、计算脚本 | 求解器或解释器 |
| 业务规则 | 条件、校验器、状态约束 | 可信执行服务 |
| 多媒体生成 | 幻灯片、视频编辑代码 | 渲染器与媒体工具 |
| 系统适配器 | 解析器、接口转换逻辑 | 运行时与外部系统 |
| 生成式 UI | 表单、SQL、交互界面 | 客户端和受控后端 |
| Agent 自举 | 新 Agent 的配置与实现 | 验证、测试和发布流程 |

共同链路是：理解需求 → 形成可执行表示 → 受控执行 → 独立检查结果。

来源：[元能力定义](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L332)、[六个方向](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L334)。

## 15. 代码辅助思考：正确建模仍是关键

模型负责把自然语言问题变成变量、表达式和约束；求解器负责计算。SymPy 适合符号运算，NumPy、SciPy 等支持数值任务，约束求解器处理满足条件的组合搜索。

如果模型把“只选物理”误写成“数学人数减交集”，执行器会准确执行错误公式。还要核对变量含义、单位、定义域、边界条件，以及答案是否回到原题。

因此，代码可执行、测试通过与数学证明是不同层次。数值计算还可能涉及浮点误差和容差；形式化程序也可能漏写约束。

配套实验 5-1 的代码辅助准确率更高，但统计差异未达到常用 0.05 显著性水平；实验 5-2 的代码辅助结果反而更差。它们说明应评估完整的“理解—建模—执行—解释”链路。

来源：[模型与计算的分工](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L363)、[约束求解实验](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L380)、[实测记录](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/EXPERIMENT_LEDGER.md#L26)。

## 16. 业务规则：自然语言、checklist 与真值校验

本章的三层设计非常重要：

1. 自然语言规则帮助模型理解、解释政策和寻找可行方案。
2. 工具描述及 `expected_*` 参数提示模型调用前核对条件。
3. 执行服务从数据库与可信时钟获取事实，校验通过后才执行操作。

`expected_has_insurance=True` 只代表模型声称有保险，不能作为订单真值。身份、舱位、保险、时间与订单状态都应来自可信服务。

参数 checklist 有助于引导模型，但不保证模型真的查询或理解了规则。真正的执行约束必须在模型无法绕过的位置。

工程补充：把校验放进执行函数并不自动解决并发竞态。生产系统还需要根据场景采用事务、版本检查或锁，保证校验时的状态与写入时一致。

本地实验中的字段是 `expected_refundable`、`expected_reason`，与正文示意中的 `expected_cabin_class`、`expected_has_insurance` 不完全相同；共同原则都是模型自述用于核对与审计，服务端事实决定裁决。

来源：[规则互补](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L395)、[可信事实](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L470)、[三重保障](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L472)、[本地执行函数](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/small-model-codified-rules/airline_env.py#L129)。

## 17. 多媒体生成：用渲染结果验证代码

生成 Slidev、HTML 或视频编辑代码后，需要检查真实渲染结果。语法正确并不代表文字没有溢出、图片没有裁切，也不代表讲解与画面一致。

Proposer 负责内容规划与代码修改；Reviewer 读取最新渲染，返回页码、位置、问题类型、严重程度与改进建议。循环以验收达成或预算耗尽为终点。

双 Agent 分工的一项直接收益是上下文管理：Reviewer 每轮检查最新图片，Proposer 主要保留文本反馈，避免把所有版本的图片一直堆进同一条历史。

审核标准还应包含用户的受众、风格、信息密度和可读性要求。不同模型或视角有助于减少共同盲区，但不能保证评价一定正确。

来源：[渲染检查](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L494)、[反馈与终止](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L497)、[上下文优势](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L501)。

## 18. 讲解视频与视频剪辑

论文讲解视频的流程是：幻灯片 → 逐页讲解词 → 语音 → 按实际音频时长安排页面 → 合成 → 检查同步与内容。

讲解应解释图表与论证关系，不只是朗读页面。验收既包括音画时间同步，也包括叙述是否忠实、发音和画面是否可用。

视频剪辑采用粗到细定位：先稀疏抽帧找到候选区间，再加密抽帧确定起止点。把截图分析交给子 Agent，主 Agent 保留时间区间和必要证据；随后生成编辑脚本，预览审核后再输出完整成片。

抽帧可能漏掉短暂事件，因此采样间隔和边界检查要与目标动作持续时间匹配。

来源：[讲解视频](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L516)、[两步定位](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L537)、[剪辑验收](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L545)。

## 19. 系统适配器：把新格式转成稳定接口

模型可以根据接口文档和样本生成解析器、数据映射与客户端代码。已有系统没有 API 时，也可先通过界面完成操作，再将经过验证的步骤固化为可复用流程。

自适应日志解析的闭环是：解析失败 → 保存样本与错误 → 生成候选解析器 → 新旧样本验证 → 加载 → 继续处理。

要区分格式演化与上游故障。字段改名可以适配，但时间倒退、关键字段消失、数量异常等可能需要报警。不能通过补默认值或吞掉异常让坏数据“看起来正常”。

本地 `LogParserEngine` 管理解析器注册表，并能从文件动态导入 `parse()`。动态导入会执行 Python 模块，因此测试与隔离应覆盖加载阶段，而不只是调用函数后的返回格式。

来源：[系统适配](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L550)、[自适应闭环](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L562)、[解析器引擎](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/adaptive-log-parser/engine.py#L57)。

## 20. 日志诊断：让失败轨迹变成回归样例

诊断 Agent 结合轨迹、架构文档和 PRD 判断行为偏差，输出问题位置、证据、影响与修复建议，再生成可重放的回归测试。

关键验证是“修复前失败，修复后通过”，并核对测试覆盖的确是原始问题。测试应保留轨迹 ID、关键轮次、输入和预期行为，避免只验证新代码能运行。

创建 Issue 是后续任务分派的一步。问题报告有价值的基础是可复现证据，不能仅凭模型解释就认定根因已经确定。

来源：[日志诊断](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L575)、[回归与工作项](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L577)、[既有实验记录](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/EXPERIMENT_LEDGER.md#L54)。

## 21. 生成式 UI：把对话变成合适的交互形式

生成式 UI 可以帮助收集多个字段、展示级联条件、筛选数据和解释系统过程。例如，订票表单一次收集出发地、日期、单程或往返；选择往返才显示返程日期。

两种实现方式有不同取舍：

| 方式 | 模型提供什么 | 客户端承担什么 |
|---|---|---|
| 直接生成代码 | HTML、CSS、JavaScript 等 | 隔离执行、限制能力、验证结果 |
| 声明式组件描述 | 组件类型、数据、布局和动作 | 校验描述并使用受信任组件渲染 |

按正文定义，A2UI 类协议描述“显示什么界面”；AG-UI 侧重消息、工具状态和状态更新的事件传输。两者可以配合使用。

受信任组件减少任意脚本的攻击面，但按钮提交后的业务操作仍需服务端鉴权。界面校验用于改善体验，服务端校验决定请求是否合法。

来源：[生成式 UI](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L594)、[协议区别](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L598)、[动态表单](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L614)。

## 22. Artifact 模式：让数据直接到达呈现端

Artifact 是独立的产物，例如 SQL、图表程序、HTML 或文件。模型生成处理逻辑，执行系统使用真实数据完成处理和呈现。

```text
自然语言问题 → 模型生成 SQL → 受控后端校验与查询 → 数据库结果 → 表格/图表
                         模型不必接收并复述每一行结果
```

这种分工节约上下文，减少抄写和转述错误，但 SQL 的筛选、连接、日期和统计口径仍可能写错。需要让用户或独立计算程序核对结果含义。

执行层需要受限账号、允许的语句与对象范围、参数绑定、时间与行数限制。仅检查字符串以 `SELECT` 开头不构成完整 SQL 安全边界；前端触发查询也不意味着应把数据库凭证交给浏览器。

本地 `SQLAgent.generate_sql()` 生成字符串，`demo.py` 执行查询并与独立 Python 参考值比较。该演示的 `_clean_sql()` 主要去掉代码围栏，不能等同于正文要求的完整执行防护。

来源：[Artifact 数据路径](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L632)、[执行约束](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L634)、[生成函数](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/erp-agent/agent.py#L88)、[查询与比对](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/erp-agent/demo.py#L137)。

## 23. 动态软件：把最终权限裁决放在稳定层

模型不断修改界面和业务代码时，权限检查如果也只存在于这些生成代码中，就可能被遗漏或绕过。

本章提出让稳定的数据访问层拥有最终裁决权：动态应用负责展示与流程，可信运行时绑定用户、租户、角色等访问上下文，数据层检查每次读写、状态转换和引用关系。

条件是所有数据路径都必须经过这层，生成代码拿不到绕过它的高权限连接，也不能伪造身份。仅仅定义一个 `AccessContext` 类，不能自动保证调用者身份可信。

PEDO 是 PostgreSQL 之上的对象存储中间层。其 `update()` 体现了权限检查 → 候选状态校验 → 写入 → reactions 的顺序。工程上还需关注并发一致性、reaction 深度、失败补偿，以及原型中管理接口与普通业务接口的隔离。

来源：[动态代码的权限风险](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L671)、[可信访问上下文](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L675)、[所有访问路径](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L677)、[PEDO 更新流水线](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/permission-embedded-data-objects/pedo/core/store.py#L235)。

## 24. Agent 自举：复制可靠结构，再做有边界的修改

自举在本章中指用 Coding Agent 创建、修改或修复 Agent 实现。确定性的健康检查可以处理常见配置故障，模型负责分析规则未覆盖的长尾问题。

创建新 Agent 时，经过验证的参考实现提供消息结构、工具循环、状态管理和错误处理。模型主要修改角色、领域工具和业务逻辑，再用共同的验收标准验证。

验证要覆盖结构、语法、测试、真实工具协议、多轮状态和目标任务。只成功生成文件，或只通过模型自己编写的测试，不足以证明新 Agent 可用。

模板也可能携带过时依赖和原有缺陷，需要持续维护。一次创建或修复完成，是候选能力产物；要形成持续进化，还要有跨任务证据、版本比较、发布和回滚机制。

来源：[确定性检查与模型分析](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L705)、[基于范例生成](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L716)、[验收标准](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L728)、[本地验证入口](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/agent-creator/validator.py#L331)。

## 25. 配套代码的阅读路线

建议沿着“模型输出 → 谁执行 → 返回什么 → 怎样验证”的顺序阅读。

| 关注点 | 代码入口 | 需要理解的分工 |
|---|---|---|
| 熟悉的 ReAct 循环 | [reference_agent/agent.py](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/agent-creator/reference_agent/agent.py#L27) | 生成调用、写入 assistant 消息、执行、回填 tool 结果 |
| 业务政策 | [airline_env.py](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/small-model-codified-rules/airline_env.py#L129) | 模型预期值与服务端事实分别如何使用 |
| SQL 产物 | [erp-agent/agent.py](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/erp-agent/agent.py#L88) | 模型生成 SQL，系统执行和呈现 |
| 解析器扩展 | [engine.py](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/adaptive-log-parser/engine.py#L57) | 注册表、失败信号、持久化与热加载 |
| 数据层约束 | [store.py](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/permission-embedded-data-objects/pedo/core/store.py#L235) | 每次修改如何经过权限与状态检查 |
| 新 Agent 验收 | [validator.py](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/agent-creator/validator.py#L331) | 静态检查、测试与真实任务逐级门禁 |

ERP 的 `demo.py` 使用 SQLite；正式 PostgreSQL 对照另见 `campaign_postgres.py`。阅读项目说明、演示路径和正式记录时，要确认它们说的是同一条运行路径。

来源：[ERP 演示建库](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/erp-agent/demo.py#L169)、[正式台账](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/EXPERIMENT_LEDGER.md#L61)。

## 26. 十三个实验：目标与已记录结果

下表依据当前本地台账。`official_complete` 是台账维护者的证据完成标记，不代表所有正文预期都已得到支持，也不是本次重新复现的结论。

| 实验 | 项目 / 主题 | 已记录结果或范围 |
|---|---|---|
| 5-1 | code-for-math：数学思考 | 30 题；代码 53.3%，纯 CoT 36.7%，配对检验 p=0.125，未显著 |
| 5-2 | code-for-logic：逻辑约束 | 84 题；代码 39.3%，纯思考 75.0%；代码臂更差，未达预期 90% |
| 5-3 | small-model-codified-rules：政策执行 | 60×2；规则臂 91.7%，控制组 95.0%，p=0.6875，未显著提升 |
| 5-4 | paper-to-ppt：论文到幻灯片 | 两组各 20 页，独立审核均 95 分；分工方案峰值上下文 24,186，单 Agent 92,601 |
| 5-5 | paper-to-video：论文讲解 | 12 页，成片 513.010 秒；最大页漂移 0.024 秒 |
| 5-6 | video-edit：视频剪辑 | 完成粗细定位、脚本执行、审核纠错，边界误差满足三秒要求 |
| 5-7 | adaptive-log-parser：新格式适配 | 两种新日志格式的解析器生成、测试、热加载和浏览器呈现 |
| 5-8 | log-diagnosis：轨迹诊断 | 回归测试修复前失败、修复后通过，并记录真实 Issue 创建 |
| 5-9 | dynamic-form：级联表单 | 浏览器验证一次提交与条件显示的返程日期 |
| 5-10 | erp-agent：SQL Artifact | PostgreSQL 执行十题查询，独立校验并呈现结果 |
| 5-11 | conversational-ui：界面定制 | 三轮实际修改、浏览器观察 HMR、生产构建通过 |
| 5-12 | permission-embedded-data-objects：数据权限 | 实现、确定性 demo 与测试可用；台账为 available，official_complete=false |
| 5-13 | agent-creator：模板与从零创建 | 两组确定性质量均 39/39；模板创建更高效，没有观察到质量与效率同时严格占优 |

5-13 的历史结果目录仍带 `exp5-12`，是实验重编号前留下的路径；不要与当前 PEDO 的 5-12 混为一谈。

这些记录能说明特定模型、任务和设置下发生了什么。要推断通用收益，还需要比较样本、任务难度、模型能力和成本。规则校验能否拦截违规、整体任务成功率是否提升，也应分别测量。

来源：[实验索引与状态](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/EXPERIMENT_LEDGER.md#L8)、[详细观察](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/EXPERIMENT_LEDGER.md#L26)、[编号说明](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/EXPERIMENT_LEDGER.md#L77)。

## 27. 本章最容易混淆的概念

| 容易产生的理解 | 更准确的理解 |
|---|---|
| 代码运行了，任务就正确 | 还要验证需求、建模和实际结果 |
| 所有 Agent 都要围绕 Coding 组织 | 开放任务尤其适合；固定领域流程可以有其他中心 |
| 写入记忆等于训练模型 | 写文件更新外部状态，不自动改变权重 |
| checklist 参数是安全保证 | 参数是模型自述，最终校验依赖可信事实 |
| 工具失败后多试几次即可 | 先分类，改变必要条件，限制恢复预算 |
| 并行调用中一个失败就全部中止 | 按依赖传播失败，保留独立结果 |
| Reviewer 换个模型就足够 | 还要共享验收标准并检查实际产物 |
| Artifact 不让模型读数据就安全 | 查询与前端执行仍需要鉴权、校验和隔离 |
| 有对象存储中间层就不能越权 | 前提是身份可信、所有路径受控、凭证不能绕过 |
| 实验 passed 表示技术显著更优 | 完成、正确性、效果差异和统计证据是不同维度 |

## 28. 后续可提炼的 Skill 候选

| 候选 | 触发场景 | 核心流程与验收 |
|---|---|---|
| 仓库任务实现 | 修复或扩展已有项目 | 定位相关实现 → 修改 → 验证 → 文档同步 |
| 失败轨迹诊断 | 重复错误或任务停滞 | 分类 → 证据定位 → 有界恢复 → 回归样例 |
| 业务规则编码 | 多条件政策与重要操作 | 澄清规则 → 真值来源 → 边界样例 → 执行内校验 |
| 多媒体审核循环 | 幻灯片、视频、报告 | 生成 → 渲染 → 结构化反馈 → 复验 |
| 数据格式适配 | 新日志或接口格式 | 契约判断 → 解析器 → 新旧样本测试 → 发布 |
| SQL Artifact | 自然语言数据分析 | 明确口径 → 生成查询 → 受控执行 → 结果核对 |
| Agent 模板改造 | 创建新的领域 Agent | 选模板 → 替换领域部分 → 协议与任务验收 |

这些是以后提炼的候选流程。Skill 中应写清输入、操作顺序、失败处理与完成标准；执行权限、数据库鉴权和隔离机制仍由运行环境实现。

## 29. 学完本章后应能回答的问题

以下答案结合原文归纳。引号内为原文短摘录；代码与实验结论各自注明来源。

### 29.1 为什么代码生成被称为元能力？

**参考答案：** 因为它能在任务执行中生成新的工具、规则和表达方式。模型可以写一个解析器处理未支持的数据，写一个校验器约束业务动作，或生成界面收集信息。新增能力仍须经过执行和验证；代码存在不等于能力已经可用。

**原文依据：** 「一种“能创造其他能力”的能力」。见[元能力定义](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L332)。

### 29.2 Coding Agent 与第一章的 Web Search Agent 有什么共同点？

**参考答案：** 都通过模型决策、工具执行、结果回填和再次决策形成循环。Web Search Agent 主要观察搜索结果；Coding Agent 还观察文件、运行日志、测试和渲染结果。任务更复杂，但工具名称与参数仍由模型生成，真实操作由外部执行器完成。

**原文依据：** “这五类操作覆盖了几乎所有 Coding Agent 的核心动作。”见[核心动作](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L19)。代码补充见[参考 Agent 主体](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/agent-creator/reference_agent/agent.py#L27)，其中仍使用 assistant 工具调用与 tool 结果配对。

### 29.3 为什么文件系统和项目文档重要？

**参考答案：** 文件保存超出当前上下文的代码、日志、知识与产物；文档解释结构、约定和决策，让下一轮、其他 Agent 或人类能够继续工作。模型只有读取到相关材料才能使用它，写入文件不会自动变成永久可见的上下文或模型参数。

**原文依据：** “文件系统是信息流转的枢纽”。见[文件系统](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L74)；项目约定见[指令文件](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L98)。

### 29.4 为什么推荐流程可以裁剪，却不能随意省略验收？

**参考答案：** 简单修改通常无需完整设计与多轮确认，但仍应检查对应结果。例如改动一个公式，可以验证边界样例；改动跨模块行为，则需要更完整的测试。流程步骤按风险调整，完成标准依然应来自任务目标和可观察证据。

**原文依据：** 「会按需裁剪这套流程」。见[推荐流程的适用范围](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L88)；验证要求见[测试—修复循环](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L116)。

### 29.5 为什么结果验证和过程约束都需要？

**参考答案：** 结果检查回答“是否达到目标”，过程约束回答“是否用了允许的手段”。删除失败测试、清空数据库等动作可能让某些指标通过，却破坏任务价值。需要对重要动作设置边界，并用测试、数据校验和审查评价实际结果。

**原文依据：** “即使结果正确，用错误的方法达成也不行。”见[过程性错误](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L167)。

### 29.6 API、工具、上下文和控制流故障应怎样区别处理？

**参考答案：** API 的暂时故障可以限次退避；工具输入错误应反馈具体约束并修改参数；上下文问题要处理预算与消息结构；控制流问题要识别无进展循环并熔断。四层可以相互影响，因此还需设置全局预算，防止每个局部恢复都“继续再试”。

**原文依据：** 「第一个判断不是“要不要重试”，而是“值不值得重试”。」见[检测与分类](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L184)。

### 29.7 无进展循环和死亡螺旋有什么区别？

**参考答案：** 无进展循环是在主任务中反复尝试而没有新结果；死亡螺旋则是恢复或清理逻辑又触发失败，形成连锁。前者可用调用指纹、状态变化和失败计数检测；后者需要切断错误路径中的额外模型调用，并限制递归与恢复次数。

**原文依据：** “错误路径上触发的逻辑自身又调用 LLM，再次出错，连锁触发。”见[死亡螺旋](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L202)；调用指纹见[模式检测](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L186)。

### 29.8 为什么流式工具调用不能拿到几个参数字符就执行？

**参考答案：** 部分字符串可能不是完整 JSON，也可能还缺目标路径和操作选项。必须等本次调用参数完整，再做 schema、权限与依赖校验。通过后可以提前启动，与后续输出重叠；依赖它的操作仍需等待真实结果。

**原文依据：** “第一个工具调用的参数一经生成完整、通过校验，即可立即开始执行”。见[流式执行](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L214)。

### 29.9 Glob、Grep、语义搜索和符号搜索怎样选择？

**参考答案：** 已知路径模式用 Glob；已知名称或报错文本用 Grep；只知道功能概念可用语义搜索；要追踪定义、引用和重构影响则用符号搜索。搜索后仍要读取相关上下文，确认匹配结果属于目标逻辑。

**原文依据：** “这四种搜索方式构成互补的工具箱”。见[搜索策略](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L270)。

### 29.10 字符串编辑和行号编辑各有什么风险？

**参考答案：** 字符串替换依赖精确且唯一的原文，空格、引号或重复片段都可能影响定位。行号替换表达简短，但前一次编辑会使后续行号改变。批量编辑应基于明确版本或初始坐标约定，完成后检查实际差异。

**原文依据：** “每次编辑后后续行号都会变化”。见[行号定位](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L280)；文本定位见[字符串替换](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L278)。

### 29.11 沙盒与持久记忆分别涉及什么安全边界？

**参考答案：** 沙盒控制代码能访问的文件、网络和资源；持久记忆控制哪些信息能够跨会话影响后续决策。只有执行隔离，仍可能保留恶意指令；只有记忆审查，也不能防止本轮越权执行。需让权限、来源审查和记忆版本管理共同作用。

**原文依据：** “它不是并列的第四个必要条件，而是攻击的放大器”。见[持久记忆风险](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L296)；执行边界见[沙盒工程选型](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L308)。

### 29.12 为什么精确求解器不能保证整个回答正确？

**参考答案：** 求解器处理的是模型构造的形式化问题。变量、约束或目标写错，得到的就是另一个问题的正确解；数值方法还受精度与收敛条件影响。应分别验证问题理解、建模、计算和解释，并用真实对照评价收益。

**原文依据：** “让 LLM 负责理解问题并写出代码，让代码解释器负责精确计算”。见[分工](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L363)。边界判断结合[5-1、5-2 实测结果](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/EXPERIMENT_LEDGER.md#L26)。

### 29.13 为什么 `expected_*` 不能决定业务操作是否获准？

**参考答案：** 它是模型的自述，可能源于误读、幻觉或提示注入。服务端可以将其与事实比较并记录差异，但最终政策判断必须查询可信数据和时钟。校验与执行需要处于同一受控路径，并处理两者之间状态变化的问题。

**原文依据：** “没有任何一条政策事实来自模型自报的参数。”见[服务端真值校验](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L470)。

### 29.14 Proposer–Reviewer 为什么要检查渲染结果？

**参考答案：** 渲染提供代码文本里看不到的实际信息，例如溢出、字号和画面裁切。Reviewer 检查最新截图并给出可定位反馈，Proposer 修改代码再验证。分工还能减少主上下文累积大量历史图片的成本；用户偏好和内容真实性也应进入验收标准。

**原文依据：** “Agent 编写完代码后并不知道实际渲染效果”。见[多媒体审核](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L494)；分工收益见[上下文管理](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L501)。

### 29.15 日志自动适配和日志自动诊断分别解决什么问题？

**参考答案：** 适配解决“这种数据怎样读取和展示”，诊断解决“行为是否符合需求，为什么失败”。前者产出解析器，后者产出证据化的问题报告与回归样例。解析成功仍可能暴露上游故障，不能把二者混成“消除所有报错”。

**原文依据：** “自动判断执行流程是否符合预期，定位出问题的环节和模块。”见[日志诊断](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L577)；适配流程见[新格式处理](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L562)。

### 29.16 生成式 UI 与 Artifact 模式是什么关系？

**参考答案：** 生成式 UI 关注用户如何输入和浏览；Artifact 模式关注模型产出的代码或文件怎样交给系统执行。一个动态表单可以是 UI artifact，一段 SQL 则不必本身就是界面。二者可以组合，让模型生成查询与图表逻辑，系统直接把数据呈现给用户。

**原文依据：** “Agent 可以生成两个 artifact 形成流水线：SQL 查询 + 可视化代码”。见[SQL 与呈现流水线](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L639)；UI 定义见[生成式 UI](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L594)。

### 29.17 为什么动态应用需要稳定的数据权限层？

**参考答案：** 动态生成的 handler 可能遗漏检查，因此最终鉴权不能只依靠它。可信运行时绑定身份，稳定数据层检查每次操作，生成代码只得到受限访问能力。若它还能直连高权限数据库或伪造租户，权限层就可被绕过。

**原文依据：** “所有数据访问路径都必须经过受信任的数据层”。见[数据层的必要条件](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L677)。代码补充见[对象更新检查](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/permission-embedded-data-objects/pedo/core/store.py#L235)。

### 29.18 Agent 自举成功与实验预期得到支持有什么区别？

**参考答案：** 创建出的 Agent 能完成规定任务，说明该次构建通过验收；模板是否优于从零生成，还要比较同一标准下的质量与成本。台账中的 5-13 两组质量均为 39/39，模板更高效，支持效率收益，却不支持“质量也严格更优”。持续进化还需要更广泛任务和版本证据。

**原文依据：** “对比从零生成和基于范例修改两种模式”。见[自举实验要求](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L728)。实测见[5-13 的质量与效率](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter5/EXPERIMENT_LEDGER.md#L71)。

## 30. 原书十道思考题：参考分析

本节按原书题号对应，题意作了缩写。开放题的方案是基于原文的工程推演，不代表原书给出了唯一答案。

### 30.1 怎样平衡代码执行的能力与安全？

**参考分析：** 按任务授予最小可用能力：分析文件时开放指定输入和输出目录；需要查询 API 时提供受限网络与身份；编译任务再开放必要依赖。为资源、执行时间和外部影响设定边界，并记录实际拒绝与失败原因，用任务成功率、恢复成本和越界情况迭代配置。不存在脱离任务的统一“最优权限”。

**原文依据：** “沙盒不是一个开关，而是一系列工程决策。”见[沙盒设计](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L308)；[原题 1](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L755)。

### 30.2 怎样防止 Agent 自举中的错误代际累积？

**参考分析：** 保留经过验证的基线模板，让每一代通过相同的协议、功能和权限检查，再用未参与生成的任务评估泛化。记录父版本、改动、依赖与失败轨迹，未通过的候选不替换基线。评价规则也应受控，避免新 Agent 同时重写实现和降低验收标准。

**原文依据：** “提供高质量 Agent 实现作为参考范例”。见[范例生成](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L716)，检查范围见[自举验收](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L728)；[原题 2](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L756)。

### 30.3 怎样区分格式演化与需要报告的异常？

**参考分析：** 先检查版本说明、上游契约和多条样本，再验证核心语义是否保留。字段改名且含义一致，可以增加兼容解析；关键字段缺失、金额单位变化或数据违反业务不变量，应隔离样本并报告。候选解析器必须同时通过新格式和历史样本，保留原始输入与适配原因。

**原文依据：** “代码先在虚拟浏览器中自动测试”。见[自动适配闭环](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L562)。异常判断是对该流程的工程扩展；[原题 3](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L757)。

### 30.4 怎样让用户偏好进入 Reviewer 循环？

**参考分析：** 把受众、字体大小、信息密度、风格参考与用户实际反馈写成共享验收标准。Reviewer 将“文字溢出”等可检查缺陷与“更喜欢简洁风格”等偏好分开报告，Proposer 据此修改。关键样页可先获得反馈，再扩展到整份产物，避免模型评分提高而用户满意度下降。

**原文依据：** “生成结构化的改进建议”。见[Reviewer 反馈](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L497)；用户偏好的纳入方式是本题的方案推演，见[原题 4](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L758)。

### 30.5 怎样清理沉淀规则，为什么一次成功不等于持续进化？

**参考分析：** 为规则保存来源、适用条件、版本与验证样例；合并同义规则，遇到条件变化时重新评估，删除过时条目并检查回归。低频细则可以移入按需读取的文档，减少系统提示词膨胀。一次成功只能证明一个场景通过，持续进化还要证明新版本在目标任务集合上改善、未引入明显退化，并可发布与回滚。

**原文依据：** “记录何时足以成为可靠知识、指令或程序，仍需结合更多轨迹与结果验证”。见[经验记录的边界](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L78)；[原题 5](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L759)。

### 30.6 怎样判断团队是否对 Agent 友好？

**参考分析：** 用真实小任务检验新人能否独立找到入口、运行环境、复现问题、理解约束并完成验证。记录阻塞来自文档过期、缺少样例、权限不清还是口头知识，再优先补齐这些入口。对本学习项目，可以从“新会话能否根据总结定位原文、代码和实验记录”开始检查。

**原文依据：** “一个远程新人只靠代码仓库和文档，能不能独立开展工作。”见[团队文档化](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L100)；[原题 6](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L760)。

### 30.7 必须同时处理私有数据、外部内容、通信和记忆时怎样防护？

**参考分析：** 将数据读取、外部内容处理和对外发送分成受限能力，由可信执行层校验每次跨边界操作。秘密尽量不进入模型上下文，发送内容与目标需要符合用户授权。记忆保存来源与状态，外部指令不能直接提升为长期规则；污染时可定位版本并撤销。监控真实执行与数据流，避免只判断模型回答是否“看起来安全”。

**原文依据：** “写入长期记忆的内容需经过与外部内容同等的信任审查”。见[跨会话防线](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L304)；执行约束见[网络出口](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L310)；[原题 7](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L761)。

### 30.8 怎样安全执行 SQL 和前端 Artifact？

**参考分析：** SQL 在受控后端执行，使用受限角色与允许的表、列和语句，绑定参数并限制时间、行数及资源。前端优先使用受信任组件描述；确需运行生成代码时，隔离来源与可用能力，限制网络和敏感数据访问。保存生成物及执行记录，以便审计和复现；不能让代码自己决定访问权限。

**原文依据：** “Artifact 模式缩短了数据路径，但不能替代权限检查与执行隔离。”见[Artifact 约束](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L634)，UI 方案见[受信任组件](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L600)；[原题 8](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L762)。

### 30.9 “代码即规则”有什么优势和局限？

**参考分析：** 它可以重复执行复杂条件、检查边界并留下可测试的裁决记录，尤其适合阻止不合法的写操作。局限是规则本身可能理解错、实现可能有 bug、依赖的事实可能过期。需要业务口径澄清、可信数据、边界测试与版本维护；自然语言规则继续负责解释政策和协助用户选择方案。

**原文依据：** “系统提示词包含自然语言规则供理解和沟通，关键决策点配备代码化校验工具”。见[两类规则的结合](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L401)；[原题 9](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L763)。

### 30.10 Artifact 与模型直接回答各有什么优劣？

**参考分析：** Artifact 适合大结果集、重复计算、交互图表和需要复现的查询，能缩短数据路径并保留处理逻辑。模型直接回答适合解释、归纳和复杂语义分析，但需要控制数据规模与核对事实。两者可以组合：系统呈现完整数据，模型读取必要摘要并解释异常；代价是增加执行、展示与校验基础设施。

**原文依据：** “数据从数据库直达用户界面”。见[Artifact 工作方式](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L632)；[原题 10](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter5.md#L764)。

## 31. 一句话复习

Coding 能力让 Agent 现场生成可执行的解决方案；可靠的 Harness 把理解、执行、验证、纠错与权限约束连接起来，使这些方案成为可以交付和复用的成果。
