---
title: "MCP 2026-07-28 无状态协议革命——5天后正式发布，你准备好了吗？"
date: 2026-07-23
tags: ["MCP", "AI", "协议", "无状态", "Agent"]
summary: "MCP 自 2024 年发布以来最大一次协议升级即将落地：移除 Session、废弃初始化握手、引入 Extensions 框架。本文深度解析这场无状态革命的技术细节与迁移路径。"
---

# MCP 2026-07-28 无状态协议革命——5天后正式发布，你准备好了吗？

> 7 月 28 日，Model Context Protocol 将发布自诞生以来最大规模的破坏性升级。如果你正在运营 MCP 服务器，或者你的 AI Agent 依赖 MCP 连接外部工具——是时候认真对待了。

## 一、那个凌晨三点的生产事故

想象这个场景：你的 AI Agent 正在通过 MCP 调用一个远端数据库工具，执行一个需要 30 秒的复杂查询。就在结果即将返回时，负载均衡器切断了 SSE 长连接，`Mcp-Session-Id` 失效，整个调用链中断。Agent 不知道该重试还是放弃，用户看到的只是一个莫名的超时错误。

这不是假设——这是 MCP `2025-11-25` 版本有状态架构下的真实痛点。**粘性路由**、**共享 Session Store**、**SSE 断连恢复**……这些分布式系统的经典难题，每一个都在侵蚀 MCP 作为"AI 工具协议标准"的可靠性。

5 天后，这一切将成为历史。

## 二、协议演进全景：从诞生到革命

让我们先回顾 MCP 的关键里程碑：

| 版本 | 时间 | 核心变化 |
|------|------|----------|
| `2024-11-05` | 2024年11月 | Anthropic 发布 MCP，传输层为 stdio + HTTP/SSE |
| `2025-03-26` | 2025年3月 | 引入 Streamable HTTP，废弃旧 HTTP+SSE |
| `2025-06-18` | 2025年6月 | 安全加固、结构化输出、OAuth 对齐 |
| `2025-11-25` | 2025年11月 | MCP 一周年，Streamable HTTP 成熟，Tasks 实验特性 |
| **`2026-07-28`** | **2026年7月** | **无状态核心、Extensions 框架、MCP Apps、授权强化** |

`2026-07-28` 版本的 Release Candidate 于 **5 月 21 日**锁定，经历了整整 10 周的验证窗口，即将在 7 月 28 日正式发布。这是 MCP 自诞生以来**唯一一次包含破坏性变更的大版本**。

## 三、核心变革：协议层彻底无状态化

### 3.1 初始化握手被移除（SEP-2575）

旧版本中，客户端必须先发送 `initialize` 请求建立会话：

```http
POST /mcp HTTP/1.1
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"initialize",
 "params":{"protocolVersion":"2025-11-25","capabilities":{},
           "clientInfo":{"name":"my-app","version":"1.0"}}}
```

服务端返回 `Mcp-Session-Id`，之后每个请求都必须携带这个 ID——一旦丢失，整个会话作废。

**新版本完全移除了这个握手。** 协议版本、客户端信息和能力声明现在随每个请求在 `_meta` 中携带：

```http
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: search
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"search","arguments":{"q":"otters"},
           "_meta":{
             "io.modelcontextprotocol/clientInfo":{"name":"my-app","version":"1.0"},
             "io.modelcontextprotocol/protocolVersion":"2026-07-28"
           }}}
```

一个请求，自包含，任何服务器实例都能处理。

### 3.2 Session 彻底消失（SEP-2567）

`Mcp-Session-Id` 头部被移除。协议层不再管理会话状态。

**这意味着什么？**

| 方面 | 旧版（2025-11-25） | 新版（2026-07-28） |
|------|-------------------|-------------------|
| 负载均衡 | 粘性路由（Sticky Session） | 普通轮询（Round-Robin） |
| 水平扩展 | 需要共享 Session Store | 无状态，直接加机器 |
| 故障恢复 | 重连复杂，Session 可能丢失 | 简单重试，无状态天然容错 |
| CDN/Edge 部署 | 几乎不可能 | 原生兼容 Serverless |
| 网关 | 需要深度包检测（DPI） | 只看 Header 即可路由 |

### 3.3 有状态的应用怎么办？

协议无状态 ≠ 应用必须无状态。如果你的工具需要跨调用状态（比如购物车、浏览器会话），只需：

1. 工具调用返回一个显式 handle（如 `basket_id`）
2. AI 模型在下次调用时把 handle 作为参数传回

```
模型 → create_basket() → 服务器返回 {"basket_id": "bsk_abc123"}
模型 → add_item(basket_id="bsk_abc123", item="keyboard") → 成功
```

这种模式比隐式 Session 更强大——模型可以**推理** handle、跨工具组合 handle、在多步骤之间传递 handle，而不是依赖于对它不可见的传输层元数据。

## 四、Multi Round-Trip：无状态下的交互式请求

旧版中，服务端如果需要用户确认（比如"确定删除 3 个文件？"），依赖持续打开的 SSE 流。新版引入了 **Multi Round-Trip Requests**（SEP-2322）模式：

服务端返回一个 `InputRequiredResult`：

```json
{
  "resultType": "inputRequired",
  "inputRequests": {
    "confirm": {
      "type": "elicitation",
      "message": "确定删除这 3 个文件？",
      "schema": { "type": "boolean" }
    }
  },
  "requestState": "eyJzdGVwIjoxLCJmaWxlcyI6WyJhIiwiYiIsImMiXX0="
}
```

