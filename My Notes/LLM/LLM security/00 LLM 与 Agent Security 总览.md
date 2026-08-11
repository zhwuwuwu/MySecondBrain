
# LLM 与 Agent Security 总览

> 研究目标：为刚接触 security 的 LLM agent 研究者建立一张由浅入深、覆盖模型生命周期与 Agent 运行时的地图。
>
> 研究时间：2026-08-06。本文优先引用已核验的综述、论文、标准与工程指南；博客/项目文档用于补充实践视角，不与同行评审结论混同。

## 0. 先给结论：应该怎样理解这个领域

LLM security 不是“给模型加一个更强的拒答 prompt”。更准确的抽象是：

> **保护模型、数据、凭证、工具、运行时状态和外部系统的机密性（C）、完整性（I）与可用性（A），同时保证模型行为符合授权目标。**

可以把研究对象分成两层：

1. **Model security**：数据、训练、微调、权重、推理 API、模型行为与隐私。
2. **Agent security**：在模型外增加的 harness/orchestrator、规划循环、context/memory、RAG、工具/MCP、身份权限、sandbox、审批和监控。

Agent security 不是 Model security 的简单子集。模型的一次错误输出，在 Agent 中可能变成跨工具的控制流劫持、越权调用、数据外传、持久记忆污染或真实世界副作用。

---

## 1. 基础边界：五个相邻但不同的概念

### 1.1 Security / Safety / Privacy / Reliability / Robustness

| 概念 | 核心问题 | 典型例子 |
|---|---|---|
| Security（安全） | 对抗性主体或未授权路径能否破坏 C/I/A、扩大能力或越过授权边界？ | prompt injection 诱导 Agent 读取密钥、调用发邮件工具；Skill 被篡改后窃取凭证 |
| Safety（安全性/无伤害性） | 即使没有恶意攻击，系统行为是否会使人、财产、组织或环境处于不可接受的伤害风险？ | 幻觉导致错误医疗建议；Agent 在用户意图正确但判断错误时误删文件 |
| Privacy（隐私） | 个人或敏感主体是否能够控制其信息的收集、使用、推断、记忆和披露？ | 训练数据记忆、RAG 跨租户泄露、把用户私密对话写入长期 memory |
| Reliability（可靠性） | 在给定条件和时间内，系统能否持续按要求工作，并在失败后可恢复？ | 无限循环、重复扣款、工具参数错误、任务失败后无法回滚 |
| Robustness（鲁棒性） | 输入、环境、模型、工具或运行条件发生扰动时，系统性能和安全性质是否仍保持在可接受范围？ | 拼写扰动、分布外输入、对抗样本、工具返回格式变化、网络抖动后仍不越权 |

这五个概念的范围可以这样划分：

1. **Security 关注对抗者和授权边界**。是否有攻击者不是唯一判据，但“未授权的主体/路径”和 C/I/A 破坏是核心。
2. **Safety 关注伤害结果，不要求存在攻击者**。错误决策、误操作和危险自动化都可能是 Safety 问题。恶意 prompt 导致危险动作时，它同时属于 Security（攻击路径）和 Safety（伤害后果）。
3. **Privacy 关注信息主体和数据处理边界**。隐私泄露可以由攻击造成，也可以由过度收集、错误配置或模型记忆造成，因此 Privacy 不是 Security 的同义词。
4. **Reliability 关注按规格工作和故障恢复**。系统可能非常可靠但不安全（稳定地执行过大的权限），也可能安全但不可靠（每次都拒绝合法任务）。
5. **Robustness 是跨领域性质，不是第五种独立威胁类型**。它描述系统面对扰动、异常或对抗输入时是否保持能力。鲁棒性可以服务于 Reliability、Safety 或 Security；普通噪声鲁棒性不自动等于 Security。

因此，记录一个问题时最好同时写**问题性质**和**受影响目标**：

```text
indirect prompt injection（攻击技术）
→ MCP/tool（攻击面）
→ secret disclosure（影响：Privacy + Security）
→ excessive privilege（根因：Authorization）
→ deterministic policy + sandbox（控制）
```

没有一套被所有标准采用的唯一五分法；上表是本文的工作分类。NIST AI RMF、NIST Privacy Framework、ISO/IEC AI 术语和 OWASP GenAI 分类的关注点不同，使用时应保留来源和上下文。

