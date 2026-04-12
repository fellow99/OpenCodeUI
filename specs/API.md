# OpenCodeUI 后端 API 参考

本文档列出 OpenCode 后端所有 REST API 端点，以及前端调用它们的源文件。

> **开发代理**：前端通过 `/api` 代理访问后端（`/api → http://127.0.0.1:4096`），代理会自动剥离 `/api` 前缀。
>
> **SDK 客户端**：前端使用 `@opencode-ai/sdk` 封装 HTTP 调用，SDK 客户端在 `src/api/sdk.ts` 中创建。

---

## Global（全局）

| 方法  | 路径              | 说明                  | 前端源文件          |
| ----- | ----------------- | --------------------- | ------------------- |
| GET   | `/global/health`  | 健康检查              | `src/api/global.ts` |
| GET   | `/global/event`   | 订阅全局事件流（SSE） | `src/api/events.ts` |
| GET   | `/global/config`  | 获取全局配置          | —                   |
| PATCH | `/global/config`  | 更新全局配置          | —                   |
| POST  | `/global/dispose` | 释放全局资源          | `src/api/global.ts` |

## Auth（认证）

| 方法   | 路径                 | 说明         | 前端源文件 |
| ------ | -------------------- | ------------ | ---------- |
| PUT    | `/auth/{providerID}` | 设置认证凭据 | —          |
| DELETE | `/auth/{providerID}` | 删除认证凭据 | —          |

## Project（项目）

| 方法  | 路径                   | 说明                           | 前端源文件 |
| ----- | ---------------------- | ------------------------------ | ---------- |
| GET   | `/project`             | 列出所有项目                   | —          |
| GET   | `/project/current`     | 获取当前项目                   | —          |
| PATCH | `/project/{projectID}` | 更新项目（名称、图标、命令等） | —          |

## PTY（终端会话）

| 方法   | 路径                   | 说明                        | 前端源文件       |
| ------ | ---------------------- | --------------------------- | ---------------- |
| GET    | `/pty`                 | 列出所有 PTY 会话           | `src/api/pty.ts` |
| POST   | `/pty`                 | 创建 PTY 会话               | `src/api/pty.ts` |
| GET    | `/pty/{ptyID}`         | 获取 PTY 会话详情           | `src/api/pty.ts` |
| PUT    | `/pty/{ptyID}`         | 更新 PTY 会话（标题、尺寸） | `src/api/pty.ts` |
| DELETE | `/pty/{ptyID}`         | 删除 PTY 会话               | `src/api/pty.ts` |
| GET    | `/pty/{ptyID}/connect` | 连接 PTY 会话（WebSocket）  | `src/api/pty.ts` |

## Config（配置）

| 方法  | 路径                | 说明                     | 前端源文件          |
| ----- | ------------------- | ------------------------ | ------------------- |
| GET   | `/config`           | 获取当前配置             | `src/api/config.ts` |
| PATCH | `/config`           | 更新配置                 | `src/api/config.ts` |
| GET   | `/config/providers` | 列出 AI 提供者及默认模型 | `src/api/config.ts` |

## Experimental / Tool（实验性 · 工具）

| 方法 | 路径                     | 说明                                | 前端源文件        |
| ---- | ------------------------ | ----------------------------------- | ----------------- |
| GET  | `/experimental/tool/ids` | 获取所有工具 ID 列表                | `src/api/tool.ts` |
| GET  | `/experimental/tool`     | 获取工具列表（含 JSON Schema 参数） | `src/api/tool.ts` |

## Experimental / Worktree（实验性 · Git Worktree）

| 方法   | 路径                           | 说明                     | 前端源文件            |
| ------ | ------------------------------ | ------------------------ | --------------------- |
| POST   | `/experimental/worktree`       | 创建 Git Worktree        | `src/api/worktree.ts` |
| GET    | `/experimental/worktree`       | 列出所有 Worktree        | `src/api/worktree.ts` |
| DELETE | `/experimental/worktree`       | 删除 Worktree 及其分支   | `src/api/worktree.ts` |
| POST   | `/experimental/worktree/reset` | 重置 Worktree 到默认分支 | `src/api/worktree.ts` |

