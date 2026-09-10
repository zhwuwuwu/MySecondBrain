# 邮件草稿：SuperClaw 安全能力（Security Feature）立项汇报

> 日期：2026-08-20
> 收件人：Senior Manager
> 主题：关于 SuperClaw 增加安全能力的调研结论、候选项目与落地建议

---

Hi All,

近期我围绕"SuperClaw 要不要把安全做成正式 feature、怎么做"做了一轮调研，并评估了两个公司内部候选项目（agentrail secure-sandbox 与 sentineld from Andy Xiong's Team）。以下是结论摘要，供你决策。

## 1. Agent Security 基本概念：为什么现在要谈这个

**Security ≠ Privacy**。Privacy 关注"用户数据如何被收集、使用、记忆和披露"（例如私密对话写入长期 memory、跨任务泄露），本质是用户数据控制权问题；**Security 关注"对抗者或未授权路径是否破坏机密性/完整性/可用性"**（例如 prompt injection 诱导 Agent 读取密钥并发邮件）。可以粗略理解为：**Privacy 是 Security 的一个子模块**——防泄露是安全的一部分，但安全远不止防泄露。Agent 输出的是**真实副作用**——读文件、写文件、发请求、执行命令、对外发数据。模型的一次误判可能变成实际动作。

常见的攻击面（按生命周期）：

| 攻击面 | 典型风险 |
|---|---|
| 用户输入/外部内容 | Prompt Injection、越狱、恶意网页/文档诱导 |
| Memory/RAG | Memory 投毒、跨任务污染 |
| Skill/插件/MCP | 恶意工具、工具结果注入、供应链依赖 |
| Shell/代码执行 | 任意命令、危险参数、安装脚本 |
| 文件/数据库/API | 越权读写、跨项目访问 |
| 网络/运行时 | SSRF、内网访问、sandbox 逃逸 |
| 输出/外发 | 敏感信息泄露、错误发送 |

要打造一个安全的 LLM Agent 系统，**两条途径缺一不可**：

1. **风险/攻击检测**——判断"这个输入/计划/工具调用危不危险"（规则 + 模型 judge）；
2. **可审批的执行环境**——即使判断漏了，动作在物理上也过不了隔离/审批边界。

判断做得好可以让危险动作"不被发起"，执行边界做得好可以让危险动作"即使发起也做不成"。二者是互补关系，不是替代关系。

## 2. 候选项目介绍

Andy Xiong team 的 agentrail 围绕 OpenClaw 加强安全性和可信性，有secure-sandbox，prompt injection防护，TEE vault，Remote DICE attestation四个方向的9个仓库，对我们最有价值的是两个：

### 2.1 sentinel：全 pipeline 的风险检测

仓库：[intel-sandbox/icase.ai.agent.agentrail.sentinel](https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentinel/tree/main)（agent platform 插件胶水）+ [intel-sandbox/icase.ai.agent.agentrail.sentineld](https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentineld/tree/main)（判定核心 daemon）

**功能定位**：对 agent 工具调用全 pipeline 做风险检测（prompt injection、路径泄露等）。

- **判定核心 daemon**（sentineld）：与 agent platform 解耦的独立服务，规则引擎 + guard 模型 + 意图对齐 judge，输出 allow / block / approve 三态；
- **7 类风险判定**：直接注入、间接注入/工具结果投毒、命令混淆、路径与应用层、记忆投毒、数据外泄链、意图对齐；
- **两种检测手段并行：**
	- **规则地板永不关机**：注入 13 旗标 / 混淆 17 模式 / 敏感路径 / 外泄三阶段等规则是 hard floor 永远跑，模型是增强层；
	- **Model as a Judge**：guard（Qwen3Guard-0.6B）管"话里有没有毒"（注入分类），judge（可复用 core agent model）管"事对不对"（意图对齐）；

这对应上面第 1 条路径：**风险/攻击检测**。

