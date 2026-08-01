> **先纠偏**：深入调研后没有找到 Moonshot AI（月之暗面）官方发表过任何"原生 1-bit 训练"的 Kimi 模型或论文。你在网上看到的"Kimi 1-bit 模型"，指的是 **Unsloth AI 对开源 Kimi 权重做的第三方动态低比特后训练量化（PTQ）**——这是社区/厂商对 Kimi 权重的二次加工，不是 Kimi/Moonshot 自己提出的研究成果。**更新（2026-07-27 K3 权重开放当天）**：Unsloth 已经为刚开源的 **Kimi K3** 出了官方文档级的 Dynamic 1-bit GGUF（[unsloth.ai/docs/models/kimi-k3](https://unsloth.ai/docs/models/kimi-k3)），且这次真的跑起来了、有实测精度数字——不再是 K2.6 时代那种"跑了但质量不达标被搁置"的状态。本文把 K2 → K2.6 → K3 三代的"1-bit 化"进展讲清楚，并顺带梳理"真正的原生 1-bit LLM"研究现状。

## 目录

1. [结论先行：这不是 Kimi 自己的工作](#结论先行这不是-kimi-自己的工作)
2. [K2 时代：Unsloth 对 Kimi K2 的动态 1.8-bit 量化](#k2-时代unsloth-对-kimi-k2-的动态-18-bit-量化)
3. [K3 时代：1-bit GGUF 从"搁置"到"真的跑起来了"](#k3-时代1-bit-gguf-从搁置到真的跑起来了)
4. [GGUF 是什么，以及体积数字是怎么算出来的](#gguf-是什么以及体积数字是怎么算出来的)
5. [技术原理：Dynamic Quantization 是怎么做到 1.x bit 的](#技术原理dynamic-quantization-是怎么做到-1x-bit-的)
6. [Kimi 自己真正的低比特路线：原生 INT4/MXFP4 QAT](#kimi-自己真正的低比特路线原生-int4mxfp4-qat)
7. [对比：学术界"真 1-bit"研究现状（BitNet / OneBit / ParetoQ）](#对比学术界真-1-bit研究现状bitnet--onebit--paretoq)
8. [附录：低比特数据格式基础（FP4/FP8/MX 缩放/三值编码）](#附录低比特数据格式基础fp4fp8mx-缩放三值编码)
9. [为什么这个区分重要](#为什么这个区分重要)
10. [Sources](#sources)

---

## 结论先行：这不是 Kimi 自己的工作

按"Kimi 最近提出 1-bit 模型"这个说法去查证，搜遍 Moonshot AI 官方 GitHub（moonshotai 组织）、Hugging Face 论文列表、arXiv 上 Kimi Team 署名的论文，**没有任何一篇是关于原生 1-bit / ternary 权重训练的**。Moonshot 自己的低比特研究方向是**原生 INT4/MXFP4（4-bit）量化感知训练**（从 K2 Thinking 开始，详见第 6 节），从未做到 1-bit。

真正对应"Kimi + 1-bit"这个组合词的，是：**Unsloth AI**（一家专注模型量化/微调效率的第三方公司，与 Moonshot 无关联）在 2025-07-14（K2 发布仅 3 天后）宣布把开源的 Kimi K2 权重压缩到平均 ~1.8-bit 的 GGUF 格式。中文科技媒体后续报道时，标题里经常直接简化成"Kimi 的 1-bit 版本/1-bit 量化模型"，容易让人误以为这是 Kimi 官方的研究成果。

## K2 时代：Unsloth 对 Kimi K2 的动态 1.8-bit 量化

| 项目 | 内容 |
|---|---|
| 主体 | Unsloth AI（第三方，非 Moonshot） |
| 对象 | Kimi K2（2025-07-11 发布的 1.04T 总参数 / 32B 激活 MoE 模型） |
| 时间 | 2025-07-14 |
| 方法 | Dynamic Quantization（动态/混合精度量化），产出多档 GGUF：UD-IQ1（最激进，平均 ~1.8-bit）到 UD-Q5_K_XL（接近无损） |
| 体积变化 | 官方原生发布格式 **block-FP8**（`F8_E4M3`，8-bit，见 [K2 官方 README](https://huggingface.co/moonshotai/Kimi-K2-Instruct)："Our model checkpoints are stored in the block-fp8 format"）~1.1TB → Unsloth 1.8-bit 版本 ~245GB（压缩约 80%） |
| 验证方式 | 官方测试用例：单次生成 Flappy Bird 小游戏、通过"七边形测试"等编码任务，性能未见明显退化 |
| 部署方式 | 支持内存卸载（memory offload），可在 512GB RAM 的 Apple M3 Ultra 上跑，或多节点 NVIDIA B200 GPU 集群 |

K2.6（2026-04）发布时，社区也尝试对其做同类"1-bit 实验"，但当时报道称**质量分数不达标，1-bit/3-bit 版本被暂时搁置**（4-bit 原生 INT4 版本约 610GB 是当时最激进的可用选项）。这个"搁置"状态在 K3 上被打破了——见下一节。

## K3 时代：1-bit GGUF 从"搁置"到"真的跑起来了"

Kimi K3 权重于 2026-07-27 开放当天，Unsloth 就同步发布了 [Kimi K3 官方运行指南](https://unsloth.ai/docs/models/kimi-k3)和对应的 [`unsloth/Kimi-K3-GGUF`](https://huggingface.co/unsloth/Kimi-K3-GGUF) 权重——这次不再是"搁置"，而是给出了完整的精度/体积实测数据：

| 量化档位 | 体积 | 相比全精度缩减 | Top-1 准确率 |
|---|---|---|---|
| "全精度"基准（Q8 GGUF 无损重封装，**不是** BF16/FP16——K3 官方原生发布本身已经是 MXFP4/4-bit，见下节，Unsloth 的 Q8 只是把这份 4-bit 数据原样封进 GGUF 的 8-bit 容器，没有额外精度可补） | 1.56 TB | — | 基准 100% |
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

## GGUF 是什么，以及体积数字是怎么算出来的

**GGUF**（GPT-Generated Unified Format）是 `llama.cpp`/`ggml` 项目定义的一种**单文件二进制格式**，把权重张量和模型跑起来所需的全部元数据（分词器、架构超参、chat template 等）打包在一个文件里，专门给 CPU/GPU 混合推理用。它不是量化方法本身，只是**容器格式**——一个 GGUF 文件里的张量可以是 F32、F16，也可以是 Q8_0、Q4_K、IQ1_S 等各种量化类型，格式本身不决定精度，精度由里面每个张量选的"量化类型"决定。

**"文件是 GGUF"和"真的能用"是两件不同的事**：
1. **转换成功**（technically works）：把原始权重跑一遍量化算法，产出合法的 GGUF 文件，能被 llama.cpp 加载、不报错——这一步任何比特位都能做到，哪怕压到 0.5-bit 也能"转换成功"。
2. **能推理**（loads & runs）：文件加载后能正常做前向计算，输出的是合法 token 而不是乱码——比转换成功高一个门槛，但低比特量化很容易在这一步就退化到输出不连贯。
3. **质量达标**（真的跑起来了）：在真实基准上测出可用的准确率数字，官方愿意把这个数字和使用指南一起公开发布。K2.6 的 1-bit 卡在了第 2/3 步之间（据报道跑是能跑，但基准分数太低，官方没有把它作为正式产物发布）；K3 的 1-bit 走完了全部三步——Unsloth 给出了 ~78.9% top-1 准确率并配了完整的硬件需求指南，这才是我在上一节说"真的跑起来了"的含义：**不是文件存在，而是有实测数字、官方愿意背书发布**。

**体积怎么算**：最基础公式是

```
文件体积 ≈ 参数总量 × 平均每参数比特数 / 8 + 少量格式/元数据开销
```

用这个公式反推验证一下笔记里引用的官方数字，误差都在个位百分比内（说明数字是自洽的）：

| 模型/量化 | 参数总量 | 官方体积 | 反推的平均 bits/参数 | 说明 |
|---|---|---|---|---|
| Kimi K2 官方 block-FP8 | 1.04T | ~1.1TB | ~8.1 bit | 8-bit 格式，数字对得上 |
| Unsloth K2 Dynamic 1.8-bit | 1.04T | 245GB | **~1.88 bit** | 和"1.8-bit"标称一致 |
| Kimi K3 官方 MXFP4 原生 | 2.8T | ~1.4TB（中文技术解读估算） | ~4.0 bit | 4-bit 格式，数字对得上 |
| Unsloth K3 "全精度"Q8 基准 | 2.8T | 1.56TB | **~4.46 bit** | 关键点：这个数字接近 4-bit，**不是**真的 8-bit！ |
| Unsloth K3 Dynamic 2-bit XL | 2.8T | 861.3GB | ~2.46 bit | 比标称"2-bit"略高，因为不敏感层保留更高精度、块量化有共享 scale 开销 |
| Unsloth K3 Dynamic 1-bit | 2.8T | 594GB | **~1.70 bit** | 比标称"1-bit"略高，同上原因；数量级上和 BitNet 的 1.58-bit 接近 |

**为什么 K3 的"全精度基准"（1.56TB）反推出来是 ~4.46 bit，而不是 8-bit？** 这是本节最关键的发现：K3 官方发布时权重本身就已经是 **MXFP4（4-bit）**——市面上根本没有一份真正 16-bit 的 K3 主检查点在流通。Unsloth 页面上说的"Q8（Lossless，1.56TB）"，其实是把这份已经是 4-bit 的官方数据**原样重新封装进 GGUF 的 8-bit 容器**，容器格式本身有元数据/块 scale 开销，所以比原始 MXFP4（~1.4TB）略大，但并没有从更高精度"还原"出额外信息——这也解释了为什么 Unsloth 原文里 Q8 只比 Q4 版本大 50GB（正常情况下 8-bit 应该是 4-bit 体积的 2 倍，这里差距很小，正是因为两者的信息源头都封顶在 4-bit）。**换句话说，K3 的"全精度/无损"是相对官方发布版本而言的无损，不是相对某个更高精度训练检查点的无损**——这一点和 K2（官方发布本身是真 8-bit，Unsloth 的量化是从真 8-bit 往下压）不是同一种情况，容易混淆，特此说明。

**"1.8-bit"/"1-bit"这类标称数字为什么和反推出来的不完全一致**：因为 Unsloth 的 Dynamic Quantization 是**逐层混合精度**——大多数冗余的 MoE 专家权重被压到很低的比特，但 attention、共享专家、embedding 等对质量影响大的层被保留在更高精度；同时 GGUF 的块量化（block quantization）本身也有开销：比如把 32 个权重分成一组共享一个 scale 系数，这个 scale 也要占存储空间，摊到每个权重上会比"标称比特数"略高一点。所以标称的"1-bit"/"2-bit"/"Q8"都是**这一档量化方案的名字/目标定位**，不是严格保证"每个参数正好 N 位"，实际体积反推出来的是全模型的**平均**比特数。

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

## 附录：低比特数据格式基础（FP4/FP8/MX 缩放/三值编码）

这一节把本文反复出现的 MXFP4、MXFP8、1.58-bit 这几个术语的底层原理讲清楚。

### 命名规则：ExMy

现代低比特格式基本都是**浮点**，命名遵循 `ExMy`：1 个符号位 + x 个指数位（决定动态范围）+ y 个尾数位（决定精度）。

| 格式 | 符号 | 指数 | 尾数 | 总位数 |
|---|---|---|---|---|
| FP32 | 1 | 8 | 23 | 32 |
| FP16 | 1 | 5 | 10 | 16 |
| BF16 | 1 | 8 | 7 | 16 |
| FP8（E4M3） | 1 | 4 | 3 | 8 |
| FP8（E5M2） | 1 | 5 | 2 | 8 |
| FP4（E2M1） | 1 | 2 | 1 | 4 |

裸 FP4（E2M1）按 OCP 规范动态范围极窄（正规数只有 ±1.0 到 ±6.0），直接拿来表示量级差异大的权重矩阵精度会崩——这就是几乎所有低比特格式都要搭配"缩放因子（scale）"的原因，**MXFP4/MXFP8 的核心差异正是缩放因子的粒度**。

### 1.58-bit 是怎么来的

最基础的信息论：一个符号有 N 种可能状态，理论最少编码位数是 log₂N。

- 二值 BitNet 原版 {-1, +1}：2 种状态 → log₂2 = **1 bit**（正好整数）
- **BitNet b1.58** {-1, 0, +1}：3 种状态 → log₂3 ≈ **1.58496 bit**

论文原话："We have added an additional value of **0** to the original 1-bit BitNet, resulting in **1.58 bits** in the binary system"——单纯因为多了"0"这一个状态，状态数从 2 变成 3，理论下限从整数的 1 bit 变成非整数的 1.58 bit。这个 0 不是凑数的：它让权重矩阵具备"直接判定某条连接不存在"的特征筛选能力，是 b1.58 比纯二值版本效果更好的原因之一。

**理论值 vs 实际存储，容易搞混的一点**：
- 理论信息量下限：log₂3 ≈ 1.585 bit/权重（需要算术编码或多个三值联合打包才能逼近，比如 5 个三值一起编码：3⁵=243 ≤ 2⁸=256，正好塞进 1 字节，均摊 8/5=1.6 bit/权重）；
- BitNet b1.58 官方技术报告（2504.12285）里 GPU kernel 的实际做法是"**4 个三值打包进一个 int8**"——每个三值实际占 **2 bit**（2²=4 ≥ 3 种状态，够用但不是最优），用字节对齐、解包简单换掉了理论最优值。

也就是说，"1.58-bit"是设计目标/信息论下限，不代表现实里每个权重在显存中真的只占 1.58 bit——和前面几节讲的"Dynamic Quantization 标称 bit 数 vs 反推 bit 数不完全一致"是同一类现象。

### MXFP4 vs 裸 FP4

**裸 FP4** 就是 E2M1 元素格式本身，没有内置缩放机制，真要用得自己额外加一个缩放因子，常见做法是整个 tensor 或整个 channel 共享一个 scale——粒度很粗。

**MXFP4**（Microscaling FP4，OCP 2023 年规范，AMD/Intel/Microsoft/NVIDIA/ARM/Meta/Qualcomm 联合制定）元素格式还是 E2M1（4 bit），核心区别是缩放粒度更细：

```
MXFP4 一个 block（32 个元素）：
┌──────────┐  ┌────┐┌────┐   ...   ┌────┐
│  E8M0    │  │E2M1││E2M1│         │E2M1│  ← 32 × 4-bit 元素
│  8 bits  │  │4bit││4bit│         │4bit│
└──────────┘  └────┘└────┘         └────┘
     ↑
  8-bit 无符号指数：scale = 2^(e − 127)
```

- **32 个元素一组**，共享一个 **8-bit 的 E8M0 缩放因子**（不是整个 tensor 共享一个，粒度细了很多）；
- E8M0 = 纯 8-bit 无符号指数，**没有符号位、没有尾数**——只能表示"2 的幂"这种缩放倍数，反量化时是一次移位操作就能算完，不需要真的做乘法，硬件实现极便宜；
- 每个 block 的**有效比特数** = (8 bit scale + 32×4 bit 元素) / 32 元素 = 136/32 = **4.25 bit/元素**（比裸 FP4 多 0.25 bit 的 scale 开销，换来每 32 个权重就能自适应一次量级，精度比"一个 tensor 一个 scale"的裸 FP4 好得多）。

这正好和第 4 节反推出的 K3"全精度基准"约 4.46 bit/参数吻合（4.25 是 MXFP4 纯文本层理论值，模型里还有 embedding/norm 等不参与 MXFP4、保留更高精度的层，把平均值往上拉了一点）。

### MXFP8 是什么

和 MXFP4 是同一套底座（32 元素一组 + 共享 E8M0 缩放），只是元素格式从 4-bit FP4 换成 **8-bit FP8**（E4M3 或 E5M2 均合规），有效比特数 = (8+32×8)/32 = **8.25 bit/元素**。

用途上，MXFP8 通常放**激活值**而不是权重——这也是 K3"MXFP4 权重 + MXFP8 激活"的原因：权重体量大、精度容忍度相对高，压到 4-bit；激活值是每次前向传播实时算出来的，数值分布更敏感，保留在 8-bit 更稳。MXFP8 和 H100 时代那种"整个 tensor 一个 scale、且延迟更新（delayed scaling）"的传统 FP8 不是一回事——MXFP8 的 32 元素细粒度 block scale 精度更高，是 Blackwell 一代新增的原生支持格式。

**一句话串起来**：FP4/FP8 决定"每个数自己长什么样"（指数/尾数怎么分），MX 系列决定"一群数怎么共享一个缩放因子"——MXFP4/MXFP8 是"裸 FP4/FP8 元素 + 32 个一组的 E8M0 细粒度缩放"这个统一底座的两个具体实例。

## 为什么这个区分重要

1. **归因要对**：把 Unsloth 的第三方量化工作说成"Kimi 提出的",会误判 Moonshot 的技术方向（他们实际押的是 4-bit 原生 QAT,不是 1-bit）。
2. **质量预期要对**：Dynamic Quantization 的"1.8-bit"是全模型平均值、且是 PTQ,不是每个参数都严格 1-bit,性能保证的方式和 BitNet/OneBit 这类原生训练的"真 1-bit"完全不是一回事,不能拿基准直接类比。
3. **可复现性要留意**：同样的"1-bit 化"操作在 K2 上效果尚可（1.8-bit），在 K2.6 上被曝出质量不达标而搁置，到 K3 又重新做出了可用版本（1-bit，~78.9% 准确率）——说明这完全取决于 Unsloth 每一轮具体的量化工程，不是能无脑套用到"下一个 Kimi 模型"上的稳定结论，每次新模型发布都要单独看实测数字，不能延用上一代的印象。

---

## Sources

- Unsloth 动态量化 Kimi K2: [aigc.izzi.cn 报道](https://aigc.izzi.cn/article/24514.html)（2025-07-15）
- Unsloth Kimi K3 官方运行指南 + Dynamic 1-bit/2-bit 实测数据: [unsloth.ai/docs/models/kimi-k3](https://unsloth.ai/docs/models/kimi-k3) · [unsloth/Kimi-K3-GGUF (Hugging Face)](https://huggingface.co/unsloth/Kimi-K3-GGUF)
- GGUF 格式说明: [Hugging Face Hub 文档 - GGUF](https://huggingface.co/docs/hub/gguf)
- Kimi K2 官方发布精度 (block-fp8): [moonshotai/Kimi-K2-Instruct README](https://huggingface.co/moonshotai/Kimi-K2-Instruct) · [GitHub Issue #63 (训练 bf16 / 推理 PTQ 转 block-fp8)](https://github.com/MoonshotAI/Kimi-K2/issues/63)
- K2.6/K2.7-Code 量化现状与社区 1-bit 实验搁置: [modemguides.com](https://www.modemguides.com/blogs/ai-news/kimi-k2-7-code-open-source-release)
- Kimi K2 Thinking / K2.6 / K3 原生 INT4/MXFP4 QAT: [intuitionlabs.ai 技术深度解读](https://intuitionlabs.ai/articles/kimi-k2-technical-deep-dive) · [知乎 K2.6 解读](https://zhuanlan.zhihu.com/p/2032879542541493707) · [Kimi K3 官方 Tech Blog](https://www.kimi.com/zh-cn/blog/kimi-k3)
- BitNet b1.58: [Ma et al., arXiv 2402.17764](https://arxiv.org/abs/2402.17764)
- OneBit: [Xu et al., arXiv 2402.11295](https://arxiv.org/pdf/2402.11295.pdf)，中文解读见[澎湃新闻](https://m.thepaper.cn/newsDetail_forward_27682564)/[腾讯云](https://cloud.tencent.com.cn/developer/article/2394532)
- ParetoQ: [智源社区论文解读](https://hub.baai.ac.cn/paper/0e9c1d88-a326-4679-8663-cc93204faaa8)
- BiLLM: [澎湃新闻 IEEE Spectrum 报道](https://m.thepaper.cn/newsDetail_forward_27682564)
- OCP Microscaling (MX) Formats 规范: [OCP 官方 PDF](https://www.opencompute.org/documents/ocp-microscaling-formats-mx-v1-0-spec-final-pdf) · [Rouhani et al., "Microscaling Data Formats for Deep Learning", arXiv 2310.10537](https://arxiv.org/abs/2310.10537) · [MXFP4 图解 (zeroentropy.dev)](https://zeroentropy.dev/concepts/mxfp4/)
- BitNet b1.58 原始论文 (1.58-bit 命名来源): [Ma et al., arXiv 2402.17764](https://arxiv.org/abs/2402.17764)
- BitNet b1.58 2B4T 技术报告 (int8 打包三值的工程实现): [arXiv 2504.12285](https://arxiv.org/html/2504.12285v2)
