# SuperClaw 当前安全能力与缺口评估

**状态：** 调研结论（非实现计划）  
**日期：** 2026-08-12  
**对象：** 当前 SuperClaw；一个基于 OpenCode、OpenWork Server、WSL2/Docker sandbox 与 Windows host 服务的类 Claw 个人 Agent 助手。  
**更正：** SuperClaw 不是 OpenClaw。本文件不把 OpenClaw 的功能当成 SuperClaw 已有能力；OpenClaw/AgentAegis 仅可作为未来设计的外部参考。

## 1. 结论

SuperClaw 已具备一组实用但分散的安全相关能力：数据面与控制面的 token 认证、owner/collaborator/viewer 权限、人工审批框架、工作区授权根、WSL2/Docker 运行隔离、OpenCode 工具前置 guard、受保护文件掩码流程、PII 文本/文件脱敏、会话轨迹、工作区 JSONL 审计和遥测。

它们能降低误操作、未授权 API 访问、将部分 PII 直接提交给模型、错误 Agent 路由和局部资源耗尽风险。但它们**尚未组成统一的 Agent Security 系统**：没有一个覆盖工具、MCP、网络、记忆与最终输出的统一策略决策点，也无法证明所有安全关键动作都有完整、脱敏、可长期保存的审计记录。

## 2. 已有能力（按安全效力重新分类）

| 类别 | 当前能力与证据 | 实际覆盖 | 评价与限制 |
|---|---|---|---|
| API 身份与权限 | OpenWork token 有 `owner`、`collaborator`、`viewer` scope，token 存 hash；见 `sandbox/server/src/tokens.ts`。OpenWork 代理对部分路径按 scope 限制。 | 限制未授权客户端与 viewer 的部分写操作/自我审批。 | 依赖 token 文件与系统 ACL；持有 owner token 的本地进程可获得 owner 权限。新敏感代理路径需要显式纳入授权规则。 |
| 控制面认证 | ServiceHub bearer token 保护控制面。 | 防止未授权控制面操作。 | 不保护 chat data plane；token 泄露仍可被完全滥用。 |
| 人工审批 | `ApprovalService.requestApproval/respond` 支持 `manual` 或 `auto`；见 `sandbox/server/src/approvals.ts`。 | 为接入它的操作提供 allow/deny/timeout。 | `pending` 为内存 Map；重启即丢失。不是所有高风险工具调用的强制统一入口，`auto` 模式会直接放行。 |
| 工作区边界 | OpenWork 配置有 authorized roots/workspace 概念；OpenCode 有 agent/tool permission；容器在 WSL2/Docker sandbox 内运行。 | 缩小服务访问目录和 Windows host 的暴露面。 | OpenCode permission 不能代替 OS 级隔离；必须逐项验证 bash、MCP、挂载和自定义工具都遵守根目录策略。容器内可访问的 workspace 与网络仍是攻击面。 |
| 工具调用 guard | 本地 OpenCode 插件在 `tool.execute.before` 执行检查。例：`90_superclaw_default_tool_guard.ts` 限制 agent/tool 路由；`08_protected_pipeline_confirm_guard.ts` 阻止未经确认派发到 protected-file-agent。 | 在副作用前拦截特定 agent、task 和受保护文件流程。 | 规则分散且偏业务流程；并非统一的 shell、文件、网络、MCP、外发策略。当前 guard 不等于通用恶意命令检测。 |
| 可用性/资源 guard | `93_superclaw_repeat_command_guard.ts`、`94_superclaw_read_output_cap.ts`、输出 token/step/lineage 插件。 | 限制重复调用、过大 read 输出和 context overflow。 | 主要是可靠性/成本保护；不能视为数据泄漏、命令执行或权限安全控制。 |
| PII 文件和文本保护 | Windows host `security_manager` 对支持的文件和自由文本执行掩码，模式为 `none`/`deterministic`/`hybrid`/`llm`；见 `host/security_manager/README.md`。 | 已接入保护流水线的 CSV/XLSX/Parquet/TXT/MD/HTML/DOCX 和文本，在进入模型前可降低 PII 暴露。 | `none` 不做保护；绕过服务的 bash/read/MCP/自定义工具路径不受保护。placeholder→原值 registry 在 TTL 内仍保存敏感映射。 |
| 受保护文件确认 | protected pipeline 使用一次性 session 确认 token；见 `08_protected_pipeline_confirm_guard.ts`。 | 阻止未确认的 `task → protected-file-agent`。 | 仅覆盖该特定 subagent 派发，不是任意敏感文件读写或外发控制。 |
| 模型与数据本地性 | LLM Router 允许 local/cloud 路由，产品支持本地模型。 | 本地路径可减少向云模型发送数据。 | 这不是强制数据出口策略；被选中 cloud route 时，调用内容可能离开本机。必须按请求/项目给出可验证的路由和数据处理证明。 |
| Trajectory / 调试日志 | `52_superclaw_llm_logger.ts` 将 parent/subagent 的 LLM 输入、输出和 tool call 汇总到 parent session；默认 raw prompt/response 受 `SUPERCLAW_DEBUG_OBSERVABILITY=1` 控制。 | 调试 agent delegation 与按 session 回放。 | 是调试可观测性，不是安全审计。开启 raw 日志会扩大 PII/secret 留存面；没有安全专用留存、访问、完整性或规则命中语义。 |
| 工作区审计 | `sandbox/server/src/audit.ts::recordAudit` 写 `%HOME%/.openwork/openwork-server/audit/<workspace>.jsonl`。 | 记录部分 workspace actor/action/target/summary。 | JSONL 无签名、无防篡改、无内置轮转/保留，依赖 OS 文件权限；不能单独作为完整法证审计。 |
| 事件与遥测 | OTLP metrics/logs、崩溃 scrub、内存 file-session/reload event store。 | 运维指标、崩溃与短期文件/重载事件。 | OTLP 未启用则没有集中留存；内存事件有容量/TTL 且重启丢失；不保证覆盖所有 Agent action。 |

