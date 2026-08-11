
# Task Decomposition：如何判断 Agent 的 Unsafe Action

> 研究问题：当我担心 LLM Agent 越权时，如何拆解“什么是不安全行为”、如何判断、如何阻止，以及有哪些开源实现可以落地？

## 0. 核心结论

“Agent 是否产生不安全结果”不是一个可以交给单一分类器的问题。

更准确的统一抽象是：

> **在不确定性下，维持 Agent 的委托权限不被扩大、转移或持久化。**

工程上必须拆成四类能力：

1. **Prevention / Enforcement**：动作执行前用确定性策略阻止越权。
2. **Authorization**：判断动作是否被具体用户、任务、资源和时间窗口授权。
3. **Detection / Evaluation**：分析 Agent 是否偏离意图、泄露数据、产生危险轨迹。
4. **Recovery / Response**：发现问题后吊销凭证、隔离运行时、回滚状态、保留证据。

推荐总体架构：

```text
User goal
   ↓
Intent / scope binding
   ↓
Agent planner（不可信规划器）
   ↓
Capability gateway（PEP）
   ↓
Deterministic policy engine（PDP）
   ↓
Tool / API / Filesystem / Network adapter
   ↓
Sandbox / Runtime isolation
   ↓
External effect

All stages → Audit / Trace / Detection / Recovery
```

最重要的原则：

> **LLM 可以参与理解意图和风险评估，但不能单独承担权限控制、凭证保护、沙箱隔离和网络边界。**

---

## 1. 先定义三个容易混淆的问题

### 1.1 Agent 想做什么？——Intent

这是语义问题：Agent 的计划和工具参数是否符合用户任务？例如用户说“总结当前项目”，Agent 却修改代码、安装依赖、访问生产数据库或发送邮件，属于意图偏离。

### 1.2 Agent 被允许做什么？——Authorization

这是安全策略问题。用户目标不是权限授予：

> “总结项目”不自动授权读取 `.env`、访问其他项目、访问外网、修改文件或发送外部请求。

### 1.3 实际发生了什么？——Observed Effect

这是执行和审计问题。必须观察工具调用、完整参数、文件/数据库访问、网络出站、凭证使用、memory 写入、重试链和最终副作用，而不能只看 Agent 的自然语言解释。

---

## 2. Unsafe Action 的分解维度

### 2.1 授权越界

Agent 执行了用户、租户、任务或工具没有授权的动作：允许读却删除、访问项目 B、普通用户调用管理员 API、查询变成写入。

建议指标：Unauthorized Action Rate、Privilege Escalation Distance、Denied-call Bypass Rate、Cross-tenant Access Rate。

### 2.2 意图偏离

动作不再服务于用户任务，或为完成任务擅自增加目标。适合用 LLM judge 或人工复核，但不能直接决定最终权限。

### 2.3 能力过大（Excessive Agency）

1. **Excessive Functionality**：任意 shell、任意 URL、任意 SQL。
2. **Excessive Permissions**：读工具拥有写权限，通用高权限身份代替用户身份。
3. **Excessive Autonomy**：高影响动作无需审批、确认或回滚。

### 2.4 数据机密性违规

Agent 读取、推断、输出或外传未授权数据，包括 `.env`、SSH key、云凭证、跨租户数据、其他任务 memory 和工具返回的秘密。

```text
secret file → model context → tool argument → external API
```

没有显式 declassification，应拒绝这条数据流。

### 2.5 动作完整性违规

动作类型可能被允许，但目标、参数或内容错误：修改错文件、给错账户付款、部署未经审批代码、将不可信文档命令作为执行参数。

```text
动作类型 + 目标资源 + 完整参数 + 当前状态 + 用户授权
```

### 2.6 外部副作用

为每个动作标注 reversible/irreversible、low/high impact、internal/external、read/write/delete/send/execute 以及单步/累积影响。删除、支付、发邮件、部署、改权限和公开发布通常需要审批。

### 2.7 运行时突破

