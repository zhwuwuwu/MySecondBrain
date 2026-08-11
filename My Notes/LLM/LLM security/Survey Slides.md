# LLM / Agent Security 汇报底稿

> 面向对象：高级经理、产品经理、OEM/平台负责人
>
> 汇报目标：让非安全背景的决策者理解为什么本地类 Claw Agent 需要安全能力、应该从哪里开始做、模型护栏能带来什么，以及会对延迟、回答质量和本地资源造成什么影响。
>
> 本次汇报暂不预设结论，只聚焦两个决策问题：**产品是否需要把 Security 做成正式 feature？如果需要，应该先完善 Agent platform 的权限/隔离/恢复能力，还是先加入 safety 判断模型？**

---

## 汇报开场：今天需要做什么决策？

本次不是讨论“要不要把模型变得更保守”，而是讨论产品是否要支持一个可以：

- 读取本地文件、长期记忆和工作区；
- 调用 Shell、浏览器、MCP、代码执行和业务 API；
- 修改文件、安装依赖、发送消息或访问网络；
- 在无人逐步确认的情况下连续执行多步任务；
- 由 OEM 预装、交付给普通 C 端用户使用的本地 Agent。

需要管理层首先判断两件事：

1. 是否批准 Agent Security 作为产品能力，而不是上线前的一次性测试；
2. 如果批准，第一阶段是优先投入确定性平台控制，还是优先投入安全判断模型。

### 两条待比较的建设路径

| 路径 | 主要投入 | 希望解决的问题 | 需要验证的风险 |
|---|---|---|---|
| **Platform-first** | 权限、工具网关、workspace sandbox、网络、凭证、审批、trace、回滚 | 即使模型判断错误，也不能越过真实系统边界 | 覆盖是否完整、产品复杂度、开发周期、合法任务体验 |
| **Model-first** | SingGuard / SingGuard-NSFA 推理、策略、阈值、模型服务 | 先增强对复杂内容、意图偏离和危险动作的识别 | 误报/漏报、延迟、资源占用、模型不可用时如何阻断 |

本次汇报后再基于风险覆盖、收益、性能和实施成本提出建议，不在开场预先假定哪条路径正确。

---

## LLM Security Domain

### 01 Introduction：为什么需要讨论 Security feature？

传统聊天模型主要输出文字。风险通常是回答错误、内容不合适或用户被误导。

Agent 在模型外增加了规划循环、记忆、检索、工具、身份和运行时。于是模型的一次错误理解可能变成真实副作用：读文件、写文件、发请求、修改配置、执行命令或对外发送数据。

因此，LLM/Agent Security 的工作定义是：

> **保护模型、数据、凭证、工具、运行时状态和外部系统的机密性（C）、完整性（I）与可用性（A），同时保证 Agent 行为符合授权目标。**

需要讨论 Security feature 的原因是：一旦 Agent 能够执行外部动作，产品就需要回答高影响动作是否：

- 不会因为一段不可信内容就自动发生；
- 在执行前有明确授权和风险判断；
- 发生问题时能被发现、隔离、吊销和回滚；
- 能保留证据，解释为什么放行或阻断。

#### 基础边界：Security / Safety / Privacy / Reliability / Robustness

| 概念 | 关注的问题 | 类 Claw 例子 | 主要控制 |
|---|---|---|---|
| **Security 安全** | 对抗者或未授权路径是否破坏 C/I/A、扩大能力或越过授权边界 | Prompt injection 诱导 Agent 读取密钥并发邮件 | 权限、策略、隔离、监控 |
| **Safety 安全性** | 即使没有恶意攻击，系统是否产生不可接受伤害 | 用户意图正常，但 Agent 误删文件 | 预览、确认、回滚、限制副作用 |
| **Privacy 隐私** | 用户是否能控制数据的收集、使用、记忆和披露 | 私密对话被写入长期 memory 或跨任务泄露 | 数据最小化、隔离、脱敏、保留策略 |
| **Reliability 可靠性** | 系统能否稳定工作，失败后能否恢复 | 工具失败后无限重试、任务无法回滚 | 超时、预算、重试策略、恢复 |
| **Robustness 鲁棒性** | 输入、环境、工具或模型变化时，安全性质是否仍保持 | 工具返回格式变化后仍执行越权动作 | 约束、schema、回归测试、fallback |

这五个概念会交叉，但不能混用：

- 拒答率不等于安全性；
- 安全性不等于回答正确性；
- 可靠运行不等于权限正确；
- 内容审核不等于运行时隔离。

