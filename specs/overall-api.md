# OpenCodeUI 外部 API 模型文档

> 本文档描述 OpenCodeUI 前端与 OpenCode 后端之间的通信协议、API 契约、集成模式及错误处理策略。完整的端点列表参见 [API.md](./API.md)。

## 1. 通信协议概览

OpenCodeUI 前端与后端之间使用三种通信协议，各自承担不同的职责：

| 协议                         | 用途                                                    | 方向       | 持久性 |
| ---------------------------- | ------------------------------------------------------- | ---------- | ------ |
| **REST API** (HTTP/HTTPS)    | CRUD 操作：会话、消息、文件、配置、权限等               | 请求-响应  | 短连接 |
| **SSE** (Server-Sent Events) | 实时事件流：状态变更、消息增量、权限请求等 38+ 事件类型 | 服务端推送 | 长连接 |
| **WebSocket**                | 终端 PTY 双向 I/O                                       | 双向       | 长连接 |

### 1.1 URL 路径约定

后端 API 路径不带前缀（如 `/session`、`/file`）。前端在不同部署模式下通过不同方式映射：

- **开发模式**：Vite 代理将 `/api/*` 重写为 `/*` 转发至 `http://127.0.0.1:4096`
- **Docker 模式**：Caddy 反向代理将 `/api/*` 转发至后端 `:4096`
- **Tauri 桌面模式**：前端直接请求后端 URL，不走浏览器代理
- **独立部署模式**：前端直接连接配置的后端地址

### 1.2 多目录支持

几乎所有目录级端点都接受 `directory` 查询参数，用于标识当前工作项目路径。前端通过 `formatPathForApi()` 统一处理路径格式，确保跨平台兼容。

```
GET /session?directory=%2Fpath%2Fto%2Fproject
```

## 2. 认证机制

### 2.1 Basic Auth（服务器访问）

前端通过 `serverStore` 管理多服务器配置，每个服务器可设置用户名和密码。认证采用 HTTP Basic Auth 方案：

- **REST 请求**：通过 `Authorization: Basic <base64>` 请求头传递，由 SDK client 的 `headers` 参数统一注入
- **SSE 连接**：浏览器模式通过 `fetch` 请求头传递；Tauri 模式通过 Rust 侧 `reqwest` 的 `authHeader` 参数传递
- **WebSocket 连接**：WebSocket 不支持自定义请求头，认证凭据通过 URL 的 userinfo 部分传递（`ws://user:pass@host/pty/{id}/connect`）

认证凭据的缓存键为 `baseUrl|username:password`，切换服务器或修改密码时自动重建 SDK client 实例。

### 2.2 OAuth 流程（AI 提供商认证）

后端支持通过 OAuth 为 AI 提供商（如 Anthropic、Google 等）进行认证，流程如下：

1. **获取认证方式**：`GET /provider/auth` 返回各提供商可用的认证方法列表
2. **发起授权**：`POST /provider/{providerID}/oauth/authorize` 返回授权 URL 和方式
3. **处理回调**：`POST /provider/{providerID}/oauth/callback` 接收 OAuth 授权码并完成认证

前端在设置面板中引导用户打开授权页面，接收回调后提交授权码。

### 2.3 API Key 认证（提供商配置）

AI 提供商的 API Key 通过 `PUT /auth/{providerID}` 设置，通过 `DELETE /auth/{providerID}` 移除。这些凭据存储在后端配置中，前端不直接持有。

### 2.4 MCP OAuth 认证

MCP 服务器支持独立的 OAuth 流程：

- `POST /mcp/{name}/auth` — 启动 OAuth，返回授权 URL
- `POST /mcp/{name}/auth/callback` — 完成 OAuth，提交授权码
- `POST /mcp/{name}/auth/authenticate` — 启动 OAuth 并等待回调（自动打开浏览器）
- `DELETE /mcp/{name}/auth` — 移除 OAuth 凭据

## 3. API 客户端架构

### 3.1 SDK 封装层（`@opencode-ai/sdk`）

