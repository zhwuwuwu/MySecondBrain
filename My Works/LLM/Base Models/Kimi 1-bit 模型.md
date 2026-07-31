> **先纠偏**：深入调研后没有找到 Moonshot AI（月之暗面）官方发表过任何"原生 1-bit 训练"的 Kimi 模型或论文。你在网上看到的"Kimi 1-bit 模型"，指的是 **Unsloth AI 对开源 Kimi 权重做的第三方动态低比特后训练量化（PTQ）**——这是社区/厂商对 Kimi 权重的二次加工，不是 Kimi/Moonshot 自己提出的研究成果。**更新（2026-07-27 K3 权重开放当天）**：Unsloth 已经为刚开源的 **Kimi K3** 出了官方文档级的 Dynamic 1-bit GGUF（[unsloth.ai/docs/models/kimi-k3](https://unsloth.ai/docs/models/kimi-k3)），且这次真的跑起来了、有实测精度数字——不再是 K2.6 时代那种"跑了但质量不达标被搁置"的状态。本文把 K2 → K2.6 → K3 三代的"1-bit 化"进展讲清楚，并顺带梳理"真正的原生 1-bit LLM"研究现状。

## 目录

1. [结论先行：这不是 Kimi 自己的工作](#结论先行这不是-kimi-自己的工作)
2. [K2 时代：Unsloth 对 Kimi K2 的动态 1.8-bit 量化](#k2-时代unsloth-对-kimi-k2-的动态-18-bit-量化)
3. [K3 时代：1-bit GGUF 从"搁置"到"真的跑起来了"](#k3-时代1-bit-gguf-从搁置到真的跑起来了)
4. [技术原理：Dynamic Quantization 是怎么做到 1.x bit 的](#技术原理dynamic-quantization-是怎么做到-1x-bit-的)
5. [Kimi 自己真正的低比特路线：原生 INT4/MXFP4 QAT](#kimi-自己真正的低比特路线原生-int4mxfp4-qat)
6. [对比：学术界"真 1-bit"研究现状（BitNet / OneBit / ParetoQ）](#对比学术界真-1-bit研究现状bitnet--onebit--paretoq)
7. [为什么这个区分重要](#为什么这个区分重要)
8. [Sources](#sources)

---

## 结论先行：这不是 Kimi 自己的工作

按"Kimi 最近提出 1-bit 模型"这个说法去查证，搜遍 Moonshot AI 官方 GitHub（moonshotai 组织）、Hugging Face 论文列表、arXiv 上 Kimi Team 署名的论文，**没有任何一篇是关于原生 1-bit / ternary 权重训练的**。Moonshot 自己的低比特研究方向是**原生 INT4/MXFP4（4-bit）量化感知训练**（从 K2 Thinking 开始，详见第 5 节），从未做到 1-bit。

真正对应"Kimi + 1-bit"这个组合词的，是：**Unsloth AI**（一家专注模型量化/微调效率的第三方公司，与 Moonshot 无关联）在 2025-07-14（K2 发布仅 3 天后）宣布把开源的 Kimi K2 权重压缩到平均 ~1.8-bit 的 GGUF 格式。中文科技媒体后续报道时，标题里经常直接简化成"Kimi 的 1-bit 版本/1-bit 量化模型"，容易让人误以为这是 Kimi 官方的研究成果。

## K2 时代：Unsloth 对 Kimi K2 的动态 1.8-bit 量化

| 项目 | 内容 |
|---|---|
| 主体 | Unsloth AI（第三方，非 Moonshot） |
| 对象 | Kimi K2（2025-07-11 发布的 1.04T 总参数 / 32B 激活 MoE 模型） |
| 时间 | 2025-07-14 |
| 方法 | Dynamic Quantization（动态/混合精度量化），产出多档 GGUF：UD-IQ1（最激进，平均 ~1.8-bit）到 UD-Q5_K_XL（接近无损） |
| 体积变化 | FP16 原始 ~1.1TB → 1.8-bit 版本 ~245GB（压缩约 80%） |
| 验证方式 | 官方测试用例：单次生成 Flappy Bird 小游戏、通过"七边形测试"等编码任务，性能未见明显退化 |
| 部署方式 | 支持内存卸载（memory offload），可在 512GB RAM 的 Apple M3 Ultra 上跑，或多节点 NVIDIA B200 GPU 集群 |

K2.6（2026-04）发布时，社区也尝试对其做同类"1-bit 实验"，但当时报道称**质量分数不达标，1-bit/3-bit 版本被暂时搁置**（4-bit 原生 INT4 版本约 610GB 是当时最激进的可用选项）。这个"搁置"状态在 K3 上被打破了——见下一节。

## K3 时代：1-bit GGUF 从"搁置"到"真的跑起来了"

Kimi K3 权重于 2026-07-27 开放当天，Unsloth 就同步发布了 [Kimi K3 官方运行指南](https://unsloth.ai/docs/models/kimi-k3)和对应的 [`unsloth/Kimi-K3-GGUF`](https://huggingface.co/unsloth/Kimi-K3-GGUF) 权重——这次不再是"搁置"，而是给出了完整的精度/体积实测数据：

| 量化档位 | 体积 | 相比全精度缩减 | Top-1 准确率 |
|---|---|---|---|
| 全精度（BF16，Q8 无损基准） | 1.56 TB | — | 基准 100% |
| Dynamic 2-bit XL | 861.3 GB | 45% | ~90% |
| **Dynamic 1-bit**（`UD-IQ1` 系） | **594 GB** | **62%** | **~78.9%** |

配套的硬件需求表（总内存 = RAM + VRAM，或统一内存）：

| Dynamic 1-bit S | Dynamic 1-bit M | Dynamic 2-bit XXS | Dynamic 2-bit XL | Q8（无损） |
|---|---|---|---|---|
| 610 GB | 665 GB | 726 GB | 880 GB | 1.6 TB |

几个技术细节（来自 Unsloth 官方文档）：
- 实现基于社区 [llama.cpp PR #26185](https://github.com/ggml-org/llama.cpp/pull/26185)，Unsloth 自己再 fork 了一份加上视觉支持和若干 bug 修复；
- K3 的 vision tower 与 K2.5 的类似，但改用 RMSNorm、无 bias、非方形的融合 QKV、post-norm projector——Unsloth 特意把这些差异适配进了 GGUF 转换流程；
- Kimi 官方默认训练时"保留思维链"（preserved thinking on），Unsloth 的量化/推理流程也保留了完整思维链，不会静默截断；
- 可运行硬件：NVIDIA DGX Station，或接了 128GB+ RAM 的 Mac Studio。

也就是说，**"1-bit 化"这件事在 Kimi 谱系上不是单调进步的**：K2（1.8-bit，可用）→ K2.6（1-bit 尝试，质量不达标被搁置）→ K3（1-bit，~78.9% 准确率，正式发布并给出实测数字）。每一代能不能把 1-bit 做到可用，取决于 Unsloth 那一轮具体的量化工程（重要性分析、逐层比特预算怎么分配），**不是 Kimi 架构本身决定的**——依然是纯第三方 PTQ 工作，和 Moonshot 官方的原生 INT4/MXFP4 QAT 路线是两条并行、不冲突的线。

## 技术原理：Dynamic Quantization 是怎么做到 1.x bit 的

Unsloth 的"Dynamic Quantization"（此前在 DeepSeek-R1 上也用过同一套方法）核心思路不是把模型所有权重统一压到 1-bit（那样质量会崩），而是**按层的重要性分配不同的比特预算**：
- 对 MoE 中大量、冗余度高的专家 FFN 权重，用更激进的低比特（逼近 1-bit/2-bit）；
- 对注意力层、共享专家、embedding 等对整体质量影响更大的权重，保留更高精度（4-bit 甚至更高）；
- 最终得到的是一个**逐层混合比特宽度**的 GGUF 文件，"1.8-bit"是全模型的**平均比特数**，不是每个参数都严格 1-bit。

这和下一节要讲的"真 1-bit"（BitNet 系）是本质不同的两件事：Dynamic Quantization 是**训练后量化（PTQ）+ 混合比特分配**，而 BitNet/OneBit 是**从训练阶段就用 1-bit/ternary 表示权重（量化感知训练甚至原生训练）**。

## Kimi 自己真正的低比特路线：原生 INT4/MXFP4 QAT

Moonshot 自己在低比特化上的真实进展，起点是 **K2 Thinking**（2025-11）：不是等模型训完再量化，而是**从训练阶段（K3 明确写的是"从 SFT 阶段起"）就把量化约束纳入训练目标**，让模型原生适配 4-bit 部署——这是量化感知训练（QAT），效果是"几乎无损"而不是"有损但可接受"。这条线一直延续到 K2.6、K2.7-Code（原生 INT4，权重 ~610GB）以及 K3（MXFP4 权重 + MXFP8 激活，2.8T 模型权重存储约 1.4TB）。

也就是说：**Kimi 官方的量化叙事从头到尾都停在 4-bit，从未官方发布过 1-bit 版本**——1-bit 这件事完全是社区/第三方（Unsloth 等）在 Kimi 开源权重基础上二次加工出来的。

## 对比：学术界"真 1-bit"研究现状（BitNet / OneBit / ParetoQ）

如果你对"原生 1-bit LLM"这个方向本身感兴趣（而不是特指 Kimi），目前公开的代表性工作是：

| 工作 | 机构 | 核心方法 | 备注 |
|---|---|---|---|
| **BitNet b1.58** | Microsoft, 2024 | 每个权重三值 {-1, 0, 1}（严格来说是 1.58-bit），从训练阶段原生使用 | 定义了"1-bit LLM 的新 scaling law"，是这个方向的起点性工作 |
| **OneBit** | 清华 + 哈工大, 2024 | 把已训练好的高精度模型迁移到真 1-bit（权重矩阵严格 1-bit，配合两个高精度值向量做 SVID 分解初始化 + QAT 知识蒸馏） | 平均每参数仅占 ~1.0073 bit，LLaMA 系列上保留约 83% 性能，是首个做到"真 1-bit"（而非 1.58-bit）的公开工作 |
| **ParetoQ** | Meta, 2024 | 统一框架系统比较 1/1.58/2/3/4-bit 量化，发现 2-bit 是"学习方式发生质变"的临界点 | 结论：三值/2-bit/3-bit 在"大小 vs 精度"上普遍优于 4-bit 和纯二值 |
| **BiLLM** | 港大 + ETH + 北航, 2024 | 训练后量化（PTQ），多数权重 1-bit + 少数关键权重 2-bit，平均 ~1.1-bit | 首次在接近 1-bit 的 PTQ 场景下做到性能有保证，ICML 2024 |

这些工作都**不是 Kimi/Moonshot 做的**，但代表了"1-bit LLM"这个方向的真实前沿——如果你的技术 blog 来源里提到了具体方法名（比如"三值权重""SVID""1.58-bit"这类术语），大概率讲的是这几篇论文之一，而不是 Kimi。

## 为什么这个区分重要

1. **归因要对**：把 Unsloth 的第三方量化工作说成"Kimi 提出的",会误判 Moonshot 的技术方向（他们实际押的是 4-bit 原生 QAT,不是 1-bit）。
2. **质量预期要对**：Dynamic Quantization 的"1.8-bit"是全模型平均值、且是 PTQ,不是每个参数都严格 1-bit,性能保证的方式和 BitNet/OneBit 这类原生训练的"真 1-bit"完全不是一回事,不能拿基准直接类比。
3. **可复现性要留意**：同样的"1-bit 化"操作在 K2 上效果尚可（1.8-bit），在 K2.6 上被曝出质量不达标而搁置，到 K3 又重新做出了可用版本（1-bit，~78.9% 准确率）——说明这完全取决于 Unsloth 每一轮具体的量化工程，不是能无脑套用到"下一个 Kimi 模型"上的稳定结论，每次新模型发布都要单独看实测数字，不能延用上一代的印象。

---

## Sources

- Unsloth 动态量化 Kimi K2: [aigc.izzi.cn 报道](https://aigc.izzi.cn/article/24514.html)（2025-07-15）
- Unsloth Kimi K3 官方运行指南 + Dynamic 1-bit/2-bit 实测数据: [unsloth.ai/docs/models/kimi-k3](https://unsloth.ai/docs/models/kimi-k3) · [unsloth/Kimi-K3-GGUF (Hugging Face)](https://huggingface.co/unsloth/Kimi-K3-GGUF)
- K2.6/K2.7-Code 量化现状与社区 1-bit 实验搁置: [modemguides.com](https://www.modemguides.com/blogs/ai-news/kimi-k2-7-code-open-source-release)
- Kimi K2 Thinking / K2.6 / K3 原生 INT4/MXFP4 QAT: [intuitionlabs.ai 技术深度解读](https://intuitionlabs.ai/articles/kimi-k2-technical-deep-dive) · [知乎 K2.6 解读](https://zhuanlan.zhihu.com/p/2032879542541493707) · [Kimi K3 官方 Tech Blog](https://www.kimi.com/zh-cn/blog/kimi-k3)
- BitNet b1.58: [Ma et al., arXiv 2402.17764](https://arxiv.org/abs/2402.17764)
- OneBit: [Xu et al., arXiv 2402.11295](https://arxiv.org/pdf/2402.11295.pdf)，中文解读见[澎湃新闻](https://m.thepaper.cn/newsDetail_forward_27682564)/[腾讯云](https://cloud.tencent.com.cn/developer/article/2394532)
- ParetoQ: [智源社区论文解读](https://hub.baai.ac.cn/paper/0e9c1d88-a326-4679-8663-cc93204faaa8)
- BiLLM: [澎湃新闻 IEEE Spectrum 报道](https://m.thepaper.cn/newsDetail_forward_27682564)