---

### 02 问题描述维度：从“有风险”到可执行的工程问题

建议把每个问题写成一条完整链路：

```text
攻击面（surface）
→ 攻击技术（technique）
→ 资产（asset）
→ 影响（impact）
→ 控制（control）
→ 证据（evidence）
```

#### 示例：恶意网页诱导 Agent 外传文件

```text
网页/浏览器工具结果
→ indirect prompt injection
→ 本地文件、Agent 工具权限、网络出口
→ 文件泄露和用户信任损失
→ 内容检测 + 工具授权 + 网络 allowlist + sandbox
→ 完整上下文、工具参数、网络请求和文件访问 trace
```

#### 需要明确的四个边界

1. **攻击者是谁**：普通用户、恶意网页作者、被入侵的 Skill/MCP、供应链攻击者或恶意 Agent；
2. **资产是什么**：系统 prompt、模型、用户文件、数据库、API key、长期 memory 和外部业务动作；
3. **信任边界在哪里**：用户指令与外部内容、模型与工具、Agent 与 Agent、workspace 与宿主机；
4. **授权是什么**：谁允许 Agent 以谁的身份对哪些资源做什么动作，授权持续多久。

#### 管理层可理解的风险指标

- Unauthorized Action Rate：未授权动作比例；
- Secret Disclosure Rate：秘密被读取或外传的比例；
- Denied-call Bypass Rate：被拒绝后通过重试、改写或其他工具绕过的比例；
- Privilege Escalation Distance：从初始能力到最终权限的扩大距离；
- Cross-tenant Access Rate：跨用户/跨项目访问比例；
- Recovery / Rollback Success Rate：发现异常后隔离、吊销和恢复成功率。

---

### 03 Model Security & Agent Security

#### Model Security：模型生命周期安全

按生命周期主要关注：

1. 数据获取、清洗、标注、授权和隐私；
2. 预训练、微调、对齐和评测配置；
3. 权重、checkpoint、量化文件和依赖供应链；
4. 推理 API、认证、限流、日志和多租户隔离；
5. 模型行为，如越狱、Prompt Injection、数据提取和拒答绕过。

典型控制：数据 provenance、敏感数据最小化、依赖 hash/signature、模型工件访问控制、推理服务认证和抽取测试。

#### Agent Security：运行时安全

Agent 在 Model Security 之外增加了：

- Harness / Orchestrator 和任务目标；
- Prompt/context 组装；
- Memory/RAG/状态持久化；
- Tool/API/MCP/connectors；
- 身份、权限、审批和用户确认；
- Sandbox、宿主机、网络和凭证；
- 监控、回滚和事件响应。

一个模型输出在聊天产品中可能只是错误文字，在 Agent 中可能变成：

```text
untrusted content
→ context
→ plan
→ tool call
→ external effect
→ memory / trace
```

本地类 Claw 的主要安全重点因此是 Agent Security，而不是只做模型拒答率优化。

---

### 04 横向攻击谱系：风险如何穿过多个边界？

| 攻击面 | 常见风险 | 影响 |
|---|---|---|
| 用户输入/外部内容 | Prompt Injection、越狱、恶意网页/文档 | 改变 Agent 目标或诱导调用工具 |
| Harness/规划循环 | 目标错配、过大 retry budget、无限循环 | 长程自主行为、资源耗尽 |
| Memory/RAG | Memory poisoning、跨任务污染、恶意检索结果 | 跨会话、跨 Agent 持久化传播 |
| Skill/插件/MCP | 恶意工具、工具结果注入、供应链依赖 | 获得超出预期的代码/网络/数据能力 |
| Shell/代码执行 | 任意命令、危险参数、安装脚本 | 文件破坏、凭证窃取、主机风险 |
| 文件/数据库/API | 越权读写、查询变写入、跨项目访问 | 数据泄露、状态篡改 |
| 网络/运行时 | SSRF、metadata、内网访问、sandbox escape | 凭证和内部服务暴露 |
| 输出/外发 | 敏感信息泄露、错误发布、错误发送 | 用户隐私和业务损失 |

高风险事件通常不是一个漏洞，而是一条攻击链。例如：

```text
外部内容注入
→ Agent 获得错误目标
→ 读取本地 secret
→ 工具参数隐藏外传
→ 网络出口允许请求
→ 结果进入 memory 或外部系统
```

OpenAI–Hugging Face 事件的研究材料也说明：高能力 Agent、弱约束评测编排、代理/代码执行工具、数据处理管线、凭证和网络边界可以组合成跨组织攻击链。结论不是“模型有恶意人格”，而是多个传统边界被 Agent 组合利用。

