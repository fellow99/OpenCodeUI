# 007-slash-command 模块技术方案（As-Built）

> 模块编号：007-slash-command
> 状态：已实现
> 技术栈：React 19 + TypeScript + Tailwind CSS v4 + i18next
> 源文件：3 个（`SlashCommandMenu.tsx`、`SlashCommandMenu.test.tsx`、`index.ts`）
> 依赖 API：`src/api/command.ts`（`Command` 类型、`getCommands` 函数）

---

## 1. Technical Context

### 1.1 模块定位

`007-slash-command` 是聊天输入框的辅助组件模块，提供 `/` 前缀触发的斜杠命令选择菜单。模块仅包含一个核心组件 `SlashCommandMenu`，通过 `forwardRef` 暴露键盘导航句柄，由父组件（`InputBox`）统一处理键盘事件。

### 1.2 文件清单

| 文件                        | 行数 | 职责                                                                 |
| --------------------------- | ---- | -------------------------------------------------------------------- |
| `SlashCommandMenu.tsx`      | 242  | 核心组件：命令菜单渲染、加载、过滤、导航、外部点击关闭               |
| `SlashCommandMenu.test.tsx` | 49   | 单元测试：加载与过滤场景                                             |
| `index.ts`                  | 2    | Barrel 导出：`SlashCommandMenu` 组件与 `SlashCommandMenuHandle` 类型 |

### 1.3 外部依赖

| 依赖              | 来源                         | 用途                                                       |
| ----------------- | ---------------------------- | ---------------------------------------------------------- |
| `Command` 类型    | `src/api/command.ts`         | 命令实体定义（`name`、`description`、`keybind`、`source`） |
| `getCommands`     | `src/api/command.ts`         | 从后端 API 加载命令列表，含缓存、防重入、前后端命令合并    |
| `apiErrorHandler` | `src/utils/errorHandling.ts` | 命令加载失败时的错误日志                                   |
| `useTranslation`  | `react-i18next`              | 国际化文本（`commands`、`common` 命名空间）                |

### 1.4 被依赖关系

- **`002-chat-feature`**：`InputBox` 组件嵌入 `SlashCommandMenu`，通过 `isOpen`/`query` 控制显隐，通过 ref 调用导航方法
- **`001-api-layer`**：`command.ts` 提供命令数据源，含 TTL 缓存（10s）和 inflight 去重

---

## 2. Constitution Check

对照项目宪法逐条审查：

| 宪法原则                    | 符合性 | 说明                                                                           |
| --------------------------- | ------ | ------------------------------------------------------------------------------ |
| 原则 2：OpenCode 兼容性优先 | ✅     | 命令列表通过 `@opencode-ai/sdk` 的 `sdk.command.list()` 获取，忠实实现后端能力 |
| 原则 3：多平台统一代码库    | ✅     | 无平台特定代码，Web 与 Tauri 共享同一组件                                      |
| 原则 4：自定义优于框架依赖  | ✅     | 未引入额外 UI 库，菜单交互完全自定义实现                                       |
| 原则 6：中文优先文档        | ✅     | 代码注释使用中文，i18n key 通过翻译文件提供多语言                              |
| 原则 10：模块化功能架构     | ✅     | 自包含目录，仅通过明确接口（`getCommands`、`Command` 类型）与外部通信          |
| 约束 C4：依赖最小化         | ✅     | 仅依赖已有的 `react-i18next` 和 API 层，零新增依赖                             |

---

## 3. Research Findings

### 3.1 命令数据流

`getCommands` 函数实现了三层策略：

1. **缓存层**：以 `serverId::language::directory` 为 key，TTL 为 10 秒
2. **Inflight 去重**：同一 key 的并发请求共享同一个 Promise，避免重复请求
3. **命令合并**：后端 API 命令（`source: 'api'`）优先，前端内置命令（`new`、`compact`，`source: 'frontend'`）作为降级补充，同名时以后端为准

### 3.2 与 spec.md 的差异

| spec 要求                                                    | 实际实现                                               | 差异说明                                                                |
| ------------------------------------------------------------ | ------------------------------------------------------ | ----------------------------------------------------------------------- |
| FR-02：前端内置命令包括 `new` 和 `compact`                   | ✅ 在 `command.ts` 的 `getFrontendCommands()` 中硬编码 | 实现位置在 API 层而非组件层，组件仅消费合并后的结果                     |
| FR-06：`maxHeight` 默认值 `min(320px, calc(100dvh - 10rem))` | ✅ 内联 style 中直接使用该 CSS 表达式                  | `dynamicMaxHeight` 为 `undefined` 时回退到此默认值                      |
| FR-09：空状态提示区分"无匹配命令"和"无可用命令"              | ✅ 根据 `query` 是否为空选择不同 i18n key              | `slashCommand.noMatchingCommands` vs `slashCommand.noCommandsAvailable` |
| ADR-001：使用 `requestAnimationFrame`                        | ✅ 命令加载、高度计算、query 重置均通过 rAF 调度       | 与测试框架的假定时器兼容                                                |

