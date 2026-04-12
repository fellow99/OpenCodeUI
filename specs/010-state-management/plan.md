# 010-state-management — 技术方案（As-Built）

> 模块编号：010-state-management
> 状态：已实现
> 最后更新：2026-04-12

---

## 1. Technical Context

### 1.1 模块定位

状态管理模块是 OpenCodeUI 的核心数据枢纽。16 个职责单一的 Store 实例按领域拆分应用状态，每个 Store 独立维护自身数据、持久化策略和变更通知机制。模块位于 `src/store/` 目录，共 22 个文件（16 个 Store + 3 个测试 + 3 个辅助文件）。

### 1.2 技术栈

| 项目       | 选型                                          |
| ---------- | --------------------------------------------- |
| 框架       | React 19 + TypeScript                         |
| 状态模式   | 自定义发布.订阅（非 Redux/Zustand/Jotai）     |
| React 桥接 | `useSyncExternalStore`（React 18+ 原生 Hook） |
| 持久化     | localStorage / sessionStorage / IndexedDB     |
| 批量通知   | `requestAnimationFrame`（仅 messageStore）    |
| 测试       | Vitest                                        |

### 1.3 文件清单

| 文件                         | 行数 | 职责                                     |
| ---------------------------- | ---- | ---------------------------------------- |
| `index.ts`                   | 70   | Barrel 文件，统一导出所有 Store 及其类型 |
| `messageStore.ts`            | 696  | 消息与会话状态（最大最复杂）             |
| `messageStoreTypes.ts`       | 80   | 消息 Store 类型定义                      |
| `messageStoreHooks.ts`       | 211  | messageStore 的 React Hooks 绑定层       |
| `messageStore.test.ts`       | 137  | messageStore 单元测试                    |
| `messageStoreHooks.test.tsx` | —    | Hooks 组件测试                           |
| `paneLayoutStore.ts`         | 436  | 分屏布局二叉树管理                       |
| `layoutStore.ts`             | 988  | 全局 UI 布局（侧边栏、面板、终端标签）   |
| `serverStore.ts`             | 429  | 多服务器配置与健康检查                   |
| `keybindingStore.ts`         | 706  | 22 个动作的快捷键配置                    |
| `themeStore.ts`              | 586  | 主题预设、明暗模式、自定义 CSS           |
| `notificationStore.ts`       | 309  | Toast + 通知历史                         |
| `activeSessionStore.ts`      | 314  | 会话忙/闲状态追踪                        |
| `childSessionStore.ts`       | 279  | 父子会话关系                             |
| `autoApproveStore.ts`        | 284  | 自动批准规则 + Full Auto 模式            |
| `soundStore.ts`              | 397  | 提示音配置 + IndexedDB 音频存储          |
| `todoStore.ts`               | 172  | 按 session 管理的待办列表                |
| `serviceStore.ts`            | 181  | Tauri 桌面端服务进程管理                 |
| `paneControllerStore.ts`     | 101  | 每个 pane 的操作控制器注册               |
| `followupQueueStore.ts`      | 220  | 忙碌会话的消息排队                       |
| `changeScopeStore.ts`        | 46   | 变更范围模式（最小 Store）               |
| `layoutStore.test.ts`        | 88   | layoutStore 单元测试                     |

---

## 2. Constitution Check

### 2.1 原则对齐

| 宪法原则                   | 对齐情况    | 说明                                                                 |
| -------------------------- | ----------- | -------------------------------------------------------------------- |
| 原则 4：自定义优于框架依赖 | ✅ 完全对齐 | 采用自定义发布.订阅模式，零第三方状态库依赖                          |
| 原则 5：实时通信优先       | ✅ 完全对齐 | SSE 事件直接驱动 Store 更新，无轮询                                  |
| 原则 3：多平台统一代码库   | ✅ 完全对齐 | 所有 Store 在 Web 和 Tauri 环境共享，仅 serviceStore 在 Tauri 下生效 |
| 原则 10：模块化功能架构    | ✅ 完全对齐 | 16 个 Store 职责单一，通过 `index.ts` 统一导出                       |