访问 workspace 外文件、宿主机凭证、云 metadata，利用包代理或浏览器间接出网，或逃逸 sandbox。这类问题必须由 OS、网络和 runtime 强制执行。

### 2.8 持久化与跨步骤传播

检查恶意内容是否进入长期 memory、RAG、下一会话、其他 Agent、工具配置、日志或缓存。

### 2.9 可用性与资源滥用

包括无限循环、retry storm、工具调用风暴、token/费用耗尽、进程/内存/磁盘耗尽和大规模云资源操作。设置时间、步数、并发、成本和副作用预算。

---

## 3. 为什么不是 `prompt → safety classifier → allow / deny`

### 3.1 不只是 classifier 难定义

更根本的问题有五个：

1. **观察对象错误**：危险事件通常是后续文件读取、工具调用、网络出站、数据库写入或状态变化，不是初始 prompt。
2. **授权信息缺失**：prompt 通常没有用户、租户、任务 scope、资源权限、数据标签、审批和时间窗口。
3. **决策粒度过粗**：一次 `ALLOW` 无法表达允许读项目 A、禁止读 `.env`，允许写 `src/`、禁止删除。
4. **多步组合不可见**：`read secret → encode → POST external` 中每一步可能都看似正常。
5. **概率模型承担硬安全责任**：分类器会漏报、误报、受对抗样本影响，不能提供权限和沙箱需要的确定性保证。

所以问题不是简单的“classifier 不够准确”，而是：

> **文本内容安全与执行安全是不同任务。**

### 3.2 五类信息不能混成一个 prompt

```text
Intent       用户想完成什么
Policy       在什么条件下允许什么动作
Capability   Agent 实际拥有的能力
Observation  外部环境返回了什么
Effect       系统状态实际发生了什么变化
```

### 3.3 classifier 的正确位置

| 判断 | classifier 可做什么 | 最终安全边界 |
|---|---|---|
| Prompt injection | 风险信号、明显模式过滤 | 否 |
| 用户意图 | 结构化意图候选 | 否，需策略/审批 |
| 工具参数语义 | 发现疑似目标偏离 | 否，需 schema/policy |
| 内容风险 | 内容审核、PII 检测 | 仅防御纵深 |
| 文件/API 权限 | 辅助风险解释 | OS/IAM/PDP |
| 数据外传 | 发现疑似语义外传 | 标签/网关/DLP |
| Sandbox escape | 不适合判断 | runtime enforcement |

推荐：

```text
classifier / semantic analysis = advisory signal
policy + capability + runtime = enforcement boundary
```

---

## 4. Security Envelope：组件、接口与连接方式

### 4.1 端到端连接

```text
User request
  ↓
Intent Interpreter → IntentCandidate
  ↓
Task / Scope Binder ← user/session/tenant policy
  ↓ TaskContext
Agent Planner → ProposedAction（不可信）
  ↓
Capability Gateway（PEP）→ Policy Engine（PDP）
  ↓ allow + obligations / deny / approval
Tool/API Adapter → Sandbox/Runtime → External effect

每一层 → Audit / Trace / Detection / Recovery
```

### 4.2 Intent Interpreter 与 Scope Binder

Intent Interpreter 只解析目标，不授予权限。

输入：

```json
{
  "user_message": "总结当前项目，不要修改文件",
  "conversation_id": "c-1",
  "user_id": "u-1",
  "tenant_id": "t-1"
}
```

输出：

```json
{
  "intent_id": "i-1",
  "goal": "summarize_project",
  "requested_effects": ["read"],
  "forbidden_effects": ["write", "delete", "send"],
  "confidence": 0.91,
  "ambiguities": []
}
```

Scope Binder 将身份、任务、资源、数据和时间窗口绑定，并要求后续只能单调收窄：

