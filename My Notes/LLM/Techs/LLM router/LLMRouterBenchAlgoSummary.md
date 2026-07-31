## 核心问题
怎么区分模型？同样架构不同训练目标vs同样训练目标不同架构？
怎么在同一向量空间表征不同模型

**核心还是看问题定义成什么：**
有监督
两个模型二分多分类/能否答对二分类/Query聚类/模型聚类
但是有个设定是使用LLM description 表示模型是不是太草率了。
我们需要一个统一的模型向量空间来表示各个模型的内在，或许包含结构，参数，数据，训练方法等关键信息。--> 参考模型水印，模型打假相关工作。

###  Comments：
**（First confirm if Ivy has known about these routers)**
1. RouterLLM/EmbedLLM & Avengers 大多数完全没有触及到模型的本质，没有分析什么模型擅长做什么事情，
	1. （给问题做分类，但是没给模型做分类，本身给模型分类比较抽象，对应的向量空间不太直观，[TODO]，需要找一些设计怎么给LLM分类的paper，例如LLM判断抄袭） 
	2. （EmbedLLM尝试这个，model embedding预测推理正确性分数，不过他是逐个question预测的，要是能和问题类别连上似乎更好？预测模型回答一个类别问题的准确率）
2. 黑盒选择模型任务，也有RL可选 （和memory选择其实是一类问题）
3. 对于Hybrid Agent,  主要目标不是同级别的模型分别擅长哪个类别问题，更重要的应该是按照参数量/price给模型分级，在级别和正确性里trade off
	1.  一种是简单vs困难问题
	2. 一种是同样难度级别问题，但是不同类别的任务
4. New Task: Safety相关router，判断隐私性然后进行选模型
5. 对于Perf-Cost 问题我们可以直接用performance/token 或者performance/$ 作为目标优化，不过本地模型的price不好算，
6. 另外模型的价格本身是变化的，新模型出了老模型会降价，不过这个不影响优化的方法论，另外在inference优化那边也是这么玩的，token/s/$, tokens/s/Mb内存

Other question
**Q: whole picture and router for which task, goals and key timelines?**
## 总结 
### Router基本情况 

| #   | TASK          | NAME        | ALGO                                                                | NOTE                                                                                                                                                      |                                                                         | 来源                            | COST？ |
| --- | ------------- | ----------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ----------------------------- | ----- |
| 1   | 二分类           | routerLLM   | embedding + 线性proj + BCE_Logits_Loss 分类器                            | 比原版简单，MF + ~~SW Ranking + Random + BERT~~                                                                                                                 |                                                                         | ICLR 2025                     | Y     |
| 2   | 聚类            | avengerPro  | embedding + L2Norm + KMeans + 分数统计                                  | 简单规划问题                                                                                                                                                    |                                                                         | DAI 2025                      | Y     |
| 3   | 判断答案正确置信度，二分类 | FrugalGPT   | 小语言模型预测置信度打分+根据打分结果制定级联策略；两阶段训练，先训练打分模型，再训练级联策略                     | 1. 模型逆向生成回答置信度-对于开源模型来说多此一举；2. 多次infer+计算score效率很低，得假设大部分问题很简单（这里缺少论证），这个效果可能还不如更简单的贪心；3. scorer模型可能比实际用的local model还大；4. scorer训练要求数据量不小；5.              | 槽点太多                                                                    | TMLR 2024                     | Y     |
| 4   | 判断正确与否二分类，    | EmbedLLM    | 几乎同routerLLM；目标换成每个模型判断预测是否正确，多模型中选取最可能正确的一个                        |                                                                                                                                                           |                                                                         | ICLR 2025 spotlight           |       |
| 5   | 二部图           | GraphRouter |                                                                     |                                                                                                                                                           |                                                                         | ICLR 2025                     | Y     |
| 6   | 使用LLM判断(多选一)  | MODEL-SAT   | 微调Qwen2.5-7B-instruct 理解模型能力，基于模型描述判断是否能够回答成功；预测生成yes/no；选取yes概率最大的 | 1. 先生成LLM描述，然后把描述和QA pair一起e2e微调，预测yes/not，描述是谁生成的啊，描述经过embedding就能代表LLM了？ 2. yes/not 是从LLM输出里扒出来用softmax（logist（yes））作比较，这玩意跨回答之间有可比性吗？还不如直接二分类区分yes/no呢 |                                                                         | AAAI 2025                     |       |
| 7   | 双对比学习（多选一）    | RouteDC     | query encoder + model embeddings同上；训练损失是双对比学习的loss                  | benchmark 中双对比损失被改成普通对比损失：Q和答对的LLM embedding接近，相似的Q接近                                                                                                     |                                                                         | NIPS 2024                     |       |
| 8   | 二分类-级联        | HybridLLM   | router可以是deberta/gte-qwen2-7B；二分类/多分类问题，backbone比较复杂而已              |                                                                                                                                                           | baselines/Best-route-llm/ （与 BEST-Route 共享代码库）自己也有benchmark；bestLLM是n选1 | 2024 ICLR；BestRoute ICML 2025 | Y     |
|     |               |             |                                                                     |                                                                                                                                                           |                                                                         |                               |       |

