# 008-terminal-system 技术规划（As-Built）

> 模块编号：008-terminal-system
> 状态：已实现
> 最后更新：2026-04-12

---

## 1. Technical Context

### 1.1 模块定位

终端系统为 OpenCodeUI 提供完整的 Web 终端能力，使用户可以在浏览器内获得与本地终端一致的交互体验。模块通过伪终端（PTY）后端与 WebSocket 双向通信，实现命令执行、输出渲染、动态调整大小等核心功能。

### 1.2 技术栈

| 技术             | 版本/用途                                                               |
| ---------------- | ----------------------------------------------------------------------- |
| React 19         | 组件框架，使用 `memo`、`useEffect`、`useRef`、`useState`、`useCallback` |
| TypeScript       | 类型安全，基于 `@opencode-ai/sdk` 的类型推导                            |
| xterm.js         | 终端渲染引擎（`@xterm/xterm`）                                          |
| FitAddon         | 容器尺寸自适应（`@xterm/addon-fit`）                                    |
| WebLinksAddon    | 超链接检测与点击（`@xterm/addon-web-links`）                            |
| WebSocket        | 浏览器原生 WebSocket API，双向 I/O 通道                                 |
| ResizeObserver   | 容器尺寸变化监听                                                        |
| MutationObserver | `data-mode` 属性变化监听（主题切换）                                    |
| Tauri Bridge     | 桌面端原生 PTY 桥接（`@tauri-apps/api/core`）                           |

### 1.3 源文件清单

| 文件                                | 行数 | 职责                                                                             |
| ----------------------------------- | ---- | -------------------------------------------------------------------------------- |
| `src/components/Terminal.tsx`       | 845  | 单个终端实例：初始化、渲染、输入输出、大小调整、主题适配、移动端辅助键、触屏滚动 |
| `src/components/BottomPanel.tsx`    | 317  | 底部面板容器：终端与其他面板（files、changes、MCP 等）的统一管理，会话恢复       |
| `src/components/PanelContainer.tsx` | 569  | 统一面板容器：标签页管理、拖拽排序、右键菜单、Add 菜单                           |
| `src/api/pty.ts`                    | 119  | PTY 会话 CRUD、WebSocket URL 构建                                                |
| `src/api/ptyBridge.ts`              | 90   | Tauri 桌面端 PTY 桥接                                                            |
| `src/types/api/pty.ts`              | 13   | PTY 类型声明（从 SDK 推导）                                                      |
| `src/store/layoutStore.ts`          | 988  | 布局状态管理：终端标签页、面板开关、高度、持久化                                 |
| `src/hooks/useInputCapabilities.ts` | 73   | 输入能力检测：触屏、粗指针、悬停能力                                             |

### 1.4 依赖关系

```
008-terminal-system
  ├── 001-api-layer（SDK 客户端、HTTP 工具、serverStore）
  ├── 009-theme-system（CSS 变量读取、data-mode 监听）
  ├── 010-state-management（layoutStore、messageStore）
  ├── 013-i18n-localization（react-i18next 翻译）
  ├── 014-tauri-desktop（isTauri 检测、Tauri PTY Bridge）
  └── 外部：@xterm/xterm、@xterm/addon-fit、@xterm/addon-web-links
```

---

## 2. Constitution Check

### 2.1 原则对齐

| 宪法原则                    | 对齐情况 | 说明                                                                                       |
| --------------------------- | -------- | ------------------------------------------------------------------------------------------ |
| 原则 2：OpenCode 兼容性优先 | 对齐     | PTY API 完全基于 `@opencode-ai/sdk` 的 `sdk.pty.*` 方法，不自行封装 HTTP 调用              |
| 原则 3：多平台统一代码库    | 对齐     | Web 端使用原生 WebSocket，Tauri 端使用 `ptyBridge.ts` 桥接，共享同一套 Terminal 组件       |
| 原则 4：自定义优于框架依赖  | 对齐     | 状态管理使用自定义 layoutStore，不引入 Redux/Zustand；仅 xterm.js 作为必要外部库           |
| 原则 5：实时通信优先        | 对齐     | 终端 I/O 使用 WebSocket 实时双向通信，SSE 用于后端事件推送（非终端模块）                   |
| 原则 9：主题与可访问性      | 对齐     | 终端主题从 CSS 变量动态读取，支持明暗模式切换；移动端辅助按键、触屏滚动优化                |
| 原则 10：模块化功能架构     | 对齐     | Terminal 组件自包含，通过 props 接收 ptyId/directory/isActive，与外部通过 layoutStore 通信 |