---

### 05 防御体系分层：判断/扫描 vs 执行/阻断

安全能力至少分为四类：

1. **Detection / Evaluation**：发现风险、判断意图偏离、分析 trace；
2. **Authorization**：结合用户、任务、资源和时间判断是否授权；
3. **Prevention / Enforcement**：在动作执行前用确定性策略阻断；
4. **Recovery / Response**：吊销凭证、隔离运行时、回滚状态、保留证据。

推荐架构：

```text
User goal
   ↓
Intent / scope binding
   ↓
Agent planner（不可信规划器）
   ↓
Detection / Evaluation（规则、模型、人工）
   ↓
Capability gateway / Policy engine（确定性授权）
   ↓
Tool / API / Filesystem / Network adapter
   ↓
Sandbox / Runtime isolation
   ↓
External effect

All stages → Audit / Trace / Detection / Recovery
```

### 判断/扫描层能做什么？

- 识别 Prompt Injection、危险内容和敏感信息；
- 判断计划与用户意图是否偏离；
- 对工具名称、完整参数、工具结果和最终输出评分；
- 对历史 Agent trace 做离线审计和风险聚类；
- 为人工审批提供风险解释和证据。

### 判断/扫描层不能单独做什么？

- 不能替代文件系统权限、API authorization、IAM；
- 不能保证容器不会逃逸；
- 不能保证网络不会访问内网或 cloud metadata；
- 不能替代源码、依赖、配置和渗透测试；
- 不能把“模型判断 safe”变成“允许执行任意工具”。

### 执行/阻断层必须做什么？

- 工具 allowlist 和参数约束；
- 路径、域名、端口、命令和资源预算限制；
- 读取、删除、部署、外发等高影响动作审批；
- workspace 与宿主机、凭证、网络的隔离；
- timeout、retry budget、kill switch、吊销和回滚。

---

## Agent Unsafe Action Judge

### 06 基础定义：什么是 Unsafe Action？

Unsafe Action 不是“模型说了不该说的话”这么简单，而是：

> **Agent 的实际或计划动作扩大、转移或持久化了原本被委托的权限，或产生了未被授权的高影响副作用。**

判断一个动作要同时回答三个问题：

#### 1. Agent 想做什么？——Intent

计划和工具参数是否服务于用户任务？

例：用户要求“总结当前项目”，Agent 却准备安装依赖、访问生产数据库或发送邮件，属于意图偏离。

#### 2. Agent 被允许做什么？——Authorization

用户目标不是权限授予：

> “总结项目”不自动授权读取 `.env`、访问其他项目、访问外网、修改文件或发送外部请求。

#### 3. 实际发生了什么？——Observed Effect

必须观察：

- 工具调用和完整参数；
- 文件、数据库和网络访问；
- 凭证使用；
- memory 写入；
- 重试链和子 Agent 调用；
- 最终状态改变和外部副作用。

### Agent 全流程中的判断位置

```text
用户请求 / 外部内容
        ↓  输入风险判断
Intent / Scope Binding
        ↓
Agent Planner
        ↓  计划与工具参数判断
Capability Gateway / Policy Engine
        ↓  确定性授权与审批
Tool Execution in Sandbox
        ↓  结果与外发判断
Memory / External Effect / Audit Trace
```

判断越接近高影响动作，越应该结合确定性策略；不能只在最终回答生成后才检查。

---

### 07 Agent 安全维度拆解：不是简单的 Prompt + Classifier

#### 1. Authorization Boundary：授权越界

允许读却删除、访问项目 B、普通用户调用管理员 API、查询变成写入。

#### 2. Intent Alignment：意图偏离

动作不再服务于用户任务，或 Agent 为完成任务擅自增加目标。适合用模型 judge 或人工复核，但不能直接决定最终权限。

#### 3. Excessive Agency：能力过大

- Excessive Functionality：任意 Shell、任意 URL、任意 SQL；
- Excessive Permissions：读工具拥有写权限，通用高权限身份代替用户身份；
- Excessive Autonomy：高影响动作无需审批、确认或回滚。

#### 4. Confidentiality：数据机密性违规

读取、推断、输出或外传 `.env`、SSH key、云凭证、跨租户数据、其他任务 memory 和工具返回的秘密。

```text
secret file → model context → tool argument → external API
```

没有显式 declassification，应拒绝这条数据流。

#### 5. Integrity：动作完整性违规

