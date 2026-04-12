# OpenCodeUI 架构文档

> OpenCodeUI 是 [OpenCode](https://github.com/anomalyco/opencode) 的第三方 Web 前端界面，采用 React 19 + TypeScript 构建，通过 REST API、SSE 和 WebSocket 与后端通信。

## 1. 系统概述

OpenCodeUI 作为单页应用（SPA），与 OpenCode 后端形成以下通信架构：

```
┌─────────────────────────────────────────────────────────────┐
│                      OpenCodeUI (前端)                       │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │  REST    │  │   SSE    │  │WebSocket │  │   Tauri    │  │
│  │  请求    │  │  事件流  │  │  PTY终端 │  │  原生桥接  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬──────┘  │
│       │              │              │               │        │
└───────┼──────────────┼──────────────┼───────────────┼────────┘
        │              │              │               │
        ▼              ▼              ▼               ▼
┌──────────────────────────────────────────────────────────────┐
│                   OpenCode Backend (:4096)                    │
│                                                              │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│  │ REST API   │  │ SSE /global/ │  │ WebSocket /pty/    │   │
│  │ /api/*     │  │   event      │  │  {id}/connect      │   │
│  └────────────┘  └──────────────┘  └────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**三条通信通道：**

| 通道      | 协议                | 用途                                                      | 端点                |
| --------- | ------------------- | --------------------------------------------------------- | ------------------- |
| REST      | HTTP/JSON           | 会话管理、消息发送、文件操作、PTY 管理                    | `/api/*`            |
| SSE       | `text/event-stream` | 38+ 种实时事件推送（消息流式输出、权限请求、Todo 更新等） | `/global/event`     |
| WebSocket | `ws://`             | PTY 终端双向通信（键盘输入 + 终端输出）                   | `/pty/{id}/connect` |

**三种部署形态：**

- **Docker 全栈**：Gateway (Caddy) + Frontend + Backend + Router，统一对外
- **Standalone**：仅前端容器，连接已有 `opencode serve`
- **Desktop (Tauri 2)**：Rust 外壳包裹 Web 应用，提供原生 SSE 桥接和系统级能力

---

## 2. 应用架构

### 2.1 入口与初始化

```
main.tsx
  │
  ├─ ensureRandomUUID()          # 非 HTTPS 环境 polyfill
  ├─ themeStore.init()           # 注入 CSS 变量，避免闪烁
  ├─ initOverlayScrollbars()     # 自定义滚动条
  ├─ serverStore.onServerChange() # 注册 server 切换回调
  │   └─ 清理状态 → 重建 SDK → 重连 SSE
  ├─ Tauri 初始化（如适用）
  │   └─ auto-start opencode serve
  ├─ 全局错误处理（error / unhandledrejection）
  │
  └─ bootstrap()
       └─ createRoot(<StrictMode><Suspense>
            <DirectoryProvider>
              <SessionProvider>
                <App />
              </SessionProvider>
            </DirectoryProvider>
          </Suspense></StrictMode>)
```

**关键初始化顺序：**

1. `themeStore.init()` 在 React 渲染前执行，确保 CSS 变量就绪
2. `serverStore.onServerChange()` 注册回调，server 切换时联动清理所有状态
3. Tauri 环境下通过 `invoke('start_opencode_service')` 自动启动后端
4. `bootstrap()` 异步初始化 SDK client 后再挂载 React 树

### 2.2 主布局结构

```
┌─────────────────────────────────────────────────────────────────┐
│                         App Root                                 │
│                                                                  │
│  ┌──────────┐  ┌──────────────────────────────┐  ┌───────────┐  │
│  │          │  │        Chat Surface          │  │           │  │
│  │ Sidebar  │  │  ┌────────────────────────┐  │  │  Right    │  │
│  │ (会话列表│  │  │   SplitContainer       │  │  │  Panel    │  │
│  │  项目切换│  │  │   ┌──────┐ ┌──────┐    │  │  │ (文件/   │  │
│  │  设置入口│  │  │   │Pane  │ │Pane  │    │  │  │  变更/   │  │
│  │          │  │  │   │      │ │      │    │  │  │  MCP)    │  │
│  │          │  │  │   └──────┘ └──────┘    │  │  │          │  │
│  │          │  │   └────────────────────────┘  │  │          │  │
│  │          │  │  ┌────────────────────────┐  │  │          │  │
│  │          │  │  │     BottomPanel        │  │  │          │  │
│  │          │  │  │  (终端/文件/变更/MCP)   │  │  │          │  │
│  │          │  │  └────────────────────────┘  │  │          │  │
│  └──────────┘  └──────────────────────────────┘  └───────────┘  │
│                                                                  │
│  [SettingsDialog] [CommandPalette] [ToastContainer]             │
└─────────────────────────────────────────────────────────────────┘
```

**布局组件关系（App.tsx）：**

- **Sidebar**：左侧会话列表，支持折叠/展开
- **SplitContainer**：多窗格分割容器，基于二叉树结构
- **ChatPane**：单个聊天窗格，承载消息列表和输入框
- **RightPanel**：右侧面板，显示文件浏览器、变更对比、MCP 工具等
- **BottomPanel**：底部面板，终端、文件预览等

### 2.3 多窗格分割系统

`paneLayoutStore` 使用递归二叉树管理窗格布局：

```
PaneNode = PaneLeaf | PaneSplit

PaneLeaf:  { type: 'leaf', id, sessionId }
PaneSplit: { type: 'split', id, direction, ratio, first, second }
                      direction: 'horizontal' | 'vertical'
                      ratio: 0~1 (first 子节点占比)
```

**树结构示例（左右分割 + 右侧上下分割）：**

```
         Split(H, 0.5)
        /              \
   Leaf(pane-1)      Split(V, 0.5)
    session:A         /            \
                 Leaf(pane-2)    Leaf(pane-3)
                 session:B       session:C
```

支持操作：分割（水平/垂直）、关闭、全屏、拖拽交换、焦点切换、比例调整。

---

## 3. 组件架构

```
src/
├── api/                    # API 层
│   ├── sdk.ts              # @opencode-ai/sdk 统一客户端（动态创建、缓存、鉴权）
│   ├── http.ts             # HTTP 工具（baseUrl、auth header、查询字符串）
│   ├── events.ts           # SSE 单例连接（心跳、重连、代次管理、生命周期监听）
│   ├── pty.ts              # PTY 终端 API（REST + WebSocket URL 构建）
│   └── types.ts            # API 类型定义
│
├── features/               # 业务功能模块
│   ├── chat/               # 聊天核心
│   │   ├── Sidebar.tsx     # 侧边栏（会话列表、项目切换）
│   │   ├── ChatPane.tsx    # 聊天窗格（消息列表 + 输入框）
│   │   ├── SplitContainer.tsx  # 多窗格分割渲染器
│   │   └── chatViewport.ts # 视口自适应（响应式/移动端适配）
│   ├── message/            # 消息渲染（Markdown、代码高亮、流式输出）
│   ├── sessions/           # 会话管理（创建、归档、切换）
│   ├── settings/           # 设置面板（外观、聊天、通知、服务、服务器、快捷键）
│   ├── mention/            # @ 提及（文件/文件夹/Agent 引用）
│   └── slash-command/      # 斜杠命令（快速操作）
│
├── components/             # 共享 UI 组件
│   ├── Terminal/           # xterm.js 终端（WebGL 渲染）
│   ├── DiffView/           # 文件 Diff 对比
│   ├── RightPanel.tsx      # 右侧面板容器
│   ├── BottomPanel.tsx     # 底部面板容器
│   ├── CommandPalette.tsx  # 命令面板（全局搜索）
│   ├── SettingsDialog.tsx  # 设置对话框
│   └── ToastContainer.tsx  # 通知提示
│
├── hooks/                  # 自定义 React Hooks
│   ├── useRouter.ts        # Hash 路由（#/session/{id}?dir={path}）
│   ├── useDirectory.ts     # 目录上下文（工作区切换）
│   ├── useGlobalEvents.ts  # SSE 事件全局监听
│   ├── useGlobalKeybindings.ts # 全局快捷键
│   ├── useKeybindings.ts   # 快捷键解析与匹配
│   ├── useViewportHeight.ts    # 移动端视口高度适配
│   └── useCloseServiceDialog.ts # 服务关闭确认对话框
│
├── contexts/               # React Contexts
│   ├── DirectoryContext.tsx    # 目录上下文（当前工作区、已保存目录列表）
│   ├── SessionContext.tsx      # 会话上下文（子会话管理）
│   └── SessionNavigationContext.tsx # 会话导航上下文
│
├── store/                  # 状态管理（22 个 store 文件）
│   ├── messageStore.ts     # 消息状态（每 session 独立存储、SSE 驱动、Undo/Redo）
│   ├── serverStore.ts      # 多服务器配置（健康检查、Basic Auth）
│   ├── layoutStore.ts      # UI 布局（侧边栏、右侧面板、底部面板、Tab 管理）
│   ├── paneLayoutStore.ts  # 窗格分割树（二叉树结构）
│   ├── paneControllerStore.ts # 窗格控制器（每 pane 独立会话操作）
│   ├── keybindingStore.ts  # 快捷键配置
│   ├── themeStore.ts       # 主题系统（3 套预设 + 自定义 CSS）
│   ├── todoStore.ts        # Todo 列表
│   ├── childSessionStore.ts    # 子会话管理
│   ├── activeSessionStore.ts   # 活跃会话追踪
│   ├── autoApproveStore.ts     # 自动审批规则
│   ├── notificationStore.ts    # 通知中心
│   ├── serviceStore.ts         # Tauri 服务管理
│   ├── soundStore.ts           # 通知音效
│   ├── followupQueueStore.ts   # 追问队列
│   ├── changeScopeStore.ts     # 变更范围模式
│   └── index.ts            # 统一导出
│
├── types/                  # 类型定义
│   ├── api/                # API 类型（event.ts、session.ts、message.ts 等）
│   └── message.ts          # 消息 UI 类型
│
├── utils/                  # 工具函数
│   ├── directoryUtils.ts   # 路径处理（Windows/Unix 兼容）
│   ├── messageConversion.ts    # API -> UI 消息转换
│   ├── errorHandling.ts    # 全局错误处理
│   ├── tauri.ts            # Tauri 环境检测
│   └── logger.ts           # 日志工具
│
├── themes/                 # 主题预设
│   ├── eucalyptus.ts       # 尤加利主题
│   ├── claude.ts           # Claude 风格主题
│   └── breeze.ts           # 微风主题
│
└── i18n/                   # 国际化
```

---

## 4. 状态管理

### 4.1 自定义 Store 模式

项目不使用 Redux、Zustand 等第三方状态库，而是实现了一套轻量级 pub-sub 模式，通过 React 19 的 `useSyncExternalStore` 接入组件渲染：

```
┌─────────────┐     subscribe()      ┌──────────────────┐
│   Store     │ ◄─────────────────── │  React Component │
│  (Class)    │ ───── notify() ────► │ useSyncExternal  │
│             │                      │    Store()       │
│  - state    │                      └──────────────────┘
│  - listeners│
│  - snapshot │
└──────┬──────┘
       │
       ├─ getSnapshot()  → 返回不可变快照
       ├─ subscribe(fn)  → 注册监听器，返回取消订阅函数
       └─ notify()       → 更新快照 + 触发所有监听器
```

**核心模式：**

```typescript
class SomeStore {
  private state = {
    /* ... */
  }
  private subscribers = new Set<() => void>()
  private _snapshot: Snapshot | null = null

  subscribe(fn: () => void): () => void {
    this.subscribers.add(fn)
    return () => this.subscribers.delete(fn)
  }

  private notify() {
    this._snapshot = this.buildSnapshot() // 更新快照
    this.subscribers.forEach(fn => fn()) // 触发 React 重渲染
  }

  getSnapshot(): Snapshot {
    return this._snapshot
  }
}

