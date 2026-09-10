# AgentRail Sentinel 安全项目调研

> 调研对象：`intel-sandbox` 下的 AgentRail 相关仓库。  
> 证据基线：本地同步代码、README、设计文档和 `icase.ai.agent.agentrail.integration` 的 Repo Manifest。  
> 结论中的“已证实”表示在上述材料中有直接证据；没有把同名的第三方 AgentRail、AgentSentinel 或 `agentsh` 项目混入本报告。

## 1. 结论摘要

AgentRail 不是一个单仓库安全产品，而是一组由 manifest 组装的分层系统：

```text
OpenClaw
  ├─ agentrail-sentinel                 Prompt Injection / Intent 插件适配层
  │    └─ sentinel/v1 TCP API
  ├─ agentrail-sentineld                 检测、规则、判决核心
  ├─ ssbx-openclaw-plugin                Secure Sandbox 的 OpenClaw 适配层
  └─ secure-sandbox / agentrail daemon   文件与网络强制执行核心

TEE / 机密材料路径
  ├─ cred-vault                          Vault 服务/适配边界
  ├─ optee_os + optee_client             TEE 基础设施与客户端
  └─ ACRN dice_coco                      Remote DICE / CoCo 相关平台改造
```

作者所说的四项独立工作，与仓库的最佳对应关系如下：

| 工作 | 对应仓库/模块 | 核心还是胶水 | 结论 |
|---|---|---|---|
| Secure Sandbox | `icase.ai.agent.agentrail.secure-sandbox` | 核心实现 | Go CLI/daemon、Docker、FUSE 文件中介、MITM 网络中介、审批 API 和审计 |
| Prompt Injection 防护 | `icase.ai.agent.agentrail.sentineld/src/checks/pi-check.ts` 及模型加载/评估模块 | 核心实现位于 sentineld | Qwen3Guard-0.6B/OpenVINO 检测；OpenClaw 插件只转发 hook 和执行 verdict |
| TEE Vault | `icase.ai.agent.agentrail.cred-vault`，配套 `optee_os`、`optee_client` | Vault 是产品核心；OP-TEE 是基础设施依赖 | `integration/project/dev.xml` 证明这些仓库共同进入系统，但当前 Sentinel 代码本身不实现 Vault |
| Remote DICE Attestation | ACRN `dice_coco` revision，加上 integration/deploy 的 ACRN/Kata 组装 | 平台核心改造 + 集成胶水 | Manifest 明确选择 ACRN `dice_coco`；没有证据证明 Sentinel 或 sentineld 自己实现 DICE 协议 |

最重要的边界是：**`icase.ai.agent.agentrail.sentinel` 不是检测引擎，也不是四项能力的总实现。** 它是 OpenClaw 的 thin plugin；检测和判决在 `agentrail-sentineld`，沙箱强制执行在 `secure-sandbox`，而 TEE/DICE 由其他仓库和部署层提供。

## 2. 仓库谱系与子模块分析

### 2.1 `icase.ai.agent.agentrail.sentinel`

仓库：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentinel>

这是 OpenClaw plugin，README 明确写出 “Thin OpenClaw security plugin”，并明确声明“不拥有 detection logic”。它的主要模块是：

- `src/hooks/`：把 OpenClaw 生命周期 hook 转换成 Sentinel 请求；涉及 `before_agent_run`、`before_agent_finalize`、`after_tool_call`、`before_prompt_build` 等路径。
- `src/client/`：通过 `sentinel/v1` 合约访问 sentineld；REST 用于异步请求，framed-sync/Atomics relay 用于需要同步结果的 hook。
- `src/transcript/`：限制会话上下文的用户轮数、工具调用数、单条/总字节数，减少把无限 transcript 送入判定面。
- `src/verdict/`：将 daemon 返回的 `block`、`approve`、`mutate`、`revise` 等结果映射为 OpenClaw 行为。
- `src/identity/`、`src/contract/`、`src/config/`、`src/logging/`：身份、协议、配置和日志胶水。

**分类：集成胶水层。** 它没有 Prompt Injection 模型、Intent Judge、规则引擎或沙箱执行器。其安全价值在于把 OpenClaw 的 hook 接到可信的本地判决服务，并在 hook 返回点执行判决。

当前代码/README 还显示一个重要限制：TCP 链路当前未认证。因此“本机 daemon + 本机插件”的信任边界成立，但跨容器部署时不能把网络可达性误认为身份认证。

### 2.2 `icase.ai.agent.agentrail.sentineld`

仓库：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentineld>

这是 standalone security daemon。README 明确列出：

- REST/JSON TCP API；
- length-prefixed framed-sync TCP 通道；
- WebSocket audit stream；
- Qwen3Guard-0.6B（OpenVINO，Prompt Injection guard）；
- A3B Intent Judge（通过 OVMS `/v3/chat/completions`）；
- rule engine、Prompt Injection guard 和 intent-alignment judge。

本地源码布局进一步说明职责：

- `src/checks/`：`pi-check`、`intent-check`、sink classifier、check registry；这是四项工作中 Prompt Injection/Intent 的检测核心。
- `src/evaluator/`：路由、规则底线、门控、合并、竞速和失败策略；把多个检查合并成最终 verdict。
- `src/model/`：本地 OpenVINO guard 和外部 OVMS 模型的模型边界。
- `src/api/`、`src/contract/`、`src/state/`：服务协议、状态和 API 胶水/服务化边界。
- `src/rules/`、`src/parsers/`：规则与输入解析。

**分类：Prompt Injection/Intent 防护核心 + 服务化适配层。** `checks`、`evaluator`、模型调用和规则是实际安全功能；REST、framed-sync、配置和审计流是把核心暴露给插件/网关的服务化胶水。

### 2.3 `icase.ai.agent.agentrail.secure-sandbox`

仓库：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.secure-sandbox>

设计文档把它定义为 Linux-only、rootful Docker 的 sandbox CLI/daemon，目标是对不可信或半可信 agent workload 提供两条强制中介路径：

1. **Filesystem mediation**：通过 host-side FUSE 文件系统把指定 host folder 暴露给 sandbox，文件操作进入审批点。
2. **Network mediation**：通过可信 MITM proxy 中介 HTTP/HTTPS egress，网络请求进入审批点。

它提供的实际能力包括：sandbox create/start/inspect/stop/remove、AgentRail-owned exec、文件/网络 approval API、状态与 enforcement readiness、audit events，以及 fail-closed 行为。设计目标还包括 sandbox lifetime 内的 policy cache 和 allow-once/allow-lifetime/deny 决策。

`docs/framework-integration-contract.md` 要求框架不要直接访问 Docker、FUSE 路径、proxy container 或 CA private key，而应调用 AgentRail API/CLI。由此可清楚区分：

- **核心实现**：Go daemon/CLI、Docker 生命周期、FUSE 文件中介、MITM 网络中介、审批和审计。
- **适配边界**：framework-neutral HTTP/CLI contract，以及把框架 sandbox 接口翻译成 AgentRail 调用的插件。

### 2.4 `icase.ai.agent.agentrail.ssbx-openclaw-plugin`

它在 `integration/project/dev.xml` 中以独立 project 出现，路径为 `ssbx-openclaw-plugin`。从命名和 secure-sandbox 的 framework integration contract 看，它的职责是把 OpenClaw 的 sandbox backend/HAL 接口翻译为 AgentRail sandbox API/CLI。