### 2.2 约束检查

| 约束          | 状态    | 说明                                                            |
| ------------- | ------- | --------------------------------------------------------------- |
| C4 依赖最小化 | ✅ 通过 | 无额外状态管理依赖，仅使用 React 原生 API                       |
| C5 SSE 支持   | ✅ 通过 | Store 层不破坏 SSE，事件通过 `useGlobalEvents` 分发到对应 Store |

---

## 3. Research Findings

### 3.1 统一的 Store 模式

所有 16 个 Store 遵循同一套发布.订阅接口范式：

```typescript
// 核心接口（所有 Store 均实现）
subscribe(listener: () => void): () => void   // 注册监听，返回取消订阅函数
getSnapshot(): State                          // 返回当前状态快照（部分 Store 省略此方法名但功能等价）
private notify(): void                        // 内部方法，遍历所有监听器并调用
```

**实现变体**：

- **Class 模式**：`messageStore`、`serverStore`、`layoutStore`、`themeStore`、`autoApproveStore`、`soundStore`、`notificationStore`、`activeSessionStore`、`childSessionStore`、`todoStore`、`serviceStore`、`followupQueueStore`、`changeScopeStore`、`keybindingStore` 均采用 `class` + 单例导出
- **Factory 模式**：`paneLayoutStore` 使用 `createPaneLayoutStore()` 工厂函数返回对象
- **直接实例**：`paneControllerStore` 使用 `class` + 单例导出

### 3.2 React 绑定策略

所有 Store 通过 `useSyncExternalStore` 桥接到 React 渲染系统，存在三种绑定模式：

**模式 A：模块级缓存快照**（messageStore、layoutStore）

```typescript
// messageStoreHooks.ts
let cachedSnapshot: MessageStoreSnapshot | null = null
messageStore.subscribe(() => {
  cachedSnapshot = null
}) // 变更时清除
function getSnapshot() {
  if (cachedSnapshot === null) cachedSnapshot = createSnapshot()
  return cachedSnapshot
}
```

**模式 B：Store 内部缓存快照**（serverStore、paneLayoutStore、activeSessionStore、serviceStore）

```typescript
// serverStore.ts
private _cachedSnapshot: State = initialState
private notify() {
  this.updateSnapshots()   // 更新内部缓存
  this.listeners.forEach(l => l())
}
getSnapshot() { return this._cachedSnapshot }
```

**模式 C：直接调用**（todoStore、changeScopeStore、childSessionStore）

```typescript
// todoStore.ts — 使用 version 计数器
useSyncExternalStore(
  useCallback(callback => todoStore.subscribe(callback), []),
  useCallback(() => todoStore.getVersion(), []),
)
```

### 3.3 持久化分层

| 存储层         | 用途                                      | 使用 Store                                                                                                                                                         |
| -------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| localStorage   | 用户偏好、配置、布局                      | layoutStore, keybindingStore, themeStore, notificationStore, serviceStore, autoApproveStore(开关), soundStore(配置), serverStore(服务器列表)                       |
| sessionStorage | 窗口级隔离的活动服务器 ID                 | serverStore                                                                                                                                                        |
| IndexedDB      | 大二进制数据（自定义音频，上限 2MB/文件） | soundStore                                                                                                                                                         |
| 内存           | 运行时状态，刷新即失                      | messageStore, activeSessionStore, childSessionStore, paneLayoutStore, paneControllerStore, todoStore, followupQueueStore, changeScopeStore, autoApproveStore(规则) |

**localStorage 写入策略**：所有写入操作包裹 `try-catch`，静默忽略 `QuotaExceededError` 等异常。

### 3.4 服务器切换时的状态清理

`serverStore.onServerChange` 回调在 `main.tsx` 中注册，触发以下清理链：

