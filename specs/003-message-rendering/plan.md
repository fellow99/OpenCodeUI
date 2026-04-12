# 003-message-rendering 模块技术方案（As-Built）

> 本文档是对 `src/features/message/` 模块的回溯性技术规划，记录"实际建成"的架构设计、组件分解与渲染策略。
>
> **模块路径**: `src/features/message/`
> **版本**: 1.0
> **状态**: 已实现

---

## 1. Technical Context

### 1.1 模块定位

消息渲染模块是 OpenCodeUI 的核心展示层，负责将后端 SSE 流式推送的多态消息数据（Message → Part）转换为可视化的用户界面。模块位于 `src/features/message/`，被 `002-chat-feature` 直接依赖，同时依赖 `001-api-layer`（类型定义）、`006-mention-system`（MentionTag 渲染）、`009-theme-system`（主题控制）、`010-state-management`（messageStore）、`011-file-diff-viewer`（Diff 展示）、`013-i18n-localization`（国际化）。

### 1.2 技术栈

| 技术                        | 用途                                    |
| --------------------------- | --------------------------------------- |
| React 19 + TypeScript       | 组件框架                                |
| Tailwind CSS v4             | 样式系统                                |
| react-markdown + remark-gfm | Markdown 渲染引擎                       |
| Shiki                       | 代码语法高亮                            |
| KaTeX                       | 数学公式渲染                            |
| diff (diffLines)            | 文本差异计算                            |
| motion/mini (animate)       | 入场生长动画                            |
| lucide-react                | 工具图标库                              |
| react-i18next               | 国际化（message / components 命名空间） |

### 1.3 目录结构

```
src/features/message/
├── index.ts                          # Barrel 导出
├── MessageRenderer.tsx               # 入口组件（980 行）
├── MessageRenderer.test.tsx          # 单元测试
├── parts/                            # Part 渲染组件
│   ├── index.ts                      # Barrel 导出
│   ├── TextPartView.tsx              # 文本 Part → MarkdownRenderer
│   ├── ReasoningPartView.tsx         # 推理 Part（三种模式）
│   ├── ToolPartView.tsx              # 工具调用视图（三种布局）
│   ├── StepFinishPartView.tsx        # 步骤完成信息
│   ├── SubtaskPartView.tsx           # 子任务卡片
│   ├── AttachmentPartViews.tsx       # FilePart / AgentPart / SyntheticTextPart
│   ├── SystemPartViews.tsx           # RetryPart / CompactionPart / PatchPart
│   └── MessageErrorView.tsx          # 消息级错误
└── tools/                            # 工具渲染器注册表
    ├── index.ts                      # Barrel 导出
    ├── types.ts                      # 类型定义
    ├── icons.tsx                     # 工具图标（lucide-react 包装）
    ├── registry.tsx                  # 注册表 + 数据提取器
    ├── registry.test.ts              # 注册表测试
    └── renderers/                    # 专用渲染器
        ├── index.ts
        ├── BashRenderer.tsx          # 终端风格
        ├── TodoRenderer.tsx          # 清单列表
        ├── TaskRenderer.tsx          # 子 Agent 会话
        ├── QuestionRenderer.tsx      # 已回答问题
        ├── DefaultRenderer.tsx       # 通用 Input/Output
        └── todoUtils.ts              # Todo 提取工具
```

### 1.4 外部依赖组件

以下组件位于 `src/components/`，被本模块广泛使用：

| 组件               | 路径                                  | 用途                                 |
| ------------------ | ------------------------------------- | ------------------------------------ |
| `MarkdownRenderer` | `src/components/MarkdownRenderer.tsx` | Markdown 渲染（GFM + KaTeX + Shiki） |
| `CodeBlock`        | `src/components/CodeBlock.tsx`        | 代码块（Shiki 高亮 + 复制）          |
| `ContentBlock`     | `src/components/ContentBlock.tsx`     | 通用内容块容器（Diff/代码/加载态）   |
| `SmoothHeight`     | `src/components/ui/`                  | 流式高度动画                         |
| `CopyButton`       | `src/components/ui/`                  | 复制按钮                             |
| `ScrollArea`       | `src/components/ui/`                  | 滚动区域                             |