### 训练难度

资源需求
![[Pasted image 20260309040215.png|697]]
![[Pasted image 20260309040243.png]]

训练数据数量：[TODO]


## 算法详情


### EmbedLLM Detail：
和routeLLM对比
![[Pasted image 20260306021701.png]]
训练过程
![[Pasted image 20260306021724.png]]
inference，eval，KNN变体：
![[Pasted image 20260306021822.png]]

### FrugalGPT Detail:

Scorer 模型结构：
![[Pasted image 20260306014021.png]]
Scorer 模型参数量：
![[Pasted image 20260306013825.png]]
Scorer 训练数据量：
![[Pasted image 20260306013411.png]]
问题总结：
![[Pasted image 20260306013452.png]]

### GraphRouter

没看明白
### MODEL-SAT

论文: Capability Instruction Tuning: A New Paradigm for Dynamic LLM Routing (AAAI 2025, 南京大学)
	arXiv: 2502.17282
#### 一、核心思想
MODEL-SAT（Model Selection with Aptitude Test）的方法和前面所有 router 完全不同 —— 它用一个 LLM 来做路由判断。
核心 idea：
	 给一个 7B 的 LLM（Qwen2.5-7B-Instruct）注入"模型能力理解能力"，让它读一段模型描述后，判断该模型能否正确回答这个问题（输出 Yes/No）。
输入 Prompt 结构:
```
	<|im_start|>system
	You are a helpful assistant.<|im_end|>
	<|im_start|>user
	[模型能力向量 — 嵌入到 token 位置]
	[用户的具体问题]
	Based on the model description and the test sample, 
	predict whether the model can handle test sample 
	by indicating 'Yes' or 'No'.<eos>
	<|im_start|>assistant
	Yes / No
```

#### 二、模型架构（CompositeModel） 

