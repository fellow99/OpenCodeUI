# OpenCodeUI 总体技术规划（As-Built）

> 本文档是对已完成项目的回溯性技术规划，记录"实际建成"的架构设计、模块分解与集成策略。

---

## 1. 模块分解

系统划分为 16 个功能模块，每个模块对应 `specs-plan-b/` 下的一个子目录。

### 001-api-layer：后端通信抽象

**职责**：封装与 OpenCode 后端的所有通信，提供类型安全的 API 调用接口。

**核心设计**：

- **SDK 客户端层**（`src/api/sdk.ts`）：基于 `@opencode-ai/sdk` 构建统一的 `OpencodeClient`，按 `baseUrl + authHash` 缓存实例，支持 Tauri 原生 fetch 注入。提供 `getSDKClient()`（同步）、`getSDKClientAsync()`（异步，确保 Tauri fetch 就绪）、`invalidateSDKClient()`（服务器切换时强制重建）三个入口。
- **统一 API 出口**（`src/api/client.ts`）：作为 barrel 文件，re-export 所有子模块（session、message、permission、file、agent、skill、events、config、vcs、mcp、pty、worktree、command、global、tool、lsp），上层只需 `import from '@/api/client'`。
- **unwrap 模式**：SDK 返回 `{ data, error }` 结构，通过 `unwrap()` 统一提取 data 或抛出 error，简化上层调用。
- **路径格式化**：通过 `formatPathForApi()` 处理跨平台路径差异（Windows 反斜杠转正斜杠）。

**子模块**：
| 文件 | 职责 |
|------|------|
| `sdk.ts` | SDK 客户端创建、缓存、失效 |
| `http.ts` | 基础 HTTP 工具（baseUrl 解析、auth header） |
| `events.ts` | SSE 全局事件订阅（单例模式） |
| `session.ts` | 会话 CRUD |
| `message.ts` | 消息发送、流式读取 |
| `permission.ts` | 工具调用权限请求/回复 |
| `file.ts` | 文件读写、目录浏览 |
| `pty.ts` | 终端 PTY 会话管理 |
| `agent.ts` / `skill.ts` | Agent 与 Skill 管理 |
| `config.ts` / `mcp.ts` | 配置与 MCP 服务器 |
| `vcs.ts` / `worktree.ts` | 版本控制与 Git worktree |
| `tool.ts` / `lsp.ts` | 工具调用与 LSP 集成 |
| `command.ts` / `global.ts` | 全局命令与事件 |

### 002-chat-feature：核心聊天界面

**职责**：提供完整的聊天交互界面，包括侧边栏、聊天区域、输入框、模型选择器等。

**核心设计**：

- **App 根组件**（`src/App.tsx`）：作为全局编排中心，组合 Sidebar、SplitContainer、BottomPanel、RightPanel 四大布局区域，管理 SettingsDialog、CommandPalette、CloseServiceDialog 等懒加载对话框。
- **ChatPane**：单个聊天窗格组件，支持 `single` 和 `split` 两种显示模式，内部包含 Header、ChatArea、InputBox。
- **SplitContainer**：递归渲染 pane 布局树（`PaneNode`），支持水平/垂直分割和全屏切换。
- **ChatViewportProvider**：响应式视口上下文，根据侧边栏状态、右面板宽度动态计算聊天区域可用空间，决定侧边栏行为（push vs overlay）。
- **Sidebar**：会话列表侧边栏，支持项目选择、会话搜索、新建会话。
- **InputBox**：消息输入框，集成 @mention、/slash-command、文件附件、图片上传。
- **ModelSelector**：模型选择器，支持模型切换、Agent 模式切换。

**关键 Hook**：
| Hook | 职责 |
|------|------|
| `useChatSession` | 单个会话的生命周期管理（发消息、取消、流式处理） |
| `useGlobalEvents` | SSE 事件分发到对应 pane 的 session |
| `useRouter` | Hash 路由管理 |
| `useDirectory` | 工作目录管理 |
| `useKeybindings` | 全局快捷键 |

### 003-message-rendering：消息显示与格式化

**职责**：渲染各类消息内容，包括文本、代码块、工具调用、权限请求、Todo 列表等。

**核心设计**：

- **MessageRenderer**（`src/features/message/MessageRenderer.tsx`）：根据消息类型和 part 类型分发到不同的渲染组件。
- **Parts 系统**：消息由多个 part 组成（text、tool、question、permission 等），每个 part 有独立的渲染逻辑。
  - `parts/` 目录：text part（Markdown 渲染）、tool part（工具调用展示）、question part（内联问答）、permission part（权限审批）。
  - `tools/` 目录：各类工具调用的详细展示（文件读写、命令执行、搜索等）。