```json
{
  "task_id": "task-1",
  "principal": {"user_id": "u-1", "tenant_id": "t-1"},
  "goal": "summarize_project",
  "resource_scope": ["workspace://project-a/src/**"],
  "allowed_verbs": ["read"],
  "denied_patterns": ["**/.env*", "**/.ssh/**", "**/*.pem"],
  "network_scope": "none",
  "expires_at": "2026-08-07T12:00:00Z",
  "delegation": false
}
```

后续扩大资源、verb 或网络范围必须重新授权。

### 4.3 Agent Planner：只提出动作

Planner 输出结构化动作，不能直接持有工具句柄：

```json
{
  "action_id": "a-17",
  "task_id": "task-1",
  "tool": "file_read",
  "verb": "read",
  "arguments": {"path": "src/app.py"},
  "purpose": "读取入口文件生成摘要",
  "depends_on": ["a-16"],
  "requested_data": ["internal_source"],
  "estimated_effect": "none"
}
```

### 4.4 Capability Gateway / PEP

所有工具、文件、网络、数据库、memory 写入和 Agent 委派都经过网关。

输入：`ProposedAction + TaskContext + CurrentState + Provenance`。

输出：

```json
{
  "decision_id": "d-17",
  "action_id": "a-17",
  "decision": "ALLOW",
  "obligations": [
    {"type": "redact_output", "labels": ["secret"]},
    {"type": "log_full_arguments"}
  ],
  "capability_token": "short-lived-signed-token",
  "expires_at": "2026-08-07T11:55:00Z"
}
```

拒绝响应必须结构化：

```json
{
  "decision": "DENY",
  "reason_code": "RESOURCE_OUT_OF_SCOPE",
  "violations": ["path is outside task.resource_scope"],
  "retryable": false
}
```

### 4.5 Policy Engine / PDP

PDP 只做策略决定，不执行副作用。输入至少包括：

```json
{
  "principal": {"user_id": "u-1", "agent_id": "agent-1", "tenant_id": "t-1"},
  "task": {"task_id": "task-1", "goal": "summarize_project"},
  "action": {"tool": "file_read", "verb": "read", "arguments": {"path": "src/app.py"}},
  "resource": {"labels": ["internal_source"], "owner": "t-1"},
  "state": {"approval": null, "step": 17, "budget_remaining": 83},
  "provenance": {"argument_source": "planner", "data_source": "workspace"}
}
```

输出包含 `decision`、`reason`、`obligations` 和 `policy_version`。OPA/Cedar/SMT 适合可形式化约束；复杂语义判断只能作为辅助信号，不能覆盖硬性 deny。

### 4.6 Data-flow Monitor

每个对象携带来源、敏感级别、租户、任务和允许去向：

```json
{
  "object_id": "o-12",
  "labels": ["secret", "tenant:t-1", "source:env"],
  "provenance": ["file:/app/.env"],
  "allowed_sinks": ["internal_audit"],
  "retention": "task_end"
}
```

`secret → model context → external HTTP POST` 应触发 `DENY` 或 `REQUIRE_DECLASSIFICATION`，而不是等输出 classifier 发现泄露。

### 4.7 Approval Gateway

审批必须绑定精确动作，而不是泛化到整个 Agent：

```json
{
  "approval_id": "ap-2",
  "action_hash": "sha256:...",
  "tool": "send_email",
  "arguments_hash": "sha256:...",
  "target": "external-recipient",
  "impact": "external_data_transfer",
  "expires_at": "2026-08-07T11:30:00Z",
  "required_approvers": 1
}
```

参数、目标或数据范围变化就必须重新审批，防止 TOCTOU 和 approval laundering。

### 4.8 Sandbox / Runtime Adapter

输入是 `SandboxSpec`：镜像 digest、只读文件系统、最小挂载、网络模式、egress allowlist、非 root、最大进程/CPU/内存和 brokered credentials。

输出应包括 sandbox id、执行结果、文件差异、网络流摘要、资源消耗、终止原因和 checkpoint。Sandbox 限制爆炸半径，但不负责业务授权。

### 4.9 Audit / Detection / Recovery