**分类：Secure Sandbox 的 OpenClaw 胶水层，不是 sandbox 核心。** 不应把它与 `agentrail-sentinel` 合并：前者负责隔离执行、文件/网络审批入口；后者负责对 agent prompt、tool call、tool result 和 output 做安全判定。

### 2.5 `icase.ai.agent.agentrail.integration`

仓库：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.integration>

这是 repo manifest 和部署编排仓库，不是主要运行时安全引擎。`project/dev.xml` 明确组装以下项目：

- secure-sandbox；
- dashboard；
- `icase.secure-ai-agent.agent-dev.openclaw`（`agentrail` revision）；
- ssbx-openclaw-plugin；
- OP-TEE OS/client；
- `agentrail.cred-vault`；
- `agentrail.sentinel`（同步路径名 `sentinel-openclaw-plugin`）；
- ACRN hypervisor（`dice_coco` revision）；
- Yuanbao 和 Weixin OpenClaw channel plugins。

`deploy/` 则提供 Docker-in-Docker、Kata QEMU/ACRN runtime、OpenClaw setup/launch、OVMS 启动和端口映射脚本。

**分类：系统集成/部署胶水层。** 它是判断“作者把哪些组件放在一个系统里”的关键证据，但不能据此声称每个子仓库都是 AgentRail Sentinel 的源码实现。

### 2.6 `icase.ai.agent.agentrail.cred-vault`、OP-TEE 和 ACRN

这些仓库在当前 checkout 中不是独立目录，但被 integration manifest 明确引用：

- `icase.ai.agent.agentrail.cred-vault`：TEE Vault 产品/服务边界，预期承载凭据或敏感材料的受保护存取；属于 Vault 能力核心，而不是 OpenClaw hook 胶水。
- `icase.ai.agent.optee.optee_os`：TEE OS 基础设施。
- `icase.ai.agent.optee.optee_client`：TEE client-side 调用基础设施。
- `icssp.virtualization.hypervisors.acrn.acrn-dev.acrn-hypervisor` 的 `dice_coco` revision：ACRN/CoCo 虚拟化侧与 DICE 相关的平台实现边界。

在没有这些子仓库源码或其 README 的情况下，不能把具体密钥协议、证书格式、quote 验证流程或 DICE chain 细节写成已实现事实。当前可以确认的是它们被 manifest 作为同一部署组合的独立 project 引入。

## 3. 支持哪些 Agent Platform

### 已证实支持

1. **OpenClaw**：Sentinel plugin 的 peer dependency 是 `openclaw >= 2026.5.22`；manifest 和 deployment README 也直接配置 OpenClaw gateway、OpenClaw sandbox backend 和 channel plugins。
2. **框架无关的 AgentRail sandbox contract**：secure-sandbox 的 integration contract 明确设计为 framework-neutral。任何能调用 AgentRail API/CLI、桥接 approval prompt 并展示 status/audit 的 agent framework，理论上可以接入。
3. **Manifest 中的 OpenClaw channel 变体**：Yuanbao 和 Weixin 是 OpenClaw channel integration，不应误称为独立的安全判定平台。

### 当前不能确认的内容

- 没有证据表明 Sentinel plugin 已经支持 LangChain、CrewAI、AutoGen、MCP host 等其他框架。
- “framework-neutral”是 sandbox API 的设计目标，不等于这些框架已经有现成 adapter。
- `agentrail-sentineld` 的 API 可以被任意 host/plugin/test harness 调用，但这说明的是调用方形态，不是已经完成的平台集成矩阵。

## 4. 以 OpenClaw 为例的集成方式

### 4.1 Sentinel（Prompt Injection/Intent）路径

```text
OpenClaw hook
  -> agentrail-sentinel plugin
  -> sentinel/v1 REST 或 framed-sync TCP
  -> agentrail-sentineld
       -> rule floor
       -> Prompt Injection guard
       -> Intent judge
       -> evaluator merge/gates/failure policy
  -> block / approve / mutate / revise
  -> plugin 映射回 OpenClaw hook 结果
```

安装和运行边界：

1. 构建 plugin，打包后执行 `openclaw plugins install`。
2. OpenClaw manifest 必须允许 `hooks.allowConversationAccess` 和 `hooks.allowPromptInjection`，否则 conversation hooks 会静默 no-op。
3. 启动 `agentrail-sentineld`，默认提供 REST `47900` 与 framed-sync `47901`。
4. 在 OpenClaw gateway 配置中设置 `plugins.entries.agentrail-sentinel.config`，指定 `frameworkId: openclaw`、`mode` 和 daemon endpoint。
5. plugin 不加载检测模型；所有判定由 sentineld 完成，plugin 只负责上下文边界、传输和 verdict 执行。

### 4.2 Secure Sandbox 路径

```text
OpenClaw sandbox backend / ssbx-openclaw-plugin
  -> AgentRail API 或 agentrail CLI
  -> secure-sandbox daemon
       -> Docker/Kata sandbox
       -> FUSE 文件审批
       -> MITM 网络审批
       -> exec/status/audit
  -> OpenClaw 展示或转发 approval prompt
```

部署 README 显示的实际流程是：构建 AgentRail container，选择 Kata QEMU 或 `kata-acrn` runtime，完成 OpenClaw onboard，启动 gateway，然后让 OpenClaw 使用 `agentrail` backend。这里的 `ssbx-openclaw-plugin` 是 adapter；Docker、FUSE、proxy、审批状态机和 fail-closed 是 secure-sandbox 的核心。

### 4.3 Vault/DICE 路径

OpenClaw 并不直接读取 TEE private key 或 DICE 原始实现。正确的集成边界应是：

- OpenClaw/AgentRail plugin 请求一个受策略保护的凭据或运行证明；
- `cred-vault`/TEE 服务处理敏感材料；
- OP-TEE 和 ACRN/CoCo 负责受保护执行环境及证明基础设施；
- integration/deploy 负责把这些组件装配到同一运行时。

当前材料没有提供足够证据证明 Sentinel plugin 已经调用了 `cred-vault`，因此不能把 Vault/DICE 写成 Sentinel plugin 的已完成运行时依赖。

## 5. Agent 完整生命周期中的安全防护层级

以下是按生命周期整理的“已证实能力”，而不是把未读到实现的能力推测进去：

| 生命周期阶段 | 安全控制 | 实现归属 | 防护级别 |
|---|---|---|---|
| 安装/部署 | manifest 固定版本/分支；Docker/Kata/ACRN 组装；OpenClaw plugin manifest 权限 | integration、deployment、plugin manifest | 供应链与装配层 |
| Agent 启动/会话建立 | session 创建、身份/关联字段、transcript 上限 | sentineld + sentinel plugin | 上下文边界与会话治理 |
| Prompt 构建 | system-context hardening、Prompt Injection 检测 | sentineld PI guard + plugin hook | 输入/上下文防护 |
| Agent 计划与意图 | Intent alignment judge、rule floor、sink classification | sentineld checks/evaluator | 意图与策略防护 |
| Tool call 发出前后 | tool call、tool result、output 的 evaluate；返回 block/approve/mutate/revise | sentineld + sentinel plugin | 工具行为运行时防护 |
| 文件访问 | FUSE host-folder mediation、allow once/lifetime/deny | secure-sandbox | 主机数据访问强制控制 |
| 网络访问 | 强制 HTTP/HTTPS MITM proxy、目标审批、deny/allow policy | secure-sandbox | 外联与数据外传控制 |
| 沙箱内执行 | Docker/Kata 隔离、AgentRail-owned exec、运行态检查、fail closed | secure-sandbox + integration | 执行环境隔离 |
| 凭据使用 | TEE Vault/OP-TEE 组合（manifest 可证实；具体调用链未完全可见） | cred-vault + OP-TEE | 机密材料保护 |
| 平台/启动可信性 | ACRN `dice_coco` 组合（具体 attestation API 未在当前材料中展开） | ACRN + integration | 远程证明基础设施层 |
| 事后审计 | daemon audit stream、sandbox audit events、dashboard 组合 | sentineld、secure-sandbox、dashboard | 可追溯性与运营检测 |

