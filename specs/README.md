# OpenCodeUI — 规范文档索引

> 本目录包含 OpenCodeUI 项目的完整规范文档体系，涵盖项目级架构文档和 16 个功能模块的详细规范。

## 项目简介

OpenCodeUI 是 [OpenCode](https://github.com/anomalyco/opencode) 的第三方 Web 前端界面，提供完整的 Chat UI、内置终端、文件差异查看、主题系统等功能。技术栈为 React 19 + TypeScript + Vite + Tailwind CSS v4，同时支持 Docker 容器化部署和 Tauri 桌面应用。

---

## 项目级文档

| 文档                                             | 说明                                                     |
| ------------------------------------------------ | -------------------------------------------------------- |
| [STRUCTURE.md](./STRUCTURE.md)                   | 项目目录结构、页面清单、路由清单                         |
| [API.md](./API.md)                               | OpenCode REST API 完整清单（路径、方法、用途、入口文件） |
| [TECH.md](./TECH.md)                             | 技术选型与依赖分析                                       |
| [ARCHITECTURE.md](./ARCHITECTURE.md)             | 整体架构设计（分层、数据流、部署拓扑）                   |
| [constitution.md](./constitution.md)             | 项目宪法 — 核心原则与约束                                |
| [overall-spec.md](./overall-spec.md)             | 整体功能规格（WHAT & WHY）                               |
| [overall-plan.md](./overall-plan.md)             | 整体技术方案（HOW）                                      |
| [overall-data-model.md](./overall-data-model.md) | 数据模型 — 实体、关系、状态机                            |
| [overall-api.md](./overall-api.md)               | 对外接口模型 — API 契约与集成点                          |
| [SPECS_CHECKLIST.md](./SPECS_CHECKLIST.md)       | 文档完成情况清单（43/43 = 100%）                         |

---

## 功能模块规范文档

每个模块目录下包含：

- `spec.md` — 功能规范（WHAT & WHY），遵循 speckit-baseline 规范格式
- `plan.md` — 实施方案（HOW），包含技术上下文、宪法合规检查、数据模型、接口契约、实现策略

| #   | 模块           | 规范                                     | 方案                                     | 说明                                             |
| --- | -------------- | ---------------------------------------- | ---------------------------------------- | ------------------------------------------------ |
| 001 | API 通信层     | [spec](./001-api-layer/spec.md)          | [plan](./001-api-layer/plan.md)          | HTTP 客户端、SDK 封装、请求/响应管道、SSE 事件流 |
| 002 | 聊天功能       | [spec](./002-chat-feature/spec.md)       | [plan](./002-chat-feature/plan.md)       | 聊天界面、消息输入、权限对话框、侧边栏           |
| 003 | 消息渲染       | [spec](./003-message-rendering/spec.md)  | [plan](./003-message-rendering/plan.md)  | Markdown 渲染、代码高亮、工具调用展示、附件预览  |
| 004 | 会话管理       | [spec](./004-session-management/spec.md) | [plan](./004-session-management/plan.md) | 会话列表、项目选择、会话生命周期                 |
| 005 | 设置面板       | [spec](./005-settings-panel/spec.md)     | [plan](./005-settings-panel/plan.md)     | 设置对话框、快捷键配置、主题切换                 |
| 006 | @提及系统      | [spec](./006-mention-system/spec.md)     | [plan](./006-mention-system/plan.md)     | 文件/符号提及、自动补全、上下文注入              |
| 007 | 斜杠命令       | [spec](./007-slash-command/spec.md)      | [plan](./007-slash-command/plan.md)      | 命令面板、命令注册与执行                         |
| 008 | 终端系统       | [spec](./008-terminal-system/spec.md)    | [plan](./008-terminal-system/plan.md)    | Web 终端、PTY 会话、WebGL 渲染                   |
| 009 | 主题系统       | [spec](./009-theme-system/spec.md)       | [plan](./009-theme-system/plan.md)       | 内置主题预设、明暗模式、自定义 CSS               |
| 010 | 状态管理       | [spec](./010-state-management/spec.md)   | [plan](./010-state-management/plan.md)   | 自定义 Store 体系、响应式状态、持久化            |
| 011 | 文件差异查看器 | [spec](./011-file-diff-viewer/spec.md)   | [plan](./011-file-diff-viewer/plan.md)   | 文件浏览、多文件 Diff 对比、语法高亮             |
| 012 | 面板布局       | [spec](./012-pane-layout/spec.md)        | [plan](./012-pane-layout/plan.md)        | 分屏容器、可拖拽面板、响应式布局                 |
| 013 | 国际化         | [spec](./013-i18n-localization/spec.md)  | [plan](./013-i18n-localization/plan.md)  | i18next 集成、中英双语支持                       |
| 014 | 桌面应用       | [spec](./014-tauri-desktop/spec.md)      | [plan](./014-tauri-desktop/plan.md)      | Tauri 2 原生客户端、跨平台构建                   |
| 015 | Docker 部署    | [spec](./015-docker-deployment/spec.md)  | [plan](./015-docker-deployment/plan.md)  | Gateway / Frontend / Backend 容器编排            |
| 016 | 路由服务       | [spec](./016-router-service/spec.md)     | [plan](./016-router-service/plan.md)     | 动态端口发现、预览链接生成                       |

---

## 文档约定

- **规范格式**: 遵循 speckit-baseline 规范，focus on WHAT and WHY, technology-agnostic
- **编码规则**: 功能模块从 `001` 开始按顺序编码
- **文档语言**: 中文为主，技术术语保留英文原文
- **维护**: 文档变更需同步更新 [SPECS_CHECKLIST.md](./SPECS_CHECKLIST.md)

---

_生成时间: 2026-04-12_
