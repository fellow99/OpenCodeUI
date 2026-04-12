# 001-api-layer 技术规划（As-Built）

> 本文档是对 `src/api/` 模块的回溯性技术规划，记录"实际建成"的架构设计、技术决策与实现策略。
> 模块编号：001-api-layer
> 对应 spec：`specs-plan-b/001-api-layer/spec.md`
> 最后更新：2026-04-12

---

## 1. Technical Context

### 1.1 运行环境

| 环境       | 说明                                                                                                       |
| ---------- | ---------------------------------------------------------------------------------------------------------- |
| 浏览器     | 标准 Web 环境，使用 `fetch` API 进行 HTTP 通信，`ReadableStream` 读取 SSE                                  |
| Tauri 桌面 | macOS / Linux / Windows，通过 `@tauri-apps/plugin-http` 提供原生 fetch，通过 `invoke` + `Channel` 桥接 SSE |
| 开发环境   | Vite dev server 将 `/api` 代理至 `http://127.0.0.1:4096`                                                   |
| 生产环境   | 静态前端由 Caddy 托管，`/api/*` 由 Gateway 反代至后端 :4096 端口                                           |

### 1.2 依赖关系

**直接依赖：**

| 依赖                      | 版本    | 用途                                          |
| ------------------------- | ------- | --------------------------------------------- |
| `@opencode-ai/sdk`        | ^1.4.1  | OpenCode 官方 SDK，提供 REST 客户端与类型定义 |
| `@tauri-apps/api`         | ^2.10.1 | Tauri 核心 API（`invoke`、`Channel`）         |
| `@tauri-apps/plugin-http` | ^2.5.7  | Tauri 原生 HTTP 插件（替代浏览器 fetch）      |
| `../store/serverStore`    | 内部    | 多服务器配置管理、活动服务器追踪、健康检查    |
| `../utils/directoryUtils` | 内部    | 路径模式检测、`formatPathForApi()` 路径格式化 |
| `../utils/tauri`          | 内部    | `isTauri()` 环境检测                          |
| `../types/api/event`      | 内部    | `EventTypes` 常量、`EventCallbacks` 接口      |
| `../i18n`                 | 内部    | 命令描述的本地化（`command.ts` 使用）         |
| `../features/attachment`  | 内部    | `fromFilePart`、`fromAgentPart` 附件转换      |

**间接依赖（通过 serverStore）：**

| 依赖                 | 用途                         |
| -------------------- | ---------------------------- |
| `../constants`       | `API_BASE_URL` 默认后端地址  |
| `./perServerStorage` | 每服务器独立存储（路径模式） |

### 1.3 集成点

**上游（本模块依赖）：**

- `serverStore`：提供 `getActiveBaseUrl()`、`getActiveAuth()`、`getActiveServerId()`、`makeBasicAuthHeader()`
- `directoryUtils`：提供 `formatPathForApi()` 路径格式化、路径模式自动检测
- `tauri.ts`：提供 `isTauri()` 环境检测

**下游（依赖本模块）：**

- 所有 `src/features/` 业务模块：通过 `@/api/client` 统一导入
- `src/store/` 状态管理层：通过 API 函数获取初始数据，通过 SSE 事件驱动更新
- `src/components/Terminal/`：通过 `getPtyConnectUrl()` 获取 WebSocket 连接地址

### 1.4 约束条件

- SDK 版本锁定为 `@opencode-ai/sdk` v1.4.1，所有 API 调用必须通过 SDK 而非自行封装 HTTP
- 桌面环境中 SSE 必须通过 Rust 侧 `reqwest` + `Channel` 桥接，不能使用浏览器 `fetch`
- WebSocket 不支持自定义请求头，认证信息必须通过 URL userinfo 传递
- 所有 API 函数的 `directory` 参数必须经过 `formatPathForApi()` 处理

---

## 2. Constitution Check

### 原则 1：AI 驱动开发（Vibe Coding）

**合规**。本模块代码结构清晰、注释充分，每个文件顶部均有职责说明。代码通过 AI 对话生成，人工审查后提交。

### 原则 2：OpenCode 兼容性优先

**合规**。所有 REST 调用均通过 `@opencode-ai/sdk` 发起，未自行封装 HTTP 请求。SDK 方法名与后端 API 路径一一对应（如 `sdk.session.list()` 对应 `GET /session`）。WebSocket URL 构建遵循后端约定的 `/pty/{ptyId}/connect` 路径格式。

### 原则 3：多平台统一代码库

**合规**。`src/api/` 为 Web 端与 Tauri 端共享代码。平台差异通过 `isTauri()` 条件分支处理：

- HTTP 传输：浏览器用 `fetch`，Tauri 用 `@tauri-apps/plugin-http`
- SSE 连接：浏览器用 `fetch` + `ReadableStream`，Tauri 用 `invoke('sse_connect')` + `Channel`
- 核心业务逻辑（session、message、permission 等）完全共享

### 原则 4：自定义优于框架依赖

**合规**。本模块未引入额外的 HTTP 客户端库（如 axios），直接使用 SDK 提供的客户端。SSE 连接管理为自定义单例实现，未使用第三方 SSE 库。缓存机制（根目录缓存、命令缓存）为手写实现。

### 原则 5：实时通信优先

