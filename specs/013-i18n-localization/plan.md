# 013-i18n-localization 技术方案（As-Built）

> 本文档是对已完成项目的回溯性技术规划，记录 i18n 国际化模块"实际建成"的架构设计、实现细节与集成策略。

---

## 1. Technical Context

### 1.1 模块定位

i18n 模块是 OpenCodeUI 的基础设施层，为所有 UI 组件提供多语言文本渲染能力。当前支持简体中文（`zh-CN`）和英语（`en`）两种语言。

### 1.2 技术栈

| 依赖包                             | 版本约束 | 职责                                |
| ---------------------------------- | -------- | ----------------------------------- |
| `i18next`                          | 核心引擎 | 翻译引擎、插值、回退链              |
| `react-i18next`                    | React 桥 | `useTranslation` Hook、`Trans` 组件 |
| `i18next-browser-languagedetector` | 检测器   | 浏览器语言自动检测                  |

### 1.3 初始化入口

`src/main.tsx` 第 5 行通过 `import './i18n'` 触发 i18n 初始化。这是一个 side-effect import，在 React 渲染前完成翻译资源加载和引擎配置。

### 1.4 目录结构（As-Built）

```
src/
├── i18n.ts                          # i18next 配置与初始化（41 行）
└── locales/
    ├── en/                          # 英语（fallbackLng）
    │   ├── common.json              # 69 个键，全局通用文案
    │   ├── chat.json                # 195 行，聊天界面（最大命名空间）
    │   ├── commands.json            # 斜杠命令
    │   ├── components.json          # 可复用组件
    │   ├── message.json             # 消息渲染
    │   └── settings.json            # 设置面板
    └── zh-CN/                       # 简体中文
        ├── common.json              # 69 个键，与 en 结构一致
        ├── chat.json                # 195 行
        ├── commands.json
        ├── components.json
        ├── message.json
        └── settings.json
```

---

## 2. Constitution Check

| 宪法原则            | 符合性 | 说明                                                  |
| ------------------- | ------ | ----------------------------------------------------- |
| 原则 2：兼容性      | 符合   | i18n 不改变后端 API 行为，纯前端能力                  |
| 原则 3：统一代码库  | 符合   | Web 端和 Tauri 桌面端共享同一套翻译文件               |
| 原则 4：自定义优先  | 符合   | 仅引入 i18next 生态 3 个包，无冗余依赖                |
| 原则 6：中文优先    | 符合   | 文档以中文为主，翻译文件 `zh-CN` 与 `en` 并列         |
| 原则 10：模块化     | 符合   | 翻译按命名空间拆分，每个 namespace 对应独立 JSON 文件 |
| 约束 C4：依赖最小化 | 符合   | 仅 3 个 i18n 相关依赖，无功能重叠库                   |

---

## 3. Research Findings

### 3.1 翻译加载机制

`src/i18n.ts` 使用 Vite 的 `import.meta.glob` 实现构建时 eager loading：

```typescript
const modules = import.meta.glob('./locales/*/*.json', { eager: true })
```

glob 模式 `./locales/*/*.json` 匹配所有语言目录下的 JSON 文件。通过正则 `/\.\/locales\/([^/]+)\/([^/]+)\.json$/` 解析出语言代码和命名空间名称，动态构建 `resources` 对象。

**关键设计决策：**

- **Eager Loading 而非 Lazy Loading**：所有翻译在应用启动时一次性加载，避免运行时异步请求导致的文本闪烁
- **动态推导 namespace 列表**：`ns: Object.keys(resources['en'] || {})` 从 `en` 目录自动读取所有可用命名空间，新增语言时无需修改配置
- **fallback 资源处理**：`modules[path].default ?? modules[path]` 兼容 Vite 不同版本的模块导出格式

### 3.2 语言检测与回退

检测优先级：

1. **localStorage**（键名 `i18nextLng`）：用户上次手动选择的语言偏好
2. **navigator.language**：浏览器语言设置

回退链：任意语言缺失翻译键时，自动回退到 `en`。这通过 `fallbackLng: 'en'` 配置实现。

