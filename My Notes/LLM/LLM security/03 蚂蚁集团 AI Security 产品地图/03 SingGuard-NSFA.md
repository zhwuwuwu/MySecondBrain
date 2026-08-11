# SingGuard-NSFA

> 一句话：SingGuard-NSFA 是面向自主 Agent 的安全护栏模型/框架，重点判断 Agent 的上下文、计划和工具行为是否可能造成安全风险。

## 1. 它是做什么的？

NSFA 面向的是“Agent 会不会做出危险动作”，而不只是“模型输出的文字是否违规”。公开资料重点覆盖的风险包括：

- prompt injection；
- 敏感数据提取或外传；
- 恶意代码执行；
- 危险工具调用；
- 权限误用；
- 资源滥用。

简单说，它试图在 Agent 从“理解内容”走向“调用工具、修改文件或产生外部副作用”时，提供一层安全判断和护栏。

## 2. 适合什么场景？

- 有 shell、浏览器、代码执行或业务 API 的自主 Agent；
- 需要检查 Agent 是否被外部内容诱导的工作流；
- 需要在高影响工具调用前做风险判断；
- 希望对 Agent 行为进行离线审查和在线快速检查的系统。

它比普通内容审核更靠近 Agent runtime，但仍不能替代操作系统权限、网络隔离、凭证保护和确定性授权。模型判断“危险”与系统真正“阻止动作”是两件事。

## 2.1 风险范围与模型形态

官方 NSFA 体系把风险组织为 7 个一级域、28 个二级风险和 185 个三级变体，覆盖两侧：

### Query-side：用户请求/外部输入

- Prompt Injection / Jailbreak；
- 恶意代码与网络攻击；
- 敏感信息窃取；
- 危险操作与工具滥用；
- 资源滥用。

### Response-side：Agent 输出/动作建议

- 危险动作生成；
- 敏感信息泄露。

NSFA 提供两种推理方式：

1. **Generative Reasoning**：生成风险分析和结构化判断，适合离线日志审计、事件调查和人工复核。
2. **Real-Time Classification**：通过分类头输出风险概率，适合在线快速检查和工具调用前的风险拦截。

官方发布 0.8B、2B、4B、9B 四种规模，采用 Apache 2.0 许可证。个人 Agent 可以先评估 0.8B 或 2B，但实际延迟和显存消耗要以本地硬件测试为准。

### 基础模型与训练方式

- **基础模型**：Qwen3.5-Base，公开规模为 0.8B、2B、4B、9B。
- **第一阶段**：基于风险分析数据进行 SFT，使骨干模型能够生成风险分析和结构化判断。
- **第二阶段**：冻结 SFT 后的骨干模型，在其表示上为不同风险域训练轻量级 MLP 分类头。
- **扩展方式**：新增风险类型时，可以只训练新的分类头，不必重训骨干，也不影响已有风险能力。
- **输入形式**：主要是文本；建议明确区分不可信用户输入/外部内容与 Agent 输出/动作描述。

它不是一个单一的“安全分数模型”，而是同一骨干上的生成式审计路径和分类式在线路径。

### Benchmark 与指标

官方 NSFA benchmark 基于 CIA（机密性、完整性、可用性）组织风险，共有 7 个一级域、28 个二级风险和 185 个三级变体，并参考/交叉验证了三套 OWASP 指南。

| 评测部分 | 规模 | 结果/指标 |
|---|---:|---:|
| Purpose-built multilingual samples | 93,000+，覆盖 133 种语言 | 四个规模模型均达到 94% 以上 F1 |
| Cross-source evaluation | 3,435 个样本，来自 5 个公开 Agent 安全数据集 | 9B 模型 F1 为 91.29% |
| 竞争模型对比 | — | 相比最强竞品护栏模型高 6–12 个 F1 百分点 |
| Real-Time Classification | 单样本在线分类 | 论文/仓库报告约 45–57 ms，测试条件为 A100，不代表本地性能 |

