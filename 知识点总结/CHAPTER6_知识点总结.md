# Chapter 6 知识点总结：Agent 的评估

> 依据：第六章正文、配套实验台账与相关代码。整理日期：2026-09-19。本文沿用本地书稿的章节编号；实验数字来自仓库已有记录，本次未重新运行实验。各节附来源，章末包含自检题答案和原书八道思考题的参考分析。标为“补充说明”的内容用于澄清概念或统计适用条件。公开版引用固定到原书提交 `d39b1d74702c7fec9e6767acf6f98aff901725ae`（与本地第六章正文和台账一致）；112 个不同来源行号已核对，所述源码逻辑也已复核。

## 1. 本章主线：用评估决定 Agent 应该怎样改

前五章主要解决怎样搭建 Agent：组织上下文、管理记忆、调用工具、生成代码并完成任务。第六章回答三个问题：系统到底好不好、为什么会失败、下一步改哪里。

评估体系包含三层：环境决定“在哪里测”，方法决定“怎样打分”，决策决定“这些分数用来做什么”。分数最终要支持模型选型、Harness 改进、成本优化和上线判断。

```text
任务与验收标准 → 可重置的评估环境 → Agent 执行与轨迹记录
                                          ↓
                              结果、过程、成本与安全评分
                                          ↓
                               失败分类与可检验的假设
                                          ↓
                         对照实验 → 独立复核 → 接受或放弃改动
                                          ↓
                              新失败沉淀为回归测试
```

贯穿本章的核心认识是：**评估模型与 Harness 的组合，并用可复核的证据指导改进。**

来源：[评估对象与三层结构](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L15)、[可观测性回流评估](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L591)。

## 2. 模型替换、对照实验与消融实验

| 方法 | 保持不变 | 主动改变 | 主要回答的问题 |
|---|---|---|---|
| 模型替换 | 任务、工具、提示词、预算等 Harness 条件 | 使用的模型 | 当前系统对模型能力有多敏感？ |
| 组件消融 | 模型、任务和其他组件 | 关闭某项记忆、压缩、规划等功能 | 这个组件实际贡献了什么？ |
| 方案对照 | 任务集与评判标准等共同条件 | 两个明确的候选实现 | 在当前条件下哪个更合适？ |
| 因子实验 | 共同任务及测量协议 | 多个因素的组合 | 组件之间有协同或相互抵消吗？ |

例如，关闭记忆后成功率下降，说明这套任务上的表现依赖记忆组件；只替换模型则是在研究模型与现有 Harness 的适配关系。

补充说明：强模型替换后分数不涨，是检查 Harness 瓶颈的线索，还需排除任务过于简单、样本不足、模型与任务不匹配、评判器不灵敏等原因。同时关闭多个组件，只能得到组合效应；要定位单个组件，需要独立开关或进一步实验。

来源：[模型替换与消融](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L15)、[独立关闭主要特性](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L671)、[组件协同](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L558)。

## 3. 一个完整评估环境的五个组成部分

| 组成 | 含义 | 退款 Agent 示例 |
|---|---|---|
| Dataset：数据集 | 任务、初始条件、目标与可选参考方案 | 哪个订单，用户想解决什么问题 |
| Environment State：环境状态 | 会随行动变化且可重置的数据 | 订单状态、退款记录、余额 |
| Tools：工具接口 | Agent 能执行哪些操作 | 查询订单、查询政策、发起退款 |
| Rubric：评分标准 | 成功、质量与违规怎样判定 | 是否符合政策、是否创建退款、是否如实告知 |
| Interaction Protocol：交互协议 | 运行与终止规则 | 最大轮数、超时、用户确认与结束条件 |

退款任务中，“已提交退款，预计到账时间为……”与“钱已经到账”是不同状态。Agent 的表述应与实际工具返回一致。

为了考察组合行动的能力，评估工具通常需要暴露足够细的操作粒度。工具直接替 Agent 完成“解决所有问题”，就难以判断规划能力。生产系统是否使用高层业务工具，则取决于效率、权限与可靠性要求。

来源：[五个组成部分](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L75)、[退款轨迹](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L29)、[工具粒度](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L79)。

## 4. 工具调用型环境与人机交互型环境

工具调用型评估重点关注 Agent 是否能操作工具并产生正确结果。书中用 Verifiers 的环境层级说明不同需求：

| 类型 | 本章中的用途 |
|---|---|
| SingleTurnEnv | 单轮问答后检查答案 |
| ToolEnv | 多轮搜索、综合信息并验证结果 |
| StatefulToolEnv | 操作数据库等可变状态后检查变化 |
| SandboxEnv | 在隔离环境中执行代码、检查文件与测试 |

这里 ToolEnv 的“无状态”是相对于需要专门维护的环境状态而言；多轮工具调用仍然可以积累 messages 和执行轨迹。

人机交互型评估增加了澄清、确认、沟通和用户配合。τ-bench 同时检查业务状态与沟通内容；τ²-bench 引入双控环境，用户模拟器也能改变共享状态。例如 Agent 指导用户切换飞行模式后，必须观察设备状态，才能判断故障是否修复。

两类环境都需要明确的成功条件。它们的区别主要在交互结构，而不是“一类看结果，另一类只看聊天”。

来源：[环境层级](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L91)、[状态与沟通双重验证](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L126)、[双控环境](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L128)。

## 5. 用户模拟：渐进式透露信息与事实锚定

真实用户经常先说“网络连不上了”，而不是一次性给出账号、设备、故障现象和全部操作记录。评估中的用户模拟器应允许 Agent 通过提问获得必要信息。

模拟器需要分开保存：

- 已知事实：订单、时间、设备状态、用户真正的需求。
- 行为指令：什么时候透露信息，怎样回应澄清或确认。
- 可执行动作：在双控环境中实际改变用户侧状态。
- 约束：不能编造未提供的事实，不能把评估答案直接泄露给 Agent。

固定任务、初始状态、模拟器配置与运行协议，有助于重复比较；LLM 模拟器仍有随机性，需要重复运行与轨迹核查。

模拟器本身也是被验证的组件。过于配合会高估 Agent，持续刁难又会扭曲实际用户分布。

来源：[渐进式信息透露](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L110)、[用户模拟约束](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L112)、[事实与指令分离](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L180)。

## 6. 任务设计：目标明确，路径开放

好的任务说明要明确目标对象、信息来源、时间范围、输出格式和业务约束，同时允许多条合法解决路径。

例如“修改联系人电话”可以固定验收条件为：指定联系人的电话字段等于目标值、其他联系人未被误改。Agent 可以搜索联系人后编辑，也可以通过其他合法入口完成；不必强制复现唯一点击序列。

参数化模板把任务结构与实例分开：

