# 012-pane-layout 模块 — 技术方案（As-Built）

> 模块编号：012-pane-layout
> 状态：已实现
> 最后更新：2026-04-12
> 本文档为回溯性技术方案，记录"实际建成"的架构设计。

---

## 1. Technical Context

### 1.1 模块定位

012-pane-layout 是 OpenCodeUI 的核心布局模块，负责管理聊天区域的多窗格分割、面板（右侧/底部）的开关与尺寸、以及面板标签页的统一管理。模块由三个 Store、若干组件和一个工具 Hook 组成，通过 `useSyncExternalStore` 桥接 React 渲染。

### 1.2 技术栈

| 维度     | 选型                                     |
| -------- | ---------------------------------------- |
| 框架     | React 19 + TypeScript                    |
| 状态管理 | 自定义 Store + `useSyncExternalStore`    |
| 布局渲染 | CSS Grid（分割节点）+ CSS Flex（面板）   |
| 拖拽调整 | Pointer Events + `requestAnimationFrame` |
| 持久化   | `localStorage`（带版本校验）             |
| 样式     | Tailwind CSS v4                          |
| 国际化   | i18next（`react-i18next`）               |

### 1.3 文件清单

| 文件路径                               | 行数 | 职责                                         |
| -------------------------------------- | ---- | -------------------------------------------- |
| `src/store/paneLayoutStore.ts`         | 436  | 窗格树状态管理：分割、关闭、全屏、交换、聚焦 |
| `src/store/paneControllerStore.ts`     | 101  | 窗格控制器注册与查询                         |
| `src/store/layoutStore.ts`             | 988  | 全局布局状态：侧边栏、面板、标签页、持久化   |
| `src/features/chat/SplitContainer.tsx` | 205  | 递归分割树渲染器                             |
| `src/features/chat/ChatPane.tsx`       | 727  | 单个窗格的聊天界面                           |
| `src/features/chat/PaneHeader.tsx`     | 306  | 窗格紧凑头部（分割/全屏/关闭/拖拽交换）      |
| `src/components/RightPanel.tsx`        | 254  | 右侧面板容器                                 |
| `src/components/BottomPanel.tsx`       | 317  | 底部面板容器（含终端恢复）                   |
| `src/components/PanelContainer.tsx`    | 569  | 统一面板容器（标签页管理、拖拽、右键菜单）   |
| `src/components/ui/ResizablePanel.tsx` | 302  | 可调整大小的面板通用组件                     |
| `src/hooks/useVerticalSplitResize.ts`  | 170  | 垂直分割调整大小的 Hook（当前未直接使用）    |
| `src/App.tsx`（布局编排部分）          | 592  | 四大布局区域的组合与路由同步                 |

---

## 2. Constitution Check

对照项目宪法，本模块的实现符合以下原则：

| 宪法原则                    | 符合情况 | 说明                                                                  |
| --------------------------- | -------- | --------------------------------------------------------------------- |
| 原则 2：OpenCode 兼容性优先 | ✅       | 窗格通过 `paneId` 与后端 Session API 关联，不引入额外后端依赖         |
| 原则 3：多平台统一代码库    | ✅       | Web 与 Tauri 共享同一套布局代码，通过 `isTauri()` 条件适配            |
| 原则 4：自定义优于框架依赖  | ✅       | 自定义 Store 模式（非 Redux/Zustand），自定义拖拽逻辑（非 react-dnd） |
| 原则 6：中文优先文档        | ✅       | 本文档以中文撰写，技术术语保留英文                                    |
| 原则 9：主题与可访问性      | ✅       | 响应式面板（overlay/push 模式）、Safe Area 适配、触屏长按拖拽         |
| 原则 10：模块化功能架构     | ✅       | Store 与组件分离，职责清晰，通过明确接口通信                          |
| 约束 C4：依赖最小化         | ✅       | 零额外布局依赖，纯 React + CSS 实现                                   |

---

## 3. Research Findings

### 3.1 实际代码与 Spec 的差异

