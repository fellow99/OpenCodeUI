# 006-mention-system 技术方案（As-Built）

> 本文档是对已完成的 @ 提及系统模块的回溯性技术规划，记录"实际建成"的架构设计、实现细节与集成策略。
>
> **模块路径**: `src/features/mention/`
> **OpenCodeUI 版本**: v0.4.8
> **技术栈**: React 19 + TypeScript + Tailwind CSS v4

---

## 1. Technical Context

### 1.1 模块定位

@ 提及系统是聊天输入框的增强功能，为用户提供在消息中引用工作区资源（文件、文件夹、Agent）的能力。模块位于 `src/features/mention/`，包含 9 个源文件（7 个实现文件 + 2 个测试文件），总计约 1200 行代码。

### 1.2 文件清单

| 文件                      | 行数 | 领域          | 主要职责                                                                              |
| ------------------------- | ---- | ------------- | ------------------------------------------------------------------------------------- |
| `index.ts`                | 32   | 模块入口      | 统一导出所有子模块（类型、工具、组件、Hook）                                          |
| `types.ts`                | 46   | 类型定义      | MentionType、MentionItem、ParsedSegment、MentionMenuState、MentionConfig 及序列化正则 |
| `utils.ts`                | 215  | 工具函数      | 路径处理、序列化/反序列化、触发检测、颜色配置                                         |
| `useMention.ts`           | 339  | 状态管理 Hook | 菜单状态、提及列表、输入处理、编辑器内容重建                                          |
| `createMentionElement.ts` | 55   | DOM 元素创建  | 创建提及标签 DOM 元素，含点击复制交互                                                 |
| `MentionMenu.tsx`         | 431  | 菜单组件      | 自动补全下拉菜单，支持搜索、浏览、键盘导航                                            |
| `MentionTag.tsx`          | 127  | 标签组件      | 提及标签渲染（React 组件），含 RichText 反序列化                                      |
| `MentionMenu.test.tsx`    | 84   | 组件测试      | MentionMenu 加载与导航测试                                                            |
| `utils.test.ts`           | 51   | 工具测试      | 路径处理、序列化、触发检测等工具函数测试                                              |

### 1.3 外部依赖

| 依赖                                         | 来源                     | 用途                                                       |
| -------------------------------------------- | ------------------------ | ---------------------------------------------------------- |
| `searchFiles`                                | `api/file.ts`            | 全局文件搜索，基于 `sdk.find.files()`                      |
| `listDirectory`                              | `api/file.ts`            | 目录内容列举，基于 `sdk.file.list()`，根目录有 10s 缓存    |
| `ApiAgent`                                   | `api/client.ts`          | Agent 类型定义，用于菜单中展示可用 Agent                   |
| `copyTextToClipboard`                        | `utils/clipboard.ts`     | 剪贴板写入，优先 `navigator.clipboard`，降级 `execCommand` |
| `clipboardErrorHandler` / `fileErrorHandler` | `utils/errorHandling.ts` | 统一错误处理                                               |
| `react-i18next`                              | 第三方                   | 国际化，使用 `commands` 和 `common` 命名空间               |
| `CheckIcon`                                  | `components/Icons`       | 复制成功后的勾选图标                                       |

### 1.4 被依赖关系

| 依赖本模块的模块                       | 使用内容                                                                                                     |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `features/chat/InputBox.tsx`           | 导入 `MentionMenu`、`MentionMenuHandle`、`MentionItem`、`detectMentionTrigger`、`normalizePath`、`toFileUrl` |
| `features/message/MessageRenderer.tsx` | 通过 `RichText` 组件反序列化提及标签                                                                         |
| `features/attachment/utils.ts`         | 提及类型引用                                                                                                 |
| `components/FileExplorer.tsx`          | 提及相关样式                                                                                                 |

---

## 2. Constitution Check

对照项目宪法逐项验证：