```text
模板：把 [CONTACT_NAME] 的电话改为 [NEW_PHONE]
实例：固定姓名、电话、初始联系人库和随机种子
验收：读取真实状态，检查目标字段与必要的保护条件
```

测试既可从干净环境开始，也可从预设中间状态开始。关键是初始状态明确、能正确重建，验证器接受任务允许的合理结果。

来源：[明确性与开放性](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L166)、[精确任务描述](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L178)、[参数化模板](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L184)、[中间状态与多解性](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L192)。

## 7. 数据集覆盖、质量控制与防泄漏

### 7.1 覆盖矩阵比单纯堆题量更有用

任务集应覆盖能力、难度、业务场景和边界条件。总体成功率下降后，分层结果能指出是基础工具使用、多步规划、信息整合还是政策判断出了问题。

质量控制要检查任务描述、初始化、工具执行和验证逻辑。某道题持续失败，原因也可能是网站失效、配置缺失或评分器有误。

来源：[分层任务](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L196)、[分布覆盖](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L222)、[四类评测问题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L230)。

### 7.2 防泄漏需要组合措施

| 方法 | 主要作用 | 使用边界 |
|---|---|---|
| 私有保留集与新任务 | 减少直接背答案的机会 | 要管理访问权限与后续训练使用 |
| 参数化生成 | 增加实例变化，削弱固定轨迹记忆 | 换人名、订单号仍可能保留相同题型 |
| 时间更新 | 引入更晚出现的任务 | 要核对模型更新与任务创建时间 |
| 金丝雀标识 | 为数据暴露提供追踪线索 | 要排除提示词、检索等其他暴露途径 |
| 检查真实状态 | 要求真实完成任务 | 验证器本身也需要防绕过与审计 |

SWE-bench Verified 的关键是人工质量筛选；书中把基于时间新鲜度的策略归于 SWE-bench-Live 等后续工作。两者不要混为一谈。

评估与训练可以复用环境生成机制，但保留评估实例必须与训练数据隔离。稳健的设计追求持续降低污染风险，而不是承诺公开题库永久无法被学习。

来源：[数据污染与各类防范措施](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L174)、[训练与评估的隔离](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L234)。

## 8. 指标体系：结果、过程、成本、安全一起看

| 指标 | 检查什么 | 常见误解 |
|---|---|---|
| 任务成功率 | 目标状态是否达成 | 把回复“完成了”当作真正完成 |
| 行动合法率 | 工具、参数格式和权限是否合法 | 格式正确就代表操作对象正确 |
| 工具调用正确率 | 查询词、路径等是否语义正确 | 成功返回 HTTP 响应就算做对 |
| 路径效率 | 步数、重复操作、回退 | 一味减少必要检查 |
| 检索覆盖 | 是否找全所需证据 | 找到一条相关片段就算信息完整 |
| 成本与延迟 | 整个任务用了多少资源 | 只比较单次请求的 token 单价 |
| 鲁棒性 | 随机性、页面变化、API 抖动下的表现 | 一次成功等于稳定可靠 |
| 安全与合规 | 越权、泄露和严重业务违规 | 让其他高分抵消严重违规 |

否决项应对应明确的高风险行为。一般质量问题可以分级扣分；严重安全问题需要独立的发布门槛。

来源：[过程指标](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L242)、[质量与安全指标](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L250)、[鲁棒性](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L267)。

## 9. Pass@k、Pass^k 和 Best@k

| 指标 | 含义 | 更接近的问题 |
|---|---|---|
| Pass@k | k 次尝试中至少成功一次 | 给多次机会，能否做出来？ |
| Pass^k | k 次尝试全部成功 | 重复执行是否可靠？ |
| Best@k | k 次尝试中的最高质量分数 | 多次生成中最好能达到什么水平？ |

若同一任务每次成功概率为 p，且各次尝试相互独立、条件不变：

```text
Pass@k = 1 - (1 - p)^k
Pass^k = p^k

p = 0.6，k = 5：
Pass@5 = 98.976%
Pass^5 = 7.776%
```

补充说明：不同任务难度不同，不能直接把全体任务的平均成功率代入上述公式，就当成总体重复运行表现。实际任务应重复采样，再按约定汇总。失败相关、环境残留或策略调整也会破坏独立同分布假设。

Best@k 还隐含“能识别哪个答案最好”。离线评委选出的最好结果，不等于线上系统已经具备可靠的自动选择器。

来源：[三个指标的定义](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L252)、[数值示例与适用场景](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L256)。

## 10. 轨迹与结果：既验证做到什么，也检查怎样做到

轨迹记录 Agent 的输入、回复、工具调用和观察；最终结果记录数据库、文件、订单等真实状态。

- 轨迹能揭示：跳过确认、使用错误账号、反复调用失败工具、隐瞒错误。
- 结果能验证：订单是否创建、退款是否发起、文件是否真的可用。
- 合法替代路径应被允许；必要的授权步骤与安全要求仍需检查。

代码评估常同时使用 FAIL_TO_PASS 与 PASS_TO_PASS：前者检查目标问题是否修复，后者检查已有功能是否回归。它们提供测试覆盖范围内的证据，不能证明程序对所有输入都无缺陷。

来源：[轨迹与结果双重覆盖](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L269)、[代码的双重验证](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L206)、[状态、沟通与流程检查](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L208)。

## 11. Rubric：把“好不好”拆成可执行的判据

Rubric 是评分准则，不是一个固定的 Python 库函数。它可以被写成文本、配置或数据结构，再交给人类、代码或评判模型使用。

书中四项设计原则：

1. 专家指导：覆盖真正重要的领域事实和风险。
2. 全面覆盖：包括正面要求，也包括高风险陷阱。
3. 明确重要性：区分必要项、重要项、可选项与否决项。
4. 自包含：给出证据和判定条件，使评委不靠猜测完成评分。

用户记忆实验把评价拆为事实正确性、事实完整性、关系推理正确性、适当主动性，并设置幻觉否决；四档评分要配具体例子和边界案例。

“回答很有帮助”过于抽象，可以改为“回答用户直接问题、提供可执行下一步、明确缺失信息，不编造用户事实”。评估推理质量时，应检查答案及可观察行为中的关系是否成立，不要求获得模型未公开的内部思维过程。

本章 Rubric 的 Precision/Recall 可以采用档位评分；它们与按检索集合直接计算的精确率、召回率要区分。

来源：[Rubric 四准则](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L289)、[档位与边界案例](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L299)、[记忆评价维度](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L351)。

## 12. LLM-as-a-Judge：评委也必须校准

LLM-as-a-Judge 使用模型依据 Rubric 对候选输出评分，适合开放式回答的大规模评价。

常见误差包括长度偏好、候选位置偏好、风格偏好、同系列模型的共同盲点，以及同一输入重复评分的波动。

