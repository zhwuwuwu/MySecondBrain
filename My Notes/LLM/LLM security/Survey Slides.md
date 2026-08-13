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

### 05 Agent 全生命周期安全架构：把安全决策放到副作用发生之前

目标不是让模型独自判断“安全/不安全”，而是在 Agent 从不可信内容到真实副作用的每一步，将**用户目标、身份权限、资源范围和可恢复性**绑定为可执行约束。模型或规则负责发现风险；运行时策略负责决定是否真的能做。

```text
不可信输入 / 上下文
        ↓  来源标识、注入检测、指令与数据分离
目标与计划
        ↓  Intent / Scope Binding、计划预算、敏感动作预检
短期状态与长期记忆
        ↓  写入隔离、provenance、污染检测、租户/任务边界
工具与动作执行
        ↓  Capability Gateway、参数校验、sandbox、网络与凭证约束
输出与数据外发
        ↓  数据流检查、脱敏 / declassification、发送前确认
        ↓
Audit Trace → 异常检测 → 隔离 / 吊销 / 回滚 / 复盘
```

#### 每个生命周期节点的控制目标

| 节点       | 主要风险                             | 执行前控制                                           | 必须留下的证据               |
| -------- | -------------------------------- | ----------------------------------------------- | --------------------- |
| 输入 / 上下文 | Prompt injection、恶意网页/Skill/工具结果 | 标记不可信来源；隔离数据与指令；风险评分                            | 来源、内容摘要、信任级别、判定结果     |
| 目标 / 计划  | 目标漂移、隐式扩权、无限循环                   | 将用户目标绑定到允许的资源和动作；限制步骤、重试与子 Agent                | 目标、scope、计划版本、预算与审批理由 |
| 记忆       | Memory poisoning、跨任务传播、秘密持久化     | 按用户/项目/任务隔离；记录 provenance；对写入做策略检查和 TTL         | 读写来源、命名空间、保留期、污染处置    |
| 工具 / 动作  | 越权读写、高危 Shell、SSRF、凭证滥用          | Capability Gateway 按身份、资源、参数和时效授权；在 sandbox 中执行 | 工具、参数、授权决策、实际 effect  |
| 输出 / 外发  | 敏感数据泄露、误发、不可逆外部副作用               | 数据流与目的地校验；脱敏；外发/发布前确认                           | 输出摘要、接收方、脱敏与确认记录      |

#### 横向控制面：运行时必须能否决模型

1. **Identity & Capability Policy**：使用最小权限、短时能力令牌和资源级 scope；“用户提出目标”不等于“Agent 获得所有实现手段”。
2. **Runtime Enforcement**：所有高影响动作进入统一网关，由确定性策略、风险判断和审批状态共同决策；最终结果只能是 **Allow / Block / Approval**，不得由模型文本直接触发副作用。
3. **Audit & Recovery**：贯穿完整 trace，支持停止任务、隔离 workspace、吊销凭证、撤销可逆动作与事后复盘；不可逆动作默认提高审批级别。

#### 上线方式：先 Observe，再 Enforce

先以 observe 模式记录会被阻断或要求审批的动作，校准误报、漏报和合法任务体验；再对高置信、可解释、影响高的策略启用 enforce。策略按风险分级上线，而非一次性把所有动作改为拦截。

*注：上述“贴近运行时与工具执行的检查、拦截、审批”借鉴 AgentAegis 所代表的设计思路。AgentAegis 仅为架构参考，不代表官方集成、正式依赖或任何产品合作关系。*

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

- [x] 完整记录用户意图、Agent 计划、工具调用、参数、结果和副作用；
- [x] workspace 与宿主机敏感文件隔离；
- [ ] Shell、文件、网络、MCP 工具的 allowlist 和参数约束；
- [ ] secret 不进入不必要的模型上下文；
- [ ] 高影响动作审批、timeout、kill switch 和恢复路径。

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

#### 11-1 候选模型

##### 模型一：SingGuard

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

##### 模型二：SingGuard-NSFA

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

#### 两个模型的同页对照

