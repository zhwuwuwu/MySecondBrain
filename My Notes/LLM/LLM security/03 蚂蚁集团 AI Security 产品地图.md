# 蚂蚁集团 AI Security 产品地图

> 目标：把蚂蚁集团及其开源生态中与大模型、Agent 和数据安全相关的产品/项目放在同一页，区分产品定位、资料来源和仍待补充的信息。

## 1. 产品总览

- [蚁天鉴](<03 蚂蚁集团 AI Security 产品地图/01 蚁天鉴>)
- [SingGuard](<03 蚂蚁集团 AI Security 产品地图/02 SingGuard>)
- [SingGuard-NSFA](<03 蚂蚁集团 AI Security 产品地图/03 SingGuard-NSFA>)
- [AgentAegis](<03 蚂蚁集团 AI Security 产品地图/04 AgentAegis>)

| 产品/项目 | 类型 | 主要关注点 | 当前资料 |
|---|---|---|---|
| [蚁天鉴](<03 蚂蚁集团 AI Security 产品地图/01 蚁天鉴>) | 企业级大模型安全检测与防御方案 | 模型、Agent 和应用的安全评估、治理与防御 | 官方发布、清华资料 |
| [SingGuard](<03 蚂蚁集团 AI Security 产品地图/02 SingGuard>) | 多模态 LLM 安全护栏模型 | 内容和图文组合的安全判断 | 论文、GitHub |
| [SingGuard-NSFA](<03 蚂蚁集团 AI Security 产品地图/03 SingGuard-NSFA>) | Agent 行为安全护栏模型/框架 | Prompt injection、工具滥用、数据外泄等 Agent 风险 | 论文、GitHub |
| [AgentAegis](<03 蚂蚁集团 AI Security 产品地图/04 AgentAegis>) | Agent runtime 开源安全插件 | 在特定 Agent 运行时附近执行检查和拦截 | GitHub README |
| 蚂蚁密算 | 隐私计算/安全计算方向 | 数据使用过程中的隐私保护与安全协作 | 本页暂缺足够一手资料 |

## 2. 蚁天鉴

蚁天鉴是蚂蚁集团面向大模型/智能体安全的评测与解决方案方向，相关报道提到其升级并新增智能体安全评测工具。阅读时应把它理解为评测与治理能力，不要仅凭媒体标题推断其覆盖了所有 Agent runtime 安全问题。

### 研究关注点

- 评测对象是模型、Agent 还是完整应用；
- 是否覆盖 prompt injection、工具越权、数据泄露、RAG/Memory 和供应链；
- 是否能从“发现风险”落到确定性阻断、审批、审计和恢复；
- 评测结果是否有可复现的 threat model、数据集、指标和版本记录。

### 资料