可用的校准流程：人工标注代表性金标集 → 检查评委一致性与分歧 → 修订 Rubric → 重新验证 → 扩大自动评估。更新评委或评分标准后，应重新校准。

盲化模型身份、交换候选顺序、检查分数与长度的关系、引入不同来源评委，都有助于发现偏差。严重分歧应回到人工审查。书中的金标集规模与 kappa 门槛是示例，实际门槛应结合任务风险预先制定。

Goodhart 问题提醒我们：持续围绕同一个分数优化，系统可能学会迎合评委。应保留独立验证集、真实状态检查和对抗案例。

来源：[人工校准与红队](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L273)、[长度偏差](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L285)、[共同偏差与 Goodhart](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L382)、[顺序偏差](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L426)。

## 13. 用户记忆对比：混合方案的协同要测出来

本章使用同一组 60 道合成题，对三套系统保留了 180 条实际 API 轨迹。

| 系统 | 基础回忆 | 多会话消歧 | 跨会话隐藏关联 | 总体成功 |
|---|---:|---:|---:|---:|
| Advanced JSON Cards | 95% | 60% | 50% | 41/60，68.3% |
| RAG | 90% | 40% | 15% | 29/60，48.3% |
| 混合系统 | 80% | 70% | 50% | 40/60，66.7% |

结构化卡片将事实放入上下文；RAG 按需检索原始对话；混合方案兼用两者。后者在 3 道题上超过两个单一方案，却在另外 8 道题上落后于其中表现较好的单一方案。

可以从中学习三点：检索相关不等于正确关联；增加组件可能引入干扰；应按失败类型决定哪些信息常驻、哪些按需检索。180 次评判中的 28 次幻觉否决，也说明安全判据会实质改变最终结果。

这些数字对应这组数据、模型和评判配置，适合分析失效边界；选择自己的方案仍需运行自己的任务集。

来源：[结果表](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L368)、[混合方案逐题分析](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L374)、[否决与实验范围](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L376)。

## 14. 多模态评价：参考对象本身就是实验设计

不同模态需要不同的质量维度：TTS 要看文字读音、自然度、情绪和音色；ASR 要区分不同转写错误的语义代价；UI 和视频还需要视觉结构与时序层面的检查。

TTS 小实验中，两个提供方各生成四类音频，总计 8 条，由音频评判模型直接听评。两者准确性和自然度均为 5.00、4.00；Fish 的情绪与音色得分为 4.00、3.00，OpenAI 为 3.75、2.75。

这里固定参考音频来自 Fish S1，因此音色相似度带有参考偏向。比较通用合成质量时，应独立报告这一维度；比较声音克隆时，应让各方案使用一致的目标说话人条件，并用人工盲听校准。

来源：[多模态维度](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L396)、[八条音频结果](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L410)、[参考音色偏差](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L412)。

## 15. 配对比较、Elo 与 Bradley–Terry

配对比较让评委在同一任务的两个输出中选择更好者，适合难以直接定义绝对分数的任务。比较时应隐藏模型身份，并控制或检查候选顺序的影响。

Elo 根据当前评分预测结果，再根据实际胜负逐场更新：

```text
预期得分 E_A = 1 / (1 + 10^((R_B - R_A) / 400))
新评分 R_A' = R_A + K × (实际得分 S_A - E_A)
胜 / 平 / 负的 S_A 分别为 1 / 0.5 / 0
```

有平局时 E_A 更准确地说是包含平局半分的预期得分。Bradley–Terry 使用潜在强度解释两两结果，书中实现对固定数据整体拟合，再转换为类似 Elo 的分数尺度。

在线 Elo 受 K 与对局顺序影响；固定数据的整体拟合不靠逐场时间更新。两者有相近的建模思想，但不应要求数值完全一致。

榜单反映采样任务、人群偏好和评估协议。模型在代码、写作、简洁性上的优势可以不同，单一总榜会压缩这些差异。历史分数变化也要结合任务和投票人群变化解释。

来源：[配对比较与 Elo](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L422)、[顺序偏差](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L426)、[在线更新与整体拟合](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L434)。

## 16. 模型选型：看完整任务与预算曲线

选型要同时考虑任务质量、延迟、吞吐、价格、限流、上下文容量和可靠性。

- Prefill：处理输入上下文。
- Decode：逐步生成输出，包括服务实际产生的相关 token。
- TTFT：从请求到首个 token 的时间，测量时要明确是否包括网络、排队等时间。
- 端到端延迟：用户发起任务到最终完成，包含多次模型调用、工具、重试与等待。
- p50 / p95 / p99：延迟分布的不同分位数；尾部指标特别依赖足够的样本。

补充说明：原文把 TTFT 简化为排队加 Prefill；端到端测量还可能包括网络、协议开销和首 token 解码。存在思考阶段时，“首个返回 token”和“首个用户可见答案 token”也应分开。

应画出或统计不同时间、token、工具预算下的表现。一个模型可能在短预算下占优，另一个在长任务中更可靠。低 token 单价也可能被更多重试抵消。

来源：[模型选型维度](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L441)、[延迟与吞吐](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L445)、[任务成本与预算](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L454)、[预算曲线](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L460)。

## 17. 行动阈值：读多少再动手，也与模型有关

模型行动阈值实验固定中性 Harness、工具声明、任务与预算，比较两个模型在三道代码任务上各三次运行，共 18 条轨迹。

记录的模型名为 GPT-5.6-sol 与 Claude Sonnet 5。前者首次修改前平均调用工具 6.89 次、读取 4.67 个文件，后者为 4.56 次、3.56 个文件；两者首个受测补丁和最终测试均为 100% 通过，首次修改耗时也接近。

本实验展示“行动策略随模型而变”：多读文件、早动手、工具步数和墙钟时间不是同一个指标。设计 Harness 时，应观察这些行为是否服务于任务质量与成本，而不把某种固定风格当成普遍最优。

模型名与结果均是书中保存的历史实验记录，不代表当前产品能力排序。

来源：[18 条轨迹与结果](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L470)、[控制变量](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L476)。

## 18. 多轮成本：上下文重复输入、缓存与压缩

Agent 每轮通常携带已有历史，因此一条工具结果可能在之后多轮继续占用输入 token。若每轮增加固定长度且每次完整重发，累计输入长度会随轮数近似二次增长。

任务成本应统计未缓存输入、缓存输入、输出、工具费用与基础设施等项目。思考 token 是否包含在输出用量中，应按实际服务字段核算，避免重复计费。

稳定前缀有助于跨请求命中提示缓存，压缩则减少历史长度。它们与单次推理内部使用的 KV cache 属于相关但不同的观察层级，实际优惠要以服务返回的缓存用量和价格规则为准。

