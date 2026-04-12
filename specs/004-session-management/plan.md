# 004-session-management 模块实现方案（As-Built）

> 模块编号：004-session-management
> 版本：v0.4.8
> 生成日期：2026-04-12
> 状态：已实现，本文档为回溯性技术规划

---

## 1. Technical Context

### 1.1 模块定位

会话管理模块是 OpenCodeUI 的核心业务模块之一，负责会话的完整生命周期管理。它在架构中处于 API 层与 UI 层之间，通过 Context Provider、自定义 Hook 和独立 Store 三层结构向消费方提供能力。

### 1.2 源码文件清单

| 文件路径                                    | 行数 | 职责                                                                  |
| ------------------------------------------- | ---- | --------------------------------------------------------------------- |
| `src/features/sessions/SessionList.tsx`     | 753  | 会话列表 UI 组件（含时间分组、搜索、编辑模式、子会话插槽）            |
| `src/features/sessions/ProjectSelector.tsx` | 236  | 项目选择器下拉组件                                                    |
| `src/features/sessions/index.ts`            | 3    | 模块 barrel 导出                                                      |
| `src/contexts/SessionContext.tsx`           | 253  | 会话列表 Context Provider（SSE 订阅、分页、CRUD）                     |
| `src/contexts/SessionContext.shared.ts`     | —    | Context 定义与类型声明                                                |
| `src/hooks/useSessions.ts`                  | 289  | 独立会话列表 Hook（与 SessionContext 逻辑等价，供懒加载场景使用）     |
| `src/hooks/useSessionManager.ts`            | 375  | 单个会话的消息加载、历史分页、undo/redo                               |
| `src/hooks/useSessionStats.ts`              | 82   | Token 消耗、费用、上下文使用率计算                                    |
| `src/hooks/useChatSession.ts`               | 1068 | 聊天会话生命周期 Hook（发消息、权限处理、分叉、Agent 切换、队列跟进） |
| `src/hooks/useGlobalEvents.ts`              | 526  | 全局 SSE 事件订阅与消费者分发机制                                     |
| `src/store/messageStore.ts`                 | 696  | 多 session 消息存储、SSE 事件处理、RAF 批量通知                       |
| `src/store/messageStoreTypes.ts`            | 80   | 消息 Store 类型定义                                                   |
| `src/store/messageStoreHooks.ts`            | —    | 细粒度 React hooks 桥接                                               |
| `src/store/activeSessionStore.ts`           | 314  | 活跃状态追踪 Store                                                    |
| `src/store/childSessionStore.ts`            | 279  | 子会话关系追踪 Store                                                  |
| `src/api/session.ts`                        | 253  | Session API 函数封装                                                  |

### 1.3 外部依赖

- **@opencode-ai/sdk**：所有 API 调用通过官方 SDK 的 `sdk.session.*` 方法
- **SSE 事件流**：通过 `src/api/events.ts` 的单例连接订阅，事件类型包括 `session.created`、`session.updated`、`session.deleted`、`session.status`、`session.idle`、`session.error` 等
- **react-i18next**：用户可见文本通过 `useTranslation(['commands', 'common', 'chat'])` 获取

### 1.4 被依赖关系

- `002-chat-feature`：通过 `useChatSession`、`SessionProvider`、`useSessionContext` 消费会话能力
- `003-message-rendering`：通过 `messageStore` 获取消息数据
- `012-pane-layout`：通过 `paneControllerStore` 间接调用会话操作
- 侧边栏组件：直接使用 `SessionList`、`ProjectSelector` 组件

---

## 2. Constitution Check

### 2.1 原则对照

| 宪法原则                    | 遵循情况    | 说明                                                               |
| --------------------------- | ----------- | ------------------------------------------------------------------ |
| 原则 2：OpenCode 兼容性优先 | ✅ 完全遵循 | 所有 API 调用通过 `@opencode-ai/sdk`，无自行封装 HTTP 请求         |
| 原则 4：自定义优于框架依赖  | ✅ 完全遵循 | 自定义 Store 模式（非 Redux/Zustand），自定义 Context 管理会话列表 |
| 原则 5：实时通信优先        | ✅ 完全遵循 | SSE 为第一优先级，`subscribeToEvents` 实现实时更新列表和状态       |
| 原则 10：模块化功能架构     | ✅ 完全遵循 | `src/features/sessions/` 自包含 UI 组件，Store 和 Hook 独立模块    |

### 2.2 约束条件对照

| 约束           | 遵循情况 | 说明                                              |
| -------------- | -------- | ------------------------------------------------- |
| C4：依赖最小化 | ✅       | 未引入额外状态管理库，复用已有的自定义 Store 模式 |
| C5：SSE 支持   | ✅       | 会话列表、活跃状态、子会话均通过 SSE 实时更新     |

---