---

## 2. Constitution Check

本模块的实现与项目宪法各项原则的对齐情况：

| 宪法原则                        | 对齐状态    | 说明                                                                                   |
| ------------------------------- | ----------- | -------------------------------------------------------------------------------------- |
| **原则 2: OpenCode 兼容性优先** | ✅ 完全对齐 | Part 类型定义严格镜像后端 OpenAPI 规范（`src/types/message.ts`），不遗漏任何 Part 类型 |
| **原则 3: 多平台统一代码库**    | ✅ 完全对齐 | 无平台特定代码，Web/Tauri 共享同一渲染逻辑                                             |
| **原则 4: 自定义优于框架依赖**  | ✅ 对齐     | 折叠/展开使用原生 grid-rows 动画而非第三方动画库；状态管理使用 React 原生 hooks        |
| **原则 5: 实时通信优先**        | ✅ 对齐     | 支持 SSE 流式增量渲染（`MESSAGE_PART_DELTA`），内容到达即显示                          |
| **原则 9: 主题与可访问性**      | ✅ 对齐     | 推理显示模式、工具卡片样式、沉浸模式等均由主题控制；折叠按钮提供 `aria-expanded`       |
| **原则 10: 模块化功能架构**     | ✅ 对齐     | 自包含的 parts/ 和 tools/ 子目录，通过明确的接口（props + registry）通信               |
| **约束 C4: 依赖最小化**         | ✅ 对齐     | 工具图标复用 lucide-react（项目已有依赖），未引入额外图标库                            |

---

## 3. Research Findings

### 3.1 核心渲染流程

消息渲染遵循以下数据流：

```
Message (from messageStore)
    ↓
MessageRenderer (按 role 分发)
    ├── role === 'user'  → UserMessageView
    └── role === 'assistant' → AssistantMessageView
            ↓
        groupPartsForRender(parts) → RenderItem[]
            ├── { type: 'single', part }     → 对应 PartView
            └── { type: 'tool-group', parts, stepFinish } → ToolGroup
                    ↓
                ToolPartView × N (三种布局变体)
                    ↓
                ToolBody (注册表匹配)
                    ├── TaskRenderer (task 工具)
                    ├── TodoRenderer (todo 工具)
                    ├── config.renderer (注册表专用渲染器)
                    └── DefaultRenderer (通用)
```

### 3.2 Part 分组策略（groupPartsForRender）

实际代码中的分组逻辑比 spec 描述的更精细：

1. **跳过基础设施**: `step-start`、`snapshot`、`patch` 被过滤
2. **跳过空内容**: 空 `text` Part（`isVisibleTextPart` 判断）和空 `reasoning` Part（`isVisibleReasoningPart` 判断）在非流式状态下被跳过
3. **工具归组**: 连续的 `ToolPart` 合并为 `tool-group`，附带关联的 `StepFinishPart`
4. **中间 StepFinish 暂存**: 当 `step-finish` 后面还有更多 tool 时，暂存不 flush，直到最后一个 tool 组结束
5. **独立渲染**: 其余 Part 各自独立渲染

关键函数 `hasMoreToolsAhead()` 用于判断当前 `step-finish` 之后是否还有有意义的 tool part，决定是否暂存 `stepFinish`。

### 3.3 三种布局变体的实际实现

`ToolPartView` 支持三种布局变体，通过 props 控制：