本章八轮退款任务的 2×2 记录如下，美元费用使用当次实验价格：

| 方案 | 输入 token | 缓存 token | 记录总成本 | 比基线节省 |
|---|---:|---:|---:|---:|
| 无缓存、无压缩 | 20,700 | 0 | $0.003776 | — |
| 仅稳定前缀 | 20,386 | 13,568 | $0.002707 | 28.3% |
| 仅压缩历史 | 16,177 | 0 | $0.003115 | 17.5% |
| 稳定前缀与压缩 | 16,035 | 6,144 | $0.002643 | 30.0% |

节省比例不能直接相加：压缩改变了可复用的历史，缓存与压缩会相互影响。自己的核算还应明确是否计入压缩调用、失败重试与工具费用。

来源：[多轮与工具成本](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L493)、[实验条件](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L499)、[成本表](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L503)、[两种优化的相互作用](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L512)。

## 19. 全链路选型：嵌入、重排、主模型存在交互

记忆检索的最终效果取决于整条链路：对话分块 → 嵌入与召回 → 重排 → 主模型使用证据 → 回答与评分。

应保留“不使用 reranker”的基线，并在相同嵌入和主模型下比较重排的边际收益。检索更准可能减少主模型补救的需要；更强的主模型也可能改变某个检索组件的价值。

配套记录完成了 60 个案例 × 4 个嵌入配置 × 3 个重排配置 × 2 个主模型，共 1,440 条轨迹。部分后端发生了明确替换：Qwen3-Embedding-8B 替代不可用的豆包嵌入端点，Doubao-LLM 重排替代无法访问的 BGE cross-encoder；另有相同模型改用 OpenRouter 路由。

因此结果应按实际运行配置解读。保存文件及部分函数仍叫 `6_9` / `run_69`，对应的是当前书稿的实验 6-10，这是历史编号，不是另一个实验。

