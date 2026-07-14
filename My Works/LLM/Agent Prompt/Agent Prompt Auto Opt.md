# 单模型自动 Prompt 优化：梯度式 vs 非梯度方法深度解读

> 范围：给定一个目标 LLM + 任务 + eval set，自动把 prompt 优化得更好（不微调模型权重）。
> 主线问题：优化器如何"知道该改什么、往哪个方向改"——这就是梯度式与非梯度的分界。

---

## 0. 先厘清：这里的"梯度"不是真梯度

这些方法都把 prompt 当作**离散字符串变量**优化，而 LLM 不可对 prompt 求导。所以"梯度式"指的是**用一个方向性的损失/批评信号来定位"该改什么、往哪个语义方向改"**：

- **梯度式**：构造一个"梯度"信号——要么是自然语言批评（TextGrad / ProTeGi-APO），要么是真实的 token 级交叉熵损失（PMPO）——然后沿"反方向"修订 prompt。类比 gradient descent。
- **非梯度**：不构造方向信号，纯靠**生成→评估→选择**的黑盒优化（搜索 / 进化 / LLM-as-optimizer）。

判别一句话：**优化过程中是否有一个"告诉模型哪里错了、该怎么改"的中间方向信号**。有 → 梯度式；无（只有分数）→ 非梯度。

---

## 一、梯度式方法

### 1.1 ProTeGi / APO — 自然语言"梯度下降"+ Beam Search + Bandit

