# 状态管理模块规范

> 模块编号：010-state-management
> 状态：已实现
> 最后更新：2026-04-12

---

## 1. 模块概述

### 1.1 目的

状态管理模块是应用的核心数据枢纽，采用自定义发布.订阅模式管理所有业务状态。通过 16 个职责单一的 Store 实例，将应用状态按领域拆分，每个 Store 独立维护自身数据、持久化策略和变更通知机制。

### 1.2 解决的问题

- **全局状态同步**：所有 UI 组件通过统一的订阅机制获取最新状态，避免组件间直接通信
- **SSE 事件驱动更新**：后端推送的事件直接修改对应 Store，变更自动广播到所有订阅者
- **状态持久化**：用户偏好、布局配置等数据在页面刷新后自动恢复
- **多会话隔离**：消息状态按 sessionId 独立存储，切换会话时互不干扰
- **撤销.重做**：消息操作支持多步撤销与重做，通过 revertState 机制实现
- **分屏多会话**：二叉树结构的分屏布局支持同时展示多个会话

### 1.3 范围

本模块涵盖：

- 16 个 Store 的定义、实例化与导出
- 发布.订阅模式的核心接口（subscribe、getSnapshot、emitChange）
- 基于快照的 UI 更新机制
- 本地存储持久化策略（localStorage、IndexedDB、sessionStorage）
- 撤销.重做机制
- 多会话状态管理
- 分屏布局的树形状态管理

本模块不涵盖：

- SSE 连接本身的建立与维护（由 API 通信层负责）
- UI 组件的具体渲染逻辑
- 后端 API 的具体实现

---

## 2. 用户故事

| 编号  | 用户故事                                                                                      | 优先级 |
| ----- | --------------------------------------------------------------------------------------------- | ------ |
| US-01 | 作为用户，我希望在一个组件中修改状态后，所有依赖该状态的组件自动更新，无需手动刷新            | P0     |
| US-02 | 作为用户，我希望后端推送的 SSE 事件能实时反映到界面上，包括消息流式输出、权限请求、待办更新等 | P0     |
| US-03 | 作为用户，我希望刷新页面后，我的主题偏好、快捷键设置、面板布局等个人配置保持不变              | P0     |
| US-04 | 作为用户，我希望在发送消息后能够撤销操作，恢复到发送前的状态                                  | P1     |
| US-05 | 作为用户，我希望能够同时查看多个会话的对话内容，通过分屏自由切换                              | P1     |
| US-06 | 作为用户，我希望切换后端服务器时，所有内存状态自动清空，避免旧服务器数据污染新服务器          | P1     |
| US-07 | 作为多窗口用户，我希望每个浏览器窗口独立记住当前选中的服务器，互不干扰                        | P2     |
| US-08 | 作为用户，我希望在忙碌的会话完成时收到通知提示音，且可以为不同类型的事件设置不同的音效        | P2     |
| US-09 | 作为用户，我希望在会话忙碌时继续输入消息，消息自动排队等待发送                                | P2     |
| US-10 | 作为用户，我希望子会话的权限请求能够冒泡到父会话界面，方便统一管理                            | P2     |

---

## 3. 功能需求

### FR-01：发布.订阅模式

**描述**：每个 Store 实现标准的发布.订阅接口，包含 subscribe、getSnapshot 和内部变更通知机制。

**要求**：

- subscribe 方法接受回调函数，返回取消订阅函数
- getSnapshot 方法返回当前状态的不可变快照
- 状态变更后通过 emitChange 通知所有订阅者
- 订阅者回调不接收参数，通过 getSnapshot 自行获取最新状态

**验收标准**：

- 给定已订阅的回调，状态变更后回调被调用
- 给定取消订阅后的回调，状态变更后回调不再被调用
- getSnapshot 返回的对象引用在状态未变时保持稳定

### FR-02：Store 目录

**描述**：系统包含 16 个专用 Store，每个 Store 负责一个独立的状态领域。

**要求**：

- 每个 Store 为单例模式，全局唯一实例
- Store 之间通过事件或回调松散耦合，不直接相互引用
- 所有 Store 通过统一入口文件导出

**验收标准**：

- 给定任意 Store，全局仅存在一个实例
- Store 之间无循环依赖
- 入口文件导出所有 Store 及其关联类型

### FR-03：SSE 事件集成

**描述**：SSE 事件处理器直接调用 Store 的更新方法，实现事件驱动的状态变更。

