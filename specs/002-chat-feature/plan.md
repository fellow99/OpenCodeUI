# 002-chat-feature 实现计划

> **模块路径**: `src/features/chat/`
> **版本**: 1.0
> **状态**: 已实现（As-Built 回溯）
> **生成日期**: 2026-04-12

---

## 1. Technical Context

### 1.1 技术栈

| 技术             | 版本     | 用途        |
| ---------------- | -------- | ----------- |
| React            | ^19.2.0  | UI 组件框架 |
| TypeScript       | ~5.9.3   | 类型系统    |
| Tailwind CSS     | ^4.2.2   | 原子化 CSS  |
| motion           | ^12.38.0 | 动画与过渡  |
| i18next          | ^26.0.1  | 国际化      |
| @opencode-ai/sdk | ^1.4.1   | 后端通信    |

### 1.2 模块文件清单

模块共 27 个文件（含 `sidebar/` 子目录 10 个、`input/` 子目录若干），核心文件如下：

| 文件                            | 行数 | 职责                                                    |
| ------------------------------- | ---- | ------------------------------------------------------- |
| `ChatPane.tsx`                  | 717  | 单个聊天窗格，组合 Header/ChatArea/InputBox/Dialogs     |
| `InputBox.tsx`                  | 1110 | 消息输入框，集成 @mention、/slash-command、附件、工具栏 |
| `ChatArea.tsx`                  | 583  | 消息滚动区域，column-reverse 布局 + 虚拟渲染            |
| `Header.tsx`                    | 275  | 单窗格模式完整头部（模型选择器、分屏按钮、面板切换）    |
| `SplitContainer.tsx`            | 205  | 递归分屏容器，管理窗格树和分隔线拖拽                    |
| `Sidebar.tsx`                   | 326  | 侧边栏外壳，处理开/关、宽度调整、移动端覆盖             |
| `ModelSelector.tsx`             | 615  | 统一模型选择器，服务桌面端和移动端                      |
| `PermissionDialog.tsx`          | 219  | 弹窗式权限确认面板                                      |
| `EmptyState.tsx`                | 195  | 无会话时的空状态页面                                    |
| `chatViewport.tsx`              | 370  | 响应式视口计算系统                                      |
| `sidebar/SidePanel.tsx`         | 1243 | 侧边栏内容面板（Header、导航、标签页、Footer）          |
| `sidebar/FolderRecentList.tsx`  | 1037 | 按项目分组的最近会话列表，支持拖拽排序                  |
| `sidebar/ActiveSessionItem.tsx` | 98   | 活跃会话列表项，显示状态指示器                          |
| `sidebar/SidebarFooter.tsx`     | 388  | 侧边栏底部，含上下文使用量、主题切换、设置入口          |

### 1.3 外部依赖

- **`features/message/`**: `MessageRenderer` 负责消息内容渲染
- **`features/mention/`**: `MentionMenu`、`detectMentionTrigger` 提供 @ 提及功能
- **`features/slash-command/`**: `SlashCommandMenu`、`detectSlashTrigger` 提供 / 命令功能
- **`features/attachment/`**: `AttachmentPreview` 提供文件附件预览
- **`features/sessions/`**: `SessionList`、`useSessions` 提供会话列表管理
- **`store/`**: `messageStore`、`paneLayoutStore`、`paneControllerStore`、`layoutStore`、`autoApproveStore`、`notificationStore`、`childSessionStore`、`activeSessionStore`
- **`hooks/`**: `useChatSession`、`useModels`、`useModelSelection`、`useDirectory`、`useGlobalEvents`、`useGlobalKeybindings`、`useRouter`、`useTheme`
- **`components/`**: `OutlineIndex`、`DiffView`、`ContentBlock`、`DropdownMenu`、`CircularProgress`、`ConfirmDialog`

### 1.4 与 App.tsx 的集成

`App.tsx`（591 行）作为全局编排中心，组合四大布局区域：