## 3. Research Findings

### 3.1 会话列表加载策略

实际实现采用**递增 limit 策略**，与 spec 中 ADR-001 一致：

- `SessionContext` 中 `currentLimitRef` 初始值为 30，每次 `loadMore` 递增 15
- `useSessions` Hook 中 `currentLimitRef` 初始值为 `pageSize`（默认 20），每次 `loadMore` 递增 `pageSize`
- 两者均使用 `requestIdRef` 防止竞态条件，仅采纳最后一次请求的结果
- 搜索态下（`search` 非空）触发 300ms 防抖，无搜索词时立即执行

**关键差异**：`SessionContext` 的 `loadMore` 使用 `append: true` 模式做数据追加去重合并，而 `useSessions` 的 `loadMore` 是全量替换。两者策略不同，`SessionContext` 更接近官方 WebUI 行为。

### 3.2 SSE 事件实时更新

`SessionContext` 和 `useSessions` 均通过 `subscribeToEvents` 订阅三类事件：

- **onSessionCreated**：子会话（有 `parentID`）直接忽略，非子会话追加到列表顶部。搜索态下触发重新拉取
- **onSessionUpdated**：匹配当前目录的会话更新并置顶，不匹配的从列表移除。搜索态下触发重新拉取
- **onSessionDeleted**：直接从本地列表过滤移除
- **onReconnected**：清空列表（`setSessions([])`）并重新拉取

### 3.3 目录匹配策略

使用 `isSameDirectory()` 函数进行语义匹配（非字符串精确匹配），配合 `normalizeToForwardSlash()` 统一路径格式。`autoDetectPathStyle()` 在首次收到后端返回的 `directory` 字段时自动检测路径风格。

### 3.4 消费者注册与事件分发

`useGlobalEvents.ts` 实现了 session 级消费者注册机制：

- `sessionConsumers` 是一个 `Map<string, SessionConsumer>`，key 为 `consumerId`（通常为 `paneId`）
- `registerSessionConsumer(consumerId, sessionId, callbacks)` 注册消费者，返回注销函数
- `updateConsumerSessionId(consumerId, sessionId)` 更新消费者关注的 sessionId，无需重新注册
- `dispatchToConsumers(sessionId, invoke)` 按 sessionId 分发事件，同时通过 `childSessionStore.belongsToSession()` 检查子会话冒泡
- 支持的事件类型：`onPermissionAsked`、`onPermissionReplied`、`onQuestionAsked`、`onQuestionReplied`、`onQuestionRejected`、`onScrollRequest`、`onSessionIdle`、`onSessionError`、`onReconnected`

### 3.5 MessageStore 多 Session 存储

`MessageStore` 采用 `Map<string, SessionState>` 存储每个 session 的消息：

- `MAX_CACHED_SESSIONS = 10`，超出时淘汰最久未访问的 session
- `protectedSessions` 集合保护分屏模式下被查看的 session 不被淘汰
- Delta 批量化：`dirtyMessages`（Set<messageID>）+ `dirtySessionId` 追踪一帧内的 mutable 修改，`flushDirtyMessages()` 在 RAF 回调中统一做不可变快照
- `notify()` 使用 `requestAnimationFrame` 批量合并通知，避免高频 SSE 事件导致 React 频繁重渲染
- `markAllStale()` 在 SSE 重连后标记所有 session 数据可能过期

### 3.6 两套会话列表管理并存

代码中存在两套会话列表管理实现：

1. **`SessionContext`**（`src/contexts/SessionContext.tsx`）：在 `App` 根组件中作为 Provider 使用，管理全局会话列表
2. **`useSessions` Hook**（`src/hooks/useSessions.ts`）：独立的 Hook 实现，逻辑与 `SessionContext` 高度相似，但 pageSize 默认 20、loadMore 增量 15（与 SessionContext 一致），供懒加载场景（如文件夹视图）使用

两者差异：

- `SessionContext` 的 `loadMore` 使用 append 模式（去重追加），`useSessions` 使用全量替换
- `useSessions` 提供 `patchLocalSession` 和 `removeLocalSession` 两个额外的本地操作方法
- `useSessions` 支持 `enabled` 参数控制懒加载启用时机

---

## 4. Data Model

### 4.1 前端镜像实体

以下类型定义来源于 `src/api/types.ts` 和 `src/types/`，镜像后端 OpenAPI 模型：

**ApiSession**（会话元数据）