### 3.3 组件使用模式

组件通过 `useTranslation` Hook 消费翻译，存在两种调用方式：

**方式 A：单命名空间**

```typescript
const { t } = useTranslation('chat')
t('header.newChat') // → "New Chat" / "新聊天"
```

**方式 B：多命名空间**

```typescript
const { t } = useTranslation(['common', 'chat'])
t('chat:header.newChat') // 显式指定 namespace
t('common:loading') // 默认 namespace 可省略前缀
```

**实际使用分布：** 项目中 244 个文件引用了 i18n 相关模块。典型组件如 `Header.tsx` 使用 `useTranslation('chat')`，`SettingsDialog.tsx` 使用 `useTranslation(['common', 'settings'])`。

### 3.4 插值能力

**变量插值（`{{variable}}`）：**
翻译字符串中的 `{{变量名}}` 在运行时被替换。例如：

```json
// en/chat.json
"remaining": "{{count}} remaining"
```

```typescript
t('chat:inputFooter.remaining', { count: 5 }) // → "5 remaining"
```

**富文本插值（`<index>content</index>`）：**
部分翻译字符串包含 React 组件级别的占位符。例如：

```json
// zh-CN/chat.json
"pressEscAgain": "再按一次 <1>Esc</1> 停止"
```

渲染时使用 `react-i18next` 的 `Trans` 组件，将 `<1>` 和 `</1>` 替换为 `components` 数组中索引 1 对应的 React 元素。当前代码中 `hints.pressEscAgain` 键使用了此语法。

### 3.5 配置参数汇总

| 参数                           | 实际值                          | 说明                         |
| ------------------------------ | ------------------------------- | ---------------------------- |
| `fallbackLng`                  | `'en'`                          | 默认回退语言                 |
| `defaultNS`                    | `'common'`                      | 默认命名空间                 |
| `ns`                           | 动态推导（`en` 目录下的文件名） | 当前为 6 个命名空间          |
| `detection.order`              | `['localStorage', 'navigator']` | 检测优先级                   |
| `detection.caches`             | `['localStorage']`              | 仅缓存到 localStorage        |
| `detection.lookupLocalStorage` | `'i18nextLng'`                  | localStorage 存储键名        |
| `interpolation.escapeValue`    | `false`                         | 关闭 HTML 转义（React 负责） |

---

## 4. Data Model

### 4.1 核心实体

**Locale（语言区域）：**

```typescript
interface Locale {
  code: string // BCP 47 语言代码，如 'zh-CN'、'en'
  label: string // 显示名称，如 '简体中文'、'English'
}
```

当前实例：

- `{ code: 'zh-CN', label: '简体中文' }`
- `{ code: 'en', label: 'English' }`

**Namespace（命名空间）：**

```typescript
type Namespace = 'common' | 'chat' | 'commands' | 'components' | 'message' | 'settings'
```

每个命名空间对应 `src/locales/{lang}/{namespace}.json` 文件。

**TranslationKey（翻译键）：**

```typescript
interface TranslationKey {
  namespace: string // 所属命名空间
  keyPath: string // 点号分隔路径，如 'chat.header.newChat'
  interpolationVars?: string[] // 插值变量名，如 ['count']
}
```

### 4.2 翻译资源结构

i18next 内部 `resources` 对象的实际结构：

```typescript
{
  'en': {
    common: { loading: 'Loading...', cancel: 'Cancel', ... },
    chat: { header: { newChat: 'New Chat', ... }, ... },
    commands: { ... },
    components: { ... },
    message: { ... },
    settings: { ... }
  },
  'zh-CN': {
    common: { loading: '加载中...', cancel: '取消', ... },
    chat: { header: { newChat: '新聊天', ... }, ... },
    commands: { ... },
    components: { ... },
    message: { ... },
    settings: { ... }
  }
}
```

### 4.3 命名空间职责矩阵

