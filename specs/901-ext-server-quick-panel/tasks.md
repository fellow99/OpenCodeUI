# 901-ext-server-quick-panel 实施任务清单

> 模块编号：901-ext-server-quick-panel
> 状态：✅ 已完成
> 最后更新：2026-04-13

---

## 任务概览

| #   | 任务                  | 状态    | 产出文件                                  |
| --- | --------------------- | ------- | ----------------------------------------- |
| T-1 | 创建 ServerQuickPanel | ✅ 完成 | `ServerQuickPanel.tsx` (617 行)           |
| T-2 | 改造 SidePanel 触发器 | ✅ 完成 | `SidePanel.tsx` 修改 (~30 行)             |
| T-3 | 补充 i18n 翻译        | ✅ 完成 | `chat.json` + `common.json` (中英各 4 键) |
| T-4 | TypeScript 类型验证   | ✅ 完成 | `tsc --noEmit` 零错误                     |

---

## T-1：创建 ServerQuickPanel 组件

**状态**：✅ 已完成

**目标**：创建独立的浮动面板组件，以三级树形结构展示服务器 → 工程 → 会话。

**子任务**：

| #      | 子任务                      | 状态    | 说明                                                      |
| ------ | --------------------------- | ------- | --------------------------------------------------------- |
| T-1.1  | 定义内部数据结构            | ✅ 完成 | `SessionInfo`、`ProjectGroup`、`ServerNode` 接口          |
| T-1.2  | 实现 SDK 客户端缓存         | ✅ 完成 | `clientCache` Map + `getServerClient()` 函数              |
| T-1.3  | 实现面板定位计算            | ✅ 完成 | 基于 trigger.getBoundingClientRect() 计算弹出位置         |
| T-1.4  | 实现数据获取逻辑            | ✅ 完成 | `fetchAll()` 并行请求，含降级处理和取消机制               |
| T-1.5  | 实现状态映射函数            | ✅ 完成 | `extractStatusType()` 将 SDK SessionStatus 映射为简化枚举 |
| T-1.6  | 实现面板头部                | ✅ 完成 | 标题、概览信息、关闭按钮                                  |
| T-1.7  | 实现 ServerTreeItem 子组件  | ✅ 完成 | 服务器名称、URL、健康状态、展开/收起                      |
| T-1.8  | 实现 ProjectTreeItem 子组件 | ✅ 完成 | 工程名称、会话计数、忙碌计数、展开/收起                   |
| T-1.9  | 实现 SessionItem 子组件     | ✅ 完成 | 会话标题、状态指示点（四色）、点击选择                    |
| T-1.10 | 实现关闭行为                | ✅ 完成 | 外部点击检测、ESC 键、关闭按钮、退出动画                  |
| T-1.11 | 添加完整注释                | ✅ 完成 | 所有函数、接口、变量均有必要注释                          |

**验收**：

- [x] 面板在触发按钮下方弹出，定位正确
- [x] 三级树形结构可展开/收起
- [x] 数据加载显示 loading/error/empty 状态
- [x] 会话状态指示点颜色正确
- [x] 关闭动画流畅
- [x] TypeScript 编译通过

---

## T-2：改造 SidePanel 触发器

**状态**：✅ 已完成

**目标**：将 OpenCode 标识从 `<a href="/">` 改为 `<button>`，集成 ServerQuickPanel。

**子任务**：

| #     | 子任务                | 状态    | 说明                                                   |
| ----- | --------------------- | ------- | ------------------------------------------------------ |
| T-2.1 | 添加导入              | ✅ 完成 | 导入 `ServerQuickPanel`、`serverStore`、`messageStore` |
| T-2.2 | 添加面板状态          | ✅ 完成 | `serverPanelOpen` state + `serverPanelTriggerRef` ref  |
| T-2.3 | 改造触发按钮          | ✅ 完成 | `<a href="/">` → `<button>`，保留视觉样式              |
| T-2.4 | 实现会话切换处理      | ✅ 完成 | `handleQuickPanelSelectSession` 回调                   |
| T-2.5 | 渲染 ServerQuickPanel | ✅ 完成 | 条件渲染，传入 triggerRef/onClose/onSelectSession      |

**验收**：

- [x] 点击 OpenCode 标识弹出面板（不刷新页面）
- [x] 跨服务器切换时清理消息并导航
- [x] 同服务器切换时直接打开会话
- [x] TypeScript 编译通过

---

## T-3：补充 i18n 翻译

**状态**：✅ 已完成

**目标**：为面板新增的 4 个 i18n 键值补充中英文翻译。

**新增键值**：

| 键值                      | zh-CN              | en                  | 文件          |
| ------------------------- | ------------------ | ------------------- | ------------- |
| `chat.sidebar.servers`    | 服务器             | Servers             | `chat.json`   |
| `chat.sidebar.sessions`   | 会话               | Sessions            | `chat.json`   |
| `chat.sidebar.noProjects` | 暂无项目           | No projects         | `chat.json`   |
| `common.servers`          | {{count}} 个服务器 | {{count}} server(s) | `common.json` |

**验收**：

- [x] 所有 JSON 文件语法正确
- [x] 中英文键值一一对应
- [x] 面板中所有文本正确显示

---

## T-4：TypeScript 类型验证

**状态**：✅ 已完成

**目标**：确保所有新增代码通过 TypeScript 类型检查。

**验证结果**：

- [x] `tsc --noEmit` 零错误
- [x] LSP 诊断无错误
- [x] 无 `as any` / `@ts-ignore` / `@ts-expect-error` 类型压制

---

## 依赖关系

```
T-1 (ServerQuickPanel) ──┐
                          ├──→ T-2 (SidePanel 集成)
T-3 (i18n) ──────────────┘
                                    ↓
                              T-4 (类型验证)
```

- T-1 和 T-3 可并行执行
- T-2 依赖 T-1 和 T-3 完成
- T-4 在所有代码完成后执行

---

## 产出清单

| 文件                                                          | 行数   | 类型 |
| ------------------------------------------------------------- | ------ | ---- |
| `src/features/chat/sidebar/ServerQuickPanel.tsx`              | 617    | 新增 |
| `src/features/chat/sidebar/SidePanel.tsx`                     | ~30    | 修改 |
| `src/locales/zh-CN/chat.json`                                 | 3      | 修改 |
| `src/locales/en/chat.json`                                    | 3      | 修改 |
| `src/locales/zh-CN/common.json`                               | 1      | 修改 |
| `src/locales/en/common.json`                                  | 1      | 修改 |
| `specs/901-ext-server-quick-panel/spec.md`                    | ~440   | 新增 |
| `specs/901-ext-server-quick-panel/plan.md`                    | ~400   | 新增 |
| `specs/901-ext-server-quick-panel/checklists/requirements.md` | ~80    | 新增 |
| `specs/901-ext-server-quick-panel/tasks.md`                   | 本文件 | 新增 |

---

_生成时间: 2026-04-13_
