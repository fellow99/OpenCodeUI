# 011-file-diff-viewer 模块技术方案（As-Built）

> 本文档是对已完成模块的回溯性技术规划，记录"实际建成"的架构设计、组件分解、数据流与集成策略。
>
> **模块路径**: `src/components/DiffViewer.tsx`, `src/components/DiffView.tsx`, `src/components/SessionChangesPanel.tsx`, `src/components/FileExplorer.tsx`, `src/utils/diffUtils.ts`, `src/store/changeScopeStore.ts`
>
> **版本**: 1.0
>
> **状态**: 已实现

---

## 1. Technical Context

### 1.1 模块定位

文件 Diff 查看器模块是 OpenCodeUI 中负责所有代码变更可视化的核心模块。用户通过该模块查看 AI 在会话中对工作区文件所做的修改，覆盖从聊天消息内嵌展示到全屏多文件浏览的完整场景链。

### 1.2 技术栈

| 维度      | 选型                                       |
| --------- | ------------------------------------------ |
| 框架      | React 19 + TypeScript                      |
| 行级 diff | `diff` 库（`diffLines`、`diffWords`）      |
| 语法高亮  | Shiki（通过 `src/lib/shiki` 封装）         |
| 虚拟滚动  | 自研固定行高 + `useDynamicVirtualScroll`   |
| 样式      | Tailwind CSS v4                            |
| 国际化    | i18next（`components`、`common` 命名空间） |
| 状态管理  | 自定义 Store（`changeScopeStore`）         |
| 全屏容器  | `FullscreenViewer` 组件                    |
| 多标签页  | `PreviewTabsBar` 组件                      |

### 1.3 源文件清单

| 文件                                          | 行数 | 职责                                                                       |
| --------------------------------------------- | ---- | -------------------------------------------------------------------------- |
| `src/components/DiffViewer.tsx`               | 1423 | 核心 diff 渲染引擎，split/unified 双模式，虚拟滚动，词级高亮，context 折叠 |
| `src/components/DiffView.tsx`                 | 170  | 轻量级内嵌 diff 组件，聊天消息中使用，含折叠/全屏入口                      |
| `src/components/SessionChangesPanel.tsx`      | 1052 | 会话变更面板，变更范围选择、文件列表（树形/扁平）、diff 预览区             |
| `src/components/FileExplorer.tsx`             | 982  | 文件浏览器，文件树（含 git 状态颜色）+ 文件内容预览                        |
| `src/utils/diffUtils.ts`                      | 28   | Unified diff 字符串解析为 before/after                                     |
| `src/store/changeScopeStore.ts`               | 46   | 变更范围模式状态管理（session/turn/git/branch）                            |
| `src/components/DiffViewer.test.tsx`          | -    | DiffViewer 单元测试                                                        |
| `src/components/DiffView.test.tsx`            | -    | DiffView 单元测试                                                          |
| `src/components/SessionChangesPanel.test.tsx` | -    | SessionChangesPanel 单元测试                                               |

### 1.4 关键发现：Spec 与实际的差异

Spec 中提到的 `DiffModal.tsx` 和 `MultiFileDiffModal.tsx` 在代码库中**不存在为独立文件**。其功能已被整合：

- **单文件全屏 diff**: 由 `DiffView.tsx` 内置的 `FullscreenViewer` 实现，无需独立组件
- **多文件 diff 浏览**: 由 `SessionChangesPanel.tsx` 承担，提供文件列表 + diff 预览的完整体验

这意味着实际组件数量比 spec 少 2 个，但功能覆盖完全一致。

---

## 2. Constitution Check

### 2.1 原则对齐

| 宪法原则                    | 对齐情况 | 说明                                                                                                  |
| --------------------------- | -------- | ----------------------------------------------------------------------------------------------------- |
| 原则 2: OpenCode 兼容性优先 | 符合     | 通过 `@opencode-ai/sdk` 调用 `getSessionDiff`、`getVcsDiff`、`getLastTurnDiff` 等 API，与后端规范一致 |
| 原则 3: 多平台统一代码库    | 符合     | 所有组件为纯 React 组件，无平台特定代码，Web 与 Tauri 共享                                            |
| 原则 4: 自定义优于框架依赖  | 符合     | 虚拟滚动自研、状态管理自定义 Store、路由自定义 Hook，仅 `diff` 库和 Shiki 为必要外部依赖              |
| 原则 6: 中文优先文档        | 符合     | 代码注释以中文为主，i18n 键值支持中英文                                                               |
| 原则 10: 模块化功能架构     | 符合     | DiffViewer、DiffView、SessionChangesPanel、FileExplorer 各自独立，通过明确接口通信                    |