```
App
├── Sidebar（侧边栏）
├── SplitContainer（分屏容器）
│   └── ChatPane × N（聊天窗格）
├── BottomPanel（底部面板 — 终端）
├── RightPanel（右侧面板）
└── 懒加载对话框（SettingsDialog、CommandPalette、CloseServiceDialog）
```

关键集成点：

- `ChatViewportProvider` 包裹整个应用，提供响应式视口上下文
- `useGlobalEvents(activeDirectories)` 建立全局唯一 SSE 连接
- URL Hash 与 `paneLayout.focusedSessionId` 双向同步，通过 `syncingFromRouteRef` 防止循环更新
- `paneControllerStore` 为每个 pane 注册控制器，快捷键通过控制器代理会话操作

---

## 2. Constitution Check

| 宪法原则                    | 符合性 | 说明                                                                                   |
| --------------------------- | ------ | -------------------------------------------------------------------------------------- |
| 原则 1: AI 驱动开发         | 符合   | 代码结构清晰，注释充分，符合 Vibe Coding 范式                                          |
| 原则 2: OpenCode 兼容性优先 | 符合   | 使用 `@opencode-ai/sdk` 官方 SDK，API 调用与后端规范一致                               |
| 原则 3: 多平台统一代码库    | 符合   | Web 端与 Tauri 桌面端共享同一套 `src/` 代码，通过 `isTauri()` 检测环境差异             |
| 原则 4: 自定义优于框架依赖  | 符合   | 自定义 Hash 路由（`useRouter`）、自定义 Store 模式（非 Redux/Zustand）、自定义视口计算 |
| 原则 5: 实时通信优先        | 符合   | SSE 为第一优先级，`useGlobalEvents` 全局单例连接，指数退避重连                         |
| 原则 6: 中文优先文档        | 符合   | 代码注释以中文为主，i18n 键值管理用户可见文本                                          |
| 原则 10: 模块化功能架构     | 符合   | Feature-Sliced 目录组织，模块内自包含，通过明确接口（store、hooks、context）通信       |

| 约束条件              | 符合性 | 说明                                              |
| --------------------- | ------ | ------------------------------------------------- |
| C1: GPL-3.0 许可证    | 符合   | 项目保持 GPL-3.0                                  |
| C2: OpenCode 后端兼容 | 符合   | 仅依赖 OpenCode 后端 API                          |
| C3: 构建校验          | 符合   | 模块包含 `.test.ts` 测试文件                      |
| C4: 依赖最小化        | 符合   | 未引入额外重量级依赖                              |
| C5: SSE 支持          | 符合   | `column-reverse` + SSE 流式推送，代理配置保留 SSE |

---

## 3. Research Findings

### 3.1 核心架构模式

**Pane（窗格）作为基本渲染单元**：`ChatPane` 是唯一的聊天表面组件，通过 `displayMode` prop 区分单窗格（`single`）和分屏（`split`）两种模式。分屏模式下使用 `PaneHeader` 替代 `Header`，并包裹在 `PANE_VIEWPORT` 常量视口中。

**递归分屏树**：`SplitContainer` 递归渲染 `PaneNode` 二叉树。叶子节点渲染 `ChatPane`，内部节点渲染 CSS Grid 容器 + 可拖拽分隔线。拖拽调整比例时直接操作 DOM（`gridTemplateColumns/Rows`），rAF 节流，绕过 React 渲染循环。

**column-reverse 底部锚定**：`ChatArea` 使用 `flex-direction: column-reverse` 实现零 JS 的 stick-to-bottom。`scrollTop=0` 表示底部，新内容向上生长，浏览器自动维持底部锚定。IntersectionObserver 在视觉顶部触发 `loadMore`。

### 3.2 响应式策略

`chatViewport.tsx` 是响应式计算引擎，通过 `useChatViewportController` Hook 在 `App.tsx` 中初始化：