| 变体            | 触发条件                                 | 视觉特征                                             |
| --------------- | ---------------------------------------- | ---------------------------------------------------- |
| **Timeline**    | 默认（多工具组）                         | 左侧时间线（图标 + 连接线），右侧内容                |
| **Compact**     | `compact=true`（单工具且非 descriptive） | 14px 固定图标列，无连接线，与 ReasoningPartView 对齐 |
| **Descriptive** | `descriptive=true`（主题开启描述型步骤） | 无图标/时间线，纯文本行，头部显示分类摘要            |

### 3.4 工具注册表的实际匹配顺序

实际注册表（`registry.tsx`）按以下优先级排列：

| 优先级 | 匹配模式                            | 匹配函数     | 图标          | 专用渲染器       | 数据提取器       |
| ------ | ----------------------------------- | ------------ | ------------- | ---------------- | ---------------- |
| 1      | `bash/sh/cmd/terminal/shell`        | `includes()` | TerminalIcon  | BashRenderer     | bashExtractData  |
| 2      | `todo`                              | `includes()` | ChecklistIcon | —                | 默认             |
| 3      | `task`（精确）                      | `exact()`    | TaskIcon      | —                | 默认             |
| 4      | `read/cat`                          | `includes()` | FileReadIcon  | —                | readExtractData  |
| 5      | `write/save`                        | `includes()` | FileWriteIcon | —                | writeExtractData |
| 6      | `edit/replace/patch`                | `includes()` | FileWriteIcon | —                | editExtractData  |
| 7      | `search/find/grep/glob`             | `includes()` | SearchIcon    | —                | 默认             |
| 8      | `web/fetch/http/browse/network/exa` | `includes()` | GlobeIcon     | —                | 默认             |
| 9      | `think/reason/plan`                 | `includes()` | BrainIcon     | —                | 默认             |
| 10     | `question/ask`                      | `includes()` | QuestionIcon  | QuestionRenderer | 默认             |

**注意**: `todo` 排在 `write/read` 之前，避免 `TodoWrite` 被误匹配为 `write`。`task` 使用精确匹配（`exact()`），避免误匹配其他含 "task" 的工具名。

### 3.5 ToolPartView 中的 ToolBody 分发逻辑

`ToolBody` 组件的分发逻辑与注册表不完全一致，存在硬编码优先匹配：

```
ToolBody
    ├── tool === 'task' → TaskRenderer（硬编码优先）
    ├── tool.includes('todo') && hasTodos(part) → TodoRenderer（硬编码 + 条件）
    ├── getToolConfig(tool).renderer → 注册表专用渲染器（BashRenderer / QuestionRenderer）
    └── DefaultRenderer
```

这意味着 `task` 和 `todo` 的分发不经过注册表的 `renderer` 字段，而是直接在 `ToolBody` 中硬编码处理。

### 3.6 推理 Part 的三种显示模式

实际代码中 `ReasoningPartView` 根据 `reasoningDisplayMode` 主题设置选择渲染模式：

| 模式       | 渲染行为                                       |
| ---------- | ---------------------------------------------- |
| `italic`   | 斜体文本，可折叠摘要，流光动画（streaming 时） |
| `markdown` | Markdown 格式渲染（支持代码块/列表），可折叠   |
| `capsule`  | 带图标的可折叠卡片，展开后在 ScrollArea 中展示 |

流式行为：`isPartStreaming` 时自动展开，完成后自动收起为摘要。

### 3.7 入场生长动画（useEntryGrowAnimation）

新消息从高度 0 平滑展开的实现：

- 使用 `motion/mini` 的 `animate()` 函数
- 动画时长 0.2s，easeOut 曲线
- 使用 `clipPath: inset(0 -100% 0 -100%)` 防止动画期间内容溢出
- 仅在消息创建后 3 秒内触发（`Date.now() - created > 3000` 则跳过）

### 3.8 流式布局动画（SmoothHeight）

- `AssistantMessageView` 使用 `SmoothHeight` 包裹内容区域
- 仅在 `isStreaming && allowStreamingLayoutAnimation` 时激活
- 用户查看历史消息时 `allowStreamingLayoutAnimation=false`，避免视口跳动

