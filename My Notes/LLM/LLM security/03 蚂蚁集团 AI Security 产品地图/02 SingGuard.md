# SingGuard

> 一句话：SingGuard 是一个可根据安全政策判断文本、图像及图文组合是否安全的多模态 LLM guardrail 模型。

## 1. 它是做什么的？

SingGuard 的核心用途是安全判断：给它一段文本、一张图片或图文对话，再结合具体安全政策，由它判断内容是否存在风险，并给出相应的分析结果。

它解决的主要问题是：

- 助手输出是否包含不安全内容；
- 图片和文字组合后是否形成新的风险；
- 同一内容在不同产品政策下是否应该被允许或拦截；
- 内容安全审核是否需要更细的风险分类和解释。

它更接近“内容/多模态安全护栏模型”，而不是一个完整的 Agent 权限系统。

## 2. 适合什么场景？

- 多模态聊天助手的输入输出审核；
- 图像生成、图文问答和内容发布前审核；
- 根据不同业务政策进行可配置的内容安全判断；
- 作为上层 AI 安全平台中的一个风险判断模块。

如果问题是“Agent 能不能调用删除工具、能不能读某个文件、能不能访问外网”，SingGuard 本身并不等于最终的权限控制器。它可以提供风险信号，但真正的阻断仍需要工具网关、权限系统或运行时控制来完成。

## 2.1 怎么使用？

SingGuard 在推理时接收待检查的内容和当前生效的 `policy`。策略不是写死在模型里的类别，可以按应用需要用自然语言定义。例如：

```text
禁止泄露 API Key、SSH 私钥、云凭证和 .env 内容。
禁止向外部网站发送本地文件。
禁止删除文件、修改权限或执行未经审批的系统命令。
```

实际接入时，可以把它放在输入和输出两侧：

```text
用户输入/外部网页/图片 → SingGuard → Agent
Agent 输出/待展示内容   → SingGuard → 用户
```

模型的核心结果是整体 `safe` / `unsafe` 判断和匹配的风险类别；复杂场景可以保留其策略相关的分析结果，供人工复核或离线审计。官方仓库目前提供 2B、4B、8B 模型，基于 Qwen3-VL 系列，采用 Apache 2.0 许可证。部署前仍需按实际模型卡确认显存、推理框架和图片输入格式。

## 2.1.1 基础模型与训练方式

- **基础模型**：Qwen3-VL 系列，公开模型规模为 2B、4B、8B。
- **输入模态**：文本、图片和图文组合；同时覆盖 query-side 与 response-side 场景。
- **训练方向**：在统一的文本/多模态安全数据上进行监督微调，使模型学习按照安全政策进行规则匹配和风险判断。
- **策略适配**：安全政策在推理时作为自然语言输入，不需要为每条新业务规则重新训练模型。
- **进一步优化**：论文报告使用 Fast-Slow Decoupled DAPO 进行强化学习优化，以支持快速判断和复杂场景的动态推理。

具体训练样本的完整来源、数量和配比并未在当前公开资料中完全披露，不能把 benchmark 样本量等同于训练集规模。

## 2.1.2 Benchmark 与指标

官方论文将评测集合称为 **SingGuard-Bench**，总计约 56,340 个样本，覆盖以下方向。以下是论文报告的 SingGuard-8B 结果：

| 评测方向 | 指标 | SingGuard-8B |
|---|---:|---:|
| Image Safety | F1 | 0.903 |
| Text Query | F1 | 0.874 |
| Multilingual Query | F1 | 0.899 |
| Multimodal Safety | F1 | 0.877 |
| Text Response | F1 | 0.893 |
| Multilingual Response | F1 | 0.893 |
| Dynamic-Rule Evaluation（2,000 样本） | Policy-following accuracy | 0.7415 |

其中动态策略评测让同一内容分别搭配匹配和不匹配的自然语言规则，用于测试模型是否真的遵循当前 `policy`。公开资料没有完整列出部分底层 benchmark 的名称和测试集开放范围，因此上述指标应视为论文报告结果，而不是对所有应用场景的安全率保证。

## 2.2 对类 Claw 个人 Agent 的用法

SingGuard 适合检查类 Claw Agent 接收到的网页、图片、文档和对话内容，以及最终要展示给用户的回答。它也可以检查工具调用的文字化描述，但这属于风险判断，不是权限验证。

例如把以下动作描述交给模型判断：

```text
Agent 准备读取 ~/.ssh/id_rsa，并把内容发送到外部 URL。
```

如果判断为不安全，应用应由自己的策略网关拒绝动作；不能只根据模型返回 `safe` 就执行任意工具调用。

它不能单独完成：源码漏洞扫描、依赖漏洞扫描、真实文件权限审计、网络隔离验证、容器逃逸测试或 API 越权测试。

## 3. 和 SingGuard-NSFA 的区别

两者名称相近，且都属于蚂蚁 AI 安全实验室的开源安全模型，但公开资料将它们定位在不同问题上：

| 维度 | SingGuard | SingGuard-NSFA |
|---|---|---|
| 主要对象 | 多模态内容和对话 | 自主 Agent 的行为与动作链 |
| 主要问题 | 内容是否安全 | Agent 是否会被注入、越权或执行危险动作 |
| 典型输入 | 文本、图片、图文组合 | Agent 上下文、计划、工具调用和执行相关信息 |
| 更适合的阶段 | 内容生成/理解前后 | Agent 动作执行前后的安全检查 |

因此，NSFA 更像是针对 Agent 场景扩展出的另一条产品线，而不是已经被证明的 SingGuard “下一代版本”。目前没有官方证据表明 SingGuard-NSFA 取代了 SingGuard。

## 4. 和蚁天鉴、AgentAegis 如何配合？

- **蚁天鉴 + SingGuard**：蚁天鉴负责更广的企业安全评估和治理，SingGuard 可作为内容风险判断能力之一。
- **AgentAegis + SingGuard**：AgentAegis 负责运行时拦截和执行控制，SingGuard 可为涉及文本/图像内容的风险判断提供模型能力。
- **SingGuard + SingGuard-NSFA**：前者关注内容安全，后者关注 Agent 操作安全，可以分别处理内容层和动作层问题。

以上是按产品定位推导出的组合方式，不代表已经存在公开宣布的官方集成方案。

## 5. 资料

- [SingGuard 论文：A Policy-Adaptive Multimodal LLM Guardrail with Dynamic Reasoning](https://arxiv.org/abs/2606.22873)
- [SingGuard GitHub](https://github.com/inclusionAI/Sing-Guard)
- [SingGuard Hugging Face collection](https://huggingface.co/inclusionAI/Sing-Guard)