**论文**：Automatic Prompt Optimization with "Gradient Descent" and Beam Search（Pryzant et al., EMNLP 2023, [arXiv 2305.03495](https://arxiv.org/abs/2305.03495)）

**核心思想**：把数值梯度下降搬到离散 prompt 上——用小批数据的错误生成"自然语言梯度"（对 prompt 缺陷的总结），再让 LLM 沿"梯度的反方向"编辑 prompt。

**算法步骤**：
1. 用当前 prompt 在一小批数据上跑，收集错误样本（按 4 个一组采样）。
2. **生成梯度**：把错误喂给 LLM，要求"给出若干条当前 prompt 可能犯错的理由"，这些理由就是"自然语言梯度"。固定模板：
   > I'm trying to write a zero-shot classifier prompt. My current prompt is: "{prompt}" But this prompt gets the following examples wrong: {error_string} give {num_feedbacks} reasons why the prompt could have gotten these wrong. Wrap each reason with `<START>` and `<END>`
3. **反向传播（编辑）**：把梯度喂给另一个 prompt 模板，要求 LLM 沿"梯度的反语义方向"改写出若干个改进版 prompt：
   > Based on these examples the problem with this prompt is that {gradient} ... I wrote {steps_per_gradient} different improved prompts.
4. **局部 Monte Carlo 扩展**：对每个改进版用"保持语义的改写"再生成若干变体，扩大邻域候选。
5. **Bandit 选择**：把候选 prompt 当作多臂赌博机的"臂"，每次拉臂 = 在一个随机数据点上评估，用 UCB / UCB-E / Successive Rejects / Successive Halving 做 best-arm 识别，保留最优候选进入下一轮。
6. 外层是 **beam search**，重复 6 步优化。文中实验每步采 4 个错误、每个父节点生成 8 个后继候选。

**关键点**：梯度 = LLM 对"prompt 为什么错"的总结；反传 = LLM 沿反方向改写；搜索效率靠 bandit。ProTeGi 自身核心走的是"自然语言梯度 → LLM 沿反方向改写 prompt → paraphrase 做局部 Monte Carlo 扩展"；而 add / paraphrase / swap / delete 这类短语级编辑操作更多是 GrIPS 等编辑式方法的搜索空间描述，并非 ProTeGi 的核心算子。

**结果**：4 个分类任务（Jailbreak/Ethos/Liar/Sarcasm），相比原始 prompt 最高 +31%，相比 MC 基线 +3.9%、相比 RL 基线 +8.2%，超过 SOTA prompt-learning 基线 4–8%。约第 3 步达到峰值，之后会过拟合或陷入局部最优。

**优点**：可解释（梯度是一段批评文字）、API 友好、不微调。
**局限**：梯度质量依赖 LLM 自我批评能力；少量步即过拟合；纯分类任务为主，多步 agent 适配未验证。

---

### 1.2 TextGrad — 文本反向传播 + PyTorch 式 autograd

**论文**：TextGrad: Automatic "Differentiation" via Text（Yuksekgonul et al., NeurIPS 2024, [arXiv 2406.07496](https://arxiv.org/abs/2406.07496)）

**核心思想**：把"复合 AI 系统"建成计算图，变量是文本（prompt / 代码 / 分子），LLM 提供的自然语言批评当作"文本梯度"，像 PyTorch autograd 一样反向传播、聚合、更新变量。这是 APO 思想的"系统化、通用化、可复合"升级版。

**形式化（论文公式）**：
- 计算图：每个变量 \(v\) 由其前驱生成，\(v = f_v(\text{PredecessorsOf}(v))\)（Eq. 10），\(f_v\) 可以是 LLM 调用、模拟器或数值求解器。
- **文本梯度反向传播规则**：变量 \(v\) 的梯度 = 聚合其所有后继 \(w\) 传回来的反馈（Eq. 11）：
  \[
  \frac{\partial \mathcal{L}}{\partial v}=\bigcup_{w\in\text{SuccessorsOf}(v)}\nabla_f\!\left(v,w,\frac{\partial\mathcal{L}}{\partial w}\right)
  \]
  即"在变量 v 被使用的每一处收集反馈并聚合"，最终梯度是一个"上下文+批评的集合"。
- **更新**：\(v_{\text{new}}=\text{TGD.step}\!\left(v,\frac{\partial\mathcal{L}}{\partial v}\right)\)（Eq. 12，TGD = Textual Gradient Descent optimizer，可带 momentum，可看到历史版本）。
- **成本**：每轮优化最多 \(n\) 次 LLM 调用（计算图每条边一次梯度调用）。
- **Loss 也是文本**：Loss 由 LLM 用自然语言评估输出，例如
  \[
  \text{Loss}(\text{code},\text{goal})=\text{LLM}(\text{Here is a code snippet:}\{code\}.\ \text{Evaluate...})
  \]
  也可以是程序化的评估器（Affinity/Druglikeness 等）。
- 支持 minibatch SGD（`tg.sum` 聚合 batch 损失）和自然语言约束。

**与跨模型适配最相关的一点（重点）**：TextGrad 把"提供梯度的模型"和"执行任务的目标模型"解耦——用**强模型 gpt-4o 当梯度引擎**去优化**弱模型 gpt-3.5-turbo 的 prompt**。论文 Section 3.3：前向推理用 gpt-3.5-turbo-0125，反馈用 gpt-4o；batch size 3、12 轮、共 36 条训练样本有放回采样，每轮跑验证集，性能提升才更新 prompt。结果（instruction-only, 0 demo）：Object Counting 91.9%、Word Sorting 79.8%、GSM8k 81.1%（叠加 DSPy 选的 demo 后到 82.1%）。这正是"用强模型生成适配弱模型的 prompt"的范式。

**结果**：GPT-4o 在 GPQA 零样本 51%→55%；LeetCode-Hard 代码解相对 +20%；还可优化分子、放疗方案。

**优点**：通用计算图、可复合、优化器与目标模型可解耦、PyTorch 语法易用。
**局限**：每轮 LLM 调用数随图边数线性增长，成本高；梯度仍是 LLM 批评，质量受限于批评模型。

---

### 1.3 PMPO — 用真实损失当信号，单次 forward 选 variant

**论文**：PMPO: Probabilistic Metric Prompt Optimization（Zhao et al., 2025, [arXiv 2505.16307](https://arxiv.org/abs/2505.16307)）

**核心思想**：前两篇的"梯度"是 LLM 批评（仍要采样输出、要裁判）。PMPO 直接用**目标模型自身的 token 级交叉熵损失**作为评估信号，单次 forward 就能在候选 variant 里选最优，**免输出采样、免 LLM 裁判**，对小模型/非指令微调模型尤其友好。

**算法步骤**：
1. **定位低质片段**：用 **masking-based analysis** 逐段掩蔽 prompt 的 token 子集，观察哪段被掩蔽后损失变化最大，定位"低质量"prompt 片段（具体的掩码调度/是否用素数等细节需以原文 PDF 为准，此处不展开）。
2. **改写提议**：对定位到的低质片段用标准生成（这一步才用 LLM 生成）改写出若干 variant。
3. **单 forward 选择**：对所有 variant 在单次 forward pass 内计算 token 级 CE 损失，**取损失最小者**作为采纳的 prompt——不需要采样完整输出，也不需要人或 LLM 打分。
4. 迭代定位→改写→选择。

**关键点**：梯度 = **真实可计算的 token 级交叉熵损失**（这是三篇里唯一用到"真损失"而非"文本批评"的）。支持有监督任务（对 gold 算 CE）和偏好任务。

**结果**：BBH 平均准确率最优；GSM8K / AQUA-RAT 表现强；AlpacaEval 2.0 胜率 +19pp。

**优点**：轻量、单 forward、不依赖裁判模型、跨模型尺寸稳健（小模型也能用，因为它不要求模型擅长自我批评）。
**局限**：损失信号反映的是"似然"而非"任务正确性"，对开放式生成任务可能不完全对齐；需要 gold 答案或偏好数据来定义损失。

### 梯度式三篇对比

| 维度 | ProTeGi/APO | TextGrad | PMPO |
|---|---|---|---|
| "梯度"信号 | LLM 对错误的语言批评 | LLM 批评，沿计算图反向聚合 | 真实 token 级交叉熵损失 |
| 反向传播 | LLM 沿反方向编辑 prompt | 通用计算图 autograd | masking 定位 + 损失最小化 |
| 选择机制 | beam search + bandit | TGD optimizer + 验证集 | 单 forward 取最小损失 |
| 优化器与目标模型是否解耦 | 否（同一 LLM 自批评自改） | **是**（强模型评、弱模型用） | 否（用目标模型自身损失） |
| 是否需采样输出/裁判 | 需 | 需 | **不需** |
| 跨模型适配潜力 | 弱（针对单模型优化） | **强**（解耦，可跨模型） | 中（小/大模型通用，但不直接迁移） |

---

## 二、非梯度方法

共性：**不构造方向信号，只生成候选 + 用分数做选择**，本质是黑盒优化（搜索/进化）。LLM 的角色是"生成器"而非"批评者"。

### 2.1 APE — LLM 逆向归纳指令 + Monte Carlo 搜索

**论文**：Large Language Models Are Human-Level Prompt Engineers（Zhou et al., ICLR 2023, [arXiv 2211.01910](https://arxiv.org/abs/2211.01910)）

**核心思想**：把 prompt 工程建模为**自然语言程序合成**——只给 LLM 一批输入-输出样例（不给指令），让它"反推"出指令候选，再用目标模型执行候选并按分数筛选。这是自动 prompt 优化的开山之作。

**算法步骤**：
1. **指令生成（逆向/inverse）**：用诱导模板让 LLM 从样例反推指令：
   > I gave a friend an instruction and some advice. Here are some examples. The instruction was: ...
   生成一批候选指令。
2. **递归搜索**：可基于语义相似度做递归式扩展，进一步扩大候选池。
3. **Monte Carlo 搜索 + 评估筛选**：把候选指令放到目标模型上执行（零样本），用所选分数函数（如对数似然或准确率）打分，选分数最高的指令。
4. 选出的指令即可零样本驱动模型。

**结果**：24 个 Instruction Induction 任务上，InstructGPT + APE 的 IQM 0.810，**超过人类手写 prompt 的 0.749**；17/21 BIG-Bench 任务与人类相当或更好。还发现了比经典 "Let's think step by step" 更好的零样本 CoT 提示："Let's work this out in a step by step way to be sure we have the right answer."（MultiArith 78.7→82.0、GSM8K 40.7→43.0）。

**优点**：无需种子 prompt（从样例归纳）；LLM 既当生成器又当评估器；零样本即可用。
**局限**：候选质量靠生成模型；评估需跑目标模型、成本随候选数线性增长；搜索是"选最好的现成候选"，不像梯度式能"定向修订"。

---

### 2.2 OPRO — LLM 当优化器，meta-prompt 维护优化轨迹

**论文**：Large Language Models as Optimizers（Yang et al., 2023, [arXiv 2309.03409](https://arxiv.org/abs/2309.03409)）

**核心思想**：让 LLM 本身充当优化器——把"过去试过的 prompt 及其分数"作为优化轨迹喂进去，让它生成"更好的新 prompt"。LLM 通过识别高分 prompt 的模式来改进。

**算法步骤**：
1. 构造 **meta-prompt**，含两部分：(a) 任务描述 + 优化目标 + 元指令（如"生成准确率更高的指令"）；(b) **优化轨迹**——历史 prompt 及其训练分数，按分数升序排列，仅保留最好的 20 条；外加少量训练样本（每步随机 3 个）。
2. 每步用 optimizer LLM 生成 8 条候选指令（温度默认 1.0 平衡探索/利用）。
3. 用 scorer LLM（温度 0）在训练子集（GSM8K 用 3.5%、BBH 用 20%）上评估每条候选，算训练准确率。
4. 把新候选+分数加入轨迹，重复。
5. 收敛条件：无法提出更高分 prompt 或达到最大步数（默认 200 步）。

**关键点**：optimizer 与 scorer **可以不同模型**（论文用了 PaLM 2-L、PaLM 2-L-IT、text-bison、gpt-3.5、gpt-4 作 optimizer；pretrained PaLM 2-L / text-bison 作 scorer）——这也是一种优化器/目标模型解耦。不同 optimizer 产出风格不同（PaLM 偏简洁、GPT 偏冗长）。

**结果**：GSM8K 上最佳 prompt "Take a deep breath and work on this problem step-by-step." 达 80.2%（人类 "Let's think step by step" 71.8%，空串 34.0%），+8%；BBH 上比人类 prompt 最多 +50%。

**优点**：纯黑盒、实现简单、optimizer/scorer 可解耦；优化轨迹可解释。
**局限**：每步要跑目标模型评估 8 个候选，成本高；轨迹长度受上下文限制（只留 20 条）；可能卡在同一条 prompt 上数十步。

---

### 2.3 GrIPS — 无梯度、基于编辑的爬山搜索

**论文**：GrIPS: Gradient-free, Edit-based Instruction Search for Prompting Large Language Models（Prasad et al., EACL 2023, [arXiv 2203.07281](https://arxiv.org/abs/2203.07281)）

**核心思想**：对人类写的指令做**编辑操作**生成邻域变体，用验证集做选择——只保留有提升的变体，丢弃退步的。它是 APE 的"有种子 prompt、做局部编辑改进"对应物，明确主张 gradient-free、API 友好。

**算法步骤**（高层；具体编辑算子与选择过程以原文为准）：
1. 以人类指令为种子。
2. 对指令施加随机编辑操作（删除/添加/替换短语、改写等）生成一批变体邻域。
3. 在验证集上评估变体。
4. 采用拒绝式选择：接受有提升的变体作为新种子，丢弃退步的，迭代改进。
5. 终止后返回最佳编辑后的 prompt。

**结果**：InstructGPT 在 Natural Instructions 的 8 个分类任务平均 +4.30pp；OPT/BLOOM/FLAN-T5 类似提升；优于人工改写和纯 example-based prompt，与部分梯度式软 prompt 微调相当。

**优点**：API 友好、计算轻、保留人类种子的可读性。
**局限**：爬山易陷局部最优；编辑可能让指令变得"不通顺但仍提升准确率"（论文自己也观察到）；提升幅度较小。

---

### 2.4 PromptBreeder — 遗传算法，task-prompt 与 mutation-prompt 协同进化

**论文**：PromptBreeder: Self-Referential Self-Improvement Through Reproduction（Fernando et al., 2023, [arXiv 2309.16797](https://arxiv.org/abs/2309.16797)）

**核心思想**：把 prompt 优化做成**遗传算法**，种群由"task-prompt + mutation-prompt"配对组成。LLM 当变异算子（\(P'=\text{LLM}(M+P)\)），不仅进化 task-prompt，还**进化"如何变异 prompt 的 mutation-prompt 本身"**——这就是"自指自改进"。

**算法步骤**：
1. **初始化种群**（50 个 unit）：把"随机 mutation-prompt + 随机 thinking-style + 任务描述"拼接送 LLM 生成初始 task-prompt；每个 unit 还携带自己的 mutation-prompt 和（少样本时）正确解题过程。
2. **fitness 评估**：随机抽 100 个训练 Q&A 对算准确率。
3. **变异**：从 9 个算子里随机选 1 个施加。9 个算子分 5 类：
   - 直接变异：zero-order（"A list of 100 hints:" 重新生成）、first-order（标准无性变异 \(P'=\text{LLM}(M+P)\)）。
   - 分布估计变异：EDA（把当前种群的 prompt 列表喂 LLM 续写）、EDA Rank/Index、Lineage（用历史精英 lineage）。
   - **超变异（hypermutation）**：变异 mutation-prompt 本身（zero/first-order），即"改进改进方式"。
   - 拉马克变异：从正确解题过程反推 task-prompt（类似 APE 的指令归纳）。
   - 交叉 + context shuffle：task-prompt 交叉（10% 概率，按 fitness 比例选）、少样本 context 重排。
4. **二元锦标赛选择**：随机抽两个 unit，取 fitness 高者变异后覆盖输者。
5. 跑 20–30 代（约 1–2k 次 fitness 评估）。
6. **多样性维护**：随机字符串前缀、基于 BERT 相似度的 fitness sharing、可变温度（1.0–2.0 起并变异）。

**自指性**：去掉任何一个自指算子都几乎普遍有害；初始化阶段对 task-prompt 的"再描述"贡献最大。

**结果**：GSM8K 零样本 83.9%（OPRO 80.2%、PS+ 59.3%），SVAMP 90.2%，CommonsenseQA 85.4%；ETHOS 89%（人类 80%）；24 个指令归纳任务中 21/24 超过 APE。

**优点**：探索能力强、能发现非常规 prompt、自指机制可改进"改进方式"本身。
**局限**：计算开销大（1–2k 评估）；进化的 prompt 有时语义诡异（如 GSM8K 最优 prompt 是 "SOLUTION""）；只优化 prompt 内容、不改变 prompting 拓扑。

---

### 2.5 PromptWizard — 批判+合成的自演化（简述）

**论文**：PromptWizard（2024, [arXiv 2405.18369](https://arxiv.org/abs/2405.18369)）

**核心思想**：全自动离散 prompt 优化，用反馈驱动的 critique + synthesis 自演化迭代改写 prompt。介于 OPRO（轨迹驱动）与 ProTeGi（批评驱动）之间，强调"自我批评—自我改写"的闭环，无需人工干预。

**定位**：可看作"批评式"非梯度方法，与 ProTeGi 的区别在于它不显式构造 bandit/beam，而走 critique-synthesize 循环；与 OPRO 的区别在于它用对失败案例的批评而非仅靠分数轨迹。

---

## 三、两类方法的本质对比与选择建议

| 维度 | 梯度式（ProTeGi / TextGrad / PMPO） | 非梯度（APE / OPRO / GrIPS / PromptBreeder / PromptWizard） |
|---|---|---|
| 优化信号 | 方向性：语言批评或真实损失，告诉你"哪错、往哪改" | 标量分数，只告诉你"好不好" |
| LLM 角色 | 既是生成器也是批评器（批评=梯度） | 主要是生成器（评估用打分函数） |
| 搜索方式 | 定向修订 + 选择 | 生成 + 评估 + 选择（黑盒优化/进化） |
| 收敛效率 | 通常更高效（有方向） | 需更多评估次数（探索开销） |
| 可解释性 | 梯度是一段批评，可读 | 轨迹/种群，可读但无"方向" |
| 对模型自我批评能力的依赖 | 高（除 PMPO 用真损失外） | 低（只需打分） |
| 跨模型适配潜力 | TextGrad 可解耦优化器/目标模型 | OPRO 可解耦 optimizer/scorer |

**工程选择建议**：
- 有 gold 答案/可计算损失 + 小模型/非指令模型 → **PMPO**（轻量、单 forward）。
- 想用强模型帮弱模型优化 prompt、或优化复合 AI 系统的多个 prompt → **TextGrad**（解耦 + 计算图）。
- 有种子 prompt + 想低成本局部改进 → **ProTeGi/APO** 或 **GrIPS**。
- 没有种子 prompt、只有样例 → **APE** 归纳。
- 想纯黑盒、optimizer/scorer 解耦、轨迹可解释 → **OPRO**。
- 探索空间大、想要非常规 prompt、可承受计算开销 → **PromptBreeder**。

**回到你的核心问题（跨模型适配）**：单模型优化方法本身不是为跨模型迁移设计的，但它们是"为每个模型各跑一遍优化"的工具；其中 **TextGrad 和 OPRO 因优化器与目标模型可解耦**，最接近"用一份通用优化器适配多个目标模型"的范式，是从"单模型优化"过渡到"跨模型适配"的天然桥梁（与 PromptBridge 的直接迁移路线互补）。