```
serverStore.setActiveServer()
  → serverChangeListeners.forEach(fn => fn(id))
    → invalidateSDKClient()        // API 层重建
    → messageStore.clearAll()      // 清空所有会话消息
    → childSessionStore.clearAll() // 清空子会话
    → todoStore.clearAll()         // 清空待办
    → resetPathModeCache()         // 重置路径模式缓存
    → autoApproveStore.reloadFromStorage()  // 重载开关，清空规则，关闭 Full Auto
    → reconnectSSE()               // 重连 SSE
```

注意：`notificationStore`、`changeScopeStore`、`followupQueueStore` 在当前实现中**未**在服务器切换时自动清理。

---

## 4. Data Model

### 4.1 核心类型

**SessionState**（messageStoreTypes.ts）

```typescript
interface SessionState {
  messages: Message[] // 所有消息（包括被撤销的）
  revertState: RevertState | null // 撤销状态
  isStreaming: boolean // 是否正在流式输出
  loadState: 'idle' | 'loading' | 'loaded' | 'error'
  hasMoreHistory: boolean // 是否还有更多历史
  directory: string // 会话工作目录
  title?: string // 会话标题
  shareUrl?: string // 分享链接
  isStale: boolean // 断线后是否需要重新全量拉取
}
```

**RevertState**（撤销.重做）

```typescript
interface RevertState {
  messageId: string // 撤销点的消息 ID
  history: RevertHistoryItem[] // 历史栈，用于多步 redo
}

interface RevertHistoryItem {
  messageId: string
  text: string
  attachments: unknown[]
  model?: { providerID: string; modelID: string; variant?: string }
  variant?: string
  agent?: string
}
```

**PaneNode**（分屏布局二叉树）

```typescript
interface PaneLeaf {
  type: 'leaf'
  id: string
  sessionId: string | null
}

interface PaneSplit {
  type: 'split'
  id: string
  direction: 'horizontal' | 'vertical'
  ratio: number // 0-1，first 子节点占比
  first: PaneNode
  second: PaneNode
}

type PaneNode = PaneLeaf | PaneSplit
```

**ActiveSessionEntry**

```typescript
interface ActiveSessionEntry {
  sessionId: string
  status: SessionStatus // { type: 'busy' | 'idle' | 'retry' }
  title?: string
  directory?: string
  pendingAction?: {
    type: 'permission' | 'question'
    description?: string
  }
}
```

### 4.2 内存淘汰策略

`messageStore` 维护最多 10 个会话的内存缓存（`MAX_CACHED_SESSIONS = 10`）：

- 每次访问会话时通过 `sessionAccessTime` Map 记录时间戳
- 新增会话时若超过上限，调用 `evictOldSessions()` 淘汰最久未访问的会话
- 跳过条件：被 `protectedSessions` Set 保护的会话（分屏 pane 使用）、正在流式输出的会话
- 保护/取消保护通过 `protectSession()` / `unprotectSession()` 方法控制

### 4.3 派生数据缓存

多个 Store 使用派生数据缓存避免重复计算：

- **activeSessionStore**：`cachedBusySessions` 和 `cachedBusyCount` 在 `recomputeDerived()` 中重建，每次 `notify()` 前调用
- **todoStore**：`statsCache` 在 `setTodos()` 时更新，`getStats()` 直接返回
- **serverStore**：`_serversSnapshot`、`_activeServerSnapshot`、`_healthMapSnapshot` 在 `updateSnapshots()` 中更新

---

## 5. Interface Contracts

### 5.1 Store 公共接口约定

每个 Store 至少提供以下能力：

| 能力 | 方法签名                                      | 必需性                            |
| ---- | --------------------------------------------- | --------------------------------- |
| 订阅 | `subscribe(listener: () => void): () => void` | 必需                              |
| 快照 | `getSnapshot(): State` 或等价 getter          | 必需（用于 useSyncExternalStore） |
| 通知 | `private notify(): void`                      | 内部必需                          |

### 5.2 messageStore 专用接口

