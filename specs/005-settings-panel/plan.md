# 005-settings-panel 技术方案（As-Built）

> 本文档是对已完成的设置面板模块的回溯性技术规划，记录"实际建成"的架构设计、数据模型与集成策略。

---

## 1. Technical Context

### 1.1 模块定位

设置面板（`005-settings-panel`）是 OpenCodeUI 中负责集中管理用户配置的 UI 模块。它以模态对话框形式呈现，通过左右分栏（桌面端）或顶部标签栏（移动端）组织六大类设置：外观、聊天、通知、服务器、服务、快捷键。

### 1.2 技术栈

| 维度     | 选型                                                               |
| -------- | ------------------------------------------------------------------ |
| 框架     | React 19 + TypeScript                                              |
| 样式     | Tailwind CSS v4                                                    |
| 国际化   | i18next + react-i18next                                            |
| 状态管理 | 自定义 Store 模式（模块级单例 + `useSyncExternalStore`）           |
| 持久化   | localStorage + IndexedDB（音频文件）+ sessionStorage（窗口级隔离） |
| 桌面桥接 | Tauri 2 `invoke` 命令（服务管理）                                  |

### 1.3 源码目录结构

```
src/features/settings/
├── SettingsDialog.tsx              # 对话框容器、标签页导航、响应式布局
├── KeybindingsSection.tsx          # 快捷键设置面板
└── components/
    ├── SettingsUI.tsx              # UI 原语（Toggle、SegmentedControl、SettingRow、SettingsSection、SettingsCard）
    ├── AppearanceSettings.tsx      # 外观设置
    ├── ChatSettings.tsx            # 聊天设置
    ├── NotificationSettings.tsx    # 通知设置
    ├── ServersSettings.tsx         # 服务器管理
    └── ServiceSettings.tsx         # 服务管理（仅桌面端）

src/store/
├── keybindingStore.ts              # 快捷键配置管理
├── serverStore.ts                  # 多服务器配置与健康检查
├── soundStore.ts                   # 声音设置 + IndexedDB 音频持久化
├── themeStore.ts                   # 主题、颜色模式、自定义 CSS、聊天显示选项
├── layoutStore.ts                  # 侧边栏相关设置
├── notificationStore.ts            # 通知与 Toast 状态
├── autoApproveStore.ts             # 自动批准状态
└── serviceStore.ts                 # Tauri 服务管理状态

src/hooks/
├── useKeybindings.ts               # 快捷键 React hooks
├── useServerStore.ts               # 服务器 React hooks
├── useNotification.ts              # 通知权限 hooks
├── usePathMode.ts                  # 路径格式 hooks
├── useTheme.ts                     # 主题 hooks
└── useIsMobile.ts                  # 移动端检测 hooks
```

### 1.4 文件规模

| 文件                                  | 行数 | 职责                                 |
| ------------------------------------- | ---- | ------------------------------------ |
| `SettingsDialog.tsx`                  | 296  | 对话框外壳、标签路由、响应式         |
| `KeybindingsSection.tsx`              | 330  | 快捷键列表、搜索、按键捕获           |
| `components/SettingsUI.tsx`           | 167  | UI 原语组件                          |
| `components/AppearanceSettings.tsx`   | 457  | 主题预设、CSS 编辑器、显示选项       |
| `components/ChatSettings.tsx`         | 224  | 路径格式、沉浸模式、步骤信息         |
| `components/NotificationSettings.tsx` | 446  | 系统通知、音效配置、上传             |
| `components/ServersSettings.tsx`      | 566  | 服务器增删改查、健康检查             |
| `components/ServiceSettings.tsx`      | 244  | 本地服务启停、环境变量               |
| `store/keybindingStore.ts`            | 706  | 快捷键解析、格式化、冲突检测、持久化 |
| `store/serverStore.ts`                | 429  | 多服务器 CRUD、健康检查、认证        |
| `store/soundStore.ts`                 | 397  | 声音设置、IndexedDB 音频管理         |

总计约 4,262 行代码。

---

## 2. Constitution Check

对照项目宪法逐项验证：