## 3. 对已有三项认知的修正

1. **Trajectory 可 trace back 所有 action：不成立。** 当前 session/LLM/tool 日志对调试很有用，但不能证明完整覆盖工具、MCP、网络、审批、文件、memory、最终 channel delivery，也不具备防篡改与安全审计留存保证。
2. **Workspace 和 sandbox 已解决隔离：只部分成立。** 它们显著降低对 Windows host 的直接风险；但 workspace mount、容器内 bash、MCP、网络出口和 cloud 模型仍可携带数据或产生副作用。
3. **Shell 运行保护已覆盖命令安全：只部分成立。** 当前主要有 agent 路由、确认、重复命令和输出尺寸 guard。尚未看到统一覆盖危险命令、编码混淆、下载执行、删除/覆盖、命令→网络外发链的 policy engine。

## 4. 现有能力缺少的安全闭环

| 优先级 | 缺口 | 为什么当前能力不足 | 建议安全能力 |
|---|---|---|---|
| P0 | Shell 与高危工具策略 | 现有重复调用、输出大小、agent 路由和特定确认 guard 主要保护可用性或业务流程，不等于命令安全。 | 在统一 pre-tool 决策中覆盖危险 shell、删除/覆盖、下载执行、编码/混淆、受保护路径、命令→网络外发链；block/approval 必须发生在副作用前。 |
| P0 | 统一 pre-tool 决策 | 多个 plugin 各管一个场景，审批/PII/guard 没有共同语义。 | 将 tool、MCP、文件、网络请求归一为 `SecurityAction`，决定 `allow/observe/redact/require_approval/block`；复用 OpenWork approval。 |
| P0 | 工具前后安全审计与审批关联 | Debug trajectory、OTLP 和 workspace JSONL 不能证明每个高风险动作在执行前被审查、执行后有结果，也不能关联审批。 | 以同一 `trajectoryId/sessionId/toolCallId/approvalId` 写 `tool.call.pre`、`approval.requested/decided`、`tool.call.post`；pre 必须先于副作用，post 覆盖成功/失败/拒绝/取消/超时。默认不保存原始内容或 secret。 |
| P0 | 网络出口、SSRF 与外发控制 | sandbox 和本地模型不限制容器可访问目标；工具可成为数据出口。 | 私网/metadata deny、域名/协议 allowlist、egress 控制、secret/PII 外发检测；网络层与 tool policy 双层执行。 |
| P0 | 最终输出 DLP | 输入掩码不保证模型回复、artifact、Slack/Telegram 输出不含 secret/PII。 | 在 OpenWork response/artifact/channel delivery 最后边界统一 redact/block，并记录红动作计。 |
| P1 | 用户输入 Prompt Injection / Jailbreak 风险 | 当前 intent classifier 用于路由，不能证明它检测或审计越狱、角色伪装、工具诱导、secret/exfiltration 请求。 | 在模型输入前产生风险标签、置信度、规则 ID 和审计事件；默认 `observe`，不得仅凭分类结果授权工具，也不应仅凭普通文本命中直接封禁正常对话。 |
| P1 | 工具结果的间接 Prompt Injection | Web、MCP、文档等外部结果会进入模型上下文，当前没有通用不可信标记和扫描闭环。 | 记录 `tool.result.scanned`，标记外部结果不可信并检测 prompt injection/role impersonation/敏感数据；先 `observe`。只有“高置信度注入 + P0 高危工具动作”的组合才选择性收紧阻断。 |
| P1 | Memory write 控制 | 当前文件保护不等于所有持久记忆写入有来源和策略。 | 对 memory write 施加来源、敏感内容、指令污染和审批策略。 |
| P1 | plugin/skill 策略自保护 | Agent 若能修改插件、manifest 或安全配置，可绕过 guard。 | 将安全资源纳入 capability projection/hash gate，保护路径与启动完整性校验。 |
| P2 | 统一、脱敏、可关联审计 | trajectory、JSONL、OTLP、PII registry 的事件模型分散。 | 结构化 audit event：session/project/correlation、policy digest、rule ID、decision、approval reference；默认不保存原文和 secret。 |
| P2 | 审计完整性与留存 | 本地 JSONL 可被同权限修改，内存事件会丢失。 | 受限 ACL、append-only/签名或远端不可变存储、保留期和访问审计。 |
| P2 | 安全验证矩阵 | 现有测试不能证明阻断时副作用未发生，也不能证明每 grade 生效。 | 为 shell/file/network/MCP/output/memory 写边界建立 negative integration tests 与 capability-grade projection tests。 |

