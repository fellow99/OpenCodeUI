# 014-tauri-desktop 技术方案（As-Built）

> 模块编号：014-tauri-desktop
> 文档类型：Implementation Plan（回溯性技术方案）
> 基于版本：OpenCodeUI v0.4.9 / Tauri 2
> 生成日期：2026-04-12

---

## 1. Technical Context

### 1.1 模块定位

本模块将 OpenCodeUI 的 Web 前端（React 19 + TypeScript + Vite 7）打包为原生桌面应用程序，同时兼顾移动端（Android）入口。使用 Tauri 2 框架，Rust 作为原生层，WebView 作为渲染层。

### 1.2 技术栈清单

| 层级         | 技术                                                               | 版本                                          |
| ------------ | ------------------------------------------------------------------ | --------------------------------------------- |
| 框架         | Tauri                                                              | 2.x                                           |
| Rust 版本    | rustc                                                              | >= 1.85.0                                     |
| HTTP 客户端  | reqwest                                                            | 0.12（rustls-tls + stream）                   |
| WebSocket    | tokio-tungstenite                                                  | 0.24（rustls-tls-webpki-roots）               |
| 异步运行时   | tokio                                                              | 1.x（macros + rt-multi-thread + sync + time） |
| 序列化       | serde + serde_json                                                 | 1.x                                           |
| 并发 HashMap | papaya                                                             | 0.2.3                                         |
| 哈希函数     | rapidhash                                                          | 4.4.1                                         |
| 流处理       | futures-util                                                       | 0.3                                           |
| 日志         | log                                                                | 0.4                                           |
| Tauri 插件   | http / notification / dialog / fs / opener / log / single-instance | 2.x                                           |

### 1.3 项目结构（实际代码）

```
src-tauri/
├── Cargo.toml                  # Rust 依赖、Release 优化配置
├── Cargo.lock                  # 依赖锁定文件
├── build.rs                    # Tauri 构建脚本（3 行）
├── tauri.conf.json             # 应用配置（窗口、安全、打包）
├── capabilities/
│   └── default.json            # 权限能力声明
├── icons/                      # 16 个图标文件（多尺寸/多格式）
├── windows/
│   └── hooks.nsi               # NSIS 安装/卸载钩子
├── gen/
│   └── android/                # Android 生成文件
└── src/
    ├── main.rs                 # 桌面入口（Windows 子系统设置）
    ├── lib.rs                  # 移动端入口（#[tauri::mobile_entry_point]）
    └── app/
        ├── mod.rs              # 应用核心：Builder 组装、插件注册、事件处理
        ├── service.rs          # 服务状态（child_pid + we_started）
        ├── dir_state.rs        # 目录状态（desktop only，papaya HashMap）
        ├── bridge/             # 统一桥接模块（SSE + WebSocket）
        │   ├── mod.rs          # 模块导出
        │   ├── args.rs         # ConnectArgs / SendArgs / DisconnectArgs
        │   ├── event.rs        # BridgeEvent 枚举
        │   └── state.rs        # BridgeState（Mutex<HashMap>）
        └── commands/
            ├── mod.rs          # 模块导出（平台条件编译）
            ├── bridge.rs       # bridge_connect / bridge_send / bridge_disconnect
            ├── opencode.rs     # 服务管理（desktop only）
            └── utils.rs        # get_cli_directory（desktop only）
```

### 1.4 前端适配文件

| 文件                 | 职责                                                |
| -------------------- | --------------------------------------------------- |
| `src/utils/tauri.ts` | `isTauri()` / `isTauriMobile()` 检测、MIME 类型映射 |

---

## 2. Constitution Check

对照项目宪法逐项验证：