| 宪法原则                    | 符合情况 | 说明                                                                                               |
| --------------------------- | -------- | -------------------------------------------------------------------------------------------------- |
| 原则 2：OpenCode 兼容性优先 | 符合     | 服务器管理直接对接 OpenCode 后端 `/global/health` 端点，使用 `@opencode-ai/sdk`                    |
| 原则 3：多平台统一代码库    | 符合     | Web 端与 Tauri 桌面端共享同一套 `src/features/settings/` 代码，通过 `isTauri()` 条件渲染服务标签页 |
| 原则 4：自定义优于框架依赖  | 符合     | 使用自定义 Store 模式（非 Redux/Zustand），自定义 UI 原语（非 Radix/Headless UI）                  |
| 原则 6：中文优先文档        | 符合     | 代码注释、i18n 键名均以中文为第一语言                                                              |
| 原则 9：主题与可访问性      | 符合     | 支持明暗模式切换、移动端响应式布局、键盘导航（ArrowUp/ArrowDown 切换标签页）                       |
| 原则 10：模块化功能架构     | 符合     | 设置面板自包含于 `src/features/settings/`，通过明确的 store 接口与外部通信                         |
| 约束 C3：构建校验           | 符合     | 代码通过 TypeScript 类型检查、ESLint、生产构建                                                     |
| 约束 C4：依赖最小化         | 符合     | 未引入额外 UI 库，所有组件均为自定义实现                                                           |

---

## 3. Research Findings

### 3.1 对话框架构

`SettingsDialog` 作为顶层容器，依赖 `../../components/ui/Dialog` 提供模态外壳。对话框宽度为 `min(97vw, 1040px)`，高度为 `min(90vh, 820px)`。

**标签页路由机制**：

- 通过 `TabContent` 函数组件实现 switch-case 路由
- 6 个标签页：`servers`、`chat`、`appearance`、`notifications`、`service`、`keybindings`
- 默认标签页为 `servers`（`initialTab` prop 默认值）
- `initialTab` 为 `'general'` 时自动映射为 `'chat'`

**分组策略**：

- 核心组（`groups.core`）：servers、chat、appearance、notifications
- 高级组（`groups.advanced`）：service、keybindings

**响应式断点**：

- `useIsMobile()` 检测移动端
- 移动端：全屏宽度、92vh 高度、顶部横向滚动标签栏（胶囊样式）
- 桌面端：左侧 200px 导航栏（xl 断点 236px）、右侧内容区

### 3.2 平台差异处理

服务标签页的可见性通过以下逻辑控制：

```typescript
const isTauriDesktop = isTauri() && !isMobile
const visibleTabIds = isTauriDesktop ? TAB_IDS : TAB_IDS.filter(id => id !== 'service')
```

`ServiceSettings` 组件内部也做了二次检查，非桌面端时显示"仅桌面端可用"提示。

### 3.3 快捷键系统实现

快捷键系统是本模块最复杂的子功能，涉及：

**解析层**（`keybindingStore.ts`）：

- `parseKeybinding()`：将字符串如 `"Ctrl+Shift+P"` 解析为 `{ key, ctrl, alt, shift, meta }` 结构
- `normalizeKeybindingString()`：规范化后比较，解决 `"Ctrl+K"` 与 `"ctrl+k"` 等价问题
- `keybindingsEqual()`：两个快捷键字符串是否等价
- `formatKeybinding()`：格式化为用户友好显示（箭头符号、⌘ 符号等）
- `keyEventToString()`：从 `KeyboardEvent` 生成快捷键字符串

**冲突检测**：

- `isKeyUsed(keyStr, excludeAction)`：检查快捷键是否已被其他动作占用
- 使用规范化字符串比较，避免格式差异导致的误判

**编辑模式**（`KeybindingRow` 组件）：

- 点击快捷键进入编辑模式
- 通过 `document.addEventListener('keydown', handler, { capture: true })` 捕获按键
- 忽略纯修饰键（Control、Alt、Shift、Meta）的单独按下
- Escape 取消，Enter 确认
- 冲突时显示红色错误提示

**持久化策略**：

- 仅保存与默认值不同的配置到 `localStorage`（key: `opencode-keybindings`）
- 存储格式：`Record<KeybindingAction, string>`

### 3.4 服务器管理实现

**健康检查流程**：

1. 标记状态为 `checking`
2. 发起 GET 请求到 `${server.url}/global/health`，超时 5 秒
3. 根据响应判断：
   - `200` → `online`，解析 `version` 字段，记录延迟
   - `401` → `unauthorized`
   - 其他 HTTP 状态 → `error`
   - 网络异常 → `offline`