### 3.3 测试覆盖现状

当前仅有一个测试用例 `loads and filters commands based on query`，覆盖场景有限。缺失的测试场景包括：

- `isOpen=false` 时不渲染任何内容
- 键盘导航（`moveUp`/`moveDown`/`selectCurrent`）
- 点击外部关闭
- 空状态提示
- 加载状态展示
- 边界保护（索引越界）

---

## 4. Data Model

### 4.1 Command 实体

```typescript
interface Command {
  name: string // 命令名称，如 "compact"、"new"
  description?: string // 命令描述，用于展示和过滤
  keybind?: string // 快捷键绑定，如 "Ctrl+K"
  source: 'frontend' | 'api' // 命令来源
}
```

**数据来源**：

- `source: 'api'`：通过 `sdk.command.list({ directory })` 从后端获取，映射时添加 `source` 字段
- `source: 'frontend'`：前端硬编码，当前为 `new`（新建会话）和 `compact`（压缩会话）

**合并规则**：

```
最终列表 = [...apiCommands, ...frontendCommands.filter(c => !apiNames.has(c.name))]
```

即后端命令优先，前端命令仅在无同名后端命令时补充。

### 4.2 组件内部状态

| 状态               | 类型                  | 初始值      | 说明                                   |
| ------------------ | --------------------- | ----------- | -------------------------------------- |
| `commands`         | `Command[]`           | `[]`        | 完整命令列表（来自 API + 前端内置）    |
| `selectedIndex`    | `number`              | `0`         | 当前选中索引（基于完整列表，非过滤后） |
| `loading`          | `boolean`             | `false`     | 是否正在加载命令                       |
| `dynamicMaxHeight` | `number \| undefined` | `undefined` | 动态计算的最大高度                     |

### 4.3 派生状态

| 派生值               | 计算方式                                                            | 依赖                                |
| -------------------- | ------------------------------------------------------------------- | ----------------------------------- |
| `filteredCommands`   | `commands.filter(cmd => name/description 包含 query)`               | `commands`, `isOpen`, `query`       |
| `commandColumnWidth` | `min(max(maxCommandLength + 1, 10), 18)ch`                          | `commands`                          |
| `activeIndex`        | `min(selectedIndex, filteredCommands.length - 1)`（列表为空时为 0） | `selectedIndex`, `filteredCommands` |

### 4.4 状态机

```
初始状态: commands=[], selectedIndex=0, loading=false, dynamicMaxHeight=undefined

isOpen=true 触发:
  → loading=true
  → requestAnimationFrame → getCommands(rootPath)
    → 成功: commands=结果, selectedIndex=0, loading=false
    → 失败: commands=[], loading=false (通过 apiErrorHandler 记录)

query 变化:
  → requestAnimationFrame → selectedIndex=0

filteredCommands 变化:
  → activeIndex 自动调整到有效范围
  → scrollIntoView 滚动选中项到可见区域
```

---

## 5. Interface Contracts

### 5.1 Props 接口

```typescript
interface SlashCommandMenuProps {
  isOpen: boolean // 菜单是否打开，为 false 时返回 null
  query: string // "/" 之后的查询文本，用于过滤
  rootPath?: string // 当前工作目录，传递给 getCommands
  onSelect: (command: Command) => void // 命令选中回调
  onClose: () => void // 菜单关闭回调（外部点击时触发）
}
```

### 5.2 暴露句柄（Imperative Handle）

```typescript
interface SlashCommandMenuHandle {
  moveUp: () => void // 上移选中项，最小为 0
  moveDown: () => void // 下移选中项，最大为 filteredCommands.length - 1
  selectCurrent: () => void // 执行当前高亮项的 onSelect 回调
  getSelectedCommand: () => Command | null // 返回当前高亮命令，无选中时返回 null
}
```

**使用方式**：

```typescript
const menuRef = useRef<SlashCommandMenuHandle>(null)
// 父组件在 onKeyDown 中调用:
// menuRef.current?.moveDown()
// menuRef.current?.selectCurrent()
```