| 宪法原则                    | 符合性 | 说明                                                                                                    |
| --------------------------- | ------ | ------------------------------------------------------------------------------------------------------- |
| 原则 2：OpenCode 兼容性优先 | 符合   | Rust 侧通过 `/global/health` 端点检查服务状态，忠实对接 OpenCode 后端                                   |
| 原则 3：多平台统一代码库    | 符合   | Web 前端与 Tauri 共享 `src/` 代码，`src-tauri/` 仅包含平台特定桥接层。通过 `isTauri()` 检测切换通信方式 |
| 原则 4：自定义优于框架依赖  | 符合   | Bridge 模块为自定义实现，未引入额外通信框架。并发数据结构选用轻量级 papaya 而非重量级方案               |
| 原则 5：实时通信优先        | 符合   | Bridge 模块同时支持 HTTP Streaming（SSE）和 WebSocket 两种实时传输方式                                  |
| 原则 8：零配置用户体验      | 符合   | 安装包一键安装，NSIS 钩子注册右键菜单，macOS 支持拖放打开                                               |
| 原则 10：模块化功能架构     | 符合   | `bridge/`、`commands/`、`service.rs`、`dir_state.rs` 各自独立，职责清晰                                 |
| 约束 C1：GPL-3.0 许可证     | 符合   | Cargo.toml 中 `license = "GPL-3.0-only"`                                                                |
| 约束 C4：依赖最小化         | 符合   | 仅引入必要的 Tauri 插件和底层库，无功能重叠                                                             |
| 约束 C5：SSE 支持           | 符合   | Bridge 模块的 HTTP Streaming 传输层完整实现 SSE 桥接                                                    |

---

## 3. Research Findings

### 3.1 架构演进：从 SSE 专用到统一 Bridge

Spec 文档描述的是独立的 SSE 模块（`src/app/sse/`），但实际代码已演进为统一的 **Bridge 架构**（`src/app/bridge/`）。这一变化将 SSE 桥接和 WebSocket 桥接合并为一套透明代理系统：

- **前端通过 URL scheme 自动选择传输层**：`http://` / `https://` 走 HTTP Streaming，`ws://` / `wss://` 走 WebSocket
- **统一的事件接口**：`BridgeEvent` 枚举覆盖 Connected / Data / Disconnected / Error 四种事件
- **统一的命令接口**：`bridge_connect` / `bridge_send` / `bridge_disconnect` 三组命令

这意味着 PTY 终端的 WebSocket 连接也通过同一套 Bridge 机制实现，而非独立的 WebSocket 模块。

### 3.2 并发数据结构选择

Spec 提到 SSE 状态使用 papaya 并发 HashMap，但实际代码中：

- **BridgeState** 使用 `std::sync::Mutex<HashMap<BridgeKey, BridgeConnection>>`。选择 Mutex 而非 papaya 的原因在于 Bridge 的读写频率相对均衡（连接建立/断开频繁），且需要原子性的 replace 操作
- **OpenDirectoryState** 使用 `papaya::HashMap<String, Arc<str>, rapidhash::fast::RandomState>`。目录状态属于读多写少场景，papaya 的无锁并发更合适

### 3.3 连接标识机制

Bridge 使用双层标识：

1. **bridge_id**：前端传入的不透明标签（如 `"sse"`、PTY ID），由前端业务决定
2. **conn_id**：Rust 侧分配的递增 `AtomicU64`，用于防止旧连接任务误清理新连接

复合键 `BridgeKey = (window_label, bridge_id)` 确保多窗口场景下连接隔离。

### 3.4 服务健康检查端点

实际代码使用 `/global/health` 作为健康检查端点（`opencode.rs` 第 16 行），而非 spec 中笼统描述的"健康检查端点"。

### 3.5 Release 构建优化

Cargo.toml 中 release profile 配置：

```toml
[profile.release]
strip = true          # 剥离符号表
lto = true            # 完整 LTO（非 thin）
panic = "abort"       # Panic 时直接终止
codegen-units = 1     # 单代码生成单元，最大化优化
opt-level = "s"       # 体积优化（非速度优化）
```

注意 `lto = true` 对应完整 LTO，而非 spec 中描述的 `LTO thin`。`opt-level = "s"` 针对二进制体积优化，spec 未提及此细节。

### 3.6 包名与产品名差异

- Cargo.toml `name = "opencodeui"`（Rust crate 名）
- tauri.conf.json `productName = "OpenCode"`（用户可见的应用名）
- tauri.conf.json `identifier = "com.opencodeui.app"`（Bundle ID）

### 3.7 图标资源

`icons/` 目录包含 16 个图标文件，覆盖：

- 桌面图标：32x32、128x128、128x128@2x、icon.icns（macOS）、icon.ico（Windows）
- Windows Store 系列：Square30x30 到 Square310x310 共 11 个尺寸
- 通用：icon.png、StoreLogo.png

---

## 4. Data Model

### 4.1 BridgeState（全局共享）