**切换服务器副作用**：

```typescript
const handleSelectServer = (id: string) => {
  messageStore.clearSession(routeSessionId) // 清理消息
  setActiveServer(id) // 触发 serverChangeListeners → reconnectSSE()
  navigateHome() // 导航回首页
}
```

**跨域安全警告**：
当添加服务器时检测到跨域（`serverUrl.origin !== window.location.origin`）且配置了密码，显示警告提示并链接到 OpenCode issue #10047。

### 3.5 通知音效系统

**双层存储架构**：

- 配置元数据（开关、音量、音效选择）→ `localStorage`（key: `opencode:sound-settings`）
- 自定义音频 Blob → IndexedDB（database: `opencode-sounds`, store: `custom-audio`）

**音频验证**：

- 大小限制：2MB
- MIME 类型：`audio/*` 或白名单中的具体类型

**懒加载策略**：

- 构造时异步预加载所有已上传的自定义音频到内存缓存
- `getCustomAudioBlobAsync()` 按需从 IndexedDB 加载，带 loading 去重

### 3.6 沉浸模式联动

开启沉浸模式时，自动联动修改四个子选项：

```typescript
const handleImmersiveModeToggle = () => {
  themeStore.setImmersiveMode(v)
  setInlineToolRequests(v) // 自动启用
  setDescriptiveToolSteps(v) // 自动启用
  setToolCardStyle(v ? 'compact' : 'classic') // 切换为紧凑
  setCompactInlinePermission(v) // 自动启用
}
```

关闭时不恢复原值，保持当前状态。

### 3.7 自定义 CSS 编辑器

- 使用原生 `<textarea>` 配合等宽字体（`font-mono`）
- 防抖保存：400ms 延迟
- 模板加载：编辑器为空时显示"加载模板"按钮，点击填充完整的 CSS 变量模板
- 清除按钮：编辑器非空时显示
- 模板包含 light/dark/auto 三套完整的 CSS 变量定义（HSL 格式）

---

## 4. Data Model

### 4.1 快捷键配置

```typescript
type KeybindingAction =
  | 'openSettings'
  | 'openProject'
  | 'commandPalette'
  | 'toggleSidebar'
  | 'toggleRightPanel'
  | 'focusInput'
  | 'newSession'
  | 'archiveSession'
  | 'previousSession'
  | 'nextSession'
  | 'toggleTerminal'
  | 'newTerminal'
  | 'selectModel'
  | 'toggleAgent'
  | 'sendMessage'
  | 'cancelMessage'
  | 'copyLastResponse'
  | 'toggleFullAuto'
  | 'focusNextPane'
  | 'focusPrevPane'
  | 'splitRight'
  | 'splitDown'
  | 'closePane'
  | 'togglePaneFullscreen'

interface KeybindingConfig {
  action: KeybindingAction
  label: string
  description: string
  defaultKey: string
  currentKey: string
  category: 'general' | 'session' | 'pane' | 'terminal' | 'model' | 'message' | 'permission'
}

interface ParsedKeybinding {
  key: string // 主键（小写）
  ctrl: boolean
  alt: boolean
  shift: boolean
  meta: boolean // Command/Win
}
```

**分类与动作数量**：

| 分类       | 动作数 | 示例                             |
| ---------- | ------ | -------------------------------- |
| general    | 6      | openSettings, commandPalette     |
| session    | 4      | newSession, archiveSession       |
| pane       | 6      | splitRight, togglePaneFullscreen |
| terminal   | 2      | toggleTerminal, newTerminal      |
| model      | 2      | selectModel, toggleAgent         |
| message    | 3      | sendMessage, cancelMessage       |
| permission | 1      | toggleFullAuto                   |

总计 24 个动作。

### 4.2 服务器配置

```typescript
interface ServerConfig {
  id: string // 自动生成: server-{timestamp}-{random}
  name: string
  url: string // 不含尾部斜杠
  isDefault?: boolean
  auth?: ServerAuth
}

interface ServerAuth {
  username: string // 默认 'opencode'
  password: string
}

interface ServerHealth {
  status: 'checking' | 'online' | 'offline' | 'error' | 'unauthorized'
  latency?: number // ms
  lastCheck?: number // timestamp
  error?: string
  version?: string
}
```