来源：[全链路选型要求](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L556)、[交互作用](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L558)、[实际矩阵与替换](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/EXPERIMENT_LEDGER.md#L18)、[历史函数名](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/user-memory-system-evaluation/experiment.py#L1119)。

## 20. 统计判断：小幅涨分是否值得相信

在近似独立的二元任务样本上，成功率估计的标准误可粗略计算为：

```text
SE ≈ sqrt(p × (1 - p) / n)
n = 100，p = 0.70 时，SE ≈ 0.0458，即 4.58 个百分点
正态近似的 95% 区间 ≈ 70% ± 9 个百分点
```

比较两个配置时，若在同一批题目上运行，默认应做配对分析，检查“旧对新错”和“旧错新对”的数量，例如采用 McNemar 检验。只看 70% 与 73% 两个汇总数字，会丢失逐题信息。

重复运行反映模型、工具与环境波动；增加不同任务则扩大任务覆盖。相同任务的多次运行不能不加区分地当成完全独立的新任务样本。

本节对原文的统计表述作三点精确化补充：

1. 95% 置信度描述构造区间的方法在重复抽样中的覆盖性质，不是给一个固定参数赋予“落在当前区间的概率”。小样本或接近 0/1 时，上述正态近似可能较差。
2. 两个成功率之差的标准误为单个的 √2 倍，需要独立且方差相等。对配对样本，还取决于协方差；独立近似并非任何情况下都成立的保守上界。
3. `1 - 0.95^6 ≈ 26.5%` 对应六次相互独立、原假设均成立、单次假阳性率 5% 的检验。逐轮试验后挑最好结果也会有选择偏差，单变量或顺序运行本身不消除多重比较问题。

实际决策应预先确定主要指标与门槛，保留独立验证集，结合效应大小、配对结果、不确定性和成本决定是否上线。

来源：[标准误示例](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L565)、[重复运行](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L567)、[配对分析](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L569)、[多重比较](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L571)。

## 21. 可观测性：Trace 和 Span 怎样支持评估

一次任务对应一条 Trace，模型调用、检索和工具执行可各自记录为 Span。完整系统可用父子关系组织这些执行单元，记录输入输出、起止时间、用量、错误和配置身份。

可观测性主要服务三件事：定位哪里失败、解释哪里耗时耗钱、为回归测试保存证据。排查时要能把一次失败对应到具体模型、提示词、工具和环境版本。

书中轻量级 `Tracer.chat()` 包装实际模型调用，读取用量、缓存 token、思考 token 和耗时，构造 Span 后加入列表。它主要演示调用级成本记录；完整的分布式 span 树还需要父子标识等机制。

两个代码阅读细节：该实现使用 `time.time()` 相减计时；若做更稳健的持续时间测量，可使用单调时钟。它在 usage 缺失时以零保存成本，这是程序回退值，分析时应将其识别为“用量未知”，不能当作请求免费。

并行场景中，各 Span 耗时相加不等于用户实际等待的墙钟时间。还应限制敏感内容采集，经过脱敏与人工筛选后，把生产失败变成可复现的新用例。代表性任务集和专门的失败回归集应分别报告。

来源：[Trace 与 Span](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L585)、[失败回流](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L591)、[调用包装](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/agent-cost-analysis/tracer.py#L81)、[缺失用量处理](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/agent-cost-analysis/tracer.py#L92)、[耗时求和](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/agent-cost-analysis/tracer.py#L191)。

## 22. AndroidWorld：从失败簇到分层假设

本章用四个 Wi-Fi 设置任务演示诊断。Agent 反复导航且无法确认结果，可能是不知道入口，也可能是根本没看到完整界面信息。

| 假设 | 本轮改动 | 成功率：对照→实验 | token：实验/对照 | 下一步 |
|---|---|---:|---:|---|
| H1 | 增加导航提示 | 25%→25% | 0.47× | 此项提示改动未提高成功率 |
| H5 | 使用 UIAutomator 元素树 | 25%→100% | 2.498× | 观察信息改善，但成本上升 |
| H5C | 精简元素树 | 100%→100% | 0.506× | 扩大任务集复测 |

三轮对照固定模型、任务参数、种子、步数上限和模拟器，并交替安排组间运行顺序。H5C 的 0.506× 相对于本轮完整元素树对照，不是最初方案；各轮比例不能脱离各自基线使用。

这组实验支持检查输入表示、进一步压缩观察内容的方向。四题各一次的结果只够筛选候选方案；H1 失败也只说明这项提示改动没有奏效。

正文提出下一步在 Pixel 6 / API 33 上运行 116 题×5 次，并检验成功率、token 与延迟门槛。台账后来记录了 580 次完整执行，但候选使用本地 Qwen2.5-7B，与配对来源的 Doubao 模型不同，因此未批准部署，也不能据此宣布同模型提升或非劣效。

台账同时给出严格 T3A 成功 26/580、平均 evaluator reward 0.133621。它们是不同口径，不能互相替换；零运行时错误也不等于任务成功。

来源：[小实验范围](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L606)、[控制变量](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L626)、[三轮结果](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L634)、[扩大验证门槛](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L644)、[后续执行与未批准部署](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/EXPERIMENT_LEDGER.md#L19)。

## 23. 内部评估基础设施：让实验成为日常工程

每个重要特性最好可独立关闭。开关需要在初始化早期生效，否则模块已缓存配置，后续关闭可能不会改变真实执行路径。

| 设计 | 作用 |
|---|---|
| 独立消融开关 | 测试组件贡献，识别失去价值的“特性债务” |
| 多臂实验 | 比较不同强度，观察效果随配置怎样变化 |
| 机制指标 | 检查改动是否按预期发生，如计划长度变短 |
| 目标指标 | 检查真正收益，如任务成功率、完整会话成本 |
| 护栏指标 | 限制安全、错误率、满意度等方面的退化 |
| 编译时开关 | 在特定构建中移除相应代码 |
| 运行时开关 | 支持分组、渐进发布与紧急关闭 |

“计划更短”是机制变化；若引起更多返工，会话总成本反而可能上升。实验应以目标指标和护栏共同判定。

运行时缓存可以降低启动依赖，但安全撤权类配置还需要明确刷新与失效策略。实验曝光按会话去重，有助于避免重复事件污染统计。

来源：[消融工程化](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L669)、[早期初始化](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L671)、[机制与目标](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L679)、[护栏](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L681)、[双层开关](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L689)。

## 24. 提示词版本与隐私约束也是评估的一部分

系统提示应能根据固定配置确定性渲染，保存最终快照，并随版本运行回归测试。只保存模板不够，因为动态条件可能改变模型真正看到的文本。

隐私保护应在采集接口设计时考虑：最小化记录字段、限制访问、脱敏、设置保留期限。书中的类型包装能让分析接口明确要求经过审查的数据，便于编译检查和代码审计。

补充说明：类型检查验证的是接口使用方式，无法自动证明字符串里没有姓名、密钥或文件路径。仍需要可信的校验/转换流程与运行时的数据管理措施。

来源：[提示词快照与回归](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L697)、[确定性渲染](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L699)、[隐私类型包装](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L703)、[隐私前置设计](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L705)。

## 25. 从评估环境到训练仿真环境

评估环境用来测量能力；训练环境让 Agent 反复执行任务并获得奖励。两者可以共享任务生成、工具、状态与验证器，但训练需要更高吞吐、更可靠的 reset，以及防止记住固定实例的随机化。

一个 episode 是从初始状态到终止的一次完整交互。reset 必须清理上一轮残留，否则下一轮的观察与奖励可能被污染。

可执行测试和状态检查可以成为可验证奖励。补充说明：把 LLM 根据 Rubric 给出的分数用作奖励，并不自动使它成为客观可验证的 RLVR 信号；评委偏差和奖励作弊仍需处理。

具身实验使用 OpenVLA 与 RoboTwin2，观测包含三视角 RGB 和 14 维关节状态，动作是 14 维控制向量。动作分块一次生成多个连续动作，改变了执行与反馈的节奏。

台账保存的 512 次 rollout 中，chunk=1 为 0/256，chunk=25 为 26/256，后者在 IID、OOD 各为 13/128。这个结果保留了低绝对成功率，体现“完整评估证据”和“高性能系统”是两件事。

领域随机化通过改变位置、视觉、物理参数、噪声或数字环境延迟，训练更广的适应能力。随机化范围要与真实场景相关，并用独立分布测试迁移效果。

来源：[评估到训练](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L713)、[reset 与吞吐](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L715)、[具身观测与动作](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L725)、[领域随机化](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L734)、[512 次评估记录](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/EXPERIMENT_LEDGER.md#L20)。

## 26. 十二个实验的学习地图与完成口径

以下是仓库保存证据的状态，区别于本次总结工作的执行情况。本次只核对资料，未调用付费模型、启动模拟器或重跑实验。

| 实验 | 学习目标 | 保存证据与关键边界 |
|---|---|---|
| 6-1 | τ²-bench 与双控环境 | 五个电信任务成功 4/5；已完成限定范围运行，不是完整榜单评测 |
| 6-2 | 体验六种 benchmark 的不同难度 | 18 个任务、13 成功、5 失败；操作者记录为 Codex-as-human，不能称为真实人类专家基线 |
| 6-3 | 构建多维 Rubric | 60 例、180 次结构化评分，含幻觉否决 |
| 6-4 | Cards / RAG / 混合对照 | 180 条真实轨迹，分析分层质量、费用与失效边界 |
| 6-5 | TTS 多维直接听评 | 8 条音频，保存参考音频身份；音色比较有参考偏向 |
| 6-6 | 在线 Elo 与 Bradley–Terry | 1,799,991 源记录中采用 1,670,250 条盲投票，129 个模型；保存排名与历史快照 |
| 6-7 | 固定 Harness 比较行动阈值 | 2 模型×3 任务×3 次，共 18 条轨迹 |
| 6-8 | 任务成本与缓存/压缩交互 | 四组各八轮，保留真实 token、缓存与耗时记录 |
| 6-9 | 吞吐、尾延迟、限流与一周可用性 | **未完成**：仅 29 条 smoke/readiness 观察，缺标准 N≥100 单元、限流/Agent 成本阶段与 168 小时运行 |
| 6-10 | 嵌入×重排×主模型选型 | 1,440 条轨迹；包含已披露的后端替换，旧文件名仍含 6_9 |
| 6-11 | AndroidWorld 诊断与改进 | 后续 580 次执行已保存；模型更换使同模型比较不成立，部署未批准 |
| 6-12 | OpenVLA / RoboTwin2 与动作分块 | 512 次 rollout；chunk=25 成功 26/256，保留低成功率与超时证据 |

实验 6-9 中，每小时探测一周只能观察采样时点及可推定的故障区间，不代表已经证明每一秒都连续可用。

来源：[完整实验台账](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/EXPERIMENT_LEDGER.md#L7)、[6-9 尚未完成](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/EXPERIMENT_LEDGER.md#L17)、[标准工作负载与一周探测要求](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L548)。

## 27. 配套代码怎样读

### 27.1 先看实验身份和完成条件

先读台账找到实验对应目录，再看配置、任务、评判器和真实结果。`smoke` 通常只验证能否跑通；判断完成还要检查预期任务数、配置覆盖、失败记录、价格缺失与产物身份。

`model-benchmark/analysis.py` 中的 `completion_audit()` 会核对 8K/32K/128K 输入、512/2048 输出、每单元至少 100 次、一周可用性和限流设计等要求。这解释了为什么代码已实现、跑过若干次，仍可标记为未完成。

来源：[完成性审计](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/model-benchmark/analysis.py#L331)。

### 27.2 记忆实验主线

```text
TestCase → 对话分块 / 构建卡片 → 选择系统与后端组合
                                      ↓
                              Agent 回答并记录用量
                                      ↓
                         Judge 评分 + 检索指标 + 成本
                                      ↓
                     RunRecord → 汇总 / 交互分析 / 完成性检查
```

`run_64()` 对应三种记忆系统对照；`run_69()` 保留旧编号，对应当前实验 6-10 的组件矩阵；`interaction_analysis()` 比较固定其他组件后的边际增益；`completion_assessment()` 检查重复、缺失和额外配置单元。

`retrieval_metrics()` 截取前五个片段，依次返回 Hit@5、Recall@5、前五名范围内的倒数排名。第一项“是否至少命中一次”不是 Precision@5，名称解释必须与代码运算一致。

来源：[检索指标计算](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/user-memory-system-evaluation/experiment.py#L824)、[三系统对照](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/user-memory-system-evaluation/experiment.py#L1014)、[配置矩阵](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/user-memory-system-evaluation/experiment.py#L1119)、[交互分析](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/user-memory-system-evaluation/experiment.py#L1301)、[完成性检查](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/user-memory-system-evaluation/experiment.py#L1587)。

### 27.3 两个适合独立阅读的小模块

- `elo_rating.py`：先读预期得分，再读胜负和平局如何更新评分，容易看清公式与程序的映射。
- `tracer.py`：先读 Span 字段，再读 `chat()`，最后看缓存、未缓存输入与输出成本如何汇总。

Bradley–Terry 的 `compute_mle_elo()` 则整理胜负与平局记录，使用加权逻辑回归拟合整体强度，再转换评分尺度。

来源：[Elo 预期与更新](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/elo-leaderboard/elo_rating.py#L39)、[Bradley–Terry 拟合](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/elo-leaderboard/bradley_terry.py#L12)、[Span 字段](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/agent-cost-analysis/tracer.py#L43)、[成本拆分](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/agent-cost-analysis/tracer.py#L198)。

## 28. 容易混淆的概念速查

| 容易混淆 | 正确理解 |
|---|---|
| 模型更强＝Agent 一定更好 | 任务表现取决于模型、Harness、预算与评估条件 |
| Pass@5 很高＝五次都稳定 | 至少一次成功与全部成功是两个指标 |
| 工具调用格式合法＝业务正确 | 还要检查对象、参数含义、权限与真实结果 |
| LLM Judge 打高分＝已客观验证 | 评委需要金标校准与独立验证 |
| 混合记忆＝两个系统优点相加 | 组件可能协同，也可能引入噪声和额外成本 |
| 运行零报错＝任务全部成功 | 运行状态、任务奖励、发布门槛分别判断 |
| 小样本 100%＝整体可靠 | 报告样本与范围，并扩大独立验证 |
| 压缩节省＋缓存节省＝总节省 | 两种改动存在交互作用 |
| 总 Span 耗时＝墙钟时间 | 并行和嵌套记录需要区分时间口径 |
| 关闭所有功能＝定位了各功能贡献 | 组合消融还需分项验证 |
| 有隐私类型＝数据一定脱敏 | 类型约束需要真实校验流程支持 |
| 验证器可以做奖励＝评估题能进训练 | 环境机制可复用，评估实例仍需隔离 |

本表归纳自前述各节；统计、隐私和缓存的补充说明用于限定这些结论的适用条件。

## 29. 以后可以提炼成哪些 Skill

适合提炼的是带输入、步骤、验收和产物的工作流程。以下仅是候选清单，本次没有创建 Skill 文件。

| 候选 Skill | 触发场景 | 核心步骤 | 预期产物 |
|---|---|---|---|
| Agent 评估方案设计 | 新 Agent 准备验收 | 定义任务、状态、工具、协议与指标 | 评估契约与覆盖矩阵 |
| Rubric 设计与评委校准 | 开放任务评分不稳定 | 设计档位、否决、金标、分歧审查 | Rubric、判例与校准报告 |
| Harness 消融与模型替换 | 不清楚瓶颈在哪里 | 固定条件、独立开关、配对分析 | 组件贡献和选型报告 |
| Trace 成本诊断 | 多轮调用过慢或过贵 | 统计输入、输出、缓存、工具、重试 | 成本与延迟拆解 |
| 评估失败归因 | 总分下降、局部连续失败 | 先查评测、回放轨迹、构造单变量假设 | 失败簇与验证计划 |
| 组件交互实验 | 多组件选型 | 定义配置矩阵、基线、交互与预算 | 实际配置清单及端到端权衡 |
| 评估证据审计 | 需要判断实验是否真的完成 | 查覆盖、版本、产物、错误和结论范围 | 完成性清单与发布门槛结论 |

共同要求：记录实际运行版本与配置，报告失败和缺失项，分开探索集与验证集，保护原始轨迹中的隐私。

来源：[评估环境结构](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L75)、[校准机制](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L273)、[改进闭环](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L646)、[台账的完成性原则](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/chapter6/EXPERIMENT_LEDGER.md#L3)。

## 30. 学完本章后应能回答的问题：20 题及参考答案

下列答案是学习归纳；“原文依据”给出书中短摘录，并附可定位来源。

### 1）为什么评估对象要包含 Harness？

答：同一个模型在不同提示词、工具、上下文管理和反馈机制下，能完成的任务不同。评估组合体才能反映实际交付能力，也才能定位改模型还是改运行框架。

原文依据：“评估的对象不应只是模型，而应是模型与 Harness 的组合体”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L15)

### 2）模型替换与消融实验有什么区别？

答：模型替换固定 Harness，只换模型；消融固定其他条件，关闭某个 Harness 组件。前者观察模型敏感性，后者检查组件贡献。结果都要结合任务和样本解释。

原文依据：“消融是**关闭 Harness 的某个组件**看整体性能如何变化”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L15)

### 3）搭建评估环境需要哪些要素？

答：任务数据集、可重置的环境状态、工具接口、评分标准和交互协议。它们共同规定从哪里开始、能做什么、怎样算完成和何时结束。

原文依据：“数据集（Dataset）”“环境状态（Environment State）”“工具接口（Tools）”。评分与协议接在同一节中。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L75)

### 4）为什么用户信息不能一次性全部提供？

答：逐步提供才能测量澄清需求、主动提问和确认信息的能力。事实必须固定，透露方式按协议变化，并检查模拟行为是否贴近目标用户。

原文依据：“信息应按需、渐进地在对话中透露”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L110)

### 5）τ²-bench 的双控是什么意思？

答：Agent 与用户模拟器都能操作共享环境。Agent 需要指导用户，并读取用户动作导致的真实状态变化，再继续处理任务。

原文依据：“用户模拟器也能操作同一个共享环境”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L128)

### 6）任务目标明确与解决路径开放矛盾吗？

答：不矛盾。目标、约束和验证条件可以固定，执行步骤允许不同。评分应接受合法的替代方案，同时检查必须遵守的授权与安全流程。

原文依据：“任务描述必须足够明确以确保评估可复现，又不能过于死板限制 Agent 的创造性”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L166)

### 7）Pass@k 和 Pass^k 应怎样选择？

答：探索多次尝试的可达能力，使用 Pass@k；检查多次执行都成功的可靠性，使用 Pass^k。报告时注明 k、任务数、重复方式与环境重置条件。

原文依据：“k 次尝试中**至少有一次**成功的概率”；“k 次尝试**全部成功**的概率”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L252)

### 8）为什么既看轨迹又看最终状态？

答：轨迹检查行为是否合法、有无必要确认；最终状态检查是否真正完成。两者结合能发现“说完成却没做到”和“做到了但过程违规”。

原文依据：“数据库里确实生成了一条订单才是结果层面的验证”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L269)

### 9）Rubric 的四项原则是什么？

答：专家指导、全面覆盖、重要性权重、自包含评估。再通过分档示例和边界案例，让评委对具体证据做判断，并为严重问题设置否决项。

原文依据：“基于专家指导”“全面覆盖”“标准重要性权重”“自包含评估”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L291)

