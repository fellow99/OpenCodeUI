# 901-ext-server-quick-panel 技术方案（As-Built）

> 本文档是对已完成的服务快捷面板模块的回溯性技术规划，记录"实际建成"的架构设计、数据模型与集成策略。

---

## 1. Technical Context

### 1.1 模块定位

服务快捷面板（`901-ext-server-quick-panel`）是 OpenCodeUI 中提供跨服务器浏览和切换会话的快捷入口。它替换了侧边栏顶部 OpenCode 品牌标识原有的页面刷新行为，改为弹出一个浮动面板，以三级树形结构（服务器 → 工程 → 会话）展示所有已配置服务器及其活跃会话。

### 1.2 技术栈

| 维度     | 选型                                         |
| -------- | -------------------------------------------- |
| 框架     | React 19 + TypeScript                        |
| 样式     | Tailwind CSS v4                              |
| 国际化   | i18next + react-i18next                      |
| 状态管理 | 自定义 Store 模式（`serverStore`）           |
| 渲染     | `createPortal` 渲染到 document.body          |
| API 通信 | `@opencode-ai/sdk/v2/client` 独立 SDK 客户端 |

### 1.3 源码目录结构

```
src/features/chat/sidebar/
├── ServerQuickPanel.tsx              # 主面板组件（含 3 个内嵌子组件）
└── SidePanel.tsx                     # 触发器集成（修改）

src/locales/
├── zh-CN/
│   ├── chat.json                     # 新增 3 个键值
│   └── common.json                   # 新增 1 个键值
└── en/
    ├── chat.json                     # 新增 3 个键值
    └── common.json                   # 新增 1 个键值
```

### 1.4 文件规模

| 文件                                | 行数 | 职责                                   |
| ----------------------------------- | ---- | -------------------------------------- |
| `ServerQuickPanel.tsx`              | 617  | 面板容器、数据获取、树形渲染、关闭行为 |
| `SidePanel.tsx`（修改部分）         | ~30  | 触发按钮改造、状态管理、会话切换处理   |
| `locales/zh-CN/chat.json`（新增）   | 3    | 面板中文翻译                           |
| `locales/en/chat.json`（新增）      | 3    | 面板英文翻译                           |
| `locales/zh-CN/common.json`（新增） | 1    | 服务器计数中文翻译                     |
| `locales/en/common.json`（新增）    | 1    | 服务器计数英文翻译                     |

总计约 655 行代码。

---

## 2. Constitution Check

对照项目宪法逐项验证：

| 宪法原则                    | 符合情况 | 说明                                                                                   |
| --------------------------- | -------- | -------------------------------------------------------------------------------------- |
| 原则 2：OpenCode 兼容性优先 | 符合     | 直接使用 `@opencode-ai/sdk/v2/client` 调用项目/会话 API，无自定义协议                  |
| 原则 3：多平台统一代码库    | 符合     | 纯 React 组件，无平台特定代码，Web 端和桌面端共享同一套实现                            |
| 原则 4：自定义优于框架依赖  | 符合     | 未引入第三方弹出层库（如 Popper/Floating UI），使用原生 `createPortal` + 绝对定位实现  |
| 原则 6：中文优先文档        | 符合     | 代码注释、i18n 键名均以中文为第一语言                                                  |
| 原则 9：主题与可访问性      | 符合     | 使用 CSS 变量主题系统（`glass-alt`、`border-border-200/60`），ESC 键关闭，点击外部关闭 |
| 原则 10：模块化功能架构     | 符合     | 自包含于 `ServerQuickPanel.tsx`，通过 props 与 `SidePanel` 通信，无全局副作用          |
| 约束 C3：构建校验           | 符合     | 代码通过 TypeScript 类型检查（`tsc --noEmit` 零错误）                                  |
| 约束 C4：依赖最小化         | 符合     | 零新增 npm 依赖，仅复用已有 SDK、Store 和 Icon 组件                                    |

---

## 3. Research Findings

### 3.1 面板定位与渲染架构

面板使用 `createPortal` 渲染到 `document.body`，通过绝对定位实现浮动效果。

**定位计算逻辑**：

```typescript
const rect = trigger.getBoundingClientRect()
const panelWidth = 320
const gap = 8
const left = Math.min(rect.left, window.innerWidth - panelWidth - 16)

setPanelPos({
  top: rect.bottom + gap,
  left: Math.max(8, left),
  width: panelWidth,
})
```

