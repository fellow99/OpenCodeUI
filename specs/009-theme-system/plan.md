# 009-Theme-System 技术方案（As-Built）

> 本文档是对已完成主题系统的回溯性技术规划，记录"实际建成"的架构设计、模块分解与集成策略。

---

## 1. Technical Context

### 1.1 模块定位

主题系统（Theme System）是 OpenCodeUI 的视觉基础设施层，为所有 UI 组件提供颜色、字体、毛玻璃效果等外观能力。它不依赖任何其他业务模块，但被几乎所有 UI 模块消费。

### 1.2 技术栈

| 维度     | 选型                                                |
| -------- | --------------------------------------------------- |
| 框架     | React 19 + TypeScript                               |
| 样式     | Tailwind CSS v4 + CSS Custom Properties             |
| 状态管理 | 自定义 Store（模块级单例 + `useSyncExternalStore`） |
| 动画     | View Transition API（降级为即时切换）               |
| 持久化   | localStorage（16 个独立 key）                       |

### 1.3 源码文件清单

| 文件                                                      | 行数 | 职责                                                    |
| --------------------------------------------------------- | ---- | ------------------------------------------------------- |
| `src/themes/index.ts`                                     | 511  | 主题预设定义、类型声明、`themeColorsToCSSVars` 转换函数 |
| `src/store/themeStore.ts`                                 | 586  | 主题状态管理、DOM 注入、持久化、订阅机制                |
| `src/hooks/useTheme.ts`                                   | 251  | React Hook 封装，提供带动画的切换方法                   |
| `src/features/settings/components/AppearanceSettings.tsx` | 457  | 设置面板外观选项卡 UI                                   |
| `src/constants/animation.ts`                              | 2    | `THEME_SWITCH_DISABLE_MS = 400`                         |

### 1.4 依赖关系

```
009-theme-system（无上游依赖）
    ↓ 被消费
002-chat-feature（ChatArea、InputBox 等通过 Tailwind 变量引用颜色）
003-message-rendering（MarkdownRenderer、CodeBlock 通过 CSS 变量适配主题）
005-settings-panel（AppearanceSettings 直接消费 useTheme）
008-terminal-system（Terminal 从 CSS 变量读取颜色，自动适配主题）
011-file-diff-viewer（DiffView 通过 CSS 变量适配颜色）
```

---

## 2. Constitution Check

| 宪法原则                   | 符合性 | 说明                                                                                       |
| -------------------------- | ------ | ------------------------------------------------------------------------------------------ |
| 原则 3：多平台统一代码库   | ✅     | Web 与 Tauri 共享同一套主题代码，通过 `data-mode` 属性驱动，无平台分支                     |
| 原则 4：自定义优于框架依赖 | ✅     | 采用自定义 Store 模式（非 Zustand/Redux），零额外依赖                                      |
| 原则 9：主题与可访问性     | ✅     | 三套内置主题 + 明暗模式 + 自定义 CSS + 毛玻璃效果，完全覆盖                                |
| 原则 10：模块化功能架构    | ✅     | `src/themes/` 与 `src/store/themeStore.ts` 自包含，通过明确接口（`useTheme` Hook）对外暴露 |
| 约束 C4：依赖最小化        | ✅     | 无新增第三方依赖，全部使用浏览器原生 API（`matchMedia`、`startViewTransition`）            |

---

## 3. Research Findings

### 3.1 主题预设架构

系统定义了 3 套内置主题预设，每套包含 light/dark 两套配色：

| 预设 ID      | 名称       | 品牌色 HSL              | 设计理念                                   |
| ------------ | ---------- | ----------------------- | ------------------------------------------ |
| `eucalyptus` | Eucalyptus | `165 45% 42%`（桉树绿） | 莫兰迪色系，高级灰调，清爽冷静（默认主题） |
| `claude`     | Claude     | `24 90% 50%`（暖橙）    | 暖调橙色品牌色，经典风格                   |
| `breeze`     | Breeze     | `187 72% 42%`（青绿）   | 冷色调蓝绿，清新护眼，低饱和度             |