### 1.2 威胁建模的最小词汇

每个问题都建议写成：

`攻击面（surface） → 攻击技术（technique） → 资产（asset） → 影响（impact） → 控制（control） → 证据（evidence）`

至少明确：

- **攻击者**：普通用户、恶意内容作者、被入侵的工具服务、供应链攻击者、内部人员、恶意 Agent。
- **资产**：系统提示、模型权重、训练数据、个人数据、API key、文件、数据库、外部业务动作、长期记忆。
- **信任边界**：用户/系统指令，可信/不可信文档，模型/工具，Agent/Agent，宿主机/sandbox，租户/租户。
- **授权边界**：谁允许 Agent 做什么、以谁的身份、在什么资源范围、持续多久。

### 1.3 三种常见混淆

1. **攻击面不是攻击技术**：MCP tool result 是攻击面；indirect prompt injection 是技术。
2. **提示词中的规则不是安全边界**：模型可理解“不要读密钥”，但不能替代操作系统权限、API authorization 或 deterministic policy。
3. **拒答率不是安全性**：真正的指标应关注未经授权的秘密披露、工具调用、状态改变、权限提升和恢复能力。

### 1.4 常见 Agent 问题应该放在哪里？

| 现象/术语 | 首要归类 | 说明 |
|---|---|---|
| Prompt injection / indirect prompt injection | **攻击技术**；通常属于 Prompt/Context 与 Agent Security | 不可信内容被模型当成高优先级指令，导致计划、工具调用或数据流偏离。它可能造成 Security、Privacy 或 Safety 影响，本身不是“输出安全”。 |
| Agent 恶意行为/危险行为 | **Agent 行为安全 + Authorization/Runtime** | 看 Agent 实际或计划执行了什么：越权读写、危险工具、提权、外传、部署、删除、资源滥用。判断器可以发现风险，但权限网关、审批、沙箱和下游授权才负责最终阻断。 |
| Agent 恶意/有害输出 | **Output safety / 内容安全**；若造成泄密则同时是 Privacy/Security | 看模型输出了什么，以及谁能看到或使用它。输出有害不一定意味着发生攻击；若包含秘密、内部提示或跨租户数据，则是隐私/机密性问题。 |
| MCP/Skill 扫描 | **Security assurance + Supply-chain security**，不是单独的攻击类型 | 扫描是在上线前或运行中检查 MCP server、Skill、依赖、描述/schema、权限和行为，属于检测/评估控制。tool poisoning、rug pull、恶意依赖和命令注入才是具体技术。 |
| MCP server / Skill | **Tool/API/Connector 与供应链攻击面** | MCP server 是协议化工具/数据连接器；Skill 是可加载的 Agent 能力包、插件或工作流。二者都可能携带指令、代码、凭证需求和工具权限。 |

最简归类口诀：

> **Prompt injection 是“怎么骗”；Agent 行为是“做了什么”；Agent 输出是“说了什么”；MCP/Skill 扫描是“怎么查”；授权、沙箱和策略执行是“怎么拦”。**

---

## 2. 总体地图：两个互补坐标系

### 2.1 按生命周期看 Model security

1. 数据获取、清洗、标注与授权
2. 预训练、指令微调、RLHF/对齐与评测
3. 模型工件、checkpoint、量化文件与依赖供应链
4. 推理服务、API、认证、限流、日志与多租户隔离
5. 模型行为：越狱、提示注入、幻觉、偏见、拒答绕过

### 2.2 按运行时栈看 Agent security

从上到下是“语言输入”逐渐落到“真实副作用”的路径：

1. 用户与外部内容
2. Prompt/context 组装
3. Harness/orchestrator 与规划循环
4. Memory/RAG/状态持久化
5. Tool/API/MCP/connectors
6. 身份、权限与审批
7. Sandbox/宿主机/网络/凭证
8. 监控、回滚、事件响应

攻击可以从任意层进入，并沿 Agent loop 传播：

`untrusted content → context → plan → tool call → external effect → memory/trace`

### 2.3 一张更实用的分类矩阵

Agent 安全问题同时有多个坐标，不能把所有名词都塞进“攻击类型”一列：