| 命名空间     | 键数量级 | 典型键路径                                | 主要消费者                |
| ------------ | -------- | ----------------------------------------- | ------------------------- |
| `common`     | ~70      | `loading`、`cancel`、`error`              | 所有组件                  |
| `chat`       | ~120     | `header.newChat`、`inputFooter.remaining` | Header、Sidebar、InputBox |
| `commands`   | ~20      | 命令名称和描述                            | SlashCommandMenu          |
| `components` | ~50      | 文件浏览器、终端标签                      | FileExplorer、Terminal    |
| `message`    | ~40      | 消息类型标签、工具调用展示                | MessageRenderer           |
| `settings`   | ~60      | 设置标签、说明文字                        | SettingsDialog            |

---

## 5. Interface Contracts

### 5.1 i18n 模块对外接口

```typescript
// src/i18n.ts 导出
export default i18n // i18next 实例

// 消费方式
import i18n from '@/i18n' // 直接访问 i18next 实例（如切换语言）
import { useTranslation, Trans } from 'react-i18next' // React 组件消费
```

### 5.2 useTranslation Hook 契约

```typescript
function useTranslation(
  ns?: Namespace | Namespace[], // 可选，指定命名空间
  options?: UseTranslationOptions,
): {
  t: TFunction // 翻译函数
  i18n: i18n // i18next 实例
  ready: boolean // 翻译资源是否已加载
}
```

**TFunction 签名：**

```typescript
function t(
  key: string, // 翻译键，如 'chat.header.newChat'
  options?: TOptions, // 插值变量
): string
```

### 5.3 语言切换接口

```typescript
// 切换语言（运行时，无需刷新）
i18n.changeLanguage('zh-CN')

// 当前语言
i18n.language // → 'zh-CN' | 'en'
```

### 5.4 命名空间加载契约

组件声明所需的命名空间后，i18next 确保这些命名空间的翻译数据可用。未声明的命名空间在 `t()` 调用时仍可访问（通过全局 `ns` 配置），但显式声明有助于 Suspense 集成和类型推导。

---

## 6. Implementation Strategy

### 6.1 当前实现概述

i18n 模块已完整实现，核心文件仅 `src/i18n.ts`（41 行），配合 12 个翻译 JSON 文件（`en/` 和 `zh-CN/` 各 6 个）。

### 6.2 关键实现细节

**Vite glob 导入的资源构建流程：**

```
import.meta.glob('./locales/*/*.json', { eager: true })
  ↓
遍历匹配路径，正则提取 lang 和 ns
  ↓
构建 resources[lang][ns] = module.default
  ↓
i18n.init({ resources, fallbackLng: 'en', ... })
```

**翻译键的点号路径解析：**
i18next 内部将 `chat.header.newChat` 解析为 `resources[lang].chat.header.newChat` 的嵌套对象访问。当前 `chat.json` 的最大嵌套深度为 3 层（如 `chat.inputFooter.remaining`），符合 spec 中"不超过 3 层"的建议。

### 6.3 新增语言扩展步骤

如需新增日语（`ja`）支持：

1. 在 `src/locales/` 下创建 `ja/` 目录
2. 复制 `en/` 下的 6 个 JSON 文件到 `ja/`
3. 将每个文件的值翻译为日语，保持键结构不变
4. 无需修改 `i18n.ts`，Vite glob 会自动发现新语言目录
5. 在语言选择器 UI 中添加 `{ code: 'ja', label: '日本語' }` 选项

### 6.4 新增命名空间扩展步骤

如需新增命名空间（如 `errors.json`）：

1. 在 `src/locales/en/` 和 `src/locales/zh-CN/` 下各创建 `errors.json`
2. 无需修改 `i18n.ts`，`ns` 会从 `en` 目录自动推导
3. 组件中通过 `useTranslation(['common', 'errors'])` 消费

---

## 7. Error Handling

### 7.1 翻译键缺失

当 `t()` 调用的键在当前语言和 fallback 语言中均不存在时，i18next 返回原始键名本身。例如 `t('nonexistent.key')` 返回 `'nonexistent.key'`。

**当前防护：** `fallbackLng: 'en'` 确保只要 `en` 中存在该键，其他语言缺失时仍能显示英文文本。