### 2.2 约束检查

| 约束          | 检查项                                        | 结果 |
| ------------- | --------------------------------------------- | ---- |
| C2 后端依赖   | PTY API 使用 `@opencode-ai/sdk`               | 通过 |
| C4 依赖最小化 | 仅引入 xterm.js 及官方 addon，无冗余依赖      | 通过 |
| C5 SSE 支持   | 终端模块不破坏 SSE，WebSocket 与 SSE 独立通道 | 通过 |

---

## 3. Research Findings

### 3.1 架构概览

终端系统采用三层架构：

```
┌─────────────────────────────────────────────────────────┐
│                    UI Layer                              │
│  BottomPanel ── PanelContainer ── Terminal (×N)          │
│  (容器编排)      (标签管理)        (渲染实例)              │
├─────────────────────────────────────────────────────────┤
│                    State Layer                           │
│  layoutStore（终端标签页、面板开关、持久化）               │
│  useInputCapabilities（触屏检测）                         │
├─────────────────────────────────────────────────────────┤
│                    Transport Layer                       │
│  pty.ts（REST API + WebSocket URL）                      │
│  ptyBridge.ts（Tauri 原生桥接）                          │
└─────────────────────────────────────────────────────────┘
```

### 3.2 关键发现

1. **Terminal 组件使用 `memo` 包裹**，防止不必要的重渲染。所有回调使用 `useCallback` 稳定引用。

2. **非活动终端保持挂载但隐藏**（`visibility: hidden` + `pointerEvents: none`），不销毁。这确保后台命令继续运行，切换回活动标签时即时响应。

3. **面板拖拽期间暂停 resize**：通过监听 `panel-resize-start` / `panel-resize-end` 自定义事件，拖拽期间 `isPanelResizingRef.current = true`，阻止 `handleResize` 执行。

4. **Tauri 环境使用动态 import**：`import('../api/ptyBridge')` 和 `import('@tauri-apps/plugin-opener')` 均为懒加载，避免 Web 端打包 Tauri 依赖。

5. **粘滞修饰键使用双源同步**：`stickyModifiers` state 用于 UI 渲染，`stickyModifiersRef` ref 用于 `sendTerminalData` 回调中读取最新值，避免闭包陈旧问题。

6. **会话恢复使用 requestId 机制**：`restoreRequestIdRef` 确保快速切换目录时仅最后一次请求生效，避免竞态条件。

7. **Terminal 组件通过 `lazy()` 懒加载**：`BottomPanel.tsx` 中使用 `const Terminal = lazy(() => import('./Terminal')...)`，减少初始包体积。

---

## 4. Data Model

### 4.1 核心实体

#### PTY 会话（后端实体）

```typescript
// 源自 @opencode-ai/sdk，通过 types/api/pty.ts 推导
type Pty = SDKPty

interface PtySize {
  cols: number
  rows: number
}

interface PtyCreateParams {
  cwd?: string
  // 其他 SDK 定义字段
}

interface PtyUpdateParams {
  size?: PtySize
  // 其他 SDK 定义字段
}
```

#### 终端标签页（前端状态）

```typescript
interface TerminalTab {
  id: string // 等于 PTY session ID
  title: string // 显示标题（由 Shell 通过 onTitleChange 动态设置）
  status: 'connecting' | 'connected' | 'disconnected' | 'exited'
}
```

#### 统一面板标签（扩展类型）