| 字段         | 类型               | 说明                                             |
| ------------ | ------------------ | ------------------------------------------------ |
| `id`         | string             | 会话唯一标识                                     |
| `slug`       | string             | 短标识                                           |
| `projectID`  | string             | 关联项目 ID                                      |
| `directory`  | string             | 工作目录路径                                     |
| `parentID`   | string?            | 父会话 ID                                        |
| `title`      | string             | 会话标题                                         |
| `version`    | string             | 版本号                                           |
| `summary`    | SessionSummary?    | 变更摘要（additions, deletions, files, diffs）   |
| `share`      | SessionShare?      | 分享信息（含 url）                               |
| `time`       | SessionTime        | 时间戳（created, updated, compacting, archived） |
| `permission` | PermissionRuleset? | 权限规则集                                       |
| `revert`     | SessionRevert?     | 回退状态（含 messageID）                         |

**SessionStatus**（多态类型）

| 变体    | 字段                         | 说明   |
| ------- | ---------------------------- | ------ |
| `idle`  | —                            | 空闲   |
| `busy`  | —                            | 忙碌   |
| `retry` | `attempt`, `message`, `next` | 重试中 |

**SessionStatusMap**：`Record<string, SessionStatus>`，sessionId 到状态的映射。

### 4.2 前端自建实体

**ActiveSessionEntry**（`activeSessionStore.ts`）

| 字段            | 类型                  | 说明                         |
| --------------- | --------------------- | ---------------------------- | -------------- |
| `sessionId`     | string                | 会话 ID                      |
| `status`        | SessionStatus         | 当前状态                     |
| `title`         | string?               | 标题缓存                     |
| `directory`     | string?               | 目录缓存                     |
| `pendingAction` | `{ type: 'permission' | 'question', description? }`? | 等待的用户操作 |

**ChildSessionInfo**（`childSessionStore.ts`）

| 字段        | 类型       | 说明          |
| ----------- | ---------- | ------------- | -------- | ---- |
| `id`        | string     | 子会话 ID     |
| `parentID`  | string     | 父会话 ID     |
| `title`     | string     | 子会话标题    |
| `agent`     | string?    | 子 Agent 名称 |
| `status`    | `'running' | 'idle'        | 'error'` | 状态 |
| `createdAt` | number     | 创建时间戳    |

**SessionState**（`messageStoreTypes.ts`）

| 字段             | 类型         | 说明             |
| ---------------- | ------------ | ---------------- | -------- | -------- | -------- |
| `messages`       | Message[]    | 消息列表         |
| `revertState`    | RevertState? | 回退状态         |
| `isStreaming`    | boolean      | 是否正在流式输出 |
| `loadState`      | `'idle'      | 'loading'        | 'loaded' | 'error'` | 加载状态 |
| `hasMoreHistory` | boolean      | 是否有更多历史   |
| `directory`      | string       | 工作目录         |
| `title`          | string?      | 会话标题         |
| `shareUrl`       | string?      | 分享链接         |
| `isStale`        | boolean      | 数据可能过期     |

**RevertState**（`messageStoreTypes.ts`）

| 字段        | 类型                | 说明               |
| ----------- | ------------------- | ------------------ |
| `messageId` | string              | 回退点消息 ID      |
| `history`   | RevertHistoryItem[] | 被撤销的消息历史栈 |

**RevertHistoryItem**（`messageStoreTypes.ts`）

| 字段          | 类型                                 | 说明         |
| ------------- | ------------------------------------ | ------------ |
| `messageId`   | string                               | 消息 ID      |
| `text`        | string                               | 消息文本内容 |
| `attachments` | unknown[]                            | 附件列表     |
| `model`       | `{ providerID, modelID, variant? }`? | 模型信息     |
| `variant`     | string?                              | 模型变体     |
| `agent`       | string?                              | Agent 名称   |

**PendingRequest**（`activeSessionStore.ts`）

| 字段          | 类型          | 说明        |
| ------------- | ------------- | ----------- | -------- |
| `requestId`   | string        | 请求 ID     |
| `sessionId`   | string        | 所属会话 ID |
| `type`        | `'permission' | 'question'` | 请求类型 |
| `description` | string?       | 请求描述    |

### 4.3 会话统计（运行时计算，非持久化实体）

`useSessionStats` Hook 返回的 `SessionStats` 接口：

| 字段              | 类型   | 计算方式                                            |
| ----------------- | ------ | --------------------------------------------------- |
| `inputTokens`     | number | 累加所有 assistant 消息的 `tokens.input`            |
| `outputTokens`    | number | 累加 `tokens.output`                                |
| `reasoningTokens` | number | 累加 `tokens.reasoning`                             |
| `cacheRead`       | number | 累加 `tokens.cache.read`                            |
| `cacheWrite`      | number | 累加 `tokens.cache.write`                           |
| `totalTokens`     | number | 以上五项之和                                        |
| `totalCost`       | number | 累加所有 assistant 消息的 `cost`                    |
| `contextUsed`     | number | 最后一条有 token 数据的 assistant 消息的总 token 数 |
| `contextLimit`    | number | 传入参数，默认 200000                               |
| `contextPercent`  | number | `Math.min(100, (contextUsed / contextLimit) * 100)` |