| 类别 | 方法                                                        | 说明                             |
| ---- | ----------------------------------------------------------- | -------------------------------- |
| 订阅 | `subscribe(fn: Subscriber): () => void`                     | 标准发布.订阅                    |
| 查询 | `getSessionState(sessionId)`                                | 获取会话原始状态                 |
| 查询 | `getVisibleMessages(sessionId)`                             | 获取可见消息（考虑撤销）         |
| 查询 | `getIsStreaming(sessionId)`                                 | 是否流式输出中                   |
| 查询 | `canUndo(sessionId)` / `canRedo(sessionId)`                 | 撤销.重做可用性                  |
| SSE  | `handleMessageUpdated(apiMsg)`                              | 处理 message.updated 事件        |
| SSE  | `handlePartUpdated(apiPart)`                                | 处理 part.updated 事件           |
| SSE  | `handlePartDelta(data)`                                     | 处理 part.delta 事件（RAF 批量） |
| SSE  | `handlePartRemoved(data)`                                   | 处理 part.removed 事件           |
| SSE  | `handleSessionIdle(sessionId)`                              | 处理 session.idle 事件           |
| SSE  | `handleSessionError(sessionId)`                             | 处理 session.error 事件          |
| CRUD | `setMessages(sessionId, apiMessages, options)`              | 全量设置消息                     |
| CRUD | `prependMessages(sessionId, apiMessages, hasMore)`          | 前置历史消息                     |
| CRUD | `upsertLocalMessage(message)`                               | 插入/更新本地消息                |
| CRUD | `removeMessage(sessionId, messageId)`                       | 删除消息                         |
| CRUD | `updateSessionMetadata(sessionId, options)`                 | 更新元数据                       |
| 撤销 | `truncateAfterRevert(sessionId)`                            | 截断撤销点后的消息               |
| 撤销 | `createSendRollbackSnapshot(sessionId)`                     | 创建发送回滚快照                 |
| 撤销 | `restoreSendRollback(sessionId, snapshot)`                  | 恢复发送回滚快照                 |
| 淘汰 | `protectSession(sessionId)` / `unprotectSession(sessionId)` | 保护/取消保护                    |
| 清理 | `clearAll()` / `clearSession(sessionId)`                    | 清空全部/单个会话                |

### 5.3 React Hooks 接口

**messageStoreHooks.ts 导出的 Hooks**：

| Hook                                             | 返回值                            | 订阅范围                     |
| ------------------------------------------------ | --------------------------------- | ---------------------------- |
| `useMessageStore()`                              | `MessageStoreSnapshot`            | 当前聚焦 pane 的完整会话快照 |
| `useMessageStoreSelector(selector, equalityFn?)` | `T`                               | 选择器模式，自定义订阅字段   |
| `useSessionState(sessionId)`                     | `SessionStateSnapshot \| null`    | 指定 sessionId 的状态        |
| `useCurrentSessionId()`                          | `string \| null`                  | 仅订阅 sessionId             |
| `useIsStreaming()`                               | `boolean`                         | 仅订阅 isStreaming           |
| `useMessages()`                                  | `Message[]`                       | 仅订阅 messages              |
| `useUndoRedoState()`                             | `{ canUndo, canRedo, redoSteps }` | 仅订阅撤销.重做状态          |

**其他 Store 导出的 Hooks**：