```typescript
interface PanelTab {
  id: string
  type: 'terminal' | 'files' | 'changes' | 'mcp' | 'skill' | 'worktree'
  position: 'bottom' | 'right'
  // Terminal 特有
  ptyId?: string
  title?: string
  status?: 'connecting' | 'connected' | 'disconnected' | 'exited'
  // Files 特有
  previewFile?: PreviewFile | null
  previewFiles?: PreviewFile[]
}
```

#### WebSocket 连接状态

```typescript
// 隐式状态机，通过 layoutStore.updateTerminalTab 驱动
type ConnectionStatus = 'connecting' | 'connected' | 'disconnected' | 'exited'

// 重连参数
const BASE_RECONNECT_DELAY = 1000 // 起始 1 秒
const MAX_RECONNECT_DELAY = 30000 // 最大 30 秒
// 退避公式：delay = min(1000 * 2^(attempt-1), 30000)
```

#### 粘滞修饰键

```typescript
type ModifierKey = 'ctrl' | 'alt'
type StickyModifiers = Record<ModifierKey, boolean>
// 初始值：{ ctrl: false, alt: false }
```

### 4.2 状态流转

#### 终端标签页状态机

```
[新建] ──createPtySession──→ [connecting]
                                │
                    WebSocket onopen
                                ↓
                           [connected]
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
        WebSocket close   WebSocket close    process exit
        (code≠1000)       (code=1000)        (后端通知)
              ↓                 ↓                 ↓
       [disconnected]     [disconnected]      [exited]
              │                 │
        指数退避重连        不重连
              │
         [connected] ←── 重连成功
```

#### 会话恢复流程

```
目录切换/页面加载
    │
    ├─ listPtySessions(directory)
    │     │
    │     ├─ 成功：sessions[]
    │     │     │
    │     │     ├─ requestId 校验（防竞态）
    │     │     │
    │     │     └─ syncTerminalSessions(directory, tabs)
    │     │           │
    │     │           ├─ 读取 localStorage 中的持久化排序
    │     │           ├─ 合并非终端 tabs（files、changes 等）
    │     │           ├─ 按持久化顺序排列终端 tabs
    │     │           └─ 恢复活动标签页
    │     │
    │     └─ 失败：uiErrorHandler
    │
    └─ setIsRestoring(false)
```

### 4.3 持久化策略

终端标签的排序和活动状态按工作目录分别存储：

```typescript
// localStorage key: 'opencode-terminal-layout'
interface PersistedTerminalLayoutMap {
  version: 1
  directories: Record<string, PersistedTerminalDirectoryLayout>
}

interface PersistedTerminalDirectoryLayout {
  order: {
    bottom: string[] // PTY ID 列表，按排序顺序
    right: string[]
  }
  activeTabId: {
    bottom: string | null
    right: string | null
  }
}
```

非终端面板（files、changes、mcp 等）使用独立的持久化 key：

```typescript
// localStorage key: 'opencode-panel-layout'
interface PersistedPanelLayout {
  version: 1
  panelTabs: PersistedPanelTab[] // 不包含 terminal 类型
  activeTabId: { bottom: string | null; right: string | null }
  rightPanelOpen: boolean
  bottomPanelOpen: boolean
}
```

---

## 5. Interface Contracts

### 5.1 Terminal 组件接口

```typescript
interface TerminalProps {
  ptyId: string // PTY 会话 ID
  directory?: string // 工作目录（用于 API 调用）
  isActive: boolean // 是否为当前活动终端
}

// 导出
export const Terminal: React.FC<TerminalProps>
// 使用 memo 包裹，防止不必要的重渲染
```

### 5.2 PTY API 接口

```typescript
// src/api/pty.ts

// 获取所有 PTY 会话列表
async function listPtySessions(directory?: string): Promise<Pty[]>

// 创建新的 PTY 会话
async function createPtySession(params: PtyCreateParams, directory?: string): Promise<Pty>

// 获取单个 PTY 会话信息
async function getPtySession(ptyId: string, directory?: string): Promise<Pty>

// 更新 PTY 会话（主要用于调整大小）
async function updatePtySession(ptyId: string, params: PtyUpdateParams, directory?: string): Promise<Pty>

// 删除 PTY 会话
async function removePtySession(ptyId: string, directory?: string): Promise<boolean>

// 构建 WebSocket 连接 URL
function getPtyConnectUrl(ptyId: string, directory?: string, options?: PtyConnectUrlOptions): string
```