---

## 5. Interface Contracts

### 5.1 Session API 接口（`src/api/session.ts`）

| 函数名                                                     | 参数                                                  | 返回值                      | 用途                    |
| ---------------------------------------------------------- | ----------------------------------------------------- | --------------------------- | ----------------------- |
| `getSessionStatus(directory?)`                             | directory?: string                                    | `Promise<SessionStatusMap>` | 获取所有 session 状态   |
| `getSessionDiff(sessionId, directory?, messageId?)`        | sessionId, directory?, messageId?                     | `Promise<FileDiff[]>`       | 获取 session 文件差异   |
| `getLastTurnDiff(sessionId, directory?)`                   | sessionId, directory?                                 | `Promise<FileDiff[]>`       | 获取当前可见轮次的 diff |
| `getSessions(params?)`                                     | SessionListParams                                     | `Promise<ApiSession[]>`     | 获取会话列表            |
| `getSession(sessionId, directory?)`                        | sessionId, directory?                                 | `Promise<ApiSession>`       | 获取单个会话            |
| `createSession(params?)`                                   | { directory?, title?, parentID? }                     | `Promise<ApiSession>`       | 创建会话                |
| `updateSession(sessionId, params, directory?)`             | sessionId, { title?, time? }, directory?              | `Promise<ApiSession>`       | 更新会话                |
| `deleteSession(sessionId, directory?)`                     | sessionId, directory?                                 | `Promise<boolean>`          | 删除会话                |
| `abortSession(sessionId, directory?)`                      | sessionId, directory?                                 | `Promise<boolean>`          | 中止会话                |
| `revertMessage(sessionId, messageId, partId?, directory?)` | sessionId, messageId, partId?, directory?             | `Promise<ApiSession>`       | 回退消息                |
| `unrevertSession(sessionId, directory?)`                   | sessionId, directory?                                 | `Promise<ApiSession>`       | 恢复回退                |
| `shareSession(sessionId, directory?)`                      | sessionId, directory?                                 | `Promise<ApiSession>`       | 分享会话                |
| `unshareSession(sessionId, directory?)`                    | sessionId, directory?                                 | `Promise<ApiSession>`       | 取消分享                |
| `forkSession(sessionId, messageId?, directory?)`           | sessionId, messageId?, directory?                     | `Promise<ApiSession>`       | 分叉会话                |
| `summarizeSession(sessionId, params, directory?)`          | sessionId, { providerID, modelID, auto? }, directory? | `Promise<boolean>`          | 总结会话                |
| `getSessionChildren(sessionId, directory?)`                | sessionId, directory?                                 | `Promise<ApiSession[]>`     | 获取子会话              |
| `getSessionTodos(sessionId, directory?)`                   | sessionId, directory?                                 | `Promise<ApiTodo[]>`        | 获取待办列表            |

### 5.2 SessionContext Value 接口

```typescript
interface SessionContextValue {
  sessions: ApiSession[]
  isLoading: boolean
  isLoadingMore: boolean
  hasMore: boolean
  search: string
  setSearch: (search: string) => void
  refresh: () => Promise<void>
  loadMore: () => Promise<void>
  createSession: (title?: string) => Promise<ApiSession>
  deleteSession: (id: string) => Promise<void>
}
```

### 5.3 useSessions Hook 接口

```typescript
interface UseSessionsResult {
  sessions: ApiSession[]
  isLoading: boolean
  isLoadingMore: boolean
  error: Error | null
  hasMore: boolean
  search: string
  setSearch: (search: string) => void
  loadMore: () => Promise<void>
  refresh: () => Promise<void>
  create: (title?: string) => Promise<ApiSession>
  remove: (sessionId: string) => Promise<void>
  patchLocalSession: (sessionId: string, patch: Partial<ApiSession>) => void
  removeLocalSession: (sessionId: string) => void
}
```

### 5.4 useSessionManager Hook 接口

```typescript
interface UseSessionManagerReturn {
  loadSession: (sid: string, options?: { force?: boolean }) => Promise<void>
  loadMoreHistory: () => Promise<void>
  handleUndo: (userMessageId: string) => Promise<void>
  handleRedo: () => Promise<void>
  handleRedoAll: () => Promise<void>
  clearRevert: () => void
}
```

### 5.5 ActiveSessionStore 公共 API