### 3.9 延迟渲染（useDelayedRender）

所有折叠/展开组件统一使用 `useDelayedRender` hook：

- 展开时立即渲染（无延迟）
- 折叠时延迟卸载（DOM 保持挂载直到动画结束）
- 避免折叠/展开时的闪烁

### 3.10 沉浸模式下的工具组自动收起逻辑

`ToolGroup` 组件在沉浸模式下的行为：

1. 判断是否包含"可读工具"（`READABLE_TOOL_PATTERNS`：bash/sh/cmd/terminal/shell/write/save/edit/replace/patch/todo/question/ask）
2. 流式传输中 + 有可读工具 → 保持展开
3. 完成后 + 无可读工具 → 自动收起
4. 完成后 + 有可读工具 → 保持展开
5. 描述型步骤模式 → 默认收起，运行时展开

---

## 4. Data Model

### 4.1 核心类型（来自 src/types/message.ts）

```typescript
// 消息容器
interface Message {
  info: MessageInfo // 消息元信息（role, id, time 等）
  parts: Part[] // 内容片段数组
  isStreaming: boolean // 是否正在流式传输
}

// Part 多态类型
type Part =
  | TextPart // type: 'text'
  | ReasoningPart // type: 'reasoning'
  | ToolPart // type: 'tool'
  | FilePart // type: 'file'
  | AgentPart // type: 'agent'
  | SubtaskPart // type: 'subtask'
  | StepStartPart // type: 'step-start'
  | StepFinishPart // type: 'step-finish'
  | SnapshotPart // type: 'snapshot'
  | PatchPart // type: 'patch'
  | RetryPart // type: 'retry'
  | CompactionPart // type: 'compaction'

// ToolPart 状态多态
type ToolState =
  | { status: 'pending'; input: unknown; raw?: unknown }
  | { status: 'running'; input: unknown; title?: string; time: { start: number } }
  | {
      status: 'completed'
      input: unknown
      output: unknown
      title?: string
      time: { start: number; end: number }
      attachments?: unknown
    }
  | { status: 'error'; input: unknown; error: unknown; time: { start: number; end: number } }
```

### 4.2 渲染项类型（RenderItem）

```typescript
type RenderItem =
  | { type: 'single'; part: Part }
  | { type: 'tool-group'; parts: ToolPart[]; stepFinish?: StepFinishPart }
```

### 4.3 工具数据提取类型（ExtractedToolData）

```typescript
interface ExtractedToolData {
  input?: string
  inputLang?: string
  output?: string
  outputLang?: string
  error?: string
  diff?: { before: string; after: string } | string
  diffStats?: { additions: number; deletions: number }
  files?: FileDiff[]
  filePath?: string
  exitCode?: number
  diagnostics?: DiagnosticInfo[]
}
```

### 4.4 工具注册表类型

```typescript
interface ToolConfig {
  match: (toolName: string) => boolean
  icon: ReactNode
  renderer?: ComponentType<ToolRendererProps>
  extractData?: (part: ToolPart) => Partial<ExtractedToolData>
}

type ToolRegistry = ToolConfig[]
```

### 4.5 描述型步骤摘要类型

```typescript
type ToolSummaryCategory = 'execute' | 'edit' | 'explore' | 'network' | 'task' | 'todo' | 'question' | 'think' | 'other'
type ToolSummaryPhase = 'done' | 'active' | 'failed'

interface SummarySegment {
  text: string
  type: 'normal' | 'error' | 'active'
}
```

---

## 5. Interface Contracts

### 5.1 MessageRenderer Props

```typescript
interface MessageRendererProps {
  message: Message
  allowStreamingLayoutAnimation?: boolean // 默认 true
  turnDuration?: number // 回合总时长（ms）
  onUndo?: (userMessageId: string) => void
  onFork?: (message: Message, forkMessageId?: string) => Promise<void> | void
  forkMessageId?: string
  canUndo?: boolean
  onEnsureParts?: (messageId: string) => void
}
```