| Spec 描述                             | 实际实现                                                                                         | 差异说明                                                 |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------- |
| 分隔条使用 `HIT_EXTEND = 4px`         | `HIT_EXTEND = 4`，`SPLIT_GAP = 6`，实际 hitSize = 14px                                           | 一致                                                     |
| 拖拽期间绕过框架渲染                  | `SplitContainer` 中直接修改 `gridTemplateColumns/Rows`                                           | 一致                                                     |
| `useVerticalSplitResize` Hook         | 文件存在但 `SplitContainer` 未使用，而是内联实现                                                 | Hook 为历史遗留，当前分割拖拽逻辑直接写在 `SplitNode` 内 |
| 全屏使用 `content-visibility: hidden` | 实际使用 `position: absolute + width/height: 0 + overflow: hidden + contentVisibility: 'hidden'` | 比 spec 更保守，同时使用多种隐藏手段                     |
| ratio 范围 0.15-0.85                  | Store 层 `setRatio` 限制 0.15-0.85，组件层 `MIN_RATIO=0.1, MAX_RATIO=0.9`                        | 组件层范围更宽，Store 层更严格                           |
| 面板响应式覆盖/推挤模式               | 通过 `ChatViewportProvider` 的 `interaction.*PanelBehavior` 决定                                 | 一致                                                     |

### 3.2 关键发现

1. **`useVerticalSplitResize` Hook 未被使用**：该 Hook 存在于 `src/hooks/` 中，但 `SplitContainer.tsx` 的拖拽逻辑是完全内联实现的。Hook 使用 CSS 变量方式，而内联实现直接修改 grid-template。

2. **SplitContainer 使用 CSS Grid 而非 Flex**：overall-plan.md 描述为"使用 CSS flex 实现分割效果"，实际代码使用 CSS Grid（`grid-template-columns` / `grid-template-rows`）。

3. **PaneController 的 `isSameController` 比较包含函数引用**：通过比较所有属性（包括方法引用）来决定是否触发通知，这要求方法引用保持稳定。

4. **LayoutStore 的 snapshot 缓存模式**：使用外部 `cachedSnapshot` 变量 + 订阅清除模式，而非 Store 内部缓存。

5. **双向路由同步使用 `syncingFromRouteRef` 防抖**：避免 URL 变化与 Store 变化之间的循环更新。

---

## 4. Data Model

### 4.1 窗格树数据结构

```typescript
// 叶子节点 — 代表一个实际窗格
interface PaneLeaf {
  type: 'leaf'
  id: string // 格式: "pane-N"，N 为自增计数器
  sessionId: string | null
}

// 分割节点 — 代表一次分割操作
interface PaneSplit {
  type: 'split'
  id: string // 格式: "split-{timestamp}-{random}"
  direction: 'horizontal' | 'vertical'
  ratio: number // 0-1，first 子节点占用比例
  first: PaneNode
  second: PaneNode
}

type PaneNode = PaneLeaf | PaneSplit
```

**ID 生成策略**：

- 叶子节点：`pane-${_nextPaneId++}`，全局自增计数器，`reset()` 时归 1
- 分割节点：`split-${Date.now()}-${Math.random().toString(36).slice(2, 6)}`

### 4.2 布局快照

```typescript
interface PaneLayoutSnapshot {
  root: PaneNode
  focusedPaneId: string | null
  focusedSessionId: string | null // 派生：focusedPane 的 sessionId
  fullscreenPaneId: string | null
  paneCount: number // 派生：countLeaves(root)
  isSplit: boolean // 派生：paneCount > 1
}
```

### 4.3 窗格控制器

```typescript
interface PaneControllerState {
  paneId: string
  sessionId: string | null
  effectiveDirectory: string
  contextLimit?: number
  isStreaming: boolean
  // 操作接口（稳定引用）
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

### 4.4 面板标签页

```typescript
type PanelPosition = 'bottom' | 'right'
type PanelTabType = 'terminal' | 'files' | 'changes' | 'mcp' | 'skill' | 'worktree'

interface PanelTab {
  id: string
  type: PanelTabType
  position: PanelPosition
  // files 类型特有
  previewFile?: PreviewFile | null
  previewFiles?: PreviewFile[]
  // terminal 类型特有
  ptyId?: string
  title?: string
  status?: 'connecting' | 'connected' | 'disconnected' | 'exited'
}