// React 组件中使用
function Component() {
  const state = useSyncExternalStore(
    store.subscribe,
    store.getSnapshot,
    store.getSnapshot, // server snapshot
  )
}
```

### 4.2 关键 Store 说明

| Store                   | 职责                                                                                                                   | 持久化                         |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| **messageStore**        | 每 session 消息独立存储，SSE 事件驱动更新，RAF 批量通知，支持 Undo/Redo（revertState），LRU 缓存（最多 10 个 session） | 无（内存）                     |
| **serverStore**         | 多服务器配置管理，健康检查，Basic Auth，server 切换时通知外部（SSE 重连）                                              | localStorage + sessionStorage  |
| **layoutStore**         | 侧边栏状态、右侧面板/底部面板开关、Tab 系统（files/changes/mcp/skill/worktree/terminal）、面板尺寸持久化               | localStorage                   |
| **paneLayoutStore**     | 二叉树窗格布局，支持分割/关闭/全屏/拖拽/焦点管理                                                                       | 无（内存）                     |
| **paneControllerStore** | 每个 pane 的独立控制器（会话切换、模型选择、Agent 切换、流式状态）                                                     | 无（内存）                     |
| **keybindingStore**     | 快捷键配置、解析、匹配，支持自定义键位绑定                                                                             | localStorage                   |
| **themeStore**          | 主题系统初始化、CSS 变量注入、明暗模式切换                                                                             | localStorage                   |
| **todoStore**           | 每 session 的 Todo 列表，SSE 事件驱动更新                                                                              | 无（内存）                     |
| **childSessionStore**   | 子会话（sub-agent）管理，父子会话关系追踪                                                                              | 无（内存）                     |
| **activeSessionStore**  | 活跃会话追踪（流式中的 session）                                                                                       | 无（内存）                     |
| **autoApproveStore**    | 自动审批规则（per-server 隔离存储）                                                                                    | localStorage（per-server key） |
| **notificationStore**   | 通知中心、Toast 管理                                                                                                   | 无（内存）                     |
| **serviceStore**        | Tauri 服务生命周期管理（启动/停止/状态）                                                                               | localStorage                   |
| **soundStore**          | 通知音效设置                                                                                                           | localStorage                   |
| **followupQueueStore**  | 追问队列管理                                                                                                           | 无（内存）                     |
| **changeScopeStore**    | 变更范围模式（diff 过滤）                                                                                              | 无（内存）                     |

### 4.3 messageStore 设计亮点

- **每 session 独立存储**：`Map<sessionId, SessionState>`，避免全局单数组的性能问题
- **LRU 淘汰**：最多缓存 10 个 session，被分屏 pane 保护的 session 不会被淘汰
- **Delta 批量化**：流式输出时直接 mutable 修改 part.text，通过 `dirtyMessages` 标记，在 rAF 回调中统一生成不可变快照，避免每帧多次数组拷贝
- **Undo/Redo**：通过 `revertState` 实现，记录回退点之后的用户消息历史

---

## 5. API 层

### 5.1 SDK Client（`src/api/sdk.ts`）

基于 `@opencode-ai/sdk` 封装的统一客户端：

```
┌─────────────────────────────────────────┐
│            getSDKClient()               │
│                                         │
│  1. buildCacheKey() → "baseUrl|auth"    │
│  2. 命中缓存 → 直接返回                  │
│  3. 未命中 → createOpencodeClient()     │
│     - baseUrl: serverStore              │
│     - headers: Basic Auth (如有)        │
│     - fetch: Tauri plugin-http (如适用) │
│  4. 缓存实例                             │
└─────────────────────────────────────────┘
```

- **按 "baseUrl + auth" 缓存**：避免重复创建实例
- **Tauri 环境适配**：使用 `@tauri-apps/plugin-http` 的 fetch 替代浏览器原生 fetch
- **`unwrap()` 工具**：将 SDK 返回的 `{ data, error }` 解包为纯数据或抛出异常

### 5.2 SSE 事件系统（`src/api/events.ts`）

**单例连接架构：**

```
                    ┌─────────────────────┐
                    │   subscribeToEvents │
                    │   (allSubscribers)  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   connectSingleton  │
                    │   (单例连接)         │
                    └──────┬──────┬──────┘
                           │      │
              ┌────────────┘      └────────────┐
              │                                 │
     ┌────────▼────────┐              ┌────────▼────────┐
     │ connectViaBrowser│              │ connectViaTauri │
     │ (fetch +        │              │ (Rust Channel)  │
     │  ReadableStream)│              │                 │
     └─────────────────┘              └─────────────────┘