**存储键**：

- 服务器列表：`localStorage['opencode-servers']`
- 活动服务器：`sessionStorage['opencode-active-server']` + `localStorage['opencode-active-server']`

### 4.3 音效配置

```typescript
interface SoundSettings {
  enabled: boolean
  currentSessionEnabled: boolean
  volume: number // 0-100
  events: Record<NotificationType, EventSoundConfig>
}

interface EventSoundConfig {
  soundId: string // 'builtin:xxx' | 'custom' | 'none'
  customFileName?: string // 展示用文件名
}

type NotificationType = 'completed' | 'permission' | 'question' | 'error'
```

**存储键**：

- 配置：`localStorage['opencode:sound-settings']`
- 音频：IndexedDB `opencode-sounds` / `custom-audio` store，key 格式 `custom-{type}`

### 4.4 主题与聊天设置（由 themeStore 管理）

设置面板读取并修改的 themeStore 字段：

| 字段                      | 类型                                | 用途             |
| ------------------------- | ----------------------------------- | ---------------- |
| `presetId`                | string                              | 当前主题预设     |
| `mode`                    | 'system' \| 'light' \| 'dark'       | 颜色模式         |
| `customCSS`               | string                              | 自定义 CSS 代码  |
| `isWideMode`              | boolean                             | 宽屏模式         |
| `glassEffect`             | boolean                             | 毛玻璃效果       |
| `codeWordWrap`            | boolean                             | 代码自动换行     |
| `diffStyle`               | 'markers' \| 'changeBars'           | Diff 样式        |
| `immersiveMode`           | boolean                             | 沉浸模式         |
| `inlineToolRequests`      | boolean                             | 内联工具请求     |
| `descriptiveToolSteps`    | boolean                             | 描述性工具步骤   |
| `toolCardStyle`           | 'classic' \| 'compact'              | 工具卡片样式     |
| `compactInlinePermission` | boolean                             | 紧凑权限提示     |
| `reasoningDisplayMode`    | 'capsule' \| 'italic' \| 'markdown' | 思考过程显示     |
| `collapseUserMessages`    | boolean                             | 长消息折叠       |
| `queueFollowupMessages`   | boolean                             | 后续消息排队     |
| `stepFinishDisplay`       | Record<string, boolean>             | 步骤完成信息开关 |

### 4.5 状态流转图

**服务器健康检查状态机**：

```
[初始] → checking → online ──────────────→ checking（刷新时）
                  → unauthorized ────────→ checking
                  → offline ─────────────→ checking
                  → error ───────────────→ checking
```

**快捷键编辑状态机**：

```
[显示] → 点击 → [编辑中] → Enter → [显示]（新快捷键生效）
                        → Escape → [显示]（不变）
                        → 冲突按键 → [编辑中]（显示错误）
```

---

## 5. Interface Contracts

### 5.1 SettingsDialog Props

```typescript
interface SettingsDialogProps {
  isOpen: boolean
  onClose: () => void
  initialTab?: SettingsTab | 'general' // 默认 'servers'
}

type SettingsTab = 'appearance' | 'chat' | 'notifications' | 'service' | 'servers' | 'keybindings'
```

### 5.2 Store 接口契约

设置面板消费的 Store 方法：

| Store               | 读取方法                                                    | 写入方法                                                                                                              |
| ------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `keybindingStore`   | `getAll()`, `isKeyUsed()`                                   | `setKeybinding()`, `resetKeybinding()`, `resetAll()`                                                                  |
| `serverStore`       | `getServers()`, `getActiveServer()`, `getHealth()`          | `addServer()`, `removeServer()`, `updateServer()`, `setActiveServer()`, `checkHealth()`, `checkAllHealth()`           |
| `soundStore`        | `getSnapshot()`, `hasCustomAudio()`, `getCustomAudioBlob()` | `setEnabled()`, `setVolume()`, `setEventSound()`, `uploadCustomAudio()`, `removeCustomAudio()`, `exportCustomAudio()` |
| `themeStore`        | 直接读取属性                                                | `setPresetWithAnimation()`, `setThemeWithAnimation()`, `setCustomCSS()`, `setImmersiveMode()` 等                      |
| `layoutStore`       | `useLayoutStore()` hook                                     | `setSidebarFolderRecents()`, `setSidebarFolderRecentsShowDiff()`, `setSidebarShowChildSessions()`                     |
| `autoApproveStore`  | `.enabled`                                                  | `setEnabled()`, `clearAllRules()`                                                                                     |
| `notificationStore` | `.toastEnabled`                                             | `setToastEnabled()`                                                                                                   |
| `serviceStore`      | `useServiceStore()` hook                                    | `setAutoStart()`, `setBinaryPath()`, `setEnvVars()`, `setRunning()`, `setStartedByUs()`, `setStarting()`              |