- **MarkdownRenderer**：基于 `react-markdown + remark-gfm`，集成 Shiki 代码高亮、KaTeX 数学公式、自定义组件（CodeBlock、MentionTag）。
- **CodeBlock**：代码块组件，支持语言检测、语法高亮、复制按钮、全屏查看。
- **流式渲染**：通过 SSE `MESSAGE_PART_DELTA` 事件实现增量更新，避免全量重渲染。

### 004-session-management：会话生命周期

**职责**：管理会话的创建、切换、归档、删除等生命周期操作。

**核心设计**：

- **SessionProvider**（Context）：在 App 初始化时加载会话列表，提供会话数据给子组件。
- **SessionList**：侧边栏中的会话列表组件，支持分组（活跃/归档）、搜索、虚拟滚动。
- **useSessions** Hook：封装会话列表的加载、刷新、分页逻辑。
- **childSessionStore**：管理子会话（fork/branch 产生的派生会话），维护父子关系树。
- **activeSessionStore**：跟踪当前活跃（正在处理中）的会话，用于 UI 状态指示。
- **followupQueueStore**：后续追问队列管理，支持自动连续提问。

### 005-settings-panel：用户配置

**职责**：提供用户设置界面，管理各项配置项。

**核心设计**：

- **SettingsDialog**（`src/features/settings/SettingsDialog.tsx`）：多标签页设置对话框，支持懒加载。
- **标签页**：
  - `appearance`：主题、颜色模式、自定义 CSS
  - `chat`：聊天相关设置（上下文限制、step-finish 显示项）
  - `notifications`：通知设置（浏览器通知、声音提醒）
  - `service`：服务管理（opencode serve 自动启动、二进制路径）
  - `servers`：服务器配置（多服务器管理、健康检查）
  - `keybindings`：快捷键自定义
- **KeybindingsSection**：快捷键配置组件，支持按键捕获、冲突检测、重置默认值。
- **配置持久化**：各 store 自行管理 localStorage 持久化，设置面板仅作为 UI 入口。

### 006-mention-system：@提及文件/符号

**职责**：在聊天输入中支持 @提及文件和符号，快速引用工作区内容。

**核心设计**：

- **MentionMenu**：弹出式提及菜单，支持文件路径搜索、符号搜索。
- **MentionTag**：渲染已插入的提及标签，显示文件/符号名称和图标。
- **useMention** Hook：管理提及菜单的显示/隐藏、选择、插入逻辑。
- **createMentionElement**：创建提及元素的 DOM 节点，使用自定义 `data-mention-*` 属性存储元数据。
- **提及解析**：发送消息时，将 DOM 中的提及元素转换为结构化数据，通过 API 传递给后端。

### 007-slash-command：命令界面

**职责**：提供 /斜杠命令 输入支持，快速执行预定义操作。

**核心设计**：

- **SlashCommandMenu**：弹出式命令菜单，在输入框中输入 `/` 时触发。
- **命令注册**：命令列表从后端 API 动态获取（`command.ts`），支持自定义命令。
- **命令执行**：选择命令后，将其转换为消息内容的一部分发送给后端。
- **与 CommandPalette 的区别**：SlashCommand 是聊天上下文内的命令（影响当前对话），CommandPalette 是全局命令面板（`Ctrl/Cmd+K`，影响应用级别操作）。

### 008-terminal-system：集成 PTY 终端

**职责**：提供基于 xterm.js 的 Web 终端，支持多标签页、WebSocket 连接。

**核心设计**：

- **Terminal**（`src/components/Terminal.tsx`）：单个 xterm 实例，通过 WebSocket 连接后端 PTY。
  - xterm.js + WebGL 渲染
  - FitAddon 自动适配容器尺寸
  - WebLinksAddon 链接点击支持
  - 主题色动态适配（从 CSS 变量读取 HSL 值）
- **TerminalPanel**（`src/components/TerminalPanel.tsx`）：终端面板容器，支持多标签页管理。
- **PTY API**（`src/api/pty.ts`）：创建 PTY 会话、获取 WebSocket 连接 URL、更新会话状态。
- **layoutStore** 集成：终端标签页作为 BottomPanel 的一部分，与聊天区域共享布局状态。
- **Tauri 支持**：在 Tauri 环境中使用原生 HTTP 客户端建立 WebSocket 连接。

