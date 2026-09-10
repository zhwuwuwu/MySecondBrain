# AgentAegis 与 Acacian Aegis 安全项目分析及 SuperClaw 集成建议

**日期：** 2026-08-13  
**状态：** 调研结论，未授权实施  
**触及契约：** `Capability`、`AgentRuntime`、`Sanitizer`、`ServiceHub`；当前为设计分析，不改变运行时契约  
**范围：** 比较 `antgroup/agent-aegis` 与 `Acacian/aegis` 的定位、功能和架构，并评估其与 SuperClaw（OpenCode/OpenWork）的集成路径。

---

## 1. 结论摘要

两个项目并非同类替代品：

- **AgentAegis** 是面向 OpenClaw 生命周期的 TypeScript 运行时防护插件。其优势是 Agent 行为附近的专用规则，尤其是工具调用、Shell、技能、记忆和自保护。
- **Acacian Aegis** 是 Python Agent 的治理库。其优势是独立的策略、审批、审计和文本 guardrail 模型，以及多个 Python 框架的适配层。

两者都**不能直接安装到 SuperClaw 的 OpenCode runtime**：前者依赖 OpenClaw Plugin API，后者没有 OpenCode 适配器且运行于 Python。若后续实施，推荐将 AgentAegis 作为“检测与执行控制规则”的参考来源，将 Acacian Aegis 作为“策略决策、审计数据模型与输出净化”的参考来源；实际拦截落在 SuperClaw 的 OpenCode 插件和 OpenWork Server 边界。

本分析以两个仓库于 2026-08-13 可见的源码为准，不将 README 的安全覆盖承诺理解为对所有攻击变体的保证。

---

## 2. 分析对象与证据范围

