# OpenCodeUI 数据模型参考

> 本文档描述 OpenCodeUI 的核心数据模型，所有实体定义来源于 OpenCode 后端 OpenAPI 规范（`openapi_doc.json`），前端通过 `src/types/` 和 `src/api/types.ts` 镜像这些模型。

---

## 目录

1. [核心实体](#1-核心实体)
   - [Session（会话）](#session会话)
   - [Message（消息）](#message消息)
   - [Part（内容片段）](#part内容片段)
   - [Todo（待办事项）](#todo待办事项)
   - [PTY（伪终端）](#pty伪终端)
   - [Project（项目）](#project项目)
   - [FileDiff（文件差异）](#filediff文件差异)
2. [配置实体](#2-配置实体)
   - [Config（全局配置）](#config全局配置)
   - [AgentConfig（Agent 配置）](#agentconfigagent-配置)
   - [ProviderConfig（Provider 配置）](#providerconfigprovider-配置)
   - [Model（模型）](#model模型)
   - [Provider（提供商）](#provider提供商)
3. [通信实体](#3-通信实体)
   - [PermissionRequest（权限请求）](#permissionrequest权限请求)
   - [QuestionRequest（问答请求）](#questionrequest问答请求)
   - [Auth（认证）](#auth认证)
4. [事件系统](#4-事件系统)
5. [实体关系图](#5-实体关系图)

---

## 1. 核心实体

### Session（会话）

会话是对话的核心容器，代表一次完整的用户与 AI 交互过程。

| 字段         | 类型              | 必填 | 说明                                    |
| ------------ | ----------------- | ---- | --------------------------------------- |
| `id`         | string            | 是   | 会话唯一标识，格式 `ses.*`              |
| `slug`       | string            | 是   | 会话短标识                              |
| `projectID`  | string            | 是   | 关联的项目 ID                           |
| `directory`  | string            | 是   | 工作目录路径                            |
| `parentID`   | string            | 否   | 父会话 ID（用于分支会话），格式 `ses.*` |
| `title`      | string            | 是   | 会话标题                                |
| `version`    | string            | 是   | 会话版本号                              |
| `summary`    | SessionSummary    | 否   | 会话摘要，包含文件变更统计              |
| `share`      | SessionShare      | 否   | 分享信息，包含分享 URL                  |
| `time`       | SessionTime       | 是   | 时间戳集合                              |
| `permission` | PermissionRuleset | 否   | 权限规则集                              |
| `revert`     | SessionRevert     | 否   | 回退状态信息                            |

**SessionSummary（会话摘要）**

| 字段        | 类型       | 必填 | 说明         |
| ----------- | ---------- | ---- | ------------ |
| `additions` | number     | 是   | 新增行数     |
| `deletions` | number     | 是   | 删除行数     |
| `files`     | number     | 是   | 变更文件数   |
| `diffs`     | FileDiff[] | 否   | 文件差异列表 |

**SessionTime（会话时间）**

| 字段         | 类型   | 必填 | 说明                    |
| ------------ | ------ | ---- | ----------------------- |
| `created`    | number | 是   | 创建时间（Unix 时间戳） |
| `updated`    | number | 是   | 更新时间                |
| `compacting` | number | 否   | 压缩时间                |
| `archived`   | number | 否   | 归档时间                |

**SessionStatus（会话状态）**

会话状态是一个多态类型：

| 状态    | 字段                         | 说明                                         |
| ------- | ---------------------------- | -------------------------------------------- |
| `idle`  | —                            | 空闲，无活跃任务                             |
| `busy`  | —                            | 忙碌，正在处理                               |
| `retry` | `attempt`, `message`, `next` | 重试中，包含重试次数、错误信息和下次重试时间 |

---

### Message（消息）

消息是会话中的通信单元，分为两种角色。

#### UserMessage（用户消息）

| 字段        | 类型                    | 必填 | 说明                             |
| ----------- | ----------------------- | ---- | -------------------------------- |
| `id`        | string                  | 是   | 消息唯一标识                     |
| `sessionID` | string                  | 是   | 所属会话 ID                      |
| `role`      | `"user"`                | 是   | 消息角色                         |
| `time`      | MessageTime             | 是   | 时间信息                         |
| `agent`     | string                  | 是   | 使用的 Agent 名称                |
| `model`     | ModelRef                | 是   | 使用的模型引用                   |
| `summary`   | MessageSummary          | 否   | 消息摘要，含标题、正文和文件差异 |
| `system`    | string                  | 否   | 系统提示词                       |
| `tools`     | Record<string, boolean> | 否   | 可用工具映射                     |
| `variant`   | string                  | 否   | 模型变体                         |

#### AssistantMessage（助手消息）

| 字段         | 类型          | 必填 | 说明                  |
| ------------ | ------------- | ---- | --------------------- |
| `id`         | string        | 是   | 消息唯一标识          |
| `sessionID`  | string        | 是   | 所属会话 ID           |
| `role`       | `"assistant"` | 是   | 消息角色              |
| `time`       | MessageTime   | 是   | 时间信息              |
| `parentID`   | string        | 是   | 指向关联的用户消息    |
| `modelID`    | string        | 是   | 模型 ID               |
| `providerID` | string        | 是   | 提供商 ID             |
| `mode`       | string        | 是   | 运行模式              |
| `agent`      | string        | 是   | Agent 名称            |
| `path`       | PathInfo      | 是   | 路径信息（cwd, root） |
| `cost`       | number        | 是   | 本次调用费用          |
| `tokens`     | TokenUsage    | 是   | Token 用量统计        |
| `finish`     | string        | 否   | 完成原因              |
| `error`      | MessageError  | 否   | 错误信息              |
| `summary`    | boolean       | 否   | 是否为摘要消息        |

**MessageTime（消息时间）**

| 字段        | 类型   | 必填 | 说明     |
| ----------- | ------ | ---- | -------- |
| `created`   | number | 是   | 创建时间 |
| `completed` | number | 否   | 完成时间 |

**TokenUsage（Token 用量）**

| 字段          | 类型   | 必填 | 说明              |
| ------------- | ------ | ---- | ----------------- |
| `input`       | number | 是   | 输入 token 数     |
| `output`      | number | 是   | 输出 token 数     |
| `reasoning`   | number | 是   | 推理 token 数     |
| `cache.read`  | number | 是   | 缓存读取 token 数 |
| `cache.write` | number | 是   | 缓存写入 token 数 |

---

### Part（内容片段）

Part 是消息内部的多态内容单元。一条消息包含多个 Part，按时间顺序排列。

#### 公共基础字段

所有 Part 共享以下基础字段：

| 字段        | 类型   | 说明          |
| ----------- | ------ | ------------- |
| `id`        | string | Part 唯一标识 |
| `sessionID` | string | 所属会话 ID   |
| `messageID` | string | 所属消息 ID   |
| `type`      | string | Part 类型标识 |

#### Part 类型一览

| Part 类型          | type 值       | 核心字段                                  | 说明              |
| ------------------ | ------------- | ----------------------------------------- | ----------------- |
| **TextPart**       | `text`        | `text`                                    | 纯文本内容        |
| **ReasoningPart**  | `reasoning`   | `text`, `time`                            | 模型推理/思考过程 |
| **ToolPart**       | `tool`        | `callID`, `tool`, `state`                 | 工具调用及其状态  |
| **FilePart**       | `file`        | `mime`, `url`, `filename`, `source`       | 文件附件          |
| **SubtaskPart**    | `subtask`     | `prompt`, `description`, `agent`, `model` | 子任务委派        |
| **StepStartPart**  | `step-start`  | `snapshot`                                | 步骤开始标记      |
| **StepFinishPart** | `step-finish` | `reason`, `cost`, `tokens`                | 步骤结束，含用量  |
| **SnapshotPart**   | `snapshot`    | `snapshot`                                | 快照引用          |
| **PatchPart**      | `patch`       | `hash`, `files`                           | 补丁信息          |
| **AgentPart**      | `agent`       | `name`, `source`                          | Agent 切换标记    |
| **RetryPart**      | `retry`       | `attempt`, `error`, `time`                | 重试记录          |
| **CompactionPart** | `compaction`  | `auto`                                    | 上下文压缩标记    |

#### TextPart 详细字段

| 字段        | 类型            | 必填 | 说明                   |
| ----------- | --------------- | ---- | ---------------------- |
| `text`      | string          | 是   | 文本内容               |
| `synthetic` | boolean         | 否   | 是否为系统生成的上下文 |
| `ignored`   | boolean         | 否   | 是否被忽略             |
| `time`      | { start, end? } | 否   | 时间范围               |
| `metadata`  | object          | 否   | 附加元数据             |

#### ReasoningPart 详细字段

| 字段       | 类型            | 必填 | 说明       |
| ---------- | --------------- | ---- | ---------- |
| `text`     | string          | 是   | 推理文本   |
| `time`     | { start, end? } | 是   | 时间范围   |
| `metadata` | object          | 否   | 附加元数据 |

#### ToolPart 详细字段

| 字段       | 类型      | 必填 | 说明               |
| ---------- | --------- | ---- | ------------------ |
| `callID`   | string    | 是   | 工具调用 ID        |
| `tool`     | string    | 是   | 工具名称           |
| `state`    | ToolState | 是   | 工具状态（见下表） |
| `metadata` | object    | 否   | 附加元数据         |

**ToolState（工具状态）** 是一个多态类型：

| 状态        | 核心字段                                          | 说明     |
| ----------- | ------------------------------------------------- | -------- |
| `pending`   | `input`, `raw`                                    | 等待执行 |
| `running`   | `input`, `title`, `time.start`                    | 正在执行 |
| `completed` | `input`, `output`, `title`, `time`, `attachments` | 执行完成 |
| `error`     | `input`, `error`, `time`                          | 执行出错 |

#### FilePart 详细字段

| 字段       | 类型           | 必填 | 说明                                                   |
| ---------- | -------------- | ---- | ------------------------------------------------------ |
| `mime`     | string         | 是   | MIME 类型                                              |
| `url`      | string         | 是   | 文件 URL                                               |
| `filename` | string         | 否   | 文件名                                                 |
| `source`   | FilePartSource | 否   | 文件来源（FileSource / SymbolSource / ResourceSource） |

**FilePartSource** 有三种子类型：

| 类型             | 核心字段                                | 说明          |
| ---------------- | --------------------------------------- | ------------- |
| `FileSource`     | `path`, `text`                          | 来自文件系统  |
| `SymbolSource`   | `path`, `range`, `name`, `kind`, `text` | 来自代码符号  |
| `ResourceSource` | `clientName`, `uri`, `text`             | 来自 MCP 资源 |

#### SubtaskPart 详细字段

| 字段          | 类型     | 必填 | 说明            |
| ------------- | -------- | ---- | --------------- |
| `prompt`      | string   | 是   | 子任务提示词    |
| `description` | string   | 是   | 子任务描述      |
| `agent`       | string   | 是   | 执行 Agent 名称 |
| `model`       | ModelRef | 否   | 使用的模型      |
| `command`     | string   | 否   | 关联命令        |

#### StepFinishPart 详细字段

| 字段       | 类型       | 必填 | 说明              |
| ---------- | ---------- | ---- | ----------------- |
| `reason`   | string     | 是   | 结束原因          |
| `cost`     | number     | 是   | 本步骤费用        |
| `tokens`   | TokenUsage | 是   | 本步骤 Token 用量 |
| `snapshot` | string     | 否   | 快照引用          |

#### PatchPart 详细字段

| 字段    | 类型     | 必填 | 说明               |
| ------- | -------- | ---- | ------------------ |
| `hash`  | string   | 是   | 补丁哈希           |
| `files` | string[] | 是   | 涉及的文件路径列表 |

#### AgentPart 详细字段

| 字段     | 类型                  | 必填 | 说明         |
| -------- | --------------------- | ---- | ------------ |
| `name`   | string                | 是   | Agent 名称   |
| `source` | { value, start, end } | 否   | 源码位置引用 |

#### RetryPart 详细字段

| 字段      | 类型        | 必填 | 说明         |
| --------- | ----------- | ---- | ------------ |
| `attempt` | number      | 是   | 当前重试次数 |
| `error`   | APIError    | 是   | 错误详情     |
| `time`    | { created } | 是   | 创建时间     |

#### CompactionPart 详细字段

| 字段   | 类型    | 必填 | 说明           |
| ------ | ------- | ---- | -------------- |
| `auto` | boolean | 是   | 是否为自动压缩 |

---

### Todo（待办事项）

Todo 是会话内的任务跟踪项。

| 字段       | 类型   | 必填 | 说明                                                     |
| ---------- | ------ | ---- | -------------------------------------------------------- |
| `id`       | string | 是   | 待办唯一标识                                             |
| `content`  | string | 是   | 任务描述                                                 |
| `status`   | string | 是   | 状态：`pending`、`in_progress`、`completed`、`cancelled` |
| `priority` | string | 是   | 优先级：`high`、`medium`、`low`                          |

> 注意：后端 OpenAPI 的 Todo 不包含 `id` 字段，前端在适配层为其补充了稳定的 `id` 作为渲染 key。

---

### PTY（伪终端）

PTY 代表一个终端进程实例。

| 字段      | 类型     | 必填 | 说明                       |
| --------- | -------- | ---- | -------------------------- |
| `id`      | string   | 是   | PTY 唯一标识，格式 `pty.*` |
| `title`   | string   | 是   | 显示标题                   |
| `command` | string   | 是   | 执行的命令                 |
| `args`    | string[] | 是   | 命令参数                   |
| `cwd`     | string   | 是   | 工作目录                   |
| `status`  | string   | 是   | 状态：`running`、`exited`  |
| `pid`     | number   | 是   | 进程 ID                    |

---

### Project（项目）

Project 代表一个代码项目。

| 字段        | 类型            | 必填 | 说明              |
| ----------- | --------------- | ---- | ----------------- |
| `id`        | string          | 是   | 项目唯一标识      |
| `worktree`  | string          | 是   | Git worktree 路径 |
| `vcs`       | `"git"`         | 否   | 版本控制系统类型  |
| `name`      | string          | 否   | 项目名称          |
| `icon`      | ProjectIcon     | 否   | 项目图标          |
| `commands`  | ProjectCommands | 否   | 项目命令配置      |
| `time`      | ProjectTime     | 是   | 时间戳集合        |
| `sandboxes` | string[]        | 是   | 沙箱列表          |

**ProjectIcon（项目图标）**

| 字段       | 类型   | 说明     |
| ---------- | ------ | -------- |
| `url`      | string | 图标 URL |
| `override` | string | 覆盖图标 |
| `color`    | string | 图标颜色 |

**ProjectTime（项目时间）**

| 字段          | 类型   | 必填 | 说明       |
| ------------- | ------ | ---- | ---------- |
| `created`     | number | 是   | 创建时间   |
| `updated`     | number | 是   | 更新时间   |
| `initialized` | number | 否   | 初始化时间 |

---

### FileDiff（文件差异）

FileDiff 描述文件的变更差异。

| 字段        | 类型   | 必填 | 说明              |
| ----------- | ------ | ---- | ----------------- |
| `file`      | string | 是   | 文件路径          |
| `before`    | string | 是   | 变更前的内容/引用 |
| `after`     | string | 是   | 变更后的内容/引用 |
| `additions` | number | 是   | 新增行数          |
| `deletions` | number | 是   | 删除行数          |

---

## 2. 配置实体

### Config（全局配置）

Config 是 OpenCode 的全局配置根对象，涵盖 Agent、Provider、MCP、快捷键、权限等。

| 字段                 | 类型                                       | 说明                                           |
| -------------------- | ------------------------------------------ | ---------------------------------------------- |
| `$schema`            | string                                     | JSON Schema 引用                               |
| `theme`              | string                                     | 界面主题名称                                   |
| `keybinds`           | KeybindsConfig                             | 快捷键配置                                     |
| `logLevel`           | LogLevel                                   | 日志级别：`DEBUG`、`INFO`、`WARN`、`ERROR`     |
| `tui`                | TUIConfig                                  | TUI 专属设置（滚动速度、diff 样式等）          |
| `server`             | ServerConfig                               | 服务器配置（端口、主机、mDNS、CORS）           |
| `command`            | Record<string, CommandConfig>              | 自定义命令配置                                 |
| `skills`             | { paths: string[] }                        | 技能文件夹路径                                 |
| `watcher`            | { ignore: string[] }                       | 文件监听忽略规则                               |
| `plugin`             | string[]                                   | 插件列表                                       |
| `snapshot`           | boolean                                    | 是否启用快照                                   |
| `share`              | `"manual"` \| `"auto"` \| `"disabled"`     | 分享行为控制                                   |
| `autoupdate`         | boolean \| `"notify"`                      | 自动更新策略                                   |
| `disabled_providers` | string[]                                   | 禁用的 Provider 列表                           |
| `enabled_providers`  | string[]                                   | 仅启用的 Provider 列表（白名单）               |
| `model`              | string                                     | 默认模型，格式 `provider/model`                |
| `small_model`        | string                                     | 小模型，用于标题生成等任务                     |
| `default_agent`      | string                                     | 默认 Agent 名称                                |
| `username`           | string                                     | 自定义显示用户名                               |
| `agent`              | Record<string, AgentConfig>                | Agent 配置（plan、build、general、explore 等） |
| `provider`           | Record<string, ProviderConfig>             | 自定义 Provider 配置                           |
| `mcp`                | Record<string, McpConfig>                  | MCP 服务器配置                                 |
| `formatter`          | boolean \| Record<string, FormatterConfig> | 代码格式化配置                                 |
| `lsp`                | boolean \| Record<string, LspConfig>       | LSP 服务器配置                                 |
| `instructions`       | string[]                                   | 指令文件路径或模式                             |
| `permission`         | PermissionConfig                           | 全局权限配置                                   |
| `tools`              | Record<string, boolean>                    | 工具启用/禁用映射                              |
| `enterprise`         | { url: string }                            | 企业版 URL                                     |
| `compaction`         | { auto: boolean, prune: boolean }          | 上下文压缩配置                                 |
| `experimental`       | ExperimentalConfig                         | 实验性功能配置                                 |

---

### AgentConfig（Agent 配置）

| 字段          | 类型                                   | 说明                                      |
| ------------- | -------------------------------------- | ----------------------------------------- |
| `model`       | string                                 | 绑定的模型                                |
| `temperature` | number                                 | 温度参数                                  |
| `top_p`       | number                                 | Top-p 采样参数                            |
| `prompt`      | string                                 | 系统提示词                                |
| `tools`       | Record<string, boolean>                | 工具启用映射（已废弃，使用 `permission`） |
| `disable`     | boolean                                | 是否禁用                                  |
| `description` | string                                 | Agent 用途描述                            |
| `mode`        | `"subagent"` \| `"primary"` \| `"all"` | Agent 模式                                |
| `hidden`      | boolean                                | 是否在 @ 菜单中隐藏                       |
| `options`     | object                                 | 附加选项                                  |
| `color`       | string                                 | 十六进制颜色代码                          |
| `steps`       | number                                 | 最大迭代步数                              |
| `permission`  | PermissionConfig                       | 权限配置                                  |

---

### ProviderConfig（Provider 配置）

| 字段        | 类型                        | 说明                                    |
| ----------- | --------------------------- | --------------------------------------- |
| `api`       | string                      | API 类型标识                            |
| `name`      | string                      | 显示名称                                |
| `env`       | string[]                    | 环境变量名列表                          |
| `id`        | string                      | Provider ID                             |
| `npm`       | string                      | NPM 包名                                |
| `models`    | Record<string, ModelConfig> | 模型配置映射                            |
| `whitelist` | string[]                    | 模型白名单                              |
| `blacklist` | string[]                    | 模型黑名单                              |
| `options`   | ProviderOptions             | 连接选项（apiKey、baseURL、timeout 等） |

---

### Model（模型）

Model 描述一个可用的 AI 模型。

| 字段           | 类型                   | 必填 | 说明                                          |
| -------------- | ---------------------- | ---- | --------------------------------------------- |
| `id`           | string                 | 是   | 模型唯一标识                                  |
| `providerID`   | string                 | 是   | 所属 Provider ID                              |
| `api`          | { id, url, npm }       | 是   | API 连接信息                                  |
| `name`         | string                 | 是   | 模型显示名称                                  |
| `family`       | string                 | 否   | 模型家族（如 claude、gpt）                    |
| `capabilities` | ModelCapabilities      | 是   | 能力描述                                      |
| `cost`         | ModelCost              | 是   | 定价信息                                      |
| `limit`        | ModelLimit             | 是   | 限制参数                                      |
| `status`       | string                 | 是   | 状态：`active`、`alpha`、`beta`、`deprecated` |
| `options`      | object                 | 是   | 附加选项                                      |
| `headers`      | Record<string, string> | 是   | 自定义请求头                                  |
| `release_date` | string                 | 是   | 发布日期                                      |
| `variants`     | Record<string, object> | 否   | 模型变体配置                                  |

**ModelCapabilities（模型能力）**

| 字段          | 类型                 | 说明                                       |
| ------------- | -------------------- | ------------------------------------------ |
| `temperature` | boolean              | 是否支持温度调节                           |
| `reasoning`   | boolean              | 是否支持推理模式                           |
| `attachment`  | boolean              | 是否支持附件                               |
| `toolcall`    | boolean              | 是否支持工具调用                           |
| `input`       | IOCapabilities       | 输入模态能力（text/audio/image/video/pdf） |
| `output`      | IOCapabilities       | 输出模态能力                               |
| `interleaved` | boolean \| { field } | 是否支持交错输出                           |

**ModelLimit（模型限制）**

| 字段      | 类型   | 说明              |
| --------- | ------ | ----------------- |
| `context` | number | 上下文窗口大小    |
| `input`   | number | 最大输入 token 数 |
| `output`  | number | 最大输出 token 数 |

**ModelCost（模型定价）**

| 字段                   | 类型   | 说明                     |
| ---------------------- | ------ | ------------------------ |
| `input`                | number | 输入单价（每百万 token） |
| `output`               | number | 输出单价                 |
| `cache.read`           | number | 缓存读取单价             |
| `cache.write`          | number | 缓存写入单价             |
| `experimentalOver200K` | object | 超过 200K 上下文的定价   |

---

### Provider（提供商）

Provider 描述一个 AI 服务提供商。

| 字段      | 类型                  | 必填 | 说明                                   |
| --------- | --------------------- | ---- | -------------------------------------- |
| `id`      | string                | 是   | Provider 唯一标识                      |
| `name`    | string                | 是   | 显示名称                               |
| `source`  | string                | 是   | 来源：`env`、`config`、`custom`、`api` |
| `env`     | string[]              | 是   | 所需环境变量列表                       |
| `key`     | string                | 否   | API Key                                |
| `options` | object                | 是   | 附加选项                               |
| `models`  | Record<string, Model> | 是   | 可用模型映射                           |

---

## 3. 通信实体

### PermissionRequest（权限请求）

当 Agent 需要执行受限操作时，向用户发起权限请求。

| 字段         | 类型                  | 必填 | 说明                       |
| ------------ | --------------------- | ---- | -------------------------- |
| `id`         | string                | 是   | 请求唯一标识，格式 `per.*` |
| `sessionID`  | string                | 是   | 所属会话 ID                |
| `permission` | string                | 是   | 权限类型                   |
| `patterns`   | string[]              | 是   | 匹配模式列表               |
| `metadata`   | object                | 是   | 附加元数据                 |
| `always`     | string[]              | 是   | 持久化权限列表             |
| `tool`       | { messageID, callID } | 否   | 关联的工具调用信息         |

**权限回复（PermissionReply）**

回复值为以下三种之一：

| 值       | 说明             |
| -------- | ---------------- |
| `once`   | 仅本次允许       |
| `always` | 本次会话永久允许 |
| `reject` | 拒绝             |

---

### QuestionRequest（问答请求）

当 Agent 需要向用户提问以获取额外信息时发出。

| 字段        | 类型                  | 必填 | 说明                       |
| ----------- | --------------------- | ---- | -------------------------- |
| `id`        | string                | 是   | 请求唯一标识，格式 `que.*` |
| `sessionID` | string                | 是   | 所属会话 ID                |
| `questions` | QuestionInfo[]        | 是   | 问题列表                   |
| `tool`      | { messageID, callID } | 否   | 关联的工具调用信息         |

**QuestionInfo（问题信息）**

| 字段       | 类型             | 必填 | 说明                            |
| ---------- | ---------------- | ---- | ------------------------------- |
| `question` | string           | 是   | 完整问题文本                    |
| `header`   | string           | 是   | 简短标题（最多 30 字符）        |
| `options`  | QuestionOption[] | 是   | 可选答案列表                    |
| `multiple` | boolean          | 否   | 是否允许多选                    |
| `custom`   | boolean          | 否   | 是否允许自定义输入（默认 true） |

**QuestionOption（答案选项）**

| 字段          | 类型   | 必填 | 说明               |
| ------------- | ------ | ---- | ------------------ |
| `label`       | string | 是   | 显示文本（1-5 词） |
| `description` | string | 是   | 选项说明           |

**QuestionAnswer（问题答案）**

类型为 `string[]`，表示用户选择的答案列表。

---

### Auth（认证）

Auth 描述 Provider 的认证方式，是一个多态类型。

| 类型            | 核心字段                                                                      | 说明           |
| --------------- | ----------------------------------------------------------------------------- | -------------- |
| `OAuth`         | `type: "oauth"`, `refresh`, `access`, `expires`, `accountId`, `enterpriseUrl` | OAuth 2.0 认证 |
| `ApiAuth`       | `type: "api"`, `key`                                                          | API Key 认证   |
| `WellKnownAuth` | `type: "wellknown"`, `key`, `token`                                           | 已知认证方式   |

---

## 4. 事件系统

OpenCode 使用 SSE（Server-Sent Events）进行实时事件推送。所有事件通过 `GlobalEvent` 结构传递，包含 `directory` 和 `payload` 两个字段。

事件类型常量定义在前端 `src/types/api/event.ts` 的 `EventTypes` 对象中。

### 事件分类与类型列表

#### 服务器事件

| 事件类型                   | 负载            | 说明             |
| -------------------------- | --------------- | ---------------- |
| `server.connected`         | —               | 服务器连接成功   |
| `server.instance.disposed` | `{ directory }` | 服务器实例已释放 |
| `global.disposed`          | —               | 全局服务已释放   |

#### 会话事件（Session）

| 事件类型            | 负载                                   | 说明         |
| ------------------- | -------------------------------------- | ------------ |
| `session.created`   | `{ info: Session }`                    | 会话已创建   |
| `session.updated`   | `{ info: Session }`                    | 会话已更新   |
| `session.deleted`   | `{ info: Session }`                    | 会话已删除   |
| `session.status`    | `{ sessionID, status: SessionStatus }` | 会话状态变更 |
| `session.idle`      | `{ sessionID }`                        | 会话进入空闲 |
| `session.error`     | `{ sessionID, error }`                 | 会话发生错误 |
| `session.diff`      | `{ sessionID, diff: FileDiff[] }`      | 会话文件差异 |
| `session.compacted` | `{ sessionID }`                        | 会话已压缩   |

#### 消息事件（Message）

| 事件类型               | 负载                               | 说明                             |
| ---------------------- | ---------------------------------- | -------------------------------- |
| `message.updated`      | `{ info: Message }`                | 消息已更新                       |
| `message.removed`      | `{ sessionID, messageID }`         | 消息已删除                       |
| `message.part.updated` | `{ part: Part, delta?: string }`   | 消息片段已更新，delta 为增量文本 |
| `message.part.removed` | `{ sessionID, messageID, partID }` | 消息片段已删除                   |

#### 权限事件（Permission）

| 事件类型             | 负载                              | 说明           |
| -------------------- | --------------------------------- | -------------- |
| `permission.asked`   | `PermissionRequest`               | 发起权限请求   |
| `permission.replied` | `{ sessionID, requestID, reply }` | 权限请求已回复 |

#### 问答事件（Question）

| 事件类型            | 负载                                | 说明         |
| ------------------- | ----------------------------------- | ------------ |
| `question.asked`    | `QuestionRequest`                   | 发起问答请求 |
| `question.replied`  | `{ sessionID, requestID, answers }` | 问答已回复   |
| `question.rejected` | `{ sessionID, requestID }`          | 问答已拒绝   |

#### 终端事件（PTY）

| 事件类型      | 负载               | 说明       |
| ------------- | ------------------ | ---------- |
| `pty.created` | `{ info: Pty }`    | 终端已创建 |
| `pty.updated` | `{ info: Pty }`    | 终端已更新 |
| `pty.exited`  | `{ id, exitCode }` | 终端已退出 |
| `pty.deleted` | `{ id }`           | 终端已删除 |

#### 文件事件（File）

| 事件类型               | 负载              | 说明                                             |
| ---------------------- | ----------------- | ------------------------------------------------ |
| `file.edited`          | `{ file }`        | 文件已被编辑                                     |
| `file.watcher.updated` | `{ file, event }` | 文件监听器更新，event 为 `add`/`change`/`unlink` |

#### MCP 事件

| 事件类型                  | 负载               | 说明               |
| ------------------------- | ------------------ | ------------------ |
| `mcp.tools.changed`       | `{ server }`       | MCP 工具列表变更   |
| `mcp.browser.open.failed` | `{ mcpName, url }` | MCP 浏览器打开失败 |

#### LSP 事件

| 事件类型                 | 负载                 | 说明               |
| ------------------------ | -------------------- | ------------------ |
| `lsp.updated`            | `{}`                 | LSP 状态更新       |
| `lsp.client.diagnostics` | `{ serverID, path }` | LSP 客户端诊断更新 |

#### VCS 事件

| 事件类型             | 负载         | 说明             |
| -------------------- | ------------ | ---------------- |
| `vcs.branch.updated` | `{ branch }` | 版本控制分支更新 |

#### 命令事件（Command）

| 事件类型           | 负载                                        | 说明       |
| ------------------ | ------------------------------------------- | ---------- |
| `command.executed` | `{ name, sessionID, arguments, messageID }` | 命令已执行 |

#### 待办事件（Todo）

| 事件类型       | 负载                           | 说明         |
| -------------- | ------------------------------ | ------------ |
| `todo.updated` | `{ sessionID, todos: Todo[] }` | 待办事项更新 |

#### Worktree 事件

| 事件类型          | 负载               | 说明              |
| ----------------- | ------------------ | ----------------- |
| `worktree.ready`  | `{ name, branch }` | Worktree 就绪     |
| `worktree.failed` | `{ message }`      | Worktree 创建失败 |

#### 安装事件（Installation）

| 事件类型                        | 负载          | 说明         |
| ------------------------------- | ------------- | ------------ |
| `installation.updated`          | `{ version }` | 安装版本更新 |
| `installation.update-available` | `{ version }` | 有新版本可用 |

#### TUI 事件

| 事件类型              | 负载                                     | 说明            |
| --------------------- | ---------------------------------------- | --------------- |
| `tui.prompt.append`   | `{ text }`                               | 提示框追加文本  |
| `tui.command.execute` | `{ command }`                            | 执行 TUI 命令   |
| `tui.toast.show`      | `{ title?, message, variant, duration }` | 显示 Toast 通知 |
| `tui.session.select`  | `{ sessionID }`                          | 选择会话        |

#### 项目事件

| 事件类型          | 负载      | 说明         |
| ----------------- | --------- | ------------ |
| `project.updated` | `Project` | 项目信息更新 |

---

## 5. 实体关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Project（项目）                            │
│  id, name, worktree, icon, sandboxes                             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 1
                           │
                           │ N（一个项目包含多个会话）
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Session（会话）                            │
│  id, slug, projectID, directory, title, version, status          │
│  summary, share, permission, revert, time                        │
└──────┬──────────────┬──────────────┬──────────────┬─────────────┘
       │ 1            │ 1            │ 1            │ 1
       │              │              │              │
       │ N            │ N            │ N            │ N
       ▼              ▼              ▼              ▼
┌──────────┐  ┌───────────┐  ┌──────────────┐  ┌───────────────┐
│ Message  │  │ Todo      │  │ Permission   │  │ Question      │
│（消息）   │  │（待办）    │  │ Request      │  │ Request       │
│          │  │           │  │（权限请求）   │  │（问答请求）    │
│ id,      │  │ id,       │  │ id,          │  │ id,           │
│ role,    │  │ content,  │  │ sessionID,   │  │ sessionID,    │
│ time,    │  │ status,   │  │ permission,  │  │ questions     │
│ agent,   │  │ priority  │  │ patterns,    │  │               │
│ model    │  │           │  │ metadata     │  │               │
└────┬─────┘  └───────────┘  └──────────────┘  └───────────────┘
     │ 1
     │
     │ N（一条消息包含多个 Part）
     ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Part（内容片段）                           │
│  id, sessionID, messageID, type                                  │
│                                                                 │
│  多态类型：                                                       │
│  TextPart │ ReasoningPart │ ToolPart │ FilePart                 │
│  SubtaskPart │ StepStartPart │ StepFinishPart                   │
│  SnapshotPart │ PatchPart │ AgentPart │ RetryPart               │
│  CompactionPart                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     Provider（提供商）                            │
│  id, name, source, env, options                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 1
                           │
                           │ N（一个提供商提供多个模型）
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Model（模型）                              │
│  id, providerID, name, family, capabilities, cost, limit         │
│  status, variants                                                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        PTY（伪终端）                              │
│  id, title, command, args, cwd, status, pid                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     Config（全局配置）                             │
│  theme, keybinds, agent{}, provider{}, mcp{}, permission         │
│  tools{}, formatter{}, lsp{}, compaction{}                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     FileDiff（文件差异）                           │
│  file, before, after, additions, deletions                       │
└─────────────────────────────────────────────────────────────────┘
  ▲
  │ 被引用
  │
  ├── Session.summary.diffs
  └── UserMessage.summary.diffs
```

### 关系说明

| 关系                           | 基数 | 说明                             |
| ------------------------------ | ---- | -------------------------------- |
| Project → Session              | 1:N  | 一个项目包含多个会话             |
| Session → Message              | 1:N  | 一个会话包含多条消息             |
| Session → Todo                 | 1:N  | 一个会话包含多个待办事项         |
| Session → PermissionRequest    | 1:N  | 一个会话可产生多个权限请求       |
| Session → QuestionRequest      | 1:N  | 一个会话可产生多个问答请求       |
| Session → PTY                  | 1:N  | 一个会话可启动多个终端           |
| Message → Part                 | 1:N  | 一条消息包含多个内容片段         |
| Provider → Model               | 1:N  | 一个提供商提供多个模型           |
| Session → FileDiff             | 1:N  | 通过 summary.diffs 关联          |
| UserMessage → FileDiff         | 1:N  | 通过 summary.diffs 关联          |
| Session → Session（自引用）    | 1:N  | 通过 parentID 实现分支会话       |
| AssistantMessage → UserMessage | N:1  | 通过 parentID 指向关联的用户消息 |

---

## 附录：事件流时序示意

```
客户端                              服务端（OpenCode SSE）
  │                                      │
  │  ─── GET /api/events ──────────────►  │
  │  ◄── event: server.connected ───────  │  连接建立
  │                                      │
  │  ◄── event: session.created ────────  │  新会话创建
  │  ◄── event: message.updated ────────  │  消息更新
  │  ◄── event: message.part.updated ───  │  Part 流式推送（delta）
  │  ◄── event: message.part.updated ───  │  ...
  │  ◄── event: todo.updated ───────────  │  待办更新
  │  ◄── event: permission.asked ───────  │  权限请求
  │  ─── POST /api/permissions ────────►  │  用户回复权限
  │  ◄── event: permission.replied ─────  │  权限回复确认
  │  ◄── event: message.part.updated ───  │  继续流式推送
  │  ◄── event: session.idle ───────────  │  会话空闲
  │                                      │
```