| 坐标 | 要回答的问题 | 例子 |
|---|---|---|
| Surface（攻击面） | 攻击者从哪里进入或影响系统？ | 用户输入、RAG 文档、Memory、MCP server、Skill、tool result、浏览器、网络 |
| Technique（攻击技术） | 通过什么方法改变控制流或数据流？ | Prompt injection、tool poisoning、rug pull、命令注入、SSRF、越权、数据投毒 |
| Behavior/Effect（行为/副作用） | Agent 实际做了什么？ | 读 secret、改文件、发请求、执行代码、写 memory、部署、删除、外传 |
| Target/Impact（目标/影响） | 哪个资产和性质受损？ | C/I/A、Privacy、Safety、费用、评测完整性、业务状态 |
| Control（控制） | 在哪里检测、授权、阻断、恢复？ | 扫描、分类器、Policy Engine、能力网关、审批、Sandbox、审计、回滚 |

例如，一个恶意 Skill 携带隐藏指令，诱导 Agent 读取 `.env` 并通过 MCP 工具外传，可写成：

```text
Skill / MCP tool（Surface）
→ indirect prompt injection + tool poisoning（Technique）
→ read secret + external send（Behavior/Effect）
→ confidentiality / Privacy breach（Target/Impact）
→ Skill scan + least privilege + approval + sandbox + egress policy（Control）
```

---

## 3. Model security：模型生命周期中的主要方向

### 3.1 数据供应链：数据投毒与隐私

**攻击面**：抓取语料、标注集、指令数据、偏好数据、RAG corpus、微调数据。

**主要技术**：

- **Data poisoning**：注入错误、偏置或恶意样本，改变总体行为。
- **Backdoor/trojan**：植入触发器，正常输入表现不变，特定触发条件下执行攻击者行为。
- **PII memorization / leakage**：模型记住并在特定提示下复现个人信息。
- **RAG poisoning**：污染检索库，使未来任务取回错误事实或恶意指令。

**防御**：数据来源与授权追踪、去重与异常检测、分层审查、敏感数据最小化、训练前完整性校验、可回滚数据版本、成员推断/提取测试。

### 3.2 训练、微调与对齐

训练阶段的安全问题不只是“数据是否有毒”，还包括：

- 供应商或开发者是否能修改 reward model、adapter、LoRA 或 checkpoint；
- 安全对齐是否因微调数据而退化；
- 评测集是否泄露或被污染，造成虚假的安全性；
- 模型加载格式、序列化依赖是否能触发任意代码执行。

**控制**：训练流水线最小权限、数据/权重签名与 hash、独立安全评测、隔离 reward model、artifact provenance、可重复构建和发布前红队。

### 3.3 推理服务与模型资产

推理 API 需要传统应用安全：认证、授权、租户隔离、限流、成本控制、审计、密钥管理、错误信息最小化。特别注意：

- **Model extraction**：通过大量查询复制功能、决策边界或近似模型。
- **Training-data extraction**：诱导模型复现训练样本。
- **Prompt stealing**：重建 system prompt、few-shot 示例和业务规则。
- **Availability attacks**：超长输入、递归调用、昂贵工具链或高并发导致资源耗尽。

### 3.4 推理期行为攻击

#### Jailbreak

目标是绕过模型的安全策略，常见方法包括角色扮演、编码/翻译、多轮诱导、自动搜索后缀、跨模型迁移。它主要关注“模型是否产生不应产生的内容”。

#### Prompt injection

目标是改变应用中指令、数据和输出的控制关系。直接注入由用户输入；间接注入把恶意指令藏在网页、邮件、PDF、代码、工具返回值或检索结果中。

**关键区别**：jailbreak 偏向模型策略绕过；prompt injection 偏向应用控制流和指令层级劫持。二者可以组合，但防御重点不同。

---

## 4. Agent security：按运行时层级展开

### 4.1 用户输入与外部内容层

任何进入 context 的外部内容都应默认视为**数据**而不是**指令**。包括网页、邮件、issue、文档、搜索结果、图片 OCR、代码注释、工具描述和其他 Agent 的消息。

**核心风险**：direct/indirect prompt injection、隐藏文本、多模态注入、长上下文劫持、社会工程和数据外传。

**控制**：来源标注、内容与指令分离、结构化输入、内容过滤、独立提取/分类模型、输出验证；但不要把 delimiter 或“请忽略文档中的指令”单独当作安全边界。