“级别”在这里表示控制所在的安全边界，而不是未经项目定义的认证等级。尤其要区分：Prompt Injection guard 是内容/行为判定；Secure Sandbox 是资源访问强制；TEE/DICE 是运行环境和机密材料信任根。四者不能互相替代。

## 6. 四项独立工作的严格归属

### 6.1 Secure Sandbox

**核心 repo：** `icase.ai.agent.agentrail.secure-sandbox`。  
**适配 repo：** `icase.ai.agent.agentrail.ssbx-openclaw-plugin`、integration/deploy。  
**安全对象：** 文件系统、网络 egress、容器执行和审批状态。  
**不是：** Prompt Injection 模型，也不是 TEE/DICE。

### 6.2 Prompt Injection 防护

**核心 repo：** `icase.ai.agent.agentrail.sentineld`，特别是 `src/checks/pi-check`、模型和 evaluator。  
**适配 repo：** `icase.ai.agent.agentrail.sentinel`。  
**安全对象：** prompt/context、tool call/result、agent output 与 intent alignment。  
**不是：** OpenClaw plugin 自己的检测逻辑；README 明确否认这一点。

### 6.3 TEE Vault

**核心 repo：** `icase.ai.agent.agentrail.cred-vault`；底层依赖为 `optee_os` 与 `optee_client`。  
**适配/装配：** integration manifest、deployment scripts，以及 OpenClaw/AgentRail 上层调用方。  
**安全对象：** 凭据和其他敏感材料的受保护存取。  
**当前未知：** Vault 的具体 TA、API、密钥生命周期和 Sentinel 是否已直接调用。

### 6.4 Remote DICE Attestation

**平台核心候选：** ACRN hypervisor 的 `dice_coco` revision。  
**适配/装配：** integration manifest、`kata-acrn` runtime wrapper、ACRN 配置和启动脚本。  
**安全对象：** 受保护虚拟机/平台的启动测量与远程信任证明。  
**当前未知：** 具体 DICE evidence 格式、verifier、attestation endpoint，以及是否已经由 OpenClaw workflow 消费证明。

## 7. Security Sandbox 详细分析

### 7.1 OpenClaw 原生 sandbox 的现状

**OpenClaw 默认不启用 sandbox。** 官方 `docs/gateway/sandboxing.md` 明确说明：
sandbox 默认关闭，由 `agents.defaults.sandbox`（全局）或 `agents.entries.*.sandbox`
（按 agent）控制；Gateway 进程始终留在宿主机，只有**工具执行**在启用后进入
sandbox。即使开启，进入 sandbox 的也只是工具执行，native plugins、
control-plane RPC 不会进入。

OpenClaw 原生 sandbox 后端包括 `docker`（默认）、`ssh`、`mxc`（策略容器）、
`openshell`。其原生模型是“把工具执行丢进容器 + bind mount 工作区”，官方文档
自己评价为 “not a perfect security boundary”——只是降低爆炸半径的手段。

因此 AgentRail secure-sandbox 的意义是：在 OpenClaw 的 sandbox 抽象之上
**替换执行后端**，并补上原生没有的**审批中介**。

### 7.2 与普通 Docker 的区别

AgentRail 同样基于 Docker 建容器（`image: openclaw-sandbox:bookworm-slim`）。
区别不在容器技术本身，而在 Docker 之上强制增加的审批中介层：

| 能力 | 普通 Docker | AgentRail secure-sandbox |
|---|---|---|
| 容器生命周期 | `docker run` 直接管理 | `agentraild` 统一管理 create/start/stop/remove |
| 宿主机目录共享 | 直接 bind mount，无拦截 | **FUSE 中介**：host 目录经 FUSE daemon 挂载，逐操作审批 |
| 出网 | 默认全部放行 | **Squid + SSL Bump MITM**，所有 HTTP(S) 过可信代理审批 |
| 代理配置 | 靠环境变量约定，可被绕过 | 拓扑强制：sandbox 只连内部网络，物理上只能走代理 |
| 审批 | 无 | allow-once / allow-for-sandbox-lifetime / deny |
| 失败模式 | 配置错误即乱跑 | **fail-closed**：审批端不可达 → 默认拒绝 |
| 执行 | `docker exec` | AgentRail-owned exec，fail-closed |

设计文档 `9.1` 的安全模型原文：

> sandbox 容器是唯一不可信侧，其余（`agentrail`、`agentraild`、FUSE daemon、
> proxy container、approval client）全部可信；网络策略靠 Docker 拓扑和代理
> 位置强制、文件策略靠只挂载 FUSE 视图强制，都不依赖 sandbox 合作。

威胁模型（`8.1`）显式包含：沙箱进程无视环境变量、尝试直连外网、路径穿越/
路径混淆、高频操作探测审批语义、符号链接与竞态。防护全部落在宿主侧。

### 7.3 与 OpenClaw 的集成

集成分两层。

**插件层（`ssbx-openclaw-plugin`）**：注册 OpenClaw sandbox backend id
`agentrail`，把 OpenClaw 的 sandbox 抽象接到 `agentraild` HTTP API（默认
`http://127.0.0.1:47891`）。OpenClaw 侧配置 `backend: "agentrail"` 即切换
后端。approval bridge 依赖 OpenClaw SDK 的 `approval-gateway-runtime` 导出，
官方 `openclaw@2026.5.22` 可能未含，需要安装 patched checkout。

配置示例（来自插件 README）：

```json5
{
  plugins: { entries: { agentrail: {
    enabled: true,
    config: {
      addr: "http://127.0.0.1:47891",        // agentraild API
      image: "openclaw-sandbox:bookworm-slim",
      guestSharedDir: "/host",
      guestWorkdir: "/workspace",
      approvalBridgeEnabled: true            // 把 AgentRail 审批桥进 OpenClaw
    }
  }}},
  agents: { defaults: { sandbox: {
    mode: "all",           // 所有工具执行进 sandbox
    backend: "agentrail",  // 替换默认 docker 后端
    scope: "session",
    workspaceAccess: "rw"
  }}}
}
```

**部署层（`integration/deploy`）**：docker-in-docker 镜像、Kata QEMU/ACRN
runtime、`openclaw onboard`、`openclaw gateway run`，以及端口映射。

数据流：

```text
OpenClaw agent 调用工具
  → ssbx-openclaw-plugin
  → agentraild HTTP API (47891)
  → secure-sandbox daemon
       ├─ Docker/Kata 容器 (untrusted)
       ├─ FUSE 文件审批（读写 host 共享目录的每笔操作）
       └─ Squid MITM 网络审批（每次 HTTP(S) 出网）
  → 审批结果经 approval bridge 回到 OpenClaw plugin approval
```

### 7.4 基础概念：daemon、FUSE daemon、bind mount