**要求**：

- 消息类事件（message.updated、message.part.delta 等）更新 messageStore
- 会话状态事件（session.status、session.idle 等）更新 activeSessionStore
- 待办事件（todo.updated）更新 todoStore
- 权限.问答事件更新 activeSessionStore 和 notificationStore
- 子会话事件（session.created）更新 childSessionStore

**验收标准**：

- 给定 SSE 消息事件，messageStore 中对应会话的消息列表自动更新
- 给定 SSE 会话状态事件，activeSessionStore 的会话状态同步变更
- 给定 SSE 待办事件，todoStore 中对应会话的待办列表更新

### FR-04：状态持久化

**描述**：部分 Store 的状态在页面刷新后自动恢复，持久化策略因 Store 而异。

**要求**：

- 用户偏好类 Store（themeStore、keybindingStore、soundStore）持久化到 localStorage
- 服务器配置（serverStore）持久化到 localStorage，活动服务器同时写入 sessionStorage 实现窗口隔离
- 布局状态（layoutStore）持久化面板配置和面板尺寸
- 通知历史（notificationStore）持久化到 localStorage，最多保留 50 条
- 自定义音频（soundStore）存储到 IndexedDB，单文件上限 2MB
- 服务配置（serviceStore）持久化自动启动、二进制路径、环境变量
- 消息状态（messageStore）不持久化，完全由服务端数据驱动

**验收标准**：

- 给定已修改的主题设置，刷新页面后设置保持不变
- 给定已配置的快捷键，刷新页面后快捷键保持不变
- 给定已添加的服务器，刷新页面后服务器列表保持不变
- 切换浏览器窗口时，每个窗口独立记住活动服务器

### FR-05：基于快照的 UI 更新

**描述**：UI 组件通过订阅 Store 的快照获取状态，而非直接访问内部数据。

**要求**：

- Store 提供 getSnapshot 方法返回不可变状态快照
- 快照在状态未变更时保持引用稳定，避免不必要的 UI 重渲染
- 支持选择器模式，组件可订阅快照的特定字段
- 快照缓存机制防止无限循环

**验收标准**：

- 给定状态未变更，连续两次 getSnapshot 返回同一对象引用
- 给定状态变更，getSnapshot 返回新对象引用
- 组件仅订阅特定字段时，其他字段变更不触发组件重渲染

### FR-06：撤销.重做

**描述**：消息 Store 支持多步撤销与重做操作。

**要求**：

- 通过 revertState 记录撤销点，包含 messageId 和历史栈
- 撤销时隐藏撤销点之后的所有消息
- 重做时逐步恢复被隐藏的消息
- 流式输出期间禁用重做

**验收标准**：

- 给定已发送的消息，执行撤销后消息从视图中隐藏
- 给定已撤销的状态，执行重做后消息重新显示
- 给定流式输出中，重做操作不可用

### FR-07：多会话状态管理

**描述**：messageStore 按 sessionId 独立管理每个会话的消息状态。

**要求**：

- 每个会话的消息独立存储在内存 Map 中
- 内存中最多缓存 10 个会话，超出时按最近访问时间淘汰
- 被分屏保护的会话跳过淘汰机制
- 支持按 sessionId 查询消息、流式状态、加载状态等

**验收标准**：

- 给定 10 个已加载的会话，访问第 11 个会话时最久未用的会话被清除
- 给定被分屏保护的会话，即使超出缓存上限也不被淘汰
- 切换会话时，新会话的消息立即显示，旧会话状态保留

### FR-08：分屏布局管理

**描述**：paneLayoutStore 使用二叉树结构管理分屏布局，支持水平.垂直分割。

**要求**：

- 树节点分为叶子节点（PaneLeaf）和分割节点（PaneSplit）
- 叶子节点持有 sessionId，分割节点持有分割方向和比例
- 支持分割、关闭、交换、全屏、聚焦等操作
- 分割比例限制在 0.15 到 0.85 之间

**验收标准**：

- 给定单个 pane，执行水平分割后产生两个并排 pane
- 给定两个 pane，关闭其中一个后恢复为单个 pane
- 给定多个 pane，交换操作互换两个 pane 的会话

### FR-09：RAF 批量通知

**描述**：messageStore 使用 requestAnimationFrame 批量通知订阅者，避免高频事件导致的性能问题。

**要求**：