### 2.2 约束检查

| 约束           | 状态 | 说明                                                        |
| -------------- | ---- | ----------------------------------------------------------- |
| C3: 构建校验   | 符合 | 模块包含 `.test.tsx` 测试文件，纳入 `npm run validate` 流程 |
| C4: 依赖最小化 | 符合 | 仅引入 `diff` 库（行级/词级 diff 算法），无功能重叠依赖     |

---

## 3. Research Findings

### 3.1 DiffViewer 架构

`DiffViewer.tsx` 是整个模块的核心，1423 行代码实现了完整的 diff 渲染管线。其架构特点：

**两列布局（Gutter + Content）**:

- Gutter 列：固定宽度，包含 change bar（3px 竖条）和行号，不参与水平滚动
- Content 列：代码内容区域，独立水平滚动
- 与 `CodePreview` 组件保持一致的布局范式

**四种渲染变体**:

1. `SplitDiffView` — 固定行高，并排模式
2. `WrappedSplitDiffView` — 动态行高，并排 + 自动换行
3. `UnifiedDiffView` — 固定行高，统一模式
4. `WrappedUnifiedDiffView` — 动态行高，统一 + 自动换行

主组件 `DiffViewer` 根据 `viewMode` 和 `wordWrap` 参数自动路由到对应变体。

**虚拟滚动策略**:

- 固定行高模式：手动计算 `startIndex`/`endIndex`/`offsetY`，`LINE_HEIGHT = 20px`，`OVERSCAN = 5`
- 动态行高模式：使用 `useDynamicVirtualScroll` Hook，通过 ref callback 测量实际行高
- 面板 resize 期间跳过虚拟滚动重计算，避免性能浪费

**Proxy 横向滚动条**:
Split 和 Unified 模式中，content 区域隐藏自身滚动条（`scrollbar-none`），底部使用 sticky proxy scrollbar 实现可见的横向滚动。content 与 proxy scrollbar 之间通过 `scrollSourceRef` guard 防止循环同步。

### 3.2 Diff 计算管线

```
before 文本 + after 文本
        │
        ▼
    diffLines() (来自 diff 库)
        │
        ▼
    computePairedLines() / computeUnifiedLines()
        │
        ├──► split 模式: PairedLine[]
        │       └── 配对删除/新增行，计算词级 diff
        │
        └──► unified 模式: UnifiedLine[]
                └── 按顺序排列所有行

        │
        ▼
    collapseContextPaired() / collapseContextUnified()
        │
        └── 折叠连续 context 行 ──► (PairedLine | CollapsedPairedLine)[]

        │
        ▼
    虚拟滚动 ──► 仅渲染可视区域内的行

        │
        ▼
    LineContent 渲染
        ├── highlightedContent (词级 diff HTML)
        ├── tokens (语法高亮 token)
        └── 纯文本 (降级)
```

### 3.3 词级 Diff 算法

词级 diff 使用 `diff` 库的 `diffWords` 函数，关键优化：

1. **相邻同类型变更合并**: 将连续的 added/removed 词合并，减少碎片
2. **空白字符合并**: 如果两个变更之间仅有空白且两侧变更类型相同，将空白合并到前一个变更中
3. **碎片化检测**: `isTooFragmented()` 函数计算公共内容占比，当 `totalLength > 10 && commonLength / totalLength < 0.4` 时跳过词级高亮
4. **Resize 期间跳过**: `isResizing` 为 true 时不计算词级 diff

### 3.4 Context 折叠机制

- 阈值：连续 context 行超过 `CONTEXT_LINES * 2 + 2 = 8` 行时触发折叠
- 保留：变更行前后各保留 `CONTEXT_LINES = 3` 行 context
- 展开状态：使用 `Set<number>` 存储起始索引 ID，O(1) 查询
- 折叠条：显示 "N lines unchanged"，点击展开

### 3.5 SessionChangesPanel 数据流