```

**核心机制：**

| 机制               | 说明                                                                    |
| ------------------ | ----------------------------------------------------------------------- |
| **单例连接**       | 所有订阅者共享一个 SSE 连接，首个订阅者触发连接，最后一个订阅者断开     |
| **心跳检测**       | 前台 60s / 后台 120s 超时，超时自动重连                                 |
| **指数退避重连**   | `[1s, 2s, 3s, 5s, 10s, 30s]`，后台更激进 `[0.5s, 1s, 2s, 3s, 5s, 10s]`  |
| **代次管理**       | `connectionGeneration` 递增，旧代次事件自动丢弃，避免重连后收到过期事件 |
| **生命周期监听**   | `visibilitychange`（前后台切换）、`online/offline`（网络状态）          |
| **后台 Keepalive** | 30s 定期检查，防止移动端后台静默断连                                    |
| **Tauri SSE 桥接** | Rust 侧使用 `reqwest` + `Channel` 实现，解决浏览器后台限制              |

### 5.3 38+ SSE 事件类型

```
会话相关:  session.created / .updated / .deleted / .idle / .error / .status / .diff / .compacted
消息相关:  message.updated / .removed / .part.updated / .part.delta / .part.removed
权限相关:  permission.asked / .replied
问答相关:  question.asked / .replied / .rejected
Todo:     todo.updated
项目:     project.updated
工作树:   worktree.ready / .failed
VCS:      vcs.branch.updated
TUI:      tui.prompt.append / .command.execute / .toast.show / .session.select
服务器:   server.connected / .instance.disposed / global.disposed
文件:     file.edited / .file.watcher.updated
安装:     installation.updated / .update-available
LSP:      lsp.client.diagnostics / .updated
MCP:      mcp.tools.changed / .browser.open.failed
PTY:      pty.created / .updated / .exited / .deleted
命令:     command.executed
工作区:   workspace.ready / .failed
```

### 5.4 PTY WebSocket

终端通过 WebSocket 连接后端 PTY 端点：

```
WebSocket URL 构建:
  http://host:4096  →  ws://host:4096
  认证: ws://user:pass@host:4096/pty/{id}/connect?directory=...
  （WebSocket 不支持自定义 Header，通过 URL userinfo 传递认证）