| 方法                                                          | 参数                          | 返回值               | 用途                |
| ------------------------------------------------------------- | ----------------------------- | -------------------- | ------------------- |
| `initialize(statusMap)`                                       | SessionStatusMap              | void                 | 初始化全量状态      |
| `initializePendingRequests(permissions, questions)`           | permissions[], questions[]    | void                 | 初始化 pending 请求 |
| `addPendingRequest(requestId, sessionId, type, description?)` | string, string, type, string? | void                 | 注册 pending 请求   |
| `resolvePendingRequest(requestId)`                            | string                        | void                 | 移除 pending 请求   |
| `updateStatus(sessionId, status)`                             | string, SessionStatus         | void                 | 更新 session 状态   |
| `setSessionMeta(sessionId, title?, directory?)`               | string, string?, string?      | void                 | 设置元信息          |
| `setSessionMetaBulk(entries)`                                 | SessionMetaEntry[]            | void                 | 批量设置元信息      |
| `getBusySessions()`                                           | —                             | ActiveSessionEntry[] | 获取活跃列表        |
| `busyCount` (getter)                                          | —                             | number               | 活跃计数            |

**React Hooks**：

- `useActiveSessionStore()`：返回完整状态
- `useBusySessions()`：返回 `ActiveSessionEntry[]`
- `useBusyCount()`：返回 `number`
- `useSessionActiveEntry(sessionId)`：返回单个 entry 或 `undefined`

### 5.6 ChildSessionStore 公共 API

| 方法                                         | 参数                    | 返回值             | 用途                   |
| -------------------------------------------- | ----------------------- | ------------------ | ---------------------- |
| `registerChildSession(session)`              | ApiSession              | void               | 注册子会话             |
| `updateChildSession(sessionId, updates)`     | string, Partial<...>    | void               | 更新子会话信息         |
| `markIdle(sessionId)`                        | string                  | void               | 标记为 idle            |
| `markError(sessionId)`                       | string                  | void               | 标记为 error           |
| `getChildSessionIds(parentId)`               | string                  | string[]           | 获取子会话 ID 列表     |
| `getChildSessions(parentId)`                 | string                  | ChildSessionInfo[] | 获取子会话信息列表     |
| `getSessionInfo(sessionId)`                  | string                  | ChildSessionInfo?  | 获取单个子会话信息     |
| `isChildOf(sessionId, parentId, recursive?)` | string, string, boolean | boolean            | 检查父子关系           |
| `getSessionAndDescendants(sessionId)`        | string                  | string[]           | 获取 session family    |
| `belongsToSession(sessionId, rootSessionId)` | string, string          | boolean            | 归属检查               |
| `clearAll()`                                 | —                       | void               | 清空所有数据           |
| `clearChildren(parentId)`                    | string                  | void               | 清理指定父会话的子记录 |

**React Hooks**：

- `useChildSessions(parentId)`：返回 `ChildSessionInfo[]`
- `useSessionFamily(sessionId)`：返回 `string[]`

### 5.7 SessionList 组件 Props

```typescript
interface SessionListProps {
  sessions: ApiSession[]
  selectedId: string | null
  isLoading: boolean
  isLoadingMore: boolean
  hasMore: boolean
  search: string
  onSearchChange: (search: string) => void
  onSelect: (session: ApiSession) => void
  onDelete: (sessionId: string) => void
  onRename: (sessionId: string, newTitle: string) => void
  onLoadMore: () => void
  onNewChat: () => void
  showHeader?: boolean
  grouped?: boolean
  density?: 'default' | 'compact' | 'minimal'
  showStats?: boolean
  showDirectory?: boolean
  expandedChildSessionIds?: Set<string>
  inlineChildSessions?: Map<string, ApiSession[]>
  onSelectChildSession?: (session: ApiSession) => void
  isEditMode?: boolean
  selectedSessionIds?: Set<string>
  onToggleSessionSelection?: (sessionId: string, options?: { shiftKey?: boolean }) => void
}
```

### 5.8 ProjectSelector 组件 Props

```typescript
interface ProjectSelectorProps {
  currentProject: ApiProject | null
  projects: ApiProject[]
  isLoading: boolean
  onSelectProject: (projectId: string) => void
  onAddProject: () => void
  onRemoveProject: (projectId: string) => void
}
```

---

## 6. Implementation Strategy

### 6.1 架构分层

```
┌─────────────────────────────────────────────────────────┐
│                    UI Components                         │
│  SessionList, ProjectSelector, SessionListItem           │
│  (src/features/sessions/)                                │
├─────────────────────────────────────────────────────────┤
│                    Context & Hooks                       │
│  SessionContext (Provider)                               │
│  useSessions, useSessionManager, useSessionStats         │
│  useChatSession (综合编排)                                │
├─────────────────────────────────────────────────────────┤
│                    Store Layer                           │
│  messageStore (多 session 消息存储)                       │
│  activeSessionStore (活跃状态追踪)                        │
│  childSessionStore (子会话关系)                           │
├─────────────────────────────────────────────────────────┤
│                    API Layer                             │
│  src/api/session.ts (SDK 封装)                            │
│  src/api/events.ts (SSE 订阅)                             │
└─────────────────────────────────────────────────────────┘
```