- **daemon（守护进程）**：长期后台运行、常驻监听、通过 socket/API 提供服务的
  进程，区别于“敲一条命令跑完就退出”的 CLI。本项目里的 `agentraild`
  （`internal/daemon/`）即为此角色：监听 HTTP API（默认 `47891`），管理
  sandbox 生命周期、FUSE 挂载、代理容器、审批会话与策略缓存，启动后不退，
  等待 CLI/插件/审批客户端调用。文档定位为 “trusted host-side daemon”，
  与 FUSE daemon、proxy container、approval client 共同构成可信侧。
- **FUSE daemon**：FUSE（Filesystem in Userspace）让文件系统实现在**用户态**
  而非内核。FUSE daemon 就是“看起来是目录、背后是自定义逻辑”的虚拟文件系统
  进程；内核把对挂载点的 open/read/write/rename 等操作转发给它，由它决定
  如何响应。本项目的 FUSE daemon 是自研的（Go + `go-fuse/v2`，见
  `internal/fusefs/`），v1 设计明确“没有现成的生产 daemon 匹配所需审批语义”。
  它把用户选定的宿主机目录挂载成中介视图（内部路径
  `/run/agentrail/sandboxes/<id>/fuse`），沙箱内每次文件操作都经过它，批准后
  才真正落到宿主机目录。可理解为“文件操作的中介门卫”。
- **bind mount（绑定挂载）**：把已有目录绑定到另一路径，两边指向同一份数据
  （非复制）。Docker 的 `-v /host:/guest` 即此机制。本项目的关键技巧是：
  **bind mount 进沙箱的不是原始目录，而是 FUSE 中介视图**——用户选
  `/home/user/shared` → 内部挂载 FUSE 视图 → 该视图再 bind mount 进沙箱
  `/host`。这就是“普通 Docker 直挂原始目录”与“AgentRail 直挂有门卫的目录”
  的本质差异。

### 7.5 sandbox 里审核哪些东西

两类、四种操作，来自 v1 设计的 mediated operation set（10.3）：

| 类型 | 具体操作 | 说明 |
|---|---|---|
| 文件读 | `open`（只读）、`readdir` | 打开文件读、列目录 |
| 文件写 | `open`（写标志）、`create`、`truncate` | 创建文件、截断文件 |
| 文件变更 | `rename`、`unlink`、`mkdir`、`rmdir` | 重命名、删除、建/删目录 |
| 网络 | 所有 HTTP/HTTPS 出网 | Squid + SSL Bump MITM，审核 method/path/目标 hostname/TLS 身份 |

**v1 显式拒绝**（不审核直接拒）：`symlink`、`link`、`chmod`、`chown`、
xattr、`mknod`、socket/FIFO/设备文件；宿主目录中已存在的 symlink 也拒绝
（“keep path safety simple, avoid race-prone edge cases”）。

**v1 不中介**：非 HTTP(S) 协议（如原始 TCP 直连）——靠 Docker 网络拓扑
强制走代理；递归目录审批、通配符策略；完整文件系统语义。

### 7.6 审批机制如何实现

**审批流程（以文件为例）**：

```text
沙箱内进程  open("/host/foo", WRITE)
  → 内核转发给 FUSE daemon（用户态）
  → FUSE daemon 生成审批请求
  → agentraild 控制面（GET /v1/approvals 或 ws 流）
  → 审批客户端（CLI / agent 框架 / dashboard）决定
  → POST /v1/approvals/decisions 提交决策
  → FUSE daemon 按决策放行/拒绝，才真正操作宿主目录
```

**审批语义（10.4）——在操作边界审批，不是每个 syscall 审批**：

| 决策 | 文件读写含义 | 变更操作含义 |
|---|---|---|
| allow once | 放行一个文件句柄创建 + 句柄关闭前的全部 I/O；重开需重新审批 | 恰好放行那一次 rename/unlink/mkdir/rmdir |
| allow lifetime | 沙箱生命周期内缓存决策，后续同操作不再打扰 | 同左 |
| deny | 拒绝 | 拒绝 |

**Fail-closed（9.2 + integration contract）**：审批客户端不可达、控制面不可达、
FUSE daemon 无法解析策略、代理无法解析策略、路径规范化失败、TLS 身份检查
失败时，全部**默认拒绝**；超时与缺失审批路径同样 deny by default。

**强制机制的关键（威胁模型 9.1）**：审核不依赖 sandbox 合作——沙箱可以不理会
`HTTP_PROXY` 环境变量（威胁模型显式列出），但它物理上只连内部 Docker 网络，
出去只有代理一条路；它看到的 `/host` 本来就是 FUSE 中介视图，绕不过去。

### 7.7 代码架构与实现方式（源码级）

基于本地仓库实际代码分析，不是文档描述。

**技术栈**：纯 Go（`go.mod`：module `agentrail`，go 1.22），仅两个运行时依赖——
`github.com/hanwen/go-fuse/v2 v2.8.0`（FUSE）与 `gopkg.in/yaml.v3`（配置）。
**没有任何 Docker Go SDK 依赖**。

**三个可执行入口（`cmd/`）**：

| 入口 | 作用 |
|---|---|
| `cmd/agentrail` | 用户 CLI：sandbox 创建/运行/inspect、exec、审批 watch（`approval_watch.go`） |
| `cmd/agentraild` | 守护进程：HTTP 控制面、审批会话、生命周期编排 |
| `cmd/agentrail-squid-helper` | Squid 的审批决策 helper：把每个出网请求转成 agentraild 决策查询（`--endpoint`/`--unix-socket`/`--sandbox-id`，超时默认 5s，支持 unix socket 传输） |

**内部包（`internal/`）职责**：

| 包 | 职责 |
|---|---|
| `docker/` | Docker 操作封装：`client.go`、`seed.go`（workspace 种子注入）、hardening |
| `fusefs/` | FUSE 文件中介：`manager.go`（挂载）、`node.go`（操作拦截）、`authorize.go`（授权）、`identity.go`（操作/密钥）、`path.go`（解析）、`ownership.go` |
| `proxy/` | 网络中介：`squid.go`（Squid 配置生成）、`policy.go`（决策请求/策略）、`normalize.go` |
| `state/` | 审批状态机：`store.go`（sandbox 记录、审批请求、lifetime 缓存、one-shot） |
| `daemon/` | HTTP 控制面：`server.go`（路由、生命周期 handler、清理策略） |
| `api/` | 类型与校验：guest 路径规范化、sandbox id 校验、运行时目录防逃逸 |
| `ca/`、`audit/`、`securityprofile/`、`config/`、`ws/` | MITM CA 管理、审计日志、安全配置、WebSocket 流 |

#### Docker 调用方式：exec CLI，而非 SDK

`internal/docker/client.go` 的核心是 `Runner` 接口（`Run` / `Output` / `Exec`），
默认实现 `ExecRunner` 用 `exec.CommandContext(ctx, "docker", args...)` **直接调用
宿主机 docker 命令行**。`NewClient(nil, true)` 的 dry-run 模式只记录命令不执行，
测试用 `Client.Commands()` 断言生成的 docker 参数（见 `hardening_test.go`）。

容器命名统一：`agentrail-<sandbox-id>-sandbox`（不可信容器）、
`agentrail-<sandbox-id>-proxy`（可信代理容器）、
`agentrail-<sandbox-id>-internal`（内部 Docker 网络）。sandbox 只连内部网络，
外部网络（`bridge`）仅代理容器可连——网络策略由 Docker 拓扑落实。

#### FUSE 文件中介：go-fuse/v2 + LoopbackNode 拦截