前端使用 `@opencode-ai/sdk` 作为主要 API 客户端，通过 `createOpencodeClient()` 创建实例。SDK 封装了绝大部分 REST 端点，提供类型安全的调用方式。

**Client 管理策略：**

- 按 `baseUrl + authHash` 缓存实例，避免重复创建
- 服务器切换时调用 `invalidateSDKClient()` 强制重建
- Tauri 环境下使用 `@tauri-apps/plugin-http` 的 fetch 替代浏览器原生 fetch

**结果解包：**

SDK 返回 `{ data, error, request, response }` 结构，前端通过 `unwrap()` 工具函数提取 `data` 或在有 `error` 时抛出异常，使上层 API 函数直接返回数据。

### 3.2 原始 HTTP 辅助函数

`src/api/http.ts` 提供 SSE 和 WebSocket 所需的 URL 构建辅助：

| 函数                 | 用途                          |
| -------------------- | ----------------------------- |
| `getApiBaseUrl()`    | 获取当前 API 基础 URL         |
| `getAuthHeader()`    | 获取 Authorization 请求头对象 |
| `buildQueryString()` | 构建 URL 编码的查询字符串     |

这些函数不执行 HTTP 请求，仅负责 URL 和认证信息的拼接。

### 3.3 目录作用域请求

所有目录级 API 调用通过 `formatPathForApi(directory)` 处理路径参数。该函数将本地文件系统路径转换为后端可识别的格式，支持多项目工作区切换。

### 3.4 错误处理模式

- **SDK 层**：`unwrap()` 将 SDK 的 `{ data, error }` 结构转换为直接返回值或抛出异常
- **HTTP 错误类型**：后端返回标准 HTTP 状态码，包括 `BadRequestError` (400)、`NotFoundError` (404) 等
- **类型化错误**：SSE 事件中的 `SessionErrorPayload` 包含 `sessionID`、`name` 和 `data`，前端通过 `normalizeSessionError()` 统一格式化

## 4. 事件流架构（SSE）

### 4.1 连接端点

| 端点                | 作用域 | 说明                                 |
| ------------------- | ------ | ------------------------------------ |
| `GET /global/event` | 全局   | 订阅全局事件，所有目录的事件均会推送 |

前端采用**单例模式**管理 SSE 连接，多个订阅者共享同一个连接实例。当第一个订阅者注册时建立连接，最后一个订阅者移除时断开连接。

### 4.2 事件类型（38+ 种）

事件按功能分为以下类别：

**会话事件：**

- `session.created` / `session.updated` / `session.deleted` — 会话生命周期
- `session.idle` — 会话进入空闲状态
- `session.error` — 会话执行出错
- `session.status` — 会话状态变更（active/idle/completed）
- `session.compacted` — 会话上下文压缩完成
- `session.diff` — 消息产生的文件变更

**消息事件：**

- `message.updated` — 消息元数据更新
- `message.removed` — 消息被移除
- `message.part.updated` — 消息片段更新
- `message.part.delta` — 消息片段增量数据（流式输出）
- `message.part.removed` — 消息片段被删除

**权限事件：**

- `permission.asked` — AI 请求操作权限
- `permission.replied` — 权限已被回复

**问答事件：**

- `question.asked` — AI 提出问题需要用户回答
- `question.replied` — 用户已回答
- `question.rejected` — 用户已拒绝回答

**待办事件：**

- `todo.updated` — 待办列表更新

**项目事件：**

- `project.updated` — 项目信息变更

**文件事件：**

- `file.edited` — 文件被编辑
- `file.watcher.updated` — 文件监视器触发

**终端事件：**

- `pty.created` / `pty.updated` / `pty.exited` / `pty.deleted` — PTY 生命周期

**Worktree 事件：**

- `worktree.ready` — 工作树就绪
- `worktree.failed` — 工作树创建失败

**VCS 事件：**

- `vcs.branch.updated` — Git 分支变更

**MCP 事件：**

- `mcp.tools.changed` — MCP 工具列表变更
- `mcp.browser.open.failed` — MCP 浏览器打开失败