| 断点                              | 值     | 效果                            |
| --------------------------------- | ------ | ------------------------------- |
| `CHAT_VIEWPORT_MOBILE_BREAKPOINT` | 768px  | 面板切换为 overlay 模式         |
| `CHAT_SURFACE_COMPACT_BREAKPOINT` | 680px  | 聊天表面切换为 compact 变体     |
| `SMALL_DESKTOP_BREAKPOINT`        | 1100px | 侧边栏/右面板最大宽度响应式缩放 |
| `CHAT_SPLIT_TOUCH_MIN_WIDTH`      | 900px  | 触摸设备分屏最小宽度（需横屏）  |

视口计算考虑侧边栏展开状态、右面板宽度、用户自定义侧边栏宽度，确保聊天表面最小 380px。当空间不足时，优先压缩右面板，再压缩侧边栏。

### 3.3 状态管理架构

模块不直接管理状态，而是通过消费全局 Store 实现：

| Store                 | 消费方式                | 用途                                                    |
| --------------------- | ----------------------- | ------------------------------------------------------- |
| `messageStore`        | `useChatSession` Hook   | 消息加载、流式状态、权限/提问请求                       |
| `paneLayoutStore`     | `usePaneLayout` Hook    | 窗格树结构、焦点窗格、全屏状态                          |
| `paneControllerStore` | `setController` 注册    | 每个 pane 注册操作代理（newSession、archiveSession 等） |
| `autoApproveStore`    | 直接 import             | Full Auto 模式切换、自动批准规则                        |
| `layoutStore`         | `useLayoutStore` Hook   | 侧边栏/右面板/底面板开关状态                            |
| `notificationStore`   | `useNotifications` Hook | 通知历史、未读计数                                      |
| `activeSessionStore`  | `useBusySessions` Hook  | 活跃会话跟踪                                            |
| `childSessionStore`   | 直接 import             | 子会话关系树                                            |

### 3.4 性能优化策略

| 优化点                  | 实现方式                                                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 消息虚拟渲染            | `ViewportMessageItem` 使用 IntersectionObserver（`rootMargin: 150%`）判断是否在视口附近，远离视口时仅保留 `measuredHeight` 占位 |
| 流式消息强制渲染        | 最后 8 条消息（`STICKY_RENDER_MESSAGE_COUNT`）始终渲染，不受虚拟渲染影响                                                        |
| 分屏拖拽性能            | 直接操作 DOM `gridTemplate`，rAF 节流，子元素 `contain: layout style` 隔离重排                                                  |
| 全屏模式性能            | 非全屏叶子使用 `content-visibility: hidden` + `width/height: 0`，避免卸载/重装开销                                              |
| 侧边栏宽度调整          | rAF 节流直接操作 DOM `style.width`，松手后持久化到 localStorage                                                                 |
| 连续助手消息分组        | `messageGroups` 将连续 assistant 消息合并渲染，减少 DOM 节点                                                                    |
| 控制器 actions 稳定引用 | `controllerActionsRef` + `useMemo` 避免 `stableControllerActions` 变化导致重渲染                                                |

---

## 4. Data Model

### 4.1 模块内部实体

#### ChatPane（聊天窗格）

```typescript
interface ChatPaneProps {
  paneId: string // 窗格唯一标识
  sessionId: string | null // 当前关联的会话 ID
  isFocused: boolean // 是否为当前聚焦窗格
  paneCount: number // 当前窗格总数
  displayMode: 'single' | 'split' // 展示模式
  isPaneFullscreen: boolean // 是否全屏
  // 回调
  onOpenSidebar: () => void
  onSplitPane: () => void
  onTogglePaneFullscreen: () => void
  navigatePaneToSession: (paneId, sessionId, directory?) => void
  navigatePaneHome: (paneId) => void
}
```

#### ChatViewport（视口上下文）