此外提供 `custom` 预设选项，允许用户通过自定义 CSS 完全接管主题配色。Custom 模式下，系统以 Eucalyptus 主题作为 fallback 底色，用户 CSS 在其上覆盖。

### 3.2 颜色格式与变量体系

所有颜色值采用 **HSL 不带 `hsl()` 包装** 的格式（如 `165 45% 42%`），由样式层统一包裹。变量命名遵循语义化规范：

**背景色（5 级）：** `--bg-000` ~ `--bg-400`
**文本色（7 级）：** `--text-000` ~ `--text-600`
**品牌色（5 个）：** `--accent-brand`、`--accent-main-000/100/200`、`--accent-secondary-100`
**语义色（14 个）：** `--success-*`、`--warning-*`、`--danger-*`、`--info-*`（含前景色与背景色）
**边框色（3 级）：** `--border-100/200/300`
**特殊色（3 个）：** `--always-black`、`--always-white`、`--oncolor-100`

总计 **37 个 CSS 自定义属性**，通过 `themeColorsToCSSVars()` 函数从 `ThemeColors` 对象生成。

### 3.3 CSS 注入策略

主题变量通过 `<style id="opencode-theme-vars">` 元素注入 `<head>`，使用 `:root:root` 选择器（特异性 0-2-0）覆盖 `index.css` 中的默认值。自定义 CSS 通过独立的 `<style id="opencode-custom-css">` 元素注入，优先级最高。

```
:root（index.css 默认值）
    ↓ 被覆盖
:root:root（opencode-theme-vars，动态注入）
    ↓ 被覆盖
:root:root（opencode-custom-css，用户自定义，最高优先级）
```

### 3.4 初始化时序

主题初始化在 `main.tsx` 中 **React 渲染前** 执行：

```
main.tsx
  ├─ ensureRandomUUID()
  ├─ history.scrollRestoration = 'manual'
  ├─ themeStore.init()          ← 注入 CSS 变量 + 监听系统主题变化
  ├─ initOverlayScrollbars()
  ├─ serverStore.onServerChange(...)
  ├─ Tauri 初始化（条件分支）
  ├─ 全局错误处理
  └─ bootstrap() → createRoot().render()
```

`themeStore.init()` 执行三件事：

1. `applyTheme()` — 注入当前主题的 CSS 变量到 DOM
2. `applyGlassClass()` — 根据 `glassEffect` 设置 `data-glass` 属性
3. 注册 `prefers-color-scheme` 变化监听器

### 3.5 主题切换动画

`useTheme` Hook 提供两套切换方法：

| 方法                                                                                | 行为                                      |
| ----------------------------------------------------------------------------------- | ----------------------------------------- |
| `setTheme(newMode)` / `setPreset(presetId)`                                         | 即时切换，无动画                          |
| `setThemeWithAnimation(newMode, event)` / `setPresetWithAnimation(presetId, event)` | 使用 View Transition API 实现圆形扩散动画 |

动画实现细节：

- 以点击位置为圆心，计算到最远角的距离作为结束半径
- 使用 `clipPath: circle()` 从 0px 扩散到最大半径
- 动画时长 380ms，缓动函数 `cubic-bezier(0.4, 0, 0.2, 1)`
- 通过 `data-theme-transition="off"` 标记在动画期间禁用 CSS 过渡
- 动画结束后延迟 `THEME_SWITCH_DISABLE_MS`（400ms）移除标记
- 浏览器不支持 View Transition API 时自动降级为即时切换

### 3.6 移动端状态栏适配

主题切换时通过 `requestAnimationFrame` 更新移动端状态栏颜色：

1. 读取 `--color-bg-100` 计算后的颜色值
2. 通过 `computedColorToHex()` 将任意格式（rgb/oklch/srgb）转为 `#RRGGBB`
3. 更新 `<meta name="theme-color">` 的 `content` 属性
4. 调用 Android Bridge `__opencode_android.setSystemBars(mode, hex)`（如果存在）