**入场动画**：通过 `isVisible` 状态控制 CSS transition：

- 弹出时：`opacity: 0 → 1`, `scale: 0.95 → 1`
- 关闭时：`opacity: 1 → 0`, `scale: 1 → 0.95`
- 动画时长 150ms，缓动 `ease-out`
- 使用 `requestAnimationFrame` 确保初始状态渲染后再触发动画

**关闭防抖**：`isClosingRef` 标记防止动画进行中重复触发关闭。

### 3.2 独立 SDK 客户端架构

面板需要同时查询多个服务器的数据，每个服务器可能有不同的 URL 和认证信息。

**客户端缓存策略**：

```typescript
const clientCache = new Map<string, OpencodeClient>()

function getServerClient(server: ServerConfig): OpencodeClient {
  const authPart = server.auth?.password ? `${server.auth.username}:${server.auth.password}` : 'no-auth'
  const cacheKey = `${server.url}|${authPart}`

  if (clientCache.has(cacheKey)) {
    return clientCache.get(cacheKey)!
  }

  const headers: Record<string, string> = {}
  if (server.auth?.password) {
    headers['Authorization'] = makeBasicAuthHeader(server.auth)
  }

  const client = createOpencodeClient({ baseUrl: server.url, headers })
  clientCache.set(cacheKey, client)
  return client
}
```

**设计要点**：

- 缓存 key 包含 URL 和认证信息，密码变化时自动创建新客户端
- 认证信息通过 `makeBasicAuthHeader` 生成 Basic Auth 头
- 无密码的服务器使用统一的 `no-auth` key，共享同一个匿名客户端

### 3.3 数据获取流程

面板打开时的数据获取流程：

```
1. 获取所有服务器配置和当前健康状态
2. 为每个服务器创建/复用 SDK 客户端
3. 并行执行（Promise.allSettled）：
   a. 调用 client.project.list() 获取项目列表
   b. 对每个项目，并行调用：
      - client.session.list() 获取会话列表
      - client.session.status() 获取会话状态
4. 合并数据，更新对应服务器节点状态
5. 单个服务器失败不影响其他服务器
```

**降级策略**：

- `project.list` 失败 → 使用兜底 "全部项目" 分组
- `session.status` 失败 → 状态标记为 `unknown`，不影响列表展示
- 整个服务器请求失败 → 节点显示错误信息

**取消机制**：`useEffect` cleanup 中设置 `cancelled = true`，避免已卸载组件的状态更新。

### 3.4 会话状态映射

SDK 的 `SessionStatus` 类型定义：

```typescript
type SessionStatus =
  | { type: 'idle' }
  | { type: 'busy' }
  | { type: 'retry'; attempt: number; message: string; next: number }
```

面板内部简化为：

```typescript
function extractStatusType(status: SessionStatus | undefined): 'idle' | 'busy' | 'paused' | 'unknown' {
  if (!status) return 'unknown'
  return status.type as SessionInfo['status']
}
```

**映射关系**：

- `idle` → `idle`（空闲，灰色圆点）
- `busy` → `busy`（忙碌，绿色脉冲圆点）
- `retry` → `unknown`（重试中，深灰色圆点）
- 无响应 → `unknown`（未知，深灰色圆点）

### 3.5 关闭行为

三种关闭方式：

| 方式         | 实现方式                                          |
| ------------ | ------------------------------------------------- |
| 关闭按钮     | `onClick={handleClose}`                           |
| 点击外部区域 | `document.addEventListener('mousedown', handler)` |
| ESC 键       | `document.addEventListener('keydown', handler)`   |

**外部点击检测逻辑**：

```typescript
const handleClickOutside = (e: MouseEvent) => {
  if (isClosingRef.current) return
  const target = e.target as Node
  if (triggerRef.current?.contains(target)) return // 排除触发按钮
  if (panelRef.current?.contains(target)) return // 排除面板自身
  handleClose()
}
```

### 3.6 会话切换处理

在 `SidePanel.tsx` 中的处理逻辑：