| 字段 | SingGuard | SingGuard-NSFA |
|---|---|---|
| **模型定位** | 策略可适配的多模态 LLM 安全护栏。 | 面向 Agent 操作风险的文本护栏框架。 |
| **功能覆盖** | 判断文本、图片、图文是否违反 policy；覆盖 Prompt Injection、越狱、敏感内容与自定义业务政策违规。 | 判断 Prompt Injection/Jailbreak、恶意代码与网络攻击、敏感信息窃取/外传、危险操作/工具滥用、资源滥用与危险动作生成。 |
| **适配输入输出** | 输入：文本、图片、图文、自然语言 `policy`。输出：`safe/unsafe`、匹配风险类别、策略相关分析。 | 输入：用户请求、外部内容、计划、工具名/完整参数、工具结果、最终输出的文本化表示；建议区分 `untrusted_input` 与 `untrusted_output`。输出：风险类别、生成式风险分析或实时分类风险概率。 |
| **基础模型和规模** | Qwen3-VL，2B/4B/8B，Apache 2.0。 | Qwen3.5-Base，0.8B/2B/4B/9B，Apache 2.0。 |
| **训练架构** | 统一文本/多模态安全数据 SFT；论文报告用 Fast-Slow Decoupled DAPO 优化动态推理。完整训练集来源、规模和配比未完全公开。 | 风险分析数据 SFT；冻结 SFT 后骨干，为不同风险域训练轻量 MLP 分类头；新增风险可仅训练新分类头。 |
| **风险体系和安全 Benchmark** | SingGuard-Bench 约 56,340 样本。8B 报告：Image Safety F1 0.903、Text Query F1 0.874、Multimodal Safety F1 0.877、Text Response F1 0.893；Dynamic Rule（2,000 样本）policy-following accuracy 0.7415。 | 7 个一级域、28 个二级风险、185 个三级变体；93,000+ purpose-built multilingual samples（133 种语言）、3,435 cross-source samples。四种规模 purpose-built 均 ≥94% F1；9B cross-source F1 91.29%；竞品对比高 6–12 个 F1 百分点。 |
| **部署 Benchmark / 注意事项** | 多模态模型需要图像处理和更高内存/显存；公开资料没有可直接承诺的本地端到端 SLA，应按设备 SKU 测模型加载、图片分辨率、上下文长度、并发和 P95 延迟。 | Real-Time Classification 在 A100 条件约 45–57 ms/sample；仅作参考，**不是本地设备 SLA**。本地需测模型加载内存、CPU/GPU、并发、上下文长度、P50/P95 延迟和超时行为。 |
| **类 Claw 集成位置** | 用户输入/网页/图片进入 Agent 前，以及 Agent 输出/待展示内容返回用户前。可检查工具调用的文字化描述。 | 用户输入检查、Agent 计划与工具参数检查、结果与外发内容检查；之后必须进入确定性 Policy Gateway 决定执行或拒绝。 |

**共同边界**：两个模型都不替代确定性授权和执行控制；均不负责真实文件权限、API 授权、网络隔离、sandbox、源码/依赖漏洞扫描或恢复系统。

#####  两个模型的互补关系

| 问题 | SingGuard | SingGuard-NSFA |
|---|---|---|
| 网页/图片/图文内容是否安全 | 主要能力 | 不适合的主要方向 |
| Prompt Injection | 可作为内容层判断 | 主要能力之一 |
| 工具调用风险 | 可检查文字化描述 | 主要能力 |
| 源码/依赖漏洞 | 不负责 | 不负责 |
| 授权和真实阻断 | 不负责 | 不负责 |
| 多模态输入 | 支持 | 主要文本 |
| 在线低延迟分类 | 需看具体实现和本地部署 | 分类头路径适合 |

#### 11-2 集成方式

NSFA 应作为 **Agent runtime 的风险判断器** 接入，而不是替代工具网关或直接执行动作。对类 Claw 个人助手，建议由 Orchestrator 在每个可观测事件处组装结构化上下文，调用 NSFA；再把风险结果与用户身份、Security Envelope 和确定性规则一起交给 Policy Gateway 决策。

```text
用户 / 外部内容
    ↓  [输入检查]
Agent Planner
    ↓  [计划检查]
工具调用提案（tool + 完整参数）
    ↓  [动作前检查]
Policy Gateway（身份 + scope + 风险 + 确定性规则）
    ├─ allow  → Sandbox / 工具执行
    ├─ approval → 用户确认后执行
    └─ block  → 返回可解释拒绝
                        ↓
             工具结果 / 外发内容 / memory 写入
                        ↓  [结果、外发与持久化检查]
                    Audit Trace / 恢复
```

#### 建议接入的五个阶段