| 宪法原则                    | 符合性 | 说明                                                                                                                          |
| --------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------- |
| 原则 1：AI 驱动开发         | 符合   | 代码结构清晰，注释规范，符合 Vibe Coding 范式                                                                                 |
| 原则 2：OpenCode 兼容性优先 | 符合   | 文件搜索使用 `sdk.find.files()`，目录列表使用 `sdk.file.list()`，Agent 数据来自后端 API                                       |
| 原则 3：多平台统一代码库    | 符合   | 无平台特定代码，路径处理兼容 Windows（`C:` 盘符检测）和 Unix                                                                  |
| 原则 4：自定义优于框架依赖  | 符合   | 未引入额外状态管理库，使用 React 原生 `useState`/`useRef`/`useCallback`                                                       |
| 原则 5：实时通信优先        | N/A    | 本模块不涉及 SSE，为纯 UI 交互模块                                                                                            |
| 原则 6：中文优先文档        | 符合   | 代码注释使用中文，i18n 提供 zh-CN 和 en 双语                                                                                  |
| 原则 7：开源与社区驱动      | 符合   | 遵循 GPL-3.0，代码结构清晰可维护                                                                                              |
| 原则 8：零配置用户体验      | 符合   | 提及系统开箱即用，无需额外配置                                                                                                |
| 原则 9：主题与可访问性      | 符合   | 使用 Tailwind 语义化颜色类（`bg-warning-bg`、`text-info-100` 等），支持明暗模式；菜单项使用 `<button>` 元素，天然支持键盘焦点 |
| 原则 10：模块化功能架构     | 符合   | 模块自包含，通过 `index.ts` barrel 文件统一导出，与 chat 模块通过 props 和回调通信                                            |

| 约束条件       | 符合性 | 说明                                        |
| -------------- | ------ | ------------------------------------------- |
| C1：许可证     | 符合   | 无许可证变更                                |
| C2：后端依赖   | 符合   | 仅使用 OpenCode 后端 API                    |
| C3：构建校验   | 符合   | 提供 Vitest 单元测试                        |
| C4：依赖最小化 | 符合   | 仅依赖已有 API 层和工具函数，无新增第三方库 |
| C5：SSE 支持   | N/A    | 本模块不涉及 SSE                            |

---

## 3. Research Findings

### 3.1 触发检测机制

`detectMentionTrigger()` 函数在 `utils.ts` 中实现，核心逻辑：

1. 截取光标前的文本 `textBeforeCursor`
2. 查找最后一个 `@` 的位置 `lastTriggerIndex`
3. 检查 `@` 之前的字符：必须是空格、NBSP（`\u00A0`）、换行符（`\n`）或文本开头
4. 检查 `@` 之后到光标之间：不能包含空格或 NBSP，否则视为触发已中断
5. 返回 `{ startIndex, query }` 或 `null`

该函数被两处调用：

- `useMention.handleInput()`：在每次输入事件时检测
- `InputBox.tsx`：在输入框的输入处理中直接调用（用于与斜杠命令触发互斥判断）

### 3.2 菜单数据加载策略

MentionMenu 组件采用双路径数据加载：

**路径 A：目录浏览**（query 包含 `/` 或为空）

- 调用 `listDirectory(path, rootPath)` 获取目录内容
- 根目录（`path === '.'`）有 10 秒缓存 + inflight 去重
- 结果按 Agent、文件夹、文件三组排序
- Agent 仅在根目录显示，过滤条件为 `!a.hidden && a.mode === 'subagent'`

**路径 B：全局搜索**（query 不包含 `/` 且非空）

- 调用 `searchFiles(query, { limit: 20, directory: rootPath })`
- 搜索结果通过文件名是否包含 `.` 或路径是否以 `/` 结尾来区分文件/文件夹
- Agent 同样参与搜索过滤

**竞态条件防护**：使用 `requestIdRef` 递增计数器，每次请求前递增，响应时检查 ID 是否匹配，不匹配则丢弃结果。

### 3.3 动态高度计算

菜单最大高度通过 `useLayoutEffect` + `requestAnimationFrame` 计算：

```
available = parentRect.top - 56(header) - 16(安全间距) - 8(marginBottom)
```

- 当 `0 < available < 360` 时，使用计算值
- 否则使用默认值 `min(320px, calc(100dvh - 10rem))`
- 监听 `window.resize` 和 `window.visualViewport.resize`（移动端键盘弹出/收起）

### 3.4 编辑器内容重建

选择提及项后，`useMention.handleSelect()` 执行以下步骤：

1. 从 `menuState` 获取 `@` 的起始位置和查询字符串
2. 将纯文本分割为 `beforeAt`（`@` 之前）和 `afterQuery`（查询之后）
3. 调用 `rebuildEditorContent()` 重建 DOM：
   - 清空 `editor.innerHTML`
   - 添加 `beforeAt` 文本节点
   - 通过 `createMentionElement()` 创建提及标签并插入
   - 添加空格 + `afterQuery` 文本节点