```typescript
const handleQuickPanelSelectSession = useCallback(
  (session: { id: string; title: string; directory?: string }, serverId: string) => {
    const activeServer = serverStore.getActiveServer()
    // 跨服务器切换
    if (activeServer?.id !== serverId) {
      messageStore.clearSession(selectedSessionId ?? '')
      serverStore.setActiveServer(serverId)
    }
    // 添加工作目录
    if (session.directory) {
      addDirectory(session.directory)
    }
    // 获取会话详情并导航
    getSession(session.id, session.directory)
      .then(apiSession => {
        onSelectSession(apiSession)
      })
      .catch(() => {})
  },
  [selectedSessionId, addDirectory, onSelectSession],
)
```

---

## 4. Data Model

### 4.1 面板内部数据结构

```typescript
/** 面板内展示的 session 信息（从 SDK 数据简化而来） */
interface SessionInfo {
  id: string
  title: string
  directory?: string
  status: 'idle' | 'busy' | 'paused' | 'unknown'
}

/** 单个服务器下的工程分组 */
interface ProjectGroup {
  id: string
  name: string
  directory?: string
  sessions: SessionInfo[]
}

/** 面板中每个服务器节点的数据结构 */
interface ServerNode {
  server: ServerConfig // 服务器配置（来自 serverStore）
  health: ServerHealth | null // 健康状态（来自 serverStore）
  projects: ProjectGroup[] // 下属工程列表
  isLoading: boolean // 是否正在加载数据
  error: string | null // 加载错误信息
}
```

### 4.2 组件 Props 接口

```typescript
interface ServerQuickPanelProps {
  /** 触发按钮的 ref，用于定位面板弹出位置 */
  triggerRef: React.RefObject<HTMLElement | null>
  /** 关闭面板的回调 */
  onClose: () => void
  /** 选中 session 时的回调，传入 session 信息和所属 serverId */
  onSelectSession: (session: SessionInfo, serverId: string) => void
}
```

### 4.3 状态流转图

**面板生命周期**：

```
[未挂载] → 点击触发按钮 → [挂载 + 加载中] → 数据加载完成 → [展示中]
                                                    ↓
                                          点击会话 → [关闭动画] → [卸载]
                                          点击外部 → [关闭动画] → [卸载]
                                          按 ESC   → [关闭动画] → [卸载]
```

**服务器节点状态**：

```
[初始化: isLoading=true] → 数据获取成功 → [展示数据: isLoading=false, error=null]
                        → 数据获取失败 → [显示错误: isLoading=false, error=消息]
```

---

## 5. Interface Contracts

### 5.1 ServerQuickPanel 组件契约

```typescript
function ServerQuickPanel({ triggerRef, onClose, onSelectSession }: ServerQuickPanelProps): React.ReactPortal
```

**行为约定**：

- 挂载时自动获取所有服务器数据
- 关闭时触发 `onClose` 回调（含 150ms 动画延迟）
- 选中会话时先调用 `onSelectSession`，再调用 `onClose`

### 5.2 内嵌子组件

| 组件              | Props                                                                                            | 职责                     |
| ----------------- | ------------------------------------------------------------------------------------------------ | ------------------------ |
| `ServerTreeItem`  | `node`, `isExpanded`, `expandedProjects`, `onToggleServer`, `onToggleProject`, `onSelectSession` | 服务器节点展示与展开控制 |
| `ProjectTreeItem` | `project`, `serverId`, `isExpanded`, `onToggle`, `onSelectSession`                               | 工程节点展示与展开控制   |
| `SessionItem`     | `session`, `serverId`, `onSelect`                                                                | 会话条目展示与点击选择   |

### 5.3 消费的 Store 接口

| Store          | 方法                        | 用途                             |
| -------------- | --------------------------- | -------------------------------- |
| `serverStore`  | `getServers()`              | 获取所有服务器配置               |
| `serverStore`  | `getAllHealth()`            | 获取所有服务器健康状态           |
| `serverStore`  | `getActiveServer()`         | 获取当前活动服务器（自动展开用） |
| `serverStore`  | `makeBasicAuthHeader(auth)` | 生成 Basic Auth 请求头           |
| `messageStore` | `clearSession(sessionId)`   | 切换服务器时清理消息             |

### 5.4 消费的 API 接口