`computedColorToHex()` 使用 Canvas 2D 做万能颜色转换，兼容浏览器返回的各种颜色格式。

---

## 4. Data Model

### 4.1 核心类型定义

```typescript
// 颜色模式枚举
type ColorMode = 'system' | 'light' | 'dark'

// 单套配色（light 或 dark）
interface ThemeColors {
  background: { bg000: string; bg100: string; bg200: string; bg300: string; bg400: string }
  text: {
    text000: string
    text100: string
    text200: string
    text300: string
    text400: string
    text500: string
    text600: string
  }
  accent: { brand: string; main000: string; main100: string; main200: string; secondary100: string }
  semantic: {
    success100: string
    success200: string
    successBg: string
    warning100: string
    warning200: string
    warningBg: string
    danger000: string
    danger100: string
    danger200: string
    dangerBg: string
    danger900: string
    info100: string
    info200: string
    infoBg: string
  }
  border: { border100: string; border200: string; border300: string }
  special?: { alwaysBlack?: string; alwaysWhite?: string; oncolor100?: string }
}

// 主题预设
interface ThemePreset {
  id: string
  name: string
  description: string
  light: ThemeColors
  dark: ThemeColors
}
```

### 4.2 ThemeState 完整状态

`ThemeState` 聚合了 16 个外观相关设置项：

| 属性                      | 类型                 | 默认值         | Storage Key                 |
| ------------------------- | -------------------- | -------------- | --------------------------- |
| `presetId`                | string               | `'eucalyptus'` | `theme-preset`              |
| `colorMode`               | ColorMode            | `'system'`     | `theme-mode`                |
| `customCSS`               | string               | `''`           | `theme-custom-css`          |
| `collapseUserMessages`    | boolean              | `true`         | `collapse-user-messages`    |
| `stepFinishDisplay`       | StepFinishDisplay    | 全 true        | `step-finish-display`       |
| `reasoningDisplayMode`    | ReasoningDisplayMode | `'capsule'`    | `reasoning-display-mode`    |
| `wideMode`                | boolean              | `false`        | `chat-wide-mode`            |
| `diffStyle`               | DiffStyle            | `'markers'`    | `diff-style`                |
| `descriptiveToolSteps`    | boolean              | `false`        | `descriptive-tool-steps`    |
| `inlineToolRequests`      | boolean              | `false`        | `inline-tool-requests`      |
| `codeWordWrap`            | boolean              | `false`        | `code-word-wrap`            |
| `toolCardStyle`           | ToolCardStyle        | `'classic'`    | `tool-card-style`           |
| `immersiveMode`           | boolean              | `false`        | `immersive-mode`            |
| `compactInlinePermission` | boolean              | `false`        | `compact-inline-permission` |
| `glassEffect`             | boolean              | `true`         | `glass-effect`              |
| `queueFollowupMessages`   | boolean              | `false`        | `queue-followup-messages`   |

### 4.3 子类型

```typescript
interface StepFinishDisplay {
  tokens: boolean
  cache: boolean
  cost: boolean
  duration: boolean
  turnDuration: boolean
}

type ReasoningDisplayMode = 'capsule' | 'italic' | 'markdown'
type DiffStyle = 'markers' | 'changeBars'
type ToolCardStyle = 'classic' | 'compact'
```

### 4.4 沉浸模式联动

`setImmersiveMode(true)` 会联动修改 4 个子设置：

- `inlineToolRequests = true`
- `descriptiveToolSteps = true`
- `toolCardStyle = 'compact'`
- `compactInlinePermission = true`

同时写入 5 个 localStorage key，确保状态一致性。

---

## 5. Interface Contracts

### 5.1 ThemeStore 公共 API