| 阶段 | 应送入 NSFA 的事件 | 重点判断 | 结果如何使用 |
|---|---|---|---|
| 1. 输入与上下文进入 | 用户请求、网页/文档/Skill/MCP 工具结果、检索结果 | Prompt Injection、Jailbreak、恶意代码/网络攻击指令、秘密诱导 | 给内容标记风险与 provenance；高风险内容不能直接变成 Agent 指令，可降权、隔离或要求重述任务。 |
| 2. 计划生成后 | 用户目标、当前 Security Envelope、Agent plan、拟调用工具集合 | 意图偏离、隐式扩权、过大自治、跨任务目标传播 | 计划不通过时要求 Planner 缩小 scope、移除高风险步骤或转人工确认；不要直接让模型结论放行。 |
| 3. 每次工具调用前 | 工具名、完整参数、目标路径/域名/账户、当前身份、历史动作、预计副作用 | 工具滥用、敏感数据外传、危险操作、SSRF/网络攻击、资源滥用 | 将 NSFA 风险概率和类别送入 Policy Gateway；由 Gateway 结合 allowlist、权限、参数 schema 和审批状态产出 **allow / approval / block**。 |
| 4. 工具结果与状态写入后 | 工具 stdout/stderr、文件/网络实际 effect、memory/RAG 写入候选、重试链 | 工具结果注入、秘密回流 context、memory poisoning、异常重试/累积副作用 | 结果按可信度标记；阻止高风险内容写入长期 memory；触发终止任务、吊销能力或收紧后续 scope。 |
| 5. 最终输出与外发前 | 面向用户的回答、邮件/消息/API payload、附件与接收方 | 敏感信息泄露、危险动作说明被再次执行、错误目的地、不可逆外发 | 输出脱敏或改写；外部发送、部署、删除、改权限等高影响动作仍由 Policy Gateway/用户审批决定。 |

#### 调用契约：把文本判断绑定到真实动作

每次调用至少保留以下结构，而不是只传一段 prompt：

```text
<untrusted_input>用户请求、网页、文档或工具结果</untrusted_input>
<untrusted_output>Agent 计划、工具调用、工具结果或最终输出</untrusted_output>

task / intent / user_identity / session_id
tool_name / full_parameters / target_resource
security_envelope / prior_actions / observed_effect / expected_side_effect
```

NSFA 输出应包括风险类别、风险概率或生成式分析；这些只作为 **风险信号和审计证据**。最终授权必须由确定性 Policy Gateway 根据身份、资源、参数、时效、影响级别和审批状态做出。

#### 两种模式如何分工

- **Real-Time Classification**：放在第 1、3、4、5 阶段的在线路径，适合高频、时延敏感的输入、工具调用、结果与外发检查；应设置按动作等级的超时预算。
- **Generative Reasoning**：用于第 2 阶段的复杂计划复核，以及离线 trace 审计、事件调查、人工复核和策略调优；不应成为所有低风险工具调用的同步前置依赖。

#### 失效、降级与可恢复性

1. NSFA 超时、OOM、模型崩溃或本地模型服务不可用时，**不得默认扩大权限或放行高影响动作**；继续执行的条件应由已有确定性策略决定。
2. 对低影响、只读且已被 allowlist 覆盖的动作，可按产品策略降级为记录风险并继续；对读取 secret、外部上传、删除、部署、改权限等动作，保持 block 或 approval。
3. 所有请求、模型版本、输入摘要、风险结果、Gateway 决策、实际 effect 与降级原因写入 trace，支持复盘、策略校准和回滚。

**部署考虑**：0.8B/2B 可作为本地 pilot 的候选规模；需在实际设备 SKU 上测模型加载内存、CPU/GPU、并发、上下文长度、P50/P95 延迟和超时行为。A100 的 45–57 ms/sample 只能作为参考，**不是本地设备 SLA**。上述阶段划分是类 Claw 的架构建议，不代表 NSFA 已官方集成到任何特定 Agent runtime。
#### 11-3 试点计划

##### 阶段 0：建立基线（1 周）

- 记录现有 Agent 的用户请求、计划、工具调用、结果、memory 写入和外部副作用；
- 统计普通任务的 P50/P95 延迟、回答成功率、工具成功率、内存峰值和 CPU/GPU 占用；
- 标注高影响动作和敏感资源。

##### 阶段 1：离线审计与数据集（1–2 周）

- 用 NSFA Generative Reasoning 审计历史 trace；
- 构造合法任务、Prompt Injection、越权读写、secret 外传、MCP/Skill 污染、资源滥用样本；
- 对每条样本记录人工标签、风险类别、动作影响和预期处理。

##### 阶段 2：Shadow Mode（1–2 周）

- NSFA 在线分类接在工具调用前，但不改变执行结果；
- 有多模态入口时，SingGuard 接在外部内容输入和最终输出两侧；
- 记录模型判断、规则判断、实际副作用和人工标签的差异；
- 测模型大小与设备 SKU 的内存、速度和准确率曲线。

##### 阶段 3：有限 Enforce（1 周）

- 只阻断高置信度、高影响、不可逆的动作；
- 读取 secret、外部上传、删除、部署、改权限默认拒绝或人工审批；
- 低风险普通读操作保持原有路径；
- 明确模型超时、模型崩溃和 Policy Gateway 不可用时的 fallback。

##### 试点验收指标