动作类型可能被允许，但目标、参数或内容错误：修改错文件、给错账户付款、部署未经审批代码、把不可信文档命令作为执行参数。

#### 6. External Side Effect：外部副作用

每个动作标注：

- reversible / irreversible；
- low / high impact；
- internal / external；
- read / write / delete / send / execute；
- 单步影响还是累积影响。

删除、支付、发邮件、部署、改权限和公开发布通常需要审批。

#### 7. Runtime Boundary：运行时突破

访问 workspace 外文件、宿主机凭证、云 metadata，利用包代理或浏览器间接出网，或逃逸 sandbox。这类问题必须由 OS、网络和 runtime 强制执行。

#### 8. Persistence：持久化与跨步骤传播

检查恶意内容是否进入长期 memory、RAG、下一会话、其他 Agent、工具配置、日志或缓存。

#### 9. Availability：可用性与资源滥用

包括无限循环、retry storm、工具调用风暴、token/费用耗尽、进程/内存/磁盘耗尽和大规模云资源操作。

### Unsafe Action 的统一判断输入

```text
动作类型
+ 目标资源
+ 完整参数
+ 当前状态
+ 用户授权
+ 历史动作
+ 预计副作用
```

而不是只把一段自然语言回答交给一个分类器。

---

### 08 Security Envelope：Agent 的安全包络

Security Envelope 是 Agent 在特定用户、任务、资源、时间和副作用范围内允许活动的边界。

至少绑定：

| 维度 | 要回答的问题 |
|---|---|
| Identity | 谁在请求？Agent 以谁的身份执行？ |
| Intent | 当前任务的目标和禁止事项是什么？ |
| Capability | 允许读、写、删、发、执行哪些动作？ |
| Resource | 允许访问哪些路径、域名、数据库、MCP 和 Skill？ |
| Time | 授权有效多久，是否需要重新确认？ |
| Impact | 哪些动作可自动执行，哪些必须审批？ |
| Recovery | 如何吊销、隔离、回滚和保留证据？ |

### 例子：用户说“整理项目”

```yaml
identity: current_user
intent: summarize_and_format_project
allowed_paths:
  - current_workspace
allowed_operations:
  - read
  - write_markdown
blocked_operations:
  - read_secrets
  - delete
  - external_upload
  - install_package
network: deny_by_default
approval_required:
  - delete
  - external_send
  - deploy
expires_in: 30m
```

模型可以帮助理解 intent、发现风险和解释决策，但 Security Envelope 必须落在确定性策略、工具网关、操作系统权限、网络和 sandbox 上。

---

## Execution：为什么做、从哪里开始、怎么落地

### 09 带来哪些好处？

#### 对本地部署类 Claw 应用的 C 端用户

- **降低恐惧成本**：用户知道 Agent 不会因为一段网页内容就随意读取和外发本地文件；
- **保护个人资产**：本地文档、照片、浏览器会话、SSH key、云凭证和长期 memory 有明确边界；
- **高影响动作可控**：删除、发送、部署和改权限能够预览、确认或回滚；
- **更少隐形数据外流**：网络出口和工具参数受控；
- **问题可解释**：发生拒绝时能够说明是哪个策略、哪个资源和哪个动作触发。

#### 对部署本地应用的 OEM

- **降低事故成本**：减少数据泄露、越权、恶意 Skill/MCP 和供应链事件；
- **提升交付可信度**：安全策略、审计日志、版本化配置可以成为产品合同的一部分；
- **减少售后定位成本**：有完整 trace，可以区分模型问题、工具问题和平台权限问题；
- **支持产品分层**：普通 SKU 使用轻量规则，高级 SKU 提供本地安全模型和更强审计；
- **形成差异化**：将“可控 Agent”作为 OEM 交付能力，而不是只比较模型回答效果。

#### 需要避免的错误预期

- 安全层不会自动提升模型事实正确性；
- 模型护栏不会替代源码和依赖漏洞扫描；
- 安全能力会有误报、漏报和资源成本；
- 一次 benchmark 不能证明所有用户场景都安全。

---

### 10 开始从哪做？三种分类方式

#### 分类 1：按生命周期和供给面

1. **Model / Data supply chain**：模型、权重、数据集、依赖、Skill、MCP 来源和版本；
2. **Agent runtime**：Harness、Prompt/context、Memory/RAG、工具、身份、审批、Sandbox；
3. **External effect**：文件、数据库、网络、消息、部署和业务 API；
4. **Observability / Response**：日志、告警、吊销、隔离、回滚和复盘。