**合规**。SSE 是本模块的核心通信机制之一（`events.ts`），采用全局单例连接、多订阅者共享模式。所有实时数据更新（消息流式输出、会话状态变更、权限请求等）均通过 SSE 推送，轮询仅用于后台 keepalive 检查。

### 原则 6：中文优先文档

**合规**。所有代码注释、JSDoc 均为中文，技术术语保留英文原文。

### 原则 7：开源与社区驱动

**合规**。代码遵循 GPL-3.0 许可证，通过 CI 校验（TypeScript 检查、ESLint、单元测试、生产构建）。

### 原则 8：零配置用户体验

**合规**。`serverStore` 默认配置本地服务器（`API_BASE_URL`），用户无需手动配置即可使用。SDK 客户端自动处理认证信息注入。

### 原则 9：主题与可访问性

**不直接相关**。本模块为纯通信层，不涉及 UI 渲染。

### 原则 10：模块化功能架构

**合规**。`src/api/` 按业务域拆分为 18 个独立文件，每个文件职责单一。`client.ts` 作为 barrel 文件统一导出，上层只需一个导入路径。

### 约束条件检查

| 约束          | 状态 | 说明                                                                     |
| ------------- | ---- | ------------------------------------------------------------------------ |
| C1 许可证     | 合规 | GPL-3.0                                                                  |
| C2 后端依赖   | 合规 | 完全通过 `@opencode-ai/sdk` 与 OpenCode 后端通信                         |
| C3 构建校验   | 合规 | 通过 `npm run validate`                                                  |
| C4 依赖最小化 | 合规 | 仅依赖 SDK 和 Tauri 插件，无额外 HTTP 库                                 |
| C5 SSE 支持   | 合规 | 浏览器端 `fetch` + `ReadableStream`，Tauri 端 Rust `reqwest`，均支持流式 |

---

## 3. Research Findings

### 3.1 SDK 客户端缓存策略

**决策**：按 `baseUrl + authHash` 组合缓存 `OpencodeClient` 实例。

**实现细节**：

- 缓存键格式：`${baseUrl}|${username}:${password}`（无密码时 authPart 为空字符串）
- 全局单例缓存：`_cachedClient` + `_cachedKey` 两个模块级变量
- 缓存命中时直接返回，未命中时调用 `createOpencodeClient()` 重建

** rationale**：

- SDK 客户端创建成本较高（内部初始化拦截器、类型校验等）
- 相同服务器 + 认证信息的请求应复用同一客户端实例
- 服务器切换时通过 `invalidateSDKClient()` 强制失效

**替代方案考虑**：

- Map 多实例缓存：未采用，因为同一时刻只有一个活动服务器
- 每次创建新实例：未采用，性能开销过大

### 3.2 Tauri fetch 懒加载

**决策**：`@tauri-apps/plugin-http` 采用动态 import 懒加载，不阻塞应用启动。

**实现细节**：

- `_tauriFetch` / `_tauriFetchLoading` 两个模块级变量实现加载状态追踪
- `getTauriFetch()` 使用 Promise 链确保并发调用共享同一加载过程
- `getSDKClient()` 同步版本：如果 `_tauriFetch` 已加载则注入，否则不传（使用默认 fetch）
- `getSDKClientAsync()` 异步版本：先 await `getTauriFetch()`，再使缓存失效重建

**rationale**：

- 应用初始化时 Tauri 插件可能尚未就绪
- 同步版本保证即使插件未加载，已有请求也能通过默认 fetch 发出
- 异步版本确保在应用初始化完成后使用原生 HTTP 传输

### 3.3 unwrap 模式

**决策**：提供统一的 `unwrap<T>()` 函数，将 SDK 的 `{ data, error }` 格式转换为直接返回 data 或抛出 error。

**实现细节**：

- 检查 `result.error` 是否为 `null` 或 `undefined`
- 错误处理分三种情况：`Error` 实例直接抛出、字符串包装为 `Error`、对象 JSON 序列化后抛出
- 成功时返回 `result.data as T`（类型断言）

**rationale**：

- SDK 设计为始终返回 `{ data, error, request, response }` 结构
- 上层业务逻辑期望直接获取数据或通过 try/catch 处理异常
- 统一解包避免每个调用点重复编写 `if (result.error) throw ...` 逻辑

### 3.4 SSE 单例连接架构

**决策**：全局维护唯一的 SSE 连接，所有订阅者共享。

**实现细节**：

- `allSubscribers`：`Set<EventCallbacks>` 存储所有订阅者回调
- `subscribeToEvents()`：首个订阅者触发 `connectSingleton()`，最后一个订阅者触发 `disconnectSingleton()`
- 连接代次机制：`connectionGeneration` 每次重连递增，旧代次的事件自动丢弃
- 防重入：`isConnecting` 标志位防止并发连接

**rationale**：

- 后端对每个目录仅维护一条事件流，多连接无额外收益
- 减少服务器资源消耗和网络连接数
- 简化连接状态管理，避免多连接间的事件重复和乱序

### 3.5 双环境 SSE 桥接

**决策**：浏览器环境使用 `fetch` + `ReadableStream`，Tauri 环境使用 `invoke('sse_connect')` + `Channel`。

**实现细节**：