`fusefs/manager.go` 用 `fs.Mount(mountPoint, root, ...)` 建立挂载，挂载选项
强制 `noexec`。`fusefs/node.go` 的 `Node` 内嵌 `fs.LoopbackNode`（底层实际文件
系统），并**重写 Lookup/Open/Readdir/Create/Truncate 等操作**：每次操作先
`resolveSelf/resolveChild`（路径解析）+ `auth.Authorize`，通过后才调用
LoopbackNode 落到真实文件系统；失败映射为 errno。这就是"操作边界审批"
的代码落点。

`authorize.go` 定义 `Authorizer` 接口与两个实现：

- `StaticAuthorizer`：固定决策（测试/干跑用），空决策默认 deny。
- `StateAuthorizer`（生产路径）：
  1. `Store.AllowedByLifetime(sandboxID, lifetimeKey)` 命中 → 直接放行；
  2. 否则 `RegisterApproval(req)` 注册审批请求；
  3. `WaitForDecision(ctx, requestID)` 阻塞等审批客户端决策；
  4. `allow_once` 还要 `ConsumeOneShot` 消费一次性令牌；任何失败 → deny。

操作与密钥在 `identity.go` 定义：文件操作类型为 `open_read`/`readdir`/
`open_write`/`create`/`truncate`/`rename`/`unlink`/`mkdir`/`rmdir`；
`LifetimeKey` 形如 `file:rename:<canonical>-><canonical>`、
`file:delete:<canonical>`——lifetime 缓存按操作+规范路径键控。

#### 网络中介：Squid + SSL Bump + helper

`proxy/squid.go` 用 Go template 生成 Squid 配置（默认监听 3128，SSL Bump、
MITM CA 证书/私钥、上游 HTTP 代理、35s 决策超时）。`proxy/policy.go` 定义
`DecisionRequest`（sandbox_id/url/scheme/hostname/port/method/path），
`IdentityKey = network:<scheme>:<host>:<port>:<method>:<path>`，
`LifetimeKey = network:<scheme>:<host>:<port>`（生命周期审批按目标主机键控）。
`agentrail-squid-helper` 是 Squid 外部 helper 进程：Squid 每来一个请求调用它，
它再查 agentraild 的 proxy decision 端点。

#### 审批状态机：state/store.go

`Store` 是内存态（`sync.Mutex` 保护），持有：

- `sandboxes`：sandbox 记录；
- `requests`：审批请求（含 `done chan` + `timer`，决策超时默认 30s）；
- `lifetime`：`map[sandboxID]map[lifetimeKey]Decision` 生命周期缓存；
- `oneShots`：一次性 allow 令牌。

所有变更经过 `CreateSandbox` / `RegisterApproval` / `WaitForDecision` /
`ConsumeOneShot` 等方法，并写审计。审批事件通过 `onRequest` 回调广播到
`ws.Hub`（WebSocket 流），供 CLI/框架/dashboard 消费。

#### 控制面与校验：daemon/server.go + api/types.go

`daemon/server.go` 用标准库 `http.ServeMux` 注册路由：`GET /v1/health`、
`POST/GET /v1/sandboxes`、`GET /v1/sandboxes/{id}`、`/{id}/start|stop|remove`、
`/{id}/exec`、`/{id}/audit`、`GET /v1/approvals`、`/v1/approvals/ws`、
`POST /v1/approvals/decisions`、proxy decision 端点。`NewServer` 组装全部依赖：
docker client、ca manager、audit logger、state store、fuse manager。

`api/types.go` 做输入校验：guest 路径必须绝对、不含 `..`/NUL、非根；sandbox id
只允许 `[A-Za-z0-9._-]`；运行时目录用 `filepath.Rel` 防逃逸出 state root。

#### 代码层面总结

- **自研核心**：审批状态机（state）、FUSE 授权（fusefs）、网络策略（proxy）、
  输入校验（api）——全部是 AgentRail 自己的 Go 代码；
- **Docker 只做执行器**：通过 docker CLI 封装创建容器/网络/exec，并附加
  hardening（`--cap-drop ALL` + 白名单 `--cap-add`、`--pids-limit 512`、
  `no-new-privileges`，禁止 `--privileged`/`--network host`/docker.sock 挂载）；
- **依赖刻意最少**：`go.mod` 只有 go-fuse 与 yaml，便于审计且无 SDK 漂移；
- **可测试性**：`Runner` 接口 + dry-run 模式让 docker 命令、FUSE 授权、
  Squid 配置都能无 Docker 环境单测。

### 7.8 谁负责决策，谁负责执行

**审核逻辑本身与平台无关，接入方式与平台有关**，职责分成两半：

| 角色 | 谁 | 负责什么 | 平台相关吗 |
|---|---|---|---|
| 决策方 | approval client（CLI 人工 / agent 框架 / dashboard） | 消费审批事件（`GET /v1/approvals`、ws 流），提交决策（`POST /v1/approvals/decisions`） | **平台相关**：框架需把自身 approval 概念窄映射为 allow_once / allow_lifetime / deny |
| 执行方 | `agentraild` 控制面 + FUSE daemon + Squid 代理容器 | 把决策落实为实际放行/拒绝，维护 sandbox 生命周期与策略缓存 | **平台无关**：framework-neutral 核心 |
| 桥接层 | `ssbx-openclaw-plugin` | 把 AgentRail 审批事件翻译成 OpenClaw plugin approval，把 OpenClaw 决策翻译回 AgentRail 决策（`approvalBridgeEnabled: true`） | **平台相关**：胶水 |
| 审计关联 | correlation metadata（`framework_id`、`session_key`、`agent_id`、`run_id`、`tool_call_id`） | 把一次审批关联回“哪个框架、哪次会话、哪个 agent 的哪次工具调用” | 中性（contract 明确：不是安全边界，不能当授权依据） |

总结：**“审核什么、如何强制、fail-closed 语义”由 secure-sandbox 决定（平台
无关）**；“谁来审批、审批长什么样、如何带回 agent 流程”由平台侧 plugin 决定
（平台相关）。换平台不需要改审核逻辑，只需换桥接层。

### 7.10 与 SuperClaw 的集成分析

本节回答：AgentRail secure-sandbox 如何与当前目录的 SuperClaw 集成。基于
SuperClaw 本地代码的实际现状（不是设计稿）。

#### SuperClaw 当前沙箱架构（事实基线）

SuperClaw 的沙箱不是 "per-tool 隔离容器"，而是一个**承载完整 backend 的
常驻容器**，管理链条如下：

```text
Windows host
  ServiceHub (19000)
    └─ sandbox_manager (FastAPI, 127.0.0.1:18821)
         ├─ ContainerLifecycle:  WSL2 distro (superclaw-docker) + Docker CE
         ├─ OWT token cache
         └─ SQLite (sbms.db): runs 台账 / config KV
              │  (Docker 调用方式: wsl.exe -d <distro> -- docker <args> 子进程，
              │   无 docker-py、无 TCP socket、无 Docker Desktop 依赖)
              ▼
         WSL2 superclaw-docker distro
              └─ Docker container: superclaw backend
                   (sandbox_port:8787 = OpenWork server)
```

容器内运行的组件（`sandbox/` 目录）：

- `sandbox/server`：**OpenWork server**（filesystem-backed API，端口 8787），
  是 agent 与文件系统的中间层；
- `sandbox/orchestrator`：**OpenWork orchestrator**，host opencode + OpenWork
  server + opencode-router 的编排器；
- `sandbox/channels`：Slack/Telegram 等 channel 桥。

关键事实（来自源码）：

1. **工具执行主体是 opencode**（opencode.jsonc 在 orchestrator 侧重写），
   OpenWork server 通过 `OPENWORK_OPENCODE_BASE_URL` 连它。