### 5.3 与 InputBox 的集成契约

`SlashCommandMenu` 不自行检测 `/` 触发，而是接收父组件传入的 `isOpen` 和 `query`。父组件负责：

1. 监听输入框的 `keydown` 事件，检测 `/` 字符
2. 设置 `isOpen=true`，提取 `/` 后的文本作为 `query`
3. 在 `ArrowUp`/`ArrowDown`/`Enter`/`Escape` 时调用 ref 暴露的方法
4. 在 `onSelect` 回调中执行命令逻辑并关闭菜单

---

## 6. Implementation Strategy

### 6.1 组件架构

```
SlashCommandMenu (forwardRef)
├── menuRef (根容器 div)
│   ├── 定位: absolute, bottom: 100%, marginBottom: 8px
│   ├── 样式: glass + border + rounded-xl + shadow-lg
│   ├── 高度: dynamicMaxHeight ?? min(320px, calc(100dvh - 10rem))
│   │
│   ├── listRef (滚动容器 div)
│   │   ├── loading 状态 → 显示 t('common:loading')
│   │   ├── 空状态 → 根据 query 显示不同提示
│   │   └── 命令列表 → map 渲染 button 项
│   │       ├── 命令名称: 等宽字体, 动态宽度, 前缀 /
│   │       ├── 命令描述: 小字号, truncate
│   │       └── 快捷键: 等宽字体, 仅当 keybind 存在时显示
│   │
│   └── Footer Hints (hidden md:flex)
│       ├── t('common:upDownSelect')
│       ├── t('common:enterRun')
│       └── t('common:escCancel')
```

### 6.2 关键实现细节

#### 6.2.1 动态高度计算

使用 `useLayoutEffect` 在布局完成后计算可用空间：

```
available = parentRect.top - 56 - 16 - 8
```

其中 `56` 为 header 高度，`16` 和 `8` 为间距余量。当 `0 < available < 360` 时使用动态高度，否则使用 CSS 默认值。监听 `window.resize` 和 `window.visualViewport.resize` 以响应窗口变化。

#### 6.2.2 请求防重入

通过 `requestIdRef` 实现竞态保护：

```typescript
const requestId = ++requestIdRef.current
// ... 异步请求完成后
if (requestId !== requestIdRef.current) return // 丢弃过期结果
```

同时 `getCommands` 自身也有 inflight 去重和 TTL 缓存，形成双重保护。

#### 6.2.3 点击外部关闭

使用 `pointerdown` 事件（而非 `click`），在 `document` 级别监听。通过 `contains()` 判断点击目标是否在菜单元素内。组件卸载或 `isOpen` 变化时正确移除监听器。

#### 6.2.4 选中项自动滚动

通过 `useEffect` 监听 `activeIndex` 变化，调用 `scrollIntoView({ block: 'nearest' })` 确保高亮项始终可见。

### 6.3 渲染条件

| 条件                                                  | 渲染内容                 |
| ----------------------------------------------------- | ------------------------ |
| `!isOpen`                                             | `null`（不渲染任何 DOM） |
| `loading=true`                                        | 加载提示文本             |
| `!loading && filteredCommands.length === 0 && query`  | "无匹配命令"提示         |
| `!loading && filteredCommands.length === 0 && !query` | "无可用命令"提示         |
| `!loading && filteredCommands.length > 0`             | 命令列表 + Footer Hints  |

---

## 7. Error Handling

### 7.1 错误场景与处理策略

| 错误场景                             | 处理方式                                                     | 用户体验                           |
| ------------------------------------ | ------------------------------------------------------------ | ---------------------------------- |
| 后端 API 不可达                      | `catch` 中调用 `apiErrorHandler` 记录日志，`setCommands([])` | 菜单仅展示前端内置命令，不阻断交互 |
| 请求竞态（快速开关菜单）             | `requestIdRef` 机制丢弃过期结果                              | 不会出现旧数据覆盖新状态           |
| 组件卸载时请求仍在进行               | `useEffect` cleanup 取消 rAF，`requestId` 检查丢弃结果       | 避免对已卸载组件调用 `setState`    |
| 父元素不存在（offsetParent 为 null） | `setDynamicMaxHeight(undefined)`，回退到 CSS 默认值          | 菜单正常显示，使用保守高度         |

### 7.2 错误日志

使用 `apiErrorHandler('load commands', err)` 记录错误，该处理器通过 `createErrorHandler('api')` 创建，在开发环境下输出 `console.error`，生产环境下静默记录。

---

## 8. Testing Considerations

