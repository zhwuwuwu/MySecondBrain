# SuperClaw Agent Security Architecture

**Status:** Proposed  
**Date:** 2026-08-12  
**Touched contracts:** `Sanitizer`, `AgentRuntime`, `Sandbox`, `Project`, `ServiceHub`  
**Boundary:** OpenCode plugin adapter, OpenWork Server data-plane adapter, ServiceHub-managed Security control plane  
**Sources:** [AgentAegis](https://github.com/antgroup/agent-aegis), [Acacian Aegis](https://github.com/Acacian/aegis)

## 1. Decision

SuperClaw will add a **config-driven agent security layer**. It will use:

- AgentAegis as the reference for lifecycle-oriented runtime protections: dangerous tool execution, command obfuscation, protected assets, memory contamination, tool-result injection, loop detection, and exfiltration chains.
- Acacian Aegis as the reference for a normalized policy decision model, risk levels, approvals, structured audit records, and final-output sanitization.

This is **not** a direct installation of either project. AgentAegis is OpenClaw-specific; Acacian Aegis is Python-only and does not instrument OpenCode. SuperClaw owns its adapters and policy schema, and keeps its existing approval, capability projection, ServiceHub, and sandbox contracts authoritative.

## 2. Goals and non-goals

### Goals

1. Stop or require approval for high-confidence unsafe actions **before side effects**.
2. Treat user input, tool results, memory writes, and model output as separate untrusted boundaries.
3. Enforce a single policy decision and audit trail across desktop, OpenWork HTTP, Slack, and Telegram paths.
4. Ship safely through `off`, `observe`, and `enforce` modes, with conservative rollout.
5. Preserve the current split: chat remains a sandbox data-plane flow; ServiceHub remains control-plane only.

### Non-goals

- Replacing Docker/WSL isolation, host ACLs, outbound firewalling, DNS policy, or model-provider controls.
- Claiming detection of every jailbreak, encoded secret, supply-chain issue, or adversarial prompt.
- Running a second product UI, approval system, or agent runtime.
- Changing the OpenCode or OpenWork upstream APIs merely to match OpenClaw hooks.

## 3. Architecture

```text
                     Windows host (control plane)
Tauri ─────────────► ServiceHub ───────────────► Security Manager
                           │                         │
                           │ policy bundles, health   │ policy/admin/audit queries
                           ▼                         ▼
                    SandboxManager            durable audit storage

                     WSL/Docker sandbox (data plane)
User/channel ─► OpenWork Server ─► OpenCode ─► tools / MCP / filesystem
                  │       │          │
                  │       │          └─ OpenCode Security Plugin Adapter
                  │       │               (pre-tool enforcement; result annotations)
                  │       └─ Response Sanitizer Adapter
                  │          (final reply/artifact redaction)
                  └─ inbound request classification / audit correlation
```

### 3.1 Ownership

| Component | Owner | Responsibility | Must not own |
|---|---|---|---|
| OpenCode Security Plugin Adapter | sandbox agent-config bundle | Normalize tool calls; pre-execution enforcement; attach tool-result trust annotations | Product policy authority, durable audit store, its own approval workflow |
| OpenWork Server Security Adapter | sandbox server | Normalize inbound/outbound messages; call policy; redact final user-visible output; correlate sessions | Host process supervision or capability authority |
| Security Manager | Windows host, managed by ServiceHub | Policy distribution, audit query/export, rule metadata, health | Chat/data-plane proxying and direct tool execution |
| ServiceHub | Windows host | Start/monitor Security Manager; authenticate control-plane requests; aggregate health/events | Domain policy state or chat traffic |
| SandboxManager projection | host/sandbox boundary | Materialize only approved security plugin/config resources for active capability grade | Runtime policy editing outside the committed/profile-governed flow |

The Security Manager uses the existing ServiceHub child-service contract: declared in `service_cfg.json`, fixed configured port, health endpoint, and ServiceHub proxy/auth. It MUST NOT create a second supervisor or a direct Tauri child process.

## 4. Core policy model

Every protected event becomes a normalized `SecurityAction`:

```text
SecurityAction {
  id, timestamp, session_id, parent_session_id, project_id,
  source: user | tool | model | memory | system,
  phase: inbound | prompt | pre_tool | post_tool | memory_write | outbound,
  kind, target, parameters, content_refs, risk_signals, correlation_id
}
```

The policy engine returns exactly one `SecurityDecision`:

```text
allow | observe | redact | require_approval | block
```

Each decision includes `risk` (`low`, `medium`, `high`, `critical`), stable rule IDs, a user-safe reason, and audit metadata. Rules are declarative, versioned, and scoped by capability grade, project, agent, tool, path, network target, and operation. Policy priority is deterministic: explicit project exceptions may narrow access but cannot disable baseline self-protection or critical deny rules.

`require_approval` delegates to the existing OpenWork approval contract. The policy layer never mints a parallel approval token or treats an LLM response as approval.

## 5. Security boundaries and controls

| Capability | Primary enforcement boundary | Secondary boundary | Initial mode | Notes |
|---|---|---|---|---|
| Plugin/config self-protection | capability projection + pre-tool checks | host filesystem/container permissions | enforce | Protect policy bundles, plugin roots, manifest, and security config from agent writes/deletes. |
| User prompt risk | OpenWork inbound adapter | OpenCode prompt context adapter | observe | Detect jailbreak, tool induction, secret/exfiltration requests; never rely on prompt text alone for enforcement. |
| Prompt security context | OpenCode `chat.params` / verified system transform | none | observe | Add compact, provenance-marked constraints; do not expose rule internals. |
| Dangerous commands/files | OpenCode `tool.execute.before` | OpenWork approval | enforce for critical rules | Shell, destructive writes, encoded/obfuscated payloads, download-and-execute, protected paths. |
| Network/SSRF/exfiltration | `tool.execute.before` | sandbox egress policy | enforce for private/metadata destinations | Tool-level policy supplements, never replaces, network controls. |
| Repeat/resource abuse | `tool.execute.before` | OpenCode step/budget controls | enforce | Reuse existing repeat and budget guards rather than duplicate counters. |
| Tool-result injection | verified post-tool hook | mark result untrusted in prompt context | observe | Cannot undo an already completed side effect. |
| Memory contamination | memory-write tool/API before write | protected path policy | observe then enforce | Require source/provenance; reject instruction-like or sensitive persistence. |
| Reply/artifact secret leakage | OpenWork response/artifact boundary | channel delivery adapter | enforce for credential patterns | Final outbound boundary is authoritative; tool-result checks are not sufficient. |

An OpenCode hook is used only after a version-pinned compatibility test proves it exists in the packaged runtime. Current known adapters include `tool.execute.before` and `chat.params`; post-tool and response behavior require a dedicated proof-of-compatibility before becoming a security dependency.

## 6. Event flows

### 6.1 Tool call

1. OpenCode produces a tool request.
2. Plugin adapter normalizes it into `SecurityAction(phase=pre_tool)`.
3. Policy returns a decision.
4. `block` prevents execution; `require_approval` routes through OpenWork approval; `allow`/`observe` continues.
5. The adapter emits a structured audit event without raw credentials or unrestricted prompt/tool payloads.
6. If a verified post-tool hook exists, returned content is scanned, labeled as untrusted, and optionally redacted before it re-enters model context.

### 6.2 User-visible reply

1. OpenWork Server receives the OpenCode response or artifact reference.
2. Response Sanitizer evaluates `SecurityAction(phase=outbound)`.
3. Credential/PII rules redact or block delivery; the audit log records rule IDs and redaction counts, not sensitive values.
4. The same sanitized result is used for desktop and channel delivery.

### 6.3 Policy update

1. Signed/profile-governed configuration selects the security resource bundle for the active capability grade.
2. SandboxManager materializes the bundle under the existing hash gate.
3. Security Manager validates policy schema/version and publishes the active policy digest.
4. OpenCode/OpenWork load only the projected bundle; manual container edits are unsupported and detected as integrity drift.

## 7. Configuration, grades, and defaults

Security is a feature manifest resource, not an untracked plugin. The authoritative sources live with `sandbox/agent_config/feature-artifacts`; generated resources remain under `sandbox/agent_config/.opencode` and are validated by the existing artifact generation/check flow.

Baseline protection is available at `minimal` because a minimal runtime can still execute tools and handle sensitive input. Grade changes may add rules for feature-specific tools, but may not remove baseline self-protection, output redaction, or critical tool deny rules.

Default release posture:

| Rule class | Default |
|---|---|
| Policy absent/invalid, integrity mismatch, unsupported critical adapter | fail closed for the affected privileged operation; report a legible capability/security error |
| Adapter crash after startup | fail closed for destructive/network/secret-bearing operations; observe + error telemetry for low-risk read-only operations |
| Self-protection, protected resource deletion, cloud metadata/private-address SSRF, obvious credentials in outbound output | enforce |
| Prompt injection, tool-result injection, memory contamination, novel exfiltration heuristics | observe first |
| Approved exception | time- and action-bound; audit required; never persists as an LLM-authored rule |

## 8. Audit, privacy, and observability

Audit events use a durable structured store owned by Security Manager. Required fields: event ID, correlation/session/project IDs, policy version/digest, action summary, matched rule IDs, decision, approval reference, timestamps, and redaction counts.

Raw prompts, tool output, command arguments, and secrets are not written by default. Debug payload capture requires an explicit local diagnostic switch, bounded retention, and sanitization before persistence. The existing raw LLM logger must not become the authoritative security audit source.

Security Manager exposes control-plane health, policy digest, aggregate counters, and paginated/redacted audit query/export routes through ServiceHub. It publishes low-cardinality security events to the existing Hub event fan-in. It does not receive user chat payloads through ServiceHub.

## 9. Rollout plan

Migration is **capability-first**, with a hook/boundary matrix maintained for every capability.

| Phase | Deliverable | Exit criteria |
|---|---|---|
| 0 — contract matrix | Map every AgentAegis capability to OpenCode hook, OpenWork boundary, host control, or explicit gap; pin compatible OpenCode version | No capability relies on an assumed hook. |
| 1 — policy/audit spine | `SecurityAction`/`SecurityDecision` schema, policy parser, redacted audit records, policy digest | Unit tests cover precedence, invalid policy, redaction, and correlation. |
| 2 — critical execution guard | Projected OpenCode plugin with protected-path, destructive command, obfuscation, SSRF, and repeat-call checks | Integration tests prove side effects do not occur for blocked actions. |
| 3 — approval and output | Connect `require_approval` to OpenWork; final response/artifact sanitizer | Approval cannot be bypassed by agent text; channel and desktop outputs match. |
| 4 — untrusted context and memory | Input/tool-result observation, security context, memory-write controls | Observe telemetry supports tuned rules with bounded false positives. |
| 5 — selective enforce | Promote measured high-confidence injection/memory/exfiltration rules | Per-rule rollout and backout confirmed on every supported grade. |

## 10. Verification and test strategy

Tests are contract tests at boundaries; they mock external models and Docker/network dependencies in PR CI.

1. **Policy tests:** precedence, scopes, mode changes, schema validation, redaction, deterministic decisions.
2. **OpenCode adapter tests:** blocked bash/write/network calls never execute; allowed calls retain arguments; unsupported hooks are detected at startup.
3. **OpenWork tests:** approval round trip, output/artifact redaction, no raw secret in HTTP/channel payload or audit record.
4. **Projection tests:** each grade materializes exactly the intended plugin/policy resources; tampered digest fails legibly.
5. **ServiceHub contract tests:** Security Manager starts from `service_cfg.json`, becomes ready through health polling, and remains control-plane only.
6. **Adversarial fixtures:** command obfuscation, encoded payloads, prompt-injection tool output, cloud metadata URLs, secret-bearing text, memory poisoning, and repeated calls.

Verification records MUST state the active capability grade from `GET /app/capability-profile`, the policy digest, exact command/test suite, and whether a real sandbox smoke was performed.

## 11. Backout

Per-rule mode permits immediate downgrade from `enforce` to `observe`; profile projection controls activation of optional adapters. Emergency backout is a revert of the policy/resource bundle followed by normal projection/reload. It must not delete audit history, bypass integrity checks, or reintroduce direct container edits.

## 12. Open questions before implementation

1. Which exact packaged OpenCode version and post-tool/system-transform hooks are contractually available?
2. What is the authoritative persistent-memory write path for every agent and channel flow?
3. Does the existing host security service already own a compatible durable store and final text/artifact sanitization API, or should Security Manager extend it?
4. Which egress controls exist below the tool layer, and how will tool policy rules align with them?
5. What retention, export, and local-user privacy policy applies to security audit records?