- 浏览器端：标准 SSE 解析（`data:` 行积累、空行 dispatch、CRLF 兼容）
- Tauri 端：Rust 侧 `reqwest` 发起 SSE 请求，通过 `Channel<TauriSseEvent>` 推送事件
- 断开串行化：`pendingDisconnect` Promise 链确保 Tauri 侧 `disconnect → connect` 顺序执行

**rationale**：

- Tauri 的 `plugin-http` 对 SSE 流式读取支持有限
- Rust 侧 `reqwest` 对 SSE 有更好的原生支持
- Channel 机制提供可靠的事件推送通道

### 3.6 重连策略

**决策**：分级指数退避，前台/后台差异化策略。

**实现细节**：

- 前台延迟：`[1s, 2s, 3s, 5s, 10s, 30s]`
- 后台延迟：`[500ms, 1s, 2s, 3s, 5s, 10s]`（更激进）
- 心跳超时：前台 60s，后台 120s
- 后台 keepalive：每 30s 检查一次连接活性

**rationale**：

- 后台时用户感知延迟更高，需要更快恢复连接
- 后台 timer 可能被系统冻结，需要更宽松的心跳超时
- keepalive 轮询弥补 timer 冻结导致的重连延迟

### 3.7 生命周期感知

**决策**：监听 `visibilitychange`、`online`、`offline` 事件，自动调整连接行为。

**实现细节**：

- 页面可见：检查连接活性，超时则强制重连
- 页面隐藏：切换后台模式（更长心跳超时、启动 keepalive 轮询）
- 网络恢复：若当前未连接，立即重连
- 网络断开：断开现有连接，停止重连计时器

**rationale**：

- 移动端后台 timer 可能被系统冻结
- 网络恢复后应立即重建连接，无需等待用户操作
- 网络断开时停止无意义的重连尝试

### 3.8 目录缓存与并发去重

**决策**：根目录列表结果按 "服务器 ID + 目录" 组合缓存，TTL 10 秒，并发请求去重。

**实现细节**：

- `rootDirectoryCache`：`Map<string, { data, expiresAt }>` 缓存
- `rootDirectoryInflight`：`Map<string, Promise<FileNode[]>>` 进行中的请求
- 缓存键：`${serverStore.getActiveServerId()}::${formatPathForApi(directory) ?? ''}`
- 并发请求同一 key 时，返回同一个 Promise

**rationale**：

- 根目录列表在文件浏览器初始化时可能被多次请求
- 10 秒 TTL 平衡了数据新鲜度与网络开销
- 并发去重避免同一时刻发起多个相同请求

### 3.9 命令缓存与 i18n 集成

**决策**：命令列表缓存 TTL 10 秒，缓存键包含语言标识。

**实现细节**：

- 缓存键：`${serverStore.getActiveServerId()}::${i18n.resolvedLanguage || i18n.language}::${directory}`
- 前端命令（`new`、`compact`）与 API 命令合并，API 命令优先
- API 不可用时降级为仅前端命令

**rationale**：

- 命令描述通过 i18n 本地化，语言切换后需要重新获取
- 前端命令不依赖后端，确保后端不可用时斜杠命令仍可用

### 3.10 WebSocket URL 构建

**决策**：手动拼接 WebSocket URL，SDK 不处理 WebSocket。

**实现细节**：

- HTTP base URL 协议转换：`http` → `ws`，`https` → `wss`
- 认证信息通过 URL userinfo 传递：`ws://user:pass@host/...`
- 目录参数通过查询字符串传递：`?directory=...`
- username 和 password 均进行 `encodeURIComponent` 编码

**rationale**：

- WebSocket 协议不支持自定义请求头
- URL userinfo 是标准的认证传递方式（RFC 3986）
- 后端已支持从 URL 解析认证信息

---

## 4. Data Model

### 4.1 SDK 客户端实例

| 字段      | 类型                   | 说明                                                  |
| --------- | ---------------------- | ----------------------------------------------------- |
| `baseUrl` | string                 | 后端服务地址（来自 `serverStore.getActiveBaseUrl()`） |
| `headers` | Record<string, string> | 请求头，含可选的 `Authorization`                      |
| `fetch`   | typeof fetch           | HTTP 传输函数（浏览器 fetch 或 Tauri 原生 fetch）     |

**缓存结构**：

```
_cachedClient: OpencodeClient | null
_cachedKey: string  // 格式: "${baseUrl}|${username}:${password}"
```

### 4.2 连接状态

| 字段               | 类型            | 说明                                                  |
| ------------------ | --------------- | ----------------------------------------------------- |
| `state`            | ConnectionState | `connecting` / `connected` / `disconnected` / `error` |
| `lastEventTime`    | number          | 最后一次收到事件的时间戳（毫秒）                      |
| `reconnectAttempt` | number          | 当前重连尝试次数                                      |
| `error`            | string?         | 错误描述信息                                          |

**内部状态变量**：

```
singletonController: AbortController | null   // 浏览器端 fetch 控制器
heartbeatTimer: ReturnType<typeof setTimeout> | null
reconnectTimer: ReturnType<typeof setTimeout> | null
keepaliveTimer: ReturnType<typeof setInterval> | null
isConnecting: boolean
connectionGeneration: number                  // 连接代次
isInBackground: boolean
isServerSwitch: boolean
pendingDisconnect: Promise<void>              // Tauri 断开操作串行化
```

### 4.3 事件订阅