MODEL-SAT 由三个可训练组件组成：
┌─────────────────────────────────────────────────────┐
│                 CompositeModel                       │
│                                                     │
│  ① Embedding Model (gte_Qwen2-7B-instruct, 7B)     │
│     └─ 输入：模型能力描述文本（自然语言）              │
│     └─ 输出：模型能力向量 [D=3584]                    │
│                                                     │
│  ② Projector (MLP 或 Linear)                         │
│     └─ 输入：模型能力向量 [3584]                      │
│     └─ 输出：LLM token 维度 [3584]                   │
│     └─ 将模型描述 embedding 投影到 LLM 的 token 空间  │
│                                                     │
│  ③ Big Model (Qwen2.5-7B-Instruct, 7B)              │
│     └─ 输入：包含模型能力向量的 token 序列             │
│     └─ 输出：Yes/No 的 logits                        │
└─────────────────────────────────────────────────────┘
##### 2.1 Embedding Model — 模型能力编码器
用 gte_Qwen2-7B-instruct（也是 7B！）来编码每个候选模型的文字描述。
模型描述是什么？ 由 generate_model_descriptions.py 自动生成，基于训练数据中该模型在各个类别上的准确率：
"The model achieves accuracy 45.2% on the task of biology, 
accuracy 67.8% on the task of computer science, 
accuracy 32.1% on the task of history, ..."
这些描述被编码成 3584 维的向量。
==Comment: 这里的类别可以换成聚类结果==
##### 2.2 Projector — 空间对齐
将 embedding model 输出的 3584 维向量投影到 Qwen2.5 LLM 的 token embedding 空间（也是 3584 维）。
两种选择：
- Linear：单层线性变换 Linear(3584 → 3584, no bias)
- Nonlinear（默认）：MLP = RMSNorm → Linear(in→out) → [Linear(out→hidden) → SiLU → Linear(hidden→out)] + Residual
##### 2.3 向量注入机制（关键技巧）
这是 MODEL-SAT 最巧妙的部分。它利用 Qwen2 tokenizer 中的一个特殊占位 token（ID = 151662），在输入序列的特定位置"替换"为模型能力向量：
```
# 构造输入时，在模型描述位置放一个占位 token
icl_vector_pad_tensor = torch.tensor([[151662]])  # 占位符
input_ids = cat([
    template_tokens,           # "<|im_start|>system\n..."
    icl_vector_pad_tensor,     # ← 占位符，之后会被替换
    sep_tokens,                # "\n\n"
    query_tokens,              # 用户问题
    instruction_tokens,        # "Based on the model description..."
    assistant_tokens,          # "<|im_start|>assistant\n"
    output_tokens              # "Yes" 或 "No"
])
# 前向传播时，替换占位符
model_input = embedding_layer(input_ids)  # 正常 token embedding
mask = (input_ids == 151662)              # 找到占位符位置
# 将占位符位置的 embedding 替换为 Projector 输出的模型能力向量
model_input[mask] = projected_model_vector
所以 LLM 实际看到的不是"某个 token"，而是一个经过 Projector 投影的、蕴含模型能力信息的连续向量。这类似于多模态模型中"把图像 patch 嵌入到语言模型 token 位置"的做法。
```
---
#### 三、训练流程
##### 3.1 数据准备（三步）
Step 1: Adaptor 生成原始数据
python -m baselines.adaptors.modelsat_adaptor \
    --config config/baseline_config.yaml \
    --seed 42 --split-ratio 0.8
输出 original_data/subset_logs_42/ 下每个 dataset 一个 `{dataset}_train.json` 和 `{dataset}_test.json`。
每条记录格式：
```
{
    "index": 0,
    "question": "Answer the following biology question...",
    "scores": {
        "gpt-4o": 1.0,
        "claude-3-5-sonnet": 1.0,
        "llama-3.1-8b": 0.0,
        ...
    }
}
```
Step 2: 合并数据集
sh ./scripts/construct_dataset.sh
将所有 dataset 的 train/test 合并成统一的 data/seed42/train.json 和 data/seed42/test.json。同时处理 OOD（只有 test 没有 train 的 dataset）。
每条输出记录变为：
```
{
    "query": "Answer the following biology question...",
    "model": "gpt-4o",
    "is_correct_sc": true,
    "task": "mmlupro",
    "index": 0
}
```
注意：每个 (query, model) 对是一条独立的训练样本。如果有 33 个模型和 1000 个 query，就有 33,000 条训练记录。
Step 3: 生成模型描述
sh ./scripts/generate_model_description.sh
基于 MMLUPro 训练集的分类别准确率，==为每个模型生成一句自然语言描述==。输出 data/seed42/model_description.json。
==Comments: 描述怎么生成的？==什么模型？怎么给任务归类？
##### 3.2 训练策略 — 两阶段渐进解冻
这是 MODEL-SAT 训练中最有特色的设计：
阶段 1（前 3000 步）: 只训练 Projector
    ├── Big Model (Qwen2.5-7B)     → 冻结 ❄️
    ├── Embed Model (gte-7B)        → 冻结 ❄️
    └── Projector (MLP)             → 训练 🔥  lr=1e-5