```rust
struct BridgeState {
    next_id: AtomicU64,                                    // 递增连接 ID 计数器
    active: Mutex<HashMap<BridgeKey, BridgeConnection>>,   // 活跃连接映射
}

struct BridgeKey {
    window_label: String,   // 窗口标签（"main" / "win-1" / ...）
    bridge_id: String,      // 前端传入的桥接 ID
}

struct BridgeConnection {
    id: u64,                                            // 连接唯一 ID
    tx: Option<UnboundedSender<BridgeCommand>>,         // WebSocket 发送通道（HTTP Stream 为 None）
}
```

**行为特征**：

- `next_conn_id()` 从 1 开始递增（`fetch_add + 1`）
- `replace()` 支持同键覆盖，返回旧连接供调用方清理
- `remove_if_current()` 带 ID 校验，防止旧任务误删新连接
- `disconnect_window()` 批量清理指定窗口的所有桥接连接

### 4.2 BridgeEvent（推送到前端）

```rust
#[serde(rename_all = "camelCase", tag = "event", content = "data")]
enum BridgeEvent {
    Connected,                                          // 连接已建立
    Data { data: String },                              // 收到数据（原始字符串）
    Disconnected { code: Option<u16>, reason: String }, // 连接断开
    Error { message: String },                          // 发生错误
}
```

序列化后 JSON 示例：

```json
{ "event": "connected" }
{ "event": "data", "data": "..." }
{ "event": "disconnected", "data": { "code": 1000, "reason": "..." } }
{ "event": "error", "data": { "message": "..." } }
```

### 4.3 ServiceState（全局共享，desktop only）

```rust
struct ServiceState {
    child_pid: AtomicU32,    // 子进程 PID（0 表示无）
    we_started: AtomicBool,  // 是否由本应用启动
}
```

### 4.4 OpenDirectoryState（全局共享，desktop only）

```rust
struct OpenDirectoryState {
    pending: PaHashMap<String, Arc<str>, RandomState>,  // window_label → directory_path
}
```

**行为特征**：

- 使用 `papaya::HashMap` 支持无锁并发读写
- 使用 `rapidhash::fast::RandomState` 作为哈希器
- 值为 `Arc<str>` 避免字符串克隆开销
- 一次性消费：`get_cli_directory` 调用后从 HashMap 中 remove

### 4.5 ConnectArgs / SendArgs / DisconnectArgs

```rust
struct ConnectArgs {
    bridge_id: String,          // 前端选择的桥接 ID
    url: String,                // 目标 URL
    auth_header: Option<String>, // 可选的 Authorization 头
}

struct SendArgs {
    bridge_id: String,          // 目标桥接 ID
    data: String,               // 发送的数据
}

struct DisconnectArgs {
    bridge_id: String,          // 目标桥接 ID
}
```

---

## 5. Interface Contracts

### 5.1 Tauri 命令清单

| 命令                        | 平台 | 参数                                   | 返回值                 | 用途                                              |
| --------------------------- | ---- | -------------------------------------- | ---------------------- | ------------------------------------------------- |
| `bridge_connect`            | 全部 | `ConnectArgs` + `Channel<BridgeEvent>` | `Result<(), String>`   | 建立桥接连接（自动选择 HTTP Stream 或 WebSocket） |
| `bridge_send`               | 全部 | `SendArgs`                             | `Result<(), String>`   | 向 WebSocket 桥接发送数据                         |
| `bridge_disconnect`         | 全部 | `DisconnectArgs`                       | `Result<(), String>`   | 断开指定桥接连接                                  |
| `get_cli_directory`         | 桌面 | 无（从 window 获取 label）             | `Option<Arc<str>>`     | 获取并清除启动时传入的目录路径                    |
| `check_opencode_service`    | 桌面 | `url: String`                          | `Result<bool, String>` | 检查 OpenCode 服务是否运行                        |
| `start_opencode_service`    | 桌面 | `url`, `binary_path`, `env_vars`       | `Result<bool, String>` | 启动 OpenCode 服务                                |
| `stop_opencode_service`     | 桌面 | 无                                     | `Result<(), String>`   | 停止 OpenCode 服务                                |
| `get_service_started_by_us` | 桌面 | 无                                     | `Result<bool, String>` | 查询服务是否由本应用启动                          |
| `confirm_close_app`         | 桌面 | `stop_service: bool`                   | `Result<(), String>`   | 确认关闭应用，可选停止服务                        |