### 4.2 Prompt / Context 组装层

这是很多 Agent 的“隐形解释器”：系统提示、开发者提示、用户输入、检索内容、tool observation、历史消息和计划被拼成一个模型可见序列。

**风险**：

- 指令层级混淆；
- system prompt 泄漏；
- context over-sharing，跨用户/跨任务泄露；
- untrusted observation 被当成高权限指令；
- 超长上下文让安全规则被稀释；
- 结构化字段中的自然语言逃逸。

**控制**：显式 provenance/type 标签、最小 context、每次任务重新授权、敏感内容不进入模型、secret redaction、独立 policy engine、对 tool call 做 schema 验证。

### 4.3 Harness / Orchestrator / Planning 层

Harness 决定 Agent 如何循环：何时继续、何时重试、如何拆任务、如何选择工具、何时写入记忆、何时把结果交给用户。因此它是控制流安全层，而不仅是 prompt 模板。

**风险**：

- 无限循环与成本耗尽；
- 计划被污染后跨步骤传播；
- retry 放大副作用；
- 计划结果直接变成执行指令；
- 多 Agent 委派导致信任和权限跨边界传播；
- 失败时没有回滚或幂等性。

**控制**：有限状态机/预算、最大步数和最大重试、动作幂等键、计划与执行分离、每步重新授权、工具 allowlist、危险动作审批、失败安全与回滚、跨 Agent 消息签名和来源验证。

### 4.4 Memory / RAG / Persistent State 层

短期 context 是一次执行流；长期 memory 是可被未来任务读取的持久状态。因此 memory poisoning 类似“持久化控制流污染”。

**攻击与故障**：写入恶意偏好/计划、检索库污染、跨租户检索、旧信息未失效、记忆越权读取、删除不可验证、反思循环把恶意内容再次写回。

**控制**：写入门、来源与时间戳、每条记忆的租户/任务 scope、可信度和过期时间、隔离候选记忆与已确认记忆、检索后授权过滤、删除/回滚、记忆审计与 adversarial replay。

### 4.5 Tool / API / Connector 层

工具是 Agent 安全与传统系统安全的连接点。模型“想调用”不等于用户“授权调用”。

**主要风险**：

- excessive functionality：提供开放式 shell、任意 URL、任意 SQL，而不是最小化函数；
- excessive permissions：读工具拥有写权限，用户操作使用通用高权限身份；
- 参数注入、命令注入、路径遍历、SSRF、SQL 注入；
- tool output injection：工具返回值携带恶意指令；
- confused deputy：Agent 用自己的高权限替用户访问不应访问的资源；
- tool poisoning/rug pull：描述、schema 或实现后来被替换；
- 工具返回秘密，Agent 再通过另一个工具外传。

**控制**：typed schema、`additionalProperties: false`、参数白名单、下游 complete mediation、每工具独立凭证、短期 token、按用户传播身份、读写分离、敏感操作审批、完整参数审计。

**归类说明**：MCP server 和 Skill 都首先属于这一层的 Tool/API/Connector 或供应链攻击面；“扫描”属于 Security assurance（检测/评估控制），不是一种攻击。扫描应覆盖来源、版本、依赖、权限、描述/schema、工具实现、网络出口和运行时行为；发现风险后仍需由授权、沙箱、策略引擎和审批执行阻断。

### 4.6 MCP 层

MCP 可以看作连接模型客户端与工具/数据源的标准化接口；它降低集成成本，也把工具发现、描述、授权、传输和供应链变成统一攻击面。

重点风险可按 OWASP MCP Top 10 记忆：

1. Token mismanagement & secret exposure
2. Privilege escalation via scope creep
3. Tool poisoning
4. Supply-chain/dependency tampering
5. Command injection & execution
6. Intent-flow subversion / prompt injection
7. Insufficient authentication & authorization
8. Lack of audit and telemetry
9. Shadow MCP servers
10. Context injection & over-sharing

**MCP 特有控制**：逐服务器最小权限和独立凭证；审查并 pin tool name/description/schema 的 hash；检测 rug pull 和 shadow server；远程服务使用认证、TLS、窄 OAuth scope、短期 token；本地服务使用 stdio 或 sandbox；危险调用显示完整参数并要求用户确认；把每个 MCP server 当成独立信任域。