| 维度 | 指标 |
|---|---|
| 安全 | Secret Disclosure Rate、Unauthorized Action Rate、Denied-call Bypass Rate |
| 判断质量 | Precision、Recall、F1、误报率、漏报率、按风险域拆分的混淆矩阵 |
| 体验 | Agent 成功率、合法任务拒绝率、用户确认率、回答正确性变化 |
| 性能 | 额外 P50/P95 延迟、模型加载时间、峰值内存/显存、CPU/GPU、功耗 |
| 稳定性 | timeout、fallback、重试风暴、模型崩溃、断网和低资源场景 |
| 恢复 | 吊销凭证、隔离运行时、回滚文件/状态、保留证据的成功率 |

---

### 12 思路 2：执行层平台控制待实施事项——按控制类型 × 生命周期阶段分类

模型护栏只产出风险信号；真正决定一个动作能不能发生，必须落在 Agent framework 执行层的确定性控制上。本节整合两份 SuperClaw 安全设计材料：

- 《SuperClaw 当前安全能力与缺口评估》（`D:\superclaw\applications.ai.superclaw-zihan\docs\design\2026-08-12_superclaw_security_current_state_zh.md`，调研结论、非实现计划）——记录**当前已验证能力**与**缺口**；
- 《SuperClaw Agent Security Architecture》（`D:\superclaw\applications.ai.superclaw-zihan\docs\design\2026-08-12_agent_security_architecture.md`，Proposed）——把缺口拆成 `SecurityAction`/`SecurityDecision` 模型、6 个生命周期阶段和 6 个 rollout phase。

AgentAegis、Acacian Aegis 不是 SuperClaw 现有组件，只作为生命周期规则和策略/审批模型的参考；接入必须通过 OpenCode plugin、OpenWork Server、capability projection 和 ServiceHub 合约实现。

#### 需要先纠正的三个假设

1. **Trajectory 不等于完整审计**：调试日志可用于排障回放，但不能证明覆盖工具、MCP、网络、审批、文件、memory 和最终 channel delivery，也不具备防篡改与留存保证。
2. **Workspace/Sandbox 只解决部分隔离**：WSL2/Docker 降低对 Windows host 的直接暴露，但容器内 bash、MCP、网络出口和 cloud 模型路由仍可能携带数据或产生副作用。
3. **Shell guard 不等于命令安全闭环**：现有插件主要覆盖 agent 路由、重复调用和输出体量，尚无统一的危险命令、编码混淆、下载执行、删除/覆盖、命令→网络外发链 policy engine。

#### 分类维度说明

- **类型**（控制在做什么）：
  - `Observe`：只记录风险信号，不阻断，用于校准误报/漏报；
  - `Guardrail`：确定性限制、脱敏、来源标记或需要审批的门控，允许继续但收窄能力；
  - `Refuse`：默认拒绝/阻断，含 fail-closed；
  - `Execution`：策略引擎、schema、审计存储等让上述判断生效的执行层基础设施本身。
- **生命周期阶段**：沿用 `SecurityAction.phase` 六段——`inbound`（外部输入进入）、`prompt`（系统上下文组装）、`pre_tool`（工具调用前）、`post_tool`（工具结果回流）、`memory_write`（持久化写入）、`outbound`（最终回复/artifact/channel 交付）；跨阶段基础设施标记为 `cross_phase`。
- **优先级**：P0 = 无这些底座不建议开放自动执行；P1 = 让平台能发现/收窄风险；P2 = 规模化治理与可验证性。

#### 任务分类表