## Experimental / Resource（实验性 · MCP 资源）

| 方法 | 路径                     | 说明                        | 前端源文件 |
| ---- | ------------------------ | --------------------------- | ---------- |
| GET  | `/experimental/resource` | 获取已连接 MCP 服务器的资源 | —          |

## Session（会话）

| 方法   | 路径                                                     | 说明                                      | 前端源文件           |
| ------ | -------------------------------------------------------- | ----------------------------------------- | -------------------- |
| GET    | `/session`                                               | 列出所有会话                              | `src/api/session.ts` |
| POST   | `/session`                                               | 创建新会话                                | `src/api/session.ts` |
| GET    | `/session/status`                                        | 获取所有会话状态（active/idle/completed） | `src/api/session.ts` |
| GET    | `/session/{sessionID}`                                   | 获取会话详情                              | `src/api/session.ts` |
| DELETE | `/session/{sessionID}`                                   | 删除会话及所有数据                        | `src/api/session.ts` |
| PATCH  | `/session/{sessionID}`                                   | 更新会话（标题、归档时间）                | `src/api/session.ts` |
| GET    | `/session/{sessionID}/children`                          | 获取子会话（Fork 产生的）                 | `src/api/session.ts` |
| GET    | `/session/{sessionID}/todo`                              | 获取会话待办列表                          | `src/api/session.ts` |
| POST   | `/session/{sessionID}/init`                              | 初始化会话（生成 AGENTS.md）              | —                    |
| POST   | `/session/{sessionID}/fork`                              | 在指定消息处分叉会话                      | `src/api/session.ts` |
| POST   | `/session/{sessionID}/abort`                             | 中止会话（停止 AI 处理）                  | `src/api/session.ts` |
| POST   | `/session/{sessionID}/share`                             | 创建会话分享链接                          | `src/api/session.ts` |
| DELETE | `/session/{sessionID}/share`                             | 取消分享会话                              | `src/api/session.ts` |
| GET    | `/session/{sessionID}/diff`                              | 获取指定消息产生的文件变更 diff           | `src/api/session.ts` |
| POST   | `/session/{sessionID}/summarize`                         | 使用 AI 压缩总结会话                      | `src/api/session.ts` |
| GET    | `/session/{sessionID}/message`                           | 获取会话消息列表                          | `src/api/message.ts` |
| POST   | `/session/{sessionID}/message`                           | 发送消息到会话（同步，流式响应）          | `src/api/message.ts` |
| GET    | `/session/{sessionID}/message/{messageID}`               | 获取单条消息                              | —                    |
| DELETE | `/session/{sessionID}/message/{messageID}/part/{partID}` | 删除消息中的部分内容                      | —                    |
| PATCH  | `/session/{sessionID}/message/{messageID}/part/{partID}` | 更新消息中的部分内容                      | —                    |
| POST   | `/session/{sessionID}/prompt_async`                      | 异步发送消息（立即返回，SSE 推送响应）    | `src/api/message.ts` |
| POST   | `/session/{sessionID}/command`                           | 向会话发送斜杠命令                        | `src/api/command.ts` |
| POST   | `/session/{sessionID}/shell`                             | 在会话上下文中执行 Shell 命令             | —                    |
| POST   | `/session/{sessionID}/revert`                            | 回退指定消息                              | `src/api/session.ts` |
| POST   | `/session/{sessionID}/unrevert`                          | 恢复所有已回退的消息                      | `src/api/session.ts` |

## Permission（权限）

| 方法 | 路径                                              | 说明                               | 前端源文件              |
| ---- | ------------------------------------------------- | ---------------------------------- | ----------------------- |
| GET  | `/permission`                                     | 列出所有待处理权限请求             | `src/api/permission.ts` |
| POST | `/permission/{requestID}/reply`                   | 回复权限请求（once/always/reject） | `src/api/permission.ts` |
| POST | `/session/{sessionID}/permissions/{permissionID}` | 响应权限请求（已弃用）             | —                       |

## Question（问题）