interface PreviewFile {
  path: string
  name: string
}
```

### 4.5 布局状态（LayoutState）

```typescript
interface LayoutState {
  // 面板标签系统
  panelTabs: PanelTab[]
  activeTabId: { bottom: string | null; right: string | null }

  // 侧边栏
  sidebarExpanded: boolean
  sidebarFolderRecents: boolean
  sidebarFolderRecentsShowDiff: boolean
  sidebarShowChildSessions: boolean

  // 右侧面板
  rightPanelOpen: boolean
  rightPanelWidth: number // 默认 450px，范围 160-1280

  // 底部面板
  bottomPanelOpen: boolean
  bottomPanelHeight: number // 默认 250px，范围 100-500
}
```

### 4.6 持久化格式

```typescript
// 面板布局持久化（version: 1）
interface PersistedPanelLayout {
  version: 1
  panelTabs: { id: string; type: Exclude<PanelTabType, 'terminal'>; position: PanelPosition; title?: string }[]
  activeTabId: { bottom: string | null; right: string | null }
  rightPanelOpen: boolean
  bottomPanelOpen: boolean
}

// 终端布局持久化（按目录，version: 1）
interface PersistedTerminalLayoutMap {
  version: 1
  directories: Record<
    string,
    {
      order: { bottom: string[]; right: string[] }
      activeTabId: { bottom: string | null; right: string | null }
    }
  >
}
```

**Storage Key 清单**：

| Key                                         | 存储内容           | 格式                              |
| ------------------------------------------- | ------------------ | --------------------------------- |
| `opencode-sidebar-expanded`                 | 侧边栏展开状态     | `"true"` / `"false"`              |
| `opencode-sidebar-folder-recents`           | 文件夹最近模式     | `"true"` / `"false"`              |
| `opencode-sidebar-folder-recents-show-diff` | 最近模式显示 Diff  | `"true"` / `"false"`              |
| `opencode-sidebar-show-child-sessions`      | 显示子会话         | `"true"` / `"false"`              |
| `opencode-panel-layout`                     | 面板标签布局       | JSON (PersistedPanelLayout)       |
| `opencode-terminal-layout`                  | 终端布局（按目录） | JSON (PersistedTerminalLayoutMap) |
| `opencode-right-panel-width`                | 右侧面板宽度       | 数字字符串                        |
| `opencode-bottom-panel-height`              | 底部面板高度       | 数字字符串                        |

---

## 5. Interface Contracts

### 5.1 paneLayoutStore 接口

```typescript
interface PaneLayoutStore {
  // useSyncExternalStore API
  subscribe(listener: () => void): () => void
  getSnapshot(): PaneLayoutSnapshot

  // 查询
  getRoot(): PaneNode
  getFocusedPaneId(): string | null
  getFocusedLeaf(): PaneLeaf | null
  getFocusedSessionId(): string | null
  getFullscreenPaneId(): string | null
  findLeaf(paneId: string): PaneLeaf | null
  allLeaves(): PaneLeaf[]
  isSinglePane(): boolean

  // 突变操作
  focusPane(paneId: string): void
  enterPaneFullscreen(paneId: string): void
  exitPaneFullscreen(): void
  togglePaneFullscreen(paneId: string): void
  setPaneSession(paneId: string, sessionId: string | null): void
  setFocusedSession(sessionId: string | null): void
  splitPane(paneId: string, direction: 'horizontal' | 'vertical', newSessionId?: string | null): string | null
  closePane(paneId: string): void
  swapPanes(paneIdA: string, paneIdB: string): void
  setRatio(splitId: string, ratio: number): void
  enterSplitMode(sessionId: string | null): string | null
  exitSplitMode(): void
  reset(): void
  focusNextPane(): void
  focusPrevPane(): void
  focusPaneByIndex(index: number): void
}
```

**导出 Hook**：

- `usePaneLayout(): PaneLayoutSnapshot` — 通过 `useSyncExternalStore` 订阅

### 5.2 paneControllerStore 接口

```typescript
class PaneControllerStore {
  subscribe(listener: () => void): () => void
  setController(paneId: string, controller: PaneControllerState): void
  removeController(paneId: string): void
  getController(paneId: string | null | undefined): PaneControllerState | null
  getControllers(): PaneControllerState[]
}
```

**导出 Hook**：

- `usePaneController(paneId): PaneControllerState | null`
- `usePaneControllers(): PaneControllerState[]`

### 5.3 LayoutStore 接口（核心方法）

```typescript
class LayoutStore {
  subscribe(fn: () => void): () => void