### 5.3 Tauri PTY Bridge 接口

```typescript
// src/api/ptyBridge.ts

interface ConnectTauriPtyParams {
  ptyId: string
  directory?: string
  onConnected: () => void
  onMessage: (chunk: string) => void
  onDisconnected: (info: { code?: number; reason?: string }) => void
  onError: (message: string) => void
}

interface TauriPtyConnection {
  send: (data: string) => void
  close: () => void
}

async function connectTauriPty(params: ConnectTauriPtyParams): Promise<TauriPtyConnection>
```

### 5.4 layoutStore 终端相关接口

```typescript
// 终端标签页管理
addTerminalTab(tab: TerminalTab, openPanel?: boolean, position?: PanelPosition): void
removeTerminalTab(id: string): void
setActiveTerminal(id: string): void
updateTerminalTab(id: string, updates: Partial<TerminalTab>): void
reorderTerminalTabs(draggedId: string, targetId: string): void
getTerminalTabs(position?: PanelPosition): TerminalTab[]
syncTerminalSessions(directory: string | undefined, sessions: TerminalTab[]): void
setCurrentTerminalDirectory(directory?: string): void

// 面板控制
toggleBottomPanel(): void
openBottomPanel(): void
closeBottomPanel(): void
setBottomPanelHeight(height: number): void

// 统一 Tab API（终端也使用）
addTab(tab: Omit<PanelTab, 'id'> & { id?: string }, openPanel?: boolean): string
removeTab(tabId: string): void
setActiveTab(position: PanelPosition, tabId: string): void
reorderTabs(position: PanelPosition, draggedId: string, targetId: string): void
moveTab(tabId: string, toPosition: PanelPosition): void
updateTab(tabId: string, updates: Partial<Omit<PanelTab, 'id' | 'type'>>): void
```

### 5.5 自定义事件

| 事件名               | 触发时机         | 监听方                       |
| -------------------- | ---------------- | ---------------------------- |
| `panel-resize-start` | 面板拖拽调整开始 | Terminal 组件（暂停 resize） |
| `panel-resize-end`   | 面板拖拽调整结束 | Terminal 组件（立即 fit）    |

### 5.6 WebSocket 认证策略

```
浏览器 WebSocket（不支持自定义 header）：
  ├── 跨域：auth_token query parameter（base64(username:password)）
  │         + userinfo fallback（部分浏览器支持）
  └── 同源：浏览器自动复用页面的 Basic auth 凭据

Tauri Bridge（通过 Rust invoke）：
  └── Authorization header 传认证（不在 URL 中暴露）
```

---

## 6. Implementation Strategy

### 6.1 组件层次

```
App
  └── BottomPanel ({ directory })
        ├── ResizablePanel（可拖拽调整高度）
        │     └── PanelContainer（标签页管理）
        │           ├── PanelTabButton × N（标签按钮，支持拖拽排序）
        │           ├── Context Menu（右键菜单：移动到其他面板）
        │           ├── Add Menu（+ 按钮：新建终端/files/changes 等）
        │           └── children(activeTab) → TerminalContent
        │                     └── Terminal × N（所有终端实例始终挂载）
        │                           ├── xterm.js 实例
        │                           ├── FitAddon
        │                           ├── WebLinksAddon
        │                           ├── MobileExtraKeys（触屏设备）
        │                           └── <style>（滚动条样式注入）
        └── TerminalContent（内部组件，渲染所有终端实例）
```

### 6.2 Terminal 组件生命周期

#### 创建阶段（useEffect #1）