4. 更新 `mentions` 状态列表
5. 关闭菜单，聚焦编辑器，光标移至末尾

**注意**：`rebuildEditorContent()` 接收 `_existingMentions` 参数但当前未使用（前缀 `_` 表示有意忽略）。这意味着每次选择新提及项时，编辑器内容会被完全重建，之前插入的提及标签不会保留在 DOM 中。这是一个已知的实现限制。

### 3.5 序列化格式

序列化格式为 `[[type:value]]`，其中：

- `type`：`agent`、`file`、`folder`
- `value`：绝对路径（文件/文件夹）或 Agent 名称

正则表达式：`/\[\[(agent|file|folder):([^\]]+)\]\]/g`

提供四个操作函数：

- `serializeMention(item)`：单个 MentionItem 转序列化字符串
- `parseMentions(text)`：序列化文本转 `ParsedSegment[]`（文本和提及交替数组）
- `extractMentions(text)`：从序列化文本提取 `MentionItem[]`
- `stripMentions(text)`：移除所有提及标记，返回纯文本

### 3.6 与 InputBox 的集成方式

InputBox 组件直接使用提及模块的导出，而非通过 `useMention` Hook：

```typescript
import {
  MentionMenu,
  detectMentionTrigger,
  normalizePath,
  toFileUrl,
  type MentionMenuHandle,
  type MentionItem,
} from '../mention'
```

InputBox 自行管理提及状态（`mentionMenuOpen`、`mentionQuery`、`mentionStartIndex`、`selectedMentions`），并通过 `useRef<MentionMenuHandle>` 控制菜单的键盘导航。这种设计让 InputBox 拥有对提及流程的完全控制权，同时协调 @ 提及与 / 斜杠命令的互斥关系。

---

## 4. Data Model

### 4.1 核心类型

```typescript
type MentionType = 'agent' | 'file' | 'folder'

interface MentionItem {
  type: MentionType
  value: string // 完整值（绝对路径或 agent name）
  displayName: string // 显示名称（文件名或 agent name）
  relativePath?: string // 相对路径（用于菜单中显示）
}

interface ParsedSegment {
  type: 'text' | 'mention'
  content: string
  mentionType?: MentionType // 仅 mention 片段
  mentionValue?: string // 仅 mention 片段
}

interface MentionMenuState {
  isOpen: boolean
  query: string // @ 之后的搜索词
  startIndex: number // @ 在文本中的索引位置
  position?: { x: number; y: number }
}

interface MentionConfig {
  trigger?: string // 触发字符，默认 '@'
  rootPath?: string // 项目根路径
  allowMultiple?: boolean // 是否允许多个提及（当前未实现）
}
```

### 4.2 序列化正则

```typescript
const MENTION_PATTERN = /\[\[(agent|file|folder):([^\]]+)\]\]/g
```

### 4.3 颜色配置

```typescript
const MENTION_COLORS: Record<
  MentionType,
  {
    bg: string // Tailwind 背景类
    text: string // Tailwind 文字类
    border: string // Tailwind 边框类
    darkText: string // 暗色模式文字类（当前为空）
  }
>
```

| 类型     | bg              | text               | border                  |
| -------- | --------------- | ------------------ | ----------------------- |
| `agent`  | `bg-warning-bg` | `text-warning-100` | `border-warning-100/20` |
| `file`   | `bg-info-bg`    | `text-info-100`    | `border-info-100/20`    |
| `folder` | `bg-success-bg` | `text-success-100` | `border-success-100/20` |

### 4.4 菜单控制器接口（Imperative Handle）

```typescript
interface MentionMenuHandle {
  moveUp(): void // 选中上一项
  moveDown(): void // 选中下一项
  selectCurrent(): void // 确认选中当前项
  enterFolder(): void // 进入当前选中的文件夹
  goBack(): void // 返回上级目录
  getSelectedItem(): MentionItem | null // 获取当前选中项
  setRestoreFolder(name: string): void // 设置返回上级时定位的文件夹名
}
```

### 4.5 useMention Hook 接口