```typescript
interface ChatViewportValue {
  presentation: {
    surfaceVariant: 'desktop' | 'compact'
    isCompact: boolean
  }
  interaction: {
    mode: 'pointer' | 'touch'
    touchCapable: boolean
    sidebarBehavior: 'docked' | 'overlay'
    rightPanelBehavior: 'docked' | 'overlay'
    bottomPanelBehavior: 'docked' | 'overlay'
    outlineInteraction: 'pointer' | 'touch'
    enableCollapsedInputDock: boolean
  }
  layout: {
    viewportWidth: number
    viewportHeight: number
    surfaceWidth: number
    surfaceMinWidth: number          // 380px
    sidebar: { railWidth, requestedWidth, openWidth, dockedWidth, overlayWidth, hardMinWidth, ... }
    rightPanel: { requestedWidth, dockedWidth, hardMinWidth, maxWidth, ... }
    bottomPanel: { maxHeight: number }
  }
  actions: { setSidebarRequestedWidth: (width: number) => void }
}
```

#### InputBox 状态

```typescript
// 内部状态
text: string
attachments: Attachment[]           // 文件、@提及、/命令、Agent
mentionOpen/slashOpen: boolean      // 菜单开关
mentionQuery/slashQuery: string     // 搜索查询
isSubmitting: boolean               // 提交中
isDragging: boolean                 // 文件拖拽中
isCollapsed: boolean                // 移动端收起状态

// CollapsedDialogInfo（收起的对话框胶囊）
interface CollapsedDialogInfo {
  label: string                     // 显示文本
  queueLength: number               // 队列剩余数量
  onExpand: () => void              // 展开回调
}
```

### 4.2 数据流

```
用户输入 → InputBox.handleSend
              ↓
        useChatSession.handleSend
              ↓
        messageStore.sendMessage → SDK → 后端 API
                                            ↓
        后端 SSE 流 ← events.ts（全局单例）
              ↓
        useGlobalEvents（按 directory + sessionID 分发）
              ↓
        paneController → messageStore 更新
              ↓
        ChatArea 实时渲染（column-reverse + 虚拟渲染）
              ↓
        权限/提问请求 → PermissionDialog / QuestionDialog
              ↓
        用户回复 → handlePermissionReply / handleQuestionReply → 后端 API
```

侧边栏数据流：

```
SessionProvider（加载会话列表）
       ↓
SidePanel → FolderRecentList / SessionList / ActiveSessionItem
       ↓
用户选择会话 → onSelectSession → navigatePaneToSession → URL 更新 → ChatPane 加载消息
```

### 4.3 窗格树结构

```typescript
type PaneNode = PaneLeaf | PaneSplit

interface PaneLeaf {
  type: 'leaf'
  id: string // paneId
  sessionId: string | null
}

interface PaneSplit {
  type: 'split'
  id: string
  direction: 'horizontal' | 'vertical'
  ratio: number // 0.1 ~ 0.9
  first: PaneNode
  second: PaneNode
}
```

---

## 5. Interface Contracts

### 5.1 ChatPane 对外接口

ChatPane 通过 `paneControllerStore.setController` 向全局注册控制器，供快捷键和外部操作调用：

```typescript
interface PaneController {
  paneId: string
  sessionId: string | null
  effectiveDirectory: string
  contextLimit?: number
  isStreaming: boolean
  // 操作
  newSession: () => void
  archiveSession: () => void
  previousSession: () => void
  nextSession: () => void
  toggleAgent: () => void
  copyLastResponse: () => void
  cancelMessage: () => void
  openModelSelector: () => void
  toggleFullAuto: () => void
}
```

### 5.2 ChatArea 命令式句柄

```typescript
interface ChatAreaHandle {
  scrollToBottom: (instant?: boolean) => void
  scrollToBottomIfAtBottom: () => void
  scrollToLastMessage: () => void
  scrollToMessageIndex: (index: number) => void
  scrollToMessageId: (messageId: string) => void
}
```