#### 分类 2：按“判断/扫描”与“执行/阻断”

| 层 | 主要能力 | 示例 |
|---|---|---|
| 判断/扫描 | 模型、规则、红队、日志分析、人工复核 | SingGuard、SingGuard-NSFA、静态扫描 |
| 授权 | 用户、任务、资源、时间和副作用绑定 | Policy Gateway、Capability Gateway |
| 执行/阻断 | OS、网络、容器、工具和凭证限制 | Sandbox、allowlist、IAM、审批 |
| 恢复 | 吊销、隔离、回滚、重建、证据保留 | Incident Response |

#### 分类 3：按同一切入点的不同威胁类型

以“工具调用”为例，同时检查：

- Prompt Injection：工具参数是否来自不可信内容；
- Authorization：用户是否有权调用；
- Data Leakage：参数是否带出 secret；
- Integrity：目标和参数是否被篡改；
- Runtime：路径和网络是否越界；
- Availability：调用是否超过时间、次数和资源预算。

### 建议的优先级

#### P0：没有这些底座，不建议上线自动执行

- 完整记录用户意图、Agent 计划、工具调用、参数、结果和副作用；
- workspace 与宿主机敏感文件隔离；
- Shell、文件、网络、MCP 工具的 allowlist 和参数约束；
- secret 不进入不必要的模型上下文；
- 高影响动作审批、timeout、kill switch 和恢复路径。

#### P1：让平台能发现和判断风险

- 建立真实攻击样本和合法任务样本；
- 用规则检查高置信边界；
- 用 SingGuard-NSFA 审计输入、计划、工具动作和结果；
- 有图片/图文输入时加入 SingGuard；
- observe mode 运行，先测误报、漏报、延迟和资源。

#### P2：规模化治理

- Memory/RAG 写入审核和隔离；
- Skill/MCP 来源、权限、版本和行为审计；
- 红队回归集、跨设备 benchmark 和 OEM 策略管理；
- 告警、凭证吊销、环境重建和可审计报告。

---

### 11 思路 1：使用模型同步判断多种风险类型

#### 11-1 模型一：SingGuard

**定位**：策略可适配的多模态 LLM 安全护栏。

**适合判断**：

- 文本、图片、图文组合是否违反当前 policy；
- 用户输入、网页截图、文档和助手输出是否包含风险；
- 是否存在 Prompt Injection、越狱、敏感内容和自定义业务政策违规。

**输入**：文本、图片、图文、当前自然语言 `policy`。

**输出**：`safe/unsafe`、匹配风险类别、策略相关分析。

**基础模型与规模**：Qwen3-VL，2B/4B/8B，Apache 2.0。

**训练简介**：统一文本/多模态安全数据上的 SFT；论文报告使用 Fast-Slow Decoupled DAPO 进一步优化动态推理。完整训练集来源、规模和配比没有完全公开。

**Benchmark**：SingGuard-Bench 约 56,340 样本。论文报告 SingGuard-8B：

| 方向 | 指标 |
|---|---:|
| Image Safety | F1 0.903 |
| Text Query | F1 0.874 |
| Multilingual Query | F1 0.899 |
| Multimodal Safety | F1 0.877 |
| Text Response | F1 0.893 |
| Multilingual Response | F1 0.893 |
| Dynamic Rule，2,000 样本 | policy-following accuracy 0.7415 |

**类 Claw 集成位置**：

```text
用户输入 / 网页 / 图片 → SingGuard → Agent
Agent 输出 / 待展示内容 → SingGuard → 用户
```

它可以检查工具调用的文字化描述，但不能替代实际授权。它不是源码漏洞扫描器，也不能保证文件权限、网络隔离、API 越权或容器安全。

**部署考虑**：多模态模型需要图像处理和更高内存/显存；当前公开资料不提供可直接承诺的本地端到端 SLA。应按设备 SKU 测量模型加载、图片分辨率、上下文长度、并发和 P95 延迟。

#### 11-1 模型二：SingGuard-NSFA

**定位**：面向 Agent 操作风险的文本护栏框架。

**适合判断**：

- Prompt Injection / Jailbreak；
- 恶意代码与网络攻击；
- 敏感信息窃取和外传；
- 危险操作与工具滥用；
- 资源滥用；
- 危险动作生成和敏感信息泄露。

**输入**：用户请求、外部内容、Agent 计划、工具名称、完整参数、工具结果和最终输出的文本化表示。

建议区分：

```text
<untrusted_input>用户请求或外部网页内容</untrusted_input>
<untrusted_output>Agent 计划、工具调用或最终回答</untrusted_output>
```

