> MCP（Model Context Protocol）从 **2025-03-26** 版本起，用 **Streamable HTTP** 传输替代了旧的 HTTP+SSE 传输，核心变化是：session（`Mcp-Session-Id`）从"必须"变成"可选"，服务端可以选择完全无状态（每个请求自包含，任意副本都能处理）。这不是"删掉了 session"，而是"把 session 做成了可插拔的可选项"，默认推荐无状态。

## 目录

1. [规范版本时间线](#规范版本时间线)
2. [机制变化：新旧对比](#机制变化新旧对比)
3. [官方 SDK 怎么写无状态 server](#官方-sdk-怎么写无状态-server)
4. [对 MCP Server 开发者的影响](#对-mcp-server-开发者的影响)
5. [对 Agent/Client 接入方的影响](#对-agentclient-接入方的影响)
6. [对基础设施/部署的影响](#对基础设施部署的影响)
7. [安全侵位：无状态 ≠ 免认证](#安全侵位无状态--免认证)
8. [向后兼容](#向后兼容)
9. [一句话结论](#一句话结论)

---

## 规范版本时间线

| 版本 | 关键改动 | 来源 |
|---|---|---|
| **2024-11-05** | 初版：HTTP+SSE 传输，两个端点（POST 发消息 + 长连接 SSE 收消息），天然有状态 | [spec](https://modelcontextprotocol.io/specification/2024-11-05) |
| **2025-03-26** | ★ 用 **Streamable HTTP** 替换 HTTP+SSE（[PR #206](https://github.com/modelcontextprotocol/specification/pull/206)），单端点 `/mcp`，session 变可选，支持真正无状态部署 | [changelog](https://modelcontextprotocol.io/specification/2025-03-26/changelog) |
| **2025-06-18** | 增加 `MCP-Protocol-Version` header 协议版本协商（必须携带）；把 MCP server 定性为 OAuth Resource Server；新增 elicitation、resource links | [changelog](https://modelcontextprotocol.io/specification/2025-06-18/changelog) |
| **2025-11-25**（最新） | 明确"不要仅靠 session ID 做身份/状态载体"的安全指引（session hijacking 防护）；OIDC Discovery；异步任务 `tasks`（长任务轮询） | [changelog](https://modelcontextprotocol.io/specification/2025-11-25/changelog) |

**结论**：Quicknotes 里说的"无状态更新"准确指向 **2025-03-26** 这次传输层改版；2025-06-18 / 2025-11-25 是在这个基础上补安全和协议协商的细节，没有再动传输的无状态属性。

## 机制变化：新旧对比

| | 旧：HTTP+SSE (2024-11-05) | 新：Streamable HTTP (2025-03-26+) |
|---|---|---|
| 端点数量 | 2 个（POST 发消息端点 + 独立 SSE 端点接收） | 1 个（同一 `/mcp` 同时接受 POST 和 GET） |
| 连接方式 | 必须维持一条长连接 SSE 才能收到 server 推送 | SSE 变成**可选**，server 也可以直接用普通 JSON 响应 POST |
| Session | 隐含在长连接生命周期里，天然有状态 | `Mcp-Session-Id` header 可选：server 初始化时给不给 session ID 完全由自己决定 |
| 水平扩展 | 客户端连的哪个实例，之后请求必须回到同一实例（需要 LB 粘性会话/sticky session） | 无状态模式下任意副本可处理任意请求，标准 Web 服务扩容方式即可 |
| Serverless 兼容性 | 差（Lambda/Workers 天生不喜欢长连接） | 好（单端点、短生命周期请求，是 serverless 的天然形态） |
| Session 过期处理 | 无标准化 | 明确规定：过期 session 返回 `404`，客户端应重新 initialize |

## 官方 SDK 怎么写无状态 server

**TypeScript SDK** — `sessionIdGenerator: undefined` 是官方"无状态惯用法"（在 SDK 源码注释里被称为 "the established stateless idiom"）：

```ts
const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined   // 不生成 session id → 无状态
});
```
每个请求用一个"全新"的 transport + server 实例处理，GET/DELETE（session 相关操作）在无状态模式下直接返回 `405 Method Not Allowed`。

**Python SDK** — 显式的 `stateless_http` 参数，推荐搭配 `json_response=True`：

```python
mcp.run(transport="streamable-http", stateless_http=True, json_response=True)
# 官方注释原话：Stateless HTTP: every request is its own world.
# No channel back to the client.
```

两个 SDK 的默认值都是"支持无状态，但需要显式开启"——不是新协议强制无状态，而是**把无状态做成了一等公民的可选项**。

## 对 MCP Server 开发者的影响

- **默认建议改成无状态**：除非有明确理由（比如需要 server 主动多轮推送/`sampling`/`elicitation` 之间的上下文延续），否则直接 `sessionIdGenerator: undefined` / `stateless_http=True`。
- **不要把状态存在进程内存里**（模块级变量、全局 dict 等）——无状态模式下下一个请求可能被另一个实例处理，进程内存不可靠。真需要跨请求状态：外部化到 Redis / DynamoDB，按认证身份（而不是 session ID）做 key。
- 如果坚持要做有状态 server（比如需要长连接 SSE 推送），2025年5月前后的现状是：**官方 SDK 都不提供跨实例的 session 持久化**，水平扩展只能靠 LB 层的粘性会话（cookie-based sticky session），而 TS 官方 Client SDK 基于 `fetch`，对 cookie 支持并不原生，扩展有额外坑。Cloudflare 的 `McpAgent` + Durable Object 是目前少数原生支持"有状态但仍可扩展"的方案（每个 session 一个 Durable Object，支持 hibernation）。
- 流式响应（SSE）在无状态模式下依然可用，只是从"必须长连接"降级为"按需可选"。

## 对 Agent/Client 接入方的影响

面向"接 MCP server 的 agent 框架/客户端开发者"，这次改动意味着：

- **不能假设每次 initialize 都会拿到 `Mcp-Session-Id`**——server 完全有权不返回它（无状态模式）。客户端逻辑要能优雅处理"没有 session id"这种情况，而不是硬编码要求。
- **必须携带 `MCP-Protocol-Version` header**（2025-06-18 起为 MUST）做协议版本协商，否则可能被判定为走旧协议路径。
- 如果 server 返回 `404`（session 过期/不存在），客户端应当**重新走一次 initialize**，而不是报错终止。
- GET（server→client 推送流）在无状态 server 上可能直接 `405`——不要假设 server 一定支持 server-initiated push，做好优雅降级。
- 老版本兼容：客户端可以先尝试新版 Streamable HTTP，initialize 失败再回退到旧版 HTTP+SSE（见下方"向后兼容"）。
- 对 LangChain / LlamaIndex / OpenAI Agents SDK / Claude Agent SDK 这类上层框架而言，实际影响主要是**传输层适配已经在各自 MCP client 适配层里处理掉了**，业务开发者一般不需要手写这部分协议细节，但如果自己直接用官方 SDK 撸 client，要留意上述几点。

## 对基础设施/部署的影响

这是本次改动里收益最直接的一块——无状态让 MCP server **终于能被当成普通 Web 服务来部署**：

- **Serverless 天然契合**：AWS Lambda（+ API Gateway / Lambda Web Adapter / Function URL）、Cloudflare Workers、Vercel（`mcp-handler`，Fluid Compute）都已经有官方/社区的无状态 MCP 部署样例（`aws-samples/sample-serverless-mcp-servers` 等）。
- **扩容方式回归常规 Web 运维**：多副本 + 负载均衡 + 健康检查 + 自动扩缩容，**不需要 session 亲和性**，因为任意副本都能处理任意请求。
- **实际踩坑点（来自实践博客，非规范文本）**：
  - Lambda 冷启动仍然明显（可到数秒级），需要 Provisioned Concurrency / SnapStart 缓解；
  - 流式响应在 API Gateway / ALB 上会被缓冲，需要 Lambda Function URL + `AWS_LWA_INVOKE_MODE=response_stream` 才能真正流式；
  - Lambda 场景下和 Powertools 之类的原生 Lambda 可观测性工具集成不顺畅（因为跑的是一个"服务器"而不是标准的 handler 模式）；
  - 2025 年中，官方 SDK 尚未提供跨实例的有状态 session 存储方案，"有状态 + 可扩展"仍需自行拼 Redis/DynamoDB 或用 Cloudflare Durable Object。

## 安全侵位：无状态 ≠ 免认证

2025-11-25 规范新增的安全最佳实践明确指出：**不要把 session ID 当作认证手段**。原因：
- session 劫持风险：多实例共享队列的架构下，攻击者拿到 session ID 就能冒充用户；
- 正确做法：每个请求都要独立校验身份（配合 2025-06-18 引入的 OAuth Resource Server 模型，用认证凭据里的 `sub` claim 做用户身份，而不是 session ID）；
- 如果真的要用 session ID，必须是密码学安全随机生成、绑定用户身份、可轮换/过期。

也就是说：无状态化解决的是**扩展性**问题，认证/授权是一个完全独立的、必须单独做对的维度——两者不要混为一谈。

## 向后兼容

- Server 可以同时挂载新的 Streamable HTTP 端点和旧的 SSE 端点，兼容老客户端。
- Client 可以先尝试新传输初始化，失败（特定 HTTP 错误码）后自动回退到旧版长连接 SSE。

## 一句话结论

MCP 的"无状态更新"本质是**把 session 从协议隐含的强制项，变成了显式的可选项**，默认推荐无状态。这直接解除了 MCP server 此前"必须像有状态应用一样部署"的枷锁，让它能用最普通的 Web/Serverless 运维方式水平扩展；代价是——如果你的 server 确实需要跨请求记忆，需要自己动手把状态外部化，协议本身不再帮你兜底。

---

**Sources**: [MCP Specification — Transports (2025-06-18)](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports) · [2025-03-26 changelog](https://modelcontextprotocol.io/specification/2025-03-26/changelog) · [2025-11-25 changelog](https://modelcontextprotocol.io/specification/2025-11-25/changelog) · [2025-11-25 security best practices](https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices) · [modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk) · [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) · [aws-samples/sample-serverless-mcp-servers](https://github.com/aws-samples/sample-serverless-mcp-servers) · [Building Serverless MCP Servers (Ran Isenberg)](https://ranthebuilder.cloud/blog/building-serverless-mcp-server/) · [Deploying a Remote MCP Server — AgentsCamp](https://agentscamp.com/guides/mcp/deploy-remote-mcp-server) · [Building a Serverless MCP Server — AgentCat](https://agentcat.com/guides/building-serverless-mcp-server/)