2. **已有审批机制**：`sandbox/server/src/approvals.ts` 的 `ApprovalService`，
   支持 `manual`（人工审批 + timeout 拒绝）与 `auto` 两种模式，配置键
   `approval.mode` / `approval.timeoutMs`（默认 30000ms）。
3. **已有 sandbox backend 配置位**：OpenWork server README 明确
   `OPENWORK_SANDBOX_BACKEND`（`docker` | `container` | `none`）——
   即 SuperClaw 已经在 OpenWork server 层预留了 sandbox backend 抽象。
4. **文件访问**：OpenWork server 是 filesystem-backed，workspace 通过
   OpenWork 的 workspace 配置授权（`workspaces[]`、`authorizedRoots`）；
   没有 FUSE 级逐操作中介。
5. **网络**：容器出网默认走 Docker bridge，没有 MITM 审批（安全边界目前
   主要是 `security_manager` 的 PII 脱敏 + `protected_file_guard`，属内容层
   而非执行层）。

#### 集成切入点：三层对位

AgentRail secure-sandbox 的能力与 SuperClaw 现状对位如下：

| 层 | SuperClaw 现状 | AgentRail 能力 | 集成方式 |
|---|---|---|---|
| 容器生命周期 | sandbox_manager / ContainerLifecycle（WSL2 docker） | agentraild 管理 create/start/exec/remove | 选 A 或 B（见下） |
| 文件访问 | OpenWork filesystem-backed + workspace 授权 | FUSE 逐操作审批中介 | 挂 FUSE 视图替换直挂 workspace |
| 网络 | Docker bridge 直出，无审批 | Squid MITM 审批 | 容器只连内部网络 + 强制走代理 |
| 审批决策 | ApprovalService（manual/auto, 30s timeout） | approval API（allow once/lifetime/deny） | 桥接：OpenWork ApprovalService → AgentRail decisions |
| 执行 | opencode 工具执行（容器内） | AgentRail-owned exec（fail-closed） | exec 经 agentraild 而非直接 docker exec |

#### 集成方案 A：容器内 sidecar（推荐路径）

在**现有 superclaw backend 容器内**跑 `agentraild` + FUSE + Squid，把
`agentrail` 作为 OpenWork server 的 sandbox backend：

```text
WSL2 superclaw-docker distro
  └─ Docker container: superclaw backend (:8787)
       ├─ opencode            ← 工具执行主体（现状不变）
       ├─ OpenWork server     ← OPENWORK_SANDBOX_BACKEND=agentrail
       ├─ agentraild          ← 新 sidecar（Go daemon）
       ├─ agentrail-squid-helper
       └─ FUSE mount: /workspace 经 agentrail 中介视图
```

实施步骤：

1. 镜像内加装 agentraild 二进制（Go 单文件，无新依赖）；
2. `OPENWORK_SANDBOX_BACKEND=agentrail` + `OPENWORK_SANDBOX_ADDR=http://127.0.0.1:47891`
   ——利用已有的 backend 抽象位，实现一个"agentrail backend adapter"；
3. OpenWork 的 `ApprovalService` 增加一个对接 agentraild 的决策通道：把
   OpenWork 的文件/命令审批转成 `POST /v1/approvals/decisions`；
4. workspace 挂载改为 agentrail 的 FUSE 视图（`guest_shared_dir=/host`，
   内部 FUSE 挂到 `/run/agentrail/.../fuse` 再 bind 进容器）；
5. 容器网络改为只连内部网络 + Squid MITM 代理（`HTTP(S)_PROXY` 指向代理容器，
   且物理拓扑强制）。

优点：不动 Windows host 的 sandbox_manager 主链路；agentraild 与 opencode
同生命周期；审批 UI 可复用 OpenWork 现有审批流。
风险：FUSE 在 WSL2 内需要内核支持（WSL2 内核已启用 FUSE，需验证）；
Squid MITM 需向容器内注入 CA（与现有 `security_manager` 的证书信任并存）。

#### 集成方案 B：host 侧 adapter（改动最小）

在 Windows host 的 sandbox_manager 中新增一个 agentraild 客户端，替代
`wsl.exe -- docker` 直调：

```text
sandbox_manager (host)
  └─ new: agentrail_adapter → agentraild (host Linux? 或 WSL2 distro 内)
       └─ agentraild → Docker 生命周期 / FUSE / Squid
```

优点：改动集中在 sandbox_manager 一个模块；AgentRail 的 framework-neutral
API 本来就不绑定 OpenClaw。
缺点：sandbox_manager 当前明确 "no docker-py / no TCP socket" 契约，改用
agentraild HTTP 意味着**更换容器管理后端**，属于 v1.2 沙箱契约变更，需
同步更新 `container_env_contract.py` 与 capability snapshot；且 AgentRail
v1 是 Linux-only、rootful Docker，Windows host 侧没有 rootful Docker，实际
仍要落在 WSL2 distro 内——方案 B 事实上只是"把 `wsl docker` 调用换成
`agentraild` API 调用"，收益小于方案 A。

#### 关键约束与风险（必须向 SuperClaw 架构回归）

1. **架构 guardrail**：AGENTS.md 要求任何 Sandbox/ServiceHub/AgentRuntime
   契约改动先读
   `docs/design/2026-06-29_v1.1_refactor_architecture.md` 与
   `docs/design/2026-07-15_v1.2_capability_architecture.md`，并声明触碰的
   契约（Sandbox / ServiceHub）。方案 A/B 都触碰 Sandbox 契约，属于
   **非 leaf 改动**，需要 contract test + 明确 backout（回退 `backend` 配置
   即可）。
2. **审批语义映射**：OpenWork 是二元 allow/deny；AgentRail 是三元
   allow_once / allow_lifetime / deny。需要窄映射，且 OpenWork 侧 timeout 拒绝
   （30s）与 AgentRail fail-closed（超时默认 deny）语义一致，可直接对齐。
3. **不重复造 trunk**：AGENTS.md 明确禁止"新增第二条 agent-dispatch/沙箱
   生命周期 trunk"。集成应作为 sandbox backend adapter（leaf + adapter 路径），
   而不是在 SuperClaw 里另起一套沙箱编排。
4. **feature gating**：任何新能力要在
   `sandbox/agent_config/feature-artifacts/superclaw-agent-manifest.source.json`
   注册最低 grade，并处理 `minimal` 无该能力时的 fail-legibly。
5. **验证基线**：当前能力 grade 取 `GET /app/capability-profile`；集成验证要
   记录 grade，不能只在 edge box 上验证。

#### 结论

**可行，且切入点明确**：SuperClaw 已在 OpenWork server 层预留 sandbox
backend 抽象（`OPENWORK_SANDBOX_BACKEND`）和审批服务（`ApprovalService`），
AgentRail secure-sandbox 的 framework-neutral API（REST + approval + exec）
可以挂上去。推荐**方案 A（容器内 sidecar）**：agentraild 作为 OpenWork
server 的 sandbox backend adapter，复用现有审批 UI 与容器生命周期，把
文件/网络逐操作审批补齐到 SuperClaw 的执行层；方案 B（host 侧替换
sandbox_manager 后端）改动面更大且违反 sandbox_manager 现有 no-docker-py/
no-TCP 契约，仅作为备选。

### 7.11 小结