### 5.3 Hook 接口

| Hook                   | 返回值                                                                                                                      | 用途               |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| `useKeybindingStore()` | `{ keybindings, setKeybinding, resetKeybinding, resetAll, isKeyUsed }`                                                      | 快捷键操作         |
| `useServerStore()`     | `{ servers, activeServer, addServer, removeServer, updateServer, setActiveServer, checkHealth, checkAllHealth, getHealth }` | 服务器操作         |
| `useSoundSettings()`   | `SoundSettings`                                                                                                             | 声音配置响应式读取 |
| `useTheme()`           | 主题相关状态和方法                                                                                                          | 主题操作           |
| `usePathMode()`        | `{ pathMode, setPathMode, effectiveStyle, detectedStyle, isAutoMode }`                                                      | 路径格式           |
| `useNotification()`    | `{ enabled, setEnabled, supported, permission, sendNotification }`                                                          | 通知权限           |
| `useServiceStore()`    | 服务相关状态                                                                                                                | 服务状态读取       |
| `useRouter()`          | `{ navigateHome, sessionId }`                                                                                               | 路由导航           |
| `useIsMobile()`        | boolean                                                                                                                     | 移动端检测         |

### 5.4 UI 原语接口

```typescript
// Toggle 开关
function Toggle({ enabled, onChange }: { enabled: boolean; onChange: () => void })

// 分段控制器
function SegmentedControl<T extends string>({
  value: T,
  options: { value: T; label: string; icon?: React.ReactNode }[],
  onChange: (value: T, event?: React.MouseEvent) => void
})

// 设置行
function SettingRow({ label, description?, icon?, children, onClick?, className? })

// 设置区块
function SettingsSection({ title, children })

// 设置卡片
function SettingsCard({ title, description?, actions?, children, className? })
```

---

## 6. Implementation Strategy

### 6.1 组件层次

```
SettingsDialog
├── [Mobile Layout]
│   ├── Sticky Header (title + close)
│   ├── Horizontal Tab Bar (scrollable)
│   └── Content Area (single scroll)
│       └── TabContent
│           ├── AppearanceSettings
│           ├── ChatSettings
│           ├── NotificationSettings
│           ├── ServiceSettings
│           ├── ServersSettings
│           └── KeybindingsSection
│
└── [Desktop Layout]
    ├── Left Nav (200px/236px)
    │   ├── Title + Subtitle
    │   ├── Group: Core (4 tabs)
    │   ├── Group: Advanced (2 tabs)
    │   └── Version Footer
    └── Right Content
        ├── Content Header (title + description + close)
        └── Scroll Area
            └── TabContent (同上)
```

### 6.2 数据流

```
用户操作 → 组件本地 state → Store 方法调用 → Store 更新
                                              ↓
                                    localStorage / IndexedDB 持久化
                                              ↓
                                    useSyncExternalStore 通知
                                              ↓
                                    React 组件重新渲染
```

**防抖保存场景**：

- 自定义 CSS 编辑器：400ms 防抖
- 二进制路径输入：400ms 防抖

**即时保存场景**：

- Toggle 开关：点击即生效
- SegmentedControl：选择即生效
- 快捷键编辑：Enter 确认即生效

### 6.3 懒加载策略

`SettingsDialog` 在 `App.tsx` 中通过 `React.lazy` 懒加载：

```typescript
const SettingsDialog = lazy(() =>
  import('./features/settings/SettingsDialog').then(m => ({ default: m.SettingsDialog })),
)
```

所有子组件（AppearanceSettings、ChatSettings 等）均为同步导入，随 SettingsDialog 一起加载。

### 6.4 Tauri 桥接

服务管理通过动态 import 加载 Tauri API，避免在非 Tauri 环境下引入依赖：