**LSP 事件：**

- `lsp.client.diagnostics` — LSP 诊断信息
- `lsp.updated` — LSP 状态更新

**服务器事件：**

- `server.connected` — 服务器已连接
- `server.instance.disposed` — 实例已释放
- `global.disposed` — 全局已释放

**TUI 事件：**

- `tui.prompt.append` / `tui.command.execute` / `tui.toast.show` / `tui.session.select`

**其他事件：**

- `installation.updated` / `installation.update-available` — 安装状态
- `workspace.ready` / `workspace.failed` — 工作区状态
- `command.executed` — 命令执行完成

### 4.3 事件 → Store 更新映射

前端通过 `EventCallbacks` 接口将 SSE 事件映射到对应的处理函数：

```
SSE 事件                  → 回调函数                → Store 更新
─────────────────────────────────────────────────────────────
message.updated          → onMessageUpdated        → 更新消息列表
message.part.updated     → onPartUpdated           → 更新消息片段
message.part.delta       → onPartDelta             → 追加流式文本
session.created          → onSessionCreated        → 添加会话
session.updated          → onSessionUpdated        → 更新会话
session.deleted          → onSessionDeleted        → 移除会话
todo.updated             → onTodoUpdated           → 更新待办
permission.asked         → onPermissionAsked       → 显示权限弹窗
question.asked           → onQuestionAsked         → 显示问答弹窗
...
```

### 4.4 重连策略

SSE 连接采用**指数退避**重连机制：

| 尝试次数 | 前台延迟 | 后台延迟 |
| -------- | -------- | -------- |
| 1        | 1s       | 500ms    |
| 2        | 2s       | 1s       |
| 3        | 3s       | 2s       |
| 4        | 5s       | 3s       |
| 5        | 10s      | 5s       |
| 6+       | 30s      | 10s      |

**心跳机制：** 每次收到事件后重置心跳计时器。前台 60 秒无事件视为超时，后台 120 秒。超时后自动触发重连。

**后台保活：** 页面进入后台时启动 30 秒间隔的 keepalive 轮询，检测连接是否静默断开。

**生命周期监听：**

- `visibilitychange` — 页面恢复前台时检查连接状态，必要时立即重连
- `online` / `offline` — 网络恢复时强制重连，网络断开时标记为断连但不尝试重连

**连接代次机制：** 每次重连递增 `connectionGeneration`，旧代次的事件自动丢弃，防止过期事件污染状态。

### 4.5 Tauri 桥接

Tauri 桌面环境下，SSE 通过 Rust 侧的 `reqwest` + `Channel` 实现桥接：

1. 前端调用 `invoke('sse_connect', { url, authHeader, onEvent })`
2. Rust 侧建立 SSE 连接，通过 `Channel` 实时推送事件
3. 前端通过 `onEvent.onmessage` 接收 `{ event, data }` 消息
4. 断开时调用 `invoke('sse_disconnect')`

Tauri 侧的 disconnect 和 connect 操作通过 Promise 串行化，避免竞争条件。

## 5. WebSocket 协议（PTY 终端）

### 5.1 连接生命周期

PTY WebSocket 遵循以下流程：

```
1. REST 创建 PTY 会话  →  POST /pty           →  返回 { id, ... }
2. REST 标记连接状态    →  GET  /pty/{id}/connect
3. 建立 WebSocket 连接  →  ws://host/pty/{id}/connect?directory=...
4. 双向 I/O 通信       →  发送输入 / 接收输出
5. 关闭连接            →  WebSocket close 或 REST DELETE /pty/{id}
```

### 5.2 WebSocket URL 构建

WebSocket 不支持自定义请求头，认证通过 URL userinfo 传递：

```
ws://username:password@host/pty/{ptyID}/connect?directory=%2Fpath
```

无认证时直接连接：

```
ws://host/pty/{ptyID}/connect?directory=%2Fpath
```

前端通过 `getPtyConnectUrl()` 函数自动拼接完整 URL，处理 HTTP→WS 协议转换、认证凭据编码和查询参数构建。