```typescript
interface UseMentionOptions extends MentionConfig {
  editorRef: React.RefObject<HTMLDivElement | null>
  onTextChange?: (text: string) => void
  onMenuClose?: () => void
}

interface UseMentionReturn {
  menuState: MentionMenuState
  mentions: MentionItem[]
  openMenu: (query: string, startIndex: number) => void
  closeMenu: () => void
  handleInput: (e: React.FormEvent<HTMLDivElement>) => string
  handleSelect: (item: MentionItem) => void
  reset: () => void
  getSerializedText: () => string
}
```

### 4.6 MentionMenu 组件 Props

```typescript
interface MentionMenuProps {
  isOpen: boolean
  query: string // @ 之后的搜索词
  agents: ApiAgent[] // 可用 Agent 列表
  rootPath?: string // 项目根路径
  excludeValues?: Set<string> // 需要排除的项（已选择的）
  onSelect: (item: MentionItem) => void
  onNavigate?: (folderPath: string) => void // 移动端文件夹导航
  onClose: () => void
}
```

---

## 5. Interface Contracts

### 5.1 与 API 层的契约

| 调用方      | 被调用方        | 接口                             | 输入                                                  | 输出                               |
| ----------- | --------------- | -------------------------------- | ----------------------------------------------------- | ---------------------------------- |
| MentionMenu | `api/file.ts`   | `searchFiles(query, options)`    | query: string, options: { directory?, type?, limit? } | `Promise<string[]>` 文件路径数组   |
| MentionMenu | `api/file.ts`   | `listDirectory(path, directory)` | path: string, directory?: string                      | `Promise<FileNode[]>` 目录节点数组 |
| MentionMenu | `api/client.ts` | `ApiAgent[]`（props 传入）       | 由 InputBox 从后端获取后传入                          | 数组                               |

### 5.2 与 InputBox 的契约

InputBox 作为消费方，向 MentionMenu 提供以下数据：

| 数据流          | 方向                   | 内容                     |
| --------------- | ---------------------- | ------------------------ |
| `isOpen`        | InputBox → MentionMenu | 菜单是否打开             |
| `query`         | InputBox → MentionMenu | `@` 之后的搜索词         |
| `agents`        | InputBox → MentionMenu | 可用 Agent 列表          |
| `rootPath`      | InputBox → MentionMenu | 项目根路径               |
| `excludeValues` | InputBox → MentionMenu | 已选提及项的 value 集合  |
| `onSelect`      | MentionMenu → InputBox | 用户选中某一项的回调     |
| `onNavigate`    | MentionMenu → InputBox | 文件夹导航请求（移动端） |
| `onClose`       | MentionMenu → InputBox | 菜单关闭回调             |

### 5.3 与 i18n 的契约

使用 `useTranslation(['commands', 'common'])`，涉及的翻译键：

| 命名空间   | 键                  | 中文       | 英文          |
| ---------- | ------------------- | ---------- | ------------- |
| `commands` | `mention.goBack`    | 返回       | Go back       |
| `commands` | `mention.agent`     | Agent      | Agent         |
| `commands` | `mention.file`      | 文件       | File          |
| `commands` | `mention.folder`    | 文件夹     | Folder        |
| `common`   | `loading`           | 加载中...  | Loading...    |
| `common`   | `noResults`         | 无结果     | No results    |
| `common`   | `emptyFolder`       | 空目录     | Empty folder  |
| `common`   | `upDownSelect`      | ↑↓ 选择    | ↑↓ Select     |
| `common`   | `enterConfirmShort` | Enter 确认 | Enter Confirm |
| `common`   | `escCancel`         | Esc 关闭   | Esc Cancel    |

### 5.4 DOM 数据属性

提及标签 DOM 元素使用以下 `data-*` 属性存储元数据：

| 属性                   | 值                          | 用途                 |
| ---------------------- | --------------------------- | -------------------- |
| `data-mention-type`    | `agent` / `file` / `folder` | 提及类型             |
| `data-mention-value`   | 绝对路径或 Agent 名称       | 完整值               |
| `data-mention-display` | 文件名或 Agent 名称         | 显示名称             |
| `class`                | 包含 `mention-tag`          | 序列化时识别提及元素 |

---

## 6. Implementation Strategy

### 6.1 模块架构