```typescript
const { invoke } = await import('@tauri-apps/api/core')
await invoke('start_opencode_service', { url, binaryPath, envVars })
await invoke('stop_opencode_service')
await invoke<boolean>('check_opencode_service', { url })
await invoke<boolean>('get_service_started_by_us')
```

服务器健康检查在 Tauri 环境下使用 `@tauri-apps/plugin-http` 的 fetch 替代浏览器 fetch。

---

## 7. Error Handling

### 7.1 错误场景与处理

| 场景                  | 处理方式                                   |
| --------------------- | ------------------------------------------ |
| 服务器 URL 格式无效   | 表单内联错误提示（`new URL()` 校验）       |
| 服务器健康检查超时    | 5 秒超时，标记为 `offline`                 |
| 服务器认证失败        | 健康检查返回 `unauthorized` 状态           |
| 快捷键冲突            | 编辑模式下显示红色"已被使用"错误           |
| 自定义音频文件过大    | 上传时验证 2MB 限制，显示错误提示          |
| 自定义音频非音频文件  | MIME 类型验证，显示错误提示                |
| IndexedDB 操作失败    | catch 静默处理，不影响主流程               |
| localStorage 配额超限 | catch 静默处理                             |
| 服务启动失败          | 显示具体错误信息（来自 Tauri invoke 异常） |
| 浏览器拒绝通知权限    | 禁用相关操作并显示提示                     |
| Web Audio API 不可用  | 显示"声音不支持"提示                       |

### 7.2 边界条件

- **无服务器配置**：显示"未配置服务器"空状态
- **快捷键搜索无匹配**：显示"无匹配结果"
- **服务标签页在浏览器环境**：显示"仅桌面端可用"提示
- **当前标签页不可见时**（如从移动端切到桌面端）：`useEffect` 自动切换到第一个可见标签页
- **默认服务器**：不可删除，删除操作返回 `false`

---

## 8. Testing Considerations

### 8.1 单元测试覆盖点

| 模块              | 测试重点                                                                                                   |
| ----------------- | ---------------------------------------------------------------------------------------------------------- |
| `keybindingStore` | 解析/格式化/规范化/比较函数、冲突检测、持久化（仅存差异项）、重置操作                                      |
| `serverStore`     | 添加/更新/删除服务器、URL 尾部斜杠处理、健康检查五种状态、活动服务器切换、sessionStorage/localStorage 双写 |
| `soundStore`      | 设置加载/保存、IndexedDB 读写、音频验证、缓存预加载                                                        |

### 8.2 组件测试覆盖点

| 组件                   | 测试场景                                                             |
| ---------------------- | -------------------------------------------------------------------- |
| `SettingsDialog`       | 默认标签页、标签切换、键盘导航、移动端布局、服务标签页可见性         |
| `KeybindingsSection`   | 搜索过滤、编辑模式、冲突检测、Escape 取消、Enter 确认、重置全部      |
| `AppearanceSettings`   | 主题预设切换、CSS 模板加载、防抖保存、语言切换                       |
| `ChatSettings`         | 沉浸模式联动、路径格式切换、步骤信息开关                             |
| `NotificationSettings` | 通知权限状态、音效选择、音频上传/预览/导出/删除、音量滑块            |
| `ServersSettings`      | 添加/编辑/删除服务器、健康检查状态图标、切换服务器清理会话、跨域警告 |
| `ServiceSettings`      | 非桌面端提示、二进制路径防抖、服务启停、环境变量增删                 |
| `SettingsUI`           | Toggle 开关状态、SegmentedControl 选择、键盘导航                     |

### 8.3 集成测试场景

| 场景             | 验证点                             |
| ---------------- | ---------------------------------- |
| 切换服务器       | 消息清理 + SSE 重连 + 导航回首页   |
| 快捷键全局绑定   | 修改后在应用其他区域生效           |
| 自定义音频持久化 | 刷新页面后音频仍可用               |
| 沉浸模式联动     | 开启后子选项自动调整，关闭后不恢复 |
| 多窗口服务器隔离 | 窗口 A 切换服务器不影响窗口 B      |

### 8.4 手动测试清单