| 任务 | 细分拆解 | 优先级 | 类型 | 生命周期阶段 |
|---|---|---|---|---|
| **1. 策略决策引擎与执行骨架**：让 `allow/observe/redact/require_approval/block` 有一个统一、可测试的落地点 | · 先实现最小、进程内/本地的 `SecurityAction`/`SecurityDecision` schema 与决策引擎，不在第一版建设独立复杂策略平台<br>· Policy 优先级与作用域规则（capability grade / project / agent / tool / path / network target）<br>· 契约矩阵：AgentAegis 能力 → OpenCode hook / OpenWork 边界 / host 控制映射，锁定兼容版本（Rollout Phase 0）<br>· Policy/审计骨架单测：precedence、非法策略、redaction、correlation（Rollout Phase 1）<br>· Fail-closed 默认策略：policy 缺失/失效/adapter crash 时高危默认拒绝，低风险 observe + 报错遥测<br>· ServiceHub/Security Manager 控制面集成：policy 分发、审计查询/导出、健康检查，不代理 chat data plane<br>· 落地前必须回答的未决问题：post-tool/system-transform hook 兼容性、权威 memory 写入路径、审计存储归属、tool 层以下出口控制、留存/隐私策略 | P0 | Execution | cross_phase |
| **2. 危险动作执行护栏**：工具调用前把 Shell、文件、网络、资源滥用收敛到统一 critical 规则 | · Plugin/policy bundle 自我防护：capability projection + hash gate，禁止 Agent 写入/删除策略资源与 manifest<br>· 危险命令与文件操作拦截：Shell 分级、破坏性写入、编码混淆、下载后执行、受保护路径<br>· 网络出口 / SSRF / metadata 地址拦截：先在工具层覆盖 web/browser/MCP/bash 等入口，且必须由 sandbox/Docker/DNS egress 控制兜底；工具规则不能替代网络边界<br>· 重复调用 / 资源滥用限制（复用现有 repeat/budget guard，不重写计数器）<br>· 集成测试证明被阻断动作确实无副作用（Rollout Phase 2） | P0 | Refuse（含 Guardrail） | pre_tool |
| **3. 审批、工具前后审计与最终输出边界**：确保审批不可被绕过，工具动作可追溯，交付前统一脱敏 | · `require_approval` 接入现有 OpenWork 审批，关联 `trajectoryId`/`toolCallId`/`approvalId` 与安全摘要；P0 要求重启后默认拒绝/标记 orphaned request，不得恢复为 allow；跨重启持久化 pending approval 仅在产品需要长时间审批时列 P1<br>· 每个高风险调用在副作用前写 `tool.call.pre`，在成功/失败/拒绝/取消/超时后写 `tool.call.post`；审批写 `approval.requested/decided`。事件默认不留原始 prompt、工具参数、工具输出、文件内容或密钥<br>· 最终响应/artifact sanitizer 接入 channel/desktop 双路径，统一 redact/block；凭证/敏感信息命中 outbound 默认 enforce | P0 | Guardrail（含 Refuse）+ Execution | pre_tool + post_tool + outbound |
| **4. 审计事件、留存与恢复边界**：先建立可关联事件骨架，再逐步提高完整性 | · P0 审计骨架：event ID、correlation/session/project、policy digest、rule IDs、decision、审批引用、redaction 计数，默认不留原文/密钥；本地 JSONL 可作为第一版审计存储，但不得称为不可篡改<br>· P2 留存治理：受限 ACL、append-only 或签名/远端不可变存储、保留期与访问审计<br>· 恢复按操作类型设计：P0 是在副作用前阻止高危操作并记录证据；文件 checkpoint/rollback、凭证吊销、运行时隔离等只对可逆/可隔离操作分别设计，不能承诺所有 Shell、网络、删除动作可回滚 | P0（事件骨架）/ P2（留存治理与可逆恢复） | Execution | cross_phase |
| **5. 不可信输入与工具结果治理**：把外部内容、用户输入和工具结果当作独立不可信边界 | · 用户输入风险检测：jailbreak/角色伪装/工具诱导/secret 与外发请求；默认只产出 risk signal，不单独依赖 prompt 文本强制拦截或授权工具<br>· Prompt 安全上下文附加：来源标记的简洁约束，不暴露规则细节<br>· 意图分类结果仅作风险信号：不能单独授权，成功/失败/fallback 都需可审计<br>· 工具结果注入检测：在已验证 post-tool hook 中记录 `tool.result.scanned`，将 web/MCP/文档结果标为不可信，在进入模型 context 前处理；后置检测不能撤回已经发生的副作用<br>· 先 Observe 校准；仅对“高置信注入 + P0 高危工具动作”组合选择性收紧为审批或阻断 | P1（先 Observe） | Observe → Guardrail | inbound + prompt + post_tool |
| **6. Memory 写入治理**：防止低可信内容变成未来会话的高可信指令 | · 先枚举实际 memory write 路径，而非仅保护 `MEMORY.md`<br>· Memory 写入污染防护：记录来源（用户/工具/模型/系统）、目标 namespace/path、是否来自外部内容，并检查敏感内容/指令污染；先 observe 校准再 enforce<br>· 缩短/移除 PII placeholder→原值映射的 TTL 内保留窗口 | P1（先 Observe） | Observe → Guardrail | memory_write |
| **7. 观测遥测上线与选择性 Enforce**：先用真实数据校准，再逐条规则收紧 | · 不可信上下文与 memory 观测遥测上线，用于校准误报率（Rollout Phase 4）<br>· 选择性 enforce：仅将测量后高置信度规则从 observe 升级为 enforce，每条规则可单独回滚；P0 关键规则（受保护资源删除、metadata/私网、明显凭证外发）不等待该阶段 | P1 → P2 | Observe → Guardrail/Refuse | cross_phase |
| **8. 安全测试与验证矩阵**：证明阻断真的生效、副作用真的没发生 | · P0 随 critical rule 同步交付：被阻断的 shell/file/network/MCP 操作没有副作用；审批 deny/timeout 不执行；`tool.call.pre/post` 与审批事件正确关联<br>· P1 对抗性夹具：命令混淆、编码 payload、prompt injection 工具结果、metadata URL、密钥文本、memory 投毒、重复调用<br>· P2：全 capability-grade projection、性能/资源、审计留存与完整性验证 | P0（关键阻断测试）→ P1（对抗性）→ P2（规模化） | Execution（测试） | cross_phase |