```
1. 检查 containerRef.current 和 hasBeenActive
2. 创建 XTerm 实例（配置主题、字体、光标、scrollback 等）
3. 加载 FitAddon 和 WebLinksAddon
4. terminal.open(containerRef.current)
5. 配置 textarea 属性（触屏设备：autocapitalize/autocomplete/spellcheck 等）
6. requestAnimationFrame → fitAddon.fit()
7. 连接 WebSocket / Tauri Bridge（带自动重连）
8. 注册 onData 事件 → sendTerminalData
9. 注册 onTitleChange 事件 → layoutStore.updateTerminalTab
```

#### 运行阶段

- **用户输入**：`terminal.onData` → `sendTerminalData` → 应用粘滞修饰键 → `transportSendRef.current(data)` → WebSocket.send() / Tauri invoke
- **后端输出**：WebSocket.onmessage / Tauri Channel → `terminal.write(chunk)` → ANSI 解析 → 像素渲染
- **大小调整**：ResizeObserver / window.resize → 防抖 16ms → fitAddon.fit() → updatePtySession()
- **主题切换**：MutationObserver 监听 `data-mode` → `terminal.options.theme = getTerminalTheme(isDarkMode())`
- **标签切换**：`isActive` 变化 → focus() + requestAnimationFrame → fitAddon.fit()

#### 销毁阶段（useEffect cleanup）

```
1. mountedRef.current = false（防止异步操作更新已卸载组件）
2. intentionalClose = true（标记主动关闭，不触发重连）
3. cancelAnimationFrame(wsConnectTimeout)
4. clearTimeout(reconnectTimer)
5. clearTimeout(resizeTimeoutRef.current)
6. transportDisconnectRef.current()（关闭 WebSocket / Tauri Bridge）
7. disposeData.dispose()（取消 onData 订阅）
8. disposeTitle.dispose()（取消 onTitleChange 订阅）
9. textarea.removeEventListener('blur')
10. resetTransport()
11. fitAddon.dispose()
12. webLinksAddon.dispose()
13. terminal.dispose()
14. terminalRef.current = null, fitAddonRef.current = null
```

### 6.3 大小调整策略

```
触发源                    机制                      防抖       同步后端
────────────────────────────────────────────────────────────────────
window.resize            window.addEventListener    16ms       updatePtySession
container ResizeObserver new ResizeObserver         16ms       updatePtySession
panel-resize-end         window.addEventListener    无（立即）  updatePtySession
标签切换（isActive）     useEffect + requestAF      无（立即）  不直接同步（fit 即可）
面板拖拽期间             isPanelResizingRef 检查    跳过       不执行
```

### 6.4 移动端辅助按键实现

```
MobileExtraKeys 组件：
  ├── 两行按键布局（CSS Grid，7 列）
  │     ├── 第一行：ESC、/、-、HOME、↑、END、PGUP
  │     └── 第二行：TAB、CTRL（粘滞）、ALT（粘滞）、←、↓、→、PGDN
  │
  ├── 粘滞修饰键逻辑：
  │     ├── 点击 CTRL/ALT → toggleStickyModifier → 高亮显示
  │     ├── 下一次输入 → applyStickyModifiers → 自动清除
  │     └── 支持同时激活（Ctrl+Alt+X）
  │
  ├── toCtrlSequence：将普通字符转换为 Ctrl 控制序列
  │     ├── A-Z → 0x01-0x1A
  │     ├── 特殊字符映射（[ → 0x1B, ? → 0x7F 等）
  │     └── Alt 修饰 → 前缀 0x1B
  │
  └── 发送后自动聚焦回终端输入区
```

### 6.5 触屏滚动优化

```
touchstart：
  ├── 停止惯性滚动
  ├── 记录起始 Y 坐标和时间
  └── velocity = 0

touchmove：
  ├── 计算 delta（超过 6px 阈值才触发）
  ├── 计算瞬时速度（归一化到每帧 px）
  ├── viewport.scrollTop = scrollStart + delta
  └── e.preventDefault()（阻止页面滚动）

touchend：
  └── startMomentum()：
        ├── 摩擦系数 0.95
        ├── 速度 < 0.5 px/帧时停止
        └── 惯性结束后 600ms 淡出滚动条
```

