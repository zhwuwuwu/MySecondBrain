### For LLM Agent Helpers

**Scope**: this queue is for *learning topics* — things Zihan wants to study/look into, whose output is a note under `My Notes/`. It does **NOT** cover things he wants to build/do — those are "ideas", and live in `My Creations/Quick Ideas.md` instead.

Use the table below as the single source of truth:

- **分析完成**: check this box only after the topic has been researched and the note has been written.
- **笔记地址**: add the resulting Obsidian wiki-link, including the target folder when useful. Use `—` until a note exists.
- **描述 / 结论**: record the research question, a short description, or the key conclusion. Keep this readable and update it when the note changes materially.
- **已读**: this is independent from analysis completion. New rows and newly generated notes must start as unchecked (`[ ]`); Zihan checks it manually after reading the note.
- Keep one row per topic. For subtopics, use a separate row with an indented topic name rather than nested checklists.

### Topic List

| 主题                                                           | 笔记地址                                    | 分析完成 | 描述 / 结论                                                                                      | 已读  |
| ------------------------------------------------------------ | --------------------------------------- | :--: | -------------------------------------------------------------------------------------------- | :-: |
| MCP 协议无状态更新                                                  | [[MCP update]]                          | [x]  | —                                                                                            | [ ] |
| Loop, graph Engineering v.s. Langchain, Langgraph            | [[Loop Graph Engineering vs LangGraph]] | [x]  | graph engineering 是旧有 graph-based orchestration 的新行业命名，不是 LangGraph 的新替代技术；loop 是 graph 的特例。 | [ ] |
| WAIC                                                         | [[WAIC Summary]]                        | [x]  | 已完成 2026 展会大类、模型/Agent、具身智能/机器人、社媒声量和大公司/初创聚集分析；具体展品与推断已标注证据等级。 | [ ] |
| Gpt2 -> Kimi K3                                              | [[GPT2 to Kimi K3 技术路径]]                | [x]  | —                                                                                            | [ ] |
| Kimi 1-bit model                                             | [[Kimi 1-bit 模型]]                       | [x]  | 非 Kimi 官方工作，是 Unsloth 对开源权重的第三方量化；已补充 K3 的实测数据：1-bit GGUF 594GB / 约 78.9% 准确率。               | [ ] |
| LLM agent 自进化                                                | —                                       | [ ]  | —                                                                                            | [ ] |
| 　↳ agent 真实任务测评数据集                                           | —                                       | [ ]  | —                                                                                            | [ ] |
| 　↳ prompt 自动优化                                               | —                                       | [ ]  | —                                                                                            | [ ] |
| 　↳ 跨 model prompt 自适应优化配置                                    | —                                       | [ ]  | —                                                                                            | [ ] |
| AI 创业主流打法是什么                                                 | —                                       | [ ]  | AI agent 创业的主流打法是什么？大厂员工、明星员工、博士生、开源大手子通常有哪些途径和方向？                                           | [ ] |
| 构建自己的 OMO 端到端组合                                              | —                                       | [ ]  | —                                                                                            | [ ] |
| 纯文本 LLM 定价和视频生成、图像生成等多模态 agent 的定价模式区别                       | [[多模态模型定价与缓存]]                          | [x]  | 已整理图像 / 视频 / 音频理解与生成的成本单位、四类缓存、Seedance 与 MiniMax-H3 价格。                                     | [ ] |
| Minimax -> context organization offline aka “model dreaming” | [[MiniMax 离线上下文组织与 Model Dreaming]]     | [x]  | 这是非 MiniMax 官方命名的概念；核心是后台整理 agent 记忆 / 上下文，而不是模型权重自我训练。                                      | [x] |