### 5.2 能力声明（capabilities/default.json）

```json
{
  "identifier": "default",
  "windows": ["main", "win-*"],
  "permissions": [
    "core:default",
    "http:default + 6 条 URL 白名单（http/https + 端口 + 路径）",
    "notification:default + allow-is-permission-granted + allow-request-permission + allow-notify",
    "dialog:default + allow-open + allow-save",
    "fs:default + allow-read-file + allow-write-file",
    "opener:default"
  ]
}
```

**注意**：能力声明中未包含 `log` 插件的显式权限。log 插件在 `setup()` 中动态注册，使用默认权限即可。

### 5.3 窗口配置（tauri.conf.json）

| 配置项               | 值                        | 说明                           |
| -------------------- | ------------------------- | ------------------------------ |
| `productName`        | `"OpenCode"`              | 用户可见的应用名称             |
| `identifier`         | `"com.opencodeui.app"`    | Bundle ID                      |
| `version`            | `"0.4.9"`                 | 应用版本                       |
| `frontendDist`       | `"../dist"`               | 生产构建产物路径               |
| `devUrl`             | `"http://localhost:5173"` | 开发服务器地址                 |
| `beforeDevCommand`   | `"npm run dev"`           | 开发前启动 Vite                |
| `beforeBuildCommand` | `"npm run build"`         | 构建前执行前端构建             |
| 窗口默认尺寸         | 800x600                   | 可自由调整                     |
| `dragDropEnabled`    | `false`                   | 禁用文件拖放至窗口             |
| `csp`                | `null`                    | 不设置 Content Security Policy |
| 打包目标             | `["nsis", "dmg", "deb"]`  | 三大平台安装包格式             |
| NSIS 钩子            | `"./windows/hooks.nsi"`   | Windows 安装/卸载脚本          |

### 5.4 前端检测接口

```typescript
// src/utils/tauri.ts
function isTauri(): boolean // 检查 window.__TAURI_INTERNALS__
function isTauriMobile(): boolean // 检查 navigator.userAgent 中的移动设备标识
function extToMime(ext: string): string // 文件扩展名到 MIME 类型映射
```

---

## 6. Implementation Strategy

### 6.1 应用启动流程

```
main.rs (desktop) / lib.rs (mobile)
    │
    ▼
app::run()
    │
    ├─ 1. Builder::default().manage(BridgeState::default())
    │
    ├─ 2. [Desktop] manage(OpenDirectoryState::default())
    │         .plugin(single_instance::init(|app, args, _cwd| {
    │             新建窗口（始终新建，类似 VSCode）
    │         }))
    │
    ├─ 3. 注册插件：http / notification / dialog / fs / opener
    │
    ├─ 4. setup(|app| {
    │       注册 log 插件（Info 级别）
    │       [Debug] 主窗口自动打开 DevTools
    │       [Desktop] 解析 CLI 参数，存入 OpenDirectoryState
    │   })
    │
    ├─ 5. [Desktop] manage(ServiceState::default())
    │         .on_window_event(窗口关闭拦截 + 销毁清理)
    │         .invoke_handler(9 个命令)
    │
    ├─ 6. [Android] invoke_handler(3 个 bridge 命令)
    │
    ├─ 7. build(generate_context!())
    │
    └─ 8. run(|app, event| {
          [macOS] 处理 RunEvent::Opened { urls }
          → 文件夹拖放 / Finder 右键打开
        })
```

### 6.2 Bridge 连接建立流程

```
前端调用 bridge_connect({ bridgeId, url, authHeader }, channel)
    │
    ├─ URL scheme 判断
    │   ├─ ws:// / wss:// → connect_ws()
    │   └─ http:// / https:// → connect_stream()
    │
    ├─ connect_stream():
    │   1. 分配 conn_id，注册到 BridgeState
    │   2. 创建 reqwest Client（connect_timeout=15s, tcp_keepalive=30s）
    │   3. 发起 GET 请求，可选携带 Authorization 头
    │   4. 响应成功 → emit Connected
    │   5. 流式读取 bytes_stream，90s 读取超时
    │   6. 每个 chunk → emit Data { data }
    │   7. 循环中检查 is_current() 检测取消
    │
    └─ connect_ws():
        1. 分配 conn_id，创建 mpsc channel，注册到 BridgeState
        2. 构建 WebSocket 请求，可选携带 Authorization 头
        3. connect_async 建立连接
        4. 响应成功 → emit Connected
        5. tokio::select! 双向处理：
           - rx.recv() → 前端发来数据 → write.send()
           - read.next() → 服务端发来数据 → emit Data
        6. 自动处理 Ping/Pong
        7. Close 帧 → emit Disconnected
```