### 6.6 主题适配实现

```
getHSLColor(varName)：
  ├── 读取 CSS 变量值（HSL 格式："h s% l%"）
  ├── HSL → RGB 转换
  └── 返回 hex 字符串（#rrggbb）

getTerminalTheme(isDark)：
  ├── foreground：从 --text-100 CSS 变量动态读取
  ├── background：#00000000（完全透明，由 CSS 覆盖）
  ├── cursor / selection：随主题变化
  └── ANSI 16 色：暗色/亮色两套完整调色板

MutationObserver：
  ├── 监听 document.documentElement 的 data-mode 属性
  ├── 变化时更新 terminal.options.theme
  └── 终端内容不闪烁、不丢失
```

### 6.7 Tauri 环境适配

```
isTauri() 检测：
  ├── true：使用 ptyBridge.ts
  │     ├── import('../api/ptyBridge') 动态加载
  │     ├── connectTauriPty() 建立桥接
  │     ├── Tauri Channel 接收事件（connected/data/disconnected/error）
  │     ├── invoke('bridge_connect/bridge_send/bridge_disconnect')
  │     └── 认证通过 Authorization header 传递
  │
  └── false：使用原生 WebSocket
        ├── new WebSocket(wsUrl)
        ├── 认证通过 URL userinfo / auth_token query parameter
        └── 链接打开使用 window.open()
```

---

## 7. Error Handling

### 7.1 WebSocket 连接错误

| 场景                                | 处理方式                                                         |
| ----------------------------------- | ---------------------------------------------------------------- |
| 连接建立失败                        | `onerror` 记录日志，`onclose` 触发重连                           |
| 连接意外断开（code ≠ 1000）         | 指数退避重连，终端内显示 `[Disconnected, reconnecting in Xs...]` |
| 主动关闭（intentionalClose = true） | 不触发重连，显示 `[Connection closed]`                           |
| 正常关闭（code = 1000）             | 不触发重连，显示 `[Connection closed]`                           |
| 重连失败                            | 延迟按 `min(1000 * 2^(attempt-1), 30000)` 增长                   |
| Tauri Bridge 初始化失败             | catch 错误，调用 `handleDisconnected({ reason: message })`       |

### 7.2 组件卸载防护

```typescript
// mountedRef 防止组件卸载后的状态更新
const mountedRef = useRef(true)

// 所有异步回调中检查
if (!mountedRef.current) return

// cleanup 中设置
return () => {
  mountedRef.current = false
  // ... 清理资源
}
```

### 7.3 竞态条件防护

```typescript
// 会话恢复 requestId 机制
const restoreRequestIdRef = useRef(0)

const requestId = ++restoreRequestIdRef.current
const sessions = await listPtySessions(normalizedDirectory)
if (restoreRequestIdRef.current !== requestId) return // 已过期，丢弃
```

### 7.4 API 调用错误处理

| 操作             | 错误处理                                             |
| ---------------- | ---------------------------------------------------- |
| createPtySession | `uiErrorHandler('create terminal', error)`           |
| removePtySession | `catch { /* ignore - may already be closed */ }`     |
| updatePtySession | `.catch(() => {})`（静默失败，不影响用户体验）       |
| listPtySessions  | `uiErrorHandler('restore terminal sessions', error)` |

### 7.5 资源泄漏防护

清理函数确保以下资源全部释放：

- WebSocket 连接（`ws.close()` / Tauri `bridge_disconnect`）
- 重连计时器（`clearTimeout(reconnectTimer)`）
- 防抖计时器（`clearTimeout(resizeTimeoutRef.current)`）
- 动画帧（`cancelAnimationFrame(wsConnectTimeout)`）
- xterm addons（`fitAddon.dispose()`、`webLinksAddon.dispose()`）
- xterm 实例（`terminal.dispose()`）
- 事件监听器（`textarea.removeEventListener`、`window.removeEventListener`）
- ResizeObserver（`resizeObserver.disconnect()`）
- MutationObserver（`observer.disconnect()`）
- Touch 事件监听器（`container.removeEventListener`）
- 惯性滚动动画帧（`cancelAnimationFrame(momentumRaf)`）