```
src/features/mention/
├── index.ts                  ← Barrel 文件，统一导出
├── types.ts                  ← 类型定义 + 序列化正则
├── utils.ts                  ← 纯函数工具（路径、序列化、触发检测、颜色）
├── useMention.ts             ← React Hook（状态管理 + 编辑器交互）
├── createMentionElement.ts   ← 原生 DOM 操作（创建提及标签元素）
├── MentionMenu.tsx           ← React 组件（自动补全菜单）
├── MentionTag.tsx            ← React 组件（标签渲染 + RichText 反序列化）
├── MentionMenu.test.tsx      ← 组件测试
└── utils.test.ts             ← 工具函数测试
```

### 6.2 数据流

```
用户输入 @ 字符
    │
    ▼
detectMentionTrigger() 检测触发条件
    │
    ├── 有效触发 → openMenu(query, startIndex)
    │               │
    │               ▼
    │         MentionMenu 加载数据
    │         ├── 目录浏览: listDirectory()
    │         └── 全局搜索: searchFiles()
    │               │
    │               ▼
    │         渲染菜单列表（Agent → 文件夹 → 文件）
    │               │
    │               ▼
    │         用户选中项 → onSelect(item)
    │               │
    │               ▼
    │         handleSelect(item)
    │         ├── toAbsolutePath() 转换路径
    │         ├── rebuildEditorContent() 重建 DOM
    │         ├── createMentionElement() 创建标签
    │         └── closeMenu()
    │
    └── 无效触发 → closeMenu()
```

### 6.3 关键实现细节

#### 6.3.1 请求 ID 竞态防护

```typescript
const requestIdRef = useRef(0)

const loadDirectory = (path, filter) => {
  const requestId = ++requestIdRef.current  // 递增
  // ... 发起请求
  .then(nodes => {
    if (requestId !== requestIdRef.current) return  // 丢弃过期结果
    // ... 处理结果
  })
}
```

#### 6.3.2 根目录缓存

`listDirectory()` 在 `api/file.ts` 中实现，对根目录请求使用 10 秒 TTL 缓存 + inflight 去重：

```typescript
const ROOT_DIRECTORY_CACHE_TTL_MS = 10_000
const rootDirectoryCache = new Map<string, { data: FileNode[]; expiresAt: number }>()
const rootDirectoryInflight = new Map<string, Promise<FileNode[]>>()
```

缓存键由 `serverStore.getActiveServerId()` + `directory` 组成，确保切换服务器后缓存自动失效。

#### 6.3.3 文件夹导航与返回定位

- 点击文件夹时，通过 `onNavigate(basePath + '/')` 通知父组件更新 query
- 点击"返回上级"时，先保存当前文件夹名到 `restoreFolderRef`，导航回父目录后自动定位到该文件夹
- `setRestoreFolder()` 通过 `useImperativeHandle` 暴露给父组件

#### 6.3.4 提及标签点击复制

两种实现路径：

1. **`createMentionElement()`**（用于 contentEditable 编辑器内）：
   - 直接操作 DOM，`span.addEventListener('click', handleClick)`
   - 复制成功后用 innerHTML 替换显示勾选图标
   - 1200ms 后恢复原文
   - 提供 `cleanup()` 函数移除事件监听和清除 timeout

2. **`MentionTag` 组件**（用于 RichText 反序列化渲染）：
   - React 组件，使用 `useState` 管理 `copied` 状态
   - 复制成功后显示 `CheckIcon` 组件
   - 1500ms 后恢复原文
   - 通过 `useEffect` 清理 timeout

#### 6.3.5 路径处理

| 函数             | 输入                      | 输出                          | 示例             |
| ---------------- | ------------------------- | ----------------------------- | ---------------- |
| `normalizePath`  | `foo\bar//baz.ts`         | `foo/bar/baz.ts`              | 统一分隔符       |
| `getFileName`    | `/project/src/main.ts`    | `main.ts`                     | 提取文件名       |
| `toAbsolutePath` | `src/main.ts`, `/project` | `/project/src/main.ts`        | 相对转绝对       |
| `toFileUrl`      | `/project/src/main.ts`    | `file:///project/src/main.ts` | 生成 file:// URL |
| `toFileUrl`      | `C:/repo/app.ts`          | `file:///C:/repo/app.ts`      | Windows 路径     |

### 6.4 响应式适配