```
SessionChangesPanel 挂载
        │
        ▼
    loadProjectState() ──► 获取项目信息 + VCS 信息
        │
        ▼
    确定可用 changeOptions (session/turn/git/branch)
        │
        ▼
    useSessionChangeScope(sessionId) ──► 从 changeScopeStore 读取当前模式
        │
        ▼
    loadDiffMode(mode) ──► 调用对应 API 获取 diff 数据
        │
        ├──► git/branch: getVcsDiff(mode, directory)
        ├──► session: getSessionDiff(sessionId, directory)
        └──► turn: getLastTurnDiff(sessionId, directory)
        │
        ▼
    数据缓存：loadedModes 标记，已加载的模式不重复请求
        │
        ▼
    buildChangesTree(diffs) ──► 树形结构（useMemo 缓存）
        │
        ▼
    reconcileDiffPreviewState() ──► 校准选中文件和高亮状态
```

### 3.6 请求 ID 防过期机制

`SessionChangesPanel` 使用 `diffRequestIdRef` 为每种模式维护独立的请求 ID 计数器。每次发起请求前递增，响应返回时比对 ID，丢弃过期结果。这解决了快速切换模式时旧请求覆盖新请求的问题。

### 3.7 FileExplorer 变更状态指示

`FileExplorer.tsx` 通过 `useFileExplorer` Hook 获取 `fileStatus` Map，在 `FileTreeItem` 中应用状态颜色：

- `added` → `text-success-100`（绿色）
- `modified` → `text-warning-100`（黄色）
- `deleted` → `text-danger-100`（红色）
- `ignored` → `opacity-50`（降低透明度）

路径兼容性：`node.path` 可能使用反斜杠（Windows），而 `fileStatus` Map 的 key 统一使用正斜杠，因此查询时尝试两种格式。

---

## 4. Data Model

### 4.1 核心类型定义（来自实际代码）

```typescript
// DiffViewer.tsx
type ViewMode = 'split' | 'unified'
type LineType = 'add' | 'delete' | 'context' | 'empty'

interface DiffLine {
  type: LineType
  content: string
  lineNo?: number
  highlightedContent?: string // 词级 diff 高亮后的 HTML
}

interface PairedLine {
  left: DiffLine
  right: DiffLine
}

interface CollapsedPairedLine {
  collapsed: true
  count: number
  id: number // 原始行数组中的起始索引
}

type PairedLineOrCollapsed = PairedLine | CollapsedPairedLine

interface UnifiedLine extends DiffLine {
  oldLineNo?: number
  newLineNo?: number
}

interface CollapsedUnifiedLine {
  collapsed: true
  count: number
  id: number
}

type UnifiedLineOrCollapsed = UnifiedLine | CollapsedUnifiedLine
```

### 4.2 DiffViewer Props

```typescript
interface DiffViewerProps {
  before: string // 修改前内容
  after: string // 修改后内容
  language?: string // 语言参数，默认 'text'
  viewMode?: ViewMode // 视图模式，默认 'split'
  maxHeight?: number // 最大高度
  isResizing?: boolean // 是否正在 resize
  wordWrap?: boolean // 是否自动换行（不传则读取 themeStore.codeWordWrap）
}
```

### 4.3 DiffView Props

```typescript
interface DiffViewProps {
  diff?: string // Unified diff 格式字符串
  before?: string // 修改前内容
  after?: string // 修改后内容
  filePath?: string // 文件路径（用于语言检测和标题）
  defaultCollapsed?: boolean // 默认折叠状态
  maxHeight?: number // 最大高度，默认 300
  language?: string // 显式指定语言
}
```

### 4.4 SessionChangesPanel Props

```typescript
interface SessionChangesPanelProps {
  sessionId: string // 会话 ID
  directory?: string // 工作目录
  isResizing?: boolean // 外部面板是否正在 resize
}
```

### 4.5 FileExplorer Props

```typescript
interface FileExplorerProps {
  panelTabId: string // 面板标签 ID
  directory?: string // 工作目录
  previewFile: PreviewFile | null // 当前预览文件
  previewFiles: PreviewFile[] // 已打开的预览文件列表
  position?: 'bottom' | 'right' // 面板位置
  isPanelResizing?: boolean // 面板是否正在 resize
  sessionId?: string | null // 会话 ID（用于 git 状态）
}
```