| 方法 | 路径                           | 说明                   | 前端源文件              |
| ---- | ------------------------------ | ---------------------- | ----------------------- |
| GET  | `/question`                    | 列出所有待处理问题请求 | `src/api/permission.ts` |
| POST | `/question/{requestID}/reply`  | 回复问题请求           | `src/api/permission.ts` |
| POST | `/question/{requestID}/reject` | 拒绝问题请求           | `src/api/permission.ts` |

## Provider（AI 提供者）

| 方法 | 路径                                     | 说明                          | 前端源文件 |
| ---- | ---------------------------------------- | ----------------------------- | ---------- |
| GET  | `/provider`                              | 列出所有 AI 提供者及模型      | —          |
| GET  | `/provider/auth`                         | 获取所有提供者的认证方法      | —          |
| POST | `/provider/{providerID}/oauth/authorize` | 发起 OAuth 授权，获取授权 URL | —          |
| POST | `/provider/{providerID}/oauth/callback`  | 处理 OAuth 回调               | —          |

## Find（搜索）

| 方法 | 路径           | 说明                         | 前端源文件        |
| ---- | -------------- | ---------------------------- | ----------------- |
| GET  | `/find`        | 全文文本搜索（基于 ripgrep） | —                 |
| GET  | `/find/file`   | 搜索文件或目录               | `src/api/file.ts` |
| GET  | `/find/symbol` | 搜索代码符号（基于 LSP）     | `src/api/file.ts` |

## File（文件）

| 方法 | 路径            | 说明                    | 前端源文件        |
| ---- | --------------- | ----------------------- | ----------------- |
| GET  | `/file`         | 列出目录内容            | `src/api/file.ts` |
| GET  | `/file/content` | 读取文件内容            | `src/api/file.ts` |
| GET  | `/file/status`  | 获取所有文件的 Git 状态 | `src/api/file.ts` |

## MCP（Model Context Protocol）

| 方法   | 路径                            | 说明                                  | 前端源文件       |
| ------ | ------------------------------- | ------------------------------------- | ---------------- |
| GET    | `/mcp`                          | 获取所有 MCP 服务器状态               | `src/api/mcp.ts` |
| POST   | `/mcp`                          | 添加 MCP 服务器                       | `src/api/mcp.ts` |
| POST   | `/mcp/{name}/auth`              | 启动 MCP OAuth 认证                   | `src/api/mcp.ts` |
| DELETE | `/mcp/{name}/auth`              | 删除 MCP OAuth 凭据                   | `src/api/mcp.ts` |
| POST   | `/mcp/{name}/auth/callback`     | 完成 MCP OAuth（使用授权码）          | `src/api/mcp.ts` |
| POST   | `/mcp/{name}/auth/authenticate` | 启动完整 MCP OAuth 流程（打开浏览器） | `src/api/mcp.ts` |
| POST   | `/mcp/{name}/connect`           | 连接 MCP 服务器                       | `src/api/mcp.ts` |
| POST   | `/mcp/{name}/disconnect`        | 断开 MCP 服务器                       | `src/api/mcp.ts` |

## TUI（终端用户界面）

| 方法 | 路径                    | 说明                            | 前端源文件 |
| ---- | ----------------------- | ------------------------------- | ---------- |
| POST | `/tui/append-prompt`    | 追加 TUI 提示词                 | —          |
| POST | `/tui/open-help`        | 打开帮助对话框                  | —          |
| POST | `/tui/open-sessions`    | 打开会话对话框                  | —          |
| POST | `/tui/open-themes`      | 打开主题对话框                  | —          |
| POST | `/tui/open-models`      | 打开模型对话框                  | —          |
| POST | `/tui/submit-prompt`    | 提交 TUI 提示词                 | —          |
| POST | `/tui/clear-prompt`     | 清除 TUI 提示词                 | —          |
| POST | `/tui/execute-command`  | 执行 TUI 命令（如 agent_cycle） | —          |
| POST | `/tui/show-toast`       | 显示 TUI 通知                   | —          |
| POST | `/tui/publish`          | 发布 TUI 事件                   | —          |
| POST | `/tui/select-session`   | 切换到指定会话                  | —          |
| GET  | `/tui/control/next`     | 获取队列中下一个 TUI 请求       | —          |
| POST | `/tui/control/response` | 提交 TUI 请求的响应             | —          |

