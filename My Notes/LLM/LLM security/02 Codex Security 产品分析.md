# Codex Security 产品分析

> 研究对象：`https://learn.chatgpt.com/docs/security` 当前实际指向的 Codex Security / Security administration 文档体系。
>
> 重要校正：这不是覆盖整个 ChatGPT 产品的通用安全白皮书，也不应被引用为 ChatGPT 已经解决 prompt injection、越狱、训练数据隐私或 Agent memory poisoning。

## 1. 项目简介与开源情况

Codex Security 是 OpenAI 面向代码安全分析推出的工具，目标是让代码 Agent 检查用户拥有或获授权评估的代码仓库，发现、验证并修复安全漏洞。它不是一个保护整台主机或整个 ChatGPT 产品的通用安全平台。

当前公开形态包括：

- **Codex Security CLI**：在终端扫描本地代码仓库、指定路径、已提交 diff 或 working tree 变更；
- **TypeScript SDK**：在其他开发工具或 CI/CD 流程中编程调用；
- **Codex Security 的托管能力**：依赖 OpenAI 的模型和服务，部分网络安全请求或受保护发现可能需要 Trusted Access for Cyber。

CLI 和 TypeScript SDK 的源码位于公开的 [openai/codex-security](https://github.com/openai/codex-security) 仓库，并以 Apache License 2.0 发布。因此，准确说法是：

> **Codex Security 是“开源 CLI/SDK + 闭源模型和托管服务”的组合，不是完全离线、完全自托管的开源安全平台。**

## 2. 产品定位

Codex Security 的核心定位是：**使用 LLM Agent 对业务代码仓库进行安全审查，并对漏洞进行验证、解释和修复辅助**。其分析对象主要是源代码及其开发上下文，而不是 LLM Agent 平台本身。

它更接近以下组合：

- AI 驱动的代码安全审计；
- 漏洞发现、可利用性验证与攻击路径分析；
- 修复建议、补丁生成与修复验证；
- 依赖、安装脚本和软件供应链风险分析；
- 代码 Agent 执行过程中的 sandbox、网络权限、审批和治理。

产品安全边界不能只靠模型拒绝危险操作，而要由运行时隔离、工具权限、审批、身份策略和审计共同实现。

## 3. 主要功能

### 3.1 代码仓库扫描

可以针对以下范围运行扫描：

- 整个 repository；
- 仓库内的指定目录或文件；
- 已提交的 Git diff；
- 当前 working tree 的未提交变更。

扫描结果应包含漏洞证据、影响、严重程度、可达性分析和修复建议，而不是只输出一个静态规则命中列表。

### 3.2 漏洞验证与修复

典型分析流程包括：

1. Threat modeling：识别资产、入口点、信任边界和安全不变量；
2. Finding discovery：发现可能失效的安全控制；
3. Validation：在隔离环境中验证候选漏洞；
4. Attack-path analysis：分析可达性、影响和严重程度；
5. Reporting：生成带证据的漏洞报告；
6. Remediation：提出或协助实施补丁，并重新验证修复结果。

“发现漏洞”不等于“漏洞已经修复”，Agent 生成的补丁仍需要测试和人工 Review。

### 3.3 Agent 执行安全控制

这些控制用于保护扫描过程本身，不应误解为“全面审计 Agent 系统漏洞”：

- 限制 workspace、文件系统、进程和容器边界；
- 控制网络访问、包安装和 network egress；
- 防止凭证和 secrets 不必要地进入模型上下文；
- 对修改、删除、提交、部署和外发操作设置人工审批；
- 记录工具调用、参数、网络请求、文件变更和审批结果。

## 4. 如何使用

### 4.1 本地 CLI 扫描

前置条件是安装受支持版本的 Node.js 和 Python，并拥有 Codex Security 的访问权限。基本流程：

```bash
npm install @openai/codex-security
npx @openai/codex-security login
npx @openai/codex-security scan .
```

也可以指定仓库路径：

```bash
npx @openai/codex-security scan /path/to/repository
```

扫描后应依次完成：查看发现 → 阅读证据和攻击路径 → 检查修复建议 → 运行项目测试 → 人工 Review → 再次扫描或比较扫描结果。

### 4.2 CI/CD 中使用

在 CI 中可以使用 `OPENAI_API_KEY` 或 `CODEX_API_KEY`，避免交互式登录：

```bash
OPENAI_API_KEY=... npx @openai/codex-security scan .
```

生产使用时应将 API Key 放在 CI 的 secret store 中，并限制扫描任务的仓库、网络、文件系统和提交权限。不要让扫描 Agent 默认拥有生产部署权限。

### 4.3 SDK 集成

如果需要把扫描接入内部开发平台，可以使用公开的 TypeScript SDK，针对 repository、path、committed-diff 或 working-tree 创建扫描任务，再将结构化发现接入 Issue、代码 Review 或发布门禁。

## 5. 安全控制面

### 5.1 Sandbox / Runtime

需要限制文件系统范围、进程/容器/宿主机隔离、网络访问、凭证暴露，以及失败后的清理和环境重建。代码 Agent 的 workspace、依赖安装和测试执行都可能产生真实副作用。

### 5.2 Tool / Network / Approval

安全策略应由工具权限和审批策略执行，而不是由 system prompt 单独执行。重点包括：

- 工具 allowlist；
- 展示完整工具参数；
- 分别控制网络请求与包安装；
- 对修改、删除、提交、部署和外发操作要求人工确认；
- 审批绑定正确用户、任务和会话，不能被重试或间接工具调用绕过。

### 5.3 Repository / Supply Chain

仓库内容、issue、README、注释、测试 fixture、恶意依赖、安装脚本和不可信 pull request 都可能携带间接 prompt injection 或供应链风险。应分别判断：

- 仓库内容能否改变 Agent 目标或工具调用；
- Agent 能否读取 secret；
- 生成的修复是否经过测试和人工 review；
- Agent 能否直接推送到生产或受保护分支。

### 5.4 Observability

审计至少要关联仓库/提交、模型版本、上下文来源、工具调用、网络请求、审批、文件变更、测试结果和最终输出。只保存最终自然语言回答，无法重建真实攻击链。

## 6. 能力边界：声明了什么，不能推断什么

### 可以作为证据的内容

- 代码 Agent 应被视为需要隔离和治理的执行系统；
- sandbox、网络权限、审批和组织管理是独立控制面；
- 安全能力要结合代码扫描、漏洞发现和开发工作流；
- 企业部署需要管理员策略、权限管理和审计。

### 不应过度推断的内容

- 不等于对所有 prompt injection 提供形式化保证；
- 不证明所有工具调用都满足最小权限；
- 不自动覆盖长期记忆、跨 Agent 通信、MCP server supply chain 或跨租户 RAG；
- “检测到漏洞”不等于“漏洞已经修复”；
- Agent 生成的修复不必然安全；
- 产品文档中的能力描述不等于独立第三方安全评估。

### 6.1 它检查业务代码，还是检查 Agent 自身？

两者需要分开理解：

```text
Codex Security：使用 LLM Agent → 审查业务代码是否有漏洞
Agent Security Testing：测试 LLM Agent 系统 → 是否越权、逃逸、泄密或被注入
```

Codex Security 可以扫描一个包含 Agent 的代码仓库，例如检查 Agent API、权限校验、工具调用和数据处理代码中的普通软件漏洞；但它不是专门的 Agent Security Testing 平台，不会自动完成以下全面评估：

- system prompt 注入和越狱；
- Agent 编排器权限提升或审批绕过；
- sandbox escape；
- MCP Server 供应链；
- Agent memory poisoning；
- 多 Agent 通信安全；
- 凭证隔离、数据外传和跨租户边界。

因此，评估一个 LLM Agent 产品时，应把 Codex Security 作为**代码安全审计器**，再单独设计 Agent Runtime、Sandbox、ModelRouter、审批、MCP、凭证和审计链的安全测试。

## 7. 与 OpenAI–Hugging Face 事件的对应

该产品视角可以直接解释事件中的薄弱点：如果 package proxy、代码执行 harness 或网络出口仍能被 Agent 组合利用，名义上的 sandbox 并不等于有效隔离；如果数据处理器、仓库和构建链携带不可信内容，工具调用就可能把内容变成代码执行和凭证访问。

因此，评估 Codex Security 时应分别测试：

1. 恶意仓库内容 → Agent context → 工具调用；
2. 工具调用 → 网络/文件副作用；
3. secret → 模型上下文/工具参数 → 外部 API 的数据流；
4. 修改、删除、部署和外发操作的审批与回滚；
5. 失败、重试、长程任务和跨会话状态的隔离。

## 8. 与主研究地图的对应

- `4.2 Prompt / Context`：仓库内容、注释、issue、测试输出是不可信 observation；
- `4.3 Harness / Orchestrator`：审批、重试、终止和任务目标；
- `4.5 Tool / API / Connector`：shell、代码执行、包安装、网络访问；
- `4.7 Sandbox / Runtime / Host`：workspace、进程、容器、网络和凭证；
- `4.8 Identity / Authorization`：用户、仓库、组织和部署权限；
- `4.9 Observability`：代码变更、工具调用、网络和审计 trace。

## 9. 参考链接

- [原始链接：learn.chatgpt.com/docs/security](https://learn.chatgpt.com/docs/security)
- [Codex Security CLI 文档](https://learn.chatgpt.com/docs/security/cli)
- [Codex Security 官方开发者文档](https://developers.openai.com/codex/security)
- [Codex Security GitHub 仓库（CLI 与 TypeScript SDK）](https://github.com/openai/codex-security)
- [Codex Security administration](https://learn.chatgpt.com/codex/security-administration)
- [Codex Agent approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security)
- [Codex Cyber Safety](https://learn.chatgpt.com/docs/cyber-safety)