### 5.3 ModelSelector 命令式句柄

```typescript
interface ModelSelectorHandle {
  openMenu: () => void
}
```

### 5.4 InlineToolRequestContext

将 pending 权限/提问请求注入消息流，供 `InlinePermission` 和 `InlineQuestion` 消费：

```typescript
interface InlineToolRequestContextValue {
  pendingPermissions: ApiPermissionRequest[]
  pendingQuestions: ApiQuestionRequest[]
  onPermissionReply: (requestId, reply, directory) => void
  onQuestionReply: (requestId, answers, directory) => void
  onQuestionReject: (requestId, directory) => void
  isReplying: boolean
}
```

### 5.5 SplitContainer 渲染接口

```typescript
interface SplitContainerProps {
  node: PaneNode
  renderLeaf: (paneId: string, sessionId: string | null) => React.ReactNode
  fullscreenPaneId?: string | null
}
```

---

## 6. Implementation Strategy

### 6.1 组件层次结构

```
App.tsx
├── ChatViewportProvider
│   ├── Sidebar
│   │   ├── SidePanel
│   │   │   ├── Header（Logo + Toggle + New Chat + Project Selector）
│   │   │   ├── Search Input
│   │   │   ├── Tab Bar（Recents / Active）
│   │   │   ├── FolderRecentList（Recents tab）
│   │   │   │   ├── FolderRecentSection × N
│   │   │   │   │   ├── SessionListItem × N
│   │   │   │   │   └── SessionChildrenSlot
│   │   │   │   └── WorkspaceFolderList（多工作区时）
│   │   │   ├── SessionList（搜索/全局模式）
│   │   │   ├── ActiveSessionTree（Active tab）
│   │   │   │   └── ActiveSessionItem × N
│   │   │   └── NotificationItem × N
│   │   └── SidebarFooter
│   │       ├── StatusIndicator（环形进度 + 连接状态点）
│   │       ├── 浮动菜单（Context Usage / Theme / Settings / Share）
│   │       ├── ContextDetailsDialog
│   │       └── ShareDialog
│   │   └── ProjectDialog
│   │
│   ├── Surface（聊天区域容器）
│   │   └── SplitContainer
│   │       ├── SplitNode（递归）
│   │       │   ├── 子 SplitNode / ChatPane
│   │       │   ├── Draggable Divider
│   │       │   └── 子 SplitNode / ChatPane
│   │       └── ChatPane（叶子节点）
│   │           ├── Header（单窗格模式）或 PaneHeader（分屏模式）
│   │           ├── ChatArea
│   │           │   ├── Session Loading Spinner
│   │           │   ├── ViewportMessageItem × N（虚拟渲染）
│   │           │   │   └── MessageRenderer
│   │           │   ├── RetryStatusInline
│   │           │   └── Top Sentinel（IntersectionObserver）
│   │           ├── OutlineIndex（大纲导航）
│   │           ├── InputBox
│   │           │   ├── MentionMenu
│   │           │   ├── SlashCommandMenu
│   │           │   ├── AttachmentPreview
│   │           │   ├── InputToolbar
│   │           │   ├── InputFooter
│   │           │   ├── FloatingActions
│   │           │   └── CollapsedCapsule
│   │           ├── PermissionDialog（弹窗模式）
│   │           └── QuestionDialog（弹窗模式）
│   │
│   ├── BottomPanel（终端面板）
│   └── RightPanel（文件/变更面板）
│
└── 懒加载对话框（Suspense）
    ├── SettingsDialog
    ├── CommandPalette
    └── CloseServiceDialog
```

### 6.2 关键实现细节

#### 6.2.1 ChatPane 生命周期