| 场景           | 策略                                                       |
| -------------- | ---------------------------------------------------------- |
| 桌面端         | 默认 `maxHeight: min(320px, calc(100dvh - 10rem))`         |
| 小屏幕         | 动态计算 `parentRect.top - 56 - 16 - 8`，限制在 360px 以下 |
| 移动端键盘弹出 | 监听 `visualViewport.resize`，重新计算可用高度             |
| 窗口 resize    | 监听 `window.resize`，重新计算                             |
| 键盘操作提示   | `hidden md:flex`，仅在 md 断点以上显示                     |

---

## 7. Error Handling

### 7.1 错误处理模式

模块使用预定义的错误处理器（来自 `utils/errorHandling.ts`）：

| 场景           | 处理器                                       | 类别        |
| -------------- | -------------------------------------------- | ----------- |
| 目录列举失败   | `fileErrorHandler('list directory', err)`    | `file`      |
| 文件搜索失败   | `fileErrorHandler('file search', err)`       | `file`      |
| 剪贴板复制失败 | `clipboardErrorHandler('copy mention', err)` | `clipboard` |

### 7.2 竞态条件处理

- **请求 ID 机制**：每次搜索/加载前递增 `requestIdRef`，响应时检查 ID 匹配性
- **requestAnimationFrame 取消**：搜索逻辑包裹在 `requestAnimationFrame` 中，cleanup 时调用 `cancelAnimationFrame`

### 7.3 内存泄漏防护

- `createMentionElement()` 返回 `cleanup()` 函数，移除事件监听和清除 timeout
- `MentionTag` 组件通过 `useEffect` 清理 timeout
- 菜单的 `pointerdown` 事件监听在 `isOpen` 变化时正确移除
- `visualViewport` 监听器在组件卸载时移除

### 7.4 边界情况处理

| 场景               | 处理方式                                       |
| ------------------ | ---------------------------------------------- |
| 搜索无结果         | 显示 `common:noResults` 提示                   |
| 空目录             | 显示 `common:emptyFolder` 提示                 |
| 加载中             | 显示 `common:loading` 动画                     |
| 菜单打开但无父容器 | `setDynamicMaxHeight(undefined)`，使用默认值   |
| 编辑器 ref 为空    | `handleSelect` 和 `getSerializedText` 提前返回 |
| 已选项排除         | 通过 `excludeValues?.has(item.value)` 过滤     |

---

## 8. Testing Considerations

### 8.1 现有测试覆盖

**`utils.test.ts`**（5 个测试用例）：

| 测试                    | 覆盖内容                                                                     |
| ----------------------- | ---------------------------------------------------------------------------- |
| 路径标准化与文件名提取  | `normalizePath('foo\\bar//baz.ts')` → `'foo/bar/baz.ts'`                     |
| 相对路径转换与 file URL | `toAbsolutePath`、`toFileUrl` 的多种输入                                     |
| 序列化与解析            | `serializeMention`、`parseMentions` 往返转换                                 |
| 提及提取与剥离          | `extractMentions`、`stripMentions`                                           |
| 触发检测                | 有效触发（`hello @src/com`）、无效触发（`email@test.com`、`hello @src com`） |

**`MentionMenu.test.tsx`**（2 个测试用例）：

| 测试                     | 覆盖内容                       |
| ------------------------ | ------------------------------ |
| 打开时加载根目录和 Agent | 验证 Agent、文件夹、文件均显示 |
| 通过面包屑导航返回       | 验证 `onNavigate` 调用参数正确 |

### 8.2 测试缺口

以下功能场景缺少测试覆盖：

| 缺口                            | 优先级 | 说明                                            |
| ------------------------------- | ------ | ----------------------------------------------- |
| `detectMentionTrigger` 边界情况 | P1     | NBSP 支持、文本开头直接输入 `@`、`@` 后紧跟空格 |
| `useMention` Hook               | P0     | 核心状态管理逻辑，无独立测试                    |
| `createMentionElement`          | P1     | DOM 元素创建、点击复制交互                      |
| `MentionTag` 组件               | P1     | 渲染、点击复制、RichText 反序列化               |
| MentionMenu 键盘导航            | P1     | moveUp/moveDown/selectCurrent                   |
| MentionMenu 文件夹进入/返回     | P1     | enterFolder/goBack                              |
| MentionMenu 已选项排除          | P2     | `excludeValues` 过滤逻辑                        |
| MentionMenu 动态高度计算        | P2     | resize 事件、visualViewport 变化                |
| 序列化往返一致性                | P1     | `serializeMention` → `parseMentions` → 重建     |
| `toAbsolutePath` Windows 路径   | P1     | `C:\Users\test` 标准化                          |