## 5. Trajectory 的安全审计闭环：现状、目标与验收

当前 trajectory、OTLP telemetry 和 workspace JSONL audit 是三类不同记录，不能相互替代：

| 记录 | 当前用途 | 能否作为安全证据 |
|---|---|---|
| Debug trajectory | 排障和回放 agent 编排。`52_superclaw_llm_logger.ts` 可按 parent session 汇总 LLM/tool 调试信息。 | 否。开启 raw 内容日志会扩大敏感数据留存面。 |
| Operational telemetry | 健康、延迟、错误率、token/tool 计数和 crash。 | 否。它不能回答一次危险操作为何被允许或拒绝。 |
| Security audit | 当前仅有部分 workspace JSONL action 记录。 | 尚不足够；需要最小化、结构化、可关联的安全证据流。 |

对每个用户请求，安全审计最终必须能回答：谁/哪个项目发起、意图分类给出何种风险信号、哪个工具动作被提议、策略为何 allow/block/require approval、工具是否执行及安全结果摘要、工具结果是否不可信、最终回复/artifact/channel 是否经过脱敏或被阻止。它不保存模型思维链，也不保存原始 prompt、工具参数、工具输出、文件内容、secret 或完整 PII。

### 5.1 当前是否已实现

| 生命周期环节 | 当前状态 | 缺口 |
|---|---|---|
| `intent.classified` | intent classifier 进程存在。 | 未形成可证明与 session/trajectory 关联、持久化、可审阅的安全 audit event；分类只能是 risk signal，不能直接授权。 |
| `tool.call.pre` | 多个插件有 `tool.execute.before` guard。 | 未统一记录 tool、风险、规则、策略决定和审批关联；必须在副作用前写入。 |
| `approval.requested/decided` | `ApprovalService` 可处理 allow/deny。 | 需要关联 `trajectoryId`、`toolCallId`、`approvalId` 和安全摘要；目前 pending approval 仅内存保存。 |
| `tool.call.post` | 调试 logger/telemetry 记录部分 completed/failed 信号。 | 未保证每次成功、失败、拒绝、取消、超时都有与 pre 对应的安全事件及安全结果摘要。 |
| `tool.result.scanned` | 未证实完整通用实现。 | 需标记外部 tool/MCP 结果是否不可信或含敏感数据，且不保存原始结果。 |
| `output.sanitized/delivery.completed` | 未证实最终回复、artifact、Slack/Telegram 有统一安全审计。 | 必须在最终出口记录 DLP/redaction/block 决策。 |