1. **初始化**：加载模型列表（`useModels`）、恢复模型选择（`useModelSelection`）、注册 PaneController
2. **会话保护**：`useEffect` 调用 `messageStore.protectSession(routeSessionId)` 防止当前查看的会话被内存回收，卸载时 `unprotectSession`
3. **模型/Agent 恢复**：从 `revertedContent` 或最后一条用户消息中恢复模型和 Agent 选择
4. **控制器注册**：通过 `paneControllerStore.setController` 注册操作代理，使用 `controllerActionsRef` + `stableControllerActions` 模式保证引用稳定

#### 6.2.2 InputBox 输入处理

- **@ 提及触发**：`detectMentionTrigger` 检测 `@` 字符，打开 `MentionMenu`，支持文件夹层级浏览（右箭头进入、左箭头返回）
- **/ 命令触发**：`detectSlashTrigger` 检测行首或空白后的 `/`，打开 `SlashCommandMenu`
- **文件附件**：根据 `fileCapabilities`（image/pdf/audio/video）判断模型支持的文件类型，支持拖拽、粘贴、文件选择器
- **输入历史**：`useInputHistory` Hook 实现上下箭头浏览发送历史，类终端体验
- **移动端收起**：`useMobileCollapse` Hook 实现滚动收起/点击展开交互
- **乐观命令提交**：`submitCommandOptimistically` 先清空输入框执行命令，失败时恢复草稿

#### 6.2.3 ChatArea 滚动与虚拟渲染

- **column-reverse 布局**：DOM 顺序反转（最新在前），`scrollTop=0` 为底部
- **底部锚定**：用户滚离底部时 `isAtBottomRef.current = false`，停止自动跟随；滚回底部时恢复
- **历史加载**：Top Sentinel（`IntersectionObserver`，`rootMargin: 200px`）触发 `loadMore`，`loadMoreBlockedRef` 防止初始加载时误触发
- **虚拟渲染**：`ViewportMessageItem` 使用 `IntersectionObserver`（`rootMargin: 150%`）判断可见性，不可见时渲染占位 `div`（`height: measuredHeight`）
- **高度测量**：`ResizeObserver` 测量已渲染内容高度，缓存到 `measuredHeight`
- **消息分组**：连续助手消息合并到同一容器渲染，形成连贯回复流

#### 6.2.4 SplitContainer 分屏拖拽

- **CSS Grid 布局**：`gridTemplateColumns/Rows` 控制分割比例，格式如 `"49.5fr 6px 50.5fr"`
- **拖拽实现**：`pointerdown` 开始，`pointermove` 计算比例，`requestAnimationFrame` 节流直接修改 DOM
- **分隔线 Hit Area**：可见 6px + 两侧各 4px 不可见扩展 = 14px 总点击区域
- **比例限制**：`MIN_RATIO = 0.1`，`MAX_RATIO = 0.9`
- **全屏模式**：全屏叶子 `position: absolute; inset: 0`，非全屏叶子 `content-visibility: hidden` + `width/height: 0`

#### 6.2.5 Sidebar 侧边栏

- **双模式**：桌面端 `docked`（push 布局），移动端 `overlay`（覆盖式，带 backdrop）
- **宽度调整**：鼠标拖拽 + 触摸拖拽，rAF 节流，宽度持久化到 `localStorage`（key: `sidebar-width`）
- **移动端滑动手势**：`touchStartX` / `touchDeltaX` 追踪滑动距离，超过 80px 松手自动关闭
- **项目分组**：Git 仓库自动识别根目录，同仓库子目录归入同一项目；多工作区下按工作区子目录进一步分组
- **编辑模式**：支持会话/项目多选，Shift+点击范围选择，批量删除/移除
- **活跃会话树**：`buildActiveSessionTree` 将扁平列表构建为树形结构，子会话挂在父会话下方
- **状态聚合**：项目分组收起时，聚合显示该目录下所有活跃会话的最高优先级状态（permission > question > retry > working）

### 6.3 国际化

所有用户可见文本通过 i18n 键值管理，涉及两个命名空间：

