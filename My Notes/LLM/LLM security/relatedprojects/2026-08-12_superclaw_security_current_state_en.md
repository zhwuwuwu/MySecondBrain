# SuperClaw Current Security Capabilities and Gaps

**Status:** Assessment, not an implementation plan  
**Date:** 2026-08-12  
**Subject:** The current SuperClaw product: a Claw-like personal agent assistant built on OpenCode, OpenWork Server, WSL2/Docker sandboxing, and Windows-host services.  
**Correction:** SuperClaw is not OpenClaw. This document does not treat OpenClaw features as existing SuperClaw capabilities. OpenClaw and AgentAegis are external references only.

## 1. Conclusion

SuperClaw already has practical but fragmented security-related controls: data-plane and control-plane token authentication, owner/collaborator/viewer scopes, an approval framework, authorized workspace roots, WSL2/Docker runtime separation, OpenCode pre-tool guards, protected-file masking, text/file PII sanitization, session trajectory, workspace JSONL audit, and telemetry.

These controls reduce accidental actions, unauthorized API access, direct submission of some PII to models, invalid agent routing, and local resource-exhaustion risk. They do **not yet form a unified Agent Security system**: there is no single policy decision point spanning tools, MCP, network access, memory, and final output, and no proof that every security-sensitive action is captured in complete, redacted, durable audit evidence.

## 2. Existing capabilities, evaluated by security effect

| Area | Current capability and evidence | Actual coverage | Assessment and limitation |
|---|---|---|---|
| API identity and authorization | OpenWork tokens have `owner`, `collaborator`, and `viewer` scopes and are stored as hashes; see `sandbox/server/src/tokens.ts`. The proxy restricts some paths by scope. | Limits unauthenticated clients and some viewer write/self-approval actions. | Depends on token-file and OS ACL protection. A local process holding an owner token has owner authority. New sensitive proxy paths must be added explicitly. |
| Control-plane authentication | ServiceHub bearer token protects the control plane. | Prevents unauthorized control-plane actions. | Does not protect the chat data plane. Token disclosure still permits full misuse. |
| Human approval | `ApprovalService.requestApproval/respond` supports `manual` and `auto`; see `sandbox/server/src/approvals.ts`. | Gives connected operations allow/deny/timeout semantics. | Pending requests are an in-memory map and disappear on restart. It is not a mandatory common entry point for all risky tool actions; `auto` permits immediately. |
| Workspace boundary | OpenWork has authorized roots/workspaces; OpenCode has agent/tool permissions; agent runtime is in WSL2/Docker. | Reduces arbitrary service-directory access and direct Windows-host exposure. | OpenCode permissions are not OS isolation. Bash, MCP, mounts, and custom tools must each be proven to obey root policy. Mounted workspace and container network remain attack surfaces. |
| Tool-call guards | Local OpenCode plugins check `tool.execute.before`. `90_superclaw_default_tool_guard.ts` restricts agent/tool routing; `08_protected_pipeline_confirm_guard.ts` blocks unconfirmed dispatch to the protected-file agent. | Stops selected agent/task/protected-file flows before side effects. | Rules are scattered and business-flow-specific, not a unified shell/file/network/MCP/exfiltration policy. They are not a general malicious-command detector. |
| Availability/resource guards | `93_superclaw_repeat_command_guard.ts`, `94_superclaw_read_output_cap.ts`, output-token, step, and lineage plugins. | Limits repeat calls, oversized reads, and context overflow. | Primarily reliability/cost controls, not data-loss, command-execution, or authorization controls. |
| PII file and text protection | Windows-host `security_manager` masks supported files and free text in `none`/`deterministic`/`hybrid`/`llm` modes; see `host/security_manager/README.md`. | Can reduce PII exposure before supported files/text enter a model. | `none` means no protection. Bash/read/MCP/custom-tool paths that do not call the service bypass it. Placeholder-to-original registries retain sensitive mappings during TTL. |
| Protected-file confirmation | The protected pipeline uses a one-shot session confirmation token; see `08_protected_pipeline_confirm_guard.ts`. | Stops unconfirmed `task → protected-file-agent` dispatch. | Covers only that subagent dispatch, not arbitrary sensitive file read/write or exfiltration. |
| Model/data locality | LLM Router supports local and cloud routes; the product supports local models. | A local route can reduce cloud disclosure. | This is not mandatory data-egress policy. A selected cloud route can send call contents off-device; routing and data handling must be provable per request/project. |
| Trajectory/debug logging | `52_superclaw_llm_logger.ts` consolidates parent/subagent LLM input, output, and tool calls; raw prompt/response logging is gated by `SUPERCLAW_DEBUG_OBSERVABILITY=1`. | Supports delegation debugging and session replay. | Debug observability is not security audit. Raw logging expands PII/secret retention and lacks security-specific retention, access, integrity, and rule-decision semantics. |
| Workspace audit | `sandbox/server/src/audit.ts::recordAudit` appends to `%HOME%/.openwork/openwork-server/audit/<workspace>.jsonl`. | Records some workspace actor/action/target/summary events. | JSONL is unsigned, not tamper-evident, and has no built-in rotation/retention; it relies on OS file permissions and is not complete forensic audit alone. |
| Events and telemetry | OTLP metrics/logs, crash scrubbing, and in-memory file-session/reload event stores. | Operational metrics, crashes, short-lived file/reload events. | No central persistence if OTLP is disabled; in-memory events are bounded and lost on restart; coverage of all agent actions is not guaranteed. |