### 5.2 PartView 组件通用模式

所有 PartView 组件遵循统一模式：

```typescript
interface XxxPartViewProps {
  part: XxxPart
  isStreaming?: boolean // 仅 TextPartView / ReasoningPartView / ToolPartView
}
```

### 5.3 ToolPartView Props

```typescript
interface ToolPartViewProps {
  part: ToolPart
  isFirst?: boolean // 时间线顶部连接线
  isLast?: boolean // 时间线底部连接线
  compact?: boolean // 紧凑布局（单工具）
  descriptive?: boolean // 描述型步骤布局
  isStreaming?: boolean // 父消息是否流式传输
}
```

### 5.4 ToolRenderer Props

```typescript
interface ToolRendererProps {
  part: ToolPart
  data: ExtractedToolData
}
```

### 5.5 工具注册表接口

```typescript
// 获取工具配置
function getToolConfig(toolName: string): ToolConfig | undefined

// 获取工具图标
function getToolIcon(toolName: string): ReactNode

// 提取工具数据
function extractToolData(part: ToolPart): ExtractedToolData

// 默认数据提取器
function defaultExtractData(part: ToolPart): ExtractedToolData
```

---

## 6. Implementation Strategy

### 6.1 组件层级关系

```
MessageRenderer (memo)
├── UserMessageView (memo)
│   ├── CollapsibleUserText (memo) — 8 行预览 + 渐变遮罩
│   ├── FilePartView (memo)
│   ├── AgentPartView (memo)
│   ├── SyntheticTextPartView (memo)
│   ├── CompactionPartView (memo)
│   └── ForkActionButton (memo) + CopyButton + UndoButton
│
└── AssistantMessageView (memo)
    ├── SmoothHeight (流式动画容器)
    │   └── RenderItem[]
    │       ├── TextPartView → MarkdownRenderer
    │       ├── ReasoningPartView (三种模式)
    │       ├── ToolGroup (memo)
    │       │   ├── 步骤摘要（描述型模式）
    │       │   ├── ToolPartView × N (三种布局)
    │       │   │   └── ToolBody
    │       │   │       ├── TaskRenderer
    │       │   │       ├── TodoRenderer
    │       │   │       ├── BashRenderer (via registry)
    │       │   │       ├── QuestionRenderer (via registry)
    │       │   │       └── DefaultRenderer
    │       │   └── StepFinishPartView (附带)
    │       ├── StepFinishPartView
    │       ├── SubtaskPartView
    │       ├── RetryPartView
    │       └── CompactionPartView
    ├── MessageErrorView
    └── ForkActionButton + CopyButton
```

### 6.2 关键设计决策

#### 6.2.1 Memo 策略

几乎所有组件都使用 `React.memo` 包裹，避免 SSE 流式更新时的不必要重渲染。`MessageRenderer`、`UserMessageView`、`AssistantMessageView`、`ToolGroup`、`ToolPartView`、`ToolBody` 均为 memo 组件。

#### 6.2.2 折叠/展开动画模式

统一使用 `grid-rows-[1fr]` / `grid-rows-[0fr]` 配合 `transition-[grid-template-rows]` 实现折叠动画，辅以 `clipPath: inset(0 -100% 0 -100%)` 防止内容溢出。这是 Tailwind CSS v4 下的零依赖动画方案。

#### 6.2.3 状态保持策略

`ToolGroup` 内的 `ToolPartView` 始终在同一 React 树位置渲染，streaming→idle / 1→N 工具切换时不 remount，确保 `expanded` 状态不丢失。

#### 6.2.4 权限请求延迟卸载

`ToolPartView` 中实现了权限请求的延迟卸载机制：用户授权后 `permissionRequest` 立即消失，但工具结果可能还没到。为避免"权限消失→空白→结果出现"的跳动，使用 `cachedPermissionRequest` 缓存最后一次权限请求，在工具完成之前继续渲染。