- `chat`：聊天界面专属文本（`chat:header.*`、`chat:inputBox.*`、`chat:sidebar.*` 等）
- `common`：通用文本（`common:reject`、`common:delete`、`common:loadingMore` 等）

组件通过 `useTranslation(['chat', 'common'])` 按需加载。

---

## 7. Error Handling

### 7.1 错误处理策略

| 场景             | 处理方式                                                           |
| ---------------- | ------------------------------------------------------------------ |
| 会话加载失败     | `loadState: 'error'`，ChatArea 显示加载动画（延迟 150ms 避免闪烁） |
| 消息发送失败     | `runSubmit` 捕获异常，`onFailure` 回调恢复草稿                     |
| 重命名失败       | `uiErrorHandler('rename session', e)` 统一错误处理                 |
| 批量删除失败     | `Promise.allSettled` 逐个处理，单个失败不影响其他                  |
| 活跃会话数据缺失 | 异步 `getSession` 拉取，失败时 `opacity: 50` 显示，不阻塞 UI       |
| 模型加载失败     | `modelsLoading` 状态，ModelSelector 显示 `...` 占位                |
| 文件处理失败     | `console.warn` 跳过失败文件，不影响其他附件                        |
| 命令执行失败     | 乐观提交后恢复草稿（`restoreDraft`）                               |

### 7.2 重试机制

`RetryStatusInline` 组件展示连接重试状态：

- 显示重试次数（`attempt`）和下次重试倒计时
- 使用 `role="status"` 和 `aria-live="polite"` 提供可访问性
- 可展开显示错误详情

### 7.3 防御性编程

- `loadMoreBlockedRef` 防止 session 初始加载时 sentinel 误触发
- `syncingFromRouteRef` 防止 URL 与 pane 状态循环更新
- `latestDraftRef` 保存最新草稿快照，用于命令失败时恢复
- `cancelled` 标志防止异步操作在组件卸载后更新状态
- `ignoreMouseRef` 防止 ModelSelector 打开时误触发 hover 高亮

---

## 8. Testing Considerations

### 8.1 已有测试文件

| 测试文件                            | 测试内容                |
| ----------------------------------- | ----------------------- |
| `ChatArea.test.ts`                  | ChatArea 组件渲染与交互 |
| `InputBox.test.tsx`                 | InputBox 组件渲染与交互 |
| `InputBoxAttachments.test.tsx`      | 输入框附件功能          |
| `ModelSelector.test.tsx`            | 模型选择器功能          |
| `ProjectDialog.test.tsx`            | 项目选择对话框          |
| `sidebar/activeSessionTree.test.ts` | 活跃会话树构建逻辑      |

### 8.2 建议测试覆盖

#### 单元测试

- **`chatViewport.tsx`**：`computeChatViewport` 纯函数，可测试各种视口宽度下的计算结果
- **`canUseSplitPane`**：测试触摸设备/非触摸设备的分屏条件判断
- **`activeSessionTree.ts`**：树形结构构建逻辑，父子关系正确性
- **`sidebarUtils.ts`**：路径处理、时间格式化等工具函数

#### 组件测试

- **ChatArea**：
  - column-reverse 布局下新消息到达时的滚动行为
  - 用户滚离底部后停止自动跟随
  - IntersectionObserver 触发 loadMore
  - 虚拟渲染：远离视口的消息仅渲染占位
  - 会话切换时 snap to bottom

- **InputBox**：
  - @ 提及触发、导航、选择、文件夹浏览
  - / 命令触发、导航、选择
  - 文件拖拽、粘贴、选择
  - 输入历史导航（上下箭头）
  - 移动端收起/展开
  - 发送后清空并聚焦

- **SplitContainer**：
  - 递归渲染窗格树
  - 分隔线拖拽调整比例
  - 全屏模式切换

- **Sidebar**：
  - 桌面端 docked 模式
  - 移动端 overlay 模式 + 滑动手势
  - 宽度调整与持久化
  - 项目分组与 Git 识别
  - 编辑模式与批量操作