### 4.6 ChangeScopeStore 状态

```typescript
type ChangeScopeMode = 'git' | 'branch' | 'session' | 'turn'

// 内部存储
Map<sessionId, ChangeScopeMode> // 按会话 ID 隔离
```

### 4.7 ChangesTreeNode（SessionChangesPanel 内部）

```typescript
interface ChangesTreeNode {
  name: string
  path: string
  type: 'file' | 'directory'
  diff?: FileDiff
  children: ChangesTreeNode[]
  additions: number
  deletions: number
  status?: FileStatus // 'added' | 'modified' | 'deleted'
}
```

### 4.8 FileStatus 推断逻辑

`getFileStatus()` 函数按以下优先级推断文件状态：

1. 优先使用 `diff.status` 字段（后端提供）
2. `deletions === 0 && additions > 0` → `added`
3. `additions === 0 && deletions > 0` → `deleted`
4. 旧版兼容：`!before.trim()` → `added`，`!after.trim()` → `deleted`
5. 默认 → `modified`

---

## 5. Interface Contracts

### 5.1 对外导出

| 导出项                          | 来源文件                  | 类型                               |
| ------------------------------- | ------------------------- | ---------------------------------- |
| `DiffViewer`                    | `DiffViewer.tsx`          | React Component                    |
| `DiffView`                      | `DiffView.tsx`            | React Component                    |
| `SessionChangesPanel`           | `SessionChangesPanel.tsx` | React Component                    |
| `FileExplorer`                  | `FileExplorer.tsx`        | React Component                    |
| `DiffPreview`                   | `FileExplorer.tsx`        | React Component（export function） |
| `ViewMode`                      | `DiffViewer.tsx`          | Type                               |
| `LineType`                      | `DiffViewer.tsx`          | Type                               |
| `ChangeScopeMode`               | `changeScopeStore.ts`     | Type                               |
| `changeScopeStore`              | `changeScopeStore.ts`     | Store 实例                         |
| `useSessionChangeScope`         | `changeScopeStore.ts`     | Hook                               |
| `extractContentFromUnifiedDiff` | `diffUtils.ts`            | Function                           |

### 5.2 API 层依赖

| API 函数                                | 来源             | 用途                            |
| --------------------------------------- | ---------------- | ------------------------------- |
| `getSessionDiff(sessionId, directory)`  | `api/session.ts` | 获取会话级变更                  |
| `getLastTurnDiff(sessionId, directory)` | `api/session.ts` | 获取最近轮次变更                |
| `getVcsDiff(mode, directory)`           | `api/vcs.ts`     | 获取 VCS 差异（git/branch）     |
| `getVcsInfo(directory)`                 | `api/vcs.ts`     | 获取 VCS 信息（分支、默认分支） |
| `getCurrentProject(directory)`          | `api/client.ts`  | 获取当前项目信息                |
| `initGitProject(directory)`             | `api/client.ts`  | 初始化 Git 项目                 |

### 5.3 Hook 依赖

| Hook                                | 来源                               | 用途                     |
| ----------------------------------- | ---------------------------------- | ------------------------ |
| `useSyntaxHighlight(code, options)` | `hooks/useSyntaxHighlight.ts`      | Shiki 语法高亮，LRU 缓存 |
| `useDynamicVirtualScroll(options)`  | `hooks/useDynamicVirtualScroll.ts` | 动态高度虚拟滚动         |
| `useVerticalSplitResize(options)`   | `hooks/useVerticalSplitResize.ts`  | 垂直分割拖拽调整         |
| `useFileExplorer(options)`          | `hooks/useFileExplorer.ts`         | 文件浏览器状态管理       |

### 5.4 Store 依赖

| Store              | 用途                                                     |
| ------------------ | -------------------------------------------------------- |
| `themeStore`       | 读取 `diffStyle`（markers/changeBars）和 `codeWordWrap`  |
| `changeScopeStore` | 按会话 ID 管理变更范围模式                               |
| `layoutStore`      | 文件预览管理（`openFilePreview`、`closeFilePreview` 等） |

### 5.5 工具函数依赖