OpenClaw 原生是“默认无 sandbox、开启后也只是 docker 容器 + bind mount”的
粗隔离；AgentRail 通过替换 sandbox 后端补上**文件级 + 网络级的逐操作审批
中介**，并把审批桥回 OpenClaw 的 approval 流程。隔离基础仍是 Docker/Kata，
真正的差异在宿主侧的强制审批层，不在容器技术本身。

## 8. Sentinel 判定核心与防护效果

本节回答三个问题：判定核心在哪实现、如何实现、是否有 benchmark 或准确率证据。
源码证据来自本地 `icase.ai.agent.agentrail.sentineld` 仓库。

### 8.1 判定核心在 sentineld，不在 sentinel 插件

上一节已确认 `agentrail-sentinel` 只是 OpenClaw 插件（事件翻译 + verdict 执行）。
真正的判定核心在 **`agentrail-sentineld`** 的 `src/` 下三个目录：

```text
src/api/handlers.ts        /evaluate 入口（REST handler）
  → src/evaluator/         Evaluator 编排
       router.ts           Router.select(eventKind) → 按事件类型选检查
       gates.ts            runMandatoryGatesOnce（强制门）
       race.ts             raceBudget（同步预算竞速）
       merge.ts            mergeVerdicts：rank 合并 allow(0) < approve(1) < block(2)
       rule-floor.ts       PI 规则底线（不依赖模型的确定性规则）
       failure.ts          policyFailureRaw / SYNC_TIMEOUT（fail-closed）
  → src/checks/            Check 实现
       registry.ts         CheckRegistry：注册/选择 pi 与 intent 两类检查
       pi-check.ts         PICheck：调 guard 模型做 Prompt Injection 判定
       intent-check.ts     IntentCheck：状态/策略门 + 调 intent judge
  → src/model/            模型后端抽象
       guard-backend.ts    PI guard 接口（OpenVINO Qwen3Guard-0.6B）
       guard-worker.ts     guard 的 worker 线程加载
       ovms-intent-backend.ts  A3B intent judge（OVMS /v3/chat/completions）
       fakes.ts            FakeGuard/FakeIntent（测试种子）
```

关键实现点（源码证据）：

- **检查注册与选择**：`CheckRegistry` 只认 `"pi" | "intent"` 两种检查；`Router.select`
  按事件 kind 从配置的 `routing` 表选要跑的检查——不同事件类型跑不同检查组合。
- **PI 规则底线（rule-floor.ts）**：不依赖模型的确定性规则先行：
  `detectDirectInjection`（直接注入）、`scanToolResultText`（工具结果扫描）、
  `detectCommandObfuscation`（命令混淆）、`inspectEncodedContent`（编码内容）、
  `detectPathTraversal`（路径穿越）、`checkMemoryContent`（记忆投毒）。规则命中
  `block` 直接阻断；`suspect` 降级为标记并放行给模型确认。
- **PICheck**：`guard.screen({text})` 调 Qwen3Guard-0.6B；`block = score >= 1`
  （`hasScore` 分支）或 `sig.injected`。同步路径下 `syncDisabled:false` 时作为
  overlay 任务在同步预算内跑。
- **IntentCheck**：先查会话状态门（`untrustedDataExposed` 污染标记 + sink 分类：
  block-sink 直接 block）、safeTools allow-stop（无污染时白名单工具直接 allow、
  不浪费 judge）、alwaysValidate 强制送审；否则调 intent judge 拿对齐判定。
- **verdict 合并（merge.ts）**：`RANK = {allow:0, approve:1, block:2}`，取最严格；
  `reason` 只列“决定性检查”的 tag（rule）→ 模型名 → source，纯 allow 不归因。
- **fail-closed（failure.ts）**：策略失败/同步超时产出 `policyFailureRaw` 与
  `SYNC_TIMEOUT` 的 block 判决；mandatory gates 兜底。

### 8.2 是否绑定到某个 agent platform

**检测/判决核心完全不绑定**，插件层绑定 OpenClaw：

| 层 | 绑定情况 | 说明 |
|---|---|---|
| `sentineld`（判定核心） | **平台无关** | 纯本地服务，`sentinel/v1` REST/同步 API，任何 host/插件/测试可调用（sentineld README 原话）；`frameworkId` 只是审计关联字段 |
| `sentinel` 插件（胶水） | **绑定 OpenClaw** | `peerDependencies: openclaw >= 2026.5.22`；hook 名是 OpenClaw 事件名；manifest 要求 `allowConversationAccess`/`allowPromptInjection` |
| 换平台成本 | 只重写翻译层 | 新平台事件 → 统一 `Event`（user-input/tool-call/tool-result/assistant-message）+ verdict → 新平台动作；判定逻辑零改动 |

### 8.3 防护效果与 benchmark 证据

**结论：仓库内没有任何 benchmark、准确率、precision/recall/F1 或模型评估数据。**

证据：

- 全仓（`sentineld` + `sentinel`）grep `benchmark`/`accuracy`/`precision`/`recall`/
  `f1`/`harness`/`score` 文件名与内容，无命中。
- README 只给出**阈值配置**（`blockThreshold: 0.8`、`maxNewTokens`、`syncDisabled`），
  没有任何“该阈值下准确率多少”的说明。
- 唯一接近“效果验证”的是 `tests/parity/eval-parity.test.ts`：**20 个语义 fixture**
  的判决正确性测试（`direct injection`、`indirect injection in tool-result`、
  `obfuscation`、`path traversal`、`tainted session + risky-sink`、
  `ungrounded final-answer`、`judge misaligned high/low-confidence`、
  `rate-limit`、`tool-loop`、`memory-guard`、`smart-approve` 等）。
  但它是**单元级行为测试**：用 `FakeGuardBackend`/`FakeIntentBackend` **种子模拟**
  模型输出，验证的是“给定模型结果时，判决管线是否正确合并”，**不是真实模型
  的准确率**。
- Qwen3Guard-0.6B 与 A3B 本身的公开准确率需查模型卡/论文，仓库未提供。

**推论**：该项目目前**没有可引用的防护准确率数字**。“0.8 阈值”是配置参数，
不是经过评估的校准点。若需要效果声明，须自建评估集（对 guard 模型跑注入样本
算 TPR/FPR，对 judge 跑对齐样本），当前仓库不包含该工具链。

## 9. 不应混淆的相似项目

调研中遇到的第三方 `AgentRail`、`AgentSentinel`、`agentsh secure-sandbox` 等项目不属于本报告的 Intel `intel-sandbox` 代码线。命名相似不能作为同一项目的证据。本报告只把以下关系视为 AgentRail 关系：

1. 仓库 remote 指向 `https://github.com/intel-sandbox/`；或
2. `icase.ai.agent.agentrail.integration` manifest 明确列出该 repo；或
3. 本地仓库 README/设计文档明确以 AgentRail 组件身份描述它。

## 10. 关键风险与验证建议

1. **Sentinel TCP 未认证**：跨容器配置时应增加本地 socket、mTLS 或其他明确的 peer authentication；不能只依赖 `host` 配置。
2. **Fail-closed 语义要端到端验证**：daemon 不可达、模型超时、同步预算耗尽时，应验证 OpenClaw 是否真的阻止了敏感动作，而不是只记录日志。
3. **Manifest 组合不等于功能已完成**：cred-vault 和 DICE 目前由 manifest 证明“被组装”，还需要读取对应 repo 的实现和集成测试才能确认生产可用性。
4. **OpenClaw 支持不应外推到其他平台**：sentinel plugin 是 OpenClaw-specific；其他框架需要各自 adapter，至少要实现 hook/生命周期映射、approval bridge、status/audit 展示和身份关联。
5. **验证时区分两个 plugin**：`agentrail-sentinel` 验证内容/行为判定；`ssbx-openclaw-plugin` 验证 sandbox 创建、exec、文件审批和网络审批。二者通过不同边界接入，不能用一套测试替代。
6. **防护效果缺少量化证据**：8.3 确认仓库无 benchmark/准确率数据。若要对客户/评审声明“PI 拦截率 99%”之类的效果，必须先自建评估集与评估工具链，当前无法从仓库导出任何准确率数字。