- **ModelSelector**：
  - 搜索过滤
  - 分组展示（置顶、最近、按提供商）
  - 键盘导航
  - 触摸设备长按置顶

#### 集成测试

- URL Hash 与 pane 状态双向同步
- SSE 事件分发到对应 pane
- 快捷键触发 pane 操作
- 分屏模式下各 pane 独立会话管理

### 8.3 测试工具链

| 工具                      | 版本    | 用途           |
| ------------------------- | ------- | -------------- |
| Vitest                    | ^4.1.2  | 单元测试框架   |
| jsdom                     | ^29.0.1 | DOM 环境模拟   |
| @testing-library/react    | ^16.3.0 | React 组件测试 |
| @testing-library/jest-dom | ^6.9.1  | DOM 断言匹配器 |

---

## 9. Risk Areas

### 9.1 高风险：ChatArea 虚拟渲染 + column-reverse

**复杂度来源**：

- `column-reverse` 下 DOM 顺序与视觉顺序相反，IntersectionObserver 的 `rootMargin` 计算需要特别注意
- 虚拟渲染的 `measuredHeight` 缓存可能在内容变化后失效（如流式消息增长）
- `STICKY_RENDER_MESSAGE_COUNT = 8` 的硬编码可能不适配所有场景

**缓解措施**：

- 流式消息和最后 8 条消息强制渲染（`forceRender` prop）
- `ResizeObserver` 持续测量已渲染内容高度
- 会话切换时重置所有状态

### 9.2 高风险：Pane 布局树与路由同步

**复杂度来源**：

- URL Hash 与 `focusedSessionId` 双向同步，需防止循环更新
- 多 pane 场景下每个 pane 独立管理会话状态
- 全屏模式与分割模式切换时的状态保持

**缓解措施**：

- `syncingFromRouteRef` 标记防止循环
- `paneControllerStore` 封装会话操作，隔离各 pane 状态
- 全屏模式使用 `content-visibility: hidden` 而非卸载组件

### 9.3 中风险：侧边栏活跃会话树

**复杂度来源**：

- 活跃会话可能来自不同目录，需要跨目录拉取 session 元数据
- 子会话挂在父会话下方，需要正确构建父子关系
- 状态聚合（收起时显示最高优先级状态）需要遍历所有子会话

**缓解措施**：

- `buildActiveSessionTree` 纯函数构建树形结构
- `fetchedSessions` 缓存跨目录拉取的 session 数据
- `sessionLookup` 合并 sessions 列表和 fetchedSessions

### 9.4 中风险：响应式视口计算

**复杂度来源**：

- 多变量输入（侧边栏状态、右面板、自定义宽度、触摸能力）
- 空间不足时的压缩策略（右面板优先、侧边栏次之）
- 混合设备（触摸屏笔记本）的交互模式判断

**缓解措施**：

- `computeChatViewport` 为纯函数，易于测试
- `useChatViewportController` 封装所有状态和计算
- `preferTouchUi` / `hasCoarsePointer` / `hasTouch` 多维度判断触摸能力

---

## 10. Future Improvements

以下改进方向不在当前实现范围内，但可作为后续优化参考：

1. **虚拟渲染优化**：当前 `STICKY_RENDER_MESSAGE_COUNT = 8` 为硬编码，可改为基于视口高度的动态计算
2. **分屏布局持久化**：当前分屏比例在页面刷新后丢失，可考虑持久化到 localStorage
3. **侧边栏性能**：`FolderRecentList` 在大量项目时可能存在渲染性能问题，可考虑虚拟列表
4. **键盘导航增强**：侧边栏会话列表的键盘导航（Arrow Up/Down 选择、Enter 打开）尚未完全实现
5. **无障碍优化**：部分交互元素缺少 `aria-label` 或 `role` 属性，可进一步完善