| 字段             | 类型                   | 说明                    |
| ---------------- | ---------------------- | ----------------------- |
| `allSubscribers` | Set<EventCallbacks>    | 所有订阅者的回调集合    |
| `EventTypes`     | Record<string, string> | 45 种事件类型的常量映射 |

**EventCallbacks 接口**（20+ 个可选回调）：

- `onMessageUpdated`、`onPartUpdated`、`onPartDelta`、`onPartRemoved`
- `onSessionCreated`、`onSessionUpdated`、`onSessionDeleted`、`onSessionIdle`、`onSessionError`、`onSessionStatus`
- `onPermissionAsked`、`onPermissionReplied`
- `onQuestionAsked`、`onQuestionReplied`、`onQuestionRejected`
- `onTodoUpdated`、`onProjectUpdated`
- `onWorktreeReady`、`onWorktreeFailed`、`onVcsBranchUpdated`
- `onError`、`onReconnected`

### 4.4 缓存结构

**根目录缓存**：

```
rootDirectoryCache: Map<string, { data: FileNode[]; expiresAt: number }>
rootDirectoryInflight: Map<string, Promise<FileNode[]>>
TTL: 10,000ms
缓存键: "${serverId}::${directory}"
```

**命令缓存**：

```
commandCache: Map<string, { data: Command[]; expiresAt: number }>
commandInflight: Map<string, Promise<Command[]>>
TTL: 10,000ms
缓存键: "${serverId}::${language}::${directory}"
```

### 4.5 PTY 连接 URL

| 组成部分 | 来源                                                         | 说明                               |
| -------- | ------------------------------------------------------------ | ---------------------------------- |
| 协议     | `httpBase.replace(/^http/, 'ws')`                            | http→ws, https→wss                 |
| userinfo | `encodeURIComponent(username):encodeURIComponent(password)@` | 仅当配置了密码时                   |
| host     | `serverStore.getActiveBaseUrl()` 的主机部分                  | 后端地址                           |
| path     | `/pty/{ptyId}/connect`                                       | 固定路径格式                       |
| query    | `?directory={encoded}`                                       | 可选，经 `formatPathForApi()` 处理 |

### 4.6 Todo 标准化

| 字段       | 类型   | 说明                                                                              |
| ---------- | ------ | --------------------------------------------------------------------------------- |
| `id`       | string | 合成 ID：`todo-{index}-{content}-{status}-{priority}`                             |
| `content`  | string | 待办内容                                                                          |
| `status`   | enum   | `pending` / `in_progress` / `completed` / `cancelled`（非法值归一化为 `pending`） |
| `priority` | enum   | `high` / `medium` / `low`（非法值归一化为 `medium`）                              |

---

## 5. Interface Contracts

### 5.1 模块入口

```typescript
// src/api/index.ts
export * from './client'
```

模块入口仅一行代码，将所有导出委托给 `client.ts`。

### 5.2 聚合层（client.ts）

`client.ts` 作为 barrel 文件，执行三类导出：

1. **类型 re-export**：`export * from './types'`
2. **附件功能 re-export**：`export { fromFilePart, fromAgentPart } from '../features/attachment'`
3. **子模块 re-export**：16 个子模块的全部导出

此外，`client.ts` 自身定义了以下函数：

| 函数名              | 参数                            | 返回类型                          | 说明             |
| ------------------- | ------------------------------- | --------------------------------- | ---------------- |
| `getActiveModels`   | `directory?: string`            | `Promise<ModelInfo[]>`            | 获取活跃模型列表 |
| `getDefaultModels`  | `directory?: string`            | `Promise<Record<string, string>>` | 获取默认模型映射 |
| `getCurrentProject` | `directory?: string`            | `Promise<ApiProject>`             | 获取当前项目     |
| `getProjects`       | `directory?: string`            | `Promise<ApiProject[]>`           | 获取项目列表     |
| `initGitProject`    | `directory?: string`            | `Promise<ApiProject>`             | 初始化 Git 仓库  |
| `updateProject`     | `projectId, params, directory?` | `Promise<ApiProject>`             | 更新项目         |
| `getPath`           | 无                              | `Promise<ApiPath>`                | 获取路径信息     |

### 5.3 SDK 客户端层（sdk.ts）

| 函数名                | 参数                                    | 返回类型                  | 说明                              |
| --------------------- | --------------------------------------- | ------------------------- | --------------------------------- |
| `getSDKClient`        | 无                                      | `OpencodeClient`          | 同步获取 SDK 客户端               |
| `getSDKClientAsync`   | 无                                      | `Promise<OpencodeClient>` | 异步获取（确保 Tauri fetch 就绪） |
| `invalidateSDKClient` | 无                                      | `void`                    | 强制使缓存失效                    |
| `unwrap<T>`           | `result: { data?: T; error?: unknown }` | `T`                       | 从 SDK 返回值提取 data            |

### 5.4 HTTP 工具（http.ts）

| 函数名             | 参数                                 | 返回类型                 | 说明                      |
| ------------------ | ------------------------------------ | ------------------------ | ------------------------- |
| `getApiBaseUrl`    | 无                                   | `string`                 | 获取当前 API Base URL     |
| `getAuthHeader`    | 无                                   | `Record<string, string>` | 获取 Authorization header |
| `buildQueryString` | `params: Record<string, QueryValue>` | `string`                 | 构建查询字符串            |