| Store               | Hook                                         | 返回值                            |
| ------------------- | -------------------------------------------- | --------------------------------- |
| layoutStore         | `useLayoutStore()`                           | `LayoutSnapshot`                  |
| paneLayoutStore     | `usePaneLayout()`                            | `PaneLayoutSnapshot`              |
| paneControllerStore | `usePaneController(paneId)`                  | `PaneControllerState \| null`     |
| paneControllerStore | `usePaneControllers()`                       | `PaneControllerState[]`           |
| serverStore         | （无专用 Hook，直接 `useSyncExternalStore`） | —                                 |
| keybindingStore     | （无专用 Hook）                              | —                                 |
| themeStore          | （无专用 Hook，直接 `useSyncExternalStore`） | —                                 |
| todoStore           | `useTodos(sessionId)`                        | `TodoItem[]`                      |
| todoStore           | `useTodoStats(sessionId)`                    | `TodoStats`                       |
| todoStore           | `useCurrentTask(sessionId)`                  | `TodoItem \| null`                |
| notificationStore   | `useNotificationStore()`                     | `NotificationState`               |
| notificationStore   | `useNotifications()`                         | `NotificationEntry[]`             |
| notificationStore   | `useUnreadNotificationCount()`               | `number`                          |
| activeSessionStore  | `useActiveSessionStore()`                    | `ActiveSessionState`              |
| activeSessionStore  | `useBusySessions()`                          | `ActiveSessionEntry[]`            |
| activeSessionStore  | `useBusyCount()`                             | `number`                          |
| activeSessionStore  | `useSessionActiveEntry(sessionId)`           | `ActiveSessionEntry \| undefined` |
| childSessionStore   | `useChildSessions(parentId)`                 | `ChildSessionInfo[]`              |
| childSessionStore   | `useSessionFamily(sessionId)`                | `string[]`                        |
| serviceStore        | `useServiceStore()`                          | `ServiceStoreSnapshot`            |
| soundStore          | `useSoundSettings()`                         | `SoundSettings`                   |
| followupQueueStore  | `useFollowupQueue(sessionId)`                | `{ items, failedId, sendingId }`  |
| changeScopeStore    | `useSessionChangeScope(sessionId)`           | `ChangeScopeMode`                 |

### 5.4 选择器模式与浅比较

`useMessageStoreSelector` 实现了选择器模式，支持自定义 `equalityFn`（默认 `shallowEqual`）：

```typescript
function shallowEqual<T>(a: T, b: T): boolean {
  if (a === b) return true
  if (typeof a !== 'object' || typeof b !== 'object') return false
  if (a === null || b === null) return false
  const keysA = Object.keys(a as object)
  const keysB = Object.keys(b as object)
  if (keysA.length !== keysB.length) return false
  for (const key of keysA) {
    if (recordA[key] !== recordB[key]) return false
  }
  return true
}
```

---

## 6. Implementation Strategy

### 6.1 RAF 批量通知机制

`messageStore` 针对高频 SSE 事件（`message.part.delta`）实现了 RAF 批量通知：

```
SSE delta #1 → mutable 修改 part.text → dirtyMessages.add(messageID) → notify()
SSE delta #2 → mutable 修改 part.text → dirtyMessages.add(messageID) → notify() 跳过（pendingNotify=true）
SSE delta #3 → mutable 修改 part.text → dirtyMessages.add(messageID) → notify() 跳过
      ↓
RAF 回调触发 → flushDirtyMessages() → 仅对 dirty 消息生成不可变快照 → 通知所有订阅者
```

关键实现细节：

- `pendingNotify` 标志位防止同一帧内重复请求 RAF
- `dirtyMessages` Set 追踪被 mutable 修改过的 messageID
- `dirtySessionId` 记录被修改的 sessionId
- `flushDirtyMessages()` 仅对 dirty 消息执行 `{ ...m, parts: m.parts.map(p => ({ ...p })) }` 不可变拷贝
- 非浏览器环境（SSR/Tauri 初始化阶段）降级为 `notifyImmediate()`，直接通知

### 6.2 快照缓存防无限循环

`useSyncExternalStore` 要求 `getSnapshot` 在状态未变时返回同一引用。各 Store 采用不同策略：

- **messageStore**：模块级 `cachedSnapshot`，订阅回调中清除，`getSnapshot` 时重建
- **paneLayoutStore**：内部 `_cachedSnapshot`，`_refreshSnapshot()` 时更新
- **serverStore**：内部 `_serversSnapshot` 等三个快照，`notify()` 时更新
- **childSessionStore**：模块级 `childSessionsCache` 和 `sessionFamilyCache` Map，订阅回调中清除
- **todoStore**：使用 `version` 计数器，`getVersion()` 返回数字作为快照

