# AgentAegis

> 一句话：AgentAegis 是蚂蚁集团开源的、面向 OpenClaw 类 Agent 的运行时安全插件，覆盖从启动到执行和输出的多个生命周期环节，可对高风险行为进行观察、告警、要求确认或拦截。

> **核验状态（2026-08）**：以下关于开源属性、配置项和防护范围的结论以 [官方 GitHub 仓库](https://github.com/antgroup/agent-aegis)、[中文 README](https://github.com/antgroup/agent-aegis/blob/main/README_zh.md)、[Apache-2.0 LICENSE](https://github.com/antgroup/agent-aegis/blob/main/LICENSE) 和仓库源码为准。README 中的产品定位不等于对所有攻击变体的检测保证；“未覆盖”表示官方公开材料中未见充分证据。

## 1. 它是做什么的？

AgentAegis 不是泛用的企业安全评测平台，也不是单独的内容安全模型。它更像一个可以接入 Agent runtime 的防护组件，关注 Agent 已经开始运行之后的安全问题，例如：

- 恶意 Skill 或插件；
- prompt injection 和工具结果注入；
- 记忆污染；
- 高危 shell 命令；
- SSRF、敏感信息外泄和输出脱敏；
- Agent 试图关闭自身防御的行为。

它的价值在于把安全检查放到 Agent 生命周期和工具执行附近，而不是只在最终回答出来之后再审核。官方将其描述为 OpenClaw 的轻量化内置安全插件，提供从初始化到执行的五层纵深防御：可信基座、感知输入、认知状态、决策对齐和执行控制。

它不是：

- 通用的企业级安全评测/治理平台；
- 独立的内容安全大模型；
- 操作系统级沙箱、容器隔离或网络防火墙；
- 可以自动覆盖所有插件、依赖包和主机进程的供应链安全平台。

更准确的定位是：**规则与策略主导的 Agent 应用层运行时防护插件，并通过安全上下文影响后续模型推理**。

## 2. 是否开源？

是。官方仓库公开，采用 Apache License 2.0，包名为 `@antgroup/agent-aegis`，`package.json` 中 `private: false`，并声明了 OpenClaw 插件入口和兼容的 Plugin API 版本。

因此它是开源软件和开源安全插件，但原生集成目标是 OpenClaw。其他“类 Claw”助手不能直接假设兼容，需要实现生命周期适配层。

## 3. 适合什么场景？

- 个人或团队部署 OpenClaw 或其他兼容其插件/生命周期接口的类 Claw Agent；
- 需要快速给现有 Agent 增加运行时保护；
- 需要保护本地文件、记忆、Skill 和工具调用；
- 希望先 observe、再 enforce，逐步上线安全策略的场景。

它尤其适合“已经有一个 Agent，希望在执行层加防护”的需求。若组织需要统一评测很多模型和应用、输出合规报告或做企业级治理，蚁天鉴更接近那个层次。

## 4. 覆盖哪些攻击面？

AgentAegis 的覆盖重点是 Agent 应用层运行时，尤其是“输入污染 → 模型决策 → 工具执行 → 数据外泄/持久化污染”的攻击链。

### 4.1 Skill、插件和 Agent 资产

可检测或保护：

- Skill 投毒；
- Skill 试图关闭 AgentAegis 或禁用安全插件；
- Skill 超出预期权限；
- Skill 中的高危命令；
- 凭据窃取；
- 远程脚本或二进制下载引导；
- 受保护 Skill、Plugin、文件和目录的删除、覆盖或篡改。

主要发生在启动扫描和运行期资产保护阶段。它不是完整的软件供应链扫描器，不能据此推断所有依赖包、远程服务或二进制都已完成安全审计。

### 4.2 用户输入和提示词注入

官方源码中的用户风险类别包括：

```text
jailbreak-bypass
policy-bypass
tool-induction
secret-request
exfiltration-request
disable-claw-aegis
dangerous-execution-request
sensitive-secret-request
```

对应攻击面包括越狱、策略绕过、诱导调用工具、密钥请求、数据外泄请求、危险执行请求和关闭防御请求。检测结果可以用于安全上下文注入、告警、确认或阻断。

### 4.3 工具调用和高危命令

在工具实际执行前，重点检查：

- 高危 Shell 命令；
- 命令混淆和编码载荷；
- 写入脚本后立即执行；
- 高危文件操作；
- 修改安全配置或关闭自身防御；
- 针对受保护路径的读写、删除和覆盖；
- 重复或变异的工具调用循环；
- 读取敏感数据后外发的数据泄露链。

这部分是 AgentAegis 的核心拦截面，涉及 `commandBlockMode`、编码检查、自保护、受保护路径、循环和外泄防护等配置或机制。

### 4.4 工具结果和间接 Prompt Injection

工具结果被视为不可信输入。官方策略类别包括：

```text
prompt-injection
special-token-injection
role-impersonation
```

因此可以审查网页搜索、浏览器、MCP 或其他外部工具返回的内容，降低其伪装成系统指令、注入特殊 Token、冒充角色或诱导下一步危险操作的风险。

注意：工具结果检查发生在工具调用之后，不能撤销工具本身已经产生的副作用。

### 4.5 记忆和上下文污染

主要保护 `memory_store`、`MEMORY.md`、`SOUL.md`、`memory/` 等持久化记忆位置，检查可疑或异常的大规模写入，降低跨会话 Prompt 投毒和上下文污染风险。

它只能可靠覆盖已经接入 AgentAegis 的记忆写入路径；外部数据库、向量库、自定义缓存或其他进程直接写入的存储，需要另外接入防护。

### 4.6 敏感信息外泄和输出脱敏

覆盖重点包括：

- API Key、Token、Secret、密码等凭据；
- 工具调用链中的疑似数据外发；
- Agent 对外发送或存储前的敏感值。

输出阶段可以进行脱敏，但这不是完整的数据防泄漏系统，不能保证识别所有业务机密、图片中的秘密、编码/加密/分片后的数据，或绕过 Agent 工具链的外发行为。

### 4.7 SSRF、网络外联和资源耗尽

官方 README 将 SSRF、数据泄露链、重复调用循环和资源耗尽风险列入运行时防护范围。实际定位更接近 Agent 工具层的风险识别和策略控制，而不是全流量网络防火墙。

它不能替代网络出口控制、DNS 安全、云安全组、主机入侵检测、CPU/内存配额或容器级资源限制。

### 4.8 自保护和安全配置篡改

可防护 Agent 或 Skill：

- 关闭 AgentAegis；
- 禁用其他安全插件；
- 修改安全配置；
- 删除防护规则；
- 篡改受保护文件、Skill 或 Plugin。

这是它区别于单纯内容审核器的重要能力。

## 5. 生命周期覆盖矩阵

| 生命周期阶段                                     | 主要检查或防护                               |
| ------------------------------------------ | ------------------------------------- |
| `gateway_start`                            | 启动初始化、受保护路径、自完整性和 Skill 扫描            |
| `message_received`                         | 用户意图、越狱、策略绕过、密钥/外泄/危险执行请求             |
| `message_sending`                          | API Key、Token 等敏感输出脱敏                 |
| `before_prompt_build`                      | 注入静态和动态安全上下文                          |
| `before_dispatch`                          | 针对受保护资源的危险操作                          |
| `before_agent_reply`                       | Agent 回复前的风险检查                        |
| `before_tool_call`                         | 命令、编码、脚本、记忆、自保护、循环和外泄防护               |
| `after_tool_call`                          | 工具结果中的 Prompt Injection、特殊 Token、角色冒充 |
| `before_message_write`                     | 消息或记忆写入前的敏感信息和污染检查                    |
| `llm_output` / `agent_end` / `session_end` | 输出、运行状态和会话收尾相关处理                      |

## 6. 检测与处置模式

每类防御通常可配置为：

| 模式 | 含义 |
|---|---|
| `observe` | 检测、记录、告警，不主动阻断；适合上线初期评估误报 |
| `enforce` | 执行阻断、拒绝、脱敏或要求用户确认 |
| `off` | 关闭该项防御 |

建议先全局 `observe`，再仅对高置信度风险启用 `enforce`，例如关闭防御、高危 Shell、受保护文件删除/覆盖、明显凭据外泄和恶意 Skill 篡改。

## 7. 覆盖边界

AgentAegis 覆盖面广，但不能等价于全栈安全：

| 安全领域                   | 覆盖判断                   |
| ---------------------- | ---------------------- |
| 用户输入、Prompt Injection  | 覆盖，规则和运行时策略为主          |
| Skill/插件投毒             | 覆盖启动扫描和资产保护，但不是完整供应链审计 |
| 工具调用、Shell、文件操作        | 覆盖较强，前提是动作经过受控 runtime |
| 工具结果注入                 | 覆盖                     |
| 记忆污染                   | 覆盖已接入的持久化路径            |
| 凭据外泄、输出脱敏              | 覆盖常见模式和输出链路            |
| SSRF、外泄链、循环调用          | 部分覆盖，依赖工具链和策略配置        |
| 主机/内核安全、容器沙箱           | 不属于主要能力                |
| 全流量网络安全                | 不属于主要能力                |
| 所有依赖包和二进制的供应链安全        | 不属于主要能力                |
| 业务逻辑漏洞、模型幻觉            | 不能保证覆盖                 |
| 任意绕过 runtime 的子进程或外部程序 | 不能可靠覆盖                 |

最重要的前提是：**所有有副作用的 Shell、文件、网络、浏览器、MCP 和记忆操作都必须经过 AgentAegis 接管的工具接口；否则插件无法观察或阻断绕过路径。**

## 8. 如何接入类 Claw 助手？

如果助手兼容 OpenClaw 插件机制，可按官方 README 安装：

```bash
git clone https://github.com/antgroup/agent-aegis
openclaw plugins install ./agent-aegis
```

如果只是类 Claw，需要把自己的生命周期映射到上述检查点：

```text
接收输入
  → 输入风险检查
  → LLM 生成计划
  → 计划/意图检查
  → 工具调用前策略检查
  → 工具执行
  → 工具结果检查
  → 记忆写入检查
  → 输出脱敏
```

不要只包住 LLM 调用；真正关键的是统一接管所有有副作用的工具和记忆写入。

## 9. 和另外三个产品的区别

| 产品 | 更像什么 | 主要回答的问题 |
|---|---|---|
| 蚁天鉴 | 企业级检测与防御方案 | 这个模型/Agent/应用整体安全吗，如何治理？ |
| SingGuard | 多模态安全判断模型 | 这段内容或图文组合是否有风险？ |
| SingGuard-NSFA | Agent 行为安全护栏模型/框架 | Agent 的计划或工具动作是否有风险？ |
| AgentAegis | 特定运行时的防护插件 | 这个动作在当前 Agent 中是否应该被拦截或放行？ |

AgentAegis 和 SingGuard-NSFA 的重叠最大：都关注 Agent 行为安全。但前者更偏“运行时接入和执行控制”，后者更偏“可复用的 Agent 风险判断能力”。

## 10. 如何配合使用？

一个合理但需要自行验证的组合是：

1. 用蚁天鉴做上线前的整体安全评估和持续治理；
2. 用 SingGuard 处理多模态内容安全；
3. 用 SingGuard-NSFA 对 Agent 计划、上下文和工具调用做风险判断；
4. 用 AgentAegis 在具体 Agent runtime 中把部分结果接入 observe/enforce、审批和执行拦截。

这不是官方公布的完整产品套装。尤其不能把 AgentAegis 当成蚁天鉴的客户端，也不能把 SingGuard-NSFA 当成 AgentAegis 的模型依赖，除非产品文档明确说明。

## 11. 是否是其他产品的下一代？

目前没有可靠证据表明 AgentAegis 是蚁天鉴、SingGuard 或 SingGuard-NSFA 的下一代。它们更可能是不同形态的产品：平台方案、内容护栏、Agent 护栏和运行时插件。

## 12. 资料

- [AgentAegis GitHub](https://github.com/antgroup/agent-aegis)
- [AgentAegis 中文 README](https://github.com/antgroup/agent-aegis/blob/main/README_zh.md)
- [AgentAegis LICENSE](https://github.com/antgroup/agent-aegis/blob/main/LICENSE)