**输出**：风险类别、生成式风险分析，或实时分类模式下的风险概率。

**基础模型与规模**：Qwen3.5-Base，0.8B/2B/4B/9B，Apache 2.0。

**训练简介**：

1. 基于风险分析数据进行 SFT；
2. 冻结 SFT 后的骨干，为不同风险域训练轻量 MLP 分类头；
3. 新增风险可以只训练新分类头，不必重训原有骨干。

**风险体系与 Benchmark**：7 个一级域、28 个二级风险、185 个三级变体；93,000+ purpose-built multilingual samples，覆盖 133 种语言；3,435 cross-source samples 来自 5 个公开 Agent 安全数据集。论文/仓库报告：

| 方向 | 指标 |
|---|---:|
| Purpose-built multilingual | 四种规模均 ≥94% F1 |
| Cross-source | 9B F1 91.29% |
| 竞品对比 | 高 6–12 个 F1 百分点 |
| Real-Time Classification | A100 条件约 45–57 ms/sample |

purpose-built 数据集以 [NSFA_Benchmarks](https://huggingface.co/datasets/inclusionAI/NSFA_Benchmarks) 发布；cross-source 的 5 个底层数据集名称和完整开放范围需要以实际资料为准。

**两种集成模式**：

- Generative Reasoning：离线审计、复杂案例、事件调查、人工复核，接近 LLM-as-a-Judge；
- Real-Time Classification：冻结骨干上的分类头直接输出风险概率，适合在线工具调用前检查，不需要先生成解释再解析。

**类 Claw 集成位置**：

```text
用户输入 → NSFA 输入检查 → Agent Planner
                         ↓
              NSFA 检查计划与工具参数
                         ↓
              确定性 Policy Gateway
                         ↓
                    执行或拒绝
                         ↓
              NSFA 检查结果和外发内容
```

**部署考虑**：0.8B/2B 是本地 pilot 的优先候选；需测实际模型加载内存、CPU/GPU、并发、上下文长度、P50/P95 延迟和超时行为。A100 的 45–57 ms 只能作为参考，不是本地设备 SLA。

#### 11-1 两个模型的互补关系

| 问题 | SingGuard | SingGuard-NSFA |
|---|---|---|
| 网页/图片/图文内容是否安全 | 主要能力 | 不适合的主要方向 |
| Prompt Injection | 可作为内容层判断 | 主要能力之一 |
| 工具调用风险 | 可检查文字化描述 | 主要能力 |
| 源码/依赖漏洞 | 不负责 | 不负责 |
| 授权和真实阻断 | 不负责 | 不负责 |
| 多模态输入 | 支持 | 主要文本 |
| 在线低延迟分类 | 需看具体实现和本地部署 | 分类头路径适合 |

### 11-2 试点计划

#### 阶段 0：建立基线（1 周）

- 记录现有 Agent 的用户请求、计划、工具调用、结果、memory 写入和外部副作用；
- 统计普通任务的 P50/P95 延迟、回答成功率、工具成功率、内存峰值和 CPU/GPU 占用；
- 标注高影响动作和敏感资源。

#### 阶段 1：离线审计与数据集（1–2 周）

- 用 NSFA Generative Reasoning 审计历史 trace；
- 构造合法任务、Prompt Injection、越权读写、secret 外传、MCP/Skill 污染、资源滥用样本；
- 对每条样本记录人工标签、风险类别、动作影响和预期处理。

#### 阶段 2：Shadow Mode（1–2 周）

- NSFA 在线分类接在工具调用前，但不改变执行结果；
- 有多模态入口时，SingGuard 接在外部内容输入和最终输出两侧；
- 记录模型判断、规则判断、实际副作用和人工标签的差异；
- 测模型大小与设备 SKU 的内存、速度和准确率曲线。

#### 阶段 3：有限 Enforce（1 周）

- 只阻断高置信度、高影响、不可逆的动作；
- 读取 secret、外部上传、删除、部署、改权限默认拒绝或人工审批；
- 低风险普通读操作保持原有路径；
- 明确模型超时、模型崩溃和 Policy Gateway 不可用时的 fallback。

#### 试点验收指标

| 维度 | 指标 |
|---|---|
| 安全 | Secret Disclosure Rate、Unauthorized Action Rate、Denied-call Bypass Rate |
| 判断质量 | Precision、Recall、F1、误报率、漏报率、按风险域拆分的混淆矩阵 |
| 体验 | Agent 成功率、合法任务拒绝率、用户确认率、回答正确性变化 |
| 性能 | 额外 P50/P95 延迟、模型加载时间、峰值内存/显存、CPU/GPU、功耗 |
| 稳定性 | timeout、fallback、重试风暴、模型崩溃、断网和低资源场景 |
| 恢复 | 吊销凭证、隔离运行时、回滚文件/状态、保留证据的成功率 |

---

### 12 思路 2：针对威胁类型优化 Platform

模型判断不能替代平台控制。对于本地类 Claw，建议至少建设以下平台能力：

#### 1. Shell 越权：命令与参数能力网关

- 将 Shell 按 read-only、workspace-write、network、privileged 分级；
- 默认命令 allowlist，而不是给任意 Shell；
- 对 `rm`、权限修改、进程控制、包安装和网络上传设置单独策略；
- 工具调用前展示完整命令和目标路径；
- 高影响命令要求确认，并记录实际执行参数。

#### 2. Workspace Sandbox：文件、进程和网络隔离

- Agent 默认只能访问当前 workspace；
- 明确禁止 `.env`、SSH key、云 credential、浏览器 cookie 等敏感路径；
- 限制进程、内存、磁盘、时间和子进程；
- 网络默认 deny，按域名/端口 allowlist；
- 避免把宿主机 Docker socket、metadata、全局凭证暴露给 Agent。

#### 3. MCP / Skill 供应链治理

- 记录来源、版本、hash、作者、权限和依赖；
- 安装前静态扫描 README、代码、安装脚本和工具 schema；
- 运行时限制 MCP/Skill 能访问的文件、网络和身份；
- 工具结果视为不可信 observation，不能直接改变系统目标；
- 高权限 Skill 需要签名、审批和可撤销能力。

#### 4. Memory / RAG 持久化隔离

- 区分用户明确保存与 Agent 自动写入；
- 长期 memory 写入前检查 Prompt Injection、秘密和越权指令；
- 按用户、任务和租户隔离 memory/RAG；
- 为 memory 保留来源、时间、版本和删除/回滚能力；
- 不让低可信内容自动成为未来会话的高可信系统指令。

#### 5. Identity / Secret / Data Flow 控制

- 采用短期、最小权限、按工具和任务隔离的凭证；
- secret 尽量不进入模型上下文；
- 读取、解密、外发采用独立能力和显式审批；
- 对 `secret file → context → tool argument → external API` 做数据流阻断；
- 对外部请求做域名、内容、大小和频率限制。

#### 6. Observability / Recovery

- 关联用户、任务、模型版本、上下文来源、工具调用、参数、审批、文件变更和网络请求；
- 发现异常后可以 kill Agent、吊销 token、隔离 workspace、重建 runtime；
- 对 memory、文件和配置保留版本，支持回滚；
- 让用户和 OEM 能看到“发生了什么、为什么阻断、如何恢复”。

### 思路 1 与思路 2 的关系

```text
模型护栏：发现“这看起来危险”
平台策略：决定“是否授权”
运行时隔离：保证“即使判断错也做不到”
恢复系统：保证“发生异常后能止损和回滚”
```

不能只选模型或只选平台：

- 只有模型，没有平台：模型漏判时仍可能执行危险动作；
- 只有平台，没有判断：规则难以理解复杂意图和外部内容；
- 没有恢复：一次误操作可能变成不可逆事故。

---

## 性能影响与产品取舍

### 可能增加的成本

1. **延迟**：同步调用模型会增加 P50/P95；多次检查比一次检查更明显；
2. **本地资源**：模型权重、KV cache、图片编码和并发会占用内存/显存/CPU；
3. **回答体验**：误报会导致合法任务被拒绝、额外确认或 fallback；
4. **产品复杂度**：需要配置策略、阈值、超时、审批、日志和回滚；
5. **可用性风险**：护栏模型加载失败或资源不足时，必须决定 fail-open 还是 fail-safe。

### 可能带来的正向收益

- 降低 secret 泄露、越权修改和错误外发的概率；
- 减少高影响事故的潜在损失，而不是提升普通回答速度；
- 通过完整 trace 缩短问题定位和售后时间；
- 让 OEM 能够按设备 SKU 交付不同安全等级；
- 将安全策略变成可配置、可测试、可审计的产品能力。

### 设计建议：不要把所有检查都放进同步主路径

| 场景 | 推荐方式 |
|---|---|
| 普通低风险对话 | 轻量规则或异步审计，不阻塞主回答 |
| 外部网页、图片、文档 | SingGuard 输入侧检查，按风险决定是否继续 |
| 工具调用前 | NSFA 分类头 + 确定性 Policy Gateway |
| 删除、外发、部署、改权限 | 强制审批或默认拒绝 |
| 历史日志和复杂攻击链 | NSFA Generative Reasoning 离线审计 |
| 模型超时/不可用 | 按动作风险选择 fallback；高影响动作 fail-safe |

### 必须自己测量的本地 benchmark

不能直接使用论文中的 A100 45–57 ms 作为本地产品承诺。至少建立以下矩阵：

| 变量 | 测试维度 |
|---|---|
| 模型 | NSFA 0.8B/2B/4B；SingGuard 2B/4B/8B（按多模态需求选择） |
| 设备 | 不同内存/显存 SKU、CPU-only 与 GPU/NPU |
| 输入 | 短文本、长上下文、图片、工具参数和历史 trace |
| 并发 | 单用户、多个 Agent、后台审计并行 |
| 指标 | P50/P95 延迟、峰值内存、CPU/GPU、功耗、加载时间、吞吐 |
| 质量 | Precision、Recall、F1、漏报、误报、合法任务拒绝率 |
| 稳定性 | 超时、OOM、模型崩溃、断网、低电量和 fallback |

最终上线门槛应由产品定义：高影响安全风险优先降低漏报；普通合法任务则控制误报和体验损失。

---

## 最终建议：面向管理层的决策页

### 是否要做？

**要做。** 只要本地类 Claw 能访问用户文件、记忆、网络、Shell、MCP 或外部 API，Agent Security 就不是可选的“锦上添花”，而是可控交付的基础能力。

### 从哪里开始？

1. 先建设 P0 平台边界：权限、sandbox、网络、凭证、trace、审批和恢复；
2. 用真实 Agent trace 建立基线和攻击样本；
3. 先以 NSFA 0.8B/2B 做离线审计和 shadow mode；
4. 有多模态内容时再加入 SingGuard；
5. 只对高置信度、高影响、不可逆动作进行早期阻断。

### 两个模型是否都要做？

- **纯文本、工具型个人 Agent**：优先 NSFA，SingGuard 延后；
- **网页/截图/图片/图文产品**：NSFA + SingGuard 形成内容层与动作层互补；
- **资源受限设备**：先做规则和平台边界，再评估 NSFA 0.8B 是否可接受；
- **OEM 多 SKU**：用 benchmark 决定不同 SKU 的模型大小和安全等级，不强制所有设备加载最大模型。

### 建议批准的 pilot 结果

4–6 周后应能回答：

- 哪些真实风险在我们的 Agent 中最常见；
- NSFA/SingGuard 对这些风险的 precision、recall 和漏报是什么；
- 增加护栏后的 P50/P95 延迟和本地资源开销是多少；
- 合法任务误报和回答正确性受到多大影响；
- 哪些动作可以自动放行，哪些必须审批或默认拒绝；
- 是否值得将模型能力产品化为 OEM 安全等级和平台能力。

> **最终判断标准不是“模型 benchmark 最高”，而是：在可接受的本地性能成本下，是否显著减少高影响未授权动作，并且平台能够阻断、解释和恢复。**

---

## 主要资料

- [00 LLM 与 Agent Security 总览](00%20LLM%20与%20Agent%20Security%20总览.md)
- [Task Decomposition：如何判断 Agent 的 Unsafe Action](04%20Task%20Decomp%20-%20judge%20agent's%20unsafe%20action.md)
- [SingGuard 说明](03%20蚂蚁集团%20AI%20Security%20产品地图/02%20SingGuard.md)
- [SingGuard-NSFA 说明](03%20蚂蚁集团%20AI%20Security%20产品地图/03%20SingGuard-NSFA.md)
- [AgentAegis 说明](03%20蚂蚁集团%20AI%20Security%20产品地图/04%20AgentAegis.md)
- [OpenAI–Hugging Face Agent 入侵全过程](01%20OpenAI–Hugging%20Face%20Agent%20入侵全过程.md)
- [SingGuard GitHub](https://github.com/inclusionAI/Sing-Guard)
- [SingGuard 论文](https://arxiv.org/abs/2606.22873)
- [SingGuard-NSFA GitHub](https://github.com/inclusionAI/SingGuard-NSFA)
- [SingGuard-NSFA 论文](https://arxiv.org/abs/2607.13081)
- [NSFA_Benchmarks](https://huggingface.co/datasets/inclusionAI/NSFA_Benchmarks)