### 6.3 多窗口管理

```
主窗口（"main"）
    │
    ├─ 启动时创建，加载 index.html
    ├─ CLI 参数中的目录路径写入 pending["main"]
    └─ 关闭时：
        ├─ 如果是最后一个窗口 且 we_started == true
        │   → prevent_close() + emit "close-requested"
        │   → 前端弹出确认对话框
        │   → 用户确认后调用 confirm_close_app(stop_service)
        └─ 否则直接关闭
            → Destroyed 事件 → disconnect_window(label)

动态窗口（"win-{N}"）
    │
    ├─ 由 single-instance 或 macOS Opened 事件触发创建
    ├─ 原子计数器 WIN_COUNTER 保证标签唯一
    ├─ 目录路径写入 pending["win-{N}"]
    └─ 关闭时：
        └─ Destroyed 事件 → disconnect_window(label)
```

### 6.4 服务管理流程

```
start_opencode_service(url, binary_path, env_vars)
    │
    ├─ 1. is_service_running(url) → GET /global/health
    │   └─ 已运行 → 返回 false
    │
    ├─ 2. spawn_opencode_serve(binary_path, env_vars)
    │   ├─ stdout/stderr → null
    │   ├─ [Windows] CREATE_NO_WINDOW 隐藏控制台
    │   └─ 注入用户配置的环境变量
    │
    ├─ 3. 记录 PID 和 we_started = true
    │
    ├─ 4. 轮询健康检查（500ms 间隔，最多 30 次 = 15 秒）
    │   └─ 就绪 → 返回 true
    │   └─ 超时 → 返回 true（warn 日志）
    │
    └─ 5. stop_opencode_service()
        ├─ swap PID → 0, we_started → false
        └─ kill_process_by_pid(pid)
            ├─ [Windows] taskkill /PID /F /T
            └─ [Unix] kill {pid}
```

### 6.5 平台条件编译矩阵

| 模块/功能              | Desktop            | Android | 条件编译方式                                                         |
| ---------------------- | ------------------ | ------- | -------------------------------------------------------------------- |
| `dir_state.rs`         | 包含               | 排除    | `#[cfg(not(target_os = "android"))]`                                 |
| `commands/opencode.rs` | 包含               | 排除    | `#[cfg(not(target_os = "android"))]`                                 |
| `commands/utils.rs`    | 包含               | 排除    | `#[cfg(not(target_os = "android"))]`                                 |
| `single-instance` 插件 | 注册               | 不注册  | `#[cfg(not(target_os = "android"))]`                                 |
| 窗口关闭拦截           | 注册               | 不注册  | `#[cfg(not(target_os = "android"))]`                                 |
| 服务管理命令           | 注册               | 不注册  | `#[cfg(not(target_os = "android"))]`                                 |
| CLI 参数解析           | 执行               | 不执行  | `#[cfg(not(target_os = "android"))]`                                 |
| DevTools 自动打开      | 执行               | 不执行  | `#[cfg(debug_assertions)]`                                           |
| macOS Opened 事件      | 执行               | 不执行  | `#[cfg(target_os = "macos")]`                                        |
| Windows 隐藏控制台     | 执行               | 不执行  | `#[cfg(target_os = "windows")]`                                      |
| Windows 杀进程         | taskkill           | 不适用  | `#[cfg(target_os = "windows")]`                                      |
| Unix 杀进程            | 不适用             | kill    | `#[cfg(not(target_os = "windows"))]`                                 |
| Windows 子系统         | windows（release） | 不适用  | `#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]` |

---

## 7. Error Handling

### 7.1 Bridge 连接错误处理