### 4.7 Sandbox / Runtime / Host 层

Sandbox 的目标不是让模型“变安全”，而是限制最坏结果：即使上层被注入，代码执行、文件访问、网络访问和凭证读取仍不能越界。

**攻击面**：shell/Python/code interpreter、workspace、容器、宿主机 socket、网络、环境变量、挂载目录、包安装、浏览器和云 metadata endpoint。

**控制**：默认无网络、只读根文件系统、最小 workspace、非 root、seccomp/capability drop、资源/时间/进程限制、凭证不进 sandbox、出站 allowlist、容器/VM 隔离、沙箱逃逸测试、宿主机与 Agent 日志分离。

### 4.8 Identity / Authorization / Human approval

真正的安全边界应在 Agent 之外可验证地执行：

- 身份绑定到用户、会话、租户和任务；
- capability token 只包含本次任务需要的能力；
- 下游服务每次请求重新检查授权；
- 高影响、不可逆、金融、外发、删除、部署动作需要审批；
- 审批 UI 不可被模型输出绕过；
- 记录“谁授权、Agent 计划什么、实际调用什么、结果是什么”。

### 4.9 Observability / Incident Response 层

至少记录：用户与租户、模型版本、prompt/context provenance、工具版本和 schema hash、完整参数、权限决策、审批、输出、状态变更、重试链和异常。

监控指标可包括：未授权工具调用率、敏感数据流、权限提升距离、异常 tool chain、循环/成本、跨租户访问、记忆写入异常、策略拒绝与绕过尝试。日志本身也要脱敏、完整性保护和访问控制。

---

## 5. 横向攻击谱系：从入口到影响

### 5.1 Prompt injection / indirect injection

攻击链：恶意内容 → context → 计划/工具选择 → 数据泄露或外部副作用。

代表性研究：

- Toyer 等的 prompt injection 攻击与防御形式化研究；
- Greshake 等对 LLM-integrated application 的间接注入展示；
- **InjecAgent**：1,054 个测试案例、17 类用户工具、62 类攻击工具；论文报告 ReAct GPT-4 在基础设定下约 24% 的攻击成功率；
- **AgentDojo**：动态环境，97 个任务、629 个安全测试案例，强调真实任务成功与安全性的联合评估。

### 5.2 Jailbreak / policy bypass

包括手工模板、自动搜索、迁移攻击、编码/多轮/角色扮演等。研究重点是攻击成功率、迁移性、语义保持、自动化成本以及防御在 adaptive attacker 下是否仍成立。

### 5.3 Poisoning / backdoor

投毒可发生在训练集、指令微调、偏好数据、RAG、memory、tool schema 和 Agent 配置。后门的危险在于正常评测可能通过，但在触发条件下改变行为。

### 5.4 Privacy / extraction

训练数据提取、PII 泄漏、membership inference、model extraction、prompt stealing 和 Agent 运行时数据外传应分开评估，因为资产、攻击权限和防御不同。

### 5.5 Tool abuse / excessive agency / confused deputy

这组问题的共同根因是：模型获得了超出任务需要的功能、权限或自主性。OWASP 2025 将 excessive agency 拆成 excessive functionality、permissions、autonomy，并建议最小扩展、最小权限、用户上下文授权、下游完整仲裁和高风险审批。

### 5.6 Supply chain

覆盖数据、模型、权重、序列化格式、依赖、Agent framework、MCP server、tool description、容器镜像和插件注册表。控制重点是 provenance、签名、SBOM、依赖扫描、版本 pin、变更检测和可回滚。

### 5.7 Availability / resource abuse

包括超长 prompt、上下文膨胀、递归 Agent、工具风暴、并发调用、昂贵推理、无限 retry 和恶意文件/网络任务。需要把 token、时间、调用次数、成本和外部副作用都纳入预算。

---

## 6. 防御不是单点，而是分层控制平面

### 6.1 设计期

- 画数据流图和 trust-boundary 图；
- 列出每个工具、凭证、数据集和持久状态；
- 做 least privilege 和 capability design；
- 明确哪些动作必须人工批准；
- 版本化 prompt、policy、tool schema、model 和 memory migration。

### 6.2 构建期