#### 与思路 1 的分工

模型（SingGuard / SingGuard-NSFA）产出风险信号；本节的 `SecurityAction`、Policy Gateway、审批与审计闭环才是唯一能把风险信号转成 `allow / observe / redact / require_approval / block` 并留下证据的地方。两者的接口就是第 11 章 NSFA/SingGuard 的输出，被本节的统一决策点消费，而不是让模型直接触发副作用。

### 13 综合实现path
#### 思路 1 与思路 2 的关系

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

#### 建议的综合实施顺序：先 P0 平台边界，再引入 NSFA 观察

NSFA 不应作为 P0 的前置依赖，也不应在现有 trace 尚不能可靠回答“哪个工具实际执行、是否审批、实际 effect 是什么”时直接接入同步阻断主路径。正确顺序是：先让危险动作在模型漏判时也无法越权，再把 NSFA 的判断接入这个已存在的策略与审计闭环。

| 阶段 | 平台/策略交付物 | NSFA / SingGuard 的位置 | 是否阻断 | 进入下一阶段的门槛 |
|---|---|---|---|---|
| **A. P0-A：动作与证据骨架** | 最小 `SecurityAction`/`SecurityDecision`、规则优先级、`trajectoryId/sessionId/toolCallId`、工具 `pre/post` 事件、审批关联；先覆盖 bash/file/network/MCP。 | **不接在线 NSFA。** 仅采集脱敏 action 摘要、决策和 effect，形成真实 trace 基线与攻击样本。 | 确定性 critical 规则可 block/approval。 | 能证明被拒绝动作无副作用；审计不记录原始 prompt/tool output/secret。 |
| **B. P0-B：高影响边界** | 危险 Shell、受保护路径、SSRF/metadata、下载执行、外发和最终输出 DLP；sandbox/网络 egress 兜底；OpenWork 审批接入。 | **离线 NSFA Generative Reasoning。** 对 A/B 的脱敏 trace 做离线审计、样本标注、规则缺口发现，不影响用户执行。 | P0 仍只依赖确定性 policy + 审批。 | 已知高影响动作都有 pre/post/approval/outbound 证据；可计算漏拦截、误拦截和性能基线。 |
| **C. P1-A：NSFA shadow observation** | 固定 policy/审批的真实决策仍为唯一授权源；增加 risk-signal adapter 和审计字段。 | **首次引入在线 NSFA Real-Time Classification，默认 shadow/observe。** 先接在用户输入、外部 tool result、以及高危工具调用提案前；输出 label/confidence/model version，不改 allow/block。纯文本工具型 Agent 优先 NSFA 0.8B/2B；多模态入口另评估 SingGuard。 | 否；模型超时/OOM 只记录 fallback，绝不扩大权限。 | 在目标 SKU 上获得 P50/P95、内存/功耗、Precision/Recall、漏报、误报、合法任务拒绝率；完成与人工标签及 P0 policy 决策的对比。 |
| **D. P1-B：风险信号辅助收窄** | 将稳定的风险信号传给 Policy Gateway；外部结果加 provenance/untrusted 标记；memory write observe。 | 对“高置信 NSFA 信号 + 已有 P0 高危动作”组合提高风险等级、缩小 scope 或要求审批；不让 NSFA 单独 `allow`。Generative Reasoning 继续用于计划复核、事件调查和规则调优，不进入所有低风险同步调用。 | 仅组合规则可 require_approval/选择性 block。 | 每条规则可单独回滚；测得的误报可接受；各 capability grade 的 projection 和 fallback 已验证。 |
| **E. P2：规模化治理** | 审计留存/完整性、查询 UI、策略灰度、OEM/SKU 配置、恢复能力按可逆操作类型推进。 | 根据 benchmark 决定是否扩大 NSFA 模型、加入 SingGuard 多模态输入/输出检查或启用更多在线阶段。 | 按规则/grade 配置。 | 留存、隐私、性能和产品支持成本可接受。 |

**NSFA 的最早合适引入点是阶段 C，而不是阶段 A。** 阶段 B 的离线 Generative Reasoning 可以更早启动，因为它不在用户同步主路径；但它的输入必须是脱敏 audit/trajectory 摘要，不能把完整 prompt、文件内容、工具输出或 secret 批量喂给审计模型。

#### NSFA 接入契约与降级规则

```text
SecurityAction（工具、输入或结果的脱敏结构化摘要）
  → NSFA risk signal { categories, confidence, model_version, latency, fallback_reason? }
  → Policy Gateway（身份、scope、确定性规则、审批状态、risk signal）
  → allow | observe | redact | require_approval | block
```