## Instance（实例）

| 方法 | 路径                | 说明             | 前端源文件          |
| ---- | ------------------- | ---------------- | ------------------- |
| POST | `/instance/dispose` | 释放当前实例资源 | `src/api/global.ts` |

## Path（路径）

| 方法 | 路径    | 说明                       | 前端源文件 |
| ---- | ------- | -------------------------- | ---------- |
| GET  | `/path` | 获取当前工作目录及路径信息 | —          |

## VCS（版本控制）

| 方法 | 路径   | 说明                         | 前端源文件       |
| ---- | ------ | ---------------------------- | ---------------- |
| GET  | `/vcs` | 获取 VCS 信息（如 Git 分支） | `src/api/vcs.ts` |

## Command（命令）

| 方法 | 路径       | 说明             | 前端源文件           |
| ---- | ---------- | ---------------- | -------------------- |
| GET  | `/command` | 列出所有可用命令 | `src/api/command.ts` |

## Log（日志）

| 方法 | 路径   | 说明               | 前端源文件 |
| ---- | ------ | ------------------ | ---------- |
| POST | `/log` | 写入服务器日志条目 | —          |

## Agent（智能体）

| 方法 | 路径     | 说明               | 前端源文件         |
| ---- | -------- | ------------------ | ------------------ |
| GET  | `/agent` | 列出所有可用 Agent | `src/api/agent.ts` |

## Skill（技能）

| 方法 | 路径     | 说明                | 前端源文件         |
| ---- | -------- | ------------------- | ------------------ |
| GET  | `/skill` | 列出所有可用 Skills | `src/api/skill.ts` |

## LSP（Language Server Protocol）

| 方法 | 路径   | 说明                | 前端源文件       |
| ---- | ------ | ------------------- | ---------------- |
| GET  | `/lsp` | 获取 LSP 服务器状态 | `src/api/lsp.ts` |

## Formatter（格式化器）

| 方法 | 路径         | 说明                 | 前端源文件       |
| ---- | ------------ | -------------------- | ---------------- |
| GET  | `/formatter` | 获取代码格式化器状态 | `src/api/lsp.ts` |

## Event（事件）

| 方法 | 路径     | 说明              | 前端源文件 |
| ---- | -------- | ----------------- | ---------- |
| GET  | `/event` | 订阅事件流（SSE） | —          |

---

## 前端 API 模块索引

| 模块文件                | 负责的 API 域                                                         |
| ----------------------- | --------------------------------------------------------------------- |
| `src/api/sdk.ts`        | SDK 客户端创建与导出                                                  |
| `src/api/http.ts`       | 原始 HTTP 工具（fetchWithAuth、getApiBaseUrl 等）                     |
| `src/api/client.ts`     | 统一导出所有子模块                                                    |
| `src/api/global.ts`     | Global、Instance                                                      |
| `src/api/config.ts`     | Config、Config Providers                                              |
| `src/api/session.ts`    | Session CRUD 及操作（fork、abort、share、diff、summarize、revert 等） |
| `src/api/message.ts`    | Session Message（获取消息、发送消息、异步发送）                       |
| `src/api/permission.ts` | Permission、Question                                                  |
| `src/api/pty.ts`        | PTY 会话管理                                                          |
| `src/api/file.ts`       | File、Find（文件/符号搜索）                                           |
| `src/api/mcp.ts`        | MCP 服务器管理与 OAuth                                                |
| `src/api/tool.ts`       | Experimental Tool                                                     |
| `src/api/worktree.ts`   | Experimental Worktree                                                 |
| `src/api/command.ts`    | Command 列表与执行                                                    |
| `src/api/agent.ts`      | Agent                                                                 |
| `src/api/skill.ts`      | Skill                                                                 |
| `src/api/lsp.ts`        | LSP、Formatter                                                        |
| `src/api/vcs.ts`        | VCS                                                                   |
| `src/api/events.ts`     | 全局事件 SSE 订阅（单例模式）                                         |
| `src/api/todo.ts`       | Todo 数据规范化（辅助模块）                                           |
| `src/api/types.ts`      | 共享类型定义                                                          |