- [蚁天鉴相关产品页（页面路径含 ASG，名称对应关系仍需以官方页面为准）](https://antdigital.com/products/ASG)
- [蚂蚁集团大模型安全解决方案“蚁天鉴”升级，新增智能体安全评测工具](https://blog.csdn.net/csdnnews/article/details/149721589)
- [清华与蚂蚁集团发布五层安全框架](https://889990.xyz/news/trends/1689/tsinghua-ant-openclaw-security-framework)

## 3. SingGuard 与 SingGuard-NSFA

### SingGuard

SingGuard 是面向自主 Agent 场景的 policy-adaptive multimodal LLM guardrail。论文和开源仓库是判断其能力边界的主要依据；它适合作为内容/行为风险判断和策略护栏的一层，不应被当作操作系统权限、凭证保护、网络隔离或 sandbox 的替代品。

### SingGuard-NSFA

SingGuard-NSFA 面向自主 AI Agent 的安全护栏框架/模型组合。研究时需要明确它位于 Agent 的哪一层：是输入输出检测、计划/工具调用评估、动作前策略执行，还是多层组合。只有确认执行点、失败策略和审计能力后，才能判断它对 Unsafe Action 的实际阻断能力。

### 资料

- [SingGuard 论文（arXiv:2606.22873）](https://arxiv.org/abs/2606.22873)
- [inclusionAI/Sing-Guard](https://github.com/inclusionAI/Sing-Guard)
- [inclusionAI/SingGuard-NSFA](https://github.com/inclusionAI/SingGuard-NSFA)
- [SingGuard-NSFA 项目介绍](https://zglg.work/ai/news/zh/2026-07-13-ant-group-open-sources-singguard-nsfa-a-new-security-guardrail-framework-for)
- [SingGuard+NSFA 部署介绍](https://blog.csdn.net/weixin_42376192/article/details/162858917)
- [蚂蚁集团开源安全模型报道](https://news.aibase.com/zh/news/29546)

## 4. AgentAegis

AgentAegis 是蚂蚁集团开源的 Agent 安全插件/工程组件。当前笔记的主要依据是中文 README，后续应从代码和配置进一步确认它具体提供哪些 hook、策略、工具调用检查、运行时隔离或审计能力。

- [AgentAegis 中文 README](https://github.com/antgroup/agent-aegis/blob/main/README_zh.md)

## 5. 蚂蚁密算

“蚂蚁密算”目前只有产品名称，缺少足够的一手资料。暂时将它放在本页的隐私计算/安全计算方向，不对其与 LLM/Agent Security 的具体结合方式作过度推断。补充资料时优先记录：官方产品页、威胁模型、保护的数据流、密码学/硬件依赖、部署边界、性能成本和审计方式。

## 6. 四个产品的横向关系

| 产品 | 主要层次 | 主要场景 | 与其他产品的关系 |
|---|---|---|---|
| 蚁天鉴 | 企业平台/方案 | 上线前评估、持续治理、Agent/MCP 扫描 | 可承载或管理其他能力，但未发现官方集成声明 |
| SingGuard | 内容安全模型 | 多模态输入输出审核 | 可作为内容风险判断模块 |
| SingGuard-NSFA | Agent 行为护栏模型/框架 | 计划、工具调用和执行前风险判断 | 与 AgentAegis 互补性较强 |
| AgentAegis | Agent runtime 插件 | 特定 Agent 的运行时拦截和控制 | 可把风险判断接到具体执行流程 |

### 简单的组合方式

```text
蚁天鉴：整体评估 / 治理
       ↓
SingGuard：内容与图文风险判断
SingGuard-NSFA：Agent 行为风险判断
       ↓
AgentAegis：在具体 Agent runtime 中执行拦截、审批或放行
```

这是基于公开定位做出的工作假设，不是官方公布的产品集成架构。

### “谁是谁的下一代？”

目前没有足够公开证据证明四者之间存在明确的版本继承关系。更稳妥的结论是：

- SingGuard-NSFA 不是已被官方确认的 SingGuard 下一代，而是更聚焦 Agent 的安全方向；
- AgentAegis 不是已被确认的 SingGuard-NSFA 运行时版本；
- 蚁天鉴也不是另外三个开源项目的简单升级版；
- 四者更像不同层次的产品：企业方案、内容护栏、Agent 护栏和运行时插件。

## 7. 统一比较框架

把这些产品放回 Agent 安全架构时，可按以下问题比较：

1. **检测**：识别什么输入、计划、工具参数或输出风险？
2. **决策**：模型判断是否能直接放行动作，还是只提供风险信号？
3. **执行**：是否存在确定性 policy enforcement point？
4. **边界**：是否覆盖身份、凭证、文件、网络、sandbox 和跨租户资源？
5. **审计**：是否保存完整上下文、工具调用、参数、审批和副作用 trace？
6. **恢复**：能否吊销凭证、隔离运行时、回滚状态并保留证据？

> 结论：护栏模型和评测工具主要解决判断与治理的一部分；真正的 Agent 安全仍需要能力网关、最小权限、运行时隔离、网络边界、审批、监控和恢复机制共同完成。