### 6.3 不可变更新模式

所有 Store 的状态变更遵循不可变更新原则：

```typescript
// paneLayoutStore — 树操作返回新树
_root = replaceNode(_root, paneId, { ...leaf, sessionId })

// messageStore — 消息数组不可变替换
state.messages = [...state.messages.slice(0, idx), newMessage, ...state.messages.slice(idx + 1)]

// serverStore — 快照不可变复制
this._serversSnapshot = [...this.servers]
```

例外：`handlePartDelta` 在 RAF 批量期间使用 mutable 修改 `(part as { text: string }).text += data.delta`，由 `flushDirtyMessages()` 在通知前统一转为不可变快照。

### 6.4 跨 Store 协调

Store 之间不直接相互引用，通过以下机制协调：

1. **事件回调机制**：`serverStore.onServerChange()` 允许外部注册切换回调
2. **共享订阅**：`messageStoreHooks.ts` 同时订阅 `messageStore` 和 `paneLayoutStore`，任一变更均清除缓存
3. **main.tsx 编排**：应用初始化时在 `main.tsx` 中统一注册跨 Store 回调

### 6.5 持久化时机

- **构造函数中读取**：所有需要持久化的 Store 在 `constructor()` 中从 storage 读取初始值
- **notify() 时写入**：`layoutStore`、`serverStore`、`notificationStore`、`soundStore`、`keybindingStore`、`themeStore`、`serviceStore` 在 `notify()` 或 setter 中同步写入 storage
- **setter 中直接写入**：部分 Store（如 `themeStore`）在 setter 中直接写入 localStorage，不依赖 `notify()`

---

## 7. Error Handling

### 7.1 Storage 异常处理

所有 localStorage/IndexedDB 操作均包裹 `try-catch`：

```typescript
try {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
} catch {
  // ignore — quota exceeded, private mode, etc.
}
```

受影响的 Store：`serverStore`、`layoutStore`、`keybindingStore`、`themeStore`、`notificationStore`、`soundStore`、`serviceStore`、`autoApproveStore`。

### 7.2 持久化数据校验

- **layoutStore**：`sanitizePersistedPanelLayout()` 和 `sanitizePersistedTerminalLayoutMap()` 对读取的 JSON 做严格类型校验，版本不匹配或字段缺失时回退默认值
- **soundStore**：`loadSettings()` 合并默认值，防止旧版本缺字段
- **serverStore**：`loadFromStorage()` 失败时创建默认本地服务器

### 7.3 SSE 事件容错

`messageStore` 的 SSE 事件处理器采用"找不到则丢弃"策略：

```typescript
handlePartDelta(data) {
  const state = this.sessions.get(data.sessionID)
  if (!state) return          // 会话不存在，丢弃
  const msg = state.messages.find(m => m.info.id === data.messageID)
  if (!msg) return            // 消息不存在，丢弃
  const part = msg.parts.find(p => p.id === data.partID)
  if (!part) return           // part 不存在，丢弃
  // ...
}
```

### 7.4 RAF 非浏览器环境降级

```typescript
if (typeof requestAnimationFrame !== 'undefined') {
  this.rafId = requestAnimationFrame(() => {
    /* ... */
  })
} else {
  this.pendingNotify = false
  this.flushDirtyMessages()
  this.subscribers.forEach(fn => fn())
}
```

### 7.5 分屏树操作安全

`paneLayoutStore` 的所有树操作在执行前校验节点存在性：

```typescript
splitPane(paneId, direction) {
  const leaf = findLeaf(_root, paneId)
  if (!leaf) return null   // 节点不存在，安全返回
  // ...
}
```

分割比例限制在 0.15 到 0.85 之间：

```typescript
setRatio(splitId, ratio) {
  const clamped = Math.max(0.15, Math.min(0.85, ratio))
  // ...
}
```

---

## 8. Testing Considerations

### 8.1 现有测试覆盖

