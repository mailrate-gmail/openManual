# Apifox 导入说明

本文档说明如何使用 `docs/opencode-openapi-3.1.json` 导入 Apifox，以及当前文档中哪些接口适合做 Mock，哪些接口需要特别处理。

## 文件

- OpenAPI 文件：`docs/opencode-openapi-3.1.json`
- Postman 文件：`docs/opencode-postman-collection.json`

优先推荐在 Apifox 中导入：`docs/opencode-openapi-3.1.json`

## 导入步骤

1. 打开 Apifox。
2. 选择 `导入`。
3. 选择 `OpenAPI/Swagger`。
4. 导入 `docs/opencode-openapi-3.1.json`。
5. 导入后确认服务地址为：`http://localhost:4096`

如果你的本地服务不是默认端口，需要在 Apifox 项目环境里修改 `server url`。

## 鉴权设置

当前文档已经在 `components.securitySchemes` 中声明了 `BasicAuth`，但没有全局强制启用。

原因：

- OpenCode 本地默认可能不开密码。
- 只有启用了 `OPENCODE_SERVER_PASSWORD` 时，server mode 才需要 Basic Auth。

如果你启用了服务密码，建议在 Apifox 中这样配置：

1. 在项目环境中设置：
   - 用户名：`opencode`
   - 密码：你的 `OPENCODE_SERVER_PASSWORD`
2. 对需要调试的接口启用 `Basic Auth`

## 哪些接口适合做 Mock

最适合直接做 Mock 的接口：

- `/global/health`
- `/project`
- `/project/current`
- `/session`
- `/session/{sessionID}`
- `/session/{sessionID}/children`
- `/session/{sessionID}/todo`
- `/session/{sessionID}/message`
- `/session/{sessionID}/message/{messageID}`
- `/provider`
- `/provider/auth`
- `/mcp`
- `/permission`
- `/question`
- `/config/providers`
- `/experimental/console`
- `/experimental/console/orgs`

这些接口的返回结构相对稳定，适合前端文档展示、静态调试和基础 Mock。

## 不适合普通 Mock 的接口

以下接口虽然已经出现在 OpenAPI 中，但更适合真实联调，不适合只靠静态 Mock：

### SSE 接口

- `GET /global/event`
- `GET /global/sync-event`
- `GET /event`

原因：

- 返回类型是 `text/event-stream`
- 需要长连接
- 会持续发送 heartbeat 和事件消息

在 Apifox 中可以保留文档，但不要指望它完整模拟真实流式行为。

### WebSocket 接口

- `GET /pty/{ptyID}/connect`

原因：

- 这是 WebSocket 升级端点
- 真实行为依赖 PTY 会话状态和 cursor 增量输出

Apifox 中建议把它当“文档接口”而不是普通 Mock 接口。

### 有副作用的执行型接口

- `POST /session/{sessionID}/message`
- `POST /session/{sessionID}/prompt_async`
- `POST /session/{sessionID}/command`
- `POST /session/{sessionID}/shell`
- `POST /session/{sessionID}/revert`
- `POST /session/{sessionID}/summarize`
- `POST /mcp/{name}/auth/authenticate`
- `POST /global/upgrade`

原因：

- 会触发实际 AI 运行、工具执行、状态机变化、外部服务调用或进程行为

这些接口建议：

- 文档中保留
- 前端联调可临时 Mock
- 真正验收必须接真实服务

## 已标记废弃的接口

当前 OpenAPI 中应视为 deprecated 的接口：

- `POST /session/{sessionID}/init`
- `POST /session/{sessionID}/permissions/{permissionID}`

建议：

- 新客户端不要继续依赖这两个接口
- 旧系统兼容时可保留

## 当前仍为宽松 Schema 的接口

虽然这版 OpenAPI 已经比初版更细，但以下接口仍然使用了较宽松的 `AnyObject` 或代表性 schema：

- `/config`
- `/global/config`
- `/experimental/workspace`
- `/experimental/worktree`
- `/experimental/resource`
- `/tui/*` 的多数请求体
- `/auth/{providerID}` 的 OAuth 扩展字段
- `/session/{sessionID}/fork`
- `/session/{sessionID}/message` 的完整 message part 类型

原因：

- 这些结构依赖仓库内部复合 schema
- 某些字段是运行期动态扩展的
- 某些字段包含 provider-specific、tool-specific 或 UI event-specific 数据

结论：

- 它们足够支持 Apifox 展示与基础 Mock
- 但还不是“源码级 100% 严格镜像”

## Mock 使用建议

推荐按三层使用：

1. 文档层
   - 所有接口都导入，作为目录和说明

2. 静态 Mock 层
   - 优先 Mock 列表、详情、状态接口

3. 真实联调层
   - 执行型、流式、终端类接口必须走真实服务

## 建议的 Apifox 分组

建议导入后按下面分组整理：

- Global
- Project
- Session
- Message
- Provider
- MCP
- Permission
- Question
- File
- Config
- PTY
- Experimental
- TUI
- Streams

其中 `Streams` 可以单独放：

- `/global/event`
- `/global/sync-event`
- `/event`
- `/pty/{ptyID}/connect`

## 常见问题

### 1. 为什么有些接口返回示例比较宽泛？

因为这版目标是：

- 先保证路由完整
- 再保证常用结构可读
- 最后逐步收紧到更精确 schema

### 2. 为什么 SSE/WebSocket 在 Apifox 里不好测？

因为这些接口本质上不是普通 request/response。

### 3. 为什么有些请求体和源码不完全一一对应？

因为部分内部 schema 组合很复杂，当前文档对它们做了“更适合产品文档和 Mock”的抽象。

## 下一步建议

如果你要把这套文档进一步用于团队协作，建议继续做两件事：

1. 为核心接口补充统一错误码与错误示例。
2. 为 `session`、`message`、`provider`、`mcp` 做更严格的真实 schema 对齐。