### 009-theme-system：视觉主题

**职责**：管理应用主题、颜色模式、自定义 CSS。

**核心设计**：

- **themeStore**（`src/store/themeStore.ts`）：主题状态管理。
  - 主题预设：Eucalyptus（默认）、Claude、Breeze
  - 颜色模式：system / light / dark
  - 自定义 CSS：用户可注入覆盖样式
  - CSS 变量注入：通过 `<style>` 标签动态生成 CSS 变量
- **主题预设**（`src/themes/index.ts`）：定义各主题的色板（背景、文本、边框、强调色等）。
- **初始化时机**：在 `main.tsx` 中 React 渲染前调用 `themeStore.init()`，提前注入 CSS 变量，避免闪烁。
- **终端主题适配**：Terminal 组件从 CSS 变量读取颜色，自动适配当前主题。
- **系统主题监听**：监听 `prefers-color-scheme` 变化，自动切换明暗模式。

### 010-state-management：应用状态协调

**职责**：管理全局应用状态，协调各模块间的数据流。

**核心设计**：

- **Store 架构**：采用自定义 Store 模式（非 Redux/Zustand），每个 store 是独立的模块级单例。
  - 状态存储：模块级变量 + `Set<Listener>` 订阅机制
  - 响应式读取：`useSyncExternalStore` 桥接 React 渲染
  - 持久化：各 store 自行管理 localStorage 读写
- **核心 Store**：
  | Store | 职责 |
  |-------|------|
  | `messageStore` | 会话消息状态（核心 store，最大最复杂） |
  | `paneLayoutStore` | 分 pane 布局树（二叉分割树） |
  | `paneControllerStore` | 每个 pane 的控制器（会话操作代理） |
  | `layoutStore` | 全局布局（侧边栏、右面板、底面板开关） |
  | `serverStore` | 多服务器配置与健康检查 |
  | `themeStore` | 主题与颜色模式 |
  | `keybindingStore` | 快捷键配置 |
  | `autoApproveStore` | 工具调用自动审批规则 |
  | `todoStore` | Todo 列表状态 |
  | `notificationStore` | 通知与 Toast |
  | `childSessionStore` | 子会话（fork）关系 |
  | `activeSessionStore` | 活跃会话跟踪 |
  | `serviceStore` | Tauri 服务管理 |
  | `soundStore` | 通知声音设置 |
  | `followupQueueStore` | 后续追问队列 |
  | `changeScopeStore` | 变更范围模式 |
- **messageStoreHooks**：提供细粒度的 React hooks（`useMessages`、`useIsStreaming`、`useSessionState` 等），避免不必要的重渲染。

### 011-file-diff-viewer：代码 Diff 可视化

**职责**：提供文件变更查看和多文件 Diff 对比功能。

**核心设计**：

- **DiffView**（`src/components/DiffView.tsx`）：单文件 diff 展示，支持统一视图和分列视图。
- **DiffViewer**：增强的 diff 查看器，支持行号、语法高亮、折叠。
- **DiffModal**：弹窗式 diff 查看，从侧边栏或消息中触发。
- **MultiFileDiffModal**：多文件 diff 对比，支持文件列表导航。
- **SessionChangesPanel**：会话变更面板，展示当前会话涉及的所有文件变更。
- **数据源**：通过 VCS API 获取 Git diff 数据，通过 File API 获取文件内容。

### 012-pane-layout：多 pane 分割布局

**职责**：实现多窗格分割布局，支持水平/垂直分割、拖拽调整、全屏切换。

**核心设计**：

- **二叉分割树**：`PaneNode = PaneLeaf | PaneSplit`
  - `PaneLeaf`：叶子节点，包含 sessionId
  - `PaneSplit`：分割节点，包含 direction（horizontal/vertical）、ratio（分割比例）、first/second 子节点
- **paneLayoutStore**：管理布局树的状态机，支持 split、close、focus、toggleFullscreen 等操作。
- **paneControllerStore**：为每个 pane 创建独立的控制器，封装会话操作（newSession、archiveSession、previousSession 等），使每个 pane 独立管理自己的会话状态。
- **SplitContainer**：递归渲染布局树的组件，使用 CSS flex 实现分割效果。
- **拖拽调整**：通过 `useVerticalSplitResize` Hook 实现分割线拖拽，动态调整 ratio。
- **与路由同步**：App 组件中维护 URL hash 与 focused pane session 的双向同步。

### 013-i18n-localization：国际化