## 3. Corrections to the three prior assumptions

1. **“Trajectory can trace every action” is not established.** Current session/LLM/tool logs help debugging but do not prove complete coverage of tools, MCP, network, approvals, files, memory, or final channel delivery. They also lack tamper evidence and a security-audit retention contract.
2. **“Workspace and sandbox solve isolation” is only partly true.** They materially reduce direct Windows-host risk, but workspace mounts, container bash, MCP, network egress, and cloud-model routes can still move data or cause side effects.
3. **“Shell protection covers command security” is only partly true.** Current controls focus on routing, confirmation, repeat calls, and output size. There is no demonstrated common policy engine for dangerous commands, obfuscation, download-and-execute, destructive writes, or command-to-network-exfiltration chains.

## 4. Missing security closures

| Priority | Gap | Why current controls are insufficient | Needed security capability |
|---|---|---|---|
| P0 | Shell and high-risk tool policy | Current repeat-call, output-size, agent-routing, and specific confirmation guards primarily protect availability or business flows; they are not command security. | Cover dangerous shell, destructive writes/deletes, download-and-execute, encoding/obfuscation, protected paths, and command-to-network-exfiltration chains in the unified pre-tool decision; block/approval must occur before side effects. |
| P0 | Unified pre-tool decision | Plugins each own narrow cases; approval, PII, and guards have no common semantics. | Normalize tools, MCP, file, and network actions as `SecurityAction`; decide `allow/observe/redact/require_approval/block`; reuse OpenWork approval. |
| P0 | Pre/post tool security audit and approval correlation | Debug trajectory, OTLP, and workspace JSONL cannot prove every risky action was reviewed before execution, had an outcome afterward, or was linked to approval. | Emit `tool.call.pre`, `approval.requested/decided`, and `tool.call.post` with shared `trajectoryId/sessionId/toolCallId/approvalId`; pre precedes a side effect and post covers success/failure/denial/cancellation/timeout. Retain neither raw content nor secrets by default. |
| P0 | Network egress, SSRF, and exfiltration control | Sandbox/local models do not constrain reachable destinations; tools can become data exits. | Block private/metadata targets, add protocol/domain allowlists and egress controls, and detect secret/PII exfiltration. Enforce at both tool and network layers. |
| P0 | Final-output DLP | Input masking does not ensure model replies, artifacts, Slack, or Telegram outputs contain no secret/PII. | Apply one redact/block control at the final OpenWork response, artifact, and channel-delivery boundary; audit redaction counts. |
| P1 | User-input prompt-injection / jailbreak risk | The current intent classifier routes work; it is not proven to detect or audit jailbreaks, role impersonation, tool induction, or secret/exfiltration requests. | Produce risk labels, confidence, rule IDs, and audit events before model input; default to `observe`. Classification must not itself authorize a tool, and ordinary text matches must not directly block normal conversation. |
| P1 | Indirect prompt injection from tool results | External web, MCP, and document results enter model context without a proven common untrusted-result marker and scan closure. | Record `tool.result.scanned`; mark external results untrusted and detect prompt injection, role impersonation, and sensitive data; start in `observe`. Tighten blocking selectively only for the combination of high-confidence injection and a P0 high-risk tool action. |
| P1 | Memory-write control | File protection does not give all persistent memory writes provenance or policy. | Apply source, sensitive-content, instruction-pollution, and approval policy to memory writes. |
| P1 | Plugin/skill/policy self-protection | An agent that can change plugins, manifests, or security config can bypass guards. | Put security resources under capability projection/hash gates, protected paths, and startup integrity checks. |
| P2 | Unified, redacted, correlated audit | Trajectory, JSONL, OTLP, and PII registry use separate event models. | Structured audit events with session/project/correlation IDs, policy digest, rule ID, decision, and approval reference; do not retain raw content/secrets by default. |
| P2 | Audit integrity and retention | Local JSONL is modifiable by same-privilege users; in-memory events are lost. | Restricted ACLs, append-only/signature or remote immutable storage, retention policy, and audit access logs. |
| P2 | Security verification matrix | Existing tests do not prove blocked actions have no side effect or that every grade activates controls. | Negative integration tests for shell/file/network/MCP/output/memory boundaries plus capability-grade projection tests. |