- 同一帧内的多次状态变更合并为一次通知
- 仅对被修改过的消息生成新的不可变引用
- 非浏览器环境下降级为立即通知

**验收标准**：

- 给定一帧内 100 次 delta 更新，订阅者仅被通知一次
- 给定未修改的消息，其对象引用保持不变

---

## 4. Store 目录

| 编号 | Store 名称          | 领域         | 主要职责                                                                 | 持久化                        |
| ---- | ------------------- | ------------ | ------------------------------------------------------------------------ | ----------------------------- |
| S-01 | messageStore        | 消息与会话   | 按 sessionId 管理消息列表、流式状态、撤销.重做、加载状态                 | 无（服务端驱动）              |
| S-02 | activeSessionStore  | 会话活跃状态 | 追踪所有会话的忙.闲状态、等待中的权限.问答请求、会话元信息               | 无                            |
| S-03 | childSessionStore   | 子会话关系   | 追踪父子会话关系、子会话状态、权限冒泡                                   | 无                            |
| S-04 | layoutStore         | UI 布局      | 侧边栏展开状态、右侧面板.底部面板的开关与尺寸、面板标签页管理、终端标签  | localStorage                  |
| S-05 | paneLayoutStore     | 分屏布局     | 二叉树分屏结构、pane 聚焦、分割比例、全屏模式                            | 无                            |
| S-06 | paneControllerStore | 分屏控制器   | 为每个 pane 注册操作控制器（新建会话、归档、切换 Agent 等）              | 无                            |
| S-07 | serverStore         | 后端服务器   | 多服务器配置管理、活动服务器切换、健康检查、认证信息                     | localStorage + sessionStorage |
| S-08 | autoApproveStore    | 自动批准     | 按 sessionId 管理自动批准规则、Full Auto 模式（off.session.global）      | 功能开关持久化，规则仅内存    |
| S-09 | keybindingStore     | 快捷键       | 22 个动作的可配置快捷键、解析、格式化、冲突检测                          | localStorage                  |
| S-10 | themeStore          | 主题与外观   | 主题预设、明暗模式、自定义 CSS、宽模式、Diff 风格、沉浸模式等 15+ 配置项 | localStorage                  |
| S-11 | todoStore           | 待办事项     | 按 sessionId 管理待办列表、统计信息、当前任务                            | 无                            |
| S-12 | notificationStore   | 通知与 Toast | 通知历史（持久化）、Toast 弹窗管理、已读.未读状态                        | localStorage                  |
| S-13 | serviceStore        | 服务进程     | Tauri 桌面端 opencode 进程的自动启动、二进制路径、环境变量、运行状态     | localStorage                  |
| S-14 | soundStore          | 提示音       | 声音总开关、音量、每类事件的音效选择、自定义音频上传                     | localStorage + IndexedDB      |
| S-15 | followupQueueStore  | 后续消息队列 | 忙碌会话的消息排队、发送状态追踪、失败重试                               | 无                            |
| S-16 | changeScopeStore    | 变更范围     | 按 sessionId 管理变更范围模式（git.branch.session.turn）                 | 无                            |

---

## 5. 架构设计

### 5.1 核心模式

所有 Store 遵循统一的发布.订阅模式：

```
┌─────────────┐     subscribe()     ┌──────────────┐
│   Store     │ ◄────────────────── │  Subscriber  │
│             │ ──────────────────► │  (UI 组件)    │
│  - state    │    notify()         │              │
│  - listeners│                     │  getSnapshot │
│  - notify() │                     │    → render  │
└─────────────┘                     └──────────────┘
```

**接口约定**：

- `subscribe(listener: () => void): () => void` — 注册监听器，返回取消订阅函数
- `getSnapshot(): State` — 返回当前状态的不可变快照
- 内部 `notify()` — 遍历所有监听器并调用

### 5.2 快照缓存机制

为避免 `getSnapshot` 每次返回新对象导致不必要的重渲染，Store 采用快照缓存策略：

1. 状态变更时生成新快照并缓存
2. 状态未变更时返回缓存的快照（引用稳定）
3. 订阅者回调触发时清除缓存，下次调用 `getSnapshot` 时重新生成

部分 Store（如 messageStore）采用更精细的缓存策略：

- `cachedSnapshot`：全局消息快照，状态变更时清除
- `sessionSnapshots`：按 sessionId 缓存的会话快照，状态变更时全部清除
- `childSessionsCache` / `sessionFamilyCache`：子会话缓存，状态变更时清除