  // 侧边栏
  getSidebarExpanded(): boolean
  setSidebarExpanded(expanded: boolean): void
  toggleSidebar(): void

  // 面板标签
  getTabsForPosition(position: PanelPosition): PanelTab[]
  getActiveTab(position: PanelPosition): PanelTab | null
  setActiveTab(position: PanelPosition, tabId: string): void
  addTab(tab: Omit<PanelTab, 'id'> & { id?: string }, openPanel?: boolean): string
  removeTab(tabId: string): void
  updateTab(tabId: string, updates: Partial<Omit<PanelTab, 'id' | 'type'>>): void
  moveTab(tabId: string, toPosition: PanelPosition): void
  reorderTabs(position: PanelPosition, draggedId: string, targetId: string): void

  // 右侧面板（兼容 API）
  get rightPanelView(): RightPanelView
  toggleRightPanel(view?: RightPanelView): void
  openRightPanel(view: RightPanelView): void
  closeRightPanel(): void
  setRightPanelView(view: RightPanelView): void
  setRightPanelWidth(width: number): void

  // 底部面板（兼容 API）
  toggleBottomPanel(): void
  openBottomPanel(): void
  closeBottomPanel(): void
  setBottomPanelHeight(height: number): void

  // 终端
  setCurrentTerminalDirectory(directory?: string): void
  syncTerminalSessions(directory: string | undefined, sessions: TerminalTab[]): void
  addTerminalTab(tab: TerminalTab, openPanel?: boolean, position?: PanelPosition): void
  removeTerminalTab(id: string): void
  getTerminalTabs(position?: PanelPosition): TerminalTab[]

  // 文件预览
  openFilePreview(file: PreviewFile, position?: PanelPosition): void
  closeFilePreview(tabId: string, path?: string): void
  closeAllFilePreviews(tabId: string): void
  reorderFilePreviews(tabId: string, draggedPath: string, targetPath: string): void