| 测试文件                     | 覆盖 Store        | 测试类型           |
| ---------------------------- | ----------------- | ------------------ |
| `messageStore.test.ts`       | messageStore      | 单元测试（Vitest） |
| `messageStoreHooks.test.tsx` | messageStoreHooks | 组件测试           |
| `layoutStore.test.ts`        | layoutStore       | 单元测试（Vitest） |

### 8.2 messageStore 测试要点

基于 `messageStore.test.ts` 的实际测试内容：

- SSE 事件处理：`handleMessageUpdated`、`handlePartUpdated`、`handlePartDelta`、`handlePartRemoved`
- 消息排序：按 `time.created` 排序
- 撤销.重做：`truncateAfterRevert`、`canUndo`、`canRedo`
- 流式状态：`handleSessionIdle` 后 `isStreaming` 变为 false
- 内存淘汰：`evictOldSessions` 行为

### 8.3 layoutStore 测试要点

基于 `layoutStore.test.ts` 的实际测试内容：

- 面板布局持久化：验证 terminal tabs 不被持久化
- 面板开关状态：syncTerminalSessions 后保持面板打开
- 数据恢复：新建 LayoutStore 实例后正确恢复

### 8.4 建议补充的测试

基于代码分析，以下 Store 缺少测试覆盖：

| Store                | 建议测试场景                                                |
| -------------------- | ----------------------------------------------------------- |
| `paneLayoutStore`    | 分割/关闭/交换/全屏操作的正确性，树结构不变性               |
| `serverStore`        | 多服务器 CRUD、健康检查、sessionStorage + localStorage 双写 |
| `keybindingStore`    | 快捷键解析、冲突检测、规范化比较                            |
| `notificationStore`  | Toast 生命周期（8 秒自动消失、悬停暂停、最多 3 个）         |
| `activeSessionStore` | pending request 注册/解析、deferred idle 逻辑               |
| `childSessionStore`  | 父子关系注册、递归查询、clearAll                            |
| `autoApproveStore`   | 通配符匹配、Full Auto 模式切换、per-pane 模式               |
| `soundStore`         | IndexedDB 读写、音频预加载、设置持久化                      |
| `followupQueueStore` | 入队/出队、失败重试、sending 状态机                         |
| `changeScopeStore`   | 模式切换、clearAll                                          |

### 8.5 测试策略建议

- **Store 逻辑测试**：直接实例化 class（如 `new LayoutStore()`），绕过单例，便于隔离测试
- **React Hook 测试**：使用 `@testing-library/react` 的 `renderHook` 测试 `useSyncExternalStore` 桥接
- **持久化测试**：每个测试前后 `localStorage.clear()`，避免测试间污染
- **RAF 测试**：使用 `vi.useFakeTimers()` 模拟 `requestAnimationFrame`

---

## 9. 风险与缓解

| 风险                             | 影响                       | 当前缓解措施                       | 建议改进                                                       |
| -------------------------------- | -------------------------- | ---------------------------------- | -------------------------------------------------------------- |
| localStorage 容量超限            | 状态写入失败，用户配置丢失 | 所有写入包裹 try-catch             | 增加用户提示或自动清理旧数据                                   |
| 快照缓存未正确清除               | UI 显示过期状态            | 订阅回调中统一清除                 | 增加缓存一致性断言                                             |
| RAF 在非浏览器环境不可用         | 通知机制失效               | 检测并降级为立即通知               | 已完善                                                         |
| 分屏树操作产生无效状态           | 界面渲染异常               | 不可变操作 + 节点存在性校验        | 增加树结构不变性测试                                           |
| 服务器切换时部分 Store 未清理    | 旧服务器数据污染新服务器   | onServerChange 回调清理 7 个 Store | notificationStore、changeScopeStore、followupQueueStore 未清理 |
| IndexedDB 操作异步导致状态不一致 | 自定义音频加载延迟         | 预加载 + 内存缓存 + loading 追踪   | 已完善                                                         |
| messageStore 体积过大（696 行）  | 维护困难                   | 类型和 Hooks 已分离到独立文件      | 考虑将 SSE handler 方法提取到独立模块                          |
| 跨 Store 事务性更新需手动协调    | 状态不一致                 | 通过事件回调机制                   | 考虑引入统一的事务协调层                                       |