| 函数                                       | 来源                     | 用途                        |
| ------------------------------------------ | ------------------------ | --------------------------- |
| `detectLanguage(filePath)`                 | `utils/languageUtils.ts` | 根据文件路径检测编程语言    |
| `getMaterialIconUrl(path, type, expanded)` | `utils/materialIcons.ts` | 获取 Material Icon 图标 URL |
| `extractContentFromUnifiedDiff(diff)`      | `utils/diffUtils.ts`     | 解析 unified diff 字符串    |
| `sessionErrorHandler(context, error)`      | `utils/index.ts`         | 统一的会话错误处理          |

---

## 6. Implementation Strategy

### 6.1 组件层次（实际建成）

```
DiffViewer (核心 diff 渲染，1423 行)
    │
    ├── SplitDiffView (并排模式，固定行高)
    │       ├── 左面板 (gutter + content)
    │       ├── 分隔线
    │       └── 右面板 (gutter + content)
    │       └── Sticky proxy scrollbar
    │
    ├── WrappedSplitDiffView (并排模式，动态行高)
    │
    ├── UnifiedDiffView (统一模式，固定行高)
    │       ├── gutter (行号 + 符号 / change bar)
    │       └── content (代码内容)
    │       └── Sticky proxy scrollbar
    │
    └── WrappedUnifiedDiffView (统一模式，动态行高)

DiffView (内嵌式 diff 组件，170 行)
    │
    ├── 折叠头部 (文件名 + 统计 + 放大按钮)
    ├── DiffViewer (内嵌内容，unified 模式)
    └── FullscreenViewer (全屏弹窗)
            └── DiffViewer (split/unified 可切换)

SessionChangesPanel (会话变更面板，1052 行)
    │
    ├── 头部 (统计 + 范围选择 + 视图切换 + 刷新)
    ├── 文件列表 (树形/扁平)
    │       ├── ChangesTreeItem (目录/文件节点)
    │       └── 扁平列表项
    └── DiffPreviewPanel (diff 预览区)
            ├── PreviewTabsBar (多标签)
            ├── DiffViewer
            └── FullscreenViewer (全屏)

FileExplorer (文件浏览器，982 行)
    │
    ├── FileTreeItem (文件树节点，含变更状态颜色)
    └── FilePreview (文件内容预览)
            ├── PreviewTabsBar
            ├── CodePreview / MediaPreview / TextMediaPreview / BinaryPlaceholder
            └── FullscreenViewer (全屏)
```

### 6.2 自动降级策略

**Split → Unified 自动降级**:

```typescript
const isAddOnly = !before.trim()
const isDeleteOnly = !after.trim()
const effectiveViewMode = isAddOnly || isDeleteOnly ? 'unified' : viewMode
```

纯新增或纯删除文件时，split 模式另一侧完全空白，自动降级为 unified 模式。

**Unified Diff 解析失败降级**:

```typescript
if (!hasContent && diff) {
  return <div className="...">{diff}</div>  // 纯文本展示
}
```

当 `extractContentFromUnifiedDiff` 无法解析时，以原始文本形式展示 diff 字符串。

### 6.3 性能优化策略

| 优化点              | 实现方式                                                      |
| ------------------- | ------------------------------------------------------------- |
| 虚拟滚动            | 固定行高手动计算 / 动态行高 `useDynamicVirtualScroll`         |
| 语法高亮缓存        | `useSyntaxHighlight` 内置 LRU 缓存（maxSize=100）             |
| Resize 期间暂停计算 | `isResizing` 标志跳过词级 diff、高亮计算、虚拟滚动重算        |
| 请求 ID 防过期      | `diffRequestIdRef` 递增计数器，丢弃过期响应                   |
| Diff 数据缓存       | `loadedModes` 标记，已加载的模式不重复请求                    |
| 树形结构缓存        | `buildChangesTree` 在 `useMemo` 中执行，仅 `diffs` 变化时重算 |
| 组件 memo 化        | 所有主要组件使用 `memo()` 包裹                                |
| Proxy scrollbar     | 仅在 content 实际溢出时渲染，避免空滚动条                     |
| Content 宽度追踪    | 使用 `scrollWidth` 历史最大值，防止滚动条闪烁                 |

### 6.4 响应式策略

| 场景                                | 行为                                    |
| ----------------------------------- | --------------------------------------- |
| 窗口宽度 >= 1000px（DiffView 全屏） | 默认 split 模式                         |
| 窗口宽度 < 1000px（DiffView 全屏）  | 默认 unified 模式                       |
| 纯新增/纯删除文件                   | 强制 unified 模式                       |
| 启用自动换行                        | 切换到 Wrapped 变体（动态行高虚拟滚动） |