### 5.5 事件系统（events.ts）

| 函数名                       | 参数                        | 返回类型         | 说明             |
| ---------------------------- | --------------------------- | ---------------- | ---------------- |
| `getConnectionInfo`          | 无                          | `ConnectionInfo` | 获取当前连接状态 |
| `subscribeToConnectionState` | `fn: (info) => void`        | `() => void`     | 订阅连接状态变化 |
| `subscribeToEvents`          | `callbacks: EventCallbacks` | `() => void`     | 订阅 SSE 事件    |
| `reconnectSSE`               | 无                          | `void`           | 强制重连 SSE     |

### 5.6 会话管理（session.ts）

| 函数名               | 参数                                        | 返回类型                    | 说明                        |
| -------------------- | ------------------------------------------- | --------------------------- | --------------------------- |
| `getSessionStatus`   | `directory?`                                | `Promise<SessionStatusMap>` | 获取所有会话状态            |
| `getSessionDiff`     | `sessionId, directory?, messageId?`         | `Promise<FileDiff[]>`       | 获取会话 diff               |
| `getLastTurnDiff`    | `sessionId, directory?`                     | `Promise<FileDiff[]>`       | 获取最后一轮用户消息的 diff |
| `getSessions`        | `params: SessionListParams`                 | `Promise<ApiSession[]>`     | 获取会话列表                |
| `getSession`         | `sessionId, directory?`                     | `Promise<ApiSession>`       | 获取单个会话                |
| `createSession`      | `params?`                                   | `Promise<ApiSession>`       | 创建会话                    |
| `updateSession`      | `sessionId, params, directory?`             | `Promise<ApiSession>`       | 更新会话                    |
| `deleteSession`      | `sessionId, directory?`                     | `Promise<boolean>`          | 删除会话                    |
| `abortSession`       | `sessionId, directory?`                     | `Promise<boolean>`          | 中止会话                    |
| `revertMessage`      | `sessionId, messageId, partId?, directory?` | `Promise<ApiSession>`       | 回退消息                    |
| `unrevertSession`    | `sessionId, directory?`                     | `Promise<ApiSession>`       | 恢复已回退消息              |
| `shareSession`       | `sessionId, directory?`                     | `Promise<ApiSession>`       | 分享会话                    |
| `unshareSession`     | `sessionId, directory?`                     | `Promise<ApiSession>`       | 取消分享                    |
| `forkSession`        | `sessionId, messageId?, directory?`         | `Promise<ApiSession>`       | Fork 会话                   |
| `summarizeSession`   | `sessionId, params, directory?`             | `Promise<boolean>`          | 总结会话                    |
| `getSessionChildren` | `sessionId, directory?`                     | `Promise<ApiSession[]>`     | 获取子会话                  |
| `getSessionTodos`    | `sessionId, directory?`                     | `Promise<ApiTodo[]>`        | 获取会话 Todo 列表          |

### 5.7 消息管理（message.ts）

| 函数名                      | 参数                            | 返回类型                         | 说明             |
| --------------------------- | ------------------------------- | -------------------------------- | ---------------- |
| `getSessionMessages`        | `sessionId, limit?, directory?` | `Promise<ApiMessageWithParts[]>` | 获取消息列表     |
| `getSessionMessageCount`    | `sessionId`                     | `Promise<number>`                | 获取消息数量     |
| `extractUserMessageContent` | `message: UserContentSource`    | `RevertedMessage`                | 提取用户消息内容 |
| `sendMessage`               | `params: SendMessageParams`     | `Promise<SendMessageResponse>`   | 同步发送消息     |
| `sendMessageAsync`          | `params: SendMessageParams`     | `Promise<void>`                  | 异步发送消息     |

### 5.8 权限与问答（permission.ts）

| 函数名                  | 参数                                     | 返回类型                          | 说明           |
| ----------------------- | ---------------------------------------- | --------------------------------- | -------------- |
| `getPendingPermissions` | `sessionId?, directory?`                 | `Promise<ApiPermissionRequest[]>` | 获取待处理权限 |
| `replyPermission`       | `requestId, reply, message?, directory?` | `Promise<boolean>`                | 回复权限       |
| `getPendingQuestions`   | `sessionId?, directory?`                 | `Promise<ApiQuestionRequest[]>`   | 获取待处理问题 |
| `replyQuestion`         | `requestId, answers, directory?`         | `Promise<boolean>`                | 回复问题       |
| `rejectQuestion`        | `requestId, directory?`                  | `Promise<boolean>`                | 拒绝问题       |

### 5.9 文件操作（file.ts）

| 函数名                  | 参数                            | 返回类型                    | 说明              |
| ----------------------- | ------------------------------- | --------------------------- | ----------------- |
| `searchFiles`           | `query, options?`               | `Promise<string[]>`         | 搜索文件/目录     |
| `listDirectory`         | `path, directory?`              | `Promise<FileNode[]>`       | 列出目录内容      |
| `prefetchRootDirectory` | `directory?`                    | `Promise<void>`             | 预取根目录        |
| `getFileContent`        | `path, directory?`              | `Promise<FileContent>`      | 读取文件内容      |
| `getFileStatus`         | `directory?`                    | `Promise<FileStatusItem[]>` | 获取文件 git 状态 |
| `searchSymbols`         | `query, directory?`             | `Promise<SymbolInfo[]>`     | 搜索代码符号      |
| `searchDirectories`     | `query, baseDirectory?, limit?` | `Promise<string[]>`         | 搜索目录          |