  getState(): LayoutState
}
```

**导出 Hook**：

- `useLayoutStore(): LayoutSnapshot` — 包含派生属性 `rightPanelView`、`terminalTabs`、`activeTerminalId`

### 5.4 SplitContainer 组件接口

```typescript
interface SplitContainerProps {
  node: PaneNode
  renderLeaf: (paneId: string, sessionId: string | null) => React.ReactNode
  fullscreenPaneId?: string | null
}
```

### 5.5 ChatPane 组件接口

```typescript
interface ChatPaneProps {
  paneId: string
  sessionId: string | null
  isFocused: boolean
  paneCount: number
  displayMode: 'single' | 'split'
  isPaneFullscreen?: boolean
  onOpenSidebar?: () => void
  showSidebarButton?: boolean
  onSplitPane?: () => void
  onTogglePaneFullscreen?: () => void
  navigatePaneToSession: (paneId: string, sessionId: string, directory?: string) => void
  navigatePaneHome: (paneId: string) => void
}
```

### 5.6 PaneHeader 组件接口

```typescript
interface PaneHeaderProps {
  paneId: string
  sessionId: string | null
  isFocused: boolean
  paneCount: number
  canSplitPane?: boolean
  isPaneFullscreen?: boolean
  showSidebarButton?: boolean
  onOpenSidebar?: () => void
  onTogglePaneFullscreen?: () => void
  onFocus: () => void
}
```

### 5.7 ResizablePanel 组件接口

```typescript
interface ResizablePanelProps {
  position: 'right' | 'bottom'
  isOpen: boolean
  overlay?: boolean
  size: number
  minSize?: number
  maxSize?: number
  onSizeChange: (size: number) => void
  onClose: () => void
  children: React.ReactNode
  className?: string
}
```

### 5.8 PanelContainer 组件接口

```typescript
interface PanelContainerProps {
  position: PanelPosition
  children: (activeTab: PanelTab | null) => React.ReactNode
  onNewTerminal?: () => void
  onCloseTerminal?: (ptyId: string) => void
}
```

---

## 6. Implementation Strategy

### 6.1 架构分层

```
┌─────────────────────────────────────────────────────────┐
│                    App.tsx (编排层)                       │
│  Sidebar | SplitContainer | BottomPanel | RightPanel     │
│  路由同步 | 快捷键绑定 | 命令面板                         │
├─────────────────────────────────────────────────────────┤
│                    Store 层                              │
│  paneLayoutStore | paneControllerStore | layoutStore     │
│  useSyncExternalStore 桥接                               │
├─────────────────────────────────────────────────────────┤
│                    组件层                                │
│  SplitContainer → ChatPane → PaneHeader                 │
│  ResizablePanel → PanelContainer → 内容组件              │
├─────────────────────────────────────────────────────────┤
│                    工具层                                │
│  useVerticalSplitResize (未使用)                         │
│  chatViewport (响应式视口计算)                           │
└─────────────────────────────────────────────────────────┘
```

### 6.2 窗格树渲染流程

```
App.tsx
  │
  ├─ usePaneLayout() → PaneLayoutSnapshot
  │
  ├─ renderPaneLeaf(paneId, sessionId) → <ChatPane ... />
  │
  └─ <SplitContainer
       node={paneLayout.root}
       renderLeaf={renderPaneLeaf}
       fullscreenPaneId={paneLayout.fullscreenPaneId}
     />
       │
       ├─ node.type === 'leaf' → renderLeaf(node.id, node.sessionId)
       │
       └─ node.type === 'split' → <SplitNode split={node} ... />
            │
            ├─ <div className="grid"> (CSS Grid 容器)
            │    ├─ <SplitContainer node={split.first} />  (第一个子节点)
            │    ├─ <div onPointerDown={handleDrag} />     (可拖拽分隔条)
            │    └─ <SplitContainer node={split.second} /> (第二个子节点)
            │
            └─ 全屏模式：absolute 定位 + content-visibility: hidden
```

### 6.3 拖拽调整实现细节

**SplitContainer 内联拖拽**（当前使用）：

1. `onPointerDown` 触发 `handleDrag`
2. 设置 `document.body.style.cursor` 和 `userSelect`
3. 注册全局 `pointermove` / `pointerup` 监听器
4. `pointermove` 中计算比例，通过 `requestAnimationFrame` 节流
5. 直接修改容器的 `gridTemplateColumns` 或 `gridTemplateRows`
6. `pointerup` 中清除内联样式，调用 `paneLayoutStore.setRatio()` 提交
7. 触发 `panel-resize-start` / `panel-resize-end` 自定义事件

**ResizablePanel 拖拽**（面板尺寸调整）：

1. Pointer Events 主路径 + Touch Events 降级
2. 拖拽期间直接修改 `panel.style.width/height`
3. `requestAnimationFrame` 节流
4. 松开后调用 `onSizeChange` 回调提交到 Store
5. 同样触发 `panel-resize-start` / `panel-resize-end` 事件

**关键参数**：

| 参数          | SplitContainer         | ResizablePanel (right) | ResizablePanel (bottom) |
| ------------- | ---------------------- | ---------------------- | ----------------------- |
| 命中区域      | 14px (6 + 4\*2)        | 触屏 16px / 鼠标 4px   | 触屏 16px / 鼠标 4px    |
| 最小比例/尺寸 | 0.1 / 0.9              | 160px / 1280px         | 100px / 500px           |
| 节流方式      | rAF                    | rAF                    | rAF                     |
| 事件通知      | panel-resize-start/end | panel-resize-start/end | panel-resize-start/end  |

### 6.4 全屏模式实现

```
正常模式:
┌─────────────────────────────────┐
│  grid container                 │
│  ┌──────┬───────────┐           │
│  │Pane A│  Pane B   │           │
│  └──────┴───────────┘           │
└─────────────────────────────────┘