### 5.3 RAF 批量通知

messageStore 针对高频 SSE 事件（如 message.part.delta）采用 RAF 批量通知：

```
SSE delta #1 → 标记 dirty → 请求 RAF
SSE delta #2 → 标记 dirty → RAF 已请求，跳过
SSE delta #3 → 标记 dirty → RAF 已请求，跳过
     ↓
RAF 回调触发 → flushDirtyMessages() → 一次性生成不可变快照 → 通知所有订阅者
```

这确保一帧内无论收到多少次 delta 更新，UI 仅重渲染一次。

### 5.4 内存淘汰策略

messageStore 维护最多 10 个会话的内存缓存：

- 每次访问会话时记录时间戳
- 新增会话时若超过上限，淘汰最久未访问的会话
- 被分屏 pane 保护的会话跳过淘汰

### 5.5 持久化分层

| 存储方式       | 用途                       | 使用 Store                                                                                                                                                         |
| -------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| localStorage   | 用户偏好、配置、布局       | layoutStore, keybindingStore, themeStore, notificationStore, serviceStore, autoApproveStore(开关), soundStore(配置), serverStore(服务器列表)                       |
| sessionStorage | 窗口级隔离的活动服务器     | serverStore(活动服务器 ID)                                                                                                                                         |
| IndexedDB      | 大二进制数据（自定义音频） | soundStore                                                                                                                                                         |
| 内存           | 运行时状态，刷新即失       | messageStore, activeSessionStore, childSessionStore, paneLayoutStore, paneControllerStore, todoStore, followupQueueStore, changeScopeStore, autoApproveStore(规则) |

### 5.6 服务器切换时的状态清理

切换后端服务器时，以下 Store 需要清空内存状态：

- `autoApproveStore`：清空所有规则，关闭 Full Auto
- `todoStore`：清空所有待办
- `childSessionStore`：清空所有子会话记录
- `changeScopeStore`：清空所有变更范围设置
- `followupQueueStore`：清空所有排队消息
- `notificationStore`：清空通知历史
- `activeSessionStore`：重置会话状态

---

## 6. 数据流

### 6.1 SSE 事件 → Store → UI

```
后端 SSE 事件
    │
    ▼
事件分发器 (useGlobalEvents)
    │
    ├── message.updated ──────► messageStore.updateMessage()
    │                                │
    │                                ▼
    │                           notify() ──► 订阅者回调
    │                                            │
    │                                            ▼
    │                                      getSnapshot()
    │                                            │
    │                                            ▼
    │                                      UI 重渲染
    │
    ├── session.status ───────► activeSessionStore.updateStatus()
    │                                │
    │                                ▼
    │                           notify() ──► 订阅者回调
    │
    ├── todo.updated ─────────► todoStore.setTodos()
    │
    ├── permission.asked ─────► activeSessionStore.addPendingRequest()
    │                         ──► notificationStore.push()
    │
    └── session.created ──────► childSessionStore.registerChildSession()
```

### 6.2 用户操作 → Store → UI

```
用户操作（点击、输入、拖拽等）
    │
    ▼
UI 组件调用 Store 方法
    │
    ├── layoutStore.setSidebarExpanded(true)
    ├── themeStore.setPreset('claude')
    ├── keybindingStore.setKeybinding('sendMessage', 'Ctrl+Enter')
    ├── paneLayoutStore.splitPane(paneId, 'horizontal')
    └── messageStore.undo(sessionId)
         │
         ▼
    Store 更新内部状态
         │
         ▼
    notify() 通知所有订阅者
         │
         ▼
    UI 组件通过 getSnapshot() 获取新状态并重渲染
```

### 6.3 持久化读写时机

```
Store 初始化（构造函数）
    │
    ▼
从 localStorage / IndexedDB 读取已保存状态
    │
    ▼
合并到初始状态
    │
    ... 运行时状态变更 ...
    │
    ▼
notify() 调用时同步写入持久化存储
    │
    ├── layoutStore.notify() → persistPanelLayout()
    ├── serverStore.notify() → saveToStorage()
    ├── notificationStore.notify() → persist()
    └── soundStore.notify() → persist()
```

---

## 7. 验收场景

### 场景 1：消息状态跨组件同步

**前置条件**：messageStore 已初始化，存在活跃会话

**步骤**：

1. 组件 A 订阅 messageStore 的消息列表
2. 组件 B 订阅 messageStore 的流式状态
3. SSE 推送一条 message.updated 事件