**职责**：支持多语言界面，当前提供英文和简体中文。

**核心设计**：

- **i18next**：使用 `i18next + react-i18next + i18next-browser-languagedetector`。
- **翻译文件**：`src/locales/{lang}/{namespace}.json`，按命名空间组织（common、chat、commands、components、settings 等）。
- **语言**：en（英文）、zh-CN（简体中文）。
- **语言检测**：优先读取 localStorage 中的 `i18nextLng`，回退到浏览器 navigator 语言。
- **Vite glob import**：通过 `import.meta.glob('./locales/*/*.json', { eager: true })` 在构建时 eager-load 所有翻译文件。
- **使用方式**：组件中通过 `useTranslation(['namespace1', 'namespace2'])` 按需加载命名空间。

### 014-tauri-desktop：桌面应用打包

**职责**：将 Web 应用打包为原生桌面应用（macOS / Linux / Windows）。

**核心设计**：

- **Tauri 2**：基于 Rust 的轻量级桌面应用框架。
- **项目结构**（`src-tauri/`）：
  - `tauri.conf.json`：应用配置（窗口、安全、构建）
  - `Cargo.toml`：Rust 依赖
  - `src/main.rs`：Rust 入口
  - `capabilities/`：权限能力配置
  - `icons/`：应用图标
- **Rust 侧功能**：
  - `start_opencode_service`：启动 opencode serve 进程
  - `sse_connect` / `sse_disconnect`：SSE 连接桥接（使用 reqwest + Channel）
  - 原生文件系统访问
  - 系统通知
- **前端适配**：
  - `isTauri()` 检测运行环境
  - Tauri 环境下使用 `@tauri-apps/plugin-http` 替代浏览器 fetch
  - Safe area 适配（`viewport-fit=cover`）
  - 添加 `.tauri-app` CSS class

### 015-docker-deployment：容器化部署

**职责**：提供 Docker 容器化部署方案，支持前后端分离和独立前端两种模式。

**核心设计**：

- **三服务架构**：
  | 服务 | Dockerfile | 职责 |
  |------|-----------|------|
  | Frontend | `Dockerfile.frontend` | Vite 构建的静态前端，Caddy 托管 |
  | Backend | `Dockerfile.backend` | OpenCode 后端 + mise 工具链 |
  | Gateway | `Dockerfile.gateway` | Caddy 反向代理 + Rust 路由服务 |
- **Gateway 路由**：
  - 端口 6658：统一入口，`/api/*` 转发到 Backend，`/routes` 和 `/preview/*` 转发到 Router，其余转发到 Frontend
  - 端口 6659：预览端口，由 Router 动态管理
- **独立前端模式**（`docker-compose.standalone.yml`）：仅部署前端容器，连接已有后端。
- **数据持久化**：
  - `opencode-home` 卷：OpenCode 配置、会话缓存、工具链
  - `opencode-router-data` 卷：动态路由状态
- **入口脚本**：`backend-entrypoint.sh` 启动时自动校验并补齐 opencode/mise。

### 016-router-service：动态端口路由

**职责**：扫描容器内运行的开发服务，动态生成预览链接。

**核心设计**：

- **技术栈**：Rust + Axum 框架。
- **端口扫描**（`scanner` 模块）：定期扫描 3000-9999 端口范围，检测活跃服务。
- **路由管理**（`router.rs`）：
  - 创建路由：生成随机 token，绑定到检测到的端口
  - 路由状态持久化：内存 `RwLock<Map>` + 磁盘 JSON 文件
  - Caddy 动态配置：通过 Caddy API 实时更新反向代理规则
- **预览管理**（`router.html`）：内置管理面板，查看/创建/删除路由。
- **环境变量**：
  - `ROUTER_SCAN_INTERVAL`：扫描间隔（秒）
  - `ROUTER_PORT_RANGE`：扫描端口范围
  - `ROUTER_EXCLUDE_PORTS`：排除端口

---

## 2. 设计决策

### 2.1 自定义 Hash 路由 vs React Router

**决策**：使用自定义 Hash 路由（`useRouter` Hook），不引入 react-router。

**理由**：

- 路由需求简单：仅需 `#/session/{id}?dir={path}` 和 `#/` 两种模式
- Hash 路由无需服务器配置，支持 GitHub Pages 等静态托管
- 避免引入额外依赖（react-router 约 15KB gzipped）
- 使用 `useSyncExternalStore` 桥接，与 React 18+ 并发特性兼容
- 模块级共享 route store，避免多组件实例间状态不一致

**权衡**：

