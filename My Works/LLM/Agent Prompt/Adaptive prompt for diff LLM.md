
# LLM Agent 跨模型 Prompt 适配：研究工作梳理

> 主题：同一个功能/任务的 agent，针对不同 LLM 使用不同（适配后的）prompt 以保持等价行为。
> 目标：帮你快速建立这个方向的研究地图、核心文献与工程落地路径。

---

## 一、问题到底是什么

你关心的现象在文献里被命名为 **Model Drifting（模型漂移）**，由 PromptBridge（Wang et al., 2025）系统刻画：为模型 A 精调的 prompt，直接搬到模型 B 上，性能往往明显低于"为 B 专门优化的 prompt"。他们用 **transfer gap** 来度量这一损失：

$$
\Delta(M_s\!\to\!M_t, T)=A(M_t, T, p^{*}_{M_s,T})-A(M_t, T, p^{*}_{M_t,T})    
$$

即"源模型最优 prompt 直接迁到目标模型的表现"相对于"目标模型自身最优 prompt 的表现"的 shortfall（通常 ≤ 0）。论文在 HumanEval/MBPP/APPS/xCodeEval/CodeContests/SWE-Bench Verified/Terminal-Bench/TravelPlanner 等 code 与 agent benchmark 上实证，model drifting **普遍且严重**：作者公开举例，为 GPT-5 优化的 prompt（自身 ~99% 准确率）直接迁到 Llama-3.1-70B 会明显低于 Llama 自身最优 prompt；在 SWE-Bench Verified 上把 o4-mini 优化的 prompt 直接迁到 o3 仅 33.40%，经 PromptBridge 迁移后恢复到 46.00%（以上数值来自论文作者公开说明与解读，精确表格请以原文为准）。根因是不同模型在训练语料/分布、分词与角色系统、对齐与 RLHF 流程、API 约定上存在系统差异——所以"同一份 prompt 一把梭"不可持续，模型一升级/一换 API 就退化。

由此可把你的问题归为三层递进的研究子问题：
1. **刻画**：同一 prompt 在不同模型上行为差异有多大、为什么（迁移性 & 鲁棒性）。
2. **适配**：为每个模型自动/半自动得到一份适配 prompt（model-specific prompt optimization）。
3. **迁移**：从源模型 prompt 直接"翻译"出目标模型 prompt，免逐任务重优化（cross-model prompt transfer）。

---

## 二、研究地图（5 类方向）

| 方向 | 核心问题 | 代表工作 |
|---|---|---|
| ① 跨模型迁移与现象刻画 | prompt 换模型为什么失效、如何度量 | PromptBridge, MAPO, Zero-Shot Continuous Prompt Transfer, PANDA |
| ② 自动 Prompt 优化 | 给定模型+任务，自动搜出好 prompt | APO, OPRO, ProTeGi, PromptBreeder, EvoPrompting, PromptWizard, TextGrad, PMPO, MIPRO |
| ③ LLM 程序 / 编译范式 | 把 prompt 当程序参数，跨模型编译优化 | DSPy, MIPRO, BetterTogether, Multi-module GRPO |
| ④ Agent / Tool-use 场景 | 多步 agent 的 prompt 优化与跨模型策略选择 | AutoPDL, RePrompt, Role-play APO |
| ⑤ 评测与鲁棒性 | 跨 prompt 变体/跨模型的评测偏差 | PromptRobust, PromptBench, Brittlebench, POSIX, Multi-Prompt Eval, AgentBench |

判别一篇论文是否真正命中你的问题，看三点：是否讨论**跨模型迁移**、是否能**针对不同模型分别优化**、是否覆盖 **agent/tool-use 多步行为**。下面表中"直接相关"一列据此标注。

---

## 三、核心论文表