**预期结果**：

- 组件 A 收到新消息并渲染
- 组件 B 获取到最新的流式状态
- 两个组件在同一帧内完成更新

### 场景 2：页面刷新后状态恢复

**前置条件**：用户已修改主题预设、快捷键、面板布局

**步骤**：

1. 将主题从默认切换为 claude
2. 修改发送消息快捷键为 Ctrl+Shift+Enter
3. 展开右侧面板并调整宽度为 600px
4. 刷新页面

**预期结果**：

- 主题预设为 claude
- 发送消息快捷键为 Ctrl+Shift+Enter
- 右侧面板展开，宽度为 600px

### 场景 3：撤销.重做操作

**前置条件**：已发送至少两条消息，当前不在流式输出中

**步骤**：

1. 执行撤销操作
2. 检查消息列表
3. 执行重做操作
4. 检查消息列表

**预期结果**：

- 步骤 2：最后一条消息从视图中隐藏
- 步骤 4：被隐藏的消息重新显示

### 场景 4：多会话内存淘汰

**前置条件**：已加载 10 个会话的消息

**步骤**：

1. 访问第 11 个会话
2. 检查内存中的会话数量
3. 确认最久未访问的会话已被清除

**预期结果**：

- 内存中始终保持最多 10 个会话
- 最久未访问的会话被清除
- 被分屏保护的会话不被清除

### 场景 5：分屏布局操作

**前置条件**：当前为单 pane 模式

**步骤**：

1. 执行水平分割
2. 为左侧 pane 分配会话 A
3. 为右侧 pane 分配会话 B
4. 调整分割比例为 0.3
5. 关闭右侧 pane

**预期结果**：

- 步骤 1：界面分为左右两个 pane
- 步骤 2-3：左侧显示会话 A，右侧显示会话 B
- 步骤 4：左侧占 30% 宽度，右侧占 70%
- 步骤 5：恢复为单 pane 模式，显示会话 A

### 场景 6：服务器切换时状态清理

**前置条件**：已连接服务器 A，存在待办、通知、自动批准规则

**步骤**：

1. 将活动服务器从 A 切换到 B
2. 检查 todoStore、notificationStore、autoApproveStore 的状态

**预期结果**：

- todoStore 数据清空
- notificationStore 通知历史清空
- autoApproveStore 规则清空，Full Auto 关闭
- 服务器列表和配置保持不变

### 场景 7：RAF 批量通知性能

**前置条件**：会话正在流式输出，每秒产生大量 delta 事件

**步骤**：

1. 观察一帧内收到的 delta 事件数量
2. 观察订阅者回调被调用的次数

**预期结果**：

- 无论一帧内收到多少次 delta，订阅者仅被通知一次
- UI 帧率保持稳定，无明显卡顿

### 场景 8：通知 Toast 管理

**前置条件**：Toast 开关已打开

**步骤**：

1. 推送第一条通知
2. 推送第二条通知
3. 推送第三条通知
4. 推送第四条通知
5. 等待 8 秒

**预期结果**：

- 步骤 1-3：Toast 依次显示，最多同时显示 3 个
- 步骤 4：最旧的 Toast 被移除，新 Toast 加入
- 步骤 5：所有 Toast 自动消失（悬停时暂停计时）

---

## 8. 依赖关系

### 8.1 外部依赖

| 依赖                  | 用途                                    | 必需性         |
| --------------------- | --------------------------------------- | -------------- |
| 浏览器 API            | localStorage、sessionStorage、IndexedDB | 必需           |
| requestAnimationFrame | RAF 批量通知                            | 浏览器环境必需 |

### 8.2 内部依赖

| 依赖模块     | 被哪些 Store 使用                        |
| ------------ | ---------------------------------------- |
| API 通信层   | 所有 Store 通过 SSE 事件接收数据更新     |
| 服务器状态   | messageStore、todoStore 等按服务器隔离   |
| 类型定义     | 所有 Store 共享消息、会话、事件等类型    |
| 日志工具     | messageStore 用于调试日志                |
| 消息转换工具 | messageStore 用于 API 消息到 UI 消息转换 |
| 国际化模块   | childSessionStore 用于默认文本           |
| 主题预设     | themeStore 用于内置主题                  |
| 音效预设     | soundStore 用于内置音效                  |

### 8.3 被依赖模块