### 5.10 终端管理（pty.ts）

| 函数名             | 参数                        | 返回类型           | 说明                    |
| ------------------ | --------------------------- | ------------------ | ----------------------- |
| `listPtySessions`  | `directory?`                | `Promise<Pty[]>`   | 获取 PTY 会话列表       |
| `createPtySession` | `params, directory?`        | `Promise<Pty>`     | 创建 PTY 会话           |
| `getPtySession`    | `ptyId, directory?`         | `Promise<Pty>`     | 获取单个 PTY            |
| `updatePtySession` | `ptyId, params, directory?` | `Promise<Pty>`     | 更新 PTY                |
| `removePtySession` | `ptyId, directory?`         | `Promise<boolean>` | 删除 PTY                |
| `getPtyConnectUrl` | `ptyId, directory?`         | `string`           | 获取 WebSocket 连接 URL |

### 5.11 其他子模块

**agent.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getAgents` | `directory?` | `Promise<ApiAgent[]>` | 获取 agent 列表 |
| `getSelectableAgents` | `directory?` | `Promise<ApiAgent[]>` | 获取可选 agent（过滤 hidden） |

**skill.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getSkills` | `directory?` | `Promise<SkillList>` | 获取可用技能列表 |

**config.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getConfig` | `directory?` | `Promise<Config>` | 获取当前配置 |
| `updateConfig` | `config, directory?` | `Promise<Config>` | 更新配置 |
| `getProviderConfigs` | `directory?` | `Promise<ProvidersResponse>` | 获取 provider 配置 |

**vcs.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getVcsInfo` | `directory?` | `Promise<VcsInfo \| null>` | 获取 VCS 信息 |
| `getVcsDiff` | `mode, directory?` | `Promise<FileDiff[]>` | 获取 VCS diff |

**mcp.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getMcpStatus` | `directory?` | `Promise<MCPStatusResponse>` | 获取 MCP 服务器状态 |
| `addMcpServer` | `name, config, directory?` | `Promise<void>` | 添加 MCP 服务器 |
| `connectMcpServer` | `name, directory?` | `Promise<void>` | 连接 MCP 服务器 |
| `disconnectMcpServer` | `name, directory?` | `Promise<void>` | 断开 MCP 服务器 |
| `startMcpAuth` | `name, directory?` | `Promise<{ url: string }>` | 开始 MCP 认证 |
| `removeMcpAuth` | `name, directory?` | `Promise<void>` | 移除 MCP 认证 |
| `completeMcpAuth` | `name, code, directory?` | `Promise<void>` | 完成 MCP OAuth 认证 |
| `authenticateMcp` | `name, directory?` | `Promise<void>` | 完整 OAuth 认证流程 |

**worktree.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `listWorktrees` | `directory?` | `Promise<string[]>` | 获取 worktree 列表 |
| `createWorktree` | `params, directory?` | `Promise<Worktree>` | 创建 worktree |
| `removeWorktree` | `params, directory?` | `Promise<boolean>` | 删除 worktree |
| `resetWorktree` | `params, directory?` | `Promise<boolean>` | 重置 worktree |

**command.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getCommands` | `directory?` | `Promise<Command[]>` | 获取命令列表（含缓存） |
| `prefetchCommands` | `directory?` | `Promise<void>` | 预取命令 |
| `executeCommand` | `sessionId, command, args?, directory?` | `Promise<unknown>` | 执行命令 |

**global.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getHealth` | 无 | `Promise<HealthInfo>` | 获取服务器健康状态 |
| `disposeGlobal` | 无 | `Promise<boolean>` | 释放所有资源 |
| `disposeInstance` | `directory?` | `Promise<boolean>` | 释放当前实例 |

**tool.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getToolIds` | `directory?` | `Promise<ToolIDs>` | 获取工具 ID 列表 |
| `getTools` | `provider, model, directory?` | `Promise<ToolList>` | 获取工具详细信息 |

**lsp.ts**：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `getLspStatus` | `directory?` | `Promise<LSPStatus>` | 获取 LSP 状态 |
| `getFormatterStatus` | `directory?` | `Promise<FormatterStatus>` | 获取格式化器状态 |

**todo.ts**（内部工具）：
| 函数名 | 参数 | 返回类型 | 说明 |
| ------ | ---- | -------- | ---- |
| `normalizeTodoItems` | `todos: SDKTodo[] \| null \| undefined` | `TodoItem[]` | 标准化 Todo 项 |

---

## 6. Implementation Strategy

### 6.1 文件组织结构