| 论文 / 项目                                            | 年份        | 核心问题                               | 方法                                                                                                                 | 是否直接支持跨模型适配                                | 与 agent 关系                                      | 链接                                                                                                         |
| -------------------------------------------------- | --------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------ | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **PromptBridge**                                   | 2025      | 模型切换后 prompt 失效（Model Drifting）    | 用少量对齐任务做 MAP-RPE 得到源/目标模型各自最优 prompt 对，学习跨模型 prompt 映射，测试时 zero-shot 把源 prompt 翻译成目标 prompt，training-free          | ✅ 直接（最命中）                                  | 覆盖单 agent 与多 agent（MapCoder 框架，含调试 agent 局部换模型） | [arXiv 2512.01420](https://arxiv.org/abs/2512.01420)                                                       |
| **MAPO** (Model-Adaptive Prompt Optimization)      | 2024      | prompt 应适配"模型"而非仅"任务"              | 用 oracle LLM 生成 ~1000 候选 prompt → 选最优 → 建热身数据集 → SFT + RL(PPO/RRMF) 训练一个为特定 LLM 重写 prompt 的优化器                     | ✅ 直接                                       | 单步任务为主，非 agent                                  | [arXiv 2407.04118](https://arxiv.org/abs/2407.04118)                                                       |
| **AutoPDL**                                        | 2025      | 自动为 agent 选 prompting pattern + 内容 | 把 agent 配置当 AutoML 问题，在 Zero-Shot/CoT/ReAct/ReWOO 等组合空间用 successive halving 搜索；输出可读可改的 PDL 程序；3 任务 × 7 LLM(3B–70B) | ✅ 部分（揭示"最优策略随模型/任务变化"）                     | ✅ 直接（agent prompt 优化）                           | [arXiv 2504.04365](https://arxiv.org/abs/2504.04365)                                                       |
| **DSPy**                                           | 2023      | prompt 当可编译程序参数                    | 声明式模块 + 编译器，针对给定 LM 与指标自动优化 instruction + few-shot；同一程序可编译到 GPT-3.5 / Llama2-13b / T5                              | ✅ 直接（编译时按目标 LM 优化）                         | 支持多跳检索、agent loop                               | [arXiv 2310.03714](https://arxiv.org/abs/2310.03714)                                                       |
| **MIPRO** (DSPy 优化器)                               | 2024      | 多模块 LM 程序联合优化 instruction+demo     | 程序/数据感知的指令提议 + 随机小批评估学习代理模型 + 元优化；Llama-3-8B 上 5/7 程序最优，最高 +13%                                                    | ✅ 间接（按目标模型优化整条程序）                          | 多步管道，agent 适用                                   | [EMNLP 2024 / ACL Anthology](https://aclanthology.org/2024.emnlp-main.525)                                 |
| **BetterTogether** (DSPy)                          | 2024      | 微调与 prompt 优化协同                    | 在同一管道里联合优化模块级 LM 与 prompt 模板，平均 +60%/+6%                                                                           | 间接                                         | 多模块 RAG/agent 管道                                | [arXiv 2407.10930](https://arxiv.org/abs/2407.10930)                                                       |
| **Multi-module GRPO** (DSPy)                       | 2025      | 在线 RL 与 prompt 优化组合于多 prompt 程序    | 模块级分组 GRPO + 自动 prompt 优化，平均 +11%（含分类、多跳、隐私委派）                                                                     | 间接                                         | ✅ 多模块 agent                                     | [ACM 3786335.3813164](https://dl.acm.org/doi/10.1145/3786335.3813164)                                      |
| **OPRO** (Large Language Models as Optimizers)     | 2023      | LLM 当优化器搜指令                        | 用 meta-prompt 带历史 prompt-分数对迭代生成新指令；GSM8K +8%、BBH +50%                                                             | 间接（可针对单模型优化）                               | 非专门 agent                                       | [arXiv 2309.03409](https://arxiv.org/abs/2309.03409)                                                       |
| **APE** (Automatic Prompt Engineer, Zhou et al.)   | 2022      | 用 LLM 生成并筛选指令                      | 把指令搜索建模为 LLM Monte-Carlo 搜索 + 评估筛选；自动 prompt 优化的早期基础                                                               | 间接（针对单模型）                                  | 通用                                              | [arXiv 2210.03429](https://arxiv.org/abs/2210.03429)<br>2211.01910                                         |
| **ProTeGi / APO** (Pryzant et al.)                 | 2023      | 自动"文本梯度下降"改 prompt                 | 用小批数据生成自然语言"梯度"批评 → 反向编辑 prompt + beam search/bandit 选择；最高 +31%（ProTeGi 即此条线）                                      | 间接（可针对单模型优化）                               | 提出 agent 通用前提                                   | [arXiv 2305.03495](https://arxiv.org/abs/2305.03495)                                                       |
| **GrIPS** (Gradient-free Instructed Prompt Search) | 2022      | 无梯度指令搜索                            | 对指令做编辑/删除/增改等操作进行启发式搜索                                                                                             | 间接（针对单模型）                                  | 通用                                              | [arXiv 2203.07281](https://arxiv.org/abs/2203.07281)                                                       |
| **PromptBreeder** / **EvoPrompting**               | 2023–2024 | 进化/遗传式搜 prompt                     | PromptBreeder：自指自改进，变异 task-prompt 种群按 fitness 迭代；domain-specific；EvoPrompting：遗传算法变异                              | 间接（可逐模型进化）                                 | 非专门 agent                                       | [arXiv 2309.16797](https://arxiv.org/abs/2309.16797)                                                       |
| **TextGrad**                                       | 2024      | 把文本反馈当梯度反传优化 prompt                | 计算图 + 文本梯度下降；可用强模型(gpt-4o)作梯度引擎优化弱模型(gpt-3.5)的 system prompt                                                       | ✅ 部分（优化器与目标模型可解耦；测了 prompt 跨模型迁移性，最高 +10%） | 通用，可用于 agent prompt                             | [arXiv 2406.07496](https://arxiv.org/abs/2406.07496)                                                       |
| **PromptWizard**                                   | 2024      | 全自动离散 prompt 优化                    | 反馈驱动 critique+synthesis 自演化                                                                                        | 间接                                         | 通用                                              | [arXiv 2405.18369](https://arxiv.org/abs/2405.18369)                                                       |
| **PMPO**                                           | 2025      | 轻量、跨模型尺寸的 prompt 优化                | 用 token 级 cross-entropy 直接评估定位低质片段并改写，单 forward 选 variant，免输出采样/裁判；BBH 最优，AlpacaEval +19pp                         | ✅ 部分（小/大模型均适用）                             | 通用                                              | [arXiv 2505.16307](https://arxiv.org/abs/2505.16307)                                                       |
| **RePrompt**                                       | 2025      | agent 场景缺 final checker 的 APE      | 用中间反馈而非终局分数做自动 prompt engineering for planning                                                                     | 间接                                         | ✅ 直接（LLM agent）                                 | [arXiv 2406.11132](https://arxiv.org/abs/2406.11132)                                                       |
| **Transfer-Prompting**                             | 2025      | 跨任务 prompt 适配                      | 双阶段：源 prompt 构造 + 目标 prompt 生成；25 个 LLM、9 数据集                                                                      | 间接（跨任务，含多模型）                               | 非专门 agent                                       | [arXiv 2502.14211](https://arxiv.org/abs/2502.14211)                                                       |
| **Zero-Shot Continuous Prompt Transfer**           | 2024      | 连续 prompt 跨模型迁移                    | 把源 prompt 编码到相对空间、搜索目标 prompt                                                                                      | 直接（连续 prompt，非文本指令）                        | 非专门 agent                                       | [arXiv 2310.01691](https://arxiv.org/abs/2310.01691)                                                       |
| **PromptRobust / PromptBench**                     | 2023–2024 | LLM 对 prompt 扰动的鲁棒性评测              | 字符/词/句/语义级对抗 prompt 攻击评测库                                                                                          | 背景鲁棒性                                      | 含 agent 评测                                      | [arXiv 2306.04528](https://arxiv.org/abs/2306.04528), [arXiv 2312.07910](https://arxiv.org/abs/2312.07910) |
| **Brittlebench**                                   | 2026      | 语义保持的 prompt 变体导致性能方差              | 语义保持扰动 → 性能最多掉 12%，单一扰动即可改变 63% 情况下的模型排名；可解释一半性能方差                                                                 | 背景鲁棒性                                      | 通用                                              | [arXiv 2603.13285](https://arxiv.org/abs/2603.13285)                                                       |
| **POSIX / Multi-Prompt Eval**                      | 2024      | 单 prompt 评测不可靠                     | POSIX 用 loglikelihood 变化度量敏感度；多 prompt 估性能分布而非单点                                                                   | 背景鲁棒性                                      | 通用                                              | [arXiv 2410.02185](https://arxiv.org/abs/2410.02185), [arXiv 2401.00595](https://arxiv.org/abs/2401.00595) |
| **AgentBench / AgentBoard**                        | 2023–2024 | 把 LLM 当 agent 跨环境评测                | 多环境交互评测 + 细粒度逐步分析                                                                                                  | 背景评测                                       | ✅ 直接（agent benchmark）                           | [arXiv 2308.03688](https://arxiv.org/abs/2308.03688), [arXiv 2401.13178](https://arxiv.org/abs/2401.13178) |

### 归类

#### 类别一：单模型自动 Prompt 优化

针对**单一目标模型** + 任务，自动搜出/优化 prompt。按是否用梯度或损失信号驱动分为梯度式、非梯度、协同三类。
精读：[[Agent Prompt Auto Opt]]

##### A. 梯度式 / 损失信号驱动（用"文本梯度"或模型损失作为优化信号）

| 论文 | 方法要点 | 信号类型 | 链接 |
|---|---|---|---|
| **TextGrad** | 计算图 + 文本梯度下降，可反向传播反馈优化 prompt | 文本梯度（LLM 反馈当梯度） | [arXiv 2406.07496](https://arxiv.org/abs/2406.07496) |
| **ProTeGi / APO** (Pryzant) | 用小批数据生成自然语言"梯度"批评，反向编辑 prompt + beam search/bandit | 自然语言"梯度"（梯度下降隐喻） | [arXiv 2305.03495](https://arxiv.org/abs/2305.03495) |
| **PMPO** | 用 token 级 cross-entropy 直接定位低质片段并改写，单 forward 选 variant | 真实模型损失（logits/CE） | [arXiv 2505.16307](https://arxiv.org/abs/2505.16307) |

##### B. 非梯度（LLM 反馈 / 进化 / 启发式搜索）

| 论文 | 方法要点 | 搜索机制 | 链接 |
|---|---|---|---|
| **APE** (Zhou) | LLM 生成候选指令 + Monte-Carlo 搜索 + 评分筛选 | LLM 生成 + 打分 | [arXiv 2210.03429](https://arxiv.org/abs/2210.03429) |
| **OPRO** | LLM 当优化器，meta-prompt 带历史 prompt-分数对迭代 | LLM-as-optimizer | [arXiv 2309.03409](https://arxiv.org/abs/2309.03409) |
| **GrIPS** | 对指令做编辑/删除/增改等操作启发式搜索 | 显式 gradient-free 搜索 | [arXiv 2203.07281](https://arxiv.org/abs/2203.07281) |
| **PromptBreeder / EvoPrompting** | 变异 task-prompt 种群按 fitness 迭代 / 遗传算法 | 进化/遗传 | [arXiv 2309.16797](https://arxiv.org/abs/2309.16797) |
| **PromptWizard** | 反馈驱动 critique + synthesis 自演化 | LLM 反馈自演化 | [arXiv 2405.18369](https://arxiv.org/abs/2405.18369) |
| **MIPRO** | 多模块程序联合优化 instruction+demo，小批评估学代理模型 + 元优化 | 随机搜索 + bandit（DSPy 优化器） | [EMNLP 2024](https://aclanthology.org/2024.emnlp-main.525) |

##### C. 协同优化（prompt 优化 + 微调/RL，含真实梯度）

| 论文 | 方法要点 | 备注 | 链接 |
|---|---|---|---|
| **BetterTogether** | 同一管道联合优化模块级 LM 与 prompt 模板 | prompt 优化 + 微调协同 | [arXiv 2407.10930](https://arxiv.org/abs/2407.10930) |
| **Multi-module GRPO** | 模块级分组 GRPO（在线 RL）+ 自动 prompt 优化 | RL policy gradient（DSPy） | [ACM 3786335.3813164](https://dl.acm.org/doi/10.1145/3786335.3813164) |

#### 类别二：跨模型 Prompt 差异 / 迁移 / 适配

这一类才真正命中你"同一 agent 功能换模型需要不同 prompt"的核心问题。

| 论文 | 核心贡献 | 是否 training-free | 链接 |
|---|---|---|---|
| **PromptBridge** | 命名 Model Drifting，学跨模型 prompt 映射，zero-shot 把源 prompt 翻译成目标 prompt；覆盖单/多 agent | 是 | [arXiv 2512.01420](https://arxiv.org/abs/2512.01420) |
| **MAPO** | 主张 prompt 适配"模型"，SFT+RL 训一个为特定 LLM 重写 prompt 的优化器 | 否（需训练） | [arXiv 2407.04118](https://arxiv.org/abs/2407.04118) |
| **Zero-Shot Continuous Prompt Transfer** | 连续 prompt 跨模型迁移，编码到相对空间搜索目标 prompt | 是 | [arXiv 2310.01691](https://arxiv.org/abs/2310.01691) |
| **Transfer-Prompting** | 双阶段跨任务 prompt 适配，25 个 LLM、9 数据集 | 否 | [arXiv 2502.14211](https://arxiv.org/abs/2502.14211) |
| **DSPy** | 编译式，把 prompt 当程序参数，可按目标 LM 编译优化（GPT-3.5/Llama2/T5） | 编译即优化 | [arXiv 2310.03714](https://arxiv.org/abs/2310.03714) |

#### 类别三：Agent 场景特化

多步 agent / tool-use 场景的 prompt 优化与跨模型策略选择。

| 论文 | 核心贡献 | 与跨模型关系 | 链接 |
|---|---|---|---|
| **AutoPDL** | 把 agent 配置当 AutoML，搜 Zero-Shot/CoT/ReAct/ReWOO；揭示最优策略随模型变化 | 直接（同一任务不同模型需不同范式） | [arXiv 2504.04365](https://arxiv.org/abs/2504.04365) |
| **RePrompt** | agent 场景缺 final checker 时用中间反馈做 APE | 间接 | [arXiv 2406.11132](https://arxiv.org/abs/2406.11132) |
| **PromptBridge**（多 agent 部分） | MapCoder 框架下局部换 sub-agent 模型的迁移 | 直接 | [arXiv 2512.01420](https://arxiv.org/abs/2512.01420) |

#### 类别四：评测与鲁棒性（背景）

不直接做适配，但支撑"为什么需要适配"的动机与评测方法。

| 论文 | 核心贡献 | 链接 |
|---|---|---|
| **PromptRobust / PromptBench** | 字符/词/句/语义级对抗 prompt 鲁棒性评测库 | [arXiv 2306.04528](https://arxiv.org/abs/2306.04528), [arXiv 2312.07910](https://arxiv.org/abs/2312.07910) |
| **Brittlebench** | 语义保持扰动可解释一半性能方差，单一扰动即可改变 63% 模型排名 | [arXiv 2603.13285](https://arxiv.org/abs/2603.13285) |
| **POSIX / Multi-Prompt Eval** | 用 loglikelihood 变化度量敏感度；用多 prompt 估性能分布而非单点 | [arXiv 2410.02185](https://arxiv.org/abs/2410.02185), [arXiv 2401.00595](https://arxiv.org/abs/2401.00595) |
| **AgentBench / AgentBoard** | LLM 当 agent 跨环境评测 + 细粒度逐步分析 | [arXiv 2308.03688](https://arxiv.org/abs/2308.03688), [arXiv 2401.13178](https://arxiv.org/abs/2401.13178) |

**类别一的"单模型优化"方法是类别二"跨模型适配"的工具**——TextGrad/DSPy/OPRO 等都可被用来为每个模型各跑一遍优化，PromptBridge 则更进一步直接学"源→目标"的映射免去逐任务重优化。


---

## 四、关键结论与研究缺口

**已确立的结论：**
- prompt 的"模型特异性"是实证事实而非工程臆测：同一 prompt 跨模型/跨版本/跨参数规模都会漂移，MAPO 与 PromptBridge 从不同角度验证了"prompt 应适配模型而非仅适配任务"。
- 自动优化（APE/OPRO/TextGrad/DSPy）能稳定把人工 prompt 提升 5–50%，且优化器与目标模型可解耦（TextGrad 用强模型优化弱模型）。
- DSPy 范式把 prompt 从"手写字符串"升格为"可编译、可按目标 LM 优化的程序参数"，是工程上最系统化的路径。
- AutoPDL 进一步揭示：即便同一任务，不同模型选中的 prompting pattern（CoT vs ReAct vs ReWOO）也不同——所以"同一个 agent 功能"在不同模型上很可能需要不同的**范式**而非仅不同措辞。

**研究缺口（值得你切入）：**
- 真正系统研究"同一 agent 功能 × 多模型 × 维护 prompt family"的工作极少；PromptBridge 是目前最接近的，但偏 code/agent benchmark，未深入多步 tool-use 的轨迹级适配。
- 跨模型 prompt 迁移多为单步任务；agent 多步轨迹下"局部换某个 sub-agent 模型"的适配尚不充分（PromptBridge 仅初步触及 MapCoder 局部替换）。
- 缺少"prompt registry + 版本化 + 回归测试 + 模型路由"的统一工程基准。

---

## 五、工程落地 workflow（建议直接照搬）

1. **定义 agent spec**：把 agent 功能拆成可复现的签名（输入/输出/工具/成功判据），而非自然语言描述。
2. **建 eval set**：覆盖核心场景 + 边缘 case 的 golden 集，指标可自动计算（这是所有自动优化的前提）。
3. **每个模型跑 baseline**：用同一 prompt 在各目标模型上跑，记录 transfer gap（用 PromptBridge 的 Δ 度量）。
4. **自动/半自动优化 prompt**：优先 DSPy（编译式，原生多模型）或 TextGrad（解耦优化器与目标模型）；轻量场景用 OPRO/APO；agent 范式选择用 AutoPDL 思路搜索 CoT/ReAct/ReWOO。
5. **跨模型迁移**：新模型上线时，用 PromptBridge 的 calibration（少量对齐任务 + prompt 对映射）从源 prompt 翻译出目标 prompt，而非从零重优化。
6. **回归测试**：每次模型版本升级/换 API，自动跑 eval set，对比 transfer gap；维护 prompt registry（模型→prompt→指标→版本，支持 diff/rollback）。
7. **可选：模型路由**：当同一功能在便宜模型上靠适配 prompt 已达标，就路由到便宜模型（DSPy 的 router agent 用例即此思路）。

---

## 六、入门阅读顺序（由浅入深）

1. **先读现象与动机**：PromptBridge（Model Drifting + 跨模型迁移，最直接）→ MAPO（模型特异性优化）。
2. **再读自动优化主干**：APO（梯度下降式）→ OPRO（LLM 当优化器）→ TextGrad（文本梯度，强模型优化弱模型）→ PromptBreeder（进化式）。
3. **再读编译范式**：DSPy（核心）→ MIPRO（多模块联合优化）→ BetterTogether（微调+prompt 协同）。
4. **再读 agent 场景**：AutoPDL（agent 配置 AutoML + 策略随模型变化）→ RePrompt（中间反馈式 agent APE）。
5. **最后读评测/鲁棒性**：PromptRobust/PromptBench → Brittlebench → POSIX/Multi-Prompt Eval → AgentBench（理解为何单 prompt 评测不可靠、跨模型排名会变）。

---

## 七、检索关键词（自己继续找用）

`cross-model prompt transfer`、`model-adaptive prompt optimization`、`prompt portability`、`prompt migration`、`agent prompt optimization`、`prompt sensitivity / robustness`、`LLM program compilation`、`instruction optimization`、`model drifting`。

## 八、工程优先级建议

- **最可立即落地**：DSPy（编译式、原生多模型、带 optimizer+eval 闭环）、TextGrad（优化器与目标模型解耦）、AutoPDL（agent 范式随模型变化）。
- **最贴题但较新**：PromptBridge（直接解决跨模型 prompt 迁移），代码可用性需确认（论文称"code will be available soon"）。
- **背景/评测必备**：PromptBench、Brittlebench、POSIX（理解为何单 prompt 评测不可靠、跨模型排名会变）。

## 九、一句话总结

这个方向没有一个统一成熟的名字，相关工作散落在 **prompt transferability / model-specific prompt optimization / LLM program compilation / agent prompt optimization / prompt robustness** 五条线里。**PromptBridge + MAPO** 是"跨模型适配"最直接的锚点，**DSPy/TextGrad** 是工程落地的主路径，**AutoPDL** 把问题推广到"同一 agent 功能在不同模型上需要不同 prompting 范式"。建议从 PromptBridge 入手，再用 DSPy 搭建可自动优化+回归的 prompt registry。