- NSFA/SingGuard **只能产生 risk signal**；唯一授权源仍是 Policy Gateway + OpenWork approval + sandbox/network 边界。
- 普通低风险对话不应每轮同步调用 NSFA；优先轻量规则、异步审计或按风险采样。
- 模型超时、OOM、不可用时：低风险只读 allowlist 动作可按策略 `observe` 后继续；删除、外发、改权限、读取 secret、私网/metadata 访问仍由 P0 确定性规则保持 `block` 或 `require_approval`。
- NSFA 输入和审计输出默认不包含原文或 secret：传入 action 类型、目标类别、脱敏参数摘要、前序 action 摘要和 policy context；必要时只传 hash/计数/类别。
- 只有在 shadow 数据证明某类信号稳定，且与 P0 高危动作组合时，才从 `observe` 升级到审批或阻断；每条升级规则必须可单独回退。

#### P0/P1 的责任边界

| 层 | P0 负责什么 | P1/NSFA 负责什么 | 不应承诺什么 |
|---|---|---|---|
| 工具执行 | 在副作用前用确定性规则、审批、sandbox/egress 限制危险动作。 | 识别复杂意图、外部内容和异常 action 链，为 P0 策略提供附加风险信号。 | 不因模型“低风险”而跳过 P0 deny/approval。 |
| 审计 | 记录 pre/post、决策、审批、effect 摘要与 correlation。 | 用离线 trace 分析发现攻击模式、校准规则和阈值。 | 不保存 chain of thought、原始敏感内容，也不把本地 JSONL 称为不可篡改。 |
| 输入/结果 | 保证高风险动作即使被 prompt injection 诱导也无法越权。 | Observe 用户输入、网页/MCP/文档结果的 injection/jailbreak/外发风险，并标记不可信来源。 | 不能保证检测所有变体；post-tool 检查不能撤销已经发生的副作用。 |
| 恢复 | 对高危动作先阻断，减少必须恢复的事故。 | 用历史 trace 帮助调查和策略改进。 | 不承诺所有 Shell、网络或删除动作可回滚；checkpoint/rollback 只对可逆操作单独设计。 |

---

#### 性能影响与产品取舍

##### 可能增加的成本

1. **延迟**：同步调用模型会增加 P50/P95；多次检查比一次检查更明显；
2. **本地资源**：模型权重、KV cache、图片编码和并发会占用内存/显存/CPU；
3. **回答体验**：误报会导致合法任务被拒绝、额外确认或 fallback；
4. **产品复杂度**：需要配置策略、阈值、超时、审批、日志和回滚；
5. **可用性风险**：护栏模型加载失败或资源不足时，必须决定 fail-open 还是 fail-safe。

##### 可能带来的正向收益

- 降低 secret 泄露、越权修改和错误外发的概率；
- 减少高影响事故的潜在损失，而不是提升普通回答速度；
- 通过完整 trace 缩短问题定位和售后时间；
- 让 OEM 能够按设备 SKU 交付不同安全等级；
- 将安全策略变成可配置、可测试、可审计的产品能力。

#### 设计建议：不要把所有检查都放进同步主路径

| 场景 | 推荐方式 |
|---|---|
| 普通低风险对话 | 轻量规则或异步审计，不阻塞主回答 |
| 外部网页、图片、文档 | P1 shadow 期先做轻量 provenance/规则与异步/采样检查；多模态入口的 SingGuard 仅在 benchmark 后进入在线路径 |
| 工具调用前 | P0 确定性 Policy Gateway 必经；P1 后 NSFA 分类仅作为 risk signal 辅助收窄，不单独放行 |
| 删除、外发、部署、改权限 | 强制审批或默认拒绝 |
| 历史日志和复杂攻击链 | NSFA Generative Reasoning 离线审计 |
| 模型超时/不可用 | 按动作风险选择 fallback；高影响动作 fail-safe |

#### 必须自己测量的本地 benchmark

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

### 14 最终建议：面向管理层的决策页

#### 是否要做？

**要做，且应批准为平台能力，而不是一次性模型评测或单点内容审核。** 只要本地类 Claw 能访问用户文件、记忆、网络、Shell、MCP 或外部 API，Agent Security 就是可控交付的基础能力。

本次建议的决策不是“是否立刻把一个安全模型放到每个请求前”，而是：

1. 批准 P0 平台边界，使高影响动作在模型漏判、超时或不可用时仍不能越权；
2. 批准在 P0 证据闭环形成后引入 NSFA 的离线审计与 P1 shadow observation；
3. 在真实设备、真实 trace 和明确回退门槛上决定哪些模型规则值得进入强制路径。

#### 建议批准的实施路径