### 7.2 插值变量缺失

翻译字符串包含 `{{count}}` 但调用时未传入 `count` 参数，i18next 保留原始占位符文本 `{{count}}`。

### 7.3 JSON 格式错误

翻译文件必须为合法 JSON。Vite 在构建时会解析 JSON 文件，格式错误将导致构建失败。这是编译时防护，不会在运行时出现。

### 7.4 语言检测失败

当 localStorage 和 navigator 均无法提供有效语言时，i18next 使用 `fallbackLng`（`en`）作为默认语言。

---

## 8. Testing Considerations

### 8.1 单元测试覆盖点

| 测试目标     | 测试内容                                       |
| ------------ | ---------------------------------------------- |
| i18n 初始化  | resources 对象正确构建，包含所有语言和命名空间 |
| 语言检测     | localStorage 优先于 navigator                  |
| 语言切换     | `changeLanguage()` 后 `t()` 返回对应语言文本   |
| 变量插值     | `t('key', { count: 5 })` 正确替换 `{{count}}`  |
| 回退链       | zh-CN 缺失键时自动返回 en 对应值               |
| 命名空间加载 | 组件声明的命名空间数据可用                     |

### 8.2 集成测试场景

| 场景                        | 验证点                                  |
| --------------------------- | --------------------------------------- |
| 首次访问（无 localStorage） | 界面语言匹配浏览器语言或 fallback 到 en |
| 手动切换语言                | 所有组件文本即时更新，无需刷新          |
| 语言偏好持久化              | 关闭浏览器后重新打开，语言设置保留      |
| 缺失翻译键                  | 显示英文版本，不显示原始键名            |
| 变量插值渲染                | `{{count}}` 被正确替换为实际值          |

### 8.3 翻译一致性校验

建议建立自动化校验机制，确保所有语言的翻译文件保持键结构一致：

- 对比 `en/` 和 `zh-CN/` 下同名文件的顶层键和嵌套键是否完全匹配
- 检测 `zh-CN` 中存在但 `en` 中不存在的多余键
- 检测 `en` 中存在但 `zh-CN` 中缺失的键（可能导致 fallback）

---

## 9. 依赖关系

### 9.1 被依赖关系

i18n 模块被几乎所有 UI 模块依赖：

| 依赖方                 | 使用方式                                   |
| ---------------------- | ------------------------------------------ |
| 002-chat-feature       | `useTranslation('chat')`                   |
| 003-message-rendering  | `useTranslation(['common', 'message'])`    |
| 004-session-management | `useTranslation(['common', 'chat'])`       |
| 005-settings-panel     | `useTranslation(['common', 'settings'])`   |
| 006-mention-system     | `useTranslation(['common'])`               |
| 007-slash-command      | `useTranslation(['common', 'commands'])`   |
| 008-terminal-system    | `useTranslation(['common', 'components'])` |
| 011-file-diff-viewer   | `useTranslation(['common', 'components'])` |
| 012-pane-layout        | `useTranslation(['common'])`               |

### 9.2 依赖关系

i18n 模块自身无内部依赖（除 i18next 生态外），是基础设施层的最底层。

---

## 10. 风险与注意事项

### 10.1 翻译文件同步风险

新增翻译键时，容易遗漏某种语言的对应翻译。当前无自动化校验，依赖人工确保 `en/` 和 `zh-CN/` 的键结构一致。

**缓解措施：** 建议在 CI 中添加翻译键一致性检查脚本，对比两种语言的 JSON 文件键树。

### 10.2 构建体积风险

Eager Loading 意味着所有翻译文件都会被打包进最终产物。当前 12 个 JSON 文件总大小约 50KB（未压缩），对构建体积影响可忽略。但未来如果新增大量语言（如 10+），需考虑切换为 Lazy Loading 策略。

### 10.3 富文本插值维护

`<1>Esc</1>` 语法中的索引数字与 `Trans` 组件的 `components` 数组位置强绑定。如果翻译字符串中组件顺序变化，索引必须同步更新。建议在代码审查时重点关注此类键的变更。