| 项目 | 固定源码版本 | 许可证 | 证据性质 |
|---|---|---|---|
| [antgroup/agent-aegis](https://github.com/antgroup/agent-aegis) | `23d59b8986b86de448d24ff973e662295768d533`，包版本 `2026.3.14` | Apache-2.0 | Hook、策略、配置和运行时状态来自源码；产品定位同时参照 README |
| [Acacian/aegis](https://github.com/Acacian/aegis) | `8743c1c25801856c7d148e20694b944959399fc0`，版本 `1.0.0` | MIT | 内核、guardrail、策略和审计来自源码；框架数量与覆盖率属于项目声明，需以 CI/发布工件复核 |

### 2.1 术语

- **observe**：记录、告警或提供上下文，但不改变执行结果。
- **enforce / block**：在副作用发生前拒绝调用，或在出口处脱敏/替换内容。
- **最终出口**：回复被桌面端、Slack/Telegram 或其他客户端获得之前的服务端响应/持久化边界。
- **安全能力闭环**：一项能力在输入、决策、执行、输出、存储与审计中有明确责任方，而不只是“某个 Hook 存在”。

---

## 3. 基础对比

| 维度 | AgentAegis | Acacian Aegis |
|---|---|---|
| 核心定位 | OpenClaw Agent 的全生命周期运行时防护插件 | 多 Python Agent 框架的策略治理与 guardrail 库 |
| 语言/运行时 | TypeScript、Node.js、ESM | Python ≥3.11、Hatchling 打包 |
| 接入方式 | OpenClaw Plugin API 生命周期 Hook | `auto_instrument()` / 环境变量触发的框架 monkey-patch，或显式调用核心 API |
| 原生框架范围 | OpenClaw | LangChain、CrewAI、OpenAI Agents、LiteLLM、Google ADK/GenAI、Pydantic AI、LlamaIndex、Instructor、DSPy，以及 OpenAI/Anthropic 原生 SDK 等 |
| 安全模型 | 规则与策略串成生命周期防线，偏执行控制 | `Action → Policy → Approval → Execute/Verify → Audit`，偏治理控制面 |
| 默认状态/审计 | 插件状态及 JSONL 防御事件 | SQLite 审计，可选导出与外部存储集成 |
| 控制台/服务 | 可选 React + Express + WebSocket WebUI | 可选 Starlette ASGI 服务、MCP server/proxy、导出器 |
| 与 OpenCode 的原生兼容 | 无 | 无 |

### 3.1 直接复用性

| 组件 | AgentAegis | Acacian Aegis |
|---|---|---|
| 可直接作为 OpenCode 插件安装 | 否。OpenClaw Hook 名称、上下文类型和插件装载机制不同 | 否。Python 包，且未提供 OpenCode instrumentation |
| 可迁移规则/模型 | 高：Shell、编码混淆、技能/记忆、自保护、外泄链的规则和顺序 | 高：Action、Risk、Policy、Approval、Audit、PII/文本 guardrail 的数据模型 |
| 可作为独立侧车服务 | 低至中：需要剥离 OpenClaw runtime | 中：可做 Python policy/audit 服务，但跨进程增加延迟、认证和故障域 |
| 推荐初始用途 | 作为 TypeScript adapter 的安全规则参考 | 作为策略语言、审计 schema 和输出净化设计参考 |

---

## 4. 功能对比

### 4.1 能力矩阵

| 安全能力 | AgentAegis | Acacian Aegis | 评估 |
|---|---|---|---|
| 用户输入与越狱检测 | `message_received` 用户风险扫描、prompt guard | 注入 guardrail，源码含多类正则模式 | 两者均可参考；不应将规则命中视为绝对攻击判定 |
| 动态安全上下文 | `before_prompt_build` 注入上下文 | 主要通过 guardrail 与策略影响执行 | AgentAegis 更贴近 Agent 生命周期；仍须防止提示注入仅成为 advisory 文本 |
| 高危工具/Shell | `before_tool_call` 的十段防御链 | 对标准化 `Action` 作 policy 决策 | AgentAegis 更有专项深度；Acacian 的 policy 更便于运营配置 |
| 编码与命令混淆 | Base64/Base32/Hex/URL 有界解码，Unicode 与命令混淆检查 | 非其专用核心 | AgentAegis 更适合做规则来源 |
| 文件/工作区破坏 | 保护路径、工作区删除、自保护策略 | 可通过 target/action policy 表达 | AgentAegis 有现成意图；须转换为 SuperClaw 的路径与权限模型 |
| Skill/插件投毒 | 启动扫描队列与 Worker，运行时资产保护 | 非核心能力 | AgentAegis 明显更接近需求，但不是完整供应链审计 |
| 工具结果间接注入 | `after_tool_call` 结果扫描 | Guardrail pipeline 可处理文本 | 二者都需要可靠的工具结果出口；不能撤销已发生的工具副作用 |
| 记忆污染 | `before_message_write`/memory guard | 可用 Action/Policy 承载，但非专用 memory runtime | AgentAegis 更贴近；需先确认 SuperClaw 的真实 memory 写入路径 |
| 凭据/PII 输出 | 输出 redaction | PII 检测、掩码和审计字段脱敏 | Acacian 的模型更系统化；最终脱敏必须置于服务端出口 |
| SSRF 与外泄链 | 外联/外泄链规则 | 以 Action/target policy 建模 | AgentAegis 有专项检查；仍不替代网络出口/DNS/容器隔离 |
| 循环与资源耗尽 | loop guard | 可用策略建模，但非专项 | AgentAegis 更直接 |
| 审批 | 可阻断或要求确认的插件语义 | `AUTO`、`APPROVE`、`BLOCK` 是显式决策 | SuperClaw 必须统一到 OpenWork Server 的现有审批，不能维护两套真相源 |
| 审计与导出 | 防御事件 JSONL | 结构化 SQLite 审计、敏感字段遮蔽、导出器 | Acacian 的设计更值得借鉴 |

### 4.2 AgentAegis 的工具调用防御链

其 `before_tool_call` 中的策略按顺序执行：自保护、工作区删除、OpenClaw 命令、命令混淆、危险命令、内联执行、记忆、脚本溯源、循环、外泄。顺序的意义是先阻止安全控制被关闭及明显破坏，再检查内容与链路风险。

该顺序可作为 SuperClaw adapter 的参考，但不能照抄：OpenClaw 命令、路径、memory 名称、工具参数 shape 和 OpenCode 不同。应首先把 OpenCode 的工具调用标准化为 `Action`，再执行规则。

### 4.3 功能限制

1. 两个项目都主要使用规则、模式或策略匹配；对于编码、分片、跨轮次或业务语义攻击均可能存在漏报与误报。
2. AgentAegis 的 Hook 采用 fail-open 包装。SuperClaw 对高风险动作需要另行定义“策略引擎不可用时”的 fail-closed 范围，不能无条件继承该选择。
3. AgentAegis 的技能扫描预算和单进程状态不构成供应链扫描、主机 EDR 或分布式审计系统。
4. Acacian 的 instrumentation 依赖 monkey-patch；它的多框架适配能力不等于适配 OpenCode。
5. 输出脱敏与工具结果扫描都不能替代主机 ACL、容器沙箱、网络出口控制及 secrets 管理。

---

## 5. 架构对比

### 5.1 AgentAegis：Hook-first 的内聚插件

```text
OpenClaw 生命周期事件
  → handlers.ts：12 个 Hook 的安全编排
  → rules.ts / security-strategies.ts：检测、清理与顺序策略
  → state.ts / scan-service.ts：运行状态、技能扫描与事件
  → allow / observe / enforce
```

五层叙事对应启动可信基座、输入感知、认知状态、决策对齐和执行控制。其核心价值在于安全检查就在 Agent 执行附近：例如工具调用前阻断、工具结果后标记、记忆写入前检查。

代码层面注册了 `gateway_start`、`message_received`、`message_sending`、`before_prompt_build`、`before_dispatch`、`before_agent_reply`、`before_tool_call`、`after_tool_call`、`before_message_write`、`llm_output`、`agent_end` 与 `session_end` 等 Hook。该完整生命周期是 **OpenClaw 专用适配层**，不是可直接映射到 OpenCode 的公共接口。

#### 5.1.1 “插件内聚”的具体含义

“插件内聚”是对 AgentAegis 结构的描述，不是 OpenClaw 的官方术语。它表示以下责任围绕同一个 Agent runtime 插件集中编排：

```text
OpenClaw
  → index.ts：Hook 注册
  → handlers.ts：生命周期编排
  → rules.ts / security-strategies.ts：检测与策略顺序
  → scan-service.ts / scan-worker.ts：Skill 扫描
  → state.ts：会话/运行状态
  → defense events：安全事件记录
```

这种内聚使插件能共享会话、Agent、workspace、工具调用和安全状态。例如“读取 secret → 访问外部 URL → 把 secret 拼入请求”的风险，需要跨多个工具调用观察链路，而不是只检查单条命令。安全检查也靠近副作用：`before_tool_call` 可以在 Shell、文件写入或网络工具真正执行前返回阻断。

内聚的代价是运行时耦合。OpenClaw 命令、Skill、memory 路径、session 对象和配置模型都嵌入规则或处理器；迁移到 OpenCode 时应迁移规则和防御顺序，重新实现生命周期 adapter，而不是复制 Hook 名称和类型。AgentAegis 的 Hook fail-open 也不能直接成为 SuperClaw 的统一故障策略：低风险动作可以告警放行，高风险 Shell、外泄和安全配置篡改可能需要 fail-closed。

### 5.2 Acacian Aegis：Policy-first 的可分层治理内核

```text
框架适配 / SDK monkey-patch / 显式调用
  → Action 标准化
  → GuardrailEngine：injection / PII / toxicity / prompt leak
  → Policy.evaluate(Action)：规则、条件、风险和审批决定
  → execute / verify
  → Audit：session / chain 关联、敏感字段脱敏、导出
```

它将纯数据模型（`core/`）、文本 guardrail（`guardrails/`）、框架插桩（`instrument/`）、执行/审计（`runtime/`）和可选 MCP/HTTP 边界分开。相对于 AgentAegis，它更容易被改造成跨运行时治理方案；代价是现成 framework adapter 对 OpenCode 没有帮助。

#### 5.2.1 各层如何实现和组合

```text
框架/SDK adapter 或显式调用
  → Action：把 tool、模型请求、文件/网络动作统一表示
  → GuardrailEngine：检查/转换文本
  → Policy.evaluate(Action)：匹配规则、条件和风险
  → AUTO / APPROVE / BLOCK
  → execute / verify
  → Audit：记录 session、chain、结果并脱敏
```

- **`core/`** 是尽量不依赖具体 Agent 框架的治理内核，包括 `Action`、风险等级、策略规则、条件、审批决定、执行计划和结果。不同框架的调用先归一化为 `Action`，策略层无需知道调用来自哪个 SDK。
- **`guardrails/`** 是内容检查层，包括 prompt injection、PII、toxicity 和 prompt leak。`check()` 只返回检查结果，`check_and_transform()` 还可以返回脱敏或转换后的文本。因此 Guardrail 解决“文本是否危险/是否要转换”，不独立承担文件删除等完整权限决策。
- **`instrument/`** 是框架适配层，把 LangChain、CrewAI、OpenAI Agents 等入口接到统一流程；`integrations/` 更靠近 OpenAI/Anthropic 原生客户端。两层都属于接入代码，不是策略本身。
- **`runtime/`** 负责将决策落实为执行、验证和审计：`AUTO` 直接执行，`APPROVE` 等待人工确认，`BLOCK` 拒绝；无论结果如何都应形成结构化审计记录。
- **MCP proxy/server 与 HTTP server** 是可选边界：可在 MCP Client 与 Server 之间集中检查工具参数和审计，但不能覆盖绕过该代理的路径，也会引入认证、延迟和可用性问题。

因此 Acacian Aegis 的“可组合”不是把所有检查放进一个 Hook，而是让不同 adapter 产出统一 `Action`，再由 Guardrail、Policy、Runtime 和 Audit 依次组合。

#### 5.2.2 Monkey-patch 是什么

Monkey-patch（运行时打补丁）是在不修改第三方库源码的情况下，运行时替换或包裹其函数、方法或类。例如：

```python
original = Client.request

def guarded_request(self, prompt):
    decision = check_prompt(prompt)
    if decision.blocked:
        raise SecurityError("blocked")
    return redact(original(self, prompt))

Client.request = guarded_request
```

应用仍然调用 `Client.request()`，但实际先进入 Aegis wrapper，再检查输入、调用原函数并处理输出。Acacian Aegis 用这种方式为多个 Python 框架提供 `auto_instrument()`，优点是业务代码改动小、接入快；缺点是依赖第三方库内部入口，升级可能导致 patch 失效或静默绕过，也可能在上层框架和底层 SDK 同时被 patch 时重复检查。

这与 OpenCode Plugin Hook 不同：monkey-patch 是安全库主动替换被调用库；Plugin Hook 是 Agent runtime 主动按照正式契约触发扩展。SuperClaw 应优先使用 OpenCode Hook，不应为了复用 Acacian 的 instrumentation 而在 Node/OpenCode 内部做运行时替换。

### 5.4 架构取舍

| 问题 | AgentAegis 取向 | Acacian Aegis 取向 | SuperClaw 应取的组合 |
|---|---|---|---|
| 安全检查离副作用多近 | 很近，直接绑定 Hook | 取决于 adapter/executor | 工具前置检查留在 OpenCode plugin |
| 策略可配置性 | 多个防御开关/模式 | YAML policy、风险/审批模型 | 用统一 policy 决策，避免分散环境变量 |
| 审批所有权 | 插件可表达确认 | 治理内核有审批状态 | OpenWork Server 为唯一审批真相源 |
| 审计形态 | 插件本地事件 | 结构化审计链 | 在服务端形成结构化审计；插件仅产生安全事件 |
| 跨语言/跨进程 | 不考虑 | 可通过 server/adapter 扩展 | 初期避免新增 Python sidecar；先实现 TypeScript adapter |

---

## 6. SuperClaw 集成对比与建议

### 6.1 已验证的本项目边界

1. OpenCode 资源位于 `sandbox/agent_config/.opencode/`，本地插件已使用 `tool.execute.before` 和 `chat.params`。例如 `08_protected_pipeline_confirm_guard.ts` 已在工具调用前阻断对受保护 Agent 的分派，`95_superclaw_output_token_cap.ts` 已用 `chat.params` 改写模型参数。
2. OpenWork Server 提供 `/opencode/*` 代理、插件管理与审批 API；它是 Chat 数据平面上的服务端边界。
3. OpenCode 配置由 capability profile 投影并被 hash gate 约束。产品资源不能通过容器内手工编辑长期生效；新增插件/策略必须进入 feature artifact source、生成流程和投影验证。
4. ServiceHub 是 Windows 主机控制平面和子服务主管理者，但架构明确规定它不承载 Chat 数据平面。因此不要为了安全检查把用户聊天强行改走 ServiceHub。
5. `host/security_manager/` 当前包含 data protection、LLM pipeline 与 protected file guard。新能力若属于跨边界 Sanitizer/数据保护，应评估是否扩展这个已有契约，而不是再造并行安全主干。

### 6.2 推荐目标形态

```text
                         ┌──────────────────────────────────────┐
                         │ OpenCode Security Adapter (TypeScript)│
                         │ - 输入风险与安全上下文                │
Agent tool call ────────▶│ - 工具 Action 归一化                  │
                         │ - before-tool 强制规则                │
                         │ - 工具结果标记/净化（经 PoC 验证）     │
                         └──────────────┬───────────────────────┘
                                        │ security decision/event
                                        ▼
                         ┌──────────────────────────────────────┐
                         │ OpenWork Server / Sanitizer Adapter   │
                         │ - 统一审批状态与用户确认              │
                         │ - 最终输出 secret/PII 脱敏            │
                         │ - 结构化安全审计与会话关联            │
                         └──────────────────────────────────────┘
```

该形态吸收两项目的长处，但不引入第二套运行时：

| 层 | 借鉴来源 | 实际责任 |
|---|---|---|
| 规则与执行控制 | AgentAegis | Shell/编码/路径/自保护/循环/外泄链在副作用前阻断 |
| Policy decision | Acacian Aegis | 把工具、网络、文件和 memory 写入标准化为 Action，输出 allow/observe/approval/block |
| 审批 | SuperClaw 既有 OpenWork Server | 唯一审批记录、超时和用户交互边界 |
| 输出 Sanitizer | Acacian Aegis 的 guardrail/audit 思想 + 现有 security manager | 最终回复及持久化数据离开服务端前脱敏 |
| 配置与上线 | SuperClaw Capability | 将功能与最低 grade、manifest、投影 hash 和 UI 配置关联 |

### 6.3 两种不推荐的集成方式

**方案 A：直接安装 AgentAegis。** 不可取，因为 OpenClaw Hook 与 OpenCode Plugin API 不兼容，且受 capability projection 管理的 `.opencode` 目录会覆盖手工配置。

**方案 B：把 Acacian Aegis 作为常驻 Python sidecar 并让所有调用同步 RPC。** 初期不推荐。它会新增跨语言 RPC、认证、可用性与延迟问题，却仍要自行完成 OpenCode Hook adapter。只有当组织需要跨多个非 Node Agent runtime 统一治理时，才值得评估。

### 6.4 实施拆分原则

主计划应按**安全能力闭环**拆分，而不是按“把 OpenClaw Hook 翻译成 OpenCode Hook”拆分；同时维护 Hook/边界映射表作为设计输入。

| 能力工作流 | 首选 SuperClaw 落点 | AgentAegis 对应 | Acacian 对应 | 关键验收 |
|---|---|---|---|---|
| 配置与自保护 | feature artifact、投影 hash、受保护路径 | gateway scan / self protection | policy config | Agent 不能关闭或改写生效策略 |
| 输入与上下文 | OpenWork 入站 + `chat.params` | message/prompt hooks | injection guardrail | 命中被审计，策略可 observe/enforce |
| 工具前阻断 | `tool.execute.before` + OpenWork approval | before tool chain | Action/Policy/Approval | 拒绝发生在副作用前，审批不出现双重真相源 |
| 工具结果与最终输出 | after-tool（先 PoC）+ OpenWork 响应/落库出口 | after tool / output | guardrail/PII/audit | 外部文本被标记，secret 不可越过最终出口 |
| 记忆写入 | 实际 memory 写入工具/存储边界 | memory guard | Action policy | 写入有来源、授权与审计 |
| 会话审计 | OpenWork 会话边界 | session end | audit chain | 事件可关联 session/operation，敏感字段被遮蔽 |

### 6.5 分阶段上线

1. **PoC / observe：** 先只采集输入、工具、输出安全事件，验证 OpenCode hook 的实际参数、工具结果边界和误报率。
2. **高置信度 enforce：** 首先阻断关闭安全控制、受保护路径删除/覆盖、明显危险 Shell、云 metadata/私网 SSRF、明显凭据输出。
3. **审批接入：** 对存在合法场景的高风险外联、数据外发、记忆写入，通过既有 OpenWork approval 请求用户确认。
4. **策略运营：** 将规则开关、模式、最低 capability grade 和审计保留期限纳入产品配置及 manifest；每个 grade 都做投影与行为测试。

---

## 7. 迁移风险与决策门槛

| 风险 | 影响 | 缓解要求 |
|---|---|---|
| 未验证 Hook | 输出或工具结果可能绕过防线 | 在任何 enforce 前针对当前锁定 OpenCode 版本做集成 PoC |
| 双重审批 | 同一动作一处放行、一处拒绝 | OpenWork Server 是唯一审批状态源 |
| fail-open 继承 | 安全插件异常会放行高风险动作 | 明确每条能力的失败语义，高风险工具原则上 fail-closed |
| 规则误报 | 破坏正常 Agent 使用体验 | observe 数据与规则版本化；只逐类推进 enforce |
| 配置漂移 | 手工配置被 materializer 覆盖，或不同 grade 行为不一致 | 所有资源经 manifest/source artifact/generator 管理 |
| 增加安全旁路 | 新 sidecar/新代理创造新的请求路径 | 初期复用 OpenCode 与 OpenWork 既有路径；新增服务须说明 ServiceHub 契约与数据平面边界 |

**实施前的决策门槛：**

- 确认工具结果的真实处理 Hook/出口，并用当前锁定的 OpenCode 版本验证；
- 确认所有 memory 持久化路径，不只保护 `MEMORY.md` 的名称假设；
- 确认 `host/security_manager/` 的 Sanitizer 所有权，避免创建第二条安全主干；
- 为最小 capability grade 明确可用的基础安全能力，并在 manifest 中声明；
- 定义审计保留、脱敏、访问控制与故障时的策略。

---

## 8. 参考资料

### 外部项目

- AgentAegis：[仓库](https://github.com/antgroup/agent-aegis)、[Hook 注册](https://github.com/antgroup/agent-aegis/blob/23d59b8986b86de448d24ff973e662295768d533/index.ts#L33-L68)、[配置](https://github.com/antgroup/agent-aegis/blob/23d59b8986b86de448d24ff973e662295768d533/src/config.ts#L81-L119)、[工具策略](https://github.com/antgroup/agent-aegis/blob/23d59b8986b86de448d24ff973e662295768d533/src/security-strategies.ts#L737)
- Acacian Aegis：[仓库](https://github.com/Acacian/aegis)、[架构](https://github.com/Acacian/aegis/blob/8743c1c25801856c7d148e20694b944959399fc0/ARCHITECTURE.md)、[自动插桩](https://github.com/Acacian/aegis/blob/8743c1c25801856c7d148e20694b944959399fc0/src/aegis/instrument/__init__.py#L43-L56)、[策略示例](https://github.com/Acacian/aegis/blob/8743c1c25801856c7d148e20694b944959399fc0/policy.example.yaml)、[审计](https://github.com/Acacian/aegis/blob/8743c1c25801856c7d148e20694b944959399fc0/src/aegis/runtime/audit.py#L25-L68)

### 本项目

- [V1.1 架构](2026-06-29_v1.1_refactor_architecture.md)
- [V1.2 capability 架构](2026-07-15_v1.2_capability_architecture.md)
- [ServiceHub 实施计划](2026-06-30_servicehub_implementation_plan.md)
- [OpenCode Router 说明](../components/opencode-router.md)
- [Capability feature artifacts](../../sandbox/agent_config/feature-artifacts/README.md)
- [OpenWork Server README](../../sandbox/server/README.md)

---