### 8.1 当前测试覆盖

现有测试文件 `SlashCommandMenu.test.tsx` 仅包含 1 个测试用例：

- ✅ `loads and filters commands based on query`：验证命令加载和过滤功能

测试配置：

- 使用 `vi.useFakeTimers()` 模拟定时器
- Mock `requestAnimationFrame` 为 `setTimeout(cb, 16)`
- Mock `getCommands` 返回固定数据

### 8.2 建议补充的测试场景

| 优先级 | 测试场景                       | 验证内容                                   |
| ------ | ------------------------------ | ------------------------------------------ |
| P0     | `isOpen=false` 时返回 null     | 组件不渲染 DOM                             |
| P0     | 键盘导航：moveDown 边界保护    | 索引不超过 `filteredCommands.length - 1`   |
| P0     | 键盘导航：moveUp 边界保护      | 索引不低于 0                               |
| P0     | selectCurrent 触发 onSelect    | 当前高亮命令被传递给回调                   |
| P0     | getSelectedCommand 返回值      | 有选中时返回 Command，无选中时返回 null    |
| P1     | 点击外部触发 onClose           | `pointerdown` 事件在菜单外时调用 onClose   |
| P1     | 点击内部不触发 onClose         | `pointerdown` 事件在菜单内时不调用 onClose |
| P1     | 空查询展示全部命令             | `query=""` 时不过滤                        |
| P1     | 无匹配时显示"无匹配命令"       | `query="xyz"` 无结果时显示正确提示         |
| P2     | 加载状态展示                   | `loading=true` 时显示加载文本              |
| P2     | query 变化时重置 selectedIndex | 过滤条件变化后选中索引归零                 |
| P2     | 动态高度计算                   | 小屏幕下 `dynamicMaxHeight` 被正确设置     |

### 8.3 测试技术要点

- **Fake timers**：必须 mock `requestAnimationFrame`，因为组件内多处使用 rAF 调度
- **Ref 测试**：通过 `render` 的 `ref` 参数获取 `SlashCommandMenuHandle`，调用暴露方法
- **事件模拟**：使用 `fireEvent.pointerDown` 模拟外部点击
- **i18n 配置**：测试环境需初始化 i18n 或 mock `useTranslation`

---

## 9. 风险与注意事项

### 9.1 已识别风险

| 风险                                                         | 影响                                                                                     | 当前缓解措施                                           |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `selectedIndex` 基于完整列表而非过滤后列表                   | 过滤后 `activeIndex` 通过 `Math.min` 修正，但 `selectedIndex` 本身不重置为过滤后的有效值 | `query` 变化时通过 `useEffect` 重置 `selectedIndex=0`  |
| `commandColumnWidth` 依赖 `commands` 而非 `filteredCommands` | 过滤后列宽可能大于实际需要                                                               | 可接受，列宽上限 18ch 防止过度膨胀                     |
| `useImperativeHandle` 依赖 `onSelect`                        | `onSelect` 每次渲染可能为新引用，导致句柄重建                                            | 父组件应使用 `useCallback` 稳定引用                    |
| 移动端虚拟键盘弹出时高度计算                                 | `visualViewport.resize` 已监听，但 `available` 计算未考虑键盘高度                        | 当前通过 `parentRect.top` 间接适配，极端情况可能不精确 |

### 9.2 与 CommandPalette 的区别

`SlashCommandMenu` 与全局 `CommandPalette`（`Ctrl/Cmd+K`）是两个独立组件：

- **SlashCommandMenu**：聊天上下文内，影响当前对话，命令来源为后端 API + 前端内置
- **CommandPalette**：应用级别，影响全局操作（如切换主题、打开设置），命令来源为前端注册

两者不应混淆，各自有独立的触发机制和命令集。

---

## 10. 总结

`007-slash-command` 模块实现精简而完整，核心组件 `SlashCommandMenu` 仅 242 行代码即覆盖了命令加载、过滤、导航、响应式高度、外部点击关闭等全部功能需求。设计上遵循了以下原则：

- **职责单一**：组件仅负责菜单渲染与交互，命令执行逻辑由父组件处理
- **降级友好**：后端不可达时自动降级为前端内置命令，不阻断用户操作
- **性能优化**：`useMemo` 缓存过滤结果和列宽计算，`requestAnimationFrame` 避免布局抖动
- **可访问性**：键盘导航句柄通过 `forwardRef` 暴露，与父组件键盘事件无缝集成

测试覆盖目前较薄弱（仅 1 个用例），建议按第 8 节补充关键场景的单元测试。