### 5.2 最小事件契约

新增共享、无 I/O 的 `SecurityAuditEvent` schema（建议源文件为 `shared/contracts/src/security-audit.ts`，并沿用现有 Zod → JSON Schema → TypeScript/Pydantic 生成流程）。每个事件至少包含：

```text
schemaVersion, eventId, timestampUtc, sourceComponent, eventType,
sessionId, trajectoryId, projectId?, toolCallId?, approvalId?, actorPseudonym?,
riskSignals(intentLabel?, intentConfidence?, ruleIds?, piiDetected?, untrustedToolResult?),
authorization(decision?, reasonCode?),
outcome(status?, errorCategory?, affectedResourceKinds?, redactionCount?),
privacy(containsRawContent=false, containsSecret=false, contentReference?)
```

允许的 `eventType`：`session.start`、`session.end`、`intent.classified`、`tool.call.pre`、`approval.requested`、`approval.decided`、`tool.call.post`、`tool.result.scanned`、`output.sanitized`、`delivery.completed`、`audit.recovery`。

关联规则：`trajectoryId` 在用户请求开始时创建；委派子 agent 保留 root `trajectoryId`；每个工具调用的 pre/post/scan 使用同一 `toolCallId`；审批 requested/decided 使用同一 `approvalId`。允许执行的工具必须先有 pre；每一次尝试都必须有 post，包括失败、拒绝、取消、超时。崩溃导致的孤立 pre 在重启后写 `audit.recovery(status=unknown)`，不得伪造成功。

### 5.3 文件落点和验收标准

| 责任 | 文件落点 | 可验收结果 |
|---|---|---|
| 共享契约 | `shared/contracts/src/security-audit.ts`、fixtures、生成产物 | schema 拒绝 raw `prompt`、`args`、`output`、`token`、`secret` 字段。 |
| 持久 audit writer | `sandbox/server/src/audit.ts` | 验证后追加/查询 `SecurityAuditEvent`；不悄悄混入现有 workspace JSONL 行。 |
| OpenCode 工具 producer | `sandbox/agent_config/.opencode/plugins/` | 版本兼容性测试通过后，在工具执行前/后写 redacted pre/post 事件。 |
| 审批关联 | `sandbox/server/src/approvals.ts`、`types.ts` | approval request/decision 具有 `trajectoryId`、`toolCallId`、`approvalId` 与安全摘要。 |
| 意图 producer | `sandbox/intent_classifier/intent_classifier_server.py` | 只记录 label/confidence/source 与 correlation ID；分类成功、失败、fallback 都可审核，且不能单独授权。 |
| 最终输出 | OpenWork response/artifact/channel delivery 的已验证边界 | secret 合成测试会产生 `output.sanitized`；交付 payload 与 audit 都不含 secret。 |

首批验收测试（P0）：schema 拒绝原文/secret；允许工具有同 ID 的 pre→post；被阻断的 destructive 工具不产生副作用；审批拒绝时工具不执行；失败/超时/取消仍有 post；owner 可查询自己的 redacted audit，而非 owner 不可读取其他 workspace。P1：intent 风险信号、tool result 扫描、最终输出 DLP、崩溃恢复和 capability-grade projection。

## 6. 正确定位外部项目

AgentAegis 和 Acacian Aegis 不是现有 SuperClaw 组件。它们只提供实现参考：前者适合借鉴 Agent 生命周期与高风险工具规则；后者适合借鉴策略决策、审计与审批模型。任何接入都必须通过 SuperClaw 的 OpenCode plugin、OpenWork Server、capability projection 和 ServiceHub 合约实现。

下一步设计见 [Agent Security Architecture](2026-08-12_agent_security_architecture.md)。