### 6.2 会话列表加载流程

```
SessionProvider 挂载
    │
    ▼
useDirectory 获取 currentDirectory
    │
    ▼
fetchSessions() — 调用 getSessions({ roots: true, limit: 30, directory })
    │
    ├── 成功 → setSessions(data), setHasMore(data.length >= limit)
    │
    └── 失败 → sessionErrorHandler
    │
    ▼
subscribeToEvents 注册 SSE 监听
    │
    ├── session.created → 追加到列表顶部（子 session 忽略）
    ├── session.updated → 更新并置顶（目录不匹配则移除）
    ├── session.deleted → 从列表移除
    └── reconnected → 清空列表 + 重新拉取
    │
    ▼
用户滚动触底 → loadMore() → limit += 15 → 重新请求全量列表 → 去重追加
```

### 6.3 单个会话消息加载流程

```
sessionId 变化（useSessionManager effect）
    │
    ▼
检查 messageStore 缓存
    │
    ├── 有完整缓存（loaded && !stale && messages.length > 0）
    │   → 直接使用缓存，更新 cursor
    │
    └── 无缓存或不完整
        │
        ▼
    loadSession(sessionId)
        │
        ├── 检查是否正在 streaming → 仅更新元数据，不覆盖消息
        │
        ├── Promise.all([getSession, getSessionMessages(limit)])
        │
        ├── 检查序列号（防竞态）
        │
        ├── 合并本地 streaming 消息（mergeWithLocalStreamingMessages）
        │
        └── messageStore.setMessages(sid, merged, metadata)
            │
            ▼
        cursor 设置为 max(INITIAL_MESSAGE_LIMIT, apiMessages.length)
```

### 6.4 Undo/Redo 流程

**Undo**：

1. 调用 `revertMessage(sessionId, userMessageId, undefined, dir)` 设置服务器端 revert 点
2. 在本地找到 revert 点的消息索引
3. 收集被撤销的所有用户消息，构建 `RevertHistoryItem[]` 历史栈
4. 调用 `messageStore.setRevertState(sessionId, { messageId, history })`

**Redo（单步）**：

1. 从 `revertState.history` 取出第一条（最早撤销的）
2. 如果还有剩余历史，调用 `revertMessage` 设置新的 revert 点为下一条的 messageId
3. 如果无剩余历史，调用 `unrevertSession` 完全清除服务器端 revert 标记
4. 更新本地 `revertState`

**Redo All**：

1. 直接调用 `unrevertSession(sessionId, dir)`
2. `messageStore.setRevertState(sessionId, null)` 清除本地状态

### 6.5 活跃状态追踪流程

```
初始化阶段
    │
    ├── getSessionStatus() → activeSessionStore.initialize(statusMap)
    │
    └── getPendingPermissions() + getPendingQuestions()
        → activeSessionStore.initializePendingRequests(permissions, questions)
    │
    ▼
SSE 事件实时更新
    │
    ├── session.status → activeSessionStore.updateStatus(sessionId, status)
    │   ├── idle + 有 pending → deferredIdleSessions.add(sessionId)
    │   ├── idle + 无 pending → 从 statusMap 移除
    │   ├── retry → 设置 retry 状态（含 attempt, message, next）
    │   └── busy → 设置 busy 状态
    │
    ├── permission.asked → activeSessionStore.addPendingRequest(...)
    │   └── 确保 session 在 busy 列表
    │
    ├── permission.replied → activeSessionStore.resolvePendingRequest(requestId)
    │   └── 检查是否还有其他 pending，无则移出 busy
    │
    └── question.asked / question.replied → 同上
    │
    ▼
派生数据缓存
    │
    └── recomputeDerived() 每次 notify 前重新计算
        ├── 过滤 busy/retry 状态的 entries
        ├── 补充 sessionMeta（title, directory）
        └── 补充 pendingAction（从 pendingRequests 查询）
```

### 6.6 子会话管理流程

```
session.created 事件到达 useGlobalEvents
    │
    ▼
检查 session.parentID 是否存在
    │
    ├── 存在 → childSessionStore.registerChildSession(session)
    │   ├── childrenByParent.set(parentID, Set<childID>)
    │   └── sessionInfo.set(childID, ChildSessionInfo)
    │
    └── 不存在 → 忽略（非子会话）
    │
    ▼
权限请求冒泡
    │
    └── dispatchToConsumers(sessionId, invoke)
        ├── 直接匹配：consumer.sessionId === sessionId
        └── 递归匹配：childSessionStore.belongsToSession(sessionId, consumer.sessionId)
    │
    ▼
删除父会话时
    │
    └── childSessionStore.clearChildren(parentId)
        ├── 递归清理所有子孙 session
        └── 从 childrenByParent 和 sessionInfo 中移除
```