```

---

## 6. 数据流

### 6.1 用户发送消息

```
用户输入 → ChatPane.onSubmit
    │
    ├─ messageStore.setLoadState('loading')
    ├─ SDK: session.create() / message.create()
    │
    └─ 等待 SSE 事件驱动 UI 更新
```

### 6.2 SSE 事件驱动更新

```
SSE 事件到达
    │
    ├─ parseGlobalEvent() → 解析 + 类型校验
    ├─ broadcastEvent() → 分发给所有订阅者
    │
    └─ handleEventForSubscriber()
         │
         ├─ onMessageUpdated  → messageStore.handleMessageUpdated()
         ├─ onPartDelta       → messageStore.handlePartDelta() (mutable + dirty 标记)
         ├─ onSessionIdle     → messageStore.handleSessionIdle() (停止流式)
         ├─ onSessionError    → messageStore.handleSessionError()
         ├─ onTodoUpdated     → todoStore.update()
         ├─ onSessionCreated  → childSessionStore / sidebar 更新
         └─ ...
              │
              └─ store.notify()
                   │
                   ├─ flushDirtyMessages() (RAF 回调中统一生成不可变快照)
                   └─ subscribers.forEach(fn => fn())
                        │
                        └─ useSyncExternalStore 触发 React 重渲染