#### 6.2.5 Diff Stats 计算策略

描述型步骤模式下的 Diff 统计有两种来源：

1. 优先使用 `metadata.filediff` 中的 `diffStats`（后端提供）
2. 回退到 `computePartDiffStats()` 从 `diff` 数据计算（使用 `diffLines`）

### 6.3 主题控制点

模块受以下主题设置控制（来自 `useTheme()` hook）：

| 主题设置                  | 类型                                              | 影响范围                          |
| ------------------------- | ------------------------------------------------- | --------------------------------- | ---------------------------- | -------------------------- |
| `reasoningDisplayMode`    | `'italic'                                         | 'markdown'                        | 'capsule'`                   | ReasoningPartView 渲染模式 |
| `toolCardStyle`           | `'standard'                                       | 'compact'`                        | DefaultRenderer 输入区块显示 |
| `descriptiveToolSteps`    | `boolean`                                         | ToolGroup / ToolPartView 布局变体 |
| `immersiveMode`           | `boolean`                                         | ToolGroup 自动收起逻辑            |
| `inlineToolRequests`      | `boolean`                                         | ToolPartView 内嵌权限/问答        |
| `compactInlinePermission` | `boolean`                                         | 权限组件显示模式                  |
| `collapseUserMessages`    | `boolean`                                         | CollapsibleUserText 折叠          |
| `stepFinishDisplay`       | `{ tokens, cache, cost, duration, turnDuration }` | StepFinishPartView 显示项         |

---

## 7. Error Handling

### 7.1 消息级错误（MessageErrorView）

处理 `AssistantMessageInfo.error` 字段，支持以下错误类型：

| 错误类型                   | 标题                 | 严重性        | 可展开详情                      |
| -------------------------- | -------------------- | ------------- | ------------------------------- |
| `ProviderAuthError`        | 认证错误             | error         | 是（provider + message）        |
| `MessageOutputLengthError` | 输出超长             | warning       | 否                              |
| `MessageAbortedError`      | 消息中止             | warning       | 是（message）                   |
| `APIError`                 | API 错误（含状态码） | error/warning | 是（responseBody，JSON 格式化） |
| `UnknownError`             | 未知错误             | error         | 是（message）                   |

JSON 格式的 `responseBody` 自动通过 `JSON.stringify(JSON.parse(details), null, 2)` 格式化后在 `CodeBlock` 中展示。

### 7.2 工具级错误

- 工具状态为 `error` 时，`ToolPartView` 头部显示红色标题和 "failed" 标签
- `DefaultRenderer` 中错误优先于输出展示
- `BashRenderer` 中错误以红色文本显示在输出区域
- `TaskRenderer` 中错误通过 `ContentBlock` 的 `variant="error"` 展示

### 7.3 重试信息（RetryPartView）

- 显示重试次数、时间戳、可重试标识
- 可展开查看错误详情和状态码
- 使用警告色（warning-100）而非错误色

### 7.4 空状态处理

- 空 `TextPart`：非流式时返回 `null`
- 空 `ReasoningPart`：返回 `null`
- 无 Parts 的 AssistantMessage：显示 40px 最小占位（CLS 防护），触发 `onEnsureParts`
- 无 Parts 且有错误：直接显示 `MessageErrorView`

### 7.5 边界情况

- **Parts 补全**: `useEffect` 检测 `parts.length === 0` 时触发 `onEnsureParts()`
- **推理结束判断**: 通过扫描后续 Part 类型判断推理是否结束（非 `snapshot`/`patch` 即结束）
- **用户消息文本合并**: 多个 `text` Part 合并为完整文本后展示
- **系统上下文延迟渲染**: 使用 `useDelayedRender` 避免未展开时的 DOM 开销

---

## 8. Testing Considerations

### 8.1 现有测试覆盖

`MessageRenderer.test.tsx` 当前覆盖：