## 11. 证据索引

- Sentinel README：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentinel/blob/main/README.md>
- Sentinel manifest：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentinel/blob/main/openclaw.plugin.json>
- Sentineld README：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.sentineld/blob/main/README.md>
- Sentineld 判定管线源码：`src/evaluator/`（evaluator.ts、router.ts、merge.ts、gates.ts、race.ts、failure.ts、rule-floor.ts）、`src/checks/`（registry.ts、pi-check.ts、intent-check.ts、sink-classifier.ts）、`src/model/`（guard-backend.ts、guard-worker.ts、ovms-intent-backend.ts）
- Sentineld 行为验证：`tests/parity/eval-parity.test.ts`（20 fixture，FakeGuard/FakeIntent 种子）
- Secure Sandbox design：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.secure-sandbox/blob/main/docs/v1-design.md>
- Framework integration contract：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.secure-sandbox/blob/main/docs/framework-integration-contract.md>
- OpenClaw HAL plan：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.secure-sandbox/blob/main/docs/openclaw-hal-mvp-patch-plan.md>
- Integration manifest：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.integration/blob/main/project/dev.xml>
- Integration deployment README：<https://github.com/intel-sandbox/icase.ai.agent.agentrail.integration/blob/main/deploy/README.md>

## 12. 概念附录：TEE Vault、DICE、ACRN 与 OpenClaw 的关系

本节补充概念性解释，帮助读者理解第 5、6 节中出现的 TEE Vault、DICE、ACRN
三个术语，并说明它们与 OpenClaw 的集成边界。概念部分为通用知识；项目相关
结论均以上文“已证实”标准为准。

### 11.1 TEE Vault（受信执行环境保险库）

**通用概念**：TEE（Trusted Execution Environment）是 CPU/固件提供的一块与
主操作系统隔离的执行区域，即使 OS 或 hypervisor 被攻破，区域内代码和数据仍
保持机密性与完整性。Intel 侧典型实现是 SGX/TDX，ARM 侧是 TrustZone；本项目
使用的是 **OP-TEE**（开源 TEE OS）。

Vault（保险库）把敏感材料——API key、证书、私钥、模型权重、token——放在该
受保护区域内存取。普通进程看不到 vault 内容，只有经过 TA（Trusted
Application）授权的调用才能读取。

**在本项目中的证据**：

- 仓库 `icase.ai.agent.agentrail.cred-vault`、`icase.ai.agent.optee.optee_os`、
  `optee_client` 由 manifest 组装进同一系统。
- 定位：**机密材料保护层**。`cred-vault` 是产品功能，OP-TEE 是它的基础设施
  依赖。
- 当前状态：manifest 证明“被组装”；具体 TA、密钥生命周期及被哪些上层调用，
  本地材料未展开（同 6.3）。

### 11.2 DICE（Device Identifier Composition Engine，设备标识符组合引擎）

**通用概念**：TCG 标准中一种**启动时信任根**方案。芯片上有一个不可变的 UDS
（Unique Device Secret），系统每启动一层固件，就用
`HASH(层i固件 || 上一层测量)` 派生一个**仅该层可见的新秘密**，同时形成一条
不可伪造的测量链（boot chain）。

核心价值：

- **每层唯一密钥**：攻击者即使劫持上层，也拿不到固件层 UDS，无法伪造“我是
  这台可信设备”。
- **远程证明**：设备把测量链 evidence 发给远程 verifier，verifier 可确认
  “该 workload 运行在我认可的固件/OS 栈上”。
- 配合 CoCo（Confidential Containers），即“容器内敏感负载 + 可远程验证的可信
  启动”。

**在本项目中的证据**：manifest 中 ACRN hypervisor 使用名为 **`dice_coco`** 的
revision，对应 Remote DICE Attestation 的平台侧实现分支。定位是**平台/启动
信任根**，证明的是“这台 VM 的可信状态”，不是内容层面的防护。

### 11.3 ACRN（Intel 开源 Hypervisor）

**通用概念**：Intel 开源的**类型 1 hypervisor**（直接跑在硬件上、不依赖宿主
OS），面向嵌入式/IoT 场景，轻量、低延迟、支持安全关键型分区。在 Kata 容器
体系中可作为 Kata 的 hypervisor 后端：容器不跑在共享内核的 Docker 中，而是
**每个容器一个轻量 VM**，隔离更强。

**在本项目中的证据**（来自 integration/deploy 的
`configuration-acrn.toml` 与 `kata-acrn-dm-cmdline.sh`）：

- Kata 配置含 `[hypervisor.acrn]`，通过 `kata-acrn-dm-cmdline.sh` 包装真正的
  `acrn-dm` 启动参数。
- 部署 README 列出两个 Kata runtime：`kata-runtime`（QEMU）与 `kata-acrn`
  （ACRN-DM 定制集成）。
- 定位：**sandbox 执行隔离层**。secure-sandbox 创建容器时，真正隔离能力一部分
  来自 ACRN VM 边界；DICE 测量链建立在 ACRN `dice_coco` 分支上。

### 11.4 与 OpenClaw 的关系

OpenClaw 是 **agent 框架/网关**，本身不实现这些安全原语。关系是“**通过适配层
接入，而不是混在一起**”：

```text
OpenClaw 网关
 ├─ agentrail-sentinel（插件）        → 调用 sentineld（PI/Intent 检测）
 ├─ ssbx-openclaw-plugin（插件）      → 调用 secure-sandbox（文件/网络强制 + ACRN VM 隔离）
 └─ （未来/上层）cred-vault           → TEE 保险库 + DICE 证明
```

| 组件 | 与 OpenClaw 的关系 | 作用 |
|---|---|---|
| **TEE Vault** | 不直接进 hook 链；是凭据供应方 | 给 agent 的敏感材料提供受保护存取 |
| **DICE/ACRN** | 透明；OpenClaw 只看到 sandbox API | 提供“VM 隔离 + 可信启动证明”的底层能力 |
| **OpenClaw 插件** | 胶水 | 把 hook 翻译成 sentinel/sandbox 调用，执行 verdict |

关键点：**OpenClaw 不需要理解 TEE 或 ACRN**。它的安全边界由插件与 daemon 在
背后落实；TEE/DICE 是更底层的信任基础设施，与 OpenClaw 是“间接、通过 AgentRail
架构桥接”的关系。

### 11.5 信任分层小结

```text
① 内容层    prompt injection / intent 判断   ← sentineld
② 行为层    tool call / 文件 / 网络审批       ← secure-sandbox
③ 执行层    VM 隔离（QEMU / ACRN）           ← Kata + ACRN
④ 信任根    可信启动测量 + 远程证明（DICE）    ← ACRN dice_coco
⑤ 机密层    敏感材料受保护存取                 ← cred-vault + OP-TEE
```

OpenClaw 处于 ①② 的调用方（通过插件），③④⑤ 对它透明。这也再次印证
第 1 节结论：**Sentinel 只是胶水层，五层安全能力分别落在 sentineld /
secure-sandbox / ACRN / cred-vault 上**。