```

### 6.3 完整数据流图

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  用户操作 │ ──► │  Store   │ ──► │  API 调用 │ ──► │ SSE 事件 │ ──► │  Store   │
│  (点击/  │     │  状态更新 │     │ (REST)   │     │ (推送)   │     │  更新    │
│  输入)   │     │          │     │          │     │          │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘     └────┬─────┘
                                                                         │
                                                                    ┌────▼─────┐
                                                                    │ UI 重渲染 │
                                                                    │(React)   │
                                                                    └──────────┘
```

### 6.4 Server 切换联动

```
serverStore.setActiveServer(newId)
    │
    ├─ saveToStorage()
    ├─ notify() → UI 更新
    └─ serverChangeListeners.forEach(fn => fn(newId))
         │
         ├─ invalidateSDKClient()          # 清除 SDK 缓存
         ├─ messageStore.clearAll()        # 清空消息
         ├─ childSessionStore.clearAll()   # 清空子会话
         ├─ todoStore.clearAll()           # 清空 Todo
         ├─ resetPathModeCache()           # 重置路径缓存
         ├─ autoApproveStore.reloadFromStorage()  # 重新加载审批规则
         └─ reconnectSSE()                 # 重连 SSE（新 URL）
```

---

## 7. 部署架构

### 7.1 Docker 全栈部署