| 错误场景                      | 处理方式                                     | 前端感知                           |
| ----------------------------- | -------------------------------------------- | ---------------------------------- |
| HTTP 客户端创建失败           | 返回 `Err(msg)`                              | invoke 返回错误                    |
| HTTP 连接失败（网络不可达等） | emit Error + 返回 `Err(msg)`                 | 先收到 Error 事件，invoke 返回错误 |
| HTTP 响应非 2xx               | emit Error + 返回 `Err(msg)`                 | 同上                               |
| HTTP Stream 读取超时（90s）   | emit Error + 返回 `Err(msg)`                 | 同上                               |
| HTTP Stream 读取错误          | emit Error + 返回 `Err(msg)`                 | 同上                               |
| HTTP Stream 正常结束          | emit Disconnected + 返回 `Ok(())`            | 收到 Disconnected 事件             |
| WebSocket 连接失败            | emit Error + 返回 `Err(msg)`                 | 先收到 Error 事件，invoke 返回错误 |
| WebSocket 写入失败            | emit Error + 返回 `Err(msg)`                 | 同上                               |
| WebSocket 服务端关闭          | emit Disconnected + 返回 `Ok(())`            | 收到 Disconnected 事件             |
| 前端主动断开                  | emit Disconnected(code=1000) + 返回 `Ok(())` | 收到 Disconnected 事件             |
| bridge_send 目标不存在        | 返回 `Err("bridge 'X' is not active")`       | invoke 返回错误                    |
| bridge_send 通道已关闭        | 返回 `Err("bridge 'X' is closed")`           | invoke 返回错误                    |

### 7.2 服务管理错误处理

| 错误场景                         | 处理方式                                                      |
| -------------------------------- | ------------------------------------------------------------- |
| 服务已在运行                     | 返回 `Ok(false)`，不启动新进程                                |
| 二进制路径错误                   | 返回 `Err("Failed to start 'X': ...")`                        |
| 健康检查超时（15 秒）            | 返回 `Ok(true)`，记录 warn 日志（进程已启动但未通过健康检查） |
| 停止服务时 PID 为 0              | 静默返回 `Ok(())`                                             |
| `confirm_close_app` 销毁窗口失败 | 返回 `Err(e.to_string())`                                     |

### 7.3 窗口管理错误处理

| 错误场景                             | 处理方式                    |
| ------------------------------------ | --------------------------- |
| 创建新窗口失败                       | 记录 error 日志，不抛出异常 |
| 窗口关闭时非最后一个窗口             | 直接关闭，不拦截            |
| 窗口关闭时最后一个但非我们启动的服务 | 直接关闭，不询问            |

### 7.4 关键防护机制

**旧连接误清理防护**：`remove_if_current()` 和 `is_current()` 使用 conn_id 校验，确保只有当前活跃的连接才会被清理。当新的 `bridge_connect` 覆盖旧连接时，旧连接的 conn_id 不再匹配，其 cleanup 代码不会误删新连接。

**并发安全**：`BridgeState` 使用 `Mutex` 保护 HashMap 的所有操作，`ServiceState` 和 `OpenDirectoryState` 使用原子类型和无锁 HashMap，确保多线程安全。

---

## 8. Testing Considerations

### 8.1 Rust 侧单元测试建议

| 测试目标             | 测试内容                                                                                                    | 优先级 |
| -------------------- | ----------------------------------------------------------------------------------------------------------- | ------ |
| `BridgeState`        | next_conn_id 递增、replace 覆盖、remove_if_current ID 校验、disconnect_window 批量清理、is_current 状态检查 | P0     |
| `BridgeKey`          | 构造、window_label 提取、Hash/Eq 正确性                                                                     | P1     |
| `ConnectArgs`        | is_websocket() 对 ws/wss/http/https 的判断                                                                  | P0     |
| `OpenDirectoryState` | pending 插入、读取、remove                                                                                  | P1     |
| `ServiceState`       | 默认值、原子操作正确性                                                                                      | P1     |
| `BridgeEvent`        | serde 序列化输出格式（camelCase、tag/content）                                                              | P0     |

### 8.2 集成测试建议