| 阶段 | 管理层批准的交付物 | NSFA / SingGuard 决策 | 放行条件 |
|---|---|---|---|
| **P0-A：动作与证据骨架** | 统一 `SecurityAction`/Policy、工具前后 audit、审批关联、关键动作 correlation；不记录原始敏感内容。 | 不接在线模型；建立脱敏 trace 基线与攻击样本。 | 被拒绝的 Shell/file/network/MCP 动作可证明没有副作用。 |
| **P0-B：高影响边界** | 危险 Shell/文件、SSRF/metadata、网络 egress、审批、最终输出 DLP；P0 critical rule 用确定性 policy enforce。 | 对脱敏 trace 做离线 NSFA Generative Reasoning，用于发现规则缺口和标注数据，不影响用户执行。 | 高影响动作均有 pre/post/approval/outbound 证据；模型缺席不扩大权限。 |
| **P1-A：Shadow observation** | 接入 risk-signal adapter、输入/工具结果 provenance、memory observe。 | 首次在线引入 NSFA Real-Time Classification，默认 shadow/observe；纯文本工具 Agent 从 0.8B/2B benchmark 起步。 | 目标 SKU 上完成 P50/P95、内存/功耗、Precision/Recall、漏报、误报与合法任务影响测量。 |
| **P1-B：选择性收窄** | 将经验证的风险信号送入 Policy Gateway，并保持每条规则可回退。 | 仅“高置信模型信号 + P0 高危动作”可升级为 require approval 或选择性 block；模型永不单独 allow。 | 误报、体验和 fallback 可接受；capability grade 和回退已验证。 |
| **P2：规模化治理** | 审计完整性/留存、查询 UI、OEM/SKU 策略、可逆操作的恢复能力。 | 根据 benchmark 决定扩大 NSFA、加入 SingGuard 多模态检查或更多在线阶段。 | 隐私、性能、支持成本和产品收益可接受。 |

#### 明确的安全与产品原则

1. **平台先于模型。** Policy Gateway、OpenWork approval、sandbox/network egress 与确定性工具规则是唯一授权边界；模型只产生 risk signal。
2. **P0 先防副作用，P1 再提升识别。** P0 对删除、外发、改权限、读取 secret、私网/metadata 等高影响动作保持 block/approval，即使 NSFA 超时、OOM 或未部署。
3. **不把 debug trace 当审计。** P0 建立最小、脱敏、可关联的 pre/post/approval/outbound 审计骨架；不可篡改留存、长期查询 UI 和可逆恢复是 P2 工作，不应在第一版过度承诺。
4. **不让模型成为唯一拦截器。** Prompt Injection、Jailbreak 和 tool-result injection 在 P1 先 observe；只有与 P0 高危动作组合并经测量后，才选择性升级为审批或阻断。
5. **性能按风险分层。** 普通低风险对话不强制每轮同步模型检查；离线 NSFA 审计可先于在线 NSFA 启动；SingGuard 仅在有多模态攻击面且 benchmark 证明可接受时引入。

#### 两个模型是否都要做？

- **纯文本、工具型个人 Agent**：P0 不依赖模型；P1 优先评估 NSFA 0.8B/2B shadow observation，SingGuard 延后。
- **网页/截图/图片/图文产品**：仍先完成 P0；在 P1 用 benchmark 决定 NSFA + SingGuard 是否形成内容层与动作层互补。
- **资源受限设备**：保留 P0 确定性规则和审批；若 NSFA 不满足性能/功耗门槛，继续离线审计或不上在线模型，不降低 P0 保护。
- **OEM 多 SKU**：基于 benchmark 为每个 SKU 决定模型大小、是否启用在线 observation、以及可用安全等级；不强制所有设备加载最大模型。

#### 建议批准的 pilot 成功标准

试点不以“模型分数最高”为唯一成功标准。P0 完成后应先能回答：

- 被 block/deny 的高危操作是否确实没有副作用；
- 是否能关联一次请求的工具前后、审批、最终输出与安全决策，同时不保存原始 secret/PII；
- 哪些确定性规则已经覆盖最常见的高影响风险，哪些仍需模型辅助判断；
- P0 模型缺席、超时或故障时，是否始终不会扩大高危权限。

满足这些门槛后，再在 P1 shadow 中回答：

- 哪些真实输入/工具结果风险是 NSFA 能够比规则更早、更准确发现的；
- NSFA/SingGuard 对这些风险的 Precision、Recall、漏报、误报和合法任务影响是什么；
- 增加 observation 后的 P50/P95、内存/显存、CPU/GPU、功耗和 fallback 是否符合目标 SKU；
- 哪些“高置信风险信号 + P0 高危动作”组合值得升级为审批或阻断；
- 是否值得将模型 observation 产品化为 OEM/SKU 安全等级。

> **最终判断标准不是“模型 benchmark 最高”，而是：在可接受的本地性能成本下，平台能否先阻断高影响未授权动作、留下可解释证据；模型观察是否在此基础上显著降低漏报，而不造成不可接受的误报和体验损失。**

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