- 不支持嵌套路由、路由守卫等高级特性（当前不需要）
- 手动处理 hash 解析和构建（代码量约 170 行，可接受）

### 2.2 自定义 Store 模式 vs Redux/Zustand

**决策**：采用自定义 Store 模式（模块级单例 + 订阅机制 + `useSyncExternalStore`）。

**理由**：

- 零额外依赖，减少 bundle 体积
- 每个 store 独立管理自己的状态和持久化逻辑，职责清晰
- `useSyncExternalStore` 原生支持 React 并发渲染，避免 tearing
- 细粒度 hooks（如 `useMessages`、`useIsStreaming`）精确控制重渲染范围
- 项目规模可控（16 个 store），不需要 Redux 的中间件生态

**权衡**：

- 缺少 Redux DevTools 等调试工具（通过 console.log 和自定义日志弥补）
- 跨 store 事务性更新需要手动协调（通过事件回调机制解决）
- 新成员需要学习自定义模式（代码风格统一，有注释说明）

### 2.3 SDK-based API 层 vs 原始 Fetch

**决策**：基于 `@opencode-ai/sdk` 构建 API 层，而非直接使用 fetch。

**理由**：

- 类型安全：SDK 提供完整的 TypeScript 类型定义
- 自动参数校验：SDK 在发送前校验请求参数
- 统一错误处理：SDK 返回 `{ data, error }` 结构，通过 `unwrap()` 统一处理
- 后端 API 变更时，只需更新 SDK 版本，上层代码无需修改
- 支持多种传输层（浏览器 fetch、Tauri 原生 HTTP）

**权衡**：

- SDK 增加约 30KB 体积（可接受，换来类型安全）
- SDK 版本需与后端保持兼容（通过 package.json 锁定版本）

### 2.4 SSE 事件 vs WebSocket 全量通信

**决策**：SSE 用于后端事件推送，WebSocket 仅用于终端 PTY 连接。

**理由**：

- SSE 是单向推送（服务器到客户端），天然适合事件通知场景
- SSE 基于 HTTP，无需特殊服务器配置，Nginx/Caddy 原生支持
- SSE 自动重连机制简单（浏览器原生支持，但本项目使用自定义 fetch + ReadableStream 实现更精细的控制）
- 终端 PTY 需要双向通信，WebSocket 更合适
- 分离关注点：事件流（SSE）与数据流（HTTP REST）解耦

**实现细节**：

- 全局单例 SSE 连接，所有 pane 通过 consumer 机制共享
- 指数退避重连策略（1s → 2s → 3s → 5s → 10s → 30s）
- 后台模式：更激进的重连策略 + keepalive 轮询
- 代次机制（`connectionGeneration`）：重连后自动丢弃旧连接的事件
- Tauri 桥接：Rust 侧使用 reqwest + Channel 实现 SSE，避免 WebView 限制

### 2.5 Feature-based 文件夹结构

**决策**：按功能模块组织代码（`features/chat/`、`features/message/` 等），而非按技术类型（`components/`、`hooks/`、`utils/`）。

**理由**：

- 高内聚：同一功能的相关组件、Hook、类型放在一起
- 低耦合：模块间通过明确的 API（store、context、hooks）交互
- 易于维护：修改某个功能时，只需关注对应目录
- 支持独立测试：每个 feature 目录可包含自己的 `.test.ts` 文件

**混合结构**：

- `src/features/`：业务功能模块（chat、message、sessions、settings、mention、slash-command、attachment）
- `src/components/`：跨功能复用的通用组件（Terminal、DiffView、CommandPalette、MarkdownRenderer 等）
- `src/hooks/`：跨功能复用的自定义 Hook
- `src/store/`：全局状态管理
- `src/api/`：后端通信层
- `src/utils/`：工具函数

---

## 3. 集成策略

### 3.1 数据流：API Layer → Store → UI Component

```
┌─────────────────────────────────────────────────────┐
│                    UI Components                     │
│  (App, ChatPane, Sidebar, SettingsDialog, etc.)     │
│         ↓ 用户操作                    ↑ 状态渲染       │
├─────────────────────────────────────────────────────┤
│                    Store Layer                       │
│  (messageStore, paneLayoutStore, themeStore, etc.)  │
│         ↓ 数据请求                    ↑ 状态更新       │
├─────────────────────────────────────────────────────┤
│                    API Layer                         │
│  (SDK Client → @opencode-ai/sdk → Backend REST)     │
└─────────────────────────────────────────────────────┘
```