```
                    外部网络
                       │
            ┌──────────▼──────────┐
            │      Gateway        │
            │    (Caddy)          │
            │                     │
            │  :6658  主入口      │
            │  :6659  预览端口    │
            └────┬───────┬───────┘
                 │       │
     ┌───────────┤       ├───────────────┐
     │           │       │               │
     ▼           ▼       ▼               ▼
 ┌────────┐  ┌────────┐ ┌────────┐  ┌──────────┐
 │ /api/* │  │  其他   │ │/routes │  │ :6659    │
 │ →      │  │ →      │ │/preview│  │ (预览)   │
 │Backend │  │Frontend│ │→Router │  │          │
 │:4096   │  │:3000   │ │:7070   │  │动态端口  │
 └────────┘  └────────┘ └────────┘  │路由      │
                                    └──────────┘
```

**路由规则（端口 6658）：**

| 路径         | 转发目标         | 说明                                          |
| ------------ | ---------------- | --------------------------------------------- |
| `/api/*`     | Backend `:4096`  | OpenCode API，支持 SSE（`flush_interval -1`） |
| `/routes`    | Router `:7070`   | 动态路由管理面板                              |
| `/preview/*` | Router `:7070`   | 预览端口切换 API                              |
| 其他         | Frontend `:3000` | 前端静态资源（SPA fallback）                  |

**服务清单：**

| 服务     | 镜像                  | 端口         | 说明                    |
| -------- | --------------------- | ------------ | ----------------------- |
| Gateway  | `opencodeui-gateway`  | 6658/6659    | Caddy 反向代理 + Router |
| Frontend | `opencodeui-frontend` | 3000（内部） | 静态前端（Caddy 托管）  |
| Backend  | `opencodeui-backend`  | 4096（内部） | `opencode serve`        |

**数据卷：**

| 卷                     | 用途                                          |
| ---------------------- | --------------------------------------------- |
| `opencode-home`        | 后端 `/root` 持久化（配置、会话缓存、工具链） |
| `opencode-router-data` | Router 状态持久化                             |

### 7.2 Standalone 部署

```
┌─────────────────────────────────────────────┐
│          宿主机 / 远程服务器                  │
│                                             │
│  ┌──────────────┐      ┌─────────────────┐  │
│  │ opencode     │      │  Frontend       │  │
│  │ serve :4096  │◄─────│  Container      │  │
│  │ (已有)       │      │  :3000 → 外部端口│  │
│  └──────────────┘      │  Caddy 反代 /api│  │
│                        └─────────────────┘  │
└──────────────────────────────────────────────┘
```

仅部署前端容器，通过 `BACKEND_URL` 环境变量指向已有后端。

### 7.3 Desktop (Tauri 2)