每个动作生成不可变事件：task/action id、policy decision、工具和参数 hash、数据标签、sandbox id、网络摘要、状态差异和时间戳。

检测器看完整轨迹：

```text
用户目标 → context 来源 → 计划 → 工具调用 → 数据读取
→ 参数变化 → 网络出站 → memory 写入 → 最终副作用
```

恢复必须预先设计 transaction journal、checkpoint、dry-run、token 吊销/轮换、sandbox 隔离、memory quarantine、回滚和轨迹重放。

### 4.10 按动作生命周期理解 Security Envelope

Security Envelope 是完整的系统架构；对单个 Agent action，可以把它投影成三个连续阶段：

```text
Pre-action → Runtime → Post-action
```

| 生命周期阶段 | 对应组件 | 核心判断 |
|---|---|---|
| **Pre-action** | Intent Interpreter、Scope Binder、Planner、Capability Gateway、Policy Engine、Approval Gateway | 这个候选动作是否被允许？ |
| **Runtime** | Tool/API Adapter、Data-flow Monitor、Sandbox/Runtime、网络与资源限制、Audit | 实际执行是否越界？ |
| **Post-action** | Audit、Trace Detection、State-difference Oracle、Recovery | 系统是否产生了未经授权的变化？如何恢复？ |

这不是另一套架构，而是 Security Envelope 的动作生命周期视图：

```text
Security Envelope = 组件、信任边界和连接关系
Action Pipeline   = 一个动作从提出到完成的检查时机
```

这些阶段不是严格的一次性线性流程。Audit、Data-flow Monitor 和 Detection 从动作提出开始持续工作；Policy 可能在每一步重新检查；Recovery 可以在执行中途暂停并隔离环境。

一个完整动作的安全路径是：

```text
Pre-action:
  intent/scope → proposed action → authorization → approval
        ↓
Runtime:
  tool adapter → data-flow check → sandbox/network/resource enforcement
        ↓
Post-action:
  observed events → state diff → harm classification → recovery
```

例如用户只允许读取 `src/`，Agent 提议读取 `.env`：

1. **Pre-action**：Scope Binder 和 Policy Engine 返回 `DENY`，动作不执行；
2. **Runtime fallback**：若策略层失效，文件系统/sandbox 仍应阻止访问；
3. **Post-action**：若访问已经发生，Audit 和状态/数据流检查确认 secret 是否进入 context、工具参数或外部网络，并触发凭证吊销、隔离和恢复。

因此，安全判断不是只有“动作前批准”，也不是只有“事后看结果”，而是：

> **执行前限制可做什么，执行中限制实际能做什么，执行后确认发生了什么。**

---

## 5. 开源实现方案

### 5.1 策略与授权