| 测试场景                               | 状态      |
| -------------------------------------- | --------- |
| Assistant 消息 fork 传递 forkMessageId | ✅ 已覆盖 |
| 无可复制文本时隐藏 fork/copy 按钮      | ✅ 已覆盖 |
| User 消息中渲染 compaction Part        | ✅ 已覆盖 |

### 8.2 建议补充的测试场景

#### Part 分组逻辑

```
- groupPartsForRender 将连续 tool parts 归组
- groupPartsForRender 跳过 step-start/snapshot/patch
- groupPartsForRender 跳过空 text/reasoning parts
- groupPartsForRender 中间 step-finish 暂存逻辑
```

#### 工具注册表

```
- getToolConfig 按优先级匹配（todo 优先于 write）
- getToolConfig 精确匹配 task
- getToolIcon 未匹配时返回 WrenchIcon
- extractToolData 调用自定义 extractData
- defaultExtractData 提取 input/output/diff/diagnostics
```

#### 渲染行为

```
- TextPartView 跳过空文本和 synthetic 文本
- ReasoningPartView 流式时自动展开
- ReasoningPartView 完成后自动收起
- ToolGroup 沉浸模式下可读工具完成后保持展开
- ToolGroup 描述型步骤摘要分类逻辑
```

#### 错误处理

```
- MessageErrorView 渲染 ProviderAuthError
- MessageErrorView 渲染 APIError 并格式化 JSON
- MessageErrorView 可重试错误显示重试标识
```

#### 边界情况

```
- 无 Parts 时显示最小占位并触发 onEnsureParts
- 有错误且无 Parts 时显示 MessageErrorView
- 用户消息多 text Part 合并渲染
```

### 8.3 Mock 策略

现有测试的 Mock 模式可作为参考：

```typescript
vi.mock('motion/mini', () => ({ animate: () => Promise.resolve() }))
vi.mock('../../hooks', () => ({ useDelayedRender: (show: boolean) => show }))
vi.mock('../../hooks/useTheme', () => ({ useTheme: () => ({ ...defaults }) }))
vi.mock('../../components/ui', () => ({ CopyButton: ..., SmoothHeight: ... }))
vi.mock('./parts', () => ({ TextPartView: ..., ... }))
```

### 8.4 性能测试建议

- 大量 Part（100+）的渲染性能
- 流式更新时的重渲染次数（React DevTools Profiler）
- `groupPartsForRender` 在长消息中的计算开销
- `extractToolData` 的缓存策略有效性

---

## 9. 风险区域与缓解

### 9.1 ToolBody 分发逻辑与注册表不一致（中风险）

**问题**: `ToolBody` 中 `task` 和 `todo` 的分发是硬编码的，不经过注册表的 `renderer` 字段。这意味着注册表中为 `task`/`todo` 配置 `renderer` 不会生效。

**影响**: 新增工具类型时，开发者可能误以为在注册表中添加 `renderer` 即可，实际需要在 `ToolBody` 中也添加硬编码分支。

**缓解**: 在 `registry.tsx` 中添加注释说明此限制，或重构 `ToolBody` 使其完全依赖注册表。

### 9.2 READABLE_TOOL_PATTERNS 重复定义（低风险）

**问题**: `READABLE_TOOL_PATTERNS` 正则表达式在 `MessageRenderer.tsx` 和 `ToolPartView.tsx` 中各定义了一次，存在不一致风险。

**影响**: 修改一处而遗漏另一处可能导致沉浸模式行为不一致。

**缓解**: 提取为共享常量（如 `tools/registry.ts` 中导出）。

### 9.3 MessageRenderer 单文件过大（低风险）

**问题**: `MessageRenderer.tsx` 达到 980 行，包含 `UserMessageView`、`AssistantMessageView`、`ToolGroup` 及多个辅助函数。

**影响**: 文件过大影响可读性和维护性。