### 10）为什么 LLM 评委也要测试？

答：评委可能偏爱长答案、特定风格或某个位置，也会波动。应与人工金标比较，检查分歧，在换评委或改 Rubric 后重新校准。

原文依据：“先构建一个人工标注的金标集”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L273)

### 11）混合记忆系统为什么没有自然胜出？

答：混合增加的信息和检索步骤可能帮助关联，也可能引入干扰。实验中它在部分题上超过两个单一方案，同时也在另一些题上退步，因此要逐题分析协同与损失。

原文依据：“混合方案并没有自然胜出”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L374)

### 12）为什么参考音频会改变 TTS 比较结果？

答：音色一致性衡量的是与指定参考的相似度。参考来自某个提供方，会影响对其音色的评价；需要区分通用 TTS 质量与模仿固定说话人的能力。

原文依据：“选什么参考答案、参考图片或参考音频，本身就是评估设计”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L412)

### 13）在线 Elo 与 Bradley–Terry 为什么不必逐分一致？

答：在线 Elo 逐场更新，受 K 与顺序影响；本章 Bradley–Terry 使用整体对局拟合强度。应检查数据、排名关系和不确定性，而不要求两种估计数值完全一样。

原文依据：“结果受学习率 K 因子和处理顺序影响”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L434)