| 测试场景          | 验证点                                                    | 优先级 |
| ----------------- | --------------------------------------------------------- | ------ |
| HTTP Stream 连接  | Connected 事件发出、Data 事件携带原始数据、超时触发 Error | P0     |
| WebSocket 连接    | 双向通信、Ping/Pong 自动响应、Close 帧处理                | P0     |
| 多窗口隔离        | 不同窗口的 bridge_id 互不干扰、窗口销毁自动清理           | P0     |
| 服务启动/停止     | 健康检查轮询、PID 记录、跨平台进程终止                    | P0     |
| CLI 目录参数      | 有效目录正确解析、无效参数忽略                            | P1     |
| 单实例模式        | 已运行时再次打开新建窗口而非新实例                        | P1     |
| macOS Opened 事件 | 冷启动设给 main、热启动新建窗口                           | P1     |

### 8.3 手动测试场景

| 场景       | 平台    | 步骤                                               |
| ---------- | ------- | -------------------------------------------------- |
| 安装包安装 | Windows | 运行 NSIS 安装程序，验证右键菜单注册               |
| 安装包安装 | macOS   | 挂载 DMG，拖拽安装，验证 Finder 右键和拖放         |
| 安装包安装 | Linux   | 安装 DEB 包，验证应用可启动                        |
| 开发模式   | 全部    | `npm run tauri dev`，验证 HMR 和 DevTools          |
| 生产构建   | 全部    | `npm run tauri build`，验证产物可运行              |
| 多窗口 SSE | 全部    | 打开两个窗口，各自建立 SSE 连接，验证事件不串扰    |
| 服务管理   | 全部    | 启动/停止 opencode serve，验证进程状态             |
| 关闭确认   | 全部    | 关闭最后一个窗口（服务由应用启动），验证确认对话框 |

### 8.4 性能关注点

| 关注点                 | 当前实现                             | 建议                                                                             |
| ---------------------- | ------------------------------------ | -------------------------------------------------------------------------------- |
| BridgeState 锁竞争     | Mutex 保护整个 HashMap               | 当前连接数有限（通常 < 10），Mutex 开销可接受                                    |
| HTTP Stream chunk 处理 | `String::from_utf8_lossy` 每个 chunk | 对于 UTF-8 多字节字符跨 chunk 截断，lossy 替换可能导致乱码。当前实现未做字节缓冲 |
| 窗口创建开销           | 每次新建窗口完整加载 WebView         | Tauri 2 的 WebView 创建开销较小，可接受                                          |
| Release 构建体积       | opt-level = "s" + LTO + strip        | 已优化，关注实际产物体积                                                         |

---

## 9. 与 Spec 的差异记录

| Spec 描述                             | 实际代码                                               | 差异类型   | 影响                              |
| ------------------------------------- | ------------------------------------------------------ | ---------- | --------------------------------- |
| 模块路径 `src/app/sse/`               | `src/app/bridge/`                                      | 架构演进   | 高：SSE 已合并为统一 Bridge       |
| 命令 `sse_connect` / `sse_disconnect` | `bridge_connect` / `bridge_send` / `bridge_disconnect` | 命名变更   | 高：前端调用接口已变              |
| SSE 事件 `Message { raw }`            | `Data { data }`                                        | 字段名变更 | 中：前端解析需适配                |
| SSE 状态使用 papaya                   | BridgeState 使用 Mutex<HashMap>                        | 实现差异   | 低：功能等价                      |
| LTO thin                              | `lto = true`（完整 LTO）                               | 配置差异   | 低：完整 LTO 优化更激进           |
| 未提及 opt-level                      | `opt-level = "s"`                                      | 补充细节   | 低：体积优化                      |
| 未提及 WebSocket 支持                 | tokio-tungstenite 完整实现                             | 功能扩展   | 高：Bridge 同时支持 PTY WebSocket |
| 未提及 `bridge_send` 命令             | 存在                                                   | 功能扩展   | 高：双向通信能力                  |
| 未提及 `ConnectArgs.is_websocket()`   | 存在                                                   | 实现细节   | 低：传输选择逻辑                  |
| 未提及 `/global/health` 端点          | 实际使用                                               | 实现细节   | 中：健康检查具体路径              |
| 未提及 `opt-level = "s"`              | 存在                                                   | 配置差异   | 低                                |
| 未提及 lib crate 类型                 | `cdylib`, `rlib`, `staticlib`                          | 补充细节   | 低：移动端支持                    |
| 未提及 `productName = "OpenCode"`     | 实际值                                                 | 配置差异   | 低                                |

---

## 10. 风险与缓解