**缓解**: 当前通过清晰的注释分隔和 memo 包裹保持可维护性。如需拆分，可将 `ToolGroup` 和辅助函数移至独立文件。

### 9.4 diffLines 计算性能（低风险）

**问题**: `computePartDiffStats` 和 `computeDiffPairStats` 使用 `diffLines` 计算差异统计，在大型文件编辑场景下可能较慢。

**影响**: 多文件编辑时描述型步骤摘要可能出现短暂延迟。

**缓解**: 优先使用后端提供的 `metadata.filediff.diffStats`，仅在缺失时才回退到 `diffLines` 计算。

---

## 10. 国际化键值清单

模块使用的 i18n 命名空间和键值：

### message 命名空间

| 键                                                           | 用途                                |
| ------------------------------------------------------------ | ----------------------------------- |
| `showSystemContext`                                          | 显示系统上下文按钮                  |
| `hideSystemContext`                                          | 隐藏系统上下文按钮                  |
| `showMore` / `showLess`                                      | 折叠/展开文本                       |
| `undoFromHere`                                               | 撤销按钮                            |
| `forkFromHere` / `forkingFromHere`                           | 分叉按钮                            |
| `stepsCount`                                                 | 步骤计数                            |
| `reasoning.thinking`                                         | 思考中标签                          |
| `reasoning.thinkingLabel`                                    | 思考过程标签                        |
| `reasoning.thoughtFor`                                       | 思考时长标签                        |
| `reasoning.thoughtProcess`                                   | 思考过程标签                        |
| `toolPart.running`                                           | 工具运行中标签                      |
| `toolPart.failed`                                            | 工具失败标签                        |
| `subtask.running` / `subtask.done` / `subtask.error`         | 子任务状态                          |
| `subtask.enter` / `subtask.viewFullSession` / `subtask.task` | 子任务交互                          |
| `stepFinish.*`                                               | 步骤完成信息                        |
| `toolSteps.*`                                                | 描述型步骤摘要                      |
| `errors.*`                                                   | 错误信息                            |
| `system.*`                                                   | 系统 Part（retry/compaction/patch） |
| `task.*`                                                     | 子任务渲染                          |
| `todo.*`                                                     | Todo 渲染                           |
| `defaultRenderer.*`                                          | 默认渲染器                          |

### components 命名空间

| 键                      | 用途       |
| ----------------------- | ---------- |
| `contentBlock.exitCode` | 退出码显示 |

---

## 11. 与 overall-plan.md 的对齐

对照 `overall-plan.md` 第 62-75 行对 003 模块的描述：

| overall-plan 描述                                 | 实际实现                                                     | 对齐状态 |
| ------------------------------------------------- | ------------------------------------------------------------ | -------- |
| MessageRenderer 根据消息类型和 part 类型分发      | ✅ 按 role 分发到 User/Assistant View，再按 part.type switch | ✅ 对齐  |
| Parts 系统（text、tool、question、permission 等） | ✅ 12 种 Part 类型，各有独立 View 组件                       | ✅ 对齐  |
| parts/ 目录：text、tool、question、permission     | ✅ 实际包含 10 个文件，覆盖所有 Part 类型                    | ✅ 对齐  |
| tools/ 目录：各类工具调用的详细展示               | ✅ 注册表模式 + 5 个专用渲染器                               | ✅ 对齐  |
| MarkdownRenderer 基于 react-markdown + remark-gfm | ✅ 通过 `src/components/MarkdownRenderer` 集成               | ✅ 对齐  |
| CodeBlock 支持语言检测、语法高亮、复制            | ✅ 通过 `src/components/CodeBlock` 实现                      | ✅ 对齐  |
| 流式渲染通过 SSE MESSAGE_PART_DELTA 事件          | ✅ `isStreaming` prop 驱动，SmoothHeight 动画                | ✅ 对齐  |

---

_本文档基于 OpenCodeUI v0.4.8 实际代码生成，所有细节来源于源码审查。_