```typescript
class ThemeStore {
  // 状态读取
  getState(): ThemeState
  getSnapshot(): ThemeState
  subscribe(listener: () => void): () => void

  // Getters（便捷属性）
  presetId: string
  colorMode: ColorMode
  customCSS: string
  isDark: boolean
  // ... 其余 14 个 getter

  // 主题预设
  getPreset(): ThemePreset | undefined
  getAvailablePresets(): { id: string; name: string; description: string }[]

  // 模式解析
  getResolvedMode(): 'light' | 'dark'

  // 初始化
  init(): void

  // 状态变更（自动持久化 + 触发更新）
  setPreset(id: string): void
  setColorMode(mode: ColorMode): void
  setCustomCSS(css: string): void
  setGlassEffect(enabled: boolean): void
  setImmersiveMode(enabled: boolean): void
  // ... 其余 12 个 setter
}
```

### 5.2 useTheme Hook 返回值

```typescript
function useTheme(): {
  // 日夜模式
  mode: ColorMode
  resolvedTheme: 'light' | 'dark'
  isDark: boolean
  setTheme: (mode: ColorMode) => void
  toggleTheme: () => void
  setThemeWithAnimation: (mode: ColorMode, event?: MouseEvent) => void
  setThemeImmediate: (mode: ColorMode) => void

  // 主题风格
  presetId: string
  setPreset: (id: string) => void
  setPresetWithAnimation: (id: string, event?: MouseEvent) => void
  availablePresets: { id: string; name: string; description: string }[]

  // 自定义 CSS
  customCSS: string
  setCustomCSS: (css: string) => void

  // 其余 14 组 state + setter
  // ...
}
```

### 5.3 themes/index.ts 导出

```typescript
// 类型
export interface ThemeColors { ... }
export interface ThemePreset { ... }

// 预设实例
export const eucalyptusTheme: ThemePreset
export const claudeTheme: ThemePreset
export const breezeTheme: ThemePreset
export const builtinThemes: ThemePreset[]
export const DEFAULT_THEME_ID = 'eucalyptus'

// 工具函数
export function getThemePreset(id: string): ThemePreset | undefined
export function themeColorsToCSSVars(theme: ThemeColors): string
```

---

## 6. Implementation Strategy

### 6.1 数据流

```
用户操作（设置面板 / 快捷键）
    ↓
useTheme Hook（setThemeWithAnimation / setPresetWithAnimation）
    ↓
themeStore.setColorMode() / setPreset()
    ↓
├─ 更新内存状态（不可变拷贝）
├─ 写入 localStorage
├─ applyTheme() → 注入 CSS 变量到 <style#opencode-theme-vars>
├─ applyCustomCSS() → 注入用户 CSS 到 <style#opencode-custom-css>
├─ applyGlassClass() → 设置/移除 data-glass 属性
├─ 更新 meta theme-color + Android Bridge
└─ emit() → 通知所有订阅者
    ↓
useSyncExternalStore 触发 React 重渲染
    ↓
组件通过 Tailwind 变量（如 bg-bg-100、text-text-100）自动响应新颜色
```

### 6.2 系统主题检测

```typescript
// 在 themeStore.init() 中注册
const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
mediaQuery.addEventListener('change', () => {
  if (this.state.colorMode === 'system') {
    this.applyTheme() // 重新注入对应配色的 CSS 变量
    this.emit() // 通知订阅者
  }
})
```

仅当 `colorMode === 'system'` 时响应系统变化，避免与用户手动设置冲突。

### 6.3 自定义 CSS 编辑器

- 使用 `<textarea>` 作为编辑器（非 Monaco/CodeMirror，保持零依赖）
- 防抖 400ms 后应用（`debounceRef`）
- 提供"加载模板"按钮，填入包含完整亮色/暗色/系统自动三套配色的模板代码
- 模板基于 One Dark Pro 配色（MIT 许可证）
- 提供"清除"按钮移除自定义 CSS

### 6.4 毛玻璃效果

通过 `data-glass` 属性驱动 CSS：

- `themeStore.setGlassEffect(true)` → `document.documentElement.setAttribute('data-glass', '')`
- CSS 中通过 `[data-glass] .glass` 选择器应用 `backdrop-filter: blur()` 效果
- 关闭时移除属性，元素回退到实色背景

---

## 7. Error Handling

