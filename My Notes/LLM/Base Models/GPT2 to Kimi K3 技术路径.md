> 从 GPT-2（2019，decoder-only Transformer 的奠基范式）到 Kimi K3（2026-07，2.8T 参数、KDA 线性注意力、原生 1M 上下文的开源前沿模型），中间横跨 7 年、大约 10 个关键技术拐点。这篇笔记按"综述"的写法，逐个拆解每一步**解决了什么问题、怎么解决的**，最后落到 Kimi 自己的完整谱系（k0-math → k1 → k1.5 → K2 → K2.5 → K2.6 → K3）。

## 目录

1. [起点：GPT-2 (2019) 的架构基线](#起点gpt-2-2019-的架构基线)
2. [架构演化时间线（GPT-2 → 现代开源前沿模型）](#架构演化时间线gpt-2--现代开源前沿模型)
   - [2.1 GPT-3：Scaling Law 与涌现能力](#21-gpt-3scaling-law-与涌现能力)
   - [2.2 RoPE：从"学出来的绝对位置"到"旋转出来的相对位置"](#22-rope从学出来的绝对位置到旋转出来的相对位置)
   - [2.3 RMSNorm + SwiGLU：LLaMA 定型的"新标配"](#23-rmsnorm--swiglullama-定型的新标配)
   - [2.4 GQA/MQA：给 KV Cache 做减法](#24-gqamqa给-kv-cache-做减法)
   - [2.5 MoE：把参数量和计算量解耦](#25-moe把参数量和计算量解耦)
   - [2.6 MLA：KV Cache 的"降维打击"](#26-mlakv-cache-的降维打击)
   - [2.7 RLHF → RLVR：从"对齐人类偏好"到"用可验证奖励练推理"](#27-rlhf--rlvr从对齐人类偏好到用可验证奖励练推理)
   - [2.8 长上下文与线性注意力：逃离 O(n²)](#28-长上下文与线性注意力逃离on)
   - [2.9 Agentic RL：奖励从"答案对不对"变成"任务做完没"](#29-agentic-rl奖励从答案对不对变成任务做完没)
3. [Kimi 全谱系深度解析](#kimi-全谱系深度解析)
   - [3.1 k0-math → k1 → k1.5：Kimi 的"推理三级跳"](#31-k0-math--k1--k15kimi-的推理三级跳)
   - [3.2 K2：万亿参数 MoE + Agentic Intelligence](#32-k2万亿参数-moe--agentic-intelligence)
   - [3.3 K2 Thinking → K2.5 → K2.6 → K2.7-Code：原生量化与多智能体](#33-k2-thinking--k25--k26--k27-code原生量化与多智能体)
   - [3.4 K3：2.8T 参数、KDA、原生 1M 上下文](#34-k32.8t-参数kda原生-1m-上下文)
4. [设计哲学的连续性](#设计哲学的连续性)
5. [全维度对比表：GPT-2 vs Kimi K3](#全维度对比表gpt-2-vs-kimi-k3)
6. [Sources](#sources)

---

## 起点：GPT-2 (2019) 的架构基线

GPT-2（OpenAI, 2019）是这条演化路径的公共祖先，后面几乎每一步演化都是在"替换掉 GPT-2 里的某个具体设计"：

| 组件 | GPT-2 的做法 | 后来被什么取代 / 为什么 |
|---|---|---|
| 归一化 | LayerNorm（Pre-LN，放在每个子块输入前） | RMSNorm（LLaMA 起）——去掉均值中心化，只做 RMS 缩放，更省计算，效果基本不掉 |
| FFN 激活 | GELU（标准两层 MLP） | SwiGLU（LLaMA 起）——门控线性单元，同 FLOPs 下质量更高 |
| 位置编码 | 学习到的绝对位置 embedding，和 token embedding 相加 | RoPE（LLaMA/GPT-NeoX 起）——旋转式相对位置编码，外推性更好 |
| 注意力 | 标准多头自注意力（MHA），每层 causal mask | GQA/MQA（推理效率）→ MLA（DeepSeek-V2 起，KV cache 压缩）→ KDA（Kimi K3，线性注意力） |
| 参数密度 | 稠密（Dense），每个 token 走全部参数 | MoE（Mixtral/DeepSeekMoE/Kimi K2 起）——稀疏激活，参数量和计算量解耦 |
| 训练目标 | 纯 next-token prediction（无监督） | + RLHF → RLVR（可验证奖励强化学习）—— 训练目标从"预测下一个词"扩展到"用奖励塑造推理/行为" |
| 分词器 | Byte-level BPE，词表 ~50K | 现代模型词表普遍拉大到 100K+（Kimi K2/K3 为 160K），覆盖多语言 |
| 规模 | 最大 1.5B（GPT-2 XL） | 一路推到万亿级（Kimi K2 1.04T，K3 2.8T） |

GPT-2 本身的关键贡献不是架构创新，而是**证明了"纯 decoder-only + 大规模无监督预训练"这条路能持续 scale**——后面所有演化都是在这个骨架上做局部替换，而不是推翻重来。

## 架构演化时间线（GPT-2 → 现代开源前沿模型）

### 2.1 GPT-3：Scaling Law 与涌现能力

GPT-3（2020）架构上和 GPT-2 差异很小（主要是交替使用稠密/局部带状稀疏注意力），真正的贡献是**用 175B 参数验证了 scaling law**（Kaplan et al. 2020）——loss 随参数量、数据量、计算量按幂律下降，且首次展示出强 few-shot/in-context learning 能力（不需要梯度更新就能"现场学会"新任务）。这确立了后续十年"堆规模"这条主线的合理性，也直接催生了 Chinchilla（Hoffmann et al. 2022）对"参数量 vs 数据量"最优配比的修正。

### 2.2 RoPE：从"学出来的绝对位置"到"旋转出来的相对位置"

RoPE（Su et al., *RoFormer*, 2021）把位置信息通过旋转矩阵直接编码进 Q/K 向量的内积计算里，而不是像 GPT-2 那样用一个独立的、长度固定的位置 embedding 表。好处：
- **天然表达相对位置**（两个 token 的注意力分数只依赖它们的相对距离，不依赖绝对位置）；
- **更容易外推到训练时没见过的长度**（配合后续 YaRN/NTK-aware scaling 等插值技巧，成为长上下文扩展的基础）。

RoPE 从 GPT-NeoX、LLaMA 开始成为事实标准，Kimi 系列（k1.5 起）也一直沿用其变体，直到 K3 里被部分替换为 KDA（见 2.8 节）。

### 2.3 RMSNorm + SwiGLU：LLaMA 定型的"新标配"

LLaMA（Touvron et al., 2023）把这两个改动打包定型，后续几乎所有开源大模型（Mistral、DeepSeek、Qwen、Kimi）都直接照搬：
- **RMSNorm**（Zhang & Sennrich, 2019）：LayerNorm 去掉均值中心化步骤，只按均方根缩放，少一次统计量计算，速度更快，效果几乎无损；
- **SwiGLU**（Shazeer, 2020）：FFN 从"Linear → GELU → Linear"换成门控结构（一路过 Swish 激活做门，另一路直接线性变换，两者相乘），同等参数量/FLOPs 下语言建模质量更好。

### 2.4 GQA/MQA：给 KV Cache 做减法

标准 MHA 推理时，每个注意力头都要各自缓存一份 K/V，上下文越长、模型越大，KV cache 显存开销越离谱。
- **MQA**（Multi-Query Attention, Shazeer, 2019）：所有 Q 头共享同一份 K/V——显存降到极致，但质量会掉一些；
- **GQA**（Grouped-Query Attention, Ainslie et al., 2023）：折中方案，把 Q 头分成若干组，组内共享一份 K/V——LLaMA-2 70B、Mistral 等广泛采用，是"质量 vs 显存"的标准折中点。

这一步的动机很直接：**推理时的 KV cache 显存，而不是训练时的算力，才是大模型服务成本的真正瓶颈**——后面 MLA、KDA 都是沿着这条线继续深挖。

### 2.5 MoE：把参数量和计算量解耦

Mixture-of-Experts 的核心思路：FFN 层不再是一个统一的大矩阵，而是拆成 N 个"专家"（更小的 FFN），每个 token 经过一个门控网络（router）后只被送进其中 K 个专家（K ≪ N）。这样**总参数量可以远大于单次推理实际用到的参数量（激活参数）**。

- **Mixtral 8x7B**（Mistral AI, 2023）：8 个专家，每 token 激活 top-2，让开源社区第一次大规模用上 MoE；
- **DeepSeekMoE**（DeepSeek, 2024）：专家做得更细粒度（更多、更小的专家），并引入"共享专家"（所有 token 都过的公共专家，负责通用知识）+ 辅助损失更精细的负载均衡；
- **稀疏度 scaling law**：Kimi K2 的技术报告显示，固定激活参数量（即固定 FLOPs）的前提下，专家总数越多（稀疏度越高），validation loss 越低——这条经验规律直接指导了 K2（384 专家/8 激活）和 K3（896 专家/16 激活）的专家数选择。

### 2.6 MLA：KV Cache 的"降维打击"

**Multi-head Latent Attention**（DeepSeek-V2, 2024）是对 2.4 节问题的更彻底解法：不是"少存几份 K/V"（GQA 的思路），而是把每个 token 的 K/V 先压缩进一个**低秩的隐向量（latent vector）**，推理时缓存的是这个小很多的隐向量，需要时再投影展开成各个头的 K/V。相比 GQA，MLA 在同等（甚至更小）KV cache 开销下能保留更接近标准 MHA 的质量。DeepSeek-V3 和 Kimi K2 都采用 MLA 作为注意力机制（K2 的架构报告里直接写"the architecture follows a similar design to DeepSeek-V3, employing Multi-head Latent Attention (MLA)"）。

### 2.7 RLHF → RLVR：从"对齐人类偏好"到"用可验证奖励练推理"

- **RLHF**（InstructGPT, OpenAI, 2022）：用人类偏好数据训练一个奖励模型，再用 PPO 优化策略模型——目标是让模型"说人话、听指令"，属于对齐范畴；
- **RLVR（Reinforcement Learning from Verifiable Rewards）**：目标变成"练推理能力"——奖励不再来自主观人类偏好，而是来自**客观可验证的正确性**（数学题答案对不对、代码能不能跑通测试）。代表作是 OpenAI o1（2024，隐藏 CoT + RL）和 DeepSeek-R1（2025，GRPO，R1-Zero 甚至跳过 SFT 冷启动纯靠 RL 涌现推理）。
- Kimi 的 k1/k1.5 正是踩在这条线上——k1.5 的技术报告明确把自己定位为"用 RL 打开继续 scaling 的新维度"，而且做法比 o1/R1 更"简化"：不用 MCTS、不用价值函数、不用过程奖励模型，只靠长上下文 + 改进的策略优化（在线镜像下降变体）+ 长度惩罚，就在 AIME/Codeforces 等基准上追平 o1。

### 2.8 长上下文与线性注意力：逃离 O(n²)

标准自注意力的计算和显存开销随序列长度**平方增长**，上下文拉到百万 token 级别时几乎不可行。演化路径：
1. **位置插值/YaRN/NTK-aware scaling**：不改架构，只在推理/微调时"欺骗"RoPE 让它适应比训练时更长的序列；
2. **线性注意力 / 状态空间模型混合**：引入固定大小的"递归状态"，把 KV cache 从随序列长度线性膨胀变成常量级——这条线最终在 Kimi K3 里落地为 **KDA（Kimi Delta Attention）**：K3 采用 3:1 的层级混合布局，每 4 层里 3 层用 KDA 线性注意力做快速长序列扫描，1 层保留 Gated MLA 做全局精确检索，在 100 万 token 上下文下解码速度最高提升 6.3 倍。

### 2.9 Agentic RL：奖励从"答案对不对"变成"任务做完没"

RLVR 解决的是单轮"给定问题、给出答案"的正确性问题；**Agentic RL** 把奖励信号扩展到**多轮、有工具调用、有真实/仿真环境交互**的任务完成度上。这是 Kimi K2 开始的核心训练范式转变：K2 的后训练包含"大规模智能体数据合成 pipeline + 联合 RL 阶段"，模型通过和真实/合成环境交互来提升能力，而不只是被动做题。K3 进一步把这条线做成基础设施——**AgentEnv**（与 KVCache.ai 合作开发的沙箱系统，支持快速快照/恢复/fork，专门服务大规模并行 Agent 训练环境）。

---

## Kimi 全谱系深度解析

Kimi（月之暗面 / Moonshot AI）的模型发布节奏极快，一年多内连续推了近十个版本，整条线始终围绕同一个技术主张：**长文本、深推理、强智能体**。按时间顺序梳理：

| 版本 | 时间 | 定位 | 关键技术 |
|---|---|---|---|
| k0-math | 2024-11 | 首个数学推理模型 | RL 用于数学推理的早期尝试 |
| k1 | 2024-12 | 视觉思考模型 | 多模态 + 推理 |
| **k1.5** | 2025-01 | 多模态思考模型，对标 o1 | 长上下文 RL（128k）、简化策略优化、long2short 蒸馏 |
| **K2** | 2025-07 | 万亿参数 MoE，Agentic Intelligence | MuonClip 优化器、MLA、384 专家、大规模 agentic RL |
| K2 Thinking | 2025-11 | K2 的推理增强版 | 原生 INT4 量化感知训练（QAT）首次引入 |
| **K2.5** | 2026-01 | 原生多模态 Agentic，Agent Swarm | K2-Base 继续预训练 ~15T 视觉+文本混合 token；自主多智能体协同 |
| K2.6 | 2026-04 | 原生多模态智能体 | 原生 INT4（从预训练阶段就考虑量化约束） |
| K2.7-Code | 2026-06 | 编程专精版（沿用 K2.6 架构） | 同架构，编程场景微调 |
| **K3** | 2026-07 | 全球首个开放 3T-class 模型 | KDA、Attention Residuals、Stable LatentMoE、Per-Head Muon、MXFP4 QAT |

### 3.1 k0-math → k1 → k1.5：Kimi 的"推理三级跳"

三个月内连续三次迭代（2024-11 → 2024-12 → 2025-01），路径很清晰：先在数学这个"可验证性最强"的领域验证 RL 训练推理的可行性（k0-math）→ 扩展到视觉模态的思考能力（k1）→ 最终整合成同时具备长 CoT 推理和多模态理解的完整模型（k1.5）。

k1.5 的技术报告（arXiv 2501.12599）里最值得记的两个工程贡献：
- **Partial Rollouts（部分展开）**：训练 128k 长上下文的 RL 极其昂贵，Partial Rollouts 让新的训练轨迹可以复用之前轨迹的一大段前缀，避免每次都从头生成，大幅降低长 CoT RL 的训练成本；
- **Long2Short**：先训一个"长思考"模型追求上限性能，再通过模型合并/最短拒绝采样/专门的 long2short RL（对超长回答加惩罚）把能力压缩进一个"短思考"模型——k1.5-short 用平均 3272 token 就在 AIME 上拿到 60.8 分，比 GPT-4o（9.3 分，~2000 token）单 token 效率高 6.5 倍。

### 3.2 K2：万亿参数 MoE + Agentic Intelligence

K2（arXiv 2507.20534，2025-07）是 Kimi 系列第一次冲到万亿参数规模（1.04T 总参数，32B 激活），架构直接照搬 DeepSeek-V3 的 MLA + MoE 思路，但把稀疏度推得更高（384 专家 vs DeepSeek-V3 的 256，每 token 激活 8 个）。两个核心工程贡献：

1. **MuonClip 优化器**：Muon（一种基于 Newton-Schulz 正交化动量矩阵的优化器）比 AdamW token 效率更高，但在万亿参数规模训练时会出现"attention logits 爆炸"导致的训练不稳定。K2 提出的 **QK-Clip** 机制：监控每个注意力头的最大 logit，一旦超过阈值 τ，就对该头的 Q/K 投影权重做 rescale（只影响权重范数，不改变当前 step 的前向/反向计算）。效果：15.5T token 的预训练全程零 loss spike。
2. **智能体数据合成 + 联合 RL 后训练**：后训练阶段引入大规模 agentic 数据合成 pipeline，并让模型和真实/合成环境交互式地做联合 RL——这是 K2 主打"Agentic Intelligence"（而非单纯推理）定位的直接体现，在 SWE-Bench Verified（65.8）、Tau2-Bench（66.1）等智能体基准上超过大多数开源和闭源基线（不开启扩展思考模式）。

### 3.3 K2 Thinking → K2.5 → K2.6 → K2.7-Code：原生量化与多智能体

这四个版本（2025-11 到 2026-06）沿两条线并行推进：

- **量化线**：K2 Thinking 首次引入**原生 INT4 量化感知训练（QAT）**——不是训完之后再量化（PTQ），而是在训练阶段就把量化约束考虑进去，让模型直接原生适配 4-bit 部署，几乎无损。K2.6/K2.7-Code 延续同一套 QAT 方法，权重原生 INT4 分发（约 610GB，相比 FP16 的 ~2TB 降到约 3 成）。
- **多智能体线**：K2.5（论文 arXiv 2602.02276）在 K2-Base 基础上用 ~15T 视觉+文本混合 token 继续预训练，做成原生多模态模型，并推出 **Agent Swarm**（研究预览）——从"单智能体把任务做深"转向"自主协调一群专职智能体并行做事"，官方比喻为蜂群式协作。K2.6 进一步把这套原生多模态智能体能力做成正式版本；K2.7-Code 则是在同一套架构上专精编程场景的衍生版本。

### 3.4 K3：2.8T 参数、KDA、原生 1M 上下文

K3（2026-07-16 上线产品/API，2026-07-27 开放权重）是目前谱系里架构改动最大的一代，官方把它定位为对"深度学习十年不变的三大组件——注意力机制、残差连接、优化器"的系统性重构：

| 维度 | 数值/设计 |
|---|---|
| 总参数 / 激活参数 | 2.8T / 约 104B（16 of 896 专家路由） |
| MoE 设计 | **Stable LatentMoE**：SiTU-GLU 激活 + Quantile Balancing（用路由分数的分位数直接推导专家分配，替代启发式负载均衡超参） |
| 注意力 | **KDA（Kimi Delta Attention）**+ Gated MLA，3:1 混合布局（每 4 层 3 层 KDA 线性注意力 + 1 层 Gated MLA 全局注意力） |
| 跨层信息流 | **Attention Residuals（AttnRes）**：按块选择性检索早期层输出，而非均匀混合——训练效率提升约 25%，额外计算成本 <2% |
| 优化器 | **Per-Head Muon**：把 Muon 扩展到对每个注意力头独立优化 |
| 上下文窗口 | 原生 1M token，KDA 把长上下文解码速度提升最高 6.3 倍 |
| 多模态 | 原生视觉，视觉编码器 **MoonViT-V2** 从零用 next-token prediction 训练（不做对比学习预训练） |
| 量化 | 从 SFT 阶段起做 QAT，权重 MXFP4（4-bit 浮点+per-block scale）、激活 MXFP8——整体 scaling 效率比 K2 提升约 2.5 倍 |
| 训练 Infra | **MoonEP**（超大细粒度 MoE 专家并行通信库）、**FlashKDA**（KDA 高性能算子，H20 上 prefill 速度比 flash-linear-attention 基线快 1.72–2.22 倍）、**AgentEnv**（与 KVCache.ai 合作的智能体沙箱系统） |

几个值得单独记一下的"自证"实验（技术报告披露，可信度：官方一手信源，但缺乏第三方复现验证）：
- K3 在 24 小时内**自主完成** AttnRes、DeepSeek Sparse Attention、KDA、MLA 四种 GPU 内核的优化（AttnRes 延迟从 283.6ms 降到 114.4ms）；
- 自主开发了紧凑型 GPU 编译器 **MiniTriton**（Python 前端到 PTX 全链路由模型完成），在 NVIDIA L20 上性能超过 PyTorch eager / torch.compile / 标准 Triton；
- 作为概念验证，48 小时自主运行完成了一款推理芯片原型的设计（4mm²、146 万标准单元、集成 INT4 MAC 阵列），基于开源 EDA 工具 + Nangate 45nm 工艺库。

基准表现：WebDev Arena 开源模型第一（1679 Elo）；Artificial Analysis Intelligence Index v4.1 得分 57.1（全球第四，落后于 Claude Fable 5 和 GPT-5.6 Sol）；Terminal-Bench 2.1、BrowseComp、SWE Marathon、OmniDocBench 等智能体/编程基准领先多数开源模型。生态方面阿里云、华为昇腾、vLLM、SGLang 均 Day0 适配。

---

## 设计哲学的连续性

跨越 k0-math 到 K3 近两年、近十个版本，Kimi 系列有三条几乎没有变过的主线：

1. **长文本一直是旗舰能力**：k1.5 把 RL 上下文推到 128k，K2 到 128k/256K，K3 直接原生 1M——每一代都在往"更长"这个方向砸资源，且始终认为"长上下文本身就是推理能力的一个维度"（k1.5 报告原话："context length as a key dimension of the continued scaling of RL"）。
2. **推理训练范式持续简化再复杂化**：k1.5 主张"不需要 MCTS/价值函数/过程奖励模型，长上下文 + 策略优化就够"；到 K2/K3，这套简化框架又被套进更复杂的 agentic/多智能体环境里，本质上是同一套"用 RL 换取能力"的思路在不同任务粒度上的复用。
3. **工程效率的自主权**：从 MuonClip 解决训练稳定性，到原生 INT4/MXFP4 QAT 解决部署成本，到 K3 自主做内核优化/编译器/芯片设计——Moonshot 越来越倾向于"用模型自己的智能去优化训练/部署这套智能所需的基础设施"，形成某种递归自我改进的雏形。

## 全维度对比表：GPT-2 vs Kimi K3

| 维度 | GPT-2 (2019) | Kimi K3 (2026) |
|---|---|---|
| 参数规模 | 最大 1.5B | 2.8T（激活 ~104B） |
| 架构密度 | 稠密 | 稀疏 MoE（896 专家，激活 16） |
| 注意力 | 标准 MHA | KDA 线性注意力 + Gated MLA 混合 |
| 归一化/激活 | LayerNorm + GELU | RMSNorm 体系 + SiTU-GLU |
| 位置编码 | 学习到的绝对位置 | RoPE 变体 + KDA 递归状态 |
| 上下文窗口 | 1024 token | 1,000,000 token |
| 模态 | 纯文本 | 原生文本+视觉 |
| 训练目标 | 纯 next-token prediction | 预训练 + SFT + Agentic RL（可验证奖励 + 环境交互） |
| 优化器 | Adam | Per-Head Muon（+ QK-Clip 稳定性机制的前代技术） |
| 部署精度 | FP32/FP16 | 原生 MXFP4 QAT |
| 词表 | ~50K | 160K |

---

## Sources

- GPT-2: [Radford et al., "Language Models are Unsupervised Multitask Learners", OpenAI, 2019](https://cdn.openai.com/better-language-models/language_models_are_unsupervised_multitask_learners.pdf)
- GPT-3 / Scaling laws: [Kaplan et al., 2020](https://arxiv.org/abs/2001.08361); [Brown et al., "Language Models are Few-Shot Learners", 2020](https://arxiv.org/abs/2005.14165)
- RoPE: [Su et al., "RoFormer", 2021](https://arxiv.org/abs/2104.09864)
- RMSNorm: [Zhang & Sennrich, 2019](https://arxiv.org/abs/1910.07467); SwiGLU: [Shazeer, 2020](https://arxiv.org/abs/2002.05202); LLaMA: [Touvron et al., 2023](https://arxiv.org/abs/2302.13971)
- MQA: [Shazeer, 2019](https://arxiv.org/abs/1911.02150); GQA: [Ainslie et al., 2023](https://arxiv.org/abs/2305.13245)
- Mixtral: [Jiang et al., 2024](https://arxiv.org/abs/2401.04088); DeepSeekMoE: [Dai et al., 2024](https://arxiv.org/abs/2401.06066)
- MLA / DeepSeek-V2/V3: [DeepSeek-AI, 2024](https://arxiv.org/abs/2405.04434)
- InstructGPT/RLHF: [Ouyang et al., 2022](https://arxiv.org/abs/2203.02155); DeepSeek-R1: [DeepSeek-AI, 2025](https://arxiv.org/abs/2501.12948)
- Kimi k1.5: [arXiv 2501.12599](https://arxiv.org/abs/2501.12599) · [GitHub](https://github.com/MoonshotAI/kimi-k1.5)
- Kimi K2: [arXiv 2507.20534](http://arxiv.org/abs/2507.20534) · [GitHub](https://github.com/MoonshotAI/Kimi-K2)
- Kimi K2.5: [arXiv 2602.02276](https://huggingface.co/moonshotai/Kimi-K2.5) · [Tech Blog](https://www.kimi.com/en/blog/kimi-k2-5) · [VentureBeat](https://venturebeat.com/orchestration/moonshot-ai-debuts-kimi-k2-5-most-powerful-open-source-llm-beating-opus-4-5)
- Kimi K2.6/K2.7-Code: [知乎解读](https://zhuanlan.zhihu.com/p/2032879542541493707) · [modemguides.com](https://www.modemguides.com/blogs/ai-news/kimi-k2-7-code-open-source-release)
- Kimi K3: [官方 Tech Blog](https://www.kimi.com/zh-cn/blog/kimi-k3) · [MarkTechPost](https://www.marktechpost.com/2026/07/16/moonshot-ai-releases-kimi-k3-a-2-8-trillion-parameter-open-moe-model-with-kimi-delta-attention-and-1m-context/) · [腾讯云技术解析](https://developer.cloud.tencent.com/article/2711349) · [IT之家](https://www.ithome.com/0/982/259.htm) · [量子位](https://www.qbitai.com/2026/07/455179.html)
