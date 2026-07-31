# SuperClaw LLMRouter 算法笔记

> 代码路径：`integrations/llmrouter/custom_routers/`
> 当前默认路由器：**LatentFactorRouter**（`serve.yaml` 中配置）

---

## 目录

1. [整体架构](#整体架构)
2. [训练数据：来源与格式](#训练数据来源与格式)
3. [LatentFactorRouter](#latentfactorrouter)
4. [ComplexityRouter](#complexityrouter)
5. [SuperRouter](#superrouter)
6. [CustomLLMRouter](#customllmrouter)
7. [四种路由器总结](#四种路由器总结)

---

## 整体架构

SuperClaw 实现了 4 个不同的路由算法，全部读取同一格式的离线 benchmark 数据，服务于同一个推理入口 `pii_serve.py`：

```
用户请求 (model="auto")
    ↓
pii_serve.py → router_adapter.router.route_single({"query": user_query})
    ↓
predicted_llm (qwen* → local-model, gpt* → cloud-model)
    ↓
转发到对应模型
```

4 个路由算法：

| 算法                     | 文件                    | 核心思路                                |
| ---------------------- | --------------------- | ----------------------------------- |
| **LatentFactorRouter** | `latentfactorrouter/` | Ridge 矩阵分解：`E @ Lp ≈ P`             |
| **ComplexityRouter**   | `complexityrouter/`   | IRT：先估难度 δ，再查模型能力 β                 |
| **CustomLLMRouter**    | `customllmrouter/`    | LLM-as-judge：让本地模型做路由决策             |
| **SuperRouter**        | `superrouter/`        | Intent + KMeans 聚类，per-cluster 模型签名 |

---

## 训练数据：来源与格式

### 数据格式（JSONL，wide format）

每条记录对应一个 query，记录所有模型在该 query 上的得分和费用：

```json
{
  "query": "Solve: let x,y,z satisfy log2(x/yz)=1/2 ...",
  "dataset": "aime",
  "records": {
    "gpt-5":          1.0,
    "qwen3-coder-next": 1.0
  },
  "usages": {
    "gpt-5":          {"cost": 0.01516, "completion_tokens": 1490, "prompt_tokens": 209},
    "qwen3-coder-next": {"cost": 0.0,   "completion_tokens": 1483, "prompt_tokens": 214}
  }
}
```

- `records[model]`：**performance 真值**，0.0 或 1.0，程序自动对比模型答案与 ground truth 打分
- `usages[model].cost`：**cost 真值**，该次 API 调用的美元费用

### 数据来源

1. 离线跑 benchmark（AIME 数学竞赛、HumanEval、SWE-bench-verified 等）
2. 每道题有标准答案，程序直接比较得出 0/1 分
3. `data_adaptor.py` 把 benchmark JSON 结果转成上述 JSONL 格式

**关键约束**：必须是有客观标准答案、可以自动评分的任务。

### 矩阵化

从 JSONL 构建三个矩阵（`extract_matrices`）：

```
E[i, :]  = query i 的 embedding 向量         shape: (n_queries, embed_dim)
P[i, j]  = LLM j 在 query i 上的得分（0/1）  shape: (n_queries, n_llms)，稀疏含 NaN
C[i, j]  = LLM j 在 query i 上的费用（$）    shape: (n_queries, n_llms)，稀疏含 NaN
```

P 和 C 是**稀疏矩阵**——不是每条 query 都跑了所有模型，未跑的位置填 NaN。

### 为什么不做在线更新？

1. **无 ground truth**：chat 场景（"帮我重构代码"）没有标准答案，无法自动打分
2. **标签延迟**：路由决策是毫秒级，用户反馈可能是分钟/天后
3. **矩阵更新代价高**：ridge 矩阵求逆是批量运算，在线更新数学上复杂
4. **框架设计**：离线训练 → 保存 pkl → 服务启动 load → 只读推理

---

## LatentFactorRouter

> 文件：`latentfactorrouter/router.py` + `latent_factor_model.py`
> 配置：`config_test.yaml`（`perf_weight=0.95`，`latent_dim=32`，`pca_n_components=0.99`）

### 核心思想

学两个线性映射矩阵，使得 query embedding 可以直接预测所有模型的 performance 和 cost：

```
E @ Lp ≈ P      (embedding → 每个模型的 performance)
E @ Lc ≈ log(C) (embedding → 每个模型的 log-cost)
```

### Lp 和 Lc 的结构

```
Lp: shape (embed_dim + 1, n_llms)   ← +1 是 bias 列
Lc: shape (embed_dim + 1, n_llms)
```

- **不是**每个模型有独立的 Lp/Lc，而是两个共享矩阵
- 每个模型对应 Lp/Lc 的**一列**，共享同一个 embedding 空间
- 训练时对每列**分别**做 ridge regression（各自只用有观测值的行）

### 训练阶段

#### 预处理

- Cost 取 log 变换：`log(C + eps)`，原因：cost 是长尾分布，log 空间更线性
- TargetScaler 对 P 和 log(C) 做 z-score 标准化

#### Ridge Regression（核心）

对 Lp 的每一列（每个 LLM j）独立求解：

```
Lp[:, j] = (Eᵀ_obs E_obs + αI)⁻¹ Eᵀ_obs P_obs[:, j]
```

- `E_obs`：只取 P[i,j] 有观测值（非 NaN）的行
- 这样不同模型用不同的有效样本集训练
- 同理求 Lc 的每一列

#### 两阶段超参数搜索（LatentFactorModelWithCV）

**Phase 1 — SVD + 粗搜索**：
1. 对 E 做一次 SVD：`E = UΣVᵀ`（只算一次）
2. 遍历 log-spaced 的 `latent_dim` 候选值 × alpha 网格
3. 5-fold CV，评估指标：`(RMSE_P + RMSE_C) / 2`

**Phase 2 — 三分法精搜索（ternary search）**：
- 在 Phase 1 最优区间里，在 log 尺度上三分搜索精确最优 `d`，约 14 次评估
- 速度比暴力搜索快 O((d/D)³) 倍

最终用全量数据以最优 `(d, α)` 重新拟合一次，得到最终 Lp 和 Lc。

#### 可选：残差 Boosting

若开启 `enable_residual_boost`：

```
P_final = E @ Lp + GBM_P(E)    (HistGradientBoostingRegressor 拟合残差)
C_final = E @ Lc + GBM_C(E)
```

捕捉 ridge 覆盖不到的非线性模式。

### 推理阶段

```python
E_aug = [embed(query) | 1]         # 加 bias 维
P_pred = E_aug @ Lp                # shape: (1, n_llms)，一次矩阵乘法
C_pred = exp(E_aug @ Lc)           # 反 log 变换

# 归一化
p_norm = P_pred / P_pred.max()
c_norm = log(C_pred) / log(C_pred).max()

# Balance score
balance[j] = perf_weight * p_norm[j] - cost_weight * c_norm[j]

# 返回 argmax
predicted_llm = available_models[argmax(balance)]
```

### 特点总结

| 项目 | 值 |
|------|-----|
| 推理速度 | 极快（两次矩阵乘法） |
| 可解释性 | 低（黑盒矩阵） |
| 新模型冷启动 | 需要新增一列并重新训练 |
| 核心假设 | query embedding 可以线性预测各模型表现 |

---

## ComplexityRouter

> 文件：`complexityrouter/router.py` + `complexity_model.py`
> 配置：`config_test.yaml`（`two_pl=true`，`n_iter=400`，`n_knn=15`，`knn_blend_weight=0.6`，`complexity_threshold=0.5`）

### 核心思想

把路由问题转化为**难度匹配**问题：

> query 有难度 δ，模型有能力 β。能力够了选便宜的，不够才选贵的。

与 LatentFactorRouter 的根本区别：不直接从 embedding 预测各模型表现，而是先把 query 压缩成一个标量（难度 δ），再用 IRT 公式估计各模型在该难度下的表现。

### 训练阶段（4 步）

#### Step 1：IRT 估计隐变量 δ 和 β

**关键问题**：δ（难度）不在数据里，它是隐变量，需要从 P 矩阵推断。

**IRT 的直觉**：

```
         gpt-5  qwen3  claude  llama
query1 [  1.0    1.0    1.0    1.0  ]  ← 全对 → 简单 → δ 小
query2 [  1.0    0.0    1.0    0.0  ]  ← 强模型对，弱模型错 → 中等难度
query3 [  0.0    0.0    0.0    0.0  ]  ← 全错 → 很难 → δ 大
```

**IRT 2PL 模型**：

```
P(LLM j 答对 query i) = σ(a_j * (β_j - δ_i))
```

- `δ_i`：query i 的难度（越大越难）
- `β_j`：LLM j 的能力（越大越强）
- `a_j`：LLM j 的区分度（越大对难度差异越敏感）
- `σ`：sigmoid 函数

**训练**：对 P 矩阵做梯度上升（最大化对数似然），400 次迭代，同时估出所有 δ 和 β。

**初始化（避免随机初始化的不稳定）**：

```
δ_i = -logit(mean_row_i)   ← 行均值低（大家都答错）→ 难度高
β_j =  logit(mean_col_j)   ← 列均值高（模型答对多）→ 能力强
```

#### Step 2：训练 E→δ 复杂度预测器

IRT 给了训练集每条 query 的 δ 真值，但推理时新 query 没有 δ，需要从 embedding 预测。

**把 IRT 估出的 δ 当作标签，训练 ridge regression**：

```
W = (EᵀE + αI)⁻¹ Eᵀ δ
```

α 通过 5-fold CV 在 `[1e-4, 1e2]` log-spaced 网格上选最优。

**两步的逻辑关系**：
- Step 1：从 P 矩阵"挖掘"δ 作为标签（IRT 是挖掘工具）
- Step 2：用 δ 监督训练预测器，让 δ 可以泛化到新 query

#### Step 3：建 KNN 查找表

存储所有训练 query 的 **L2 归一化 embedding** + 对应 P 矩阵：

```python
# 推理时
sims = E_train_normed @ query_normed        # 余弦相似度
top_k = argpartition(sims, -k)[-k:]
weights = softmax(top_sims * 5.0)           # ×5 放大差异，让最近邻主导
knn_perf[j] = Σ weights * P_train[top_k, j]
```

#### Step 4：建复杂度 Bin 统计表

把训练 queries 按 δ 分成 5 个**等频 bin**，每个 bin 内对每个 LLM 计算平均 perf 和 cost：

```
bin_perf[b, j] = mean(P[δ ∈ bin_b, j])
bin_cost[b, j] = mean(C[δ ∈ bin_b, j])
bandwidth = 平均 bin 宽度 × 1.5   (Gaussian kernel 插值用)
```

### 推理阶段

```python
E = embed(query)
δ_raw = E @ W                          # 复杂度预测器
norm_delta = (δ_raw - δ_min) / (δ_max - δ_min)   # 归一化到 [0,1]
```

**快速路径（SuperClaw 实际使用）**：

```python
if norm_delta > complexity_threshold (0.5):
    chosen = cloud_llm     # 复杂 → 云端强模型
else:
    chosen = local_llm     # 简单 → 本地轻模型
```

**全量排序路径（top_k > 1 时）**：

对每个候选模型预测 performance，融合三个信号：

```
# 信号1：IRT 公式（泛化能力强，对新 query 也能算）
irt_perf  = σ(a_j * (β_j - δ_raw))

# 信号2：KNN（分布内精度高，直接查训练数据）
knn_perf  = softmax加权邻居的观测 P

# 融合（KNN 占主导）
final_perf = 0.6 * knn_perf + 0.4 * irt_perf

# 信号3：cost 用 Gaussian kernel 插值（平滑，避免 bin 硬边界）
weights[b] = exp(-0.5 * ((δ - bin_center[b]) / bandwidth)²)
cost = Σ weights[b] * bin_cost[b, j] / Σ weights[b]

# 最终排序
balance = perf_weight * perf - cost_weight * cost
按 (-perf, cost) 排序返回 top_k
```

### Cold-start：5~20 条 probe query 注册新模型

不需要重训练，只需：
1. 从训练时选出的 probe set（KMeans 覆盖全 complexity 空间的代表性 queries）取 5~20 条
2. 让新模型跑这些 queries，拿到 0/1 分
3. 固定所有 δ，对新模型的 β 做 **MAP 1D 优化**：

```
β* = argmax Σ_k [X_k * log σ(β - δ_k) + (1-X_k) * log(1 - σ(β - δ_k))] - α * β²
```

4. 注册到 `_registered_models`，立即可用于路由

### 与 LatentFactorRouter 对比

| 维度 | LatentFactorRouter | ComplexityRouter |
|------|-------------------|-----------------|
| 预测 perf 的方式 | `E @ Lp`（线性投影） | IRT `σ(a*(β-δ))` + KNN 融合 |
| 信息压缩路径 | embedding → n_llms 个 perf（高维到高维） | embedding → 标量 δ → perf |
| 可解释性 | 低（黑盒矩阵） | 高（δ=难度，β=能力，直观） |
| 新模型冷启动 | 需重新训练新列 | 只需估一个 β（5~20 条 probes） |
| 核心假设 | embedding 直接线性预测各模型表现 | 模型表现由复杂度决定，复杂度是一维的 |
| 推理速度 | 极快（矩阵乘法） | 快（标量预测 + 公式） |

### 完整训练流程图

```
P 矩阵 (谁答对了谁)
  ──IRT (400次梯度上升)──→  δ[n_queries]（每条 query 隐含难度）
                             β[n_llms]  （每个 LLM 隐含能力）
                             a[n_llms]  （每个 LLM 区分度）
                    ↓
                  δ 当标签
  ──Ridge CV──→  W（embedding → 难度预测器，5-fold CV 选 α）
                    ↓
  ──存储──→  KNN 表（L2-normed E_train + P_train）
             Bin 表（bin_perf, bin_cost, Gaussian bandwidth）
                    ↓ 保存 artefacts (.pkl)

推理:
  embed(query) → W → δ_raw → normalize → norm_delta
      ↓ 快速路径                    ↓ 全量路径
  norm_delta > 0.5?        IRT σ(a*(β-δ)) + KNN blend → perf
    yes → cloud_llm        Gaussian kernel interp → cost
    no  → local_llm        balance sort → top_k
```

---

## SuperRouter

> 文件：`superrouter/router.py`, `trainer.py`, `cluster_backends.py`, `updater.py`
> 配置：`top_k=3`，`beta=9.0`，`perf_weight=0.7`，`optimal_k_method=silhouette`

### 核心思想

把路由问题变成**按意图聚类，再在每个簇里比较模型签名**的问题：

> 不同类型的 query（代码/数学/通用问答），模型的相对优劣是不同的。
> 先把 query 空间按意图分组，再在每组内、每个 embedding 聚类中，记录每个模型的平均 accuracy 和 cost，推理时加权找最优模型。

与前两个路由器的根本区别：**没有全局线性映射**，而是**局部聚类+统计签名**，每个 intent×cluster 单元独立维护一张模型成绩单。

---

### 关键数据结构

```python
@dataclass
class ModelSignature:
    model_name: str
    accuracy:   float   # 该 cluster 内该模型的平均 0/1 准确率
    avg_cost:   float   # 该 cluster 内该模型的平均 API 费用 ($)
    n_queries:  int     # 该 cluster 内有效观测数

@dataclass
class ClusterModel:
    # 每个簇的归一化上界（acc_max, cost_max）
    norm_params: Dict[int, Dict[str, float]]
    # { cluster_id: {"acc_max": float, "cost_max": float} }

    # 核心索引：每个簇里每个模型的表现签名
    signatures:  Dict[int, Dict[str, ModelSignature]]
    # { cluster_id: { model_name: ModelSignature } }

    cluster_quality_metrics: ClusterQualityMetrics
    # silhouette / davies_bouldin / calinski_harabasz
```

全局索引：`_cluster_models: Dict[str, ClusterModel]`，按 intent 分组：

```
{
  "coding":  ClusterModel(k=32 个簇),
  "math":    ClusterModel(k=24 个簇),
  "general": ClusterModel(k=16 个簇),
}
```

---

### 训练流水线（5 步）

#### Step 1：prepare_data

```python
E       = embedder.embed(all_queries)           # 嵌入所有 query
intents = [classify_intent(q) for q in queries] # 意图分类
train, val, test = stratified_split(            # 按 (dataset × intent) 分层三路分割
    data, intents, ratio=[0.7, 0.15, 0.15])
```

stratified split 的意义：保证三个集合里各 intent 和 dataset 的比例相同，避免某个 intent 的 query 全跑到 test 集造成评估偏差。

#### Step 2：fit_clusters（per intent）

**直觉**：把每个 intent 下的 query embedding 在空间里聚成若干簇，相似语义的 query 归到同一簇。

```python
for intent in ["coding", "math", "general", ...]:
    E = get_embeddings(train_data[intent])      # shape: (n, embed_dim)
    E_normed = E / norm(E, axis=1, keepdims=True)  # L2 归一化

    k = find_optimal_k(E_normed, method=optimal_k_method)
    backend = build_backend(method="kmeans", n_clusters=k)
    backend.fit(E_normed)
    _cluster_models[intent].backend = backend
```

**backend 是什么**：`cluster_backends.py` 对多种 sklearn 聚类算法的统一包装，对外暴露同一接口：

- `backend.fit(X)` — 在训练数据上拟合
- `backend.predict(X)` — 给新的点分配到最近的簇
- `backend.centers` — 每个簇的质心向量（L2 归一化），shape: (k, embed_dim)

不同聚类算法接口差异很大（有的没有 `predict()`，有的连 k 都不用指定），backend 层把这些差异全部抹平。

**支持的 backend 类型**：

| method key | sklearn 类 | 特点 |
|---|---|---|
| `kmeans`（默认） | KMeans | 快，需预设 k，有 predict |
| `minibatch` | MiniBatchKMeans | 大数据集首选 |
| `bisecting` | BisectingKMeans | 层次化 KMeans |
| `agglomerative` | AgglomerativeClustering | 无 predict，用最近质心推理 |
| `spectral` | SpectralClustering | 图聚类，慢 |
| `dbscan` / `hdbscan` / `optics` | DBSCAN 系列 | k 自动推断，噪点归入最近簇 |
| `gaussian` | GaussianMixture | 软分配转硬分配 |

**find_optimal_k 的三种方法**：

| 方法 | 原理 | 特点 |
|---|---|---|
| `silhouette`（默认） | 遍历候选 k，取 silhouette score（簇分离度）最高的 | 质量最好 |
| `elbow` | 对 inertia 曲线求二阶导，找曲率最大点 | 快，偶尔不稳定 |
| `gap_statistic` | 与随机 uniform 数据对比，找 gap 最大的 k | 统计严谨，最慢 |

k 的范围 clamped 到 `[2, min(optimal_k_max, n_samples-1)]`。

#### Step 3：build_signatures（在 val 集上）

聚类训好后，在 val 集上统计每个簇内每个模型的平均准确率和平均费用：

```python
for intent, cm in _cluster_models.items():
    val_normed  = l2_normalize(get_embeddings(val_data[intent]))
    cluster_ids = cm.backend.predict(val_normed)  # 每条 val query 归属哪个簇

    for cid in range(k):
        for model in available_models:
            subset = [i for i, c in enumerate(cluster_ids) if c == cid]
            accs   = [P[i][model] for i in subset if P[i][model] is not NaN]
            costs  = [C[i][model] for i in subset if C[i][model] is not NaN]
            cm.signatures[cid][model] = ModelSignature(
                accuracy  = mean(accs),
                avg_cost  = mean(costs),
                n_queries = len(accs),
            )

    # 存归一化上界
    for cid in range(k):
        cm.norm_params[cid] = {
            "acc_max":  max(sig.accuracy for sig in cm.signatures[cid].values()),
            "cost_max": max(sig.avg_cost  for sig in cm.signatures[cid].values()),
        }
```

举例（k=3，两个模型）：

```
簇 0（简单问答类）:  gpt-5: acc=0.91, cost=$0.012 | qwen3: acc=0.89, cost=$0.001
簇 1（复杂数学类）:  gpt-5: acc=0.85, cost=$0.015 | qwen3: acc=0.42, cost=$0.001
簇 2（代码调试类）:  gpt-5: acc=0.78, cost=$0.014 | qwen3: acc=0.71, cost=$0.002
```

**为什么用 val 集**：train 集拟合了聚类，在上面算签名会偏乐观；val 集独立，签名更真实。

**norm_params 设计动机**：只存原始数值，推理时才乘 perf_weight/cost_weight。改 YAML 权重配置无需重训练。

#### Step 4：evaluate（test 集）

```
accuracy               ← route 选的模型在该 query 上的实际得分
avg_cost               ← route 选的模型在该 query 上的实际费用
oracle_gap             ← 与"每次都选最优模型"的差距
vs_best_single         ← 与"永远选固定最优单一模型"的差距
cost_savings_vs_best   ← 相对固定最优模型节省了多少费用
quality_cost_score     ← accuracy / avg_cost（综合性价比）
```

#### Step 5：save_artefacts

```python
pickle.dump({
    "_cluster_models":   _cluster_models,
    "_val_data":         val_data,      # ← 关键：保存 val 集供 updater 用
    "_available_models": available_models,
}, "saved_models/superrouter/superrouter_artefacts.pkl")
```

---

### 推理阶段（_route_query）

```python
def _route_query(self, query, intent):
    cm = _cluster_models.get(intent) or _cluster_models.get("general")

    # 嵌入 + L2 归一化
    q_emb  = embed([query])[0]
    q_emb /= norm(q_emb) + 1e-12

    # 余弦距离到所有簇中心
    dists   = 1.0 - cm.centers @ q_emb      # shape: (k_clusters,)
    top_idx = argsort(dists)[:top_k]         # 最近的 top_k 个簇

    # Softmax 加权（beta=9.0 → 很尖锐，几乎只看最近的簇）
    logits = -beta * dists[top_idx]
    probs  = softmax(logits)

    # 加权聚合每个模型的 balance score
    agg = {}
    for cid, prob in zip(top_idx, probs):
        for model, sig in cm.signatures[cid].items():
            score = _balance_score(sig, cm.norm_params[cid])
            agg[model] = agg.get(model, 0) + prob * score

    return sort_descending(agg)[0]   # balance score 最高的模型
```

**_balance_score 计算**：

```python
def _balance_score(sig, norm_params):
    if budget_limit and sig.avg_cost > budget_limit:
        return 0.0                              # 硬成本上限

    norm_acc  = sig.accuracy / norm_params["acc_max"]
    norm_cost = sig.avg_cost / norm_params["cost_max"]

    if norm_acc < min_acc_thresh:
        return 0.0                              # 硬精度下限

    return perf_weight * norm_acc + cost_weight * (1.0 - norm_cost)
    #                                            ^^^^^ cost 取反：越低越好
```

与 LatentFactorRouter 的对比：LFR 是 `perf_weight*p - cost_weight*c`，SuperRouter 是 `perf_weight*norm_acc + cost_weight*(1-norm_cost)`，数学等价，但 SuperRouter 分数在 [0,1] 更规整。

**beta 参数直觉**：
- `beta=9.0`（默认）→ softmax 极尖，几乎等于"硬分配到最近簇"
- `beta=1.0` → softmax 平滑，top-k 簇权重接近均匀

**推理示例**（query="帮我写一个快速排序"，intent="coding"）：

```
embed → L2-normalize → q_emb
dists = 1 - centers @ q_emb  →  [0.05, 0.18, 0.12, ...]
top_idx = [0, 2, 1]  (最近3个簇)
logits  = -9 * [0.05, 0.12, 0.18]  =  [-0.45, -1.08, -1.62]
probs   = softmax(logits)           =  [0.62,  0.25,  0.13]

agg["gpt-5"]  = 0.62×0.81 + 0.25×0.78 + 0.13×0.85 = 0.808
agg["qwen3"]  = 0.62×0.74 + 0.25×0.65 + 0.13×0.70 = 0.716

predicted_llm = "gpt-5"
```

---

### Updater：不重训练地添加新 LLM

**核心洞察**：聚类只依赖 query embedding，与模型无关。添加新模型只需重新校准签名，不需要重新跑聚类。

**路径 A（推荐，val 集对齐）**：

```
用户提供新模型在部分 query 上的 eval_data
  ↓
与 _val_data 里的 query 文本对齐（按 query text 匹配）
未覆盖的 val query → intent 级均值填充（imputation）
  ↓
对每个 ClusterModel：predict cluster_id → 聚合 acc/cost → 写入 signatures
重算 norm_params（新模型可能改变 acc_max/cost_max）
  ↓
新模型加入 _available_models，保存 artefact
```

好处：所有模型共用同一批 val query，簇分配完全一致，公平对比。

**路径 B（fallback）**：artefact 不含 `_val_data` 时，直接嵌入 eval_data 里的 query，分配到最近簇。

---

### 与其他路由器的横向对比

| 维度 | LatentFactor | ComplexityRouter | SuperRouter |
|------|-------------|-----------------|------------|
| **路由粒度** | 全局线性映射 | 全局 δ-β 映射 | per-intent × per-cluster |
| **模型表示** | Lp/Lc 的列向量 | IRT β（标量能力） | 签名（acc + cost 统计量） |
| **可解释性** | 低 | 高（δ=难度） | 中（签名直观但聚类不可解释） |
| **intent 感知** | ✗ | ✗ | ✓ |
| **新模型冷启动** | 重训练 | 5~20 probes | eval data（不重聚类） |
| **权重可热更新** | ✗ | ✗ | ✓（norm_params 解耦） |
| **核心假设** | embedding 线性预测表现 | 表现由一维难度决定 | 同 intent 同 cluster 的模型排名稳定 |

---

### 完整训练流程图

```
JSONL 训练数据
  ↓ prepare_data
  embed + classify_intent + stratified split (train 70% / val 15% / test 15%)
  ↓ fit_clusters（per intent）
  L2-normalize embeddings
  → find_optimal_k (silhouette/elbow/gap_statistic)
  → build_backend (kmeans/...) → fit
  → ClusterModel[intent] = {backend, centers}
  ↓ build_signatures（在 val 集上）
  → predict cluster_id for each val query
  → per-cluster per-model: mean(acc), mean(cost) → ModelSignature
  → norm_params[cid] = {acc_max, cost_max}
  ↓ evaluate（在 test 集上）
  → route_batch → metrics (oracle_gap, vs_best_single, ...)
  ↓ save_artefacts
  → pickle {_cluster_models, _val_data, _available_models}

推理:
  query → embed → L2-normalize
    ↓
  IntentClassifier → intent
    ↓
  ClusterModel[intent].centers @ q_emb → top-k 最近簇
    ↓
  softmax(−beta × dist) → 簇权重 probs
    ↓
  Σ prob[k] × balance_score(signatures[k][model]) → per model score
    ↓
  argmax → predicted_llm
```

---

## CustomLLMRouter

> 文件：`customllmrouter/router.py`
> 配置：`timeout=15s`，无需训练数据，zero-shot

### 核心思想

直接让一个**本地 LLM 做路由决策**（LLM-as-judge）：

> 把"应该用哪个模型"这个问题，本身也当作一个 LLM 推理任务，让本地模型读 query 和候选模型描述，输出最优选择。

这是所有路由器里**唯一不需要训练数据**的方案。

### 工作流程

```python
def route_single(self, query_input):
    query = query_input["query"]

    # 1. 构建 routing prompt
    prompt = f"""You are a model router. Given the following query,
    select the most appropriate model from the candidates.

    Query: {query}

    Available models: {model_descriptions}

    Respond with ONLY the model name."""

    # 2. 调用本地 LLM（HTTP 请求，timeout=15s）
    response = http_post(local_llm_endpoint, prompt=prompt, timeout=15)

    # 3. 解析返回的模型名
    predicted = parse_model_name(response.text)

    # 4. fallback：解析失败时使用第一个可用模型
    if predicted not in available_models:
        predicted = available_models[0]

    return {"predicted_llm": predicted, ...}
```

### 特点与局限

| 项目 | 值 |
|------|-----|
| **训练需求** | 无（zero-shot） |
| **推理延迟** | 高（等一次 LLM 推理，15s timeout） |
| **可解释性** | 高（prompt 可读） |
| **灵活性** | 极高（修改 prompt 即改路由逻辑） |
| **稳定性** | 低（依赖 LLM 输出格式，需 fallback） |
| **新模型冷启动** | 极简（只改 prompt 里的模型描述） |

适用：路由逻辑复杂、无训练数据、latency 不敏感。
不适用：latency 敏感、路由比例高（路由本身开销不可忽略）。

---

## 四种路由器总结

| | LatentFactor | ComplexityRouter | SuperRouter | CustomLLMRouter |
|---|---|---|---|---|
| **算法范式** | 矩阵分解（Ridge） | IRT 隐变量 | 聚类 + 签名 | LLM-as-judge |
| **训练数据** | E、P、C 矩阵 | P 矩阵 | E、P、C + intent | 无 |
| **推理速度** | ⚡ 极快 | ⚡ 快 | ⚡ 快 | 🐢 慢 |
| **可解释性** | ❌ 低 | ✅ 高（δ=难度） | 🔶 中 | ✅ 高 |
| **Intent 感知** | ❌ | ❌ | ✅ | ✅（prompt） |
| **新模型冷启动** | 重训练 | 5~20 probes | eval data | 改 prompt |
| **核心假设** | 线性可预测 | 一维难度足够 | 局部簇内稳定 | LLM 理解路由 |
| **默认配置** | ✅ | ❌ | ❌ | ❌ |