**流程说明**：

1. 用户操作触发 UI 组件中的事件处理函数
2. 事件处理函数调用 API 层函数（如 `sendMessage()`）
3. API 层通过 SDK Client 发送 HTTP 请求到后端
4. 响应数据通过 `unwrap()` 提取后，更新对应 Store
5. Store 通过 `useSyncExternalStore` 通知 React 重新渲染
6. UI 组件读取最新状态并渲染

### 3.2 SSE 事件处理 → Store 更新

```
┌──────────────┐    SSE Stream     ┌──────────────────┐
│  OpenCode    │ ────────────────→ │  events.ts       │
│  Backend     │   (单向推送)       │  (单例连接)       │
└──────────────┘                   └────────┬─────────┘
                                           │ broadcastEvent()
                                           ↓
                              ┌────────────────────────┐
                              │  useGlobalEvents Hook  │
                              │  (App 组件中调用)       │
                              └────────────┬───────────┘
                                           │ 按 directory + sessionID 分发
                                           ↓
                              ┌────────────────────────┐
                              │  PaneController        │
                              │  (每个 pane 独立实例)   │
                              └────────────┬───────────┘
                                           │ 更新对应 store
                                           ↓
                              ┌────────────────────────┐
                              │  messageStore /        │
                              │  todoStore / etc.      │
                              └────────────────────────┘
```

**事件分发机制**：

- `useGlobalEvents` 在 App 组件中订阅 SSE 事件（全局唯一订阅点）
- 收到事件后，根据 `directory` 和 `sessionID` 匹配到对应的 pane
- 通过 `paneControllerStore` 获取对应 pane 的控制器
- 控制器调用对应 store 的更新方法
- 事件类型包括：`MESSAGE_UPDATED`、`MESSAGE_PART_UPDATED`、`MESSAGE_PART_DELTA`、`SESSION_UPDATED`、`TODO_UPDATED`、`PERMISSION_ASKED`、`QUESTION_ASKED` 等 20+ 种

### 3.3 Context Providers 共享状态

```
<StrictMode>
  <Suspense>
    <DirectoryProvider>          ← 工作目录上下文
      <SessionProvider>          ← 会话数据上下文
        <ChatViewportProvider>   ← 视口尺寸上下文（App 内部）
          <App />
        </ChatViewportProvider>
      </SessionProvider>
    </DirectoryProvider>
  </Suspense>
</StrictMode>
```

**各 Context 职责**：

- **DirectoryProvider**：管理当前工作目录、已保存目录列表、侧边栏展开状态
- **SessionProvider**：在应用启动时加载会话列表，提供会话数据
- **ChatViewportProvider**：提供响应式视口信息（可用宽度、侧边栏行为模式）

**Store 不使用 Context 的原因**：

- Store 是模块级单例，组件直接 import 即可访问
- `useSyncExternalStore` 已提供响应式订阅，无需 Context 传递
- 减少组件树层级，避免 Context 重渲染问题

### 3.4 路由与 Pane 同步

```
URL Hash ←──双向同步──→ Focused Pane Session
  #/session/abc            paneLayout.focusedSessionId = "abc"
  ?dir=/path/to/project    focusedRouteDirectory = "/path/to/project"
```

**同步机制**：

- URL → Pane：`routeSessionId` 变化时，`paneLayoutStore.setFocusedSession()` 同步到 store
- Pane → URL：`paneLayout.focusedSessionId` 变化时，`replaceSession()` 更新 URL hash
- 防抖：使用 `syncingFromRouteRef` 标记避免循环更新
- Directory 参数：与 sessionID 一起编码在 URL query string 中

---

## 4. 依赖关系图