---

## 8. Testing Considerations

### 8.1 单元测试范围

| 测试目标                             | 测试内容                                                 |
| ------------------------------------ | -------------------------------------------------------- |
| `getHSLColor`                        | CSS 变量解析、HSL 转 RGB 转换、空值兜底                  |
| `toCtrlSequence`                     | A-Z 转换、特殊字符映射、非单字符透传                     |
| `applyStickyModifiers`               | 单独 Ctrl、单独 Alt、同时激活、无修饰                    |
| `getPtyConnectUrl`                   | 跨域 URL 构建、同源 URL 构建、Tauri 模式、无认证         |
| `normalizePty`                       | 兼容旧版 `running` 字段到 `status` 的转换                |
| `layoutStore` 终端方法               | addTerminalTab、removeTerminalTab、reorder、syncSessions |
| `sanitizePersistedTerminalLayoutMap` | 非法数据过滤、版本校验、字段类型检查                     |

### 8.2 集成测试场景

| 场景               | 验证点                                             |
| ------------------ | -------------------------------------------------- |
| 创建终端并执行命令 | PTY 创建成功、WebSocket 连接建立、命令输出正确渲染 |
| 多终端并行         | 多个终端独立运行、切换标签不影响后台命令           |
| 动态大小调整       | 窗口 resize 后 cols/rows 正确更新、后端收到新尺寸  |
| 网络断开与重连     | 断开后显示重连提示、恢复后自动重连、指数退避正确   |
| 页面刷新后会话恢复 | 标签页恢复、排序一致、不自动打开面板               |
| 主题切换           | 明暗模式切换后终端颜色正确、内容不丢失             |
| 移动端辅助按键     | 粘滞修饰键激活/清除、Ctrl+C 发送正确               |
| 触屏滚动           | 惯性滚动、滚动条淡入淡出、不触发页面滚动           |
| 标签拖拽排序       | 拖拽后顺序正确、持久化到 localStorage              |
| 关闭终端           | PTY 会话销毁、标签移除、最后一个标签关闭面板       |

### 8.3 手动测试清单

| 测试项           | 操作步骤                    | 预期结果                                      |
| ---------------- | --------------------------- | --------------------------------------------- |
| ANSI 色彩渲染    | 执行 `ls --color=auto`      | 目录、文件、可执行文件以不同颜色显示          |
| 面板拖拽 resize  | 拖拽底部面板上边缘          | 拖拽期间不闪烁，结束后立即适配                |
| Tauri 环境终端   | 在 Tauri 桌面应用中打开终端 | 使用原生 Bridge 连接，功能与 Web 端一致       |
| 链接点击         | 终端输出中包含 URL          | 点击后在新窗口打开（Tauri 中使用原生 opener） |
| 快速切换目录     | 连续切换多个工作目录        | 仅最后一次请求生效，终端标签状态正确          |
| 组件快速挂载卸载 | 快速打开关闭终端面板        | 无内存泄漏、无残留异步操作                    |

### 8.4 性能关注点

| 关注点       | 当前策略                        | 建议                                          |
| ------------ | ------------------------------- | --------------------------------------------- |
| 终端实例数量 | 所有终端始终挂载（隐藏非活动）  | 监控内存占用，考虑超过 N 个时卸载最久未使用的 |
| Resize 防抖  | 16ms（约 60fps）                | 当前策略合理，无需调整                        |
| 主题切换     | MutationObserver 监听单属性     | 高效，无需优化                                |
| 触屏滚动     | 自定义 touch 事件处理           | 避免使用 passive: false 的过度调用            |
| 懒加载       | Terminal 组件通过 `lazy()` 加载 | 减少初始包体积，策略正确                      |

---

## 9. 架构决策记录（As-Built）

### ADR-001：WebSocket 认证通过 URL userinfo / auth_token