- [ ] 移动端横向滚动标签栏不溢出屏幕
- [ ] 键盘 ArrowUp/ArrowDown 在导航栏中循环移动
- [ ] 快捷键编辑时单独按下 Ctrl/Alt/Shift 不被记录
- [ ] 自定义 CSS 模板加载后修改，400ms 自动应用
- [ ] 服务器健康检查刷新全部按钮
- [ ] 上传超过 2MB 的音频文件显示错误
- [ ] 浏览器拒绝通知权限后测试按钮被禁用
- [ ] 服务启动失败显示具体错误信息

---

## 9. 依赖关系总结

### 9.1 外部依赖

| 依赖                      | 用途                          | 必需性       |
| ------------------------- | ----------------------------- | ------------ |
| `@tauri-apps/api/core`    | Tauri invoke 命令（服务管理） | 桌面环境     |
| `@tauri-apps/plugin-http` | Tauri 环境 HTTP 请求          | 桌面环境     |
| 浏览器 Notification API   | 系统通知                      | 可选         |
| Web Audio API             | 提示音播放                    | 可选         |
| IndexedDB                 | 自定义音频存储                | 声音功能需要 |

### 9.2 内部依赖

| 依赖模块             | 使用内容                                                                   |
| -------------------- | -------------------------------------------------------------------------- |
| `themeStore`         | 主题预设、颜色模式、自定义 CSS、沉浸模式、思考显示、工具卡片样式、步骤信息 |
| `layoutStore`        | 侧边栏文件夹样式、Diff 信息、子会话显示                                    |
| `keybindingStore`    | 快捷键 CRUD、冲突检测                                                      |
| `serverStore`        | 服务器 CRUD、健康检查、活动服务器                                          |
| `soundStore`         | 声音设置、自定义音频管理                                                   |
| `notificationStore`  | Toast 通知开关                                                             |
| `autoApproveStore`   | 自动批准开关                                                               |
| `serviceStore`       | 服务状态、二进制路径、环境变量                                             |
| `messageStore`       | 切换服务器时清理会话消息                                                   |
| `Dialog` 组件        | 模态对话框外壳                                                             |
| `ConfirmDialog` 组件 | 删除服务器确认                                                             |
| `Button` 组件        | 按钮样式                                                                   |
| `Icons`              | 各类图标                                                                   |
| `useRouter`          | 切换服务器时导航                                                           |
| `i18n`               | 所有文本国际化                                                             |

### 9.3 被依赖模块

| 模块                                    | 使用内容                         |
| --------------------------------------- | -------------------------------- |
| 全局快捷键系统（`useKeybindings` hook） | 读取当前快捷键配置并绑定事件监听 |
| 通知系统                                | 读取通知和声音设置               |
| API 通信层                              | 读取服务器配置和健康状态         |
| 主题系统                                | 读取外观设置并应用样式           |
| App 根组件                              | 懒加载 SettingsDialog            |

---

## 10. 风险区域

### 10.1 快捷键编辑的 capture 模式（中风险）

`KeybindingRow` 使用 `document.addEventListener('keydown', handler, { capture: true })` 在捕获阶段拦截按键事件。这可能导致：

- 与其他全局快捷键监听器冲突
- 在编辑模式下，其他组件的键盘事件被意外拦截

**缓解**：编辑模式下通过 `isEditing` 状态精确控制，退出时立即 `removeEventListener`。

### 10.2 服务器切换的级联副作用（中风险）

切换服务器触发：消息清理 → SDK 客户端失效 → SSE 重连 → 导航回首页。这一系列操作涉及多个 store 的协调。

**缓解**：通过 `serverStore.onServerChange()` 回调机制解耦，`main.tsx` 中注册回调统一处理。

### 10.3 IndexedDB 异步加载与同步快照（低风险）

`soundStore.getSnapshot()` 是同步方法，但自定义音频 Blob 需要异步从 IndexedDB 加载。

**缓解**：音频 Blob 不纳入 `getSnapshot()` 返回范围，仅通过 `getCustomAudioBlobAsync()` 异步获取。配置元数据（soundId、customFileName）始终同步可用。

### 10.4 沉浸模式联动的不可逆性（低风险）

开启沉浸模式后自动修改子选项，关闭时不恢复。用户可能期望关闭后恢复原值。

**现状**：这是设计决策而非 bug，spec 中明确要求"关闭沉浸模式后，上述选项保持当前值，不自动恢复"。

---

_生成时间: 2026-04-12_
_模块版本: OpenCodeUI v0.4.8_