| 风险                                        | 影响                                | 当前缓解措施                   | 建议                                       |
| ------------------------------------------- | ----------------------------------- | ------------------------------ | ------------------------------------------ |
| HTTP Stream 多字节字符跨 chunk 截断         | UTF-8 数据可能出现乱码              | `from_utf8_lossy` 替换无效字节 | 实现字节缓冲，累积完整 UTF-8 字符后再 emit |
| BridgeState Mutex 在高并发下成为瓶颈        | 多窗口频繁连接/断开时延迟增加       | 当前连接数有限，实际影响小     | 如未来需要更高并发，可考虑 papaya 或分片锁 |
| 服务健康检查超时仍返回 true                 | 前端误以为服务就绪                  | 记录 warn 日志                 | 考虑返回错误而非 true，让前端重试          |
| Windows NSIS 钩子使用 HKCU                  | 仅当前用户可用右键菜单              | 避免权限问题，适合个人使用     | 如需全用户安装，需 HKLM + 管理员权限       |
| `confirm_close_app` 调用 `window.destroy()` | 强制销毁窗口，不触发 CloseRequested | 由前端确认后调用，预期行为     | 确保前端只在用户明确确认后调用             |
| 移动端入口存在但功能裁剪                    | Android 仅保留 Bridge 命令          | 条件编译排除服务管理和目录关联 | 定期验证 Android 构建是否通过              |

---

## 11. 构建与分发

### 11.1 开发模式

```bash
npm run tauri dev
# 等价于：
#   1. 执行 beforeDevCommand: npm run dev（启动 Vite :5173）
#   2. Tauri 加载 devUrl: http://localhost:5173
#   3. 主窗口自动打开 DevTools（debug_assertions）
```

### 11.2 生产构建

```bash
npm run tauri build
# 等价于：
#   1. 执行 beforeBuildCommand: npm run build（Vite 生产构建到 ../dist）
#   2. cargo build --release（LTO + strip + opt-level=s）
#   3. 打包为 NSIS（Windows）/ DMG（macOS）/ DEB（Linux）
```

### 11.3 构建产物

| 平台    | 输出格式      | 输出目录                                |
| ------- | ------------- | --------------------------------------- |
| Windows | NSIS 安装程序 | `src-tauri/target/release/bundle/nsis/` |
| macOS   | DMG 磁盘镜像  | `src-tauri/target/release/bundle/dmg/`  |
| Linux   | DEB 软件包    | `src-tauri/target/release/bundle/deb/`  |

### 11.4 交叉构建限制

Tauri 不支持跨平台交叉编译。需要在目标平台上执行构建：

- macOS 上构建 macOS 版本
- Windows 上构建 Windows 版本
- Linux 上构建 Linux 版本

---

## 12. 依赖关系

### 12.1 外部依赖

| 依赖                     | 用途                                               | 必需性 |
| ------------------------ | -------------------------------------------------- | ------ |
| Rust 工具链（>= 1.85.0） | 编译 Rust 代码                                     | 必需   |
| Tauri CLI                | 开发和构建桌面应用                                 | 必需   |
| Node.js + npm            | 前端构建（beforeDevCommand / beforeBuildCommand）  | 必需   |
| 系统 WebView             | macOS: WebKit, Linux: WebKitGTK, Windows: WebView2 | 必需   |
| OpenCode 二进制          | 本地后端服务                                       | 可选   |

### 12.2 内部依赖

| 依赖模块                    | 使用内容                   |
| --------------------------- | -------------------------- |
| Web 前端（`../dist`）       | WebView 加载的静态资源     |
| `src/api/sdk.ts`            | Tauri 环境下注入原生 fetch |
| `src/utils/tauri.ts`        | 平台检测、MIME 映射        |
| `src/store/serviceStore.ts` | 前端服务管理状态           |

### 12.3 被依赖模块

| 模块                | 使用内容                                     |
| ------------------- | -------------------------------------------- |
| 001-api-layer       | Tauri 环境下使用 Bridge 替代浏览器 fetch/SSE |
| 002-chat-feature    | 桌面端通过 Bridge 建立 SSE 连接              |
| 005-settings-panel  | 服务管理 UI 调用 opencode 命令               |
| 008-terminal-system | 桌面端通过 Bridge 建立 WebSocket 连接        |

---

_本文档基于实际代码回溯生成，所有技术细节均来自 src-tauri/ 目录下的真实实现。_