### 7.1 自定义 CSS 容错

- 自定义 CSS 直接作为 `<style>` 元素的 `textContent` 注入，浏览器会忽略无效 CSS
- 不影响主题系统正常运行（NFR-05 容错性要求）
- `applyCustomCSS()` 在 CSS 为空时移除 `<style>` 元素

### 7.2 颜色转换容错

`computedColorToHex()` 使用 try-catch 包裹 Canvas 操作：

- Canvas 2D context 获取失败时返回 `null`
- 颜色格式不匹配时返回 `null`
- 返回 `null` 时跳过 meta theme-color 更新，不影响主题切换

### 7.3 localStorage 容错

- `stepFinishDisplay` 的 JSON 解析使用 try-catch，解析失败时使用默认值
- 所有 boolean 类型的读取使用显式字符串比较（`=== 'true'`），避免 truthy 误判
- 枚举类型（`ColorMode`、`DiffStyle`、`ToolCardStyle`、`ReasoningDisplayMode`）使用白名单校验，非法值回退到默认值

### 7.4 View Transition API 降级

```typescript
if (!document.startViewTransition || !event) {
  // 降级：即时切换，无动画
  themeStore.setColorMode(newMode)
  return
}
```

浏览器不支持 View Transition API 或缺少点击事件对象时，自动降级为即时切换（NFR-02 兼容性要求）。

### 7.5 prefers-reduced-motion

当前实现未显式检测 `prefers-reduced-motion`。View Transition API 的动画由浏览器原生控制，但 380ms 的 clipPath 动画不受该媒体查询影响。这是一个潜在的无障碍改进点（NFR-03）。

---

## 8. Testing Considerations

### 8.1 当前测试覆盖

**主题系统目前没有专属单元测试文件。** 以下方面建议补充测试：

### 8.2 建议测试用例

**单元测试（themeStore）：**

- `getResolvedMode()` 在 system/light/dark 三种模式下的返回值
- `setImmersiveMode(true)` 联动修改 4 个子设置的正确性
- `themeColorsToCSSVars()` 生成的 CSS 变量字符串格式正确性
- localStorage 读取异常值时的默认值回退行为
- `subscribe` / `emit` 订阅机制的正确性

**组件测试（AppearanceSettings）：**

- 主题预设卡片点击后正确调用 `setPresetWithAnimation`
- SegmentedControl 切换颜色模式
- 自定义 CSS 编辑器的防抖行为（400ms）
- "加载模板"按钮填入模板代码
- "清除"按钮清空自定义 CSS
- 毛玻璃效果开关切换

**集成测试：**

- `themeStore.init()` 在 React 渲染前正确注入 CSS 变量
- 系统主题变化时自动响应（mock `matchMedia`）
- 主题切换后 `<style#opencode-theme-vars>` 内容更新
- 页面刷新后从 localStorage 恢复主题状态

### 8.3 测试难点

- `document.startViewTransition` 在测试环境中可能不可用，需要 mock
- `window.matchMedia` 需要 mock 才能测试系统主题检测
- Canvas 2D 颜色转换在 Node.js 环境中不可用，需要 mock 或跳过
- `useSyncExternalStore` 的测试需要 Testing Library 的 `renderHook`

---

## 9. 风险与改进点

| 风险                                 | 等级 | 说明                                                       |
| ------------------------------------ | ---- | ---------------------------------------------------------- |
| 无 `prefers-reduced-motion` 检测     | 低   | NFR-03 要求尊重用户动画偏好，当前未实现                    |
| 自定义 CSS 无语法校验                | 低   | 无效 CSS 被浏览器静默忽略，用户无反馈                      |
| ThemeState 职责过重                  | 中   | 16 个设置项中仅 3 个与"主题"直接相关，其余属于 UI 偏好设置 |
| 无主题测试文件                       | 中   | 核心模块缺乏自动化测试覆盖                                 |
| `computedColorToHex` 每次创建 Canvas | 低   | 频繁调用可能产生 GC 压力，可复用单一 Canvas 实例           |