Purpose-built 数据集以 [NSFA_Benchmarks](https://huggingface.co/datasets/inclusionAI/NSFA_Benchmarks) 形式发布；cross-source 使用的 5 个公开数据集在当前主要资料中没有完整列出名称，开放范围需以数据集实际内容为准。

## 2.2 怎么使用？

NSFA 是文本护栏，建议把不可信内容明确包在边界标签内，区分用户输入和 Agent 输出：

```text
<untrusted_input>
用户请求、网页内容或外部文档内容
</untrusted_input>
```

```text
<untrusted_output>
Agent 计划、工具调用描述或最终回答
</untrusted_output>
```

在线模式可以在工具真正执行前检查工具名、完整参数和相关用户意图；离线模式可以扫描历史 trace。它有两种集成形态：生成式路径接近 LLM-as-a-Judge，分类头路径则是直接输出风险概率，不需要生成一段解释后再解析。检查结果应作为策略引擎的输入，而不是直接作为执行授权：

```text
用户输入 → NSFA 输入检查 → Agent 规划
                    ↓
       NSFA 检查计划/工具参数
                    ↓
       确定性 Policy Gateway
                    ↓
              执行或拒绝
                    ↓
        NSFA 检查结果和外发内容
```

推荐的集成接口是把一次待执行动作序列化为结构化文本，同时保留任务意图、用户身份、资源范围和完整工具参数：

```json
{
  "user_intent": "总结当前项目",
  "allowed_paths": ["D:/workspace/project-a"],
  "tool": "filesystem.read",
  "arguments": {"path": "D:/workspace/project-a/README.md"},
  "source": "agent_plan"
}
```

然后按风险类型选择：

1. **离线审计/复杂案例**：使用 Generative Reasoning，让模型给出风险分析和结构化判断。
2. **在线工具调用**：使用 Real-Time Classification 获取风险概率，按业务阈值决定放行、拒绝或转人工审批。
3. **最终执行**：由确定性 Policy Gateway、权限系统和 Sandbox 负责，不把模型输出当成授权结果。

## 2.3 对类 Claw 个人 Agent 的适用性

NSFA 可以帮助检查：

- 用户是否通过 Prompt Injection 诱导 Agent 越权；
- Agent 是否尝试读取 `.env`、SSH key 或云凭证；
- Shell、浏览器、代码执行和 MCP 工具参数是否危险；
- 是否存在删除、外传、部署或修改权限等高影响动作；
- Agent 输出是否包含敏感信息或危险操作建议；
- 历史 Agent trace 中是否存在多步风险行为。

但它不是完整的应用安全扫描器，不能单独发现 SQL 注入、SSRF、依赖漏洞、API 越权、容器逃逸或错误的操作系统权限。建议把它与源码/依赖扫描、确定性权限策略、沙箱、网络出口限制和审计日志组合使用。

核心原则：

> **NSFA 负责判断风险，Policy Gateway 和运行时负责真正阻断。**

不要把“模型判断安全”直接等价为“允许执行任意工具”。高风险动作（读取凭证、删除文件、访问内网、向外发送数据、修改 Agent 配置）应默认拒绝或要求人工审批。

## 3. 和 SingGuard 的区别

SingGuard 主要处理多模态内容安全，NSFA 主要处理 Agent 行为安全。可以把它们粗略理解为：

> **SingGuard 看“内容说了/展示了什么”；SingGuard-NSFA 看“Agent 接下来准备做什么”。**

两者存在模型和护栏层面的相似性，但目前没有证据证明 NSFA 是 SingGuard 的正式下一代或替代版本。更稳妥的判断是：NSFA 是面向 Agent 场景的专门化方向。

## 4. 和蚁天鉴、AgentAegis 如何配合？

- **蚁天鉴**：可负责组织级的安全评估、Agent/MCP 扫描和治理；NSFA 可作为 Agent 行为风险判断的一类能力。公开资料没有确认二者已官方集成。
- **AgentAegis**：NSFA 偏“判断风险”，AgentAegis 偏“接入特定运行时并执行拦截/控制”。一个可以作为判断层，另一个负责把结果连接到 Agent 执行流程；这是合理的架构配合假设，不是官方产品承诺。
- **SingGuard**：可分别覆盖内容风险和动作风险。

## 5. 资料

- [SingGuard-NSFA 论文：Extensible Guardrails for Agentic AI](https://arxiv.org/abs/2607.13081)
- [SingGuard-NSFA GitHub](https://github.com/inclusionAI/SingGuard-NSFA)
- [SingGuard-NSFA Hugging Face collection](https://huggingface.co/collections/inclusionAI/singguard-nsfa)
- [NSFA Benchmarks 数据集](https://huggingface.co/datasets/inclusionAI/NSFA_Benchmarks)