### 6.7 消息 Store 通知机制

```
SSE 事件到达 → messageStore.handleXxxEvent()
    │
    ├── mutable 修改对应 session 的 messages 数组
    ├── dirtyMessages.add(messageID)
    └── dirtySessionId = sessionId
    │
    ▼
notify()
    │
    ├── pendingNotify = true（防止重复调度）
    └── requestAnimationFrame(() => {
        flushDirtyMessages()  // 对 dirty 消息做不可变快照
        subscribers.forEach(fn => fn())
        pendingNotify = false
      })
    │
    ▼
React 组件通过 useSyncExternalStore 收到通知
    │
    └── 仅重渲染订阅了该 session 的组件
```

### 6.8 分叉会话流程

分叉操作在 `useChatSession` 中实现：

1. 调用 `forkSession(sessionId, messageId, directory)` 创建分支
2. 后端返回新 session 元数据
3. 调用 `navigateToSession(newSession.id)` 导航到新会话
4. 如果是用户消息分叉，提取该消息内容并预填充到输入框
5. 新 session 通过 `session.created` SSE 事件被添加到列表

### 6.9 分享流程

1. `shareSession(sessionId, directory)` 调用后端 API 生成公开 URL
2. 返回的 `ApiSession.share.url` 更新到 session 元数据
3. `messageStore.updateSessionMetadata` 同步更新 `shareUrl`
4. 取消分享调用 `unshareSession(sessionId, directory)`，清除 `shareUrl`

### 6.10 总结流程

1. `summarizeSession(sessionId, { providerID, modelID, auto }, directory)` 调用后端 API
2. 异步后台任务，不阻塞 UI
3. 完成后通过 `session.compacted` SSE 事件通知

---

## 7. Error Handling

### 7.1 错误处理策略

| 场景                           | 处理方式                                                       | 代码位置                                |
| ------------------------------ | -------------------------------------------------------------- | --------------------------------------- |
| 会话列表加载失败               | `sessionErrorHandler('fetch sessions', e)` 记录日志，不阻断 UI | `SessionContext.tsx`                    |
| 单个会话加载失败               | `messageStore.setLoadState(sid, 'error')` + `onError?.()`      | `useSessionManager.ts`                  |
| 加载更多历史失败               | `sessionErrorHandler('load more history', error)` 静默处理     | `useSessionManager.ts`                  |
| Undo 操作失败                  | `sessionErrorHandler('undo', error)` 不影响当前消息显示        | `useSessionManager.ts`                  |
| Redo 操作失败                  | `sessionErrorHandler('redo', error)` 保持当前回退状态          | `useSessionManager.ts`                  |
| 创建会话失败                   | Promise reject 由调用方处理                                    | `SessionContext.tsx` / `useSessions.ts` |
| 删除会话失败                   | Promise reject 由调用方处理                                    | `SessionContext.tsx`                    |
| 元数据加载失败（streaming 中） | 静默忽略（`.catch(() => {})`）                                 | `useSessionManager.ts`                  |

### 7.2 竞态条件防护

- **请求序列号**：`loadSequenceRef`（Map<sessionId, number>）和 `requestIdRef` 确保仅采纳最后一次请求的结果
- **isStale 检查**：异步操作完成后检查 `isStale()`，过期结果直接丢弃
- **isLoadingMoreRef**：防止并发 `loadMore` 请求
- **force 模式**：SSE 重连后强制用服务器数据覆盖本地，避免数据不一致

### 7.3 内存泄漏防护

- **删除会话时清理**：`childSessionStore.clearChildren(id)` + `followupQueueStore.clearSession(id)`
- **服务器切换时清理**：`messageStore.clearAll()` + `childSessionStore.clearAll()` + `todoStore.clearAll()`
- **Session 缓存淘汰**：`MAX_CACHED_SESSIONS = 10`，超出时 LRU 淘汰（受保护的除外）
- **定时器清理**：`searchTimerRef` 在组件卸载时清除，`longPressTimer` 在 SessionListItem 卸载时清除

### 7.4 SSE 断流恢复

- `onReconnected` 事件触发后：
  1. 会话列表清空并重新拉取
  2. 所有 session 标记为 stale（`messageStore.markAllStale()`）
  3. 当前查看的 session 以 force 模式重新加载
  4. 待处理的权限/问题请求重新拉取（通过 `getPendingPermissions` + `getPendingQuestions`）

---

## 8. Testing Considerations

### 8.1 已有测试文件

| 测试文件                               | 覆盖范围                                          |
| -------------------------------------- | ------------------------------------------------- |
| `src/hooks/useSessions.test.tsx`       | useSessions Hook 的加载、搜索、分页、SSE 事件处理 |
| `src/hooks/useChatSession.test.tsx`    | useChatSession 的消息发送、权限处理、分叉等       |
| `src/store/messageStore.test.ts`       | MessageStore 的 CRUD、SSE 事件处理、回退状态      |
| `src/store/messageStoreHooks.test.tsx` | 细粒度 hooks 的重渲染行为                         |