### 6.5 主题集成

`DiffViewer` 通过 `useSyncExternalStore` 订阅 `themeStore`，响应以下主题设置：

- `diffStyle`: `'markers'`（默认，显示行号 +/- 符号）或 `'changeBars'`（3px 变更指示条）
- `codeWordWrap`: 布尔值，控制是否启用自动换行

Gutter 宽度根据 `diffStyle` 动态调整：

- Split 模式：markers = 52px，changeBars = 35px
- Unified 模式：markers = 84px，changeBars = 67px

---

## 7. Error Handling

### 7.1 错误场景与处理

| 错误场景                  | 处理方式                                        | 位置                                         |
| ------------------------- | ----------------------------------------------- | -------------------------------------------- |
| Unified diff 格式不规范   | 降级为纯文本展示，不崩溃                        | `DiffView.tsx`                               |
| API 请求失败（diff 加载） | `sessionErrorHandler` 记录错误，UI 显示友好提示 | `SessionChangesPanel.tsx`                    |
| API 请求失败（项目信息）  | 显示错误信息，不阻塞 UI                         | `SessionChangesPanel.tsx`                    |
| 无 VCS 的项目             | 显示引导界面，提供初始化 Git 按钮               | `SessionChangesPanel.tsx`                    |
| 无变更                    | 显示 "No changes" 提示文本                      | `DiffViewer.tsx` / `SessionChangesPanel.tsx` |
| 文件图标加载失败          | `onError` 隐藏图标，不影响布局                  | `FileTreeItem` / `ChangesTreeItem`           |
| 快速切换文件导致并发请求  | 请求 ID 递增机制，丢弃过期结果                  | `SessionChangesPanel.tsx`                    |
| 模式切换时可用选项变化    | 自动校准到默认模式                              | `SessionChangesPanel.tsx` useEffect          |

### 7.2 加载状态管理

`SessionChangesPanel` 使用细粒度的加载状态：

```typescript
const [loadingModes, setLoadingModes] = useState({
  git: false,
  branch: false,
  session: false,
  turn: false,
})
const [loadedModes, setLoadedModes] = useState({
  git: false,
  branch: false,
  session: false,
  turn: false,
})
```

每种模式独立跟踪加载中和已加载状态，支持并行加载和缓存复用。

### 7.3 状态重置

切换 `sessionId` 或 `directory` 时，`SessionChangesPanel` 完整重置所有状态：

```typescript
setProject(null)
setVcsInfo(null)
setGitDiffs([])
setBranchDiffs([])
setSessionDiffs([])
setTurnDiffs([])
setLoadedModes({ git: false, branch: false, session: false, turn: false })
setLoadingModes({ git: false, branch: false, session: false, turn: false })
setError(null)
setOpenDiffFiles([])
setSelectedFile(null)
setExpandedDirs(new Set())
setChangeMenuOpen(false)
resetSplitHeight()
```

---

## 8. Testing Considerations

### 8.1 现有测试文件

| 测试文件                       | 测试目标           |
| ------------------------------ | ------------------ |
| `DiffViewer.test.tsx`          | 核心 diff 渲染组件 |
| `DiffView.test.tsx`            | 内嵌 diff 组件     |
| `SessionChangesPanel.test.tsx` | 会话变更面板       |

### 8.2 建议测试覆盖

**DiffViewer 核心逻辑**:

- `computePairedLines`: 验证 before/after 变更块的配对正确性
- `computeUnifiedLines`: 验证 unified 模式下行的顺序和行号
- `collapseContextPaired` / `collapseContextUnified`: 验证折叠阈值和展开状态
- `isTooFragmented`: 验证碎片化阈值边界条件
- `computeWordDiff`: 验证词级 diff 合并逻辑

**DiffView 组件**:

- 折叠/展开交互
- 全屏模式打开/关闭
- 响应式视图模式切换（>= 1000px split, < 1000px unified）
- Unified diff 字符串解析
- 无变更时的 "No changes" 展示

**SessionChangesPanel**:

- 变更范围模式切换（session/turn/git/branch）
- 树形/扁平列表切换
- 文件选中与多标签预览
- 无 VCS 时的引导界面
- 请求 ID 防过期机制
- 模式切换时的状态校准