### 14）为什么不能只用 token 单价选模型？

答：真实任务还包括输入历史、生成长度、工具、重试和等待。便宜模型如果需要更多轮才能完成，任务总成本可能更高；质量与尾延迟也要同时衡量。

原文依据：“成本与延迟”关注“请求次数、Token 花费”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L246)

### 15）缓存与压缩的节省比例能直接相加吗？

答：不能。压缩改变后续请求内容，也影响缓存可复用前缀。要在相同任务上运行两因素四组实验，测量最终成本和质量。

原文依据：成本表分别报告“仅稳定前缀”“仅压缩历史”“稳定前缀 + 压缩”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L503)

### 16）新配置从 70% 提升到 73%，能马上切换吗？

答：需要样本量、逐题配对结果和波动信息才能判断。应检查改对与改错的任务，估计差异的不确定性，再结合收益大小和发布门槛决定。

原文依据：“同一批任务上对比两个配置，正确的默认做法是**配对分析**”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L569)

### 17）可观测性怎样变成评估资产？

答：保存任务执行过程，筛选真实失败与可疑案例，脱敏后重建为可复现测试。以后修改系统时，检查这些问题是否再次出现。

原文依据：“从生产轨迹中筛选出失败与可疑案例 → 脱敏处理”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L591)

### 18）AndroidWorld 四题从 25% 到 100%，说明可以上线吗？

答：说明输入表示的改动值得扩大验证。原文要求补齐标准环境、116 题多种子测试，以及成功率、费用和延迟门槛；后续台账还必须检查模型是否一致。

原文依据：“H5C 通过了这四项任务的检查，只说明它值得进入下一轮，并不等于可以部署”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L644)

### 19）机制指标、目标指标和护栏有什么区别？

答：机制指标说明改动是否发生；目标指标说明改动是否有价值；护栏说明是否损害不能退让的要求。例如计划长度、会话总成本、任务错误率分别属于三类。

原文依据：“计划长度是机制指标”；“不能变差的底线”。[来源一](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L679)、[来源二](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L681)

### 20）评估环境怎样转化为训练环境？

答：复用任务生成、工具和验证器，将可用判据转成奖励，同时强化 reset、吞吐与随机化。训练实例和保留评估实例仍需隔离，模型评委奖励也要防偏差与作弊。

原文依据：“可靠的 reset 语义”；“远高于评估的吞吐”。[来源](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L715)

## 31. 原书八道思考题：参考分析与原文依据

以下题意为归纳，链接指向原题。答案是在本章原则上提出的参考方案，不是原书给出的唯一标准答案。

### 思考题 1：怎样发现并校正模型评委的系统性盲区？

先用人工金标确定目标人群的评价标准，再构造控制案例：事实相同但篇幅不同、风格不同；篇幅相近但一份含隐蔽错误；交换两个候选的位置。分别检查正确性、长度、位置、风格和任务类型上的偏差。

校正时使用明确 Rubric、盲化身份、顺序平衡和不同来源评委；分歧大的样本交由人工裁决。要保留未用于修改 Rubric 的验证集，防止只是把评委调到适应校准样本。多个评委意见一致仍可能存在共同盲点。

原文依据：“当评判者之间严重分歧时，标记为需进一步人工审查”。[原题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L754)、[校准与多评委](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L273)、[共同偏差](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L382)。

### 思考题 2：怎样设计持续抵抗数据泄漏的评估？

可以公开能力目标与任务生成规范，把一部分实例、组合方式和随机种子保留在隔离环境中；评估时生成新实例，要求实际操作随机初始化的环境，并直接检查状态或产物。定期加入新问题、保留跨时间测试、审计数据访问与训练使用。

要同时改变任务参数与有意义的组合结构，仅替换姓名可能仍只测到熟悉模板。金丝雀用于调查暴露路径，不能单凭标识出现就忽略检索或提示输入的可能来源。

开放生态下更现实的目标是持续增加泛化证据、减少背答案优势。任何固定公开测试都可能被反复适配，因此“从根本上抵抗”需要制度、任务更新与技术验证共同工作。