全屏模式 (fullscreenPaneId = "pane-B"):
┌─────────────────────────────────┐
│  relative container             │
│  ┌───────────────────────────┐  │
│  │ Pane B (absolute, inset:0)│  │
│  │ z-index: 1                │  │
│  └───────────────────────────┘  │
│  Pane A (width:0, height:0,    │
│          overflow:hidden,       │
│          contentVisibility:hidden)│
└─────────────────────────────────┘
```

**关键实现**：

- 全屏窗格所在分支使用 `position: absolute; inset: 0; z-index: 1`
- 非全屏分支使用 `position: absolute; width: 0; height: 0; overflow: hidden; contentVisibility: 'hidden'`
- 分割操作时自动退出全屏（`splitPane` 中检查 `_fullscreenPaneId`）
- 关闭全屏窗格时自动退出全屏（`closePane` 中检查）

### 6.5 路由同步机制

```
URL Hash                          paneLayoutStore
#/session/abc?dir=/path              │
       │                             │
       │  useEffect(routeSessionId)  │
       ├─────────────────────────────▶│ setFocusedSession(abc)
       │   syncingFromRouteRef=true   │
       │                             │
       │                             │ focusPane 变化
       │  useEffect(focusedSessionId)│
       │◀─────────────────────────────┤
       │   if syncingFromRouteRef     │
       │     → skip (防循环)          │
       │   else → replaceSession()    │
```

**防循环机制**：`syncingFromRouteRef` 标记在 URL→Store 同步时设为 `true`，Store→URL 的 effect 检测到此标记时跳过。

### 6.6 PaneController 注册与清理

```
ChatPane mount
  │
  ├─ useEffect(() => {
  │    paneControllerStore.setController(paneId, {
  │      paneId, sessionId, effectiveDirectory,
  │      contextLimit, isStreaming,
  │      newSession: stableRef, ...
  │    })
  │  }, [deps])
  │
  └─ useEffect(() => {
       return () => paneControllerStore.removeController(paneId)
     }, [paneId])
```

**稳定性策略**：

- 操作函数通过 `useMemo` 包装为稳定引用（`stableControllerActions`）
- 实际执行通过 `useRef` 桥接最新实现（`controllerActionsRef`）
- `isSameController` 比较所有属性（包括函数引用）决定是否通知

### 6.7 终端会话恢复流程

```
BottomPanel mount / directory change
  │
  ├─ requestId = ++restoreRequestIdRef
  │
  ├─ listPtySessions(directory)
  │
  ├─ if requestId changed → abort (竞态保护)
  │
  ├─ layoutStore.syncTerminalSessions(directory, sessions)
  │    │
  │    ├─ 读取该目录的持久化终端布局
  │    ├─ 合并非终端标签和终端标签
  │    ├─ 按持久化顺序排列
  │    └─ 更新 panelTabs 和 activeTabId
  │
  └─ setIsRestoring(false)
