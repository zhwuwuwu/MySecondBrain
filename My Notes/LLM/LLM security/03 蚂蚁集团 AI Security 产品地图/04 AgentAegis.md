# AgentAegis

> 一句话：AgentAegis 是面向 OpenClaw、Hermes Agent 等 Agent 运行时的开源安全插件，重点是在 Agent 执行过程中检查并拦截高风险行为。

## 1. 它是做什么的？

AgentAegis 不是泛用的企业安全评测平台，也不是单独的内容安全模型。它更像一个可以接入 Agent runtime 的防护组件，关注 Agent 已经开始运行之后的安全问题，例如：

- 恶意 Skill 或插件；
- prompt injection 和工具结果注入；
- 记忆污染；
- 高危 shell 命令；
- SSRF、敏感信息外泄和输出脱敏；
- Agent 试图关闭自身防御的行为。

它的价值在于把安全检查放到 Agent 生命周期和工具执行附近，而不是只在最终回答出来之后再审核。

## 2. 适合什么场景？

- 个人或团队部署 OpenClaw/Hermes Agent；
- 需要快速给现有 Agent 增加运行时保护；
- 需要保护本地文件、记忆、Skill 和工具调用；
- 希望先 observe、再 enforce，逐步上线安全策略的场景。

它尤其适合“已经有一个 Agent，希望在执行层加防护”的需求。若组织需要统一评测很多模型和应用、输出合规报告或做企业级治理，蚁天鉴更接近那个层次。

## 3. 和另外三个产品的区别

| 产品 | 更像什么 | 主要回答的问题 |
|---|---|---|
| 蚁天鉴 | 企业级检测与防御方案 | 这个模型/Agent/应用整体安全吗，如何治理？ |
| SingGuard | 多模态安全判断模型 | 这段内容或图文组合是否有风险？ |
| SingGuard-NSFA | Agent 行为安全护栏模型/框架 | Agent 的计划或工具动作是否有风险？ |
| AgentAegis | 特定运行时的防护插件 | 这个动作在当前 Agent 中是否应该被拦截或放行？ |

AgentAegis 和 SingGuard-NSFA 的重叠最大：都关注 Agent 行为安全。但前者更偏“运行时接入和执行控制”，后者更偏“可复用的 Agent 风险判断能力”。

## 4. 如何配合使用？

一个合理但需要自行验证的组合是：

1. 用蚁天鉴做上线前的整体安全评估和持续治理；
2. 用 SingGuard 处理多模态内容安全；
3. 用 SingGuard-NSFA 对 Agent 计划、上下文和工具调用做风险判断；
4. 用 AgentAegis 在具体 Agent runtime 中把部分结果接入 observe/enforce、审批和执行拦截。

这不是官方公布的完整产品套装。尤其不能把 AgentAegis 当成蚁天鉴的客户端，也不能把 SingGuard-NSFA 当成 AgentAegis 的模型依赖，除非产品文档明确说明。

## 5. 是否是其他产品的下一代？

目前没有可靠证据表明 AgentAegis 是蚁天鉴、SingGuard 或 SingGuard-NSFA 的下一代。它们更可能是不同形态的产品：平台方案、内容护栏、Agent 护栏和运行时插件。

## 6. 资料

- [AgentAegis GitHub](https://github.com/antgroup/agent-aegis)
- [AgentAegis 中文 README](https://github.com/antgroup/agent-aegis/blob/main/README_zh.md)
- [AgentAegis LICENSE](https://github.com/antgroup/agent-aegis/blob/main/LICENSE)