## 5. Security-audit closure for trajectory: current state, target, and acceptance

Current trajectory, OTLP telemetry, and workspace JSONL audit are different record types and cannot substitute for each other:

| Record | Current purpose | Security evidence? |
|---|---|---|
| Debug trajectory | Diagnose and replay agent orchestration. `52_superclaw_llm_logger.ts` can consolidate LLM/tool diagnostics by parent session. | No. Enabling raw-content logs expands sensitive-data retention. |
| Operational telemetry | Health, latency, error rate, token/tool counts, and crashes. | No. It cannot explain why a risky action was allowed or denied. |
| Security audit | Only partial workspace JSONL action records currently exist. | Not yet sufficient; a minimal, structured, correlated evidence stream is required. |

For every user request, security audit must ultimately answer: who/which project initiated it, which risk signal intent classification produced, which tool action was proposed, why policy allowed/blocked/required approval, whether the tool ran and its safe outcome summary, whether its result was untrusted, and whether final reply/artifact/channel delivery was sanitized or blocked. It does not store chain of thought, raw prompts, tool arguments, tool output, file contents, secrets, or complete PII.

### 5.1 What is implemented now

| Lifecycle stage | Current state | Gap |
|---|---|---|
| `intent.classified` | The intent-classifier process exists. | There is no proven session/trajectory-correlated, durable, reviewable security audit event. Classification is a risk signal only and cannot authorize an action. |
| `tool.call.pre` | Multiple plugins use `tool.execute.before` guards. | No common record of tool, risk, rule, policy decision, and approval linkage; it must be written before a side effect. |
| `approval.requested/decided` | `ApprovalService` can allow/deny. | It must correlate `trajectoryId`, `toolCallId`, `approvalId`, and a safe action summary; pending approvals are currently memory-only. |
| `tool.call.post` | Debug logger/telemetry capture some completed/failed signals. | No guarantee that every success, failure, denial, cancellation, or timeout has a security event paired with pre and a safe outcome summary. |
| `tool.result.scanned` | No complete general implementation is proven. | External tool/MCP results need untrusted/sensitive marking without retaining raw result content. |
| `output.sanitized/delivery.completed` | No unified security audit is proven for final replies, artifacts, Slack, or Telegram. | The final egress boundary must record DLP/redaction/block decisions. |