```
src/api/
├── index.ts          # 模块入口，一行代码 re-export client.ts
├── client.ts         # 聚合层：re-export 所有子模块 + Model/Project API
├── sdk.ts            # SDK 客户端：创建、缓存、失效、unwrap
├── http.ts           # HTTP 工具：baseUrl、auth header、query string
├── events.ts         # SSE 全局事件订阅（单例，823 行，最大文件）
├── session.ts        # 会话 CRUD、状态、Diff、Fork、回退、分享、总结
├── message.ts        # 消息查询、内容提取、同步/异步发送
├── permission.ts     # 权限请求列表与回复、问题请求列表与回复/拒绝
├── file.ts           # 目录列表、文件读取、文件搜索、符号搜索、文件状态
├── agent.ts          # 智能体列表、可选智能体过滤
├── skill.ts          # 可用技能列表
├── config.ts         # 配置读取与更新、Provider 配置列表
├── vcs.ts            # VCS 信息获取、Diff 获取
├── mcp.ts            # MCP 服务器状态、添加、连接、断开、OAuth 认证
├── pty.ts            # PTY 会话 CRUD、WebSocket URL 构建
├── worktree.ts       # Worktree 列表、创建、删除、重置
├── command.ts        # 命令列表（含缓存）、命令执行
├── global.ts         # 健康检查、资源释放
├── tool.ts           # 工具 ID 列表、工具详细信息
├── lsp.ts            # LSP 状态、格式化器状态
├── todo.ts           # Todo 项标准化（内部工具，被 events.ts 和 session.ts 使用）
├── types.ts          # 类型重新导出、发送消息参数与响应类型
├── http.test.ts      # buildQueryString 单元测试（7 个用例）
└── command.test.ts   # getCommands 单元测试（2 个用例）
```

### 6.2 依赖流向

```
                    ┌─────────────┐
                    │  serverStore │
                    │  (外部依赖)   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼────┐ ┌────▼─────┐
        │  sdk.ts   │ │ http.ts│ │ pty.ts   │
        │ (客户端)   │ │(URL工具)│ │(WebSocket)│
        └─────┬─────┘ └───┬────┘ └────┬─────┘
              │            │           │
              └────────────┼───────────┘
                           │
                    ┌──────▼──────┐
                    │  events.ts  │
                    │  (SSE 单例)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼────┐ ┌────▼─────┐
        │ session.ts│ │message │ │permission│
        │   .ts     │ │  .ts   │ │   .ts    │
        └───────────┘ └────────┘ └──────────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────▼──────┐
                    │  client.ts  │
                    │  (barrel)   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  index.ts   │
                    │  (入口)     │
                    └─────────────┘
```

所有业务域子模块（session、message、permission、file、agent、skill、config、vcs、mcp、pty、worktree、command、global、tool、lsp）均直接依赖 `sdk.ts` 的 `getSDKClient()` 和 `unwrap()`，彼此之间无横向依赖（除 `session.ts` 调用 `message.ts` 的 `getSessionMessages` 用于 `getLastTurnDiff`）。

### 6.3 初始化序列

```
1. 应用启动
   │
   ├─ serverStore 初始化（从 localStorage 加载服务器配置）
   │
   ├─ 调用 getSDKClientAsync()
   │   ├─ isTauri() ? await getTauriFetch() : 跳过
   │   └─ 创建 OpencodeClient 实例（注入 Tauri fetch）
   │
   ├─ 订阅 serverStore.onServerChange()
   │   └─ 切换时调用 invalidateSDKClient() + reconnectSSE()
   │
   └─ 业务模块按需调用 API 函数
       ├─ getSDKClient() 同步获取（浏览器环境或 Tauri fetch 已加载）
       └─ 首次订阅 SSE 事件时自动建立连接
```

### 6.4 服务器切换流程

```
1. 用户选择新服务器
   │
   ├─ serverStore.setActiveServer(newId)
   │   ├─ 更新 activeServerId
   │   ├─ 保存到 storage
   │   └─ 通知 serverChangeListeners
   │
   ├─ 监听器调用 invalidateSDKClient()
   │   └─ 清除 _cachedClient 和 _cachedKey
   │
   ├─ 监听器调用 reconnectSSE()
   │   ├─ 清除所有 timer
   │   ├─ isServerSwitch = true（标记重连原因）
   │   ├─ connectionGeneration++（旧代次事件失效）
   │   ├─ disconnectTauri()（Tauri 环境）
   │   ├─ singletonController.abort()（浏览器环境）
   │   └─ connectSingleton()（使用新 URL 重建连接）
   │
   └─ 下次 getSDKClient() 创建新客户端
       └─ 使用新服务器的 baseUrl 和 auth
```

### 6.5 路径模式处理

`formatPathForApi()` 是全局路径格式化函数，行为取决于路径模式设置：

- **auto 模式**：根据后端响应自动检测（通过分析返回路径中的斜杠类型）
- **unix 模式**：强制将所有反斜杠转为正斜杠
- **windows 模式**：强制将所有正斜杠转为反斜杠

所有 API 函数在传递 `directory` 参数前均调用 `formatPathForApi()`，确保路径格式与后端期望一致。

---

## 7. Error Handling Strategy

### 7.1 错误解包（unwrap 层）

所有通过 SDK 发起的 API 调用均经过 `unwrap()` 处理：

```
SDK 返回 { data, error }
         │
         ├─ error != null → 抛出 Error
         │   ├─ Error 实例 → 直接抛出
         │   ├─ 字符串 → new Error(str)
         │   └─ 对象 → new Error(JSON.stringify(obj))
         │
         └─ error == null → 返回 data
```

上层业务函数不需要检查 `error` 字段，直接使用返回值或通过 `try/catch` 处理异常。

### 7.2 SSE 错误处理

SSE 连接错误不会抛出到上层，而是通过内部机制处理：

