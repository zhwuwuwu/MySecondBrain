> **结论先行**：截至 2026-08，`graph engineering` 更像是对既有工程实践的**行业命名/抽象层上移**，不是一个新框架或新算法。它描述的是：把多个 agent loop、确定性函数、工具、路由器、验证器和人类审批组织成显式的节点—边—状态系统。LangGraph 在 2024 年就已经把这套核心机制实现出来，并在 2026 年的官方回顾中明确说：这个名字是新的，但 graph-based agent engineering 已经做了三年。

`loop engineering` 和 `graph engineering` 不是二选一：**loop 是 graph 的最小特例；实际的 graph 节点内部仍然可以运行 loop**。真正的新变化不是“终于发明了图”，而是模型能力提高、长任务和多 agent 生产化后，大家开始把原来藏在一个大 agent loop 里的调度、状态、并行、恢复、审批和预算显式化。

## 目录

1. [这两个词到底指什么](#这两个词到底指什么)
2. [概念来源与时间线](#概念来源与时间线)
3. [Loop engineering：工程对象是一个 agent 的执行循环](#loop-engineering工程对象是一个-agent-的执行循环)
4. [Graph engineering：工程对象是多个过程之间的拓扑](#graph-engineering工程对象是多个过程之间的拓扑)
5. [Graph 和 loop 的严格关系](#graph-和-loop-的严格关系)
6. [LangChain 与 LangGraph 到底已经做到了什么](#langchain-与-langgraph-到底已经做到了什么)
7. [与 LangGraph 的差异：概念 vs 产品](#与-langgraph-的差异概念-vs-产品)
8. [是不是重新炒概念](#是不是重新炒概念)
9. [什么时候该用 loop，什么时候该上 graph](#什么时候该用-loop什么时候该上-graph)
10. [给 agent 开发者的工程检查表](#给-agent-开发者的工程检查表)
11. [Sources](#sources)

---

## 这两个词到底指什么

| 层次 | 主要问题 | 典型工程对象 | 不解决什么 |
|---|---|---|---|
| Prompt engineering | 一次模型调用怎么说得更好？ | prompt、few-shot、输出格式 | 多轮执行和失败恢复 |
| Context engineering | 这一次调用应该看到什么？ | 检索、记忆、摘要、上下文预算 | 跨节点调度 |
| **Loop engineering** | 一个 agent 怎么持续行动并可靠结束？ | 工具循环、观察、重试、验证、停止条件、外部状态 | 多个工作单元之间的拓扑 |
| **Graph engineering** | 多个 agent/函数/工具/人怎么协作？ | nodes、edges、state、fan-out/fan-in、审批、恢复、权限、预算 | 它本身不保证节点里的 agent 可靠 |

这里的 graph 不是 GraphRAG/knowledge graph：

- **knowledge graph** 组织的是知识实体和关系，回答“什么和什么有关”；
- **agent graph** 组织的是执行单元和转移，回答“谁在什么条件下做什么”；
- 两者可以放在同一个系统里，但不是同一个概念。

## 概念来源与时间线

这些词没有一个由标准组织维护的正式定义，时间线应该理解为“公共讨论开始流行的时间”，不是技术首次出现的时间。

| 时间 | 事件 | 判断 |
|---|---|---|
| 2024-09 | Anthropic《Building effective agents》总结 chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer 等模式 | 这些模式本身已经构成 agent graph 的基本形状，只是没有叫 graph engineering |
| 2025-09 | Simon Willison 讨论“designing agentic loops”是新的关键技能 | loop engineering 的思想早于这个词的爆发 |
| 2026-06 | Peter Steinberger、Boris Cherny 等人的观点被 Addy Osmani 等文章集中整理为 loop engineering | 从“手动逐轮 prompt”转向“设计自动运行、验证、记录、停止的循环” |
| 2026-06-16 | LangChain 发布《The Art of Loop Engineering》，把 agent loop、verification loop、event-driven loop、hill-climbing loop 分层 | 主流框架开始把 loop 当成可组合的工程对象 |
| 2026-07 | “graph engineering”在 X、博客和中文技术文章中快速传播 | 新的行业标签，核心语义是显式编排多个 loop/节点 |
| 2026-07-22 | LangChain 发布《3 Years of Graph Engineering with LangGraph》 | 官方明确承认术语是新的，但 graph-based agent engineering 已经实践三年 |

因此，“graph engineering 是不是刚发明的”要分两层回答：**名字是新的，工程问题不是新的**。

## Loop engineering：工程对象是一个 agent 的执行循环

最小 agent 通常是：

```text
目标
  ↓
模型决定动作 → 调用工具 → 观察结果
      ↑                 ↓
      └── 失败/未完成 ──┘
              ↓
       验证成功 / 达到上限 → 停止
```

Loop engineering 关注的不是“把 prompt 写得漂亮”，而是围绕这个循环设计运行时：

1. **工具边界**：agent 能读什么、写什么、调用什么；
2. **上下文管理**：哪些历史保留，何时压缩，哪些结果写入外部状态；
3. **验证机制**：测试、lint、外部 API 状态、独立 reviewer，不能只相信 agent 自己说 done；
4. **失败处理**：重试、改策略、回滚、升级给人，而不是无限重复相同动作；
5. **停止条件**：成功条件、最大迭代次数、时间/Token/费用预算；
6. **外部记忆**：下一轮或下一次运行知道已经做过什么、还剩什么；
7. **触发方式**：手动调用、cron、webhook、CI 事件或持续 heartbeat。

要注意“loop”有两种常见含义：

- **inner loop**：单次 agent run 内部的 plan → act → observe 循环；
- **outer loop**：跨多次 run 的自动化循环，例如每天扫描 issue、创建 worktree、派 agent 修复、独立验证、写回状态。

不同文章有时只讲其中一种，所以不能把所有 loop engineering 文章直接当成同一套标准。

## Graph engineering：工程对象是多个过程之间的拓扑

一个最小 graph 可以写成：

```text
输入
  ↓
分类/路由 ──→ 需要人工？ ──→ 人工审批
  ↓                    ↓
并行分解：研究 / 编码 / 安全检查
  └──────────┬─────────┘
             ↓
           汇总
             ↓
          外部验证
        失败 ↺  成功 → 发布
```

Graph engineering 把下面这些东西从“藏在 agent prompt 或框架黑盒里”提升成显式架构：

- **Node**：可以是一个 LLM call、一个完整 agent loop、工具、检索、数据库操作、验证器或人；
- **Edge**：节点间的允许转移，可以是确定性条件，也可以由模型做路由判断；
- **State**：跨节点传递的、有 schema 的状态，不等于当前上下文窗口；
- **Artifact**：代码、报告、测试结果、审批记录等实际产物；
- **Fan-out / fan-in**：并行拆分和汇合；
- **Guard / veto**：安全扫描或人工节点可以阻断后续边；
- **Checkpoint**：跨边保存状态，节点失败时只重试节点，而不是从头重跑；
- **Budget / identity / permissions**：在 graph state 或运行时中追踪成本、调用者和工具权限；
- **Trajectory observability**：观察实际走过的路径，而不只看最终答案。

一个重要的现实是：graph engineering 还包含两个不同时间尺度的 graph：

1. **组织图（org graph）**：长期存在的角色、职责、工具权限和稳定协作关系；
2. **工作图（work graph）**：某次任务运行时动态生成的临时节点、分支和依赖。

“动态 agent org”是 graph 话题进一步上移的版本：运行中根据新证据生成、删除、重排或合并节点。这个方向可能有实质工程难度，但目前更多是架构设想和行业文章，不能等同于已经成熟的通用技术。

## Graph 和 loop 的严格关系

从计算图/状态机角度，loop 是一个含有回边的有向图：

```text
loop = graph(node, edge, state) + at least one back-edge
```

所以：

- graph 不会取代 loop；
- loop 是 graph 的简单特例；
- graph 节点内部可以包含另一个 graph，或一个完整的 agent loop；
- 生产 agent graph 通常**不是 DAG**，因为重试、修订、等待人工、工具失败恢复都需要回边；
- `LangChain` 的 agent loop 本身也建立在 `LangGraph` 的运行时之上（当前 LangChain 1.x）。

更准确的分层是：

```text
Graph（系统级拓扑、状态、依赖、审批、并发）
  └── Node（可能是确定性函数，也可能是完整 agent）
        └── Agent loop（模型反复思考、调用工具、观察）
              └── Context / prompt engineering（每次调用看什么）
```

## LangChain 与 LangGraph 到底已经做到了什么

### LangChain：高层 agent harness / 组件生态

早期 LangChain 更常见的是 chain / Runnable 组合：把 prompt、model、parser、retriever 和 tool 串起来。它适合快速拼装单次调用或较规则的链路，但复杂的持久化、循环、人工暂停和恢复不是它最初的核心抽象。

现在的 LangChain 1.x `create_agent` 已经是一个高层 agent harness：它管理 model loop、prompt、tools 和 middleware；官方文档说明它内部生成的是 LangGraph graph，因此 agent 可以直接嵌入更大的 `StateGraph`，并保留 middleware 与持久化能力。

### LangGraph：显式状态图和运行时

LangGraph 把 graph 作为一等抽象，核心能力包括：

| LangGraph 能力 | 对应 graph engineering 语言 |
|---|---|
| `StateGraph(State)` | 有 schema 的共享状态 |
| `add_node` | 注册 LLM、agent、工具或确定性函数节点 |
| `add_edge` | 静态转移关系 |
| `add_conditional_edges` | 条件路由/模型决策边 |
| 回边 | retry、revision、evaluator-optimizer loop |
| `Send` | 运行时动态 fan-out/map-reduce |
| checkpointer | 跨步骤/跨会话 checkpoint 和恢复 |
| `interrupt` / `Command(resume=...)` | 人类审批、暂停与恢复 |
| retry policy | 节点级失败隔离与重试 |
| subgraph | 将完整 agent 或子流程封装成节点 |
| time travel | 从 checkpoint replay/fork，调试轨迹 |

例如官方文档展示的流程中，`read_email`、`classify_intent`、`search_documentation`、`human_review`、`send_reply` 都可以成为节点；搜索节点有独立 retry policy，整个图通过 checkpointer 持久化。官方还提供 `Send` 做动态 map-reduce，以及从历史 checkpoint 重放或 fork 执行的能力。

这已经覆盖了目前流行文章对 graph engineering 的大部分“核心定义”。因此如果一篇文章说 graph engineering 的核心是 node、edge、state、conditional routing、parallel branches、checkpoint、human-in-the-loop，那么它和 LangGraph 的技术对应关系几乎是一一映射。

## 与 LangGraph 的差异：概念 vs 产品

| 维度 | Graph engineering | LangGraph |
|---|---|---|
| 类型 | 工程方法/架构视角/行业术语 | 可执行的开源框架和 runtime |
| 范围 | 可覆盖任意 orchestrator、Temporal、Prefect、AutoGen、CrewAI、自研系统 | 重点是 LangChain 生态中的 stateful agent graph |
| 节点 | agent、函数、工具、人、服务、队列等都可以 | 通过 node/subgraph 表达，支持确定性和 agentic 节点 |
| 状态 | 强调 schema、所有权、跨边传递和审计 | `StateGraph` + reducer + checkpointer |
| 拓扑 | 设计组织图和运行时工作图，可能动态改写 | 静态图 + 条件边 + `Send` 动态派发；动态改写组织图不是其默认产品抽象 |
| 运行可靠性 | 还要求幂等、背压、权限、预算、观测、部署和外部证据 | 提供 persistence、durable execution、streaming、interrupt 等运行时基础能力，但应用仍需负责业务治理 |
| 评测 | 关注节点质量、边路由、整条 trajectory、成本和失败传播 | 可接 LangSmith 等评测/观测产品，但 graph engineering 不等于 LangSmith |

所以不能说“graph engineering = LangGraph”。更准确是：

> **LangGraph 是 graph engineering 的一个成熟实现；graph engineering 是 LangGraph 之外的更宽架构实践。**

## 是不是重新炒概念

### 结论：名字有炒作成分，问题有真实增量

支持“概念新炒”的证据：

1. 有向图、状态机、DAG、工作流编排、fan-out/fan-in、重试和人工审批，在 Airflow、Temporal、Prefect、分布式系统和 LangGraph 之前就存在；
2. Anthropic 2024 年的 agent patterns 已经覆盖 chaining、routing、parallelization、orchestrator-workers 和 evaluator-optimizer；
3. LangGraph 早在此次热词出现前就用 graph、node、edge、state、checkpoint 和 cycle 解决了相同问题；
4. 许多“graph engineering”文章只是把已有的 multi-agent orchestration 换成了更宏大的词，并没有提出新的算法、协议或执行语义。

支持“并非纯炒作”的证据：

1. 现在节点里可以放一个能力更强、可独立完成长任务的完整 coding agent，而不只是一个 LLM call；
2. 工作从单 agent 的“能否完成一步”转向跨小时/跨仓库/跨团队的“多步协调是否可靠”；
3. fan-out、并发、checkpoint、预算、权限、背压和跨节点 artifact 传递，在规模上会产生真实的新故障面；
4. trajectory eval 比单次输出 eval 更重要：最终结果正确不代表路径、成本、权限和副作用正确；
5. 当 graph 与真实工具、人类审批和持久化状态相结合后，它的工程对象已经比“一个 prompt 外面加个 while”更大。

因此最准确的判断是：**graph engineering 是旧技术问题的新抽象名，但也反映了 agent 系统规模变大后真实的工程重心迁移。它不是 LangGraph 的替代技术，也不是新的基础算法。**

### 新词什么时候有价值

如果它帮助团队明确讨论以下问题，它就是有用的术语：

- 哪些步骤必须确定性执行，哪些步骤交给模型判断？
- 哪些节点可以并行，哪些依赖必须串行？
- 状态和 artifact 用什么 schema 跨边传递？
- 节点失败时 retry、fallback、人工升级还是取消整条分支？
- 谁能调用哪个工具？预算在哪里扣减？
- 需要从哪个 checkpoint 恢复？能否幂等重放？
- 如何评估一条完整 trajectory，而不是只看最终文本？

如果文章只是把“LangGraph 的 node/edge/state”重新命名，却没有新的执行语义、评估方法、部署机制或可复现系统，那么就是概念包装。

## 什么时候该用 loop，什么时候该上 graph

### 只用 loop

适合：

- 单一领域、单一 agent、工具集合有限；
- 任务天然是“做—验证—修正”；
- 没有强制审批、长期暂停或多分支并行需求；
- 能用一个外部状态文件记录进度；
- 失败时从最近一次循环继续即可。

例：每日扫描 issue → 修一个简单 bug → 跑测试 → 生成报告。

### 使用 graph

适合：

- 多个领域/agent 需要明确职责边界；
- 分支可以并行，之后需要汇合；
- 有明确的审批、权限、审计或 veto 节点；
- 任务跨小时/跨天，必须 checkpoint、暂停和恢复；
- 不同分支有不同模型、工具、预算或安全策略；
- 失败传播需要被隔离，不能让一个节点重跑整条流程。

例：需求分析 → API/数据库/前端/安全并行 → 汇总 → 集成测试 → 人工发布审批。

### 不要为了“graph”强行画 graph

LangChain 官方自己的回顾也承认：有些任务本质上是开放式 deep research，提前把所有路径硬编码成 graph 反而限制 agent；这类场景更适合一个带计划、委派和上下文管理的 agent harness。图的价值来自**需要显式约束的结构**，不是节点越多越先进。

## 给 agent 开发者的工程检查表

无论最后叫 loop 还是 graph，至少要回答：

1. **目标**：成功的可验证条件是什么？
2. **作用域**：每个 agent/节点拥有哪个领域、文件、工具和权限？
3. **状态**：哪些状态在模型上下文内，哪些必须外置并持久化？
4. **边界**：哪些转移是代码确定的，哪些必须让模型判断？
5. **验证**：谁或什么能否决结果？是否有外部证据（测试、真实 API、人工）？
6. **恢复**：节点失败能否幂等重试？从哪个 checkpoint 恢复？
7. **成本**：并行、重试和动态派生会产生多少 token、时间和工具费用？
8. **安全**：不同节点是否需要不同身份、密钥和工具白名单？
9. **观测**：能否看到 node、edge、run、artifact、延迟、成本和实际轨迹？
10. **简化**：如果删掉大部分节点，单 loop 是否已经足够？

## Sources

- Anthropic, [Building effective agents](https://www.anthropic.com/research/building-effective-agents)（2024；chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer）
- Addy Osmani, [Loop Engineering](https://addyosmani.com/blog/loop-engineering/)（2026-06；loop engineering 的流行化解释）
- LangChain, [The Art of Loop Engineering](https://www.langchain.com/blog/the-art-of-loop-engineering)（2026-06-16；agent/verification/event-driven/hill-climbing loops）
- Josh C. Simmons, [We Are Entering the Graph Engineering Phase](https://www.drjoshcsimmons.com/writing/we-are-entering-the-graph-engineering-phase)（2026-07；graph engineering 的流行定义）
- Louis-François Bouchard, [Graph Engineering Explained: What Actually Changed](https://www.louisbouchard.ai/graph-engineering-explained/)（2026-07；对“新名词、旧结构”的批判）
- LangChain, [3 Years of Graph Engineering with LangGraph](https://www.langchain.com/blog/3-years-of-graph-engineering-with-langgraph)（2026-07-22；官方回顾与 novelty 判断）
- LangGraph 官方文档：[Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)、[Thinking in LangGraph](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph)、[Persistence / Checkpointers](https://docs.langchain.com/oss/python/langgraph/persistence)、[Time travel](https://docs.langchain.com/oss/python/langgraph/use-time-travel)
- LangChain 官方文档：[Overview / create_agent](https://docs.langchain.com/oss/python/langchain/overview)、[Middleware](https://docs.langchain.com/oss/python/langchain/middleware)
- LangGraph GitHub：[langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)
- IBM, [What Is Loop Engineering?](https://www.ibm.com/think/topics/loop-engineering)（2026-07；loop 的工程组成与运行状态）