- 数据与 artifact provenance；
- 训练/微调投毒检测；
- secret scanning、依赖扫描、容器扫描；
- 合约测试：授权拒绝、跨租户隔离、schema 验证、sandbox 边界；
- 安全回归集而不是只测正常任务。

### 6.3 运行期

- 输入和检索来源标注；
- context 最小化与 secret redaction；
- 工具 allowlist、参数校验、下游授权；
- sandbox、资源预算、审批和可回滚；
- trace、审计、告警和异常检测。

### 6.4 评测期

至少同时测：

1. 任务成功率；
2. attack success rate / unsafe action rate；
3. 未授权秘密披露；
4. 未授权工具/参数/资源访问；
5. 权限提升距离；
6. 跨会话持久化与污染；
7. 防御在 adaptive attack 下的效果；
8. 安全性与可用性的 trade-off。

推荐从 AgentDojo、InjecAgent、ASB（Agent Security Bench）开始，再针对自己的工具和权限模型构造测试。ASB 覆盖 10 个场景、400+ 工具、23 类攻击/防御方法和 8 类指标；其结果提醒研究者：不同操作阶段的脆弱性并不相同，不能用单一 jailbreak 分数代表 Agent 安全。

---

## 7. 文献与工程资料：建议阅读顺序

### 7.1 第一层：建立地图