客户端收集用户输入后，带着 `inputResponses` 和不透明的 `requestState` 重新发起原始调用。任何服务器实例都能接手处理，因为所有状态都在 payload 里。

**精妙之处：** `requestState` 是一个 Base64 编码的不透明 blob，由服务端加密/签名，客户端只负责原样传回。这就像 HTTP 的 cookie，但更显式、更安全。

## 五、路由、缓存与可观测性

### 5.1 Header 级路由（SEP-2243）

新增 `Mcp-Method` 和 `Mcp-Name` 两个必需头部：

```http
Mcp-Method: tools/call
Mcp-Name: database_query
```

负载均衡器不再需要解析 JSON body 就能做基于操作类型的路由和限流。这让 MCP 流量可以走标准的 API 网关基础设施。

### 5.2 响应缓存（SEP-2549）

`tools/list`、`resources/list` 等端点现在返回 `ttlMs` 和 `cacheScope`：

```json
{
  "tools": [...],
  "_meta": {
    "ttlMs": 300000,
    "cacheScope": "public"
  }
}
```

客户端知道工具列表 5 分钟内有效，不需要每次调用都重新获取。这在高并发场景下显著降低了不必要的请求。

### 5.3 分布式追踪（SEP-414）

`_meta` 中正式支持 W3C Trace Context：`traceparent`、`tracestate`、`baggage`。一个从 Host 应用发起的 trace，能贯穿 Client SDK → MCP Server → 下游服务，在 OpenTelemetry 后端呈现为完整的 span 树。

## 六、Extensions 框架：协议的未来扩展模式

`2026-07-28` 引入了正式的 Extensions 机制。`ClientCapabilities` 和 `ServerCapabilities` 新增 `extensions` 字段，客户端和服务端通过它协商可选能力。

两个首批 Extensions 值得关注：

### 6.1 MCP Apps（SEP-1865）

服务器可以下发交互式 HTML UI，宿主在沙箱 iframe 中渲染。用户在 UI 中的操作通过标准 JSON-RPC 路径触发工具调用——审计和权限模型与直接工具调用完全一致。

**应用场景：** 在 IDE 中直接渲染数据库查询界面、OAuth 授权流程、复杂表单，而不需要切换到浏览器。

### 6.2 Tasks Extension（SEP-2663）

Tasks 从 `2025-11-25` 的实验性核心功能迁移为正式 Extension：
- 阻塞式的 `tasks/result` → 轮询式的 `tasks/get`
- 新增 `tasks/update` 供客户端向服务端提供输入
- 移除 `tasks/list`
- 服务端可以主动返回 task handle

## 七、什么被废弃了？

三个核心功能被标记为 **Deprecated**（SEP-2577）：

| 功能 | 状态 | 替代方案 |
|------|------|----------|
| Roots | Deprecated | 显式 handle 模式 |
| Sampling | Deprecated | Multi Round-Trip Requests |
| Logging | Deprecated | 标准 OpenTelemetry 集成 |

根据新的生命周期策略（SEP-2596），被废弃的功能至少保留 **12 个月**才可能被移除。所以不用恐慌，但也不要在新项目中重度依赖它们了。

## 八、7 月 28 日前的迁移检查清单

### 服务端开发者 ✅

- [ ] 移除 `initialize`/`initialized` 握手处理逻辑
- [ ] 实现 `server/discover` RPC 方法
- [ ] 从 `_meta` 读取协议版本和客户端能力
- [ ] 给 list/read 响应添加 `ttlMs` 和 `cacheScope`
- [ ] 如果使用 Tasks，迁移到 Extension 版本
- [ ] 验证网关能基于 `Mcp-Method`/`Mcp-Name` 路由

### 客户端开发者 ✅

- [ ] 每个请求携带 `_meta` 中的协议版本和客户端信息
- [ ] 发送 `Mcp-Method` 和 `Mcp-Name` 头部
- [ ] 处理 `InputRequiredResult` 响应类型
- [ ] 支持 `ttlMs` 缓存语义
- [ ] 移除 Session 管理代码

### 基础设施 ✅

- [ ] 移除粘性路由配置
- [ ] 下线共享 Session Store
- [ ] 更新网关规则：从 DPI 改为 Header 路由
- [ ] 确认 Round-Robin 负载均衡正常工作

## 九、这意味着什么？

MCP `2026-07-28` 的无状态化不只是一次技术升级——它是 AI Agent 基础设施的**成人礼**。

当协议层不再绑定连接状态，MCP 服务就能像普通的 REST API 一样部署在 Cloudflare Workers、AWS Lambda、Vercel Edge Functions 上。AI Agent 的工具调用不再是脆弱的长连接，而是坚固的、可重试的、可缓存的 HTTP 请求。

这让 MCP 真正具备了成为 **"AI 时代的 HTTP"** 的潜力：无状态、可组合、可扩展，任何规模的团队都能在标准基础设施上运行。

5 天后，协议正式发布。现在开始准备，还来得及。

---

**参考资料：**
- [MCP 2026-07-28 Release Candidate 官方公告](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)
- [Draft Changelog](https://modelcontextprotocol.io/specification/draft/changelog)
- [Feature Lifecycle Policy](https://modelcontextprotocol.io/community/feature-lifecycle)
- [MCP Specification Repository](https://github.com/modelcontextprotocol/modelcontextprotocol)