原文依据：“评估集本身的那些具体题目必须与训练数据严格隔离”。[原题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L755)、[防污染策略](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L174)、[实例隔离](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L234)。

### 思考题 3：怎样给“有帮助”“语气恰当”设计可靠 Rubric？

先指定受众和情境，再定义可观察行为。例如投诉回复可以看：是否回应实际诉求、是否承认已知问题、是否给出可执行下一步、是否避免无依据承诺。将严重失误与一般风格偏好分开，再提供高分、低分与边界样例。

由多位相关领域评委独立标注，分析分歧是否来自描述模糊、事实不足或价值偏好不同。前两类可以补规则和证据；后一类应记录目标人群与权重，而不是伪装成唯一客观答案。

可靠 Rubric 的目标是让判断透明、稳定且可复核，主观维度仍然保留其情境与价值选择。

原文依据：“为每个维度定义客观可验证的评分档次，提供具体示例和**边界案例**”。[原题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L756)、[四准则](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L291)、[判例设计](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L299)。

### 思考题 4：怎样验证用户模拟器本身的质量？

验证分三层：事实忠实度、交互行为、环境动作。检查它是否编造信息、提前透露答案、无条件配合；是否覆盖模糊表达、情绪与需求变化；双控任务中是否真的执行用户侧动作并报告实际结果。

在获得授权并脱敏的真实交互样本上，由人工比较模拟器的提问响应、信息透露节奏、确认行为与终止方式。测试多个模拟器配置，观察 Agent 排名是否对某个模拟器过度敏感。

还可注入故意跳过确认、问错对象的 Agent 行为：如果模拟器仍帮助其完成，说明测试过于宽松。普通用户分布与专门的困难用户压力测试应分开汇报。

原文依据：“不要编造指令中未提供的信息”。[原题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L757)、[模拟器行为约束](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L112)、[事实锚定](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L180)。

### 思考题 5：非传递偏好会怎样影响排名？

在不同任务或人群混合时，A 可能因代码正确率胜 B，B 因沟通质量胜 C，C 又因简洁性和速度胜 A。上下文相关偏好汇总后可能形成循环，单一总分会掩盖这种结构。

Bradley–Terry 的标量强度对胜率施加有序结构；这不等于模型要求每一条实际投票都传递，有限样本中的循环可以自然出现。需要检查的是系统性的两两结果与拟合预测是否不一致。

实践中可展示经验胜率矩阵、按任务与用户分层、报告排名不确定性，必要时使用任务条件化评价。业务选型优先看自己的任务分布，而不是强行给所有场景排唯一名次。

原文依据：原文区分“在线增量更新的 Elo”与“Bradley-Terry 极大似然拟合”。[原题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L758)、[配对比较](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L422)、[拟合方法区别](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L434)。

### 思考题 6：预算有限，怎样提高每次评估的信息量？

先从轨迹识别集中失败，再提出可区分的假设；选择少量代表任务做配对试验，固定模型、种子、环境和预算，只改变待验证因素。测量中间机制，判断假设是否真的改变了输入或行为。

明显无效或违反安全底线的方向尽早停止；有潜力的方向再扩到分层任务集和多次重复。预先记录主要指标、最小有意义收益、停止规则和验证集，避免反复试到显著才宣布成功。

选择探索样本时可以优先高信息量病例，最终效果仍要在代表性保留集上确认。大量候选的筛选与最终验证要分离，多重比较不会因为实验逐轮进行而自动消失。

原文依据：“一轮证据只支持与它规模相称的下一步”。[原题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L759)、[配对与样本量](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L569)、[逐轮决策](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L646)。

### 思考题 7：怎样裁剪 UI 树又不丢关键语义？

可以采用分层保留规则：

1. 保留可交互节点及必要定位属性，例如点击、输入、滚动、选择状态和边界信息。
2. 保留文本、可访问性描述、错误提示、加载或禁用状态、当前焦点等语义节点。
3. 保留理解分组、父子关系与继承状态所需的上下文；折叠容器时向子节点传递必要属性。
4. 只删除已确认无语义、无操作用途的结构节点，压缩重复内容，并允许按需展开完整子树。
5. 用已有成功任务和失败任务回归，比较元素可定位性、最终状态判断、成功率与总成本。

“没有文字”不能单独作为删除标准：图标按钮可能通过可访问性描述表示含义，容器也可能携带状态或滚动能力。裁剪规则要有版本与可回放的原始观察。

原文依据：“提示写得更详细，也补不回 Agent 根本没有看到的信息”。[原题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L760)、[元素树替换与精简](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L624)、[输入表示的作用](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L640)。

### 思考题 8：渐进式透露怎样影响结果，模拟与真实不一致怎么办？

渐进式透露增加了澄清与确认的要求，也会增加对话轮数、费用和超时风险。信息一次性全部给出时，评估更偏向静态求解；逐步透露时，还在测主动获取信息和协作能力。

如果模拟器比真实用户更配合，结果可能偏高；如果过度隐藏信息，也可能偏低。应在事实相同的条件下比较多种透露策略，做敏感性分析，再用目标用户的授权、脱敏样本校准策略分布。

当策略差异足以改变排名，应报告条件化结果，并补充真实用户验证；不要把单一模拟器下的成功率直接当成线上成功率。

原文依据：“Agent 需要通过主动提问来澄清需求，这个过程本身就是能力的重要体现”。[原题](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L761)、[渐进透露原则](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L110)、[真实性与可控性](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L112)。

## 32. 一页复习：本章应建立的工程判断

评估对象是模型与 Harness 的组合。先定义任务、环境、验收和协议，再运行对照并保存轨迹；评分同时覆盖真实结果、必要过程、成本与安全。

指标必须回答具体问题：Pass@k 看多次机会下的可达性，Pass^k 看可靠性；Rubric 分解质量，金标校准评委；模型和组件选型要检查交互、预算与不确定性。

分数变化后先排除评测问题，再按失败簇构造假设。小样本用来筛方向，独立且充分的验证支持发布。代码跑通、实验完成、质量达标、允许上线，是不同的判断。

最后，把生产失败变成回归测试，把提示词、开关和评判器纳入版本管理。环境生成机制还能支持训练，但训练数据与保留评估实例必须隔离。

与前五章连接起来，可以把本章概括为：**前面学习怎样让 Agent 行动，这一章学习怎样证明行动有效，并据此持续改进。**

来源：[组合体评估](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L15)、[先检查评测系统](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L612)、[证据与决策范围](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L646)、[连接后续训练与演化](https://github.com/bojieli/ai-agent-book/blob/d39b1d74702c7fec9e6767acf6f98aff901725ae/book/chapter6.md#L750)。