### 5.2 Minimum event contract

Add a shared, I/O-free `SecurityAuditEvent` schema (recommended authored file: `shared/contracts/src/security-audit.ts`, using the existing Zod → JSON Schema → TypeScript/Pydantic generation flow). Every event contains at least:

```text
schemaVersion, eventId, timestampUtc, sourceComponent, eventType,
sessionId, trajectoryId, projectId?, toolCallId?, approvalId?, actorPseudonym?,
riskSignals(intentLabel?, intentConfidence?, ruleIds?, piiDetected?, untrustedToolResult?),
authorization(decision?, reasonCode?),
outcome(status?, errorCategory?, affectedResourceKinds?, redactionCount?),
privacy(containsRawContent=false, containsSecret=false, contentReference?)
```

Allowed `eventType` values: `session.start`, `session.end`, `intent.classified`, `tool.call.pre`, `approval.requested`, `approval.decided`, `tool.call.post`, `tool.result.scanned`, `output.sanitized`, `delivery.completed`, and `audit.recovery`.

Correlation rules: create `trajectoryId` at user-request start; delegated child agents retain the root `trajectoryId`; every tool pre/post/scan shares one `toolCallId`; approval requested/decided shares one `approvalId`. An allowed tool must have a pre event before execution; every attempt must have a post event, including failure, denial, cancellation, and timeout. After a crash, an orphaned pre gets `audit.recovery(status=unknown)` on restart; success must never be fabricated.

### 5.3 File landing points and acceptance criteria

| Responsibility | File landing point | Observable acceptance result |
|---|---|---|
| Shared contract | `shared/contracts/src/security-audit.ts`, fixtures, generated outputs | Schema rejects raw `prompt`, `args`, `output`, `token`, and `secret` fields. |
| Durable audit writer | `sandbox/server/src/audit.ts` | Validates then appends/queries `SecurityAuditEvent`; does not silently mix it into existing workspace JSONL lines. |
| OpenCode tool producer | `sandbox/agent_config/.opencode/plugins/` | After version compatibility tests, writes redacted pre/post events around tool execution. |
| Approval correlation | `sandbox/server/src/approvals.ts`, `types.ts` | Approval request/decision contain `trajectoryId`, `toolCallId`, `approvalId`, and a safe summary. |
| Intent producer | `sandbox/intent_classifier/intent_classifier_server.py` | Records only label/confidence/source plus correlation IDs; success, failure, and fallback are reviewable, and classification alone cannot authorize. |
| Final output | Verified OpenWork response/artifact/channel-delivery boundary | A synthetic-secret test creates `output.sanitized`; neither delivery payload nor audit contains the secret. |

Initial acceptance tests (P0): schema rejects raw content/secrets; an allowed tool has same-ID pre→post; a blocked destructive tool has no side effect; a denied approval never executes a tool; failed/timed-out/cancelled calls still have post; an owner can query its redacted audit while a non-owner cannot read another workspace. P1: intent risk signals, tool-result scans, final-output DLP, crash recovery, and capability-grade projection.

## 6. Correct use of external projects

AgentAegis and Acacian Aegis are not current SuperClaw components. They are implementation references only: AgentAegis informs lifecycle and high-risk-tool rules; Acacian Aegis informs policy-decision, audit, and approval models. Any adoption must be implemented through SuperClaw's OpenCode plugins, OpenWork Server, capability projection, and ServiceHub contracts.

See [Agent Security Architecture](2026-08-12_agent_security_architecture.md) for the proposed target design.