---

## 10. 架构决策回溯

### ADR-001：自定义发布.订阅 vs 第三方状态库

**决策**：不使用 Redux/Zustand/Jotai，采用自定义发布.订阅模式。

**实际实现印证**：

- 16 个 Store 全部为自定义实现，零第三方状态库
- 通过 `useSyncExternalStore` 桥接 React，利用 React 18+ 并发特性避免 tearing
- 每个 Store 独立管理持久化，职责清晰

### ADR-002：messageStore 使用 RAF 批量通知

**决策**：messageStore 的 notify 通过 `requestAnimationFrame` 节流。

**实际实现印证**：

- `pendingNotify` 标志 + `rafId` 追踪实现 RAF 节流
- `dirtyMessages` Set + `flushDirtyMessages()` 实现精准不可变快照生成
- 非浏览器环境降级为 `notifyImmediate()`

### ADR-003：消息状态不持久化

**决策**：messageStore 完全由服务端数据驱动，不持久化到本地存储。

**实际实现印证**：

- messageStore 无任何 localStorage/IndexedDB 读写
- `clearAll()` 在服务器切换时被调用
- 消息数据完全存储在内存 Map 中

### ADR-004：分屏布局使用二叉树结构

**决策**：paneLayoutStore 使用递归二叉树管理分屏。

**实际实现印证**：

- `PaneNode = PaneLeaf | PaneSplit` 联合类型
- `replaceNode`、`removeLeaf`、`swapLeafSessions` 等不可变树操作
- 分割比例限制 0.15-0.85

### ADR-005：活动服务器 sessionStorage + localStorage 双写

**决策**：serverStore 的活动服务器 ID 同时写入两种存储。

**实际实现印证**：

- 读取优先级：`sessionStorage` > `localStorage`
- 写入：同时写入两者
- 新窗口首次打开时从 localStorage 继承，窗口内刷新从 sessionStorage 读取

---

## 11. 数据流总结

### 11.1 SSE 事件 → Store → UI

```
后端 SSE 事件
  │
  ▼
useGlobalEvents Hook（App 组件中，全局唯一订阅点）
  │
  ├── message.updated ──────► messageStore.handleMessageUpdated()
  ├── message.part.updated ─► messageStore.handlePartUpdated()
  ├── message.part.delta ───► messageStore.handlePartDelta()     (RAF 批量)
  ├── message.part.removed ─► messageStore.handlePartRemoved()
  ├── session.status ───────► activeSessionStore.updateStatus()
  ├── session.idle ─────────► messageStore.handleSessionIdle()
  ├── session.error ────────► messageStore.handleSessionError()
  ├── session.created ──────► childSessionStore.registerChildSession()
  ├── todo.updated ─────────► todoStore.setTodos()
  ├── permission.asked ─────► activeSessionStore.addPendingRequest()
  │                         ──► notificationStore.push()
  └── question.asked ───────► activeSessionStore.addPendingRequest()
                                ──► notificationStore.push()
```

### 11.2 用户操作 → Store → UI

```
用户操作
  │
  ├── layoutStore.setSidebarExpanded(true)
  ├── themeStore.setPreset('claude')
  ├── keybindingStore.setKeybinding('sendMessage', 'Ctrl+Enter')
  ├── paneLayoutStore.splitPane(paneId, 'horizontal')
  ├── serverStore.setActiveServer(id)  → 触发 onServerChange 回调链
  ├── messageStore.truncateAfterRevert(sessionId)
  └── notificationStore.dismiss(id)
       │
       ▼
  Store 更新内部状态 → notify() → 订阅者回调 → getSnapshot() → UI 重渲染
```

---

_本文档基于 OpenCodeUI v0.4.8 实际代码回溯生成，所有细节均来自源码分析。_