阶段 2（3000 步之后）: 全部解冻
    ├── Big Model (Qwen2.5-7B)     → 训练 🔥  lr=1e-6
    ├── Embed Model (gte-7B)        → 训练 🔥  lr=1e-6
    └── Projector (MLP)             → 继续训练 🔥  lr=1e-5
为什么这样做？
- 阶段 1：让 Projector 先学会"把 embedding 空间映射到 LLM 能理解的空间"
- 阶段 2：Projector 稳定后，微调 LLM 和 Embedding Model，让整个系统端到端优化
损失函数：标准的 CrossEntropyLoss（Next Token Prediction），只在 "Yes"/"No" token 位置计算 loss。
3.3 训练配置
```
# 默认配置（train.sh）
base_model: Qwen2.5-7B-Instruct    # 主 LLM
embed_model: gte_Qwen2-7B-instruct  # Embedding 模型
projector_type: nonlinear            # MLP Projector
batch_size: 4
max_length: 1024
proj_lr: 1e-5                        # Projector 学习率
llm_lr: 1e-6                         # LLM 学习率（阶段2）
embed_lr: 1e-6                       # Embed 学习率（阶段2）
llm_unfreeze_step: 1000              # 解冻 step（默认脚本写 1000，代码默认 3000）
num_epochs: 5
num_gpus: 8                          # 默认 8 卡！
deepspeed: ZeRO Stage 2
warmup: 10% of total steps
scheduler: WarmupCosineLR
```
---
#### 四、推理决策
对每个 query，遍历所有候选模型：
```
for model in all_candidate_models:
    # 1. 用 Embed Model 编码模型描述 → 模型能力向量
    model_vector = embed_model(model_description[model])
    
    # 2. Projector 投影
    projected = projector(model_vector)
    
    # 3. 构造 prompt：[模型能力向量占位] + 用户问题 + "Yes or No?"
    # 4. LLM 输出 "Yes" 的概率
    yes_prob = softmax(logits)[yes_token_id]
    
    results[model] = yes_prob
# 选 yes_prob 最高的模型
selected = argmax(results)
```
评估指标（代码中的 evaluate_metrics）：
- 按 (task, index) 分组w
- 每组中选 yes_prob 最高的那个 (model, is_correct) 对
- 如果该 model 确实答对了（is_correct_sc = True），算正确
- 最终计算 per-task accuracy，再取平均
---
#### 五、数据流全景
Step 1: Adaptor
    LLMRouterBench 原始结果 
    → per-dataset JSON {question, scores: {model: score}}
Step 2: construct_dataset.py
    per-dataset JSONs 
    → 统一 train.json / test.json {query, model, is_correct_sc, task, index}
Step 3: generate_model_descriptions.py
    训练数据中的分类别准确率 
    → model_description.json {"gpt-4o": "The model achieves accuracy 45.2% on..."}
Step 4: model_sat_train.py
    train.json + model_description.json + Qwen2.5-7B + gte-7B 
    → 训练 CompositeModel → checkpoints/

#### 六、训练难度
![[Pasted image 20260308202725.png]]


### RouteDC
##### 三个关键组件
1. Backbone（文本编码器）：gte_Qwen2-7B-instruct
- 这个和 MODEL-SAT / FrugalGPT 用的是同一个 7B 编码器
- 输入 tokenized question，输出 last_hidden_state
- Pooling 方式：last-token pooling（不是 CLS token，不是 mean pooling）
  - 对于 right-padding 模型：取每条样本最后一个非 padding token 的 hidden state
  - sequence_lengths = (attention_mask.sum(dim=1) - 1) → 取对应位置的向量