| 错误类型          | 处理方式                                        |
| ----------------- | ----------------------------------------------- |
| HTTP 非 200 响应  | 记录状态码，进入 `scheduleReconnect()`          |
| 流意外结束        | 记录日志，进入 `scheduleReconnect()`            |
| 心跳超时          | 标记 `disconnected`，进入 `scheduleReconnect()` |
| 网络离线          | 标记 `disconnected`，停止重连计时器             |
| Tauri invoke 失败 | 记录错误信息，通知订阅者 `onError`，进入重连    |

订阅者通过 `onError` 回调接收错误通知，通过 `onReconnected` 回调接收恢复通知（携带 `reason: 'network' | 'server-switch'`）。

### 7.3 优雅降级

| 场景                   | 降级策略                             |
| ---------------------- | ------------------------------------ |
| 后端不可达（命令列表） | 仅返回前端命令（`new`、`compact`）   |
| VCS 不可用             | `getVcsInfo()` 捕获异常返回 `null`   |
| Tauri fetch 未加载     | 使用浏览器默认 `fetch`               |
| SSE 解析失败           | 记录警告日志，跳过该事件             |
| 路径模式检测失败       | 回退到 `auto` 模式，默认 `unix` 风格 |

### 7.4 并发安全

| 场景                | 保护机制                                 |
| ------------------- | ---------------------------------------- |
| SSE 并发连接        | `isConnecting` 标志位防止重入            |
| Tauri 并发断开/连接 | `pendingDisconnect` Promise 链串行化     |
| 根目录并发请求      | `rootDirectoryInflight` Map 共享 Promise |
| 命令并发请求        | `commandInflight` Map 共享 Promise       |
| 旧连接事件泄漏      | `connectionGeneration` 代次检查          |

---

## 8. Testing Considerations

### 8.1 现有测试覆盖

当前 `src/api/` 目录下有 2 个测试文件：

**http.test.ts**（7 个用例）：

- `buildQueryString` 的空参数、跳过 undefined、多类型值、特殊字符编码（反斜杠、空格、`&`、`=`）、Unicode 编码

**command.test.ts**（2 个用例）：

- `getCommands` 的 API + 前端命令合并、命令来源标记（`source: 'api' | 'frontend'`）

### 8.2 建议补充的测试

**SDK 客户端层（sdk.ts）**：

- `unwrap()` 的三种错误格式处理（Error 实例、字符串、对象）
- `unwrap()` 成功时返回 data 字段
- `getSDKClient()` 缓存命中返回同一实例
- `invalidateSDKClient()` 后下次获取返回新实例
- Tauri 环境下 `getSDKClientAsync()` 确保 fetch 已注入

**事件系统（events.ts）**：

- 首个订阅者触发连接建立
- 最后一个订阅者取消后连接断开
- 多订阅者共享同一连接
- 连接代次机制：旧代次事件被丢弃
- 心跳超时触发重连
- 页面可见性变化时的连接检查
- 网络恢复/断开时的连接行为
- `reconnectSSE()` 携带 `server-switch` 原因

**文件模块（file.ts）**：

- 根目录缓存 TTL 过期后重新请求
- 并发请求同一目录时仅发起一次网络请求
- 绝对路径与相对路径的处理差异

**PTY 模块（pty.ts）**：

- `getPtyConnectUrl()` 的 URL 构建（有密码/无密码、有目录/无目录）
- HTTP 到 WebSocket 协议转换
- URL userinfo 编码

**消息模块（message.ts）**：

- `extractUserMessageContent()` 提取文本和附件
- `buildPromptParams()` 构建正确的 SDK 参数
- `toFileUrl()` 的路径转换（Windows、Unix、data URI）

### 8.3 Mock 策略

**SDK 客户端 Mock**：

```typescript
vi.mock('./sdk', () => ({
  getSDKClient: () => mockClient,
  unwrap: (result: { data?: unknown }) => result.data,
}))
```

如 `command.test.ts` 所示，mock `getSDKClient` 返回模拟客户端对象，mock `unwrap` 直接返回 `data` 字段。

**serverStore Mock**：

```typescript
vi.mock('../store/serverStore', () => ({
  serverStore: {
    getActiveServerId: () => 'test-server',
    getActiveBaseUrl: () => 'http://localhost:4096',
    getActiveAuth: () => null,
  },
  makeBasicAuthHeader: () => 'Basic dGVzdDp0ZXN0',
}))
```

**Tauri 环境 Mock**：

```typescript
vi.mock('../utils/tauri', () => ({
  isTauri: () => false, // 或 true 测试 Tauri 分支
}))
```

### 8.4 边缘情况

| 边缘情况                      | 测试方法                         |
| ----------------------------- | -------------------------------- |
| 空目录参数                    | 调用 API 时不传 directory        |
| Windows 绝对路径              | `C:\Users\dev\project` 格式化    |
| 路径末尾斜杠                  | `/home/user/` 移除尾部斜杠       |
| 认证信息含特殊字符            | username/password 含 `@`、`:` 等 |
| SSE 事件 JSON 解析失败        | 发送非法 JSON 字符串             |
| 事件类型不在已知集合中        | 发送未知事件类型                 |
| Todo 状态/优先级为非法值      | 传入非标准状态字符串             |
| 并发服务器切换                | 快速连续调用 `setActiveServer`   |
| 网络断开期间页面进入/退出后台 | 模拟 offline + visibilitychange  |