```
                        ┌─────────────────┐
                        │   013-i18n      │
                        │  (国际化)        │
                        └────────┬────────┘
                                 │ 被所有 UI 模块依赖
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ↓                        ↓                        ↓
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│ 009-theme     │      │ 010-state     │      │ 001-api-layer │
│   (主题系统)   │      │   (状态管理)   │      │   (API 层)     │
└───────┬───────┘      └───────┬───────┘      └───────┬───────┘
        │                      │                      │
        │                      │                      │
        ↓                      ↓                      ↓
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│ 002-chat      │◄─────│ 012-pane      │      │ 004-session   │
│   (聊天界面)   │      │   (Pane 布局)  │      │   (会话管理)   │
└───────┬───────┘      └───────┬───────┘      └───────┬───────┘
        │                      │                      │
        │                      │                      │
        ↓                      ↓                      ↓
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│ 003-message   │      │ 008-terminal  │      │ 011-file-diff │
│   (消息渲染)   │      │   (终端系统)   │      │   (Diff 查看)  │
└───────┬───────┘      └───────────────┘      └───────────────┘
        │
        ↓
┌───────────────┐      ┌───────────────┐
│ 006-mention   │      │ 007-slash     │
│   (@ 提及)    │      │   (斜杠命令)   │
└───────────────┘      └───────────────┘


┌───────────────┐      ┌───────────────┐
│ 005-settings  │      │ 014-tauri     │
│   (设置面板)   │      │   (桌面应用)   │
└───────────────┘      └───────┬───────┘
        │                      │
        │ 依赖多个 store        │ 依赖 API 层 + 主题
        ↓                      ↓
┌───────────────┐      ┌───────────────┐
│ 015-docker    │      │ 016-router    │
│   (Docker 部署)│      │   (动态路由)   │
└───────────────┘      └───────────────┘
```

**详细依赖矩阵**：

| 模块                   | 依赖                                        | 被依赖                  |
| ---------------------- | ------------------------------------------- | ----------------------- |
| 001-api-layer          | 010-state (serverStore)                     | 002, 004, 008, 011, 014 |
| 002-chat-feature       | 001, 003, 006, 007, 009, 010, 012, 013, 016 | -                       |
| 003-message-rendering  | 001, 006, 009, 010, 011, 013                | 002                     |
| 004-session-management | 001, 010, 013                               | 002                     |
| 005-settings-panel     | 009, 010, 013, 014                          | -                       |
| 006-mention-system     | 001, 010, 013                               | 002, 003                |
| 007-slash-command      | 001, 010, 013                               | 002                     |
| 008-terminal-system    | 001, 010, 013                               | 002                     |
| 009-theme-system       | -                                           | 002, 003, 005, 008      |
| 010-state-management   | -                                           | 几乎所有模块            |
| 011-file-diff-viewer   | 001, 009, 013                               | 003                     |
| 012-pane-layout        | 010, 013                                    | 002                     |
| 013-i18n-localization  | -                                           | 所有 UI 模块            |
| 014-tauri-desktop      | 001, 009, 010                               | 002, 005, 008           |
| 015-docker-deployment  | 016                                         | -                       |
| 016-router-service     | -                                           | 015                     |

---

## 5. 风险区域

### 5.1 SSE 连接管理（高风险）

**复杂度来源**：

- 全局单例 SSE 连接需要服务所有 pane 的事件需求
- 多场景重连：网络断开、页面后台/前台切换、服务器切换、心跳超时
- Tauri 桥接：Rust 侧 reqwest + Channel 实现，需要串行化 disconnect/connect 操作
- 代次机制（`connectionGeneration`）：重连后需丢弃旧连接的事件，避免状态不一致

**缓解措施**：

- 后台 keepalive 轮询（30s 间隔），弥补移动端 timer 冻结问题
- 指数退避重连 + 后台更激进策略
- `pendingDisconnect` Promise 串行化 Tauri 侧的 disconnect/connect
- 完善的连接状态机（connecting → connected → disconnected → error）

### 5.2 Message Store 状态管理（高风险）

**复杂度来源**：

- 消息 store 是最大的 store（约 1000+ 行），管理所有会话的消息状态
- 流式消息的增量更新（part delta）需要精确的 diff 和合并逻辑
- Undo/Redo 状态管理（revert history）
- 多 pane 场景下，每个 pane 独立消费同一份消息数据

**缓解措施**：

- 细粒度 hooks（`useMessages`、`useIsStreaming` 等）减少不必要的重渲染
- 不可变更新模式，确保状态一致性
- 单元测试覆盖核心逻辑（`messageStore.test.ts`、`messageStoreHooks.test.tsx`）

### 5.3 Pane 布局树与路由同步（中高风险）

**复杂度来源**：

- 二叉分割树的递归操作（split、close、focus）需要精确的树遍历
- URL hash 与 focused pane session 的双向同步，需防止循环更新
- 全屏模式与分割模式的切换
- 每个 pane 独立管理自己的会话状态（paneController）

**缓解措施**：

- `syncingFromRouteRef` 标记防止循环更新
- 不可变树操作，每次操作返回新树
- PaneController 封装会话操作，隔离各 pane 的状态

### 5.4 跨平台路径处理（中风险）

**复杂度来源**：

- Windows 使用反斜杠，Linux/macOS 使用正斜杠
- URL 编码/解码路径时的特殊字符处理
- 不同服务器可能是不同操作系统（路径模式缓存）

