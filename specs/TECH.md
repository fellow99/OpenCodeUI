# OpenCodeUI 技术栈文档

> 本文档记录 OpenCodeUI 项目使用的所有技术、框架、库和工具。版本信息以 `package.json` 和 `Cargo.toml` 为准。

## 目录

- [1. 核心框架](#1-核心框架)
- [2. 样式系统](#2-样式系统)
- [3. 代码渲染与 Markdown](#3-代码渲染与-markdown)
- [4. 终端模拟](#4-终端模拟)
- [5. 状态管理](#5-状态管理)
- [6. 国际化 (i18n)](#6-国际化-i18n)
- [7. SDK / API 通信](#7-sdk--api-通信)
- [8. 桌面应用 (Tauri 2)](#8-桌面应用-tauri-2)
- [9. 动态端口路由 (Router)](#9-动态端口路由-router)
- [10. 部署与运维](#10-部署与运维)
- [11. 动画与交互](#11-动画与交互)
- [12. 工具库](#12-工具库)
- [13. 开发工具链](#13-开发工具链)
- [14. 测试](#14-测试)
- [15. CI/CD](#15-cicd)

---

## 1. 核心框架

| 技术                 | 版本    | 用途                                    |
| -------------------- | ------- | --------------------------------------- |
| React                | ^19.2.0 | UI 组件框架                             |
| React DOM            | ^19.2.0 | React 的 DOM 渲染层                     |
| TypeScript           | ~5.9.3  | 类型系统                                |
| Vite                 | ^8.0.3  | 构建工具与开发服务器                    |
| @vitejs/plugin-react | ^6.0.1  | Vite 的 React 插件（支持 Fast Refresh） |
| @tailwindcss/vite    | ^4.2.2  | Tailwind CSS v4 的 Vite 插件            |

**TypeScript 配置要点：**

- 采用 Project References 模式，拆分为 `tsconfig.app.json`（应用代码）和 `tsconfig.node.json`（Node 端配置）
- 目标语言级别：`ES2022`（应用）/ `ES2023`（Node）
- 模块系统：`ESNext`，模块解析：`bundler`
- JSX 模式：`react-jsx`
- 严格模式开启，包含 `noUnusedLocals`、`noUnusedParameters`、`erasableSyntaxOnly` 等 lint 规则

**Vite 构建配置要点：**

- 通过 `manualChunks` 将第三方库拆分为独立 chunk：
  - `vendor-terminal`：xterm 相关
  - `vendor-shiki`：Shiki 代码高亮
  - `vendor-markdown`：Streamdown / Remend Markdown 渲染
  - `vendor-tauri`：Tauri API
- 开发服务器端口严格模式（`strictPort: true`），`/api` 代理至 `http://127.0.0.1:4096`
- 注入 `__APP_VERSION__` 全局常量（取自 `package.json`）

---

## 2. 样式系统

| 技术         | 版本   | 用途            |
| ------------ | ------ | --------------- |
| Tailwind CSS | ^4.2.2 | 原子化 CSS 框架 |
| clsx         | ^2.1.1 | 条件类名拼接    |

Tailwind CSS v4 通过 `@tailwindcss/vite` 插件集成到 Vite 构建流程中，无需单独的 `tailwind.config.js` 配置文件。`clsx` 用于动态组合 CSS 类名。

---

## 3. 代码渲染与 Markdown

| 技术             | 版本   | 用途                                  |
| ---------------- | ------ | ------------------------------------- |
| Shiki            | ^4.0.2 | 语法高亮引擎（基于 VS Code 语法主题） |
| Streamdown       | ^2.5.0 | Markdown 渲染组件库                   |
| @streamdown/math | ^1.0.2 | Streamdown 的数学公式渲染扩展         |

Shiki 提供与 VS Code 一致的代码高亮效果。Streamdown 负责 Markdown 内容的解析与渲染，`@streamdown/math` 扩展支持数学公式（LaTeX）渲染。

---

## 4. 终端模拟

| 技术                   | 版本    | 用途                 |
| ---------------------- | ------- | -------------------- |
| @xterm/xterm           | ^6.0.0  | 终端模拟器核心库     |
| @xterm/addon-fit       | ^0.11.0 | 终端自适应容器尺寸   |
| @xterm/addon-web-links | ^0.12.0 | 终端内链接识别与点击 |

xterm.js 提供完整的终端模拟能力，包括 PTY 交互、ANSI 转义序列解析等。`addon-fit` 使终端自动适配容器大小，`addon-web-links` 自动检测并渲染终端输出中的 URL。

---

## 5. 状态管理

项目未使用 Redux、Zustand 等第三方状态管理库，而是采用**自定义 Store 模式**，基于 pub-sub 发布订阅 + `React.useSyncExternalStore` 实现。

| Store 文件            | 职责                                       |
| --------------------- | ------------------------------------------ |
| `messageStore`        | 消息与会话状态（核心 store，含 undo/redo） |
| `activeSessionStore`  | 活跃会话跟踪                               |
| `childSessionStore`   | 子会话（分支会话）管理                     |
| `layoutStore`         | 全局布局状态                               |
| `paneLayoutStore`     | 窗格布局（支持嵌套分割）                   |
| `paneControllerStore` | 窗格控制器状态                             |
| `themeStore`          | 主题与明暗模式                             |
| `keybindingStore`     | 快捷键配置与解析                           |
| `serverStore`         | 服务器连接配置与健康检查                   |
| `autoApproveStore`    | 自动审批规则                               |
| `changeScopeStore`    | 变更范围模式                               |
| `todoStore`           | Todo 列表状态                              |
| `notificationStore`   | 通知管理                                   |
| `serviceStore`        | 服务状态                                   |
| `soundStore`          | 音效设置                                   |
| `followupQueueStore`  | 追问队列                                   |

**模式特点：**

- 每个 store 是独立模块，维护自身状态与订阅者列表
- 通过 `useSyncExternalStore` 桥接到 React 组件树
- 提供 selector 钩子实现细粒度订阅，避免不必要的重渲染

---

## 6. 国际化 (i18n)

| 技术                             | 版本    | 用途               |
| -------------------------------- | ------- | ------------------ |
| i18next                          | ^26.0.1 | 国际化核心库       |
| react-i18next                    | ^17.0.1 | React 绑定         |
| i18next-browser-languagedetector | ^8.2.1  | 浏览器语言自动检测 |

**配置要点：**

- 通过 Vite `import.meta.glob` 在构建时预加载所有翻译文件（`src/locales/{lang}/{namespace}.json`）
- 支持语言：`en`（英语）、`zh-CN`（简体中文）
- 语言检测顺序：`localStorage` → `navigator`（浏览器语言）
- 检测结果缓存至 `localStorage`（键名 `i18nextLng`）
- 默认命名空间：`common`
- 翻译文件按功能域拆分：`chat`、`commands`、`common`、`components`、`message`、`settings`

---

## 7. SDK / API 通信

| 技术             | 版本   | 用途                                         |
| ---------------- | ------ | -------------------------------------------- |
| @opencode-ai/sdk | ^1.4.1 | OpenCode 官方 SDK，封装 REST API 与 SSE 通信 |

前端通过 `@opencode-ai/sdk` 与 OpenCode 后端通信，支持：

- REST API 调用（会话管理、消息发送、文件操作等）
- SSE（Server-Sent Events）流式接收 AI 回复
- 开发环境通过 Vite proxy 将 `/api` 前缀代理至后端 `:4096` 端口
- Tauri 桌面模式下通过 `tauri-plugin-http` 直接请求后端，不走 Vite proxy

---

## 8. 桌面应用 (Tauri 2)

### 前端侧

| 技术                            | 版本    | 用途                          |
| ------------------------------- | ------- | ----------------------------- |
| @tauri-apps/api                 | ^2.10.1 | Tauri 前端 API                |
| @tauri-apps/plugin-dialog       | ^2.6.0  | 系统对话框（文件选择等）      |
| @tauri-apps/plugin-fs           | ^2.4.5  | 文件系统操作                  |
| @tauri-apps/plugin-http         | ^2.5.7  | HTTP 请求（替代浏览器 fetch） |
| @tauri-apps/plugin-notification | ^2.3.3  | 系统通知                      |
| @tauri-apps/plugin-opener       | ^2.5.3  | 打开外部链接/文件             |
| @tauri-apps/cli                 | ^2.10.0 | Tauri CLI 工具（开发依赖）    |

### Rust 后端 (src-tauri/Cargo.toml)

| 依赖                         | 版本  | 用途                              |
| ---------------------------- | ----- | --------------------------------- |
| tauri                        | 2     | Tauri 核心框架                    |
| tauri-plugin-dialog          | 2     | 对话框插件                        |
| tauri-plugin-fs              | 2     | 文件系统插件                      |
| tauri-plugin-http            | 2     | HTTP 插件                         |
| tauri-plugin-log             | 2     | 日志插件                          |
| tauri-plugin-notification    | 2     | 通知插件                          |
| tauri-plugin-opener          | 2     | 打开器插件                        |
| tauri-plugin-single-instance | 2     | 单实例运行插件                    |
| tauri-build                  | 2     | 构建时工具                        |
| tokio                        | 1     | 异步运行时                        |
| reqwest                      | 0.12  | HTTP 客户端（rustls-tls, stream） |
| serde                        | 1     | 序列化/反序列化                   |
| serde_json                   | 1     | JSON 处理                         |
| futures-util                 | 0.3   | 异步工具                          |
| papaya                       | 0.2.3 | 并发哈希表                        |
| rapidhash                    | 4.4.1 | 高性能哈希                        |
| log                          | 0.4   | 日志门面                          |

**Rust 版本要求：** 1.85.0+，Edition 2021

**Release 优化：** `strip = true`、`lto = "thin"`、`panic = "abort"`、`codegen-units = 1`

---

## 9. 动态端口路由 (Router)

独立的 Rust 服务（`src-router/`），负责在 Docker Gateway 模式下动态扫描和路由容器内开发服务的端口。

| 依赖       | 版本    | 用途                         |
| ---------- | ------- | ---------------------------- |
| axum       | 0.8.8   | Web 框架（支持 HTTP/2）      |
| tokio      | 1.50.0  | 异步运行时                   |
| serde      | 1.0.228 | 序列化/反序列化              |
| serde_json | 1.0.149 | JSON 处理                    |
| rand       | 0.9.2   | 随机数生成（用于路由 token） |
| base64     | 0.22.1  | Base64 编解码                |
| log        | 0.4.29  | 日志门面                     |
| env_logger | 0.11.8  | 环境变量驱动的日志           |

**Rust Edition：** 2024

**Release 优化：** 与 Tauri 相同（strip、thin LTO、panic abort）

---

## 10. 部署与运维

### 容器化

| 技术           | 版本          | 用途                 |
| -------------- | ------------- | -------------------- |
| Docker         | -             | 容器化部署           |
| Docker Compose | -             | 多服务编排           |
| Caddy          | 2-alpine      | Web 服务器与反向代理 |
| Node.js        | 22 (Alpine)   | 前端构建环境         |
| Rust           | 1.94 (Alpine) | Router 构建环境      |

### 服务架构

| 服务            | 端口         | 说明                   |
| --------------- | ------------ | ---------------------- |
| Gateway (Caddy) | 6658         | 统一入口，反代所有请求 |
| Gateway (Caddy) | 6659         | 开发服务预览专用       |
| Frontend        | 3000（内部） | 静态前端（Caddy 托管） |
| Backend         | 4096（内部） | OpenCode API           |
| Router          | 7070（内部） | 动态端口路由           |

### Docker Compose 文件

| 文件                            | 用途                                     |
| ------------------------------- | ---------------------------------------- |
| `docker-compose.yml`            | 完整部署（Gateway + Frontend + Backend） |
| `docker-compose.standalone.yml` | 纯前端部署（连接已有后端）               |
| `docker-compose.build.yml`      | 从源码构建镜像                           |

### Dockerfile

| 文件                         | 用途                                      |
| ---------------------------- | ----------------------------------------- |
| `docker/Dockerfile.frontend` | 前端构建 + Caddy 运行时（多阶段构建）     |
| `docker/Dockerfile.gateway`  | Router 构建 + Caddy Gateway（多阶段构建） |
| `docker/Dockerfile.backend`  | OpenCode 后端容器                         |

### 反向代理

支持 Nginx 和 Caddy 两种反向代理配置。SSE 流式响应要求：

- Nginx：`proxy_buffering off`
- Caddy：`flush_interval -1`

---

## 11. 动画与交互

| 技术         | 版本     | 用途              |
| ------------ | -------- | ----------------- |
| motion       | ^12.38.0 | UI 动画与过渡效果 |
| lucide-react | ^1.7.0   | 图标库            |

`motion`（原 Framer Motion）提供声明式动画 API，用于界面过渡、展开/折叠动画等。`lucide-react` 提供一致的 SVG 图标组件。

---

## 12. 工具库

| 技术        | 版本   | 用途              |
| ----------- | ------ | ----------------- |
| diff        | ^8.0.4 | 文本差异计算      |
| @types/diff | ^7.0.2 | diff 库的类型声明 |

`diff` 库用于文件变更对比、多文件 diff 视图等功能。

---

## 13. 开发工具链

| 技术                        | 版本    | 用途                                   |
| --------------------------- | ------- | -------------------------------------- |
| ESLint                      | ^9.39.4 | 代码检查                               |
| @eslint/js                  | ^9.39.4 | ESLint 官方 JS 规则                    |
| typescript-eslint           | ^8.57.2 | TypeScript ESLint 集成                 |
| eslint-plugin-react-hooks   | ^7.0.1  | React Hooks 规则检查                   |
| eslint-plugin-react-refresh | ^0.5.2  | React Fast Refresh 兼容检查            |
| eslint-config-prettier      | ^10.1.8 | 禁用与 Prettier 冲突的 ESLint 规则     |
| Prettier                    | ^3.6.2  | 代码格式化                             |
| globals                     | ^17.4.0 | 全局变量定义（ESLint 使用）            |
| material-icon-theme         | ^5.32.0 | Material Design 文件图标（构建时复制） |

**校验脚本（`npm run validate`）：** 依次执行 TypeScript 类型检查 → ESLint → 单元测试 → 生产构建。

---

## 14. 测试

| 技术                      | 版本    | 用途                  |
| ------------------------- | ------- | --------------------- |
| Vitest                    | ^4.1.2  | 单元测试框架          |
| jsdom                     | ^29.0.1 | DOM 环境模拟          |
| @testing-library/react    | ^16.3.0 | React 组件测试工具    |
| @testing-library/jest-dom | ^6.9.1  | 额外的 DOM 断言匹配器 |

测试命令：`npm run test`（watch 模式）/ `npm run test:run`（单次运行）。

---

## 15. CI/CD

| Workflow 文件                           | 用途                                                       |
| --------------------------------------- | ---------------------------------------------------------- |
| `.github/workflows/build.yml`           | PR 与 main 分支的构建验证（类型检查 + lint + 测试 + 构建） |
| `.github/workflows/deploy.yml`          | GitHub Pages 部署前端                                      |
| `.github/workflows/docker-frontend.yml` | 前端 Docker 镜像构建与推送                                 |
| `.github/workflows/docker-backend.yml`  | 后端 Docker 镜像构建与推送                                 |
| `.github/workflows/docker-gateway.yml`  | Gateway Docker 镜像构建与推送                              |
| `.github/workflows/release.yml`         | 版本发布（含 Tauri 桌面应用构建）                          |

---

## 完整依赖清单

### 生产依赖 (dependencies)

```
@opencode-ai/sdk          ^1.4.1
@streamdown/math          ^1.0.2
@tauri-apps/api           ^2.10.1
@tauri-apps/plugin-dialog ^2.6.0
@tauri-apps/plugin-fs     ^2.4.5
@tauri-apps/plugin-http   ^2.5.7
@tauri-apps/plugin-notification ^2.3.3
@tauri-apps/plugin-opener ^2.5.3
@xterm/addon-fit          ^0.11.0
@xterm/addon-web-links    ^0.12.0
@xterm/xterm              ^6.0.0
clsx                      ^2.1.1
diff                      ^8.0.4
i18next                   ^26.0.1
i18next-browser-languagedetector ^8.2.1
lucide-react              ^1.7.0
motion                    ^12.38.0
react                     ^19.2.0
react-dom                 ^19.2.0
react-i18next             ^17.0.1
shiki                     ^4.0.2
streamdown                ^2.5.0
```

### 开发依赖 (devDependencies)

```
@eslint/js                ^9.39.4
@tailwindcss/vite         ^4.2.2
@tauri-apps/cli           ^2.10.0
@testing-library/jest-dom ^6.9.1
@testing-library/react    ^16.3.0
@types/diff               ^7.0.2
@types/node               ^25.5.0
@types/react              ^19.2.5
@types/react-dom          ^19.2.3
@vitejs/plugin-react      ^6.0.1
eslint                    ^9.39.4
eslint-config-prettier    ^10.1.8
eslint-plugin-react-hooks ^7.0.1
eslint-plugin-react-refresh ^0.5.2
globals                   ^17.4.0
jsdom                     ^29.0.1
material-icon-theme       ^5.32.0
prettier                  ^3.6.2
tailwindcss               ^4.2.2
typescript                ~5.9.3
typescript-eslint         ^8.57.2
vite                      ^8.0.3
vitest                    ^4.1.2
```

### Rust 依赖 (src-tauri/Cargo.toml)

```
futures-util              0.3
log                       0.4
papaya                    0.2.3
rapidhash                 4.4.1
reqwest                   0.12 (rustls-tls, stream)
serde                     1 (derive)
serde_json                1
tauri                     2 (devtools)
tauri-plugin-dialog       2
tauri-plugin-fs           2
tauri-plugin-http         2
tauri-plugin-log          2
tauri-plugin-notification 2
tauri-plugin-opener       2
tauri-plugin-single-instance 2
tokio                     1 (full)
```

### Router 依赖 (src-router/Cargo.toml)

```
axum                      0.8.8 (http2)
base64                    0.22.1
env_logger                0.11.8
log                       0.4.29 (release_max_level_info)
rand                      0.9.2
serde                     1.0.228 (derive)
serde_json                1.0.149
tokio                     1.50.0 (fs, process, macros, net, parking_lot, rt-multi-thread, signal, time)
```