| API 方法                  | 用途         | 失败处理            |
| ------------------------- | ------------ | ------------------- |
| `client.project.list()`   | 获取项目列表 | 使用兜底 "全部项目" |
| `client.session.list()`   | 获取会话列表 | 该工程会话列表为空  |
| `client.session.status()` | 获取会话状态 | 状态标记为 unknown  |

---

## 6. Implementation Strategy

### 6.1 组件层次

```
SidePanel
├── <button ref={serverPanelTriggerRef} onClick={() => setServerPanelOpen(true)}>
│       OpenCode
│   </button>
│
└── {serverPanelOpen && (
      <ServerQuickPanel
        triggerRef={serverPanelTriggerRef}
        onClose={() => setServerPanelOpen(false)}
        onSelectSession={handleQuickPanelSelectSession}
      />
    )}

ServerQuickPanel (Portal → document.body)
├── Header (标题 + 概览 + 关闭按钮)
└── Scrollable Content
    ├── ServerTreeItem × N
    │   ├── Server Header (名称 + URL + 状态 + 计数)
    │   └── [expanded]
    │       ├── Loading State
    │       ├── Error State
    │       ├── Empty State
    │       └── ProjectTreeItem × M
    │           ├── Project Header (名称 + 计数)
    │           └── [expanded]
    │               └── SessionItem × K
    │                   ├── Status Dot
    │                   ├── Icon
    │                   └── Title
```

### 6.2 数据流

```
用户点击 OpenCode 标识
  ↓
SidePanel.setServerPanelOpen(true)
  ↓
ServerQuickPanel 挂载 (createPortal)
  ↓
useEffect: 获取 serverStore.getServers() + getAllHealth()
  ↓
初始化 ServerNode[] (isLoading=true)
  ↓
fetchAll(): Promise.allSettled(servers.map(...))
  ├── getServerClient(server) → 创建/复用 SDK 客户端
  ├── client.project.list() → 获取项目列表
  ├── Promise.allSettled(projects.map(...))
  │   ├── client.session.list() → 获取会话列表
  │   └── client.session.status() → 获取会话状态
  └── setServerNodes(更新数据)
  ↓
用户点击会话
  ↓
onSelectSession(session, serverId)
  ↓
SidePanel.handleQuickPanelSelectSession
  ├── 跨服务器？→ messageStore.clearSession + serverStore.setActiveServer
  ├── 有 directory？→ addDirectory
  └── getSession → onSelectSession
  ↓
handleClose → 退出动画 → onClose → 卸载
```

### 6.3 样式策略

面板使用 Tailwind CSS 工具类，无自定义 CSS：

| 视觉元素       | Tailwind 类名                                                                              |
| -------------- | ------------------------------------------------------------------------------------------ |
| 面板容器       | `fixed z-[9999] rounded-xl border border-border-200/60 glass-alt shadow-lg`                |
| 入场动画       | `transition-all duration-150 ease-out` + 条件 `opacity/scale`                              |
| 头部           | `flex items-center justify-between px-3 py-2.5 border-b border-border-200/40 bg-bg-100/60` |
| 服务器行       | `w-full flex items-center gap-2 px-3 py-2 hover:bg-bg-200/40`                              |
| 工程行         | `w-full flex items-center gap-2 px-3 py-1.5 pl-8 hover:bg-bg-200/40`                       |
| 会话行         | `w-full flex items-center gap-2 px-3 py-1.5 pl-12 hover:bg-bg-200/50`                      |
| 忙碌状态点     | `bg-success-100 animate-pulse`                                                             |
| 当前服务器标签 | `text-[9px] font-medium text-accent-main-100 bg-accent-main-100/10 rounded-full`           |

---

## 7. Error Handling

### 7.1 错误场景与处理

| 场景                   | 处理方式                                            |
| ---------------------- | --------------------------------------------------- |
| 服务器连接失败         | 节点显示错误信息，不影响其他服务器                  |
| 项目 API 不可用        | 使用兜底 "全部项目" 分组                            |
| 会话状态 API 返回异常  | 状态标记为 `unknown`，不影响列表展示                |
| 会话列表 API 失败      | 该工程下显示空列表                                  |
| 面板关闭时请求仍在进行 | `cancelled` 标志阻止 setState                       |
| 触发按钮 ref 为空      | 定位 useEffect 提前 return                          |
| 关闭动画中重复触发关闭 | `isClosingRef` 标志阻止重复执行                     |
| 面板弹出位置超出视口   | `Math.min(rect.left, window.innerWidth - 336)` 限制 |