### 5.3 数据传输

- **输入**：前端将键盘输入作为二进制数据通过 WebSocket 发送
- **输出**：后端通过 WebSocket 推送终端输出的原始字节流
- **尺寸调整**：通过 `PUT /pty/{ptyID}` 更新 `size: { rows, cols }` 通知后端终端尺寸变化

### 5.4 重连策略

WebSocket 断连时，前端根据 PTY 会话状态决定是否重连。如果 PTY 会话已被删除（收到 `pty.deleted` 事件），则不再尝试重连。

## 6. API 代理配置

### 6.1 开发环境（Vite）

```typescript
// vite.config.ts
proxy: {
  '/api': {
    target: 'http://127.0.0.1:4096',
    changeOrigin: true,
    ws: true,
    rewrite: path => path.replace(/^\/api/, ''),
  },
}
```

- 所有 `/api/*` 请求转发至后端 `:4096`
- `/api` 前缀在转发时被剥离
- 支持 WebSocket 代理（`ws: true`）
- Tauri 模式下不走此代理，直接通过 `plugin-http` 请求后端

### 6.2 Docker 部署（Caddy）

```caddyfile
opencode.example.com {
    reverse_proxy 127.0.0.1:6658 {
        flush_interval -1
    }
}
```

Gateway（Caddy）统一监听 `:6658`，路由规则：

| 路径         | 转发目标         | 说明                   |
| ------------ | ---------------- | ---------------------- |
| `/api/*`     | Backend `:4096`  | OpenCode API，支持 SSE |
| `/routes`    | Router `:7070`   | 动态路由管理           |
| `/preview/*` | Router `:7070`   | 预览端口               |
| 其他         | Frontend `:3000` | 前端静态资源           |

**SSE 要求**：`flush_interval -1` 禁用缓冲，确保事件实时推送。

### 6.3 独立部署

前端容器通过 `BACKEND_URL` 环境变量指定后端地址，直接连接后端 API，不经过代理层。

### 6.4 公网反向代理注意事项

Nginx 配置需特别注意：

```nginx
# SSE：禁用缓冲
proxy_buffering off;
proxy_cache off;
proxy_set_header Connection '';

# WebSocket
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";

# 长超时（SSE 和 WebSocket 都是长连接）
proxy_read_timeout 86400s;
```

## 7. 错误处理

### 7.1 HTTP 错误类型

后端使用标准 HTTP 状态码配合类型化错误响应：

| 状态码 | 错误类型          | 常见场景                                    |
| ------ | ----------------- | ------------------------------------------- |
| 400    | `BadRequestError` | 参数校验失败、请求体格式错误                |
| 404    | `NotFoundError`   | 会话/消息/PTY 不存在                        |
| 204    | —                 | 异步操作已接受（如 `session.prompt_async`） |

### 7.2 SSE 错误处理

- **连接失败**：`fetch` 返回非 200 状态码时触发重连
- **流中断**：`reader.read()` 返回 `done: true` 时触发重连
- **解析错误**：无法解析的 JSON 事件被静默丢弃，记录警告日志
- **心跳超时**：超过阈值无事件时主动断开并重连
- **网络离线**：标记为断连状态，不尝试重连（等待 `online` 事件）

### 7.3 WebSocket 错误处理

- **连接失败**：WebSocket `onerror` 事件触发，前端根据 PTY 状态决定是否重试
- **异常断开**：WebSocket `onclose` 事件触发，如果 PTY 会话仍存在则尝试重连
- **PTY 已删除**：收到 `pty.deleted` SSE 事件后，关闭对应 WebSocket 连接并不再重连

### 7.4 会话错误事件

后端通过 `session.error` SSE 事件推送运行时错误，前端通过 `normalizeSessionError()` 统一处理：

```typescript
interface SessionErrorPayload {
  sessionID: string // 出错的会话 ID
  name: string // 错误名称（如 "ProviderError"）
  data: unknown // 错误详情
}
```

前端在聊天界面中展示错误提示，允许用户重试或切换模型。