- [Open Policy Agent](https://www.openpolicyagent.org/)：通用 PDP，Rego，适合工具网关、API、Kubernetes、CI/CD；需要自己实现 PEP。
- [Cedar](https://www.cedarpolicy.com/)：主体、资源、动作、上下文的细粒度授权；需要自己接工具代理和审计。
- [OpenLeash](https://github.com/openleash/openleash)：Agent authorization layer，支持 `ALLOW`、`DENY`、`REQUIRE_APPROVAL`、短期 proof token；项目较新。
- [AgentWard](https://github.com/agentward-ai/agentward)：MCP/HTTP tool-call proxy，支持扫描、运行时拦截、参数限制和审计；使用前应审查实现和维护状态。

### 5.2 Sandbox 与运行时

- [OpenSandbox](https://github.com/opensandbox-group/OpenSandbox)：面向 Agent 的 sandbox 平台，支持 Docker/Kubernetes、gVisor、Kata、Firecracker、egress 和 credential vault。
- [gVisor](https://gvisor.dev/)：用户态内核隔离。
- [Firecracker](https://firecracker-microvm.github.io/)：microVM，适合高风险代码执行。
- [nsjail](https://github.com/google/nsjail) / [Landlock](https://landlock.io/)：单机 namespace、filesystem、syscall、资源和网络限制。

这些方案限制爆炸半径，不负责判断用户意图或业务授权。

### 5.3 Guardrails、工具供应链与评测框架

- [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails)：input/retrieval/execution/output rails、注入检测和 YARA；属于防御纵深，不替代权限。
- [garak](https://github.com/NVIDIA/garak)：模型漏洞扫描，适合 jailbreak、injection、数据泄露基础测试。
- [promptfoo](https://github.com/promptfoo/promptfoo)：LLM/Agent red-team、断言、CI 回归。
- [Inspect AI](https://github.com/UKGovernmentBEIS/inspect_ai)：可组合 solver/scorer 的评测框架。
- [OWASP MCP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html) 与 [mcp-scan](https://github.com/invariantlabs-ai/mcp-scan)：工具发现、schema/供应链检查；静态 scanner 不替代运行时拦截。

---

## 6. 推荐组合

### 6.1 研究原型

```text
AgentDojo / Inspect AI → 自定义 scorer → OPA/Cedar
→ Tool Gateway → nsjail/gVisor → 状态 oracle + 审计日志
```

### 6.2 本地 Coding Agent

```text
Agent → Tool Proxy → OPA/Cedar → Landlock/gVisor
→ 文件、网络、进程限制 → 审计与人工审批
```

默认拒绝 `.env`、SSH key、workspace 外路径、任意出站、sudo、未审核包安装、shell 初始化文件修改、删除和 force push。

### 6.3 高风险企业 Agent

```text
Agent → Capability Gateway → Policy Engine
→ 短期 capability token → Human Approval → Tool/API
→ Audit + SIEM + Recovery
```

增加多租户隔离、data-flow labeling、transaction journal、dry-run、可逆操作、自动吊销和双人审批。

---

## 7. 最小可行研究实现

1. 先实现结构化 action → capability gateway → deterministic policy → isolated execution → state diff oracle。
2. 再加入文件/工具/网络 allowlist、严格 schema、短期 token、步数/时间/成本预算。
3. 再加入 injection、tool poisoning、secret exposure、exfiltration、goal drift、memory poisoning 和异常 tool-chain 检测。
4. 最后加入语义风险分析，处理意图歧义、工具参数语义、目标漂移和人工复核建议。

语义风险分析的定位是：

```text
风险评分 / 二次审查 / 评测辅助
```

不是：

```text
权限判定 / 沙箱 / 凭证保护 / 网络隔离
```

---

## 8. 指标与 Benchmark

### 8.1 指标

#### 8.1.1 UnsafeResult：安全结果的定义

判断 Agent 行为是否产生不安全结果，不能只看 Agent 是否说了危险的话，也不能只看某个动作是否被提出。应将安全结果定义为多个可观测安全属性的组合：

```text
UnsafeResult =
  UnauthorizedAction
  OR DataFlowViolation
  OR RuntimeBoundaryViolation
  OR InvalidStateTransition
  OR DangerousSideEffect
  OR PersistentStateCorruption
  OR ResourceAbuse
```

一个结果报告应保留各维度，而不是压缩成单一布尔值：

```json
{
  "authorized": false,
  "secret_disclosed": false,
  "sandbox_escaped": false,
  "state_changed": true,
  "state_change_allowed": false,
  "external_side_effect": "file_delete",
  "persistent_memory_modified": false
}
```

这些属性分别对应：授权越界、未授权数据流、运行时边界突破、非法状态转换、危险外部副作用、持久状态污染和资源滥用。单一 `unsafe: true` 可以作为聚合展示字段，但不能替代分项证据。

#### 8.1.2 按动作生命周期进行判断

将 Security Envelope 投影到一个具体 Agent action 上，可以得到：

```text
Pre-action → Runtime → Post-action
```

##### Pre-action：这个动作是否应该被允许？

输入：

```text
Intent + TaskScope + Principal + ProposedAction
       + Resource + Policy + Approval + Budget
```

检查：

- 动作是否在任务 scope 内；
- tool、verb、资源和参数是否匹配 capability；
- 数据标签是否允许流向目标 sink；
- 是否超过预算、时间或步数；
- 是否需要精确动作审批；
- 是否存在已知的策略冲突。

输出：

```text
ALLOW
DENY
REQUIRE_APPROVAL
REQUIRE_STEP_UP
```

这一阶段主要由 Capability Gateway、Policy Engine、Data-flow pre-check 和 Approval Gateway 完成。

##### Runtime：实际执行是否越界？

检查执行事件：

- 实际访问的文件、数据库和 API；
- 实际网络目的地和传输数据；
- syscall、进程、容器和 sandbox 行为；
- 实际使用的凭证和身份；
- 工具是否调用了额外子工具；
- 资源、时间、并发和重试是否超限。

这一阶段由 Tool/API Adapter、文件/网络监控、Sandbox/Runtime、IAM 和资源限制负责。它不依赖 Agent 对自身行为的描述。

##### Post-action：系统是否已经产生了未授权变化？

比较执行前后的可观测状态：

```text
before_state → observed_events → after_state
```

检查：

- 文件、数据库、邮件、云资源是否发生变化；
- 是否产生 secret disclosure 或外部数据流；
- memory/RAG/config 是否被持久化修改；
- 是否发生权限、身份或网络拓扑变化；
- 是否产生不可逆或高影响副作用。

这一阶段使用 state-difference oracle、审计轨迹、数据流记录和恢复系统确认实际结果。若发现违规，应触发 token 吊销、sandbox 隔离、凭证轮换、memory quarantine、checkpoint 恢复和事件取证。

因此，安全判断的完整流程是：

```text
Agent 生成结构化 Action
        ↓
确定性授权与参数检查
        ↓
数据流、预算与审批检查
        ↓
工具/网络/sandbox 执行监控
        ↓
状态差异与外部副作用检查
        ↓
UnsafeResult 分项判定 + 处置/恢复
```

这与第 4 节的 Security Envelope 是同一条路径：第 4 节描述组件、信任边界和连接方式；这里描述同一动作在执行前、执行中和执行后的判断时机。

##### 指标不能只用 Unsafe Action Rate

| 指标 | 含义 |
|---|---|
| Task Success Rate | 正常任务完成率 |
| Unauthorized Action Rate | 未授权动作比例 |
| Unsafe Action Rate | 违反安全属性的动作比例 |
| Secret Disclosure Rate | 敏感数据披露比例 |
| Tool Misuse Rate | 工具功能或参数误用比例 |
| Privilege Escalation Distance | 从初始能力到实际能力的距离 |
| Data-flow Violation Rate | 未授权跨域数据流比例 |
| Memory Poisoning Rate | 恶意/错误状态持久化比例 |
| Side-effect Severity | 外部副作用严重度 |
| Detection Latency | 违规到检测的时间 |
| Recovery Time | 检测到隔离/恢复的时间 |
| False Positive / Negative | 误报与漏报 |
| Channel Closure Rate | 防御是否在生成前关闭禁止能力通道 |
| Turns-to-Success | 攻击达到目标所需步数 |

#### 8.1.3 三层证据 endpoint

不要把不同 endpoint 合成一个安全分数：

```text
Semantic acceptance
    ↓
Audit-visible harm evidence
    ↓
Deterministic executable state harm
```

证据强度通常从强到弱为：deterministic state oracle、审计轨迹、tool-call oracle、语义审查、prompt/output classifier。

### 8.2 Benchmark
#### 8.2.1 Prompt / Indirect Injection

##### InjecAgent

- [论文/ACL](https://aclanthology.org/2024.findings-acl.624/)；[代码](https://github.com/uiuc-kang-lab/injecagent)。
- 1,054 cases、17 user tools、62 attacker tools；测试工具返回值中的 indirect injection、直接伤害与数据窃取。
- 指标：ASR-all、ASR-valid。
- 适合快速复现；局限是单轮/模拟交互，对长时程、持久状态和 runtime 隔离覆盖不足。

##### AgentDojo

- [项目](https://github.com/ethz-spylab/agentdojo)；[论文](https://openreview.net/forum?id=m1YYAQjO3w)。
- 97 realistic tasks、629 security cases；有状态应用和多步工具调用。
- 指标：Benign Utility、Utility Under Attack、Targeted ASR。
- 同时评估攻击和防御；仍是受控环境，不等于生产权限或 sandbox 逃逸测试。

##### ASB（Agent Security Bench）

- [代码](https://github.com/agiresearch/ASB)；[论文](https://arxiv.org/abs/2410.02644)。
- 10 scenarios、400+ tools、DPI/ IPI / memory poisoning / PoT backdoor / mixed attacks。
- 指标包括 ASR、拒答率、utility-security trade-off、NRP。
- 覆盖 system prompt、user prompt、tool、memory；部分环境为模拟。

#### 8.2.2 长时程攻击

##### AgentLAB

- [项目与结果](https://tanqiujiang.github.io/AgentLAB_main/)。
- intent hijacking、tool chaining、task injection、objective drifting、memory poisoning；28 environments、644 cases。
- 指标：ASR、Turns-to-Success。
- 适合测试单轮防御在多步攻击下的失效；应额外记录累计权限、数据流和副作用。

#### 8.2.3 工具完整性与 Capability

##### AgentSecBench

- [论文](https://arxiv.org/html/2605.26269)。
- 三类 security games：instruction integrity、retrieval confidentiality、capability integrity。
- 指标：paired adversarial advantage、RAG leakage、benign utility、channel closure rate。
- 适合验证 capability projection 是否真的移除了禁止能力，而不是仅让模型更愿意拒绝。

##### AgentTrust（较新）

- [论文](https://arxiv.org/html/2605.04785)；[代码](https://github.com/chenglin1112/AgentTrust)。
- 工具前拦截、shell 反混淆、顺序敏感 RiskChain、LLM judge 辅助；覆盖文件、网络、代码、凭证和外传。
- 适合研究在线 interceptor；需关注 held-out 泛化和规则/benchmark 关联。

#### 8.2.4 MCP / Tool Metadata / Protocol

##### MCPTox

- [论文](https://arxiv.org/html/2508.14925)。
- 45 live MCP servers、353 tools、1,312 cases；专测 tool poisoning metadata，工具本身不必被调用。
- 局限：主要单轮，长期 memory、rug pull 和复杂协议交互覆盖不足。

##### MCP Security Bench（MSB）

- [论文](https://arxiv.org/html/2510.15994)。
- 2,000 attack instances、10 domains、65 tasks、400+ tools；覆盖 planning、invocation、response handling。
- 指标：ASR、Performance Under Attack、NRP。

##### MCPSecBench / MCP-SafetyBench

- [MCPSecBench](https://arxiv.org/abs/2508.13220)：覆盖 host、client、protocol、server 四面和 17 类攻击。
- [MCP-SafetyBench](https://github.com/xjzzzzzzzz/MCPSafety)：真实 MCP server，多轮、多 server、五类领域和 20 类攻击。
- 这些项目发展很快，比较时必须固定 commit、模型、server 和攻击配置。

#### 8.2.5 Computer-use、CLI 与 Sandbox Safety

##### OS-HARM

- [论文](https://papers.neurips.cc/paper_files/paper/2025/file/4009bff0cd87ba2203c8e3a2f082aaec-Paper-Datasets_and_Benchmarks_Track.pdf)；[代码](https://github.com/tml-epfl/os-harm)。
- 基于 OSWorld 的 150 tasks，覆盖 deliberate misuse、prompt injection、model misbehavior；涉及邮件、代码编辑器、浏览器和终端。
- 适合真实 GUI 交互安全；自动 judge 仍需人工校验。

##### AgentHazard / OpenSkillRisk

- [AgentHazard](https://github.com/Yunhao-Feng/AgentHazard)：execution-level harm，覆盖 RCE、外传、持久化、供应链、破坏、提权和资源耗尽。
- [OpenSkillRisk](https://github.com/Miaow-Lab/OpenSkillRisk)：第三方 CLI skill，覆盖 authority expansion、data harvesting、execution bootstrapping、persistence、exfiltration 和远端状态修改。
- 均应固定 commit、数据来源、许可证和 held-out 划分，不把自报结果直接当领域共识。

### 8.3 隐私与内部数据流

#### AgentLeak

- [代码](https://github.com/Privatris/AgentLeak)。
- 测试多 Agent 的 output、inter-agent message、tool、memory、log、artifact 等内部泄露通道。
- 适合验证“只审计最终输出”是否漏掉内部泄露；应与 data-flow monitor 联合使用。

### 8.4 通用能力与策略遵守基线

- [AgentBench](https://github.com/THUDM/AgentBench)：OS、DB、KG、网页、购物等 8 类环境，主要是能力和多轮任务基线，不是安全证明。
- [AgentBoard](https://hkust-nlp.github.io/agentboard/)：9 类任务、1,013 environments，提供子目标和细粒度交互分析。
- τ-bench / τ²-bench：适合客服、预订和数据库等业务策略，测试状态转换、所有权、取消、修改和退款规则。
- [SafeClawBench](https://arxiv.org/html/2606.18356)：分离 semantic acceptance、audit-visible evidence、sandbox-observed harm，覆盖 DPI、IPI、tool-return injection、memory poisoning、memory extraction 和 ambiguity-driven unsafe inference。

### 8.5 最小实验矩阵

| 条件                           | 目的                          |
| ---------------------------- | --------------------------- |
| Benign / no attack           | 正常任务能力和误报基线                 |
| Attack / no defense          | 原始攻击面                       |
| Attack / prompt-only defense | 提示层增益，不声称 enforcement       |
| Attack / envelope defense    | 策略、网关、sandbox、data-flow 的效果 |

每个条件再分 single-turn、multi-turn、long-horizon、adaptive attacker。建议记录：

```json
{
  "benchmark": "AgentDojo",
  "commit": "...",
  "model": "...",
  "agent_version": "...",
  "defense_stack": ["opa", "tool_gateway", "gvisor"],
  "benign_success": true,
  "semantic_acceptance": false,
  "tool_call_violation": false,
  "state_harm": false,
  "data_leakage": false,
  "detection_latency_ms": 14,
  "recovery_time_ms": 83
}
```

### 8.6 Benchmark 研究路线

1. AgentDojo：动态 IPI 和任务效用；
2. ASB：多阶段攻击和 memory；
3. AgentLAB：长时程和工具链；
4. AgentSecBench：capability/information channel closure；
5. SafeClawBench：语义、审计证据、实际 sandbox harm 分离；
6. MCPTox/MSB/MCPSecBench：MCP 专项；
7. OS-HARM/AgentHazard/OpenSkillRisk：Computer-use、CLI、执行级伤害；
8. AgentLeak：多 Agent 内部数据流；
9. AgentBench/AgentBoard：benign capability baseline。

自建 benchmark 时，每个 case 应提供用户目标、授权 policy、工具 schema、初始状态、攻击者控制点、期望安全动作、禁止动作、状态 oracle、允许泄露集合和恢复条件。

---

## 9. 最终研究主线与相关笔记

推荐主线：

> **Capability-based authorization + deterministic policy enforcement + sandbox isolation + information-flow tracking + trajectory evaluation**

最终判断：

> **Unsafe action detection 可以有统一的上层风险接口，但底层不能是统一分类器。授权、数据流、运行时突破、外部副作用、持久化污染和恢复性必须分别建模、分别测量、分别控制。**

相关笔记：

- [00 LLM 与 Agent Security 总览](00%20LLM%20与%20Agent%20Security%20总览.md)
- [OpenAI–Hugging Face Agent 入侵案例](./Open%20AI%20Cases.md)