**决策**：浏览器 WebSocket 连接将认证信息编码到 URL 中。跨域使用 `auth_token` query parameter（base64 编码），同源依赖浏览器自动复用 Basic auth。Tauri 环境通过 Authorization header 传递。

**理由**：WebSocket 协议不支持自定义请求头。URL userinfo 是 RFC 标准方式。Tauri 环境通过 Rust invoke 可以设置 header，无需 URL 编码。

### ADR-002：非活动终端保持挂载而非销毁

**决策**：切换到其他标签时，非活动终端实例保持挂载但隐藏（`visibility: hidden` + `pointerEvents: none`）。

**理由**：销毁重建会导致终端内容丢失。保持连接确保后台命令继续运行。切换回活动标签时即时响应，无需重新连接。

### ADR-003：面板拖拽期间暂停终端 resize

**决策**：通过自定义事件 `panel-resize-start` / `panel-resize-end` 通知 Terminal 组件，拖拽期间不执行 fit 操作。

**理由**：拖拽过程中容器尺寸频繁变化，频繁 fit 导致性能问题和内容闪烁。结束后一次性 fit 即可得到正确尺寸。

### ADR-004：粘滞修饰键方案

**决策**：CTRL 和 ALT 键采用粘滞（sticky）模式，点击激活后下一次输入自动附带修饰。

**理由**：参考 Termux、Blink Shell 等成熟终端应用。长按交互在触屏上不够精确。粘滞模式允许先按修饰键再按目标键，符合物理键盘习惯。

### ADR-005：终端标签按目录分组持久化

**决策**：终端标签的排序和活动状态按工作目录分别存储到 localStorage（`opencode-terminal-layout`）。

**理由**：不同项目可能有不同的终端布局偏好。切换目录时恢复该目录下的终端布局。避免不同项目的终端标签互相干扰。

### ADR-006：统一 PanelTab 类型

**决策**：将终端标签页纳入统一的 `PanelTab` 类型体系，与 files、changes、mcp 等面板类型共享同一套管理逻辑。

**理由**：减少代码重复，PanelContainer 可以统一管理所有类型的标签页。终端标签的持久化独立存储（不包含在 `opencode-panel-layout` 中），避免与非终端面板混淆。

### ADR-007：Terminal 组件懒加载

**决策**：BottomPanel 中使用 `React.lazy()` 懒加载 Terminal 组件。

**理由**：xterm.js 及其 addon 体积较大，懒加载减少初始包体积。用户不一定需要打开终端，按需加载更合理。

---

## 10. 风险与缓解

| 风险                          | 影响                               | 缓解措施                                           | 当前状态           |
| ----------------------------- | ---------------------------------- | -------------------------------------------------- | ------------------ |
| WebSocket 连接泄漏            | 组件卸载后连接未关闭，占用后端资源 | 清理函数中显式关闭 WebSocket，使用 mountedRef 防护 | 已实现             |
| 频繁 resize 导致性能下降      | 窗口快速变化时终端频繁重绘         | 16ms 防抖 + 面板拖拽期间暂停 resize                | 已实现             |
| 移动端虚拟键盘遮挡终端输入区  | 用户无法看到输入内容               | textarea 设置 inputmode、enterkeyhint 等属性优化   | 已实现             |
| 大量终端标签导致内存占用过高  | 同时打开过多终端实例消耗资源       | 每个终端独立生命周期管理，关闭时彻底释放资源       | 已实现，可考虑上限 |
| 主题切换时 CSS 变量未就绪     | 终端颜色短暂异常                   | getHSLColor 函数在变量为空时返回默认颜色兜底       | 已实现             |
| 并发目录切换导致状态混乱      | 快速切换目录时终端标签状态不一致   | requestId 机制确保仅最后一次请求生效               | 已实现             |
| Tauri Bridge 动态 import 失败 | 桌面端终端无法连接                 | catch 错误并调用 handleDisconnected，触发重连      | 已实现             |
| xterm.js 版本升级兼容性       | addon API 变更导致功能异常         | 锁定 xterm.js 版本，升级时全面测试                 | 需关注             |
