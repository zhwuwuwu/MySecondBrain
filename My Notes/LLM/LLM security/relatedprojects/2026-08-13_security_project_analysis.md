# Agent Security External Project Analysis

**Status:** Research note — not an adoption decision  
**Date:** 2026-08-13  
**Scope:** Compare external security projects against SuperClaw's OpenCode/OpenWork runtime.  
**Related:** [SuperClaw Agent Security Architecture](2026-08-12_agent_security_architecture.md), [Current Security State (Chinese)](2026-08-12_superclaw_security_current_state_zh.md)

## 1. Executive summary

The three projects address different layers and should not be treated as interchangeable libraries:

| Project | Primary layer | Form | Direct integration assessment |
| --- | --- | --- | --- |
| [AgentAegis](https://github.com/antgroup/agent-aegis) | Agent lifecycle and tool execution | OpenClaw TypeScript plugin | Do not install directly. Adapt selected rules to OpenCode hooks. |
| [Acacian Aegis](https://github.com/Acacian/aegis) | Policy, approval, guardrails, audit | Python multi-framework SDK | Do not embed directly in the Node runtime. Use its policy and audit design as reference, or expose a separately owned adapter only when justified. |
| [AgentRail Integration](https://github.com/intel-sandbox/icase.ai.agent.agentrail.integration/tree/main) | Sandbox, confidential credentials, platform isolation | Multi-repository integration manifest | Do not integrate directly. Treat as a reference architecture for a future hardened execution environment. |

For SuperClaw, the recommended direction remains:

1. Adapt AgentAegis-style execution rules at the OpenCode tool boundary.
2. Use an Acacian-style normalized decision and audit model at SuperClaw-owned boundaries.
3. Keep OpenWork approval, capability projection, ServiceHub, Security Manager, and the existing WSL2/Docker sandbox authoritative.
4. Evaluate AgentRail components only as separately licensed, separately deployable future infrastructure work.

## 2. Evaluation baseline: SuperClaw

SuperClaw is not OpenClaw. It runs OpenCode behind OpenWork Server in a WSL2/Docker sandbox, while ServiceHub and Security Manager run on the Windows host. Existing local OpenCode plugins already use `tool.execute.before` for pre-side-effect guards and `chat.params` for model-call configuration. Product-owned OpenCode resources are capability-profile projected and hash-gated; hand-editing a running container configuration is not a valid integration method.

Existing security controls include scoped OpenWork tokens, approval flows, WSL2/Docker isolation, protected-file confirmation and masking, PII services, and several tool/reliability guards. They are useful but not yet a single policy decision and audit system. Details and limits are recorded in [Current Security State (Chinese)](2026-08-12_superclaw_security_current_state_zh.md).

## 3. Project profiles

### 3.1 Ant Group AgentAegis

**Observed form:** Apache-2.0 TypeScript/Node package, version `2026.3.14`, inspected at commit [`23d59b8`](https://github.com/antgroup/agent-aegis/tree/23d59b8986b86de448d24ff973e662295768d533). It is an OpenClaw plugin, not a framework-neutral security SDK.

**Architecture:** its plugin entry registers twelve OpenClaw lifecycle hooks, then routes each event through handlers, pattern rules, ordered execution strategies, state, and optional asynchronous skill scanning. The implementation is hook-first and intentionally close to agent actions.

| Area | Source-verified behavior | SuperClaw relevance |
| --- | --- | --- |
| Lifecycle | `gateway_start`, message, prompt, dispatch, tool before/after, memory/message write, output, agent/session-end hooks are registered. | Map each required capability to a verified OpenCode hook or an OpenWork Server boundary; do not assume one-to-one hook compatibility. |
| Execution control | Ordered pre-tool strategies cover self-protection, workspace deletion, OpenClaw commands, obfuscation, command blocking, inline execution, memory, script provenance, loops, and exfiltration. | Strong source for `tool.execute.before` policies. |
| Input and tool-result risk | User-risk scans, prompt guard/context, and tool-result injection scanning. | Input needs an OpenWork/OpenCode adapter; result handling must be verified with an OpenCode after-tool PoC and a server-side fallback. |
| Assets and memory | Startup skill scanning, protected resources, memory-write checks, persisted local state. | Useful threat model; paths and state must be made SuperClaw-specific. |
| Output | Output redaction exists. | Final redaction must additionally run at the OpenWork/channel delivery boundary. |
| Modes | Global and per-defense `off`, `observe`, and `enforce`. | Adopt this rollout model, but express it through SuperClaw-owned policy/configuration. |

**Strengths:** concrete tool and shell protections; encoded/obfuscated command handling; loop and exfiltration-chain concepts; a lifecycle-oriented threat model.

**Limits:** OpenClaw-specific runtime API; rule-driven detection cannot guarantee adversarial-input detection; plugin hook failures are fail-open; state is process-local; evidence from the inspected repository showed Vitest configuration but no test files. Its JSONL event storage and standalone UI should not become SuperClaw's parallel audit plane.

**Key sources:** [hook registration](https://github.com/antgroup/agent-aegis/blob/23d59b8986b86de448d24ff973e662295768d533/index.ts#L33-L68), [configuration](https://github.com/antgroup/agent-aegis/blob/23d59b8986b86de448d24ff973e662295768d533/src/config.ts#L81-L119), [execution strategies](https://github.com/antgroup/agent-aegis/blob/23d59b8986b86de448d24ff973e662295768d533/src/security-strategies.ts#L737).

#### 3.1.1 What “cohesive plugin” means here

“Plugin cohesion” is an architectural description, not an OpenClaw API term. AgentAegis keeps lifecycle registration, event handlers, detection rules, ordered defense strategies, skill scanning, runtime state, and defense events inside one OpenClaw plugin boundary:

```text
OpenClaw
  → index.ts: hook registration
  → handlers.ts: lifecycle orchestration
  → rules.ts / security-strategies.ts: detection and ordering
  → scan-service.ts / scan-worker.ts: skill scanning
  → state.ts: runtime/session state
  → defense events: security records
```

This cohesion lets the defenses share session, agent, workspace, tool-call, and security state. It can detect a chain such as “read a secret → contact an external URL → place the secret in the request”, and it keeps pre-side-effect checks close to shell, file, and network execution. The trade-off is runtime coupling: OpenClaw commands, skills, memory paths, session objects, and configuration are embedded in the handlers and rules. SuperClaw should port the threat rules and ordering, but reimplement the lifecycle adapter. AgentAegis's fail-open hook wrapper also must not become SuperClaw's universal failure policy; high-risk actions may require fail-closed behavior.

### 3.2 Acacian Aegis

**Observed form:** MIT-licensed Python package, version `1.0.0`, inspected at commit [`8743c1c`](https://github.com/Acacian/aegis/tree/8743c1c25801856c7d148e20694b944959399fc0). It supports Python agent frameworks through optional integrations and auto-instrumentation; OpenCode is not in its supported framework list.

**Architecture:** framework adapters normalize work into an `Action`; the policy engine assigns a decision; guardrails inspect/transform text; a runtime executes/verifies; audit persists a linked record. This is policy-first rather than hook-first.

| Area | Source-verified behavior | SuperClaw relevance |
| --- | --- | --- |
| Policy | YAML rules with target/type matching and conditions; decisions are `AUTO`, `APPROVE`, or `BLOCK`. | Good reference for a SuperClaw decision contract; do not introduce a second approval system. |
| Guardrails | Prompt injection patterns, PII detection/masking, toxicity, and prompt-leak modules run through a guardrail pipeline. | Useful design for input/output sanitization and policy tests. |
| Audit | SQLite audit records include session and chain identifiers; sensitive parameter keys are redacted. | Preferred reference for structured audit schema over plugin-local JSONL. |
| Integrations | Auto-instrumentation/patching targets Python frameworks and raw OpenAI/Anthropic clients. | Not usable for the Node/OpenCode runtime without a separate adapter/service. |
| MCP | Contains MCP proxy/server components. | Useful reference only; validate SuperClaw's actual MCP control points before adoption. |

**Strengths:** explicit policy, approval and audit model; reusable core concepts; structured observability; mature test claims in repository material.

**Limits:** Python-only; OpenCode unsupported; monkey-patching adapters are framework-version-sensitive; default SQLite does not by itself establish a production multi-process audit design; approval authentication remains an integration responsibility.

**Key sources:** [instrumentation entry](https://github.com/Acacian/aegis/blob/8743c1c25801856c7d148e20694b944959399fc0/src/aegis/instrument/__init__.py#L43-L56), [policy example](https://github.com/Acacian/aegis/blob/8743c1c25801856c7d148e20694b944959399fc0/policy.example.yaml), [audit redaction](https://github.com/Acacian/aegis/blob/8743c1c25801856c7d148e20694b944959399fc0/src/aegis/runtime/audit.py#L25-L68).

#### 3.2.1 How the Acacian layers compose

```text
framework/SDK adapter or explicit call
  → Action: normalize tool, model, file, and network actions
  → GuardrailEngine: inspect/transform text
  → Policy.evaluate(Action): match rules, conditions, and risk
  → AUTO / APPROVE / BLOCK
  → execute / verify
  → Audit: persist session/chain-linked, redacted records
```

- **`core/`** contains framework-neutral governance data and decisions: `Action`, risk levels, policy rules, conditions, approval, plans, and results. Adapters normalize different framework calls into `Action`, so policy code does not need to know whether a request came from LangChain or an SDK.
- **`guardrails/`** is the content inspection layer for prompt injection, PII, toxicity, and prompt leak. `check()` reports findings; `check_and_transform()` may also return masked or transformed text. It answers “is this content unsafe or transformable?”, not the complete authorization question for a file deletion.
- **`instrument/`** adapts full Python frameworks; **`integrations/`** is closer to raw OpenAI/Anthropic clients. These are ingress adapters, not the policy engine itself.
- **`runtime/`** applies the decision: `AUTO` executes, `APPROVE` waits for human approval, and `BLOCK` rejects. Execution, verification, and structured audit are kept as separate responsibilities.
- **MCP proxy/server and HTTP server** are optional boundaries. They can govern MCP traffic without modifying an Agent, but only for traffic that actually crosses the proxy and with additional authentication, latency, and availability concerns.

The key composability property is therefore `adapter → Action → guardrail/policy → runtime decision → audit`, rather than putting every check into one lifecycle hook.

#### 3.2.2 What monkey-patching means

Monkey-patching is replacing or wrapping a third-party function, method, or class at runtime without editing the third-party source. Conceptually:

```python
original = Client.request

def guarded_request(self, prompt):
    decision = check_prompt(prompt)
    if decision.blocked:
        raise SecurityError("blocked")
    return redact(original(self, prompt))

Client.request = guarded_request
```

Application code still calls `Client.request()`, but the call enters the Aegis wrapper first. Acacian uses this approach in `auto_instrument()` for several Python frameworks. It minimizes application changes and is quick to deploy, but it depends on framework internals: an upgrade can make the patch ineffective or silently bypassed, and patching both a high-level framework and its low-level SDK can cause duplicate checks.

This differs from an OpenCode plugin hook. Monkey-patching is the security library actively replacing a called library; a plugin hook is the Agent runtime deliberately invoking an extension under a documented contract. SuperClaw should prefer OpenCode hooks and should not reproduce Acacian's Python instrumentation by runtime-replacing Node/OpenCode internals.

### 3.3 Intel Sandbox AgentRail Integration

**Observed form:** inspected at commit `742d1bb5107c17586f59fa1f5af6d70caab1b44c`. The repository is an Android `repo` manifest/integration repository, not an installable npm or Python package. The inspected manifest repository does not declare a license; every constituent repository must be independently reviewed before reuse.

**Architecture:** the manifest composes a modified OpenClaw distribution and OpenClaw plugins with secure sandbox, Sentinel, dashboard, credential vault, Kata-container/VM isolation, OP-TEE components, and ACRN hypervisor components. It is infrastructure-first and assumes a substantially different deployment substrate from SuperClaw.

| Area | Observed composition | SuperClaw relevance |
| --- | --- | --- |
| Agent layer | Modified OpenClaw plus Sentinel and secure-sandbox OpenClaw plugins. | Not a compatible direct runtime extension for OpenCode. |
| Isolation | Secure-sandbox work, Kata containers, QEMU/ACRN-related components. | Future reference for per-task hardened execution, not a current plugin dependency. |
| Credentials | Credential-vault and OP-TEE-related components. | Future reference for hardware-backed secret boundaries; requires separate platform/product review. |
| Operations | Dashboard and integration deployment assets. | Do not add another SuperClaw control plane or UI. |

**Strengths:** shows a defense-in-depth model where agent controls are complemented by workload isolation and protected credential boundaries.

**Limits:** multi-repository synchronization and deployment complexity; OpenClaw-fork coupling; hardware/virtualization assumptions; no manifest-level license; no evidence from the manifest alone that Sentinel's detailed rules or dashboard controls meet SuperClaw requirements.

**Integration conclusion:** do not run `repo sync` or adopt the AgentRail OpenClaw fork as part of this work. If a future roadmap needs hardened task execution, inspect `sentinel`, `secure-sandbox`, and `cred-vault` as separate dependencies with license, maintenance, threat-model, Windows/WSL compatibility, and operational-cost reviews.

#### 3.3.1 Complete manifest inventory and compatibility

The inventory below is based on [`project/dev.xml`](https://github.com/intel-sandbox/icase.ai.agent.agentrail.integration/blob/742d1bb/project/dev.xml) at `742d1bb`. It declares twelve unique projects. Co-listing in a manifest is not evidence of API, license, trust-boundary, or runtime compatibility. “Manifest-only” entries were not source-accessible during this review.

| # | Manifest project/path | Verified role | Evidence/license | SuperClaw class | Integration path or blocker |
| --- | --- | --- | --- | --- | --- |
| 1 | `agentrail.secure-sandbox`<br>`secure-sandbox` | Go `agentraild`/CLI; execution, file/network approval, audit, sandbox HTTP contract; default port `47891`. | README, `go.mod`, framework contract; no license found. | **Adapter** | Future SandboxManager provider only; map approval/audit to OpenWork/Security Manager. Must not own container or host control-plane lifecycle. |
| 2 | `agentrail.dashboard`<br>`dashboard` | TypeScript UI/API observing agentraild and vaultd; default port `3000`. | README, `package.json`; no license found. | **Adapter / reference-only** | No second UI. At most feed existing Tauri UI through ServiceHub contracts. |
| 3 | `agent-dev.openclaw`<br>`openclaw`, `agentrail` revision | Customized OpenClaw gateway/assistant and plugin SDK. | README, `package.json`; MIT. | **Incompatible** | Replaces the OpenCode/OpenWork runtime path; reference-only for lifecycle/channel ideas. |
| 4 | `ssbx-openclaw-plugin` | OpenClaw sandbox backend calling agentraild and bridging file/network approvals. | README, `package.json`; no license found. | **Adapter (reimplement)** | Cannot load directly; rebuild a minimal OpenCode/OpenWork adapter if secure-sandbox is adopted. |
| 5 | `agentrail.sentinel` | OpenClaw hook forwarder to `agentrail-sentineld` REST `47900`/sync `47901` for prompt-injection/intent threats. | README, `package.json`; no license found. | **Adapter / reference-only** | Verify rules/API/models/license first; map classifications to owned policy, not a parallel security decision plane. |
| 6 | `agentrail.integration` | Current manifest, Docker/Kata configuration, OpenClaw setup scripts. | README, deploy README; no license found. | **Reference-only** | Orchestration example, not an SDK; do not vendor via `repo sync`. |
| 7-8 | `optee_os`, `optee_client` | OP-TEE OS/client projects. | Manifest-only; source was 404; license unknown. | **Reference-only** | Require supported hardware, drivers, deployment, fallback, and Security Manager ownership before reconsideration. |
| 9 | `agentrail.cred-vault` | Credential vault; dashboard indirectly identifies local `vaultd:9470`. | Manifest + dashboard indirect evidence; source/license unknown. | **Adapter / reference-only** | Only behind Security Manager after API, lifecycle, auth, audit, and Windows/WSL validation. |
| 10 | `acrn-hypervisor`, `dice_coco` revision | ACRN hypervisor; Kata-ACRN deployment assets. | Manifest + deployment configuration; source/license unknown. | **Reference-only / incompatible** | Incompatible if it replaces WSL2/Docker or host virtualization assumptions. |
| 11-12 | `yuanbao-openclaw-plugin`, `openclaw-weixin` | Yuanbao and Weixin OpenClaw channel plugins. | Manifest-only / indirect OpenClaw README evidence; source/license unknown. | **Adapter, pending evidence** | Independently assess source, license and protocol; map to `Channel` with existing identity, approval, audit, and absent-agent handling. |

#### 3.3.2 Compatibility gates

| Class | Meaning | Minimum acceptance gate |
| --- | --- | --- |
| **Direct** | Leaf resource with no architecture change. | No new runtime trunk or privileged host access; projection/hash gate and declared minimum grade; no ownership of approvals, credentials, or sandbox. None of the twelve currently qualifies. |
| **Adapter** | Useful capability behind a SuperClaw contract. | Translate at `Capability`, `Sandbox`, `Channel`, `ServiceHub`, or `Security Manager`; retain control-plane ownership, isolated data plane, approval, audit, and policy; contract-test denial, unavailability and malformed input. |
| **Reference-only** | Threat model, protocol, documentation, or test ideas only. | No runtime import or platform dependency; record the deferral reason. |
| **Incompatible** | Replaces a core trunk or weakens a boundary. | Any bypass of projection/hash gate, Security Manager, approval, ServiceHub, SandboxManager, or the OpenCode/OpenWork/WSL2-Docker path fails immediately. |

Relevant SuperClaw constraints: SandboxManager owns WSL2/Docker lifecycle ([README](../../host/sandbox_manager/README.md)); `service_cfg.json` owns fixed ports, proxy prefixes and data-plane declarations ([file](../../host/servicehub/service_cfg.json)); Security Manager owns data protection ([README](../../host/security_manager/README.md)); Profile owns projected OpenCode configuration and lifecycle authority ([capability architecture](2026-07-15_v1.2_capability_architecture.md)); and Tauri starts ServiceHub as the sole sidecar ([as-built diagram](2026-07-21_system_diagram_as_built_v1.2.md)). No independent SuperClaw TEE-vault contract was verified, so a vault must first be designed as a Security Manager-owned interface.

#### 3.3.3 Evaluation order

1. Reuse no code: convert Sentinel threat categories and secure-sandbox approval ideas into SuperClaw-owned policy/tests.
2. Review channel repositories only after obtaining source, license, and protocol evidence.
3. If stronger isolation is required, define a SandboxManager provider contract, capability detection, fail-closed unavailable behavior, and end-to-end approval tests before a secure-sandbox PoC.
4. Define a Security Manager vault interface and Windows/WSL threat model before considering cred-vault/OP-TEE; keep ACRN outside the near-term desktop release scope.

## 4. Comparative matrix

| Dimension | AgentAegis | Acacian Aegis | AgentRail |
| --- | --- | --- | --- |
| Primary abstraction | Lifecycle hook | Policy/action/audit | Platform stack |
| Best capability | Tool execution defense | Governance and observability | Isolation and confidential-computing direction |
| Runtime portability | Low | Medium conceptually, low directly | Low |
| OpenCode direct support | No | No | No |
| Approval model | Security enforcement/confirmation semantics | Explicit policy decision | Not established from manifest-level evidence |
| Audit model | Local defense events | Structured, linked audit | Dashboard exists; detailed event contract unverified |
| Sandbox/TEE | No | No | Yes, through composed projects |
| Best use in SuperClaw | Adapt rule categories | Adapt design contracts | Architecture reference only |

## 5. Recommended SuperClaw target shape

```text
OpenCode plugin adapter
  ├─ input and prompt-context checks
  ├─ before-tool decision: shell/file/network/MCP/memory/self-protection
  └─ after-tool untrusted-result handling

OpenWork Server data-plane adapter
  ├─ final output sanitization before client/channel delivery
  ├─ single approval integration
  └─ structured security event emission

ServiceHub-managed Security Manager
  ├─ policy bundle distribution and health
  ├─ durable audit query/storage
  └─ product configuration; no query hot-path ownership
```

This preserves the current control-plane/data-plane split: chat does not move through ServiceHub, and security decisions on the data path remain local to the sandbox/OpenWork boundary.

## 6. Migration planning rule

Plan by **security capability**, not by mechanically translating source hooks. For every capability, maintain a mapping record containing:

1. threat and expected decision (`allow`, `observe`, `require approval`, `block`, `redact`);
2. pre-side-effect boundary and final-output boundary;
3. OpenCode/OpenWork integration point and any hook gap;
4. owner configuration, capability grade, and projection resource;
5. audit event and redaction requirements;
6. contract and adversarial test cases.

Suggested sequence:

1. protected policy resources and self-protection;
2. high-confidence pre-tool rules: dangerous shell, protected paths, direct defense disablement, SSRF/private targets, obvious secret exfiltration;
3. user input and dynamic security context in `observe` mode;
4. tool-result injection treatment plus final output redaction;
5. memory-write controls and durable correlated audit;
6. only then evaluate sandbox or credential-boundary changes inspired by AgentRail.

## 7. Adoption guardrails

- No direct drop-in installation: all three projects assume runtimes or platform layers SuperClaw does not use.
- No duplicate approval or security UI: extend OpenWork approval and the existing desktop/control-plane surfaces.
- No unmanaged files in the running OpenCode bundle: new security resources must join the capability manifest, generator, projection hash, and grade tests.
- Begin with `observe`; enable `enforce` only for high-confidence controls supported by regression and adversarial tests.
- Treat all external detection claims as bounded heuristics. Keep Docker/WSL isolation, host ACLs, network controls, and model-routing disclosure as independently necessary controls.
- Before importing code: pin the source commit, review dependency and transitive licenses, define update ownership, and add SuperClaw-owned tests for the copied behavior.