此外还开源的判别框架（例如 蚂蚁集团的 AgentAegis 与 Acacian Aegis）和判别模型 （SingGuard [[2606.22873] SingGuard: A Policy-Adaptive Multimodal LLM Guardrail with Dynamic Reasoning](https://arxiv.org/abs/2606.22873), a **policy-adaptive** multimodal LLM guardrail model 以及 SingGuard-NSFA, 面向自主 AI Agent 不同攻击面的同意安全护栏模型) 被调研，在benchmarking的时候可以加进去作比较。

### 2.2 secure-sandbox：带审批的执行沙箱环境

仓库：[intel-sandbox/icase.ai.agent.agentrail.secure-sandbox](https://github.com/intel-sandbox/icase.ai.agent.agentrail.secure-sandbox)（agentrail + secure-sandbox）

**功能定位**：给 agent 的 tool call 提供一个**强制审批 + 隔离的执行环境**。

- **执行隔离**：所有工具调用在 agentraild 管理的 sandbox 容器内执行，容器是唯一不可信侧；
- **强制审批**：控制面（agentraild daemon）在容器外，对每次工具调用做 allow_once / allow_lifetime / deny 三态审批，超时 fail-closed（默认拒绝）；
- **两层中介**：文件访问经 FUSE 逐操作中介，网络出网经 Squid MITM 代理审批——即使模型判断出错，动作在物理层也过不了关。

这对应上面第 2 条路径：**可审批的执行环境**。


## 3. 集成到 SuperClaw 的方案

### 3.1 secure-sandbox 集成：工具委托桥 + 执行 sandbox

**核心思路**：新增一个"委托桥插件"，利用 opencode 插件系统提供三个已验证的委托点：`tool.execute.before`（拦截/改写）、`tool()`（注册自定义工具）、per-agent `tools: {"bash": false}`（禁内置工具），把 opencode 的 tool call（bash/MCP 等对外交互）委托到 agentraild 管理的执行 sandbox，让所有工具执行都落在带审批的隔离环境里。现有 guard 插件（projects-delete、protected-pipeline 等）在委托桥之上继续生效。Sandbox 生命周期管理必须外置：构建sandbox_manager → agentraild → secure sandbox 的单向信任链。

**这使实现方案与 WSL 的去留强相关**：

| 选项                         | 形态                                                       | 优点                                                 | 代价                                                                                            |
| -------------------------- | -------------------------------------------------------- | -------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **A. 保留 WSL + backend 容器** | secure sandbox 由 sandbox_manager 统一管理，嵌套在当前 backend 容器内部 | 不改变 sandbox_manager 的 WSL 编排，改动集中在容器内 opencode 插件层 | 接受 sandbox 嵌套带来的臃肿；需处理路径映射（容器内 /workspace ↔ sandbox FUSE 视图）                                  |
| **B. 移除 WSL，backend 跑裸机**  | 形态与 OpenClaw 一致（backend 在 host，仅工具执行进 sandbox）           | 容器运行时只服务 sandbox 一个目的，攻击面/运维面更小                    | **secure sandbox 本身仍基于 Linux 部署**——**只能移除 backend sandbox，无法彻底移除 WSL**；且与当前正在讨论的 WSL 移除工作互相影响 |
### 3.2 sentineld 集成：规则 + 模型 judge 双轨

核心思路：**检测服务以 sidecar 形式运行**（判定核心 sentineld 作为独立 daemon 保留，重写 OpenClaw 专属胶水层为 opencode 插件 `99_superclaw_sentinel_gate.ts`），把每个攻击面的检测通过插件形式挂在对应 agent runtime 位置上。分成规则检测和模型判断兜底两种方法结合。

**双轨结合**：

- **基于规则的检测**：立即可以尝试集成——集成计划已完成（5 个 Phase：决策冻结 → 差异分析 → glue 插件原型 → HITL/approve → 模型落地 → 验证）测试集也已经就绪（promptfoo），**计划下周二（WW35.2）可以先以规则地板模式（无模型）运行**，并且可以立刻进一步评估效果与引入的延迟；
- **Model as a judge**：需要更多 benchmark 测试，比较不同 SKU（硬件档次）上的模型选择——guard 模型候选 Qwen3Guard-0.6B需要验证在不同SKU上的部署情况和推理延迟，judge 模型为通用模型，因此可以复用各 SKU 实际选中的 core agent model，但需要比较风险判断性能和responce 延迟。在模型验证完成前，默认两模型均 disabled，纯规则模式运行（符合 fail-closed 与"gated 能力显式报告"契约）。**这需要额外的1周左右的时间来评估**。

## 4. 工作量评估与建议

风险检测和可控执行两条路径本不冲突，但是bandwidth和人手限制，需要我们先从其中一个开始。建议**先启动 sentineld 规则检测**及benchmarking（下周，低风险、见效快、有 evaluation set）后续逐步比较和添加不同的判别模型；secure-sandbox 执行环境作为第二阶段（依赖 WSL 去留决策）但也可以同步制定集成计划；


**附件：**
所有的调研文档和集成计划均已上传至 XX 分支； Agent security survey 在 PPT：XX

Best,
Zihan

---

# Email Draft (English Version): SuperClaw Security Feature Proposal

> Date: 2026-08-20
> To: Senior Manager
> Subject: Research Findings, Candidate Projects, and Rollout Recommendations for Adding Security Capabilities to SuperClaw

---

Hi All,

I recently completed a research round on whether SuperClaw should make security a formal feature and how to approach it, and evaluated two internal candidate projects (agentrail secure-sandbox and sentineld from Andy Xiong's Team). Below is a summary of the findings for your decision.

## 1. Agent Security Basics: Why This Matters Now

**Security ≠ Privacy.** Privacy concerns how user data is collected, used, remembered, and disclosed (e.g., private conversations being written into long-term memory or leaking across tasks); at its core it is a question of user data control. **Security concerns whether an adversary or an unauthorized path can break confidentiality / integrity / availability** (e.g., prompt injection tricking an agent into reading a secret key and sending an email). A rough way to think about it: **Privacy is a sub-module of Security** — leak prevention is part of security, but security goes far beyond leak prevention. Agents produce **real side effects** — reading files, writing files, making requests, executing commands, sending data externally. A single model misjudgment can turn into an actual action.

Common attack surfaces (by lifecycle):

| Attack Surface | Typical Risks |
|---|---|
| User input / external content | Prompt injection, jailbreak, malicious web pages / documents |
| Memory/RAG | Memory poisoning, cross-task contamination |
| Skill / plugins / MCP | Malicious tools, tool-result injection, supply-chain dependencies |
| Shell / code execution | Arbitrary commands, dangerous parameters, install scripts |
| Files / databases / APIs | Unauthorized reads/writes, cross-project access |
| Network / runtime | SSRF, intranet access, sandbox escape |
| Output / exfiltration | Sensitive information disclosure, sending to wrong recipients |

A secure LLM agent system requires **both of two complementary approaches**:

1. **Risk / attack detection** — judging whether an input, plan, or tool call is dangerous (rules + model judge);
2. **Approved execution environment** — even if detection misses something, the action physically cannot pass the isolation / approval boundary.

Good detection prevents dangerous actions from being initiated; a good execution boundary ensures dangerous actions cannot succeed even if initiated. They are complementary, not substitutes.

## 2. Candidate Projects

The agentrail effort led by Andy Xiong's team strengthens the security and trustworthiness of OpenClaw across 9 repositories in four directions: secure-sandbox, prompt-injection protection, TEE vault, and Remote DICE attestation. Two are most valuable to us:

### 2.1 sentinel: Full-Pipeline Risk Detection

Repos: [intel-sandbox/icase.ai.agent.agentrail.sentinel](https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentinel/tree/main) (agent-platform plugin glue) + [intel-sandbox/icase.ai.agent.agentrail.sentineld](https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentineld/tree/main) (decision-core daemon)

**Purpose**: Performs risk detection across the full agent tool-call pipeline (prompt injection, path leakage, etc.).

- **Decision-core daemon** (sentineld): A standalone service decoupled from the agent platform, combining a rules engine + a guard model + an intent-alignment judge, outputting three-state verdicts (allow / block / approve);
- **Seven classes of risk checks**: direct injection, indirect injection / tool-result poisoning, command obfuscation, path & application layer, memory poisoning, data-exfiltration chains, and intent alignment;
- **Two detection mechanisms in parallel:**
  - **Always-on rule floor**: rules such as 13 injection flags / 17 obfuscation patterns / sensitive paths / three-stage exfiltration always run as a hard floor; models are an enhancement layer;
  - **Model as a Judge**: the guard model (Qwen3Guard-0.6B) detects whether the text is malicious (injection classification), while the judge (can reuse the core agent model) determines whether an action matches user intent (intent alignment);

This corresponds to approach #1 above: **risk / attack detection**.

In addition, we surveyed open-source discrimination frameworks (e.g., Ant Group's AgentAegis and Acacian Aegis) and discrimination models (SingGuard [[2606.22873] SingGuard: A Policy-Adaptive Multimodal LLM Guardrail with Dynamic Reasoning](https://arxiv.org/abs/2606.22873), a **policy-adaptive** multimodal LLM guardrail model, and SingGuard-NSFA, a consent-oriented safety guardrail model covering different attack surfaces of autonomous AI agents). These can be included as baselines for comparison during benchmarking.

### 2.2 secure-sandbox: Execution Sandbox with Mandatory Approval

Repo: [intel-sandbox/icase.ai.agent.agentrail.secure-sandbox](https://github.com/intel-sandbox/icase.ai.agent.agentrail.secure-sandbox) (agentrail + secure-sandbox)

**Purpose**: Provides a **mandatory-approval + isolated execution environment** for agent tool calls.

- **Execution isolation**: all tool calls execute inside sandbox containers managed by agentraild; the container is the only untrusted side;
- **Mandatory approval**: the control plane (agentraild daemon) sits outside the container and makes three-state approvals (allow_once / allow_lifetime / deny) for each tool call, failing closed on timeout (deny by default);
- **Two-layer mediation**: file access goes through per-operation FUSE mediation; network egress goes through a Squid MITM proxy with approval — even if the model misjudges, the action cannot pass at the physical layer.

This corresponds to approach #2 above: **an approved execution environment**.

## 3. Integration Plans for SuperClaw

### 3.1 secure-sandbox Integration: Tool Delegation Bridge + Execution Sandbox

**Core idea**: Add a "delegation bridge plugin" that leverages three verified delegation points in the opencode plugin system — `tool.execute.before` (intercept / rewrite), `tool()` (register custom tools), and per-agent `tools: {"bash": false}` (disable built-in tools) — to delegate opencode tool calls (bash, MCP, and other external interactions) to an execution sandbox managed by agentraild, so that all tool execution lands in an approved, isolated environment. Existing guard plugins (projects-delete, protected-pipeline, etc.) continue to apply on top of the bridge. Sandbox lifecycle management must stay external: build a one-way trust chain of sandbox_manager → agentraild → secure sandbox.

**This makes the implementation approach tightly coupled with the WSL keep-or-remove decision**:

| Option | Form | Pros | Costs |
|---|---|---|---|
| **A. Keep WSL + backend container** | secure sandbox uniformly managed by sandbox_manager, nested inside the current backend container | No change to sandbox_manager's WSL orchestration; changes concentrated in the in-container opencode plugin layer | Accept the bloat of nested sandboxes; must handle path mapping (in-container /workspace ↔ sandbox FUSE view) |
| **B. Remove WSL, backend on bare metal** | Form matches OpenClaw (backend on host, only tool execution goes into the sandbox) | Container runtime serves only the sandbox; smaller attack surface and lower ops burden | **secure sandbox itself still requires Linux** — **only the backend sandbox can be removed; WSL cannot be fully eliminated**; also intersects with the ongoing WSL-removal discussion |

### 3.2 sentineld Integration: Rules + Model Judge Dual Track

Core idea: **Run the detection service as a sidecar** (keep sentineld as the standalone decision-core daemon; rewrite the OpenClaw-specific glue layer as an opencode plugin `99_superclaw_sentinel_gate.ts`), attaching detection for each attack surface to the corresponding agent-runtime hook points via plugins. It combines rule-based detection with a model-judge fallback.

**Dual-track combination**:

- **Rule-based detection**: can be integrated immediately — the integration plan is complete (5 phases: decision freeze → gap analysis → glue-plugin prototype → HITL / approve → model rollout → validation) and the test set is ready (promptfoo). **We plan to start next Tuesday (WW35.2) in rule-floor mode (no models)**, and can immediately evaluate effectiveness and added latency;
- **Model as a judge**: requires more benchmarking to compare model choices across SKUs (hardware tiers). The guard candidate Qwen3Guard-0.6B needs deployment and inference-latency validation on different SKUs; the judge is a general-purpose model, so it can reuse the core agent model selected for each SKU, but we need to compare risk-judgment quality and response latency. Until model validation is complete, both models stay disabled by default and the system runs in pure rule mode (consistent with the fail-closed and "explicit reporting of gated capabilities" contract). **This requires roughly one additional week of evaluation.**

## 4. Effort Estimate and Recommendations

The two tracks — risk detection and controlled execution — are not in conflict, but bandwidth and staffing constraints mean we need to start with one of them. We recommend **starting with sentineld rule-based detection plus benchmarking** (next week; low risk, fast results, ships with an evaluation set), gradually comparing and adding different discrimination models afterward; the secure-sandbox execution environment comes second (pending the WSL keep-or-remove decision), though we can prepare its integration plan in parallel.

**Attachments:**
All research documents and integration plans have been uploaded to branch [intel-sandbox/applications.ai.superclaw-zihan: AI SuperClaw for AI Agent](https://github.com/intel-sandbox/applications.ai.superclaw-zihan) ; The agent security survey is in the PPT at XX.

Best,
Zihan