| 依赖本模块的模块 | 使用的 Store                                                                         |
| ---------------- | ------------------------------------------------------------------------------------ |
| 聊天界面         | messageStore, paneLayoutStore, paneControllerStore                                   |
| 会话管理         | activeSessionStore, childSessionStore, messageStore                                  |
| 设置面板         | themeStore, keybindingStore, serverStore, soundStore, serviceStore, autoApproveStore |
| 终端系统         | layoutStore                                                                          |
| 文件浏览器       | layoutStore                                                                          |
| 通知系统         | notificationStore, soundStore                                                        |
| 权限管理         | autoApproveStore, activeSessionStore                                                 |
| 待办面板         | todoStore                                                                            |
| 消息输入         | followupQueueStore                                                                   |
| 变更范围选择     | changeScopeStore                                                                     |

---

## 9. 架构决策记录

### ADR-001：为什么采用自定义发布.订阅而非第三方状态库

**决策**：使用自定义的发布.订阅模式，不引入 Redux、Zustand、Jotai 等第三方状态管理库。

**理由**：

- 应用的状态更新主要由 SSE 事件驱动，模式简单直接
- 每个 Store 职责单一，不需要复杂的状态组合和派生
- 避免引入额外依赖，保持构建产物体积
- 发布.订阅模式天然支持多订阅者共享，与 SSE 单例连接模式匹配

### ADR-002：为什么 messageStore 使用 RAF 批量通知

**决策**：messageStore 的 notify 方法通过 requestAnimationFrame 节流，同一帧内多次变更合并为一次通知。

**理由**：

- SSE 的 message.part.delta 事件频率极高（每个 token 一次）
- 每次 delta 都触发 UI 重渲染会导致严重性能问题
- RAF 确保每帧最多通知一次，与浏览器渲染周期同步
- 配合 dirtyMessages 集合，仅对被修改的消息生成新引用

### ADR-003：为什么消息状态不持久化

**决策**：messageStore 的所有状态存储在内存中，页面刷新后从服务端重新拉取。

**理由**：

- 消息数据量大，localStorage 容量有限
- 消息状态需要与服务端保持一致，本地持久化可能导致数据陈旧
- 服务端是消息的唯一真实来源，客户端仅作为缓存
- 撤销.重做状态依赖完整的消息历史，持久化成本高

### ADR-004：为什么分屏布局使用二叉树结构

**决策**：paneLayoutStore 使用递归二叉树（PaneLeaf / PaneSplit）管理分屏布局。

**理由**：

- 二叉树天然支持嵌套分割（水平中嵌套垂直，反之亦然）
- 分割比例通过 ratio 属性精确控制
- 关闭 pane 时，兄弟节点自动提升，树结构自平衡
- 不可变操作（replaceNode、removeLeaf）保证状态变更可追踪

### ADR-005：为什么活动服务器使用 sessionStorage + localStorage 双写

**决策**：serverStore 的活动服务器 ID 同时写入 sessionStorage 和 localStorage。

**理由**：

- sessionStorage 提供窗口级隔离，同一浏览器多个窗口可连接不同服务器
- 新窗口首次打开时从 localStorage 读取，继承上次使用的服务器
- 窗口内刷新时从 sessionStorage 读取，保持当前选择
- 兼顾隔离性和连续性

---

## 10. 风险与缓解

| 风险                             | 影响                       | 缓解措施                                               |
| -------------------------------- | -------------------------- | ------------------------------------------------------ |
| localStorage 容量超限            | 状态写入失败，用户配置丢失 | 所有写入操作包裹 try-catch，静默忽略异常               |
| 快照缓存未正确清除               | UI 显示过期状态            | 订阅者回调中统一清除缓存，getSnapshot 时重建           |
| RAF 在非浏览器环境不可用         | 通知机制失效               | 检测 requestAnimationFrame 是否存在，不存在时立即通知  |
| 分屏树操作产生无效状态           | 界面渲染异常               | 所有树操作返回新树，不修改原节点，操作前校验节点存在性 |
| 服务器切换时部分 Store 未清理    | 旧服务器数据污染新服务器   | 在 serverStore 的 onServerChange 回调中统一清理        |
| IndexedDB 操作异步导致状态不一致 | 自定义音频加载延迟         | 预加载机制 + 内存缓存 + loading 状态追踪               |
| 快照引用不稳定导致过度重渲染     | 性能下降                   | 快照缓存 + 选择器模式 + 浅比较                         |