```
┌──────────────────────────────────────────────┐
│                  Tauri App                   │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │           Webview (Web 应用)            │  │
│  │  React + Vite + 所有前端逻辑            │  │
│  └──────────────────┬─────────────────────┘  │
│                     │                        │
│  ┌──────────────────▼─────────────────────┐  │
│  │           Rust Shell                    │  │
│  │                                         │  │
│  │  - sse_connect / sse_disconnect        │  │
│  │    (reqwest + Channel 桥接 SSE)        │  │
│  │  - start_opencode_service              │  │
│  │    (启动/管理 opencode serve 进程)     │  │
│  │  - @tauri-apps/plugin-http             │  │
│  │    (原生 HTTP 客户端)                  │  │
│  └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

Tauri 提供以下原生能力：

- **SSE 桥接**：Rust 侧使用 `reqwest` 建立 SSE 连接，通过 `Channel` 推送事件到 Webview，解决浏览器后台限制
- **服务管理**：`invoke('start_opencode_service')` 启动/停止 `opencode serve` 进程
- **原生 HTTP**：`@tauri-apps/plugin-http` 替代浏览器 fetch，支持更灵活的网络控制
- **安全区域适配**：自动注入 `viewport-fit=cover`，支持 macOS 刘海屏

---

## 8. 路由系统

### 8.1 自定义 Hash 路由

项目不使用 `react-router`，而是基于 `window.location.hash` 实现轻量级路由：

```
路由格式:
  #/                     → 首页（无选中会话）
  #/session/{sessionId}  → 指定会话
  ?dir={path}            → 可选的目录参数（URL 编码）

示例:
  #/session/abc-123?dir=%2Fhome%2Fuser%2Fproject
```

### 8.2 路由 Store 设计

```
┌─────────────────────────────────────────┐
│           useRouter()                    │
│                                          │
│  subscribe(listener)                     │
│    ├─ 首次调用注册 hashchange 监听        │
│    └─ 返回取消订阅函数                    │
│                                          │
│  getSnapshot()                           │
│    └─ parseHash() → { sessionId, dir }  │
│                                          │
│  navigateToSession(id, dir?)             │
│    └─ window.location.hash = buildHash() │
│       emitRoute(next)                    │
│                                          │
│  navigateHome()                          │
│    └─ hash = '#/' (+ dir)               │
│                                          │
│  replaceSession(id, dir?)                │
│    └─ history.replaceState()            │
│       emitRoute(next)                    │
│                                          │
│  setDirectory(dir?) / replaceDirectory() │
│    └─ 更新 dir 参数 + 持久化到 storage   │
└─────────────────────────────────────────┘
```

**关键设计：**

- **模块级单例**：所有 `useRouter()` 调用共享同一个 `routeSnapshot`，避免多实例状态不一致
- **`useSyncExternalStore` 接入**：与 React 18+ 并发渲染兼容
- **`replaceState` 防抖**：`syncingFromRouteRef` 防止 URL → Store → URL 的循环更新
- **目录持久化**：`serverStorage` 存储上次使用的目录，刷新后恢复

### 8.3 路由与窗格的同步

```
URL 变化 (hashchange)
    │
    └─► syncingFromRouteRef = true
        paneLayoutStore.setFocusedSession(routeSessionId)
             │
             └─► focused pane session 变化
                  │
                  └─► replaceSession() → 更新 URL（如果不同步）

Focused pane session 变化
    │
    └─► replaceSession(paneLayout.focusedSessionId, dir)
         │
         └─► history.replaceState() → 更新 URL
```

---

## 附录：技术栈总览

| 类别     | 技术                                   |
| -------- | -------------------------------------- |
| 框架     | React 19 + TypeScript                  |
| 构建     | Vite 7                                 |
| 样式     | Tailwind CSS v4                        |
| 代码高亮 | Shiki                                  |
| Markdown | react-markdown + remark-gfm            |
| 数学公式 | KaTeX                                  |
| 终端     | xterm.js (WebGL 渲染)                  |
| 状态管理 | 自定义 Store + `useSyncExternalStore`  |
| 路由     | 自定义 Hash 路由                       |
| 桌面     | Tauri 2 (Rust)                         |
| 部署     | Docker (Caddy Gateway + Python Router) |
| 国际化   | react-i18next                          |
| 滚动条   | Overlay Scrollbars                     |