**diffUtils**:

- `extractContentFromUnifiedDiff`: 验证各种 unified diff 格式的解析
- 边界情况：空 diff、仅 header、无变更行

**changeScopeStore**:

- `setMode` / `getMode`: 验证按会话 ID 隔离
- `clearSession` / `clearAll`: 验证状态清理
- `useSessionChangeScope` Hook: 验证响应式订阅

### 8.3 集成测试场景

- 从聊天消息中的 DiffView 点击放大按钮进入全屏，验证 split/unified 切换
- SessionChangesPanel 中切换变更范围模式，验证数据缓存和状态校准
- FileExplorer 中点击文件，验证 git 状态颜色正确应用
- 大文件（10000+ 行）diff 滚动性能

---

## 9. 风险区域

### 9.1 DiffViewer 复杂度（高风险）

**复杂度来源**:

- 1423 行单文件，包含 4 种渲染变体、虚拟滚动、词级 diff、context 折叠
- Split 模式的 proxy scrollbar 同步逻辑（content ↔ scrollbar 双向同步，带 guard）
- Content 宽度追踪使用 `scrollWidth` 历史最大值 + ResizeObserver + MutationObserver

**缓解措施**:

- 所有变体组件使用 `memo()` 包裹
- Resize 期间跳过所有重计算
- 虚拟滚动 overscan 仅 5 行，控制 DOM 节点数

### 9.2 SessionChangesPanel 状态管理（中高风险）

**复杂度来源**:

- 4 种变更范围模式，每种独立缓存
- 树形结构构建与展开状态同步
- 多标签预览的打开/关闭/重排序
- 模式切换时的状态校准（`reconcileDiffPreviewState`）

**缓解措施**:

- 请求 ID 防过期机制
- `useMemo` 缓存树形结构和 diff 数据
- `openDiffFilesRef` / `selectedFileRef` 避免闭包过期

### 9.3 词级 Diff 性能（中风险）

**复杂度来源**:

- 每对变更行都调用 `diffWords`，长行计算成本高
- 碎片化检测虽能跳过无意义高亮，但仍需先计算

**缓解措施**:

- `isResizing` 期间完全跳过
- `isTooFragmented` 阈值检测
- 相邻同类型变更合并减少 DOM 节点

### 9.4 跨平台路径处理（低风险）

**复杂度来源**:

- Windows 反斜杠 vs Linux/macOS 正斜杠
- `fileStatus` Map 的 key 统一使用正斜杠，但 `node.path` 可能为反斜杠

**缓解措施**:

- `FileTreeItem` 中双重查询：`fileStatus.get(node.path) || fileStatus.get(node.path.replace(/\\/g, '/'))`
- `buildChangesTree` 中使用 `split(/[/\\]/)` 兼容两种分隔符

---

## 10. 依赖关系图

```
                    ┌─────────────────────┐
                    │   diff (npm 库)      │
                    │   diffLines/diffWords│
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   DiffViewer.tsx     │
                    │   (核心渲染引擎)      │
                    └──┬───────────────┬───┘
                       │               │
            ┌──────────▼──┐   ┌───────▼────────┐
            │useSyntax     │   │useDynamic      │
            │Highlight     │   │VirtualScroll   │
            └──────────────┘   └────────────────┘

┌──────────────────┐         ┌──────────────────────┐
│  DiffView.tsx    │────────►│  DiffViewer           │
│  (内嵌组件)       │         │  + FullscreenViewer   │
└──────────────────┘         └──────────────────────┘

┌──────────────────────────┐
│  SessionChangesPanel.tsx  │
│  (会话变更面板)            │
│                          │
│  ├── changeScopeStore    │──► 变更范围模式
│  ├── api/session.ts      │──► getSessionDiff
│  ├── api/vcs.ts          │──► getVcsDiff
│  ├── DiffViewer          │──► diff 预览
│  └── PreviewTabsBar      │──► 多标签管理
└──────────┬───────────────┘
           │
           │ 被消息渲染模块依赖
           ▼
┌──────────────────────┐
│  MessageRenderer     │
│  (聊天消息渲染)       │
└──────────────────────┘

┌──────────────────────┐
│  FileExplorer.tsx    │
│  (文件浏览器)         │
│                      │
│  ├── useFileExplorer │──► 文件树 + git 状态
│  ├── FileTreeItem    │──► 变更状态颜色
│  └── FilePreview     │──► 文件内容预览
└──────────────────────┘
```