```

### 6.8 面板响应式行为

通过 `ChatViewportProvider` 的 `interaction` 上下文决定：

| 行为                  | 值                     | 效果                     |
| --------------------- | ---------------------- | ------------------------ |
| `sidebarBehavior`     | `'push'` / `'overlay'` | 侧边栏推挤内容或覆盖内容 |
| `rightPanelBehavior`  | `'push'` / `'overlay'` | 右侧面板推挤或覆盖       |
| `bottomPanelBehavior` | `'push'` / `'overlay'` | 底部面板推挤或覆盖       |

`ResizablePanel` 根据 `overlay` prop 切换渲染模式：

- **Overlay 模式**：`fixed` 定位 + CSS `translate` 动画 + 遮罩层
- **Push 模式**：`relative` 定位 + CSS `width/height` 过渡动画

---

## 7. Error Handling

### 7.1 持久化异常处理

所有 `localStorage` 操作均包裹在 `try/catch` 中，异常时静默失败：

```typescript
try {
  localStorage.setItem(key, JSON.stringify(data))
} catch {
  // ignore
}
```

### 7.2 持久化数据校验

加载时进行严格校验，不合法数据使用默认值：

- `sanitizePersistedPanelLayout()`：检查 version、类型、ID 唯一性
- `sanitizePersistedTerminalLayoutMap()`：检查 version、目录结构、数组格式
- 类型守卫：`isPanelPosition()`、`isPersistedPanelTabType()`

### 7.3 竞态保护

终端会话恢复使用 `requestId` 机制：

```typescript
const requestId = ++restoreRequestIdRef.current
const sessions = await listPtySessions(directory)
if (restoreRequestIdRef.current !== requestId) return // 过期请求，丢弃
```

### 7.4 树操作防御

- `findLeaf` 找不到返回 `null`，调用方检查
- `splitPane` 找不到目标叶子返回 `null`
- `closePane` 在单窗格模式下仅清除 session 而非删除节点
- `swapPanes` 检查 `paneIdA === paneIdB` 时直接返回
- `enterPaneFullscreen` 检查叶子存在性

### 7.5 自定义事件通信

通过 `window.dispatchEvent(new CustomEvent('panel-resize-start/end'))` 通知其他组件：

- `RightPanel` 和 `BottomPanel` 监听这些事件，在拖拽期间设置 `isPanelResizing` 状态
- 终端等组件可根据此状态暂停不必要的重渲染

---

## 8. Testing Considerations

### 8.1 单元测试覆盖点

**paneLayoutStore**：

- 初始状态：单叶子节点，focusedPaneId 正确
- `splitPane`：创建分割节点，新叶子 ID 正确，ratio = 0.5
- `closePane`：兄弟节点提升，单窗格时清除 session
- `swapPanes`：sessionId 交换，树结构不变
- `setRatio`：比例限制在 0.15-0.85
- `focusNextPane` / `focusPrevPane`：循环导航
- `enterSplitMode` / `exitSplitMode`：模式切换
- `enterPaneFullscreen` / `exitPaneFullscreen`：全屏状态
- 不可变性：树操作返回新对象，不修改原树
- 订阅通知：每次突变触发 listener

**paneControllerStore**：

- `setController` / `removeController` / `getController`
- `isSameController` 比较逻辑
- 相同控制器不触发通知

**layoutStore**：

- `addTab` / `removeTab` / `moveTab` / `reorderTabs`
- 移除最后一个 tab 时自动关闭面板
- `syncTerminalSessions` 合并逻辑
- 持久化读写与校验
- 兼容 API（`rightPanelView`、`terminalTabs`）

### 8.2 组件测试覆盖点

**SplitContainer**：

- 单叶子渲染
- 水平/垂直分割渲染
- 嵌套分割渲染
- 全屏模式：全屏窗格可见，其他隐藏
- 拖拽分隔条交互

**PaneHeader**：

- 分割按钮触发
- 全屏按钮切换
- 关闭按钮触发
- 拖拽交换：dataTransfer 传递 paneId
- 会话标题编辑

**PanelContainer**：

- 标签切换
- 拖拽排序
- 右键菜单移动到另一面板
- 加号菜单新建标签
- 触屏长按拖拽

**ResizablePanel**：

- 打开/关闭动画
- 拖拽调整尺寸
- Overlay 模式 vs Push 模式
- 触屏拖拽

### 8.3 集成测试场景

1. **分割 + 路由同步**：分割窗格后切换会话，URL 正确反映 focused session
2. **全屏 + 分割**：全屏状态下执行分割，自动退出全屏
3. **终端恢复**：刷新页面后终端标签顺序恢复
4. **面板标签移动**：终端从底部移动到右侧，内容保持不变
5. **窗格交换**：拖拽交换后两个窗格的 session 内容互换

### 8.4 性能测试关注点

1. **拖拽流畅度**：分隔条拖拽期间保持 60fps，无卡顿
2. **多窗格渲染**：4+ 窗格同时渲染时的性能
3. **全屏切换**：进入/退出全屏时的渲染开销
4. **终端恢复**：大量 PTY 会话时的恢复速度
5. **订阅粒度**：`useSyncExternalStore` 是否导致不必要的重渲染

---

## 9. 风险区域与已知问题

### 9.1 中高风险

| 风险                                   | 影响                               | 现状                                              |
| -------------------------------------- | ---------------------------------- | ------------------------------------------------- |
| `useVerticalSplitResize` Hook 未被使用 | 代码冗余，维护困惑                 | Hook 存在但 `SplitContainer` 使用内联实现         |
| ratio 范围不一致                       | Store 层 0.15-0.85，组件层 0.1-0.9 | 拖拽期间可能设置超出 Store 限制的值，提交时被修正 |
| `isSameController` 函数引用比较        | 方法引用变化导致不必要通知         | 通过 `useMemo` + `useRef` 桥接保持稳定            |
| 深层嵌套分割的递归渲染                 | 树深度增加时渲染开销线性增长       | 实际使用中分割层级通常不超过 3 层                 |

### 9.2 低风险

| 风险                            | 影响                         | 现状                          |
| ------------------------------- | ---------------------------- | ----------------------------- |
| `localStorage` 容量限制         | 大量终端布局数据可能超出限制 | 按目录存储，单目录数据量小    |
| 全局自增 `_nextPaneId` 不持久化 | 刷新后 ID 从 1 重新开始      | 不影响功能，ID 仅用于渲染 key |
| `cachedSnapshot` 外部缓存       | 需要手动清除，可能遗漏       | 通过订阅回调清除，逻辑清晰    |

---

## 10. 依赖关系

### 10.1 模块依赖

```
012-pane-layout
  ├── 010-state-management (useSyncExternalStore 模式)
  ├── 013-i18n-localization (react-i18next)
  ├── 002-chat-feature (ChatViewportProvider, canUseSplitPane)
  ├── 004-session-management (会话数据)
  ├── 008-terminal-system (PTY API, Terminal 组件)
  ├── 011-file-diff-viewer (SessionChangesPanel)
  └── 001-api-layer (updateSession, PTY API)