### 8.3 建议测试策略

1. **优先补充 `useMention` Hook 测试**：这是模块的核心状态管理逻辑，覆盖 `handleInput`、`handleSelect`、`reset`、`getSerializedText` 四个公开方法
2. **使用 Vitest + Testing Library**：与现有测试框架保持一致
3. **Mock API 调用**：使用 `vi.mock('../../api/client')` 模拟 `searchFiles` 和 `listDirectory`
4. **Fake timers**：使用 `vi.useFakeTimers()` 控制 `requestAnimationFrame` 和 `setTimeout`
5. **DOM 环境**：提及标签测试需要 jsdom 环境，确保 `document.createElement` 和 `window.getSelection` 可用

---

## 9. 风险与已知限制

### 9.1 已选提及项在编辑器重建时丢失

`rebuildEditorContent()` 接收 `_existingMentions` 参数但未使用。每次选择新提及项时，编辑器通过 `editor.innerHTML = ''` 完全清空后重建，之前插入的提及标签 DOM 元素不会被保留。这意味着：

- 用户无法在一条消息中插入多个提及标签
- `allowMultiple` 配置项虽然定义在 `MentionConfig` 中，但实际未实现（代码中标注了 `// TODO`）

**缓解**：当前 InputBox 的使用场景中，用户通常一次只引用一个资源。如需多提及支持，需要修改 `rebuildEditorContent()` 以保留已有的 `.mention-tag` 元素。

### 9.2 文件夹/文件区分依赖文件名启发式

搜索结果的类型判断使用 `!name.includes('.') || path.endsWith('/')` 来区分文件夹和文件。这意味着：

- 无扩展名的文件（如 `Makefile`、`LICENSE`）会被误判为文件夹
- 以 `/` 结尾的路径被视为文件夹

**缓解**：目录浏览模式下，`listDirectory` 返回的 `FileNode` 包含明确的 `type: 'directory' | 'file'` 字段，不会误判。仅在搜索模式下存在此限制。

### 9.3 光标位置计算复杂度

`getCursorPosition()` 通过遍历 DOM 树计算光标在纯文本中的位置，需要处理文本节点和 `mention-tag` 元素。在复杂编辑器内容下可能存在性能问题。

### 9.4 序列化文本与纯文本的不一致

`useMention` 同时维护 `plainText`（用于触发检测）和 `serializedText`（用于发送）。两者通过不同的 DOM 遍历函数获取，在提及标签的 `textContent` 与序列化格式之间存在映射关系。如果 DOM 结构异常，可能导致两者不一致。

---

## 10. 与 overall-plan.md 的对齐

根据总体技术规划，006-mention-system 的核心设计点与实际实现的对照：

| 规划要点                      | 实现状态 | 说明                                            |
| ----------------------------- | -------- | ----------------------------------------------- |
| MentionMenu 弹出式提及菜单    | 已实现   | 支持文件路径搜索、Agent 展示、目录浏览          |
| MentionTag 渲染提及标签       | 已实现   | 支持点击复制、颜色区分、RichText 反序列化       |
| useMention Hook 管理状态      | 已实现   | 菜单状态、提及列表、输入处理、内容重建          |
| createMentionElement DOM 节点 | 已实现   | 自定义 `data-mention-*` 属性存储元数据          |
| 提及解析为结构化数据          | 已实现   | `[[type:value]]` 格式，提供 parse/extract/strip |

总体依赖关系图中，006-mention-system 被 002-chat-feature 和 003-message-rendering 依赖，依赖 001-api-layer、010-state-management、013-i18n。实际代码中：

- 依赖 `api/file.ts`（searchFiles、listDirectory）和 `api/client.ts`（ApiAgent 类型）
- 间接依赖 `store/serverStore`（通过 `listDirectory` 的缓存键）
- 依赖 `react-i18next` 进行国际化
- 不直接依赖 state management store，状态由组件和 Hook 内部管理

---

_生成时间: 2026-04-12_
_基于 OpenCodeUI v0.4.8 实际代码回溯_