- 输出维度：3584（gte_Qwen2-7B 的 hidden_size）
- 全部参数可训练（param.requires_grad = True）
2. LLM Embeddings：nn.Embedding(node_size, 3584)
- 每个候选 LLM 一个 3584 维的可学习向量
- 初始化：高斯初始化，mean=0, std=0.78
- node_size = 候选模型数量（从训练数据的 scores 字典自动推断）
- 这就是论文中每个 LLM 的 "representation"
3. 相似度计算
similarity = (hidden_state @ embeddings.T) / (‖hidden_state‖ × ‖embeddings‖)
- 默认用 余弦相似度（similarity_function="cos"）
- 也支持内积（"inner"）
- 输出：[batch_size, node_size] 的 logits 矩阵，除以温度 t
推理很简单：argmax(softmax(logits)) → 选相似度最高的模型。

##### 📐 训练损失："双对比学习"（Dual Contrastive Learning）
这是 RouterDC 最核心的创新。训练使用三个损失函数的加权和：
```
Total Loss = L_sample_llm + α × L_sample_sample + β × L_cluster
```
损失 1：compute_sample_llm_loss（Sample-LLM 对比损失）— 主损失
> "对每个 query，让它更靠近能答对的模型，远离答不对的模型"
```
def compute_sample_llm_loss(x, index_true, top_k, last_k):
    # index_true = 每个模型在这道题上的真实得分
    # top_k = 取最好的 k 个模型作为正样本 (default=3)
    # last_k = 取最差的 k 个模型作为负样本 (default=3)
    
    for i in range(top_k):
        positive = 第 i 好的模型的 logit
        negatives = 最差的 last_k 个模型的 logit
        # 如果负样本实际分数>0，设为 -inf 排除
        loss += -log(softmax(positive vs negatives))
```
关键细节：
- 对分数=0 的正样本跳过（mask = where(score > 0, 1, 0)）—— 如果"最好的模型"实际分数也是0，说明没有模型答对，不算 loss
- 对负样本中分数>0.5 的排除（设为 -inf）—— 防止把实际答对的模型当负样本
- 这本质上是 InfoNCE loss 的变体
- 
损失 2：compute_sample_sample_loss_with_task_tag（Sample-Sample 对比损失）

> "来自同一个 dataset 的 query 应该在表示空间中更近，不同 dataset 的更远"
对每个 query:
    positive = 同一 dataset_id 的随机一个 query
    negatives = 不同 dataset_id 的 H 个随机 query (H=3)
    loss += -log(softmax(sim(query, positive) vs sim(query, negatives)))
    
- 在 hidden_state 空间（不是 logits）做对比
- 目的：让编码器学到 task-aware 表示，同类型题目聚在一起
- 权重由 --sample_loss_weight 控制（默认 0，即默认不启用）

损失 3：compute_cluster_loss（Cluster 对比损失）
> "来自同一 K-means 聚类的 query 应该更近"
对每个 query:
    positive = 同一 cluster_id 的随机一个 query
    negatives = 不同 cluster_id 的 H 个随机 query (H=3)
    loss += -log(softmax(sim(query, positive) vs sim(query, negatives)))
    
- 和 Sample-Sample 损失结构相同，但用 K-means 预分配的 cluster_id 而非 dataset_id
- 权重由 --cluster_loss_weight 控制（默认 0，即默认不启用）
- 聚类 ID 由 adaptor 在数据预处理阶段通过 K-means 分配
"Dual Contrastive"名字的来源：论文中指的是 Sample-LLM 对比 + Sample-Sample 对比，这两个维度的对比学习。
##### Comments
部分解决了avenger简单聚类的问题

### HybridLLM/Best-Route
基本算法：
![[Pasted image 20260309035412.png]]
Loss function （二分类/多分类）
![[Pasted image 20260309035527.png]]