### 8.2 建议补充的测试场景

**ActiveSessionStore**：

- 初始化后 statusMap 正确填充
- idle 事件 + 有 pending 请求时 session 保留在 busy 列表
- idle 事件 + 无 pending 请求时 session 移出 busy 列表
- retry 状态正确设置 attempt/message/next 字段
- resolvePendingRequest 后无其他 pending 时移出 busy
- setSessionMetaBulk 批量更新元信息
- deferredIdleSessions 的添加和移除逻辑

**ChildSessionStore**：

- registerChildSession 正确建立 parentID -> children 映射
- isChildOf 递归检查多层嵌套关系
- getSessionAndDescendants 返回完整的 session family
- clearChildren 递归清理所有子孙记录
- clearAll 清空所有数据

**SessionList 组件**：

- 时间分组正确（today/yesterday/previous7Days/previous30Days/older）
- 搜索态下不显示分组
- 滚动触底触发 onLoadMore
- 编辑模式下 checkbox 选择逻辑
- 长按手势（preferTouchUi = true）显示操作按钮
- 删除确认对话框交互

**useSessionManager**：

- 缓存命中时不发起网络请求
- force 模式覆盖本地数据
- streaming 中不覆盖消息仅更新元数据
- mergeWithLocalStreamingMessages 正确合并本地 streaming 消息
- undo 构建正确的 revertState
- redo 单步和全部重做的边界情况

**useSessionStats**：

- 空消息列表时所有统计值为 0
- streaming 中的消息（tokens 为空）不计入统计
- 多条 assistant 消息的 token 累加正确
- contextPercent 上限为 100
- cost 累加正确

### 8.3 集成测试建议

- SSE 事件流完整链路：session.created → 列表更新 → 活跃状态更新 → 子会话注册
- Undo/Redo 端到端：回退消息 → 视图隐藏 → 重做 → 视图恢复 → 全部重做 → 状态清除
- 项目切换：切换目录 → 列表重置 → 重新加载 → 分页 limit 归位
- SSE 重连恢复：断流 → 重连 → 列表清空重拉 → 当前 session force 加载 → pending 请求恢复

---

## 9. 关键常量

| 常量                      | 值     | 用途                             | 位置                                    |
| ------------------------- | ------ | -------------------------------- | --------------------------------------- |
| `MAX_CACHED_SESSIONS`     | 10     | MessageStore 最大缓存 session 数 | `messageStore.ts`                       |
| `currentLimitRef`（初始） | 30     | SessionContext 初始加载数量      | `SessionContext.tsx`                    |
| `loadMore` 增量           | 15     | SessionContext 每次加载更多递增  | `SessionContext.tsx`                    |
| `pageSize`（默认）        | 20     | useSessions 默认每页数量         | `useSessions.ts`                        |
| 搜索防抖                  | 300ms  | 搜索输入延迟执行                 | `SessionContext.tsx` / `useSessions.ts` |
| 长按触发                  | 500ms  | 触控设备长按显示操作按钮         | `SessionList.tsx`                       |
| 滚动触发阈值              | 100px  | 距底部 100px 时触发加载更多      | `SessionList.tsx`                       |
| `contextLimit`（默认）    | 200000 | 会话统计默认上下文限制           | `useSessionStats.ts`                    |

---

## 10. 风险区域

### 10.1 双列表实现并存（中风险）

`SessionContext` 和 `useSessions` 两套实现逻辑高度重叠但行为有细微差异（append vs 全量替换）。长期维护可能导致行为不一致。建议评估是否可以统一为单一实现。

### 10.2 MessageStore 复杂度（高风险）

作为最大的 Store（696 行），管理多 session 消息、SSE 事件处理、RAF 批量通知、Delta 批量化、发送前快照回滚等。任何修改都需要谨慎测试，避免引入重渲染性能问题或状态不一致。

### 10.3 活跃状态与 Pending 请求的时序（中风险）

`activeSessionStore` 中 `deferredIdleSessions` 的逻辑依赖 SSE 事件的到达顺序。如果 `session.idle` 事件先于 `permission.asked` 到达，可能导致短暂的显示不一致。当前通过 `initializePendingRequests` 在初始化时补充，但运行时仍有时序窗口。

### 10.4 子会话递归清理（低风险）

`childSessionStore.clearChildren` 使用递归实现，在极端情况下（深层嵌套的子会话链）可能导致栈溢出。实际使用中子会话层级通常不超过 2-3 层，风险较低。

---

_本文档基于 v0.4.8 实际代码回溯生成，所有细节来源于源码分析，未引入推测性内容。_