**缓解措施**：

- `formatPathForApi()` 统一转换为正斜杠
- `normalizeToForwardSlash()` 处理输入路径
- `resetPathModeCache()` 在服务器切换时重置路径模式缓存

### 5.5 Tauri 原生集成（中风险）

**复杂度来源**：

- Tauri fetch 与浏览器 fetch 的差异
- SSE 连接在 Tauri 中通过 Rust 桥接，增加了一层复杂度
- Safe area 适配（刘海屏、状态栏）
- 自动启动 opencode serve 进程的管理

**缓解措施**：

- `isTauri()` 统一检测运行环境
- `getSDKClientAsync()` 确保 Tauri fetch 就绪后再创建 SDK client
- `ensureRandomUUID()` polyfill 处理非 HTTPS 环境下 `crypto.randomUUID` 缺失
- viewport meta 动态注入 `viewport-fit=cover`

### 5.6 主题系统与性能（低风险）

**复杂度来源**：

- 主题切换时需要重新计算所有 CSS 变量
- 终端主题需要从 CSS 变量动态读取颜色
- 自定义 CSS 注入可能影响布局

**缓解措施**：

- 主题初始化在 React 渲染前完成，避免闪烁
- Canvas 2D 万能颜色转换（处理 rgb、rgba、oklch 等各种格式）
- 自定义 CSS 通过独立 `<style>` 标签注入，便于管理

---

## 6. 初始化流程

```
main.tsx
  │
  ├─ 1. ensureRandomUUID()          ← Polyfill crypto.randomUUID
  ├─ 2. history.scrollRestoration   ← 禁用浏览器滚动恢复
  ├─ 3. themeStore.init()           ← 初始化主题（注入 CSS 变量）
  ├─ 4. initOverlayScrollbars()     ← 初始化自定义滚动条
  ├─ 5. serverStore.onServerChange  ← 注册服务器切换回调
  │    ├─ invalidateSDKClient()
  │    ├─ messageStore.clearAll()
  │    ├─ childSessionStore.clearAll()
  │    ├─ todoStore.clearAll()
  │    ├─ resetPathModeCache()
  │    ├─ autoApproveStore.reloadFromStorage()
  │    └─ reconnectSSE()
  ├─ 6. Tauri 初始化（如果是 Tauri 环境）
  │    ├─ 添加 .tauri-app class
  │    ├─ 注入 viewport-fit=cover
  │    └─ 自动启动 opencode serve（如果开启）
  ├─ 7. 全局错误处理
  │    ├─ window.addEventListener('error')
  │    └─ window.addEventListener('unhandledrejection')
  └─ 8. bootstrap()
       ├─ getSDKClientAsync()（Tauri 环境）
       └─ createRoot().render()
            ├─ DirectoryProvider
            ├─ SessionProvider
            └─ App
```

---

## 7. 构建与部署

### 7.1 开发环境

```
Vite Dev Server (:5173)
  ├── 热模块替换（HMR）
  ├── /api 代理到 http://127.0.0.1:4096
  └── TypeScript 类型检查
```

### 7.2 生产构建

```
npm run validate
  ├── TypeScript 类型检查（tsc --noEmit）
  ├── ESLint 代码检查
  ├── 单元测试（Vitest）
  └── 生产构建（Vite build）
```

### 7.3 Docker 部署

```
docker compose up -d
  ├── Gateway (Caddy + Router)
  │    ├── :6658 → 统一入口
  │    └── :6659 → 预览端口
  ├── Frontend (Caddy 静态托管)
  │    └── :3000（内部）
  └── Backend (OpenCode + mise)
       └── :4096（内部）
```

### 7.4 Tauri 桌面构建

```
npm run tauri build
  ├── 前端生产构建（Vite）
  ├── Rust 编译（cargo build --release）
  └── 打包为 .app / .deb / .rpm / .exe
```

---

## 8. 测试策略

| 测试类型 | 工具                     | 覆盖范围                         |
| -------- | ------------------------ | -------------------------------- |
| 单元测试 | Vitest                   | Store 逻辑、工具函数、API 客户端 |
| 组件测试 | Vitest + Testing Library | 独立组件渲染与交互               |
| 类型检查 | TypeScript (tsc)         | 全项目类型安全                   |
| 代码规范 | ESLint                   | 代码风格与最佳实践               |

测试文件与源文件同目录放置（`*.test.ts`、`*.test.tsx`），便于维护。