### 7.2 边界条件

- **无服务器配置**：面板显示空列表（header 仍可见）
- **服务器无项目**：展开后显示"暂无项目"提示
- **工程无会话**：Chevron 图标不可见，按钮 disabled
- **窗口尺寸极小**：面板最大高度取 `min(520, window.innerHeight - top - 16)`

---

## 8. Testing Considerations

### 8.1 组件测试覆盖点

| 组件               | 测试场景                                       |
| ------------------ | ---------------------------------------------- |
| `ServerQuickPanel` | 弹出定位、数据加载、关闭行为、ESC 键、外部点击 |
| `ServerTreeItem`   | 展开/收起、健康状态图标、当前服务器标识        |
| `ProjectTreeItem`  | 展开/收起、忙碌计数、无会话时 disabled         |
| `SessionItem`      | 状态点颜色、点击选择、hover 效果               |
| `SidePanel` 集成   | 触发按钮点击、会话切换、跨服务器切换           |

### 8.2 集成测试场景

| 场景             | 验证点                                      |
| ---------------- | ------------------------------------------- |
| 跨服务器切换     | 消息清理 + 服务器切换 + 会话打开 + 面板关闭 |
| 同服务器切换     | 直接打开会话，不清理消息                    |
| 多服务器并发加载 | 所有服务器独立加载，失败互不影响            |
| 面板快速开关     | 快速点击打开/关闭，不出现状态混乱           |

### 8.3 手动测试清单

- [ ] 面板在触发按钮正下方弹出，间距 8px
- [ ] 面板不超出视口右边界
- [ ] 当前活动服务器默认展开并高亮
- [ ] 忙碌会话显示绿色脉冲并排在前面
- [ ] 点击面板外部区域关闭
- [ ] 按 ESC 键关闭
- [ ] 跨服务器切换后会话正确打开
- [ ] 关闭动画流畅无闪烁

---

## 9. 依赖关系总结

### 9.1 外部依赖

| 依赖                  | 用途               | 必需性 |
| --------------------- | ------------------ | ------ |
| `@opencode-ai/sdk/v2` | 项目/会话 API 调用 | 必需   |
| i18next               | 面板文本国际化     | 必需   |

### 9.2 内部依赖

| 依赖模块           | 使用内容                                           |
| ------------------ | -------------------------------------------------- |
| `serverStore`      | 服务器配置、健康状态、Basic Auth 生成              |
| `messageStore`     | 切换服务器时清理会话消息                           |
| `getSession` (API) | 获取会话详细信息                                   |
| `Icons`            | Globe、Folder、ChevronDown、Wifi、MessageSquare 等 |
| `getDirectoryName` | 从工作目录路径提取目录名                           |
| `formatPathForApi` | 路径格式化为 API 兼容格式                          |

### 9.3 被依赖模块

| 模块        | 使用内容                         |
| ----------- | -------------------------------- |
| `SidePanel` | 触发按钮、面板状态管理、会话切换 |

---

## 10. 风险区域

### 10.1 多服务器并发请求性能（低风险）

每个服务器需要 1 + 2N 次 API 调用（1 次项目列表 + N 个工程 × 2 次会话列表/状态）。当服务器数量多、工程数量多时，总请求数可能较大。

**缓解**：

- 所有请求使用 `Promise.allSettled` 并行执行
- 会话列表限制 `limit: 50`
- 单个服务器失败不影响整体

### 10.2 SDK 客户端缓存与认证过期（低风险）

客户端按 "URL + 认证信息" 缓存，如果服务器密码在服务端被修改，客户端仍使用旧认证。

**缓解**：

- 健康状态检测会暴露认证失败（`unauthorized` 状态）
- 用户在设置面板修改服务器配置后，缓存 key 变化，自动创建新客户端

### 10.3 面板定位在动态布局下的准确性（低风险）

触发按钮位置可能因侧边栏折叠/展开而变化。

**缓解**：

- 定位计算在组件挂载时执行一次，面板打开期间触发按钮位置不变
- 如需响应窗口 resize，可在后续迭代中添加 resize 监听

---

_生成时间: 2026-04-13_
_模块版本: OpenCodeUI v0.4.8_