1. **Security and Privacy Challenges of Large Language Models: A Survey**：从 prompt hacking、后门/投毒、gradient leakage、membership inference、PII leakage 入门。  
   [arXiv:2402.00888](https://arxiv.org/abs/2402.00888)
2. **A Survey of Attacks on Large Language Models**：按 training-phase、inference-phase、availability/integrity 组织攻击。  
   [arXiv:2505.12567](https://arxiv.org/abs/2505.12567)
3. **NIST AI 600-1：Generative AI Profile**：把安全问题放回 AI RMF 生命周期与风险管理。  
   [NIST AI 600-1](https://doi.org/10.6028/NIST.AI.600-1)
4. **MITRE ATLAS**：用 tactics/techniques/case studies 学习威胁情报和红队语言。  
   [ATLAS](https://atlas.mitre.org/)

### 7.2 第二层：理解模型攻击

- **GCG / Universal and Transferable Adversarial Attacks on Aligned Language Models**：自动化、可迁移的 jailbreak 后缀。  
  [arXiv:2307.15043](https://arxiv.org/abs/2307.15043)
- **The Instruction Hierarchy**：让模型区分不同优先级的指令，是 prompt injection 防御的重要训练方向。  
  [arXiv:2404.13208](https://arxiv.org/abs/2404.13208)
- **A Survey on Model Extraction Attacks and Defenses for LLMs**：功能提取、训练数据提取、prompt-targeted attacks。  
  [arXiv:2506.22521](https://arxiv.org/abs/2506.22521)
- **NIST SP 800-218A**：把数据、权重、reward model、pipeline 纳入 secure software development。  
  [NIST SP 800-218A](https://doi.org/10.6028/NIST.SP.800-218A)

### 7.3 第三层：进入 Agent security

- **InjecAgent**：间接注入与工具集成 Agent 的基准。  
  [ACL 2024](https://aclanthology.org/2024.findings-acl.624/)
- **AgentDojo**：动态任务环境，评估攻击、防御、任务效用和安全属性。  
  [NeurIPS / project](https://github.com/ethz-spylab/agentdojo)
- **Agent Security Bench (ASB)**：覆盖 system prompt、user prompt、tool usage、memory retrieval 与混合攻击。  
  [arXiv:2410.02644](https://arxiv.org/abs/2410.02644)
- **From Prompt Injections to Protocol Exploits**：将 MCP、ACP、ANP、A2A 等协议纳入端到端威胁模型。  
  [arXiv:2506.23260](https://arxiv.org/abs/2506.23260)
- **Toward Secure LLM Agents**：以 agentic loop 为主线综合输入、规划、工具、记忆、监控和多 Agent 协作。  
  [arXiv:2606.10749](https://arxiv.org/abs/2606.10749)

### 7.4 第四层：工程落地

- [OWASP Top 10 for LLM Applications 2025](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OWASP AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html)
- [OWASP MCP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html)
- [OWASP Agentic AI Threats and Mitigations](https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/)
- [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/)

---

## 8. 博士阶段可以继续追的研究问题

### 8.1 指令与数据的形式化隔离

自然语言把 instruction、observation、memory、policy 混在同一 token stream 中。值得研究 typed context、provenance-aware attention、非干扰（non-interference）和可验证的 action authorization。

### 8.2 长期记忆的安全性

记忆安全应同时覆盖写入、检索、更新、删除、过期、跨租户隔离、来源可信度和回滚。短期 prompt injection 与长期 state poisoning 的威胁模型不能混用。

### 8.3 工具与协议的可组合安全

单个工具安全不代表工具链安全。需要研究跨工具数据流、confused deputy、权限传播、MCP server 间隔离、schema integrity、动态 trust 和消息级 provenance。

### 8.4 评测的现实性

未来 benchmark 应从“一次攻击是否成功”扩展到长时段、多步骤、跨会话、真实权限、真实副作用、adaptive attacker、任务效用和恢复能力。

### 8.5 Sandbox 与 Agent policy 的联合保证

模型 policy 只能提出意图，sandbox 和下游授权才执行约束。如何把 capability security、OS isolation、审计和模型规划结合为可证明或可测试的安全性质，是系统安全与 ML 的交叉前沿。

### 8.6 多 Agent 与 Agentic Web

委派会引入身份、信任、消息完整性、能力传递和错误传播问题。应将“恶意 Agent/被攻陷 Agent”作为一等攻击者，而不只测试恶意用户。

---

## 9. 一页式研究检查表

研究或实现一个 Agent 时，按下面顺序问：

1. Agent 能访问哪些数据、工具、凭证和外部系统？
2. 哪些内容是 instruction，哪些只是 untrusted data？系统如何强制区分？
3. 每个工具的最小功能、最小权限、身份和资源 scope 是什么？
4. tool description、schema、返回值和 memory 是否可被攻击者影响？
5. 哪些动作不可逆、昂贵、对外发送或具有高影响？审批在哪里执行？
6. sandbox 能否限制文件、网络、进程、凭证和资源？
7. 发生注入后，攻击能否跨步骤、跨会话、跨 Agent 传播？
8. 是否记录完整 trace，并能定位“哪个输入导致哪个计划和副作用”？
9. 是否有正常任务、恶意任务、adaptive red-team、回归和恢复测试？
10. 结论属于论文证据、工程指南、公开事件，还是仍未验证的假设？

---

## 10. 来源质量说明

- **同行评审/学术论文**：用于攻击定义、实验结果、基准和防御效果；需要注明实验设定，不能泛化为所有模型。
- **综述/SoK**：用于建立分类和发现文献，不自动证明某个攻击在生产环境成立。
- **标准与工程指南**：用于控制建议、治理和实践基线；OWASP 分类是社区共识，不等于形式化漏洞证明。
- **博客与事件报告**：用于案例、时间线和真实攻击面；应与论文或原始公告交叉核验。

本文保留现有的案例笔记不变；后续可把已核验事件按“Surface → Technique → Asset → Impact → Control → Source”补入相关文件。

---

## 11. 新增参考资料：OpenAI–Hugging Face 事件与 ChatGPT Security

### 11.1 OpenAI–Hugging Face Agent Intrusion

该事件应作为 **Agentic cyber intrusion** 研究案例，而不是简单归入 prompt injection：其攻击链涉及 sandbox escape、网络出口旁路、凭证滥用、提权、横向移动、数据处理管线 RCE 和 benchmark 数据访问。

详细官方声明、技术时间线、第三方来源及证据边界见：

- [OpenAI-Hugging Face Agent Intrusion - Official Sources](12.REF/OpenAI-Hugging%20Face%20Agent%20Intrusion%20-%20Official%20Sources.md)

### 11.2 ChatGPT Security 链接的校正与分析

`https://learn.chatgpt.com/docs/security` 当前实际指向 Codex Security / Security administration 文档。它适合研究代码 Agent 的 sandbox、网络、工具、审批、仓库供应链和审计边界，但不应被表述为覆盖整个 ChatGPT 的通用安全白皮书。

- [ChatGPT Security - Codex Security Analysis](12.REF/ChatGPT%20Security%20-%20Codex%20Security%20Analysis.md)