---

## 11. 架构决策记录（As-Built）

### ADR-001：为什么合并 DiffModal 和 MultiFileDiffModal

**决策**: 不创建独立的 `DiffModal.tsx` 和 `MultiFileDiffModal.tsx` 文件，功能整合到 `DiffView.tsx` 和 `SessionChangesPanel.tsx` 中。

**理由**:

- `DiffView` 已内置 `FullscreenViewer`，单文件全屏 diff 无需额外组件
- `SessionChangesPanel` 提供文件列表 + diff 预览的完整体验，覆盖多文件浏览需求
- 减少组件数量，降低维护成本
- 避免功能重叠和代码重复

### ADR-002：为什么使用 Proxy Scrollbar 而非原生滚动条

**决策**: Split 和 Unified 模式中，content 区域隐藏自身滚动条，底部使用 sticky proxy scrollbar 实现横向滚动。

**理由**:

- Gutter 列固定宽度不滚动，content 列独立滚动，原生滚动条会出现在 content 区域底部
- Proxy scrollbar 位于 gutter + content 整体底部，视觉上更统一
- 通过 `scrollSourceRef` guard 防止 content 和 proxy scrollbar 之间的循环同步
- 仅在 content 实际溢出时渲染 proxy scrollbar，避免空滚动条

### ADR-003：为什么使用 scrollWidth 历史最大值

**决策**: Content 宽度追踪使用 `scrollWidth` 的历史最大值，而非当前值。

**理由**:

- 虚拟滚动仅渲染可视区域内的行，不同行的宽度可能不同
- 如果使用当前 scrollWidth，滚动到窄行时滚动条范围会缩小，导致内容截断
- 历史最大值确保滚动条范围覆盖所有已渲染行的最大宽度
- ResizeObserver 触发时重置为 0，重新测量

### ADR-004：为什么变更范围状态按会话 ID 隔离

**决策**: `changeScopeStore` 使用 `Map<sessionId, ChangeScopeMode>` 存储，每个会话独立维护自己的变更范围选择。

**理由**:

- 不同会话可能处于不同的开发阶段，用户关注的变更范围不同
- 切换会话后恢复该会话上次的选择，符合用户心智模型
- 避免全局单例导致的状态污染
- 实现简单，无需额外序列化/持久化

### ADR-005：为什么 DiffViewer 不直接使用 react-virtuoso 等第三方虚拟滚动库

**决策**: 自研固定行高虚拟滚动 + `useDynamicVirtualScroll` 动态行高方案。

**理由**:

- 固定行高场景下，手动计算仅需几行代码，引入第三方库增加 bundle 体积
- 动态行高场景下，`useDynamicVirtualScroll` 通过 ref callback 测量，避免 reflow 风暴
- 与项目"自定义优于框架依赖"的宪法原则一致
- 完全控制滚动行为，便于与 proxy scrollbar、resize 暂停等特性集成

---

## 12. 与 Spec 的差异总结

| Spec 描述                         | 实际实现                                      | 差异原因                                        |
| --------------------------------- | --------------------------------------------- | ----------------------------------------------- |
| `DiffModal.tsx` 独立组件          | 不存在，功能整合到 `DiffView.tsx`             | DiffView 内置 FullscreenViewer                  |
| `MultiFileDiffModal.tsx` 独立组件 | 不存在，功能由 `SessionChangesPanel.tsx` 承担 | SessionChangesPanel 提供完整的多文件浏览体验    |
| `diffUtils.ts` 解析 unified diff  | 仅 28 行，简单字符串解析                      | 功能聚焦，仅做 before/after 提取                |
| `changeScopeStore` 持久化         | 无 localStorage 持久化                        | 内存存储，会话切换时重置                        |
| FR-04 多文件 diff 模态框          | 由 SessionChangesPanel 实现，非模态框形式     | 面板式布局更符合整体 UI 架构                    |
| FR-07 文件内容预览                | 在 FileExplorer 中实现，非本模块核心          | FileExplorer 同时服务 Files 和 Changes 两个面板 |

---

_生成时间: 2026-04-12_