```

### 10.2 被依赖

```
被 002-chat-feature 依赖：
  - App.tsx 使用 SplitContainer 作为聊天区域布局
  - ChatPane 作为叶子渲染器
  - 路由同步逻辑

被 005-settings-panel 依赖：
  - KeybindingsSection 使用 paneLayoutStore 的快捷键

被快捷键系统依赖：
  - focusNextPane, focusPrevPane, splitRight, splitDown
  - closePane, togglePaneFullscreen
```

---

## 11. 架构决策记录（As-Built）

### ADR-012-001：CSS Grid 而非 Flex 用于分割布局

**决策**：`SplitContainer` 使用 CSS Grid（`grid-template-columns` / `grid-template-rows`）实现分割，而非 Flexbox。

**理由**：

- Grid 的 `fr` 单位天然支持比例分配，无需计算像素值
- 拖拽期间直接修改 `gridTemplate` 字符串即可更新布局
- 分隔条作为独立的 grid item，天然占据间隙位置
- 比 Flexbox 的 `flex-grow` 更精确地控制比例

### ADR-012-002：内联拖拽而非 Hook 封装

**决策**：`SplitContainer` 的拖拽逻辑直接内联在 `SplitNode` 组件中，而非使用 `useVerticalSplitResize` Hook。

**理由**：

- 内联实现更直接地访问 `containerRef` 和 `split.ratio`
- 避免 Hook 的 CSS 变量方式与 Grid 布局的适配复杂度
- 拖拽逻辑与渲染逻辑紧密耦合，分离收益有限

### ADR-012-003：Snapshot 外部缓存模式

**决策**：`layoutStore` 的 snapshot 使用模块级 `cachedSnapshot` 变量 + 订阅清除模式，而非 Store 内部缓存。

**理由**：

- `getSnapshot()` 需要组合 `getState()` 和派生属性（`rightPanelView`、`terminalTabs`）
- 外部缓存避免每次 `getSnapshot` 调用都重新计算
- 订阅回调负责清除缓存，确保数据一致性

### ADR-012-004：PaneController 函数引用稳定性

**决策**：通过 `useRef` + `useMemo` 双层桥接保持 `PaneControllerState` 中方法引用的稳定性。

**理由**：

- `isSameController` 比较所有属性包括函数引用
- 方法实现频繁变化（依赖更新），但引用需保持稳定
- `useRef` 存储最新实现，`useMemo` 返回稳定包装函数

---

_本文档基于 OpenCodeUI v0.4.8 实际代码生成，所有技术细节均来自源码审查。_
