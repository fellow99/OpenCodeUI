# 015-docker-deployment 技术方案（As-Built）

> 模块编号：015-docker-deployment
> 状态：已实现
> 编写日期：2026-04-12
> 依据：spec.md、constitution.md、overall-plan.md、实际源码

---

## 1. Technical Context

### 1.1 模块定位

本模块提供 OpenCodeUI 的容器化部署方案，将前端静态资源、后端 API 服务、统一网关和动态端口路由器打包为独立可部署单元。用户通过一条 `docker compose up -d` 命令即可启动完整的应用栈。

### 1.2 已实现内容

代码已完全实现，包含以下文件：

**Compose 编排文件（项目根目录）：**

| 文件                            | 用途                                          |
| ------------------------------- | --------------------------------------------- |
| `docker-compose.yml`            | 完整部署：gateway + frontend + backend        |
| `docker-compose.build.yml`      | 本地构建 override，替换预构建镜像为本地 build |
| `docker-compose.standalone.yml` | 独立前端模式：仅 frontend，连接外部后端       |

**Docker 配置（`docker/` 目录）：**

| 文件                      | 用途                                                              |
| ------------------------- | ----------------------------------------------------------------- |
| `Dockerfile.backend`      | 后端镜像：Ubuntu 24.04 + opencode + mise                          |
| `Dockerfile.frontend`     | 前端镜像：多阶段构建，node:22-alpine 构建 + caddy:2-alpine 运行时 |
| `Dockerfile.gateway`      | 网关镜像：rust:1.94-alpine 构建 router + caddy:2-alpine 运行时    |
| `Caddyfile`               | 前端默认 Caddy 配置（纯静态，API 返回 404）                       |
| `Caddyfile.standalone`    | 前端独立模式 Caddy 配置（静态 + API 反代）                        |
| `Caddyfile.gateway`       | 网关 Caddy 配置（双端口路由）                                     |
| `backend-entrypoint.sh`   | 后端容器入口脚本（工具链自愈）                                    |
| `entrypoint-gateway.sh`   | 网关容器入口脚本（双进程协调）                                    |
| `nginx.host.conf.example` | 外部 Nginx 反向代理参考配置                                       |

**Router 源码（`src-router/` 目录）：**

| 文件              | 用途                                   |
| ----------------- | -------------------------------------- |
| `Cargo.toml`      | Rust 依赖：axum、tokio、serde、rand 等 |
| `src/main.rs`     | Router 入口                            |
| `src/router.rs`   | 路由管理核心                           |
| `src/scanner.rs`  | 端口扫描器                             |
| `src/caddy.rs`    | Caddy 动态配置生成                     |
| `src/config.rs`   | 环境变量解析                           |
| `src/state.rs`    | 路由状态持久化                         |
| `src/router.html` | 内置管理面板                           |

**其他：**

| 文件           | 用途         |
| -------------- | ------------ |
| `.env.example` | 环境变量模板 |

### 1.3 与 Constitution 的对齐检查

| 宪法原则                | 对齐情况 | 说明                                                                         |
| ----------------------- | -------- | ---------------------------------------------------------------------------- |
| 原则 5：实时通信优先    | 符合     | Gateway 的 `flush_interval -1` 确保 SSE 不被缓冲，WebSocket 连接支持协议升级 |
| 原则 6：中文优先文档    | 符合     | 所有注释、文档均为中文                                                       |
| 原则 8：零配置用户体验  | 符合     | `docker compose up -d` 即可启动，环境变量均有默认值                          |
| 原则 10：模块化功能架构 | 符合     | 网关、前端、后端、路由器各自独立，通过明确接口通信                           |
| 约束 C5：SSE 支持       | 符合     | 所有代理路径均配置 `flush_interval -1`，不破坏 SSE 流式通信                  |

---

## 2. Research Findings

### 2.1 镜像构建策略

**Gateway 镜像（`Dockerfile.gateway`）：**

采用多阶段构建。第一阶段使用 `rust:1.94-alpine` 编译 Router 二进制文件，第二阶段使用 `caddy:2-alpine` 作为运行时，将编译好的 `opencodeui-router` 复制到 `/usr/local/bin/`。同时安装 `docker-cli` 用于动态端口发现。Caddy 配置使用 `Caddyfile.gateway`，并预创建 `routes.conf` 和 `preview.conf` 两个空文件供 Router 动态写入。

**Frontend 镜像（`Dockerfile.frontend`）：**

多阶段构建。第一阶段 `node:22-alpine` 执行 `npm ci` 和 `npm run build`，设置了 npm registry 自动回退机制（npmmirror 镜像）。第二阶段 `caddy:2-alpine` 托管构建产物 `/srv`。同时携带两份 Caddy 配置：默认的 `Caddyfile`（纯静态）和 `Caddyfile.standalone`（带 API 反代），通过 `command` 参数切换。

**Backend 镜像（`Dockerfile.backend`）：**

单阶段构建，基于 `ubuntu:24.04`。安装基础依赖后，通过 curl 安装 `mise` 和 `opencode` 二进制。设置 `MISE_DATA_DIR`、`MISE_CONFIG_DIR`、`MISE_STATE_DIR` 均指向 `/root` 下，确保数据卷挂载后持久化。入口脚本为 `backend-entrypoint.sh`。

### 2.2 网关双进程模型

`entrypoint-gateway.sh` 同时启动两个进程：

1. `opencodeui-router`（后台运行）：端口扫描 + 动态路由生成
2. `caddy run`（后台运行）：HTTP 反向代理 + 静态资源分发

两个进程通过 `trap cleanup TERM INT` 协调生命周期。脚本轮询检测任一进程退出，触发 `cleanup()` 终止另一个进程，确保同生共死。这对应 spec.md 中的 ADR-001 决策。

### 2.3 后端工具链自愈

`backend-entrypoint.sh` 实现三级回退策略：

1. **检查 PATH**：`command -v mise/opencode` 是否可用
2. **从卷恢复**：在 `/root/.local/bin/` 中查找已安装的副本，复制到 `/usr/local/bin/`
3. **网络安装**：curl 下载安装脚本重新安装

此机制确保容器重建后工具链自动恢复，对应 spec.md 中的 FR-05 和 ADR-002。

### 2.4 独立前端模式的 Linux 兼容

`docker-compose.standalone.yml` 中为 frontend 服务配置了 `extra_hosts: ['host.docker.internal:host-gateway']`。这是因为 `host.docker.internal` 在 Docker Desktop（macOS/Windows）中默认可用，但在 Linux 上需要显式映射。这使得前端容器在 Linux 环境下也能正确解析宿主机地址。

### 2.5 预构建镜像与本地构建

默认 `docker-compose.yml` 使用 GitHub Container Registry 的预构建镜像（`ghcr.io/lehhair/opencodeui-*:latest`）。`docker-compose.build.yml` 作为 override 文件，将 image 替换为本地 tag（`opencodeui-*:local`）并添加 `build` 指令。通过 `-f docker-compose.yml -f docker-compose.build.yml up -d --build` 实现本地构建。

### 2.6 Router 技术栈

Router 使用 Rust + Axum 框架，依赖包括：

- `axum 0.8.8`：HTTP 服务框架
- `tokio`：异步运行时（含 fs、process、signal、time 等特性）
- `serde + serde_json`：JSON 序列化（路由状态持久化）
- `rand`：随机令牌生成
- `base64`：编码工具
- `env_logger + log`：日志

Release profile 启用 `strip`、`lto = "thin"`、`panic = "abort"`、`codegen-units = 1`，优化二进制体积。

---

## 3. Data Model

### 3.1 容器拓扑

**完整部署模式（`docker-compose.yml`）：**

```
┌─────────────────────────────────────────────────────┐
│                  外部反向代理（可选）                  │
│              Nginx / Caddy + HTTPS                   │
└──────────────────────┬──────────────────────────────┘
                       │
        ┌──────────────▼──────────────┐
        │         Gateway              │
        │  ghcr.io/...gateway:latest   │
        │  ┌────────┬───────────────┐  │
        │  │ Caddy  │    Router     │  │
        │  │ :6658  │    :7070      │  │
        │  │ :6659  │               │  │
        │  └───┬────┴───────┬───────┘  │
        └──────┼────────────┼──────────┘
               │            │
    ┌──────────▼──┐  ┌─────▼──────────┐
    │  Frontend   │  │    Backend      │
    │  :3000（内） │  │   :4096（内）   │
    │  静态资源    │  │  opencode serve │
    └─────────────┘  └────────────────┘
```

**独立前端模式（`docker-compose.standalone.yml`）：**

```
┌──────────────────────────────────────┐
│          Frontend（独立）             │
│  ghcr.io/...frontend:latest          │
│  :3000 → 暴露到宿主机                 │
│  ┌─────────────┐                     │
│  │   Caddy     │                     │
│  │  静态 + 反代 │                     │
│  └──────┬──────┘                     │
└─────────┼────────────────────────────┘
          │ /api/* → strip prefix
          ▼
┌──────────────────────────────────────┐
│      外部后端（宿主机或远程）          │
│      host.docker.internal:4096       │
│      或 BACKEND_URL 指定的地址        │
└──────────────────────────────────────┘
```

### 3.2 网络拓扑

| 网络名称       | 类型                 | 成员                        | 说明                                            |
| -------------- | -------------------- | --------------------------- | ----------------------------------------------- |
| `opencode-net` | bridge（自定义命名） | gateway, frontend, backend  | 完整部署的内部网络，服务间通过 DNS 名称互相发现 |
| 默认 bridge    | bridge               | frontend（standalone 模式） | 独立模式仅一个服务，使用默认网络                |

**端口暴露策略：**

| 服务                   | 宿主机绑定                        | 容器端口 | 说明           |
| ---------------------- | --------------------------------- | -------- | -------------- |
| Gateway 主入口         | `127.0.0.1:${GATEWAY_PORT:-6658}` | 6658     | 仅绑定回环地址 |
| Gateway 预览入口       | `127.0.0.1:${PREVIEW_PORT:-6659}` | 6659     | 仅绑定回环地址 |
| Frontend（完整模式）   | 不暴露                            | 3000     | 仅对内网可见   |
| Backend                | 不暴露                            | 4096     | 仅对内网可见   |
| Frontend（standalone） | `0.0.0.0:${PORT:-3000}`           | 3000     | 直接暴露       |

### 3.3 数据卷

| 卷名                        | 类型               | 挂载目标               | 服务    | 内容                                                                                                              |
| --------------------------- | ------------------ | ---------------------- | ------- | ----------------------------------------------------------------------------------------------------------------- |
| `opencode-home`             | named volume       | `/root`                | backend | OpenCode 配置（`~/.config/opencode/`）、会话缓存、mise 工具链（`~/.local/share/mise/`）、npm/cargo/pip 用户态缓存 |
| `opencode-router-data`      | named volume       | `/data`                | gateway | 路由状态文件（`routes.json`）                                                                                     |
| `${WORKSPACE:-./workspace}` | bind mount         | `/workspace`           | backend | 用户工作区文件，宿主机与容器双向同步                                                                              |
| `/var/run/docker.sock`      | bind mount（只读） | `/var/run/docker.sock` | gateway | Docker API 访问，用于端口扫描发现容器内服务                                                                       |

### 3.4 服务依赖关系

```
gateway
  ├── depends_on: frontend
  └── depends_on: backend

frontend（完整模式）
  └── 无依赖

backend
  └── 无依赖

frontend（standalone 模式）
  └── extra_hosts: host.docker.internal → host-gateway
```

Gateway 的 `depends_on` 确保前端和后端容器先启动，但注意这仅保证容器创建顺序，不保证服务就绪。Caddy 的 `reverse_proxy` 在目标不可用时会返回 502，直到目标服务开始监听。

---

## 4. Interface Contracts

### 4.1 端口契约

**Gateway 主入口（:6658）路由表：**

| 路径模式     | 转发目标                 | 代理配置                                                   | 说明                    |
| ------------ | ------------------------ | ---------------------------------------------------------- | ----------------------- |
| `/api/*`     | `opencode-backend:4096`  | `handle_path`（自动剥离 `/api` 前缀），`flush_interval -1` | REST API + SSE 流式响应 |
| `/routes`    | `127.0.0.1:7070`         | `handle`                                                   | Router 管理面板         |
| `/preview/*` | `127.0.0.1:7070`         | `handle`                                                   | Router 预览端口管理 API |
| 其他         | `opencode-frontend:3000` | `handle`（兜底）                                           | 前端静态资源            |

**Gateway 预览入口（:6659）路由表：**

| 路径模式                 | 转发目标               | 说明                             |
| ------------------------ | ---------------------- | -------------------------------- |
| 动态生成的 `/p/{token}/` | 扫描到的容器内服务端口 | 由 Router 动态写入 `routes.conf` |
| 其他                     | `preview.conf` 兜底    | 独占预览端口的默认目标           |

**独立模式路由表（:3000）：**

| 路径模式 | 转发目标         | 代理配置                                                                             | 说明         |
| -------- | ---------------- | ------------------------------------------------------------------------------------ | ------------ |
| `/api/*` | `${BACKEND_URL}` | `handle_path`（剥离前缀），`flush_interval -1`，`header_up Host {upstream_hostport}` | 外部后端 API |
| 其他     | `/srv` 静态文件  | `try_files {path} /index.html`                                                       | SPA 路由支持 |

### 4.2 环境变量契约

**完整部署（`docker-compose.yml`）：**

| 变量名                     | 默认值        | 服务             | 必需性     | 说明                      |
| -------------------------- | ------------- | ---------------- | ---------- | ------------------------- |
| `ANTHROPIC_API_KEY`        | 空            | backend          | 至少填一个 | Anthropic 密钥            |
| `OPENAI_API_KEY`           | 空            | backend          | 至少填一个 | OpenAI 密钥               |
| `GEMINI_API_KEY`           | 空            | backend          | 可选       | Google Gemini 密钥        |
| `DEEPSEEK_API_KEY`         | 空            | backend          | 可选       | DeepSeek 密钥             |
| `GROQ_API_KEY`             | 空            | backend          | 可选       | Groq 密钥                 |
| `MISTRAL_API_KEY`          | 空            | backend          | 可选       | Mistral 密钥              |
| `XAI_API_KEY`              | 空            | backend          | 可选       | xAI 密钥                  |
| `GATEWAY_PORT`             | 6658          | gateway          | 可选       | 主入口端口                |
| `PREVIEW_PORT`             | 6659          | gateway          | 可选       | 预览入口端口              |
| `WORKSPACE`                | `./workspace` | backend          | 可选       | 宿主机工作目录路径        |
| `OPENCODE_SERVER_USERNAME` | `opencode`    | backend, gateway | 可选       | 基础认证用户名            |
| `OPENCODE_SERVER_PASSWORD` | 空            | backend, gateway | 公网必填   | 基础认证密码              |
| `PUBLIC_BASE_URL`          | 空            | gateway          | 可选       | 公网访问地址（Router 用） |
| `PREVIEW_DOMAIN`           | 空            | gateway          | 可选       | 预览服务独立域名          |
| `ROUTER_SCAN_INTERVAL`     | 5             | gateway          | 可选       | 端口扫描间隔（秒）        |
| `ROUTER_TOKEN_LENGTH`      | 12            | gateway          | 可选       | 预览令牌长度              |
| `ROUTER_PORT_RANGE`        | `3000-9999`   | gateway          | 可选       | 扫描端口范围              |
| `ROUTER_EXCLUDE_PORTS`     | `4096`        | gateway          | 可选       | 排除的端口列表            |

**Gateway 内部环境变量（不通过 .env 配置）：**

| 变量名              | 值                  | 说明                      |
| ------------------- | ------------------- | ------------------------- |
| `TARGET_CONTAINER`  | `opencode-backend`  | Router 扫描的目标容器名称 |
| `ROUTER_STATE_FILE` | `/data/routes.json` | 路由状态持久化路径        |

**独立模式（`docker-compose.standalone.yml`）：**

| 变量名        | 默认值                      | 说明                     |
| ------------- | --------------------------- | ------------------------ |
| `BACKEND_URL` | `host.docker.internal:4096` | 外部后端地址（不含协议） |
| `PORT`        | 3000                        | 前端暴露端口             |

### 4.3 镜像地址

| 服务     | 预构建镜像                                   | 本地构建 tag                |
| -------- | -------------------------------------------- | --------------------------- |
| Gateway  | `ghcr.io/lehhair/opencodeui-gateway:latest`  | `opencodeui-gateway:local`  |
| Frontend | `ghcr.io/lehhair/opencodeui-frontend:latest` | `opencodeui-frontend:local` |
| Backend  | `ghcr.io/lehhair/opencodeui-backend:latest`  | `opencodeui-backend:local`  |

### 4.4 Router 环境变量

Router 进程（`opencodeui-router`）读取以下环境变量：

| 变量名                 | 默认值      | 说明                 |
| ---------------------- | ----------- | -------------------- |
| `TARGET_CONTAINER`     | 必需        | 要扫描的目标容器名称 |
| `ROUTER_STATE_FILE`    | 必需        | 状态文件路径         |
| `ROUTER_SCAN_INTERVAL` | 5           | 扫描间隔（秒）       |
| `ROUTER_TOKEN_LENGTH`  | 12          | 令牌长度             |
| `ROUTER_PORT_RANGE`    | `3000-9999` | 端口范围             |
| `ROUTER_EXCLUDE_PORTS` | `4096`      | 排除端口             |
| `PUBLIC_BASE_URL`      | 空          | 公网地址             |
| `PREVIEW_DOMAIN`       | 空          | 预览域名             |
| `ROUTER_USERNAME`      | 空          | 管理面板认证用户名   |
| `ROUTER_PASSWORD`      | 空          | 管理面板认证密码     |

---

## 5. Implementation Strategy

### 5.1 部署流程

**完整部署：**

```
1. cp .env.example .env
2. 编辑 .env，至少填写一个 LLM API Key
3. docker compose up -d
4. 访问 http://localhost:6658
```

**本地构建部署：**

```
docker compose -f docker-compose.yml -f docker-compose.build.yml up -d --build
```

**独立前端部署：**

```
docker compose -f docker-compose.standalone.yml up -d
```

**连接远程后端：**

```
BACKEND_URL=remote-server.com:4096 PORT=8080 docker compose -f docker-compose.standalone.yml up -d
```

### 5.2 启动时序

```
t=0   docker compose up -d
  │
  ├─ frontend 容器启动（Caddy :3000，静态文件就绪）
  ├─ backend 容器启动
  │    ├─ backend-entrypoint.sh 执行
  │    │    ├─ ensure_mise() → 检查/恢复/安装
  │    │    └─ ensure_opencode() → 检查/恢复/安装
  │    └─ exec opencode serve --port 4096 --hostname 0.0.0.0
  │
  ├─ gateway 容器启动（depends_on 等待 frontend + backend 创建）
  │    ├─ entrypoint-gateway.sh 执行
  │    │    ├─ opencodeui-router & → 开始端口扫描
  │    │    └─ caddy run & → 开始监听 :6658 和 :6659
  │    └─ 轮询等待，任一进程退出则终止另一个
  │
  └─ 所有服务就绪
```

### 5.3 工具链固化流程

首次部署后，可通过以下命令固化运行时版本：

```
docker compose exec backend mise use -g node@22 python@3.12
```

这些安装会写入 `/root/.local/share/mise/installs/`，由于 `/root` 挂载了 `opencode-home` 卷，容器重建后数据保留。入口脚本的 `ensure_mise()` 和 `ensure_opencode()` 在每次启动时检查二进制可用性，如果 `/usr/local/bin/` 中缺失则从卷中恢复。

### 5.4 旧版本迁移

从旧版本（多卷架构）升级时：

- 旧卷 `opencode-data`、`opencode-config`、`opencode-cache`、`opencode-npm`、`opencode-cargo`、`opencode-local`、`opencode-opt` 变为孤立卷
- 新架构仅保留 `opencode-home` 一个核心卷
- 入口脚本不主动清理旧卷，需用户确认数据已迁移后手动删除
- `opencode-router-data` 卷保持不变

---

## 6. Error Handling

### 6.1 工具链安装失败

**场景**：容器重建后，`mise` 或 `opencode` 既不在 PATH 中，也不在卷中，且网络不可达。

**处理**：`backend-entrypoint.sh` 的 `ensure_mise()` 和 `ensure_opencode()` 在网络安装失败时执行 `exit 1`，容器启动失败。用户需要检查网络连接后重新启动容器。

**缓解**：只要卷数据未丢失，二级回退（从卷恢复）即可成功，不依赖网络。

### 6.2 网关目标服务不可用

**场景**：Gateway 启动时，Frontend 或 Backend 尚未开始监听端口。

**处理**：Caddy 的 `reverse_proxy` 在目标不可达时返回 502 Bad Gateway。由于 `depends_on` 仅保证容器创建顺序，不保证服务就绪，存在短暂窗口期。

**缓解**：Backend 的 `opencode serve` 启动较快（通常数秒），Frontend 的 Caddy 静态文件立即可用。用户刷新页面即可。

### 6.3 Router 进程异常退出

**场景**：`opencodeui-router` 因某种原因崩溃。

**处理**：`entrypoint-gateway.sh` 的轮询循环检测到进程退出，调用 `cleanup()` 终止 Caddy 进程，整个容器退出。Docker 的 `restart: unless-stopped` 策略会自动重启容器。

### 6.4 Docker 套接字不可用

**场景**：Gateway 容器无法访问 `/var/run/docker.sock`。

**处理**：Router 的端口扫描功能依赖 Docker API 检测容器内运行的服务。套接字不可用时，扫描失败，但 Caddy 代理功能不受影响。主应用（:6658）正常工作，仅预览功能（:6659）失效。

### 6.5 SSE 连接中断

**场景**：代理层缓冲导致 SSE 消息延迟或连接断开。

**处理**：Gateway 的 Caddy 配置中 `flush_interval -1` 禁用缓冲，确保 SSE 事件逐条转发。独立模式的 `Caddyfile.standalone` 同样配置 `flush_interval -1`。

### 6.6 独立模式 Linux 地址解析

**场景**：Linux 上 `host.docker.internal` 默认不可解析。

**处理**：`docker-compose.standalone.yml` 中配置 `extra_hosts: ['host.docker.internal:host-gateway']`，将 `host.docker.internal` 映射到 Docker 的网桥网关地址（通常是 `172.17.0.1`）。

---

## 7. Testing Considerations

### 7.1 部署验证场景

基于 spec.md 第 8 节验收场景，以下场景可用于验证部署正确性：

| 场景         | 验证内容         | 关键检查点                                                                           |
| ------------ | ---------------- | ------------------------------------------------------------------------------------ |
| 完整部署     | 四服务启动       | `docker compose ps` 全部 running，访问 `:6658` 返回前端页面                          |
| 独立前端部署 | 仅前端启动       | `docker compose -f standalone.yml ps` 仅 frontend running，访问 `:3000` 连接外部后端 |
| 连接远程后端 | 地址解析         | 设置 `BACKEND_URL` 后 API 请求正确转发                                               |
| 数据持久化   | 容器重启数据保留 | `compose down` + `compose up` 后会话和工具链存在                                     |
| 动态端口预览 | Router 自动发现  | 容器内启动 HTTP 服务后，`:6659` 生成令牌化路径                                       |
| SSE 流式响应 | 代理不缓冲       | AI 回复逐字到达，非批量返回                                                          |
| 安全认证     | 基础认证生效     | 设置密码后访问 `/routes` 返回 401                                                    |
| 工具链恢复   | 容器重建自愈     | 强制删除 backend 容器后重启，工具版本一致                                            |

### 7.2 网络连通性测试

```
# 验证服务间 DNS 解析
docker compose exec gateway ping opencode-frontend
docker compose exec gateway ping opencode-backend

# 验证端口监听
docker compose exec gateway curl -s http://opencode-frontend:3000
docker compose exec gateway curl -s http://opencode-backend:4096

# 验证网关路由
curl -s http://127.0.0.1:6658/          # 应返回前端 HTML
curl -s http://127.0.0.1:6658/api/health # 应返回后端响应
curl -s http://127.0.0.1:6658/routes    # 应返回 Router 管理面板
```

### 7.3 SSE 连接测试

```
# 验证 SSE 流式传输（不应等待完成后才返回）
curl -N http://127.0.0.1:6658/api/events
```

`-N` 参数禁用 curl 缓冲，可观察事件是否逐条到达。

### 7.4 安全测试

```
# 未设置密码时，/routes 应可直接访问
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:6658/routes

# 设置密码后，应返回 401
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:6658/routes

# 提供正确认证后，应返回 200
curl -s -o /dev/null -w "%{http_code}" -u "opencode:password" http://127.0.0.1:6658/routes
```

### 7.5 外部反向代理兼容性

spec.md 和 README.md 均提供了 Nginx 和 Caddy 的参考配置。关键验证点：

- Nginx `proxy_buffering off` 是否正确传递 SSE
- Caddy `flush_interval -1` 是否正确传递 SSE
- WebSocket 升级头（`Upgrade`、`Connection`）是否正确传递
- `proxy_read_timeout` 是否足够长（86400s）以支持长时间运行的任务

`nginx.host.conf.example` 提供了物理机 Nginx 的参考配置，覆盖前端、API、WebSocket 三个 location 块。

---

## 8. 风险与缓解

| 风险                                | 影响                       | 当前缓解措施                                  | 建议                                                            |
| ----------------------------------- | -------------------------- | --------------------------------------------- | --------------------------------------------------------------- |
| Docker 套接字只读挂载               | 容器内可读取宿主机容器信息 | 以 `:ro` 方式挂载，仅用于端口发现             | 考虑使用 Docker Socket Proxy 进一步限制权限                     |
| 工具链在线安装失败                  | Backend 容器启动失败       | 三级回退：PATH 检查 → 卷恢复 → 网络安装       | 在镜像中预装常用运行时版本                                      |
| Router 扫描间隔过长                 | 新服务延迟被发现           | 默认 5 秒，可通过 `ROUTER_SCAN_INTERVAL` 调整 | 支持事件驱动发现（Docker events）替代轮询                       |
| 预览令牌泄露                        | 未授权访问开发服务         | 默认 12 位随机令牌                            | 支持令牌过期和轮换机制                                          |
| 未设置密码即公网部署                | 服务暴露于未授权访问       | 文档警告，示例中标注必填                      | 启动时检查密码是否为空并输出警告日志                            |
| 旧版本多卷迁移数据丢失              | 升级后历史数据不可用       | 入口脚本不主动清理旧卷                        | 提供迁移脚本自动将旧卷数据合并到 `opencode-home`                |
| Gateway `depends_on` 不保证服务就绪 | 短暂 502 错误              | Caddy 自动重试                                | 考虑添加 healthcheck 和 `depends_on.condition: service_healthy` |

---

## 9. 与 016-router-service 的集成

本模块与 016-router-service 模块紧密集成：

- Router 源码位于 `src-router/`，由 `Dockerfile.gateway` 的第一阶段编译
- 编译产物 `opencodeui-router` 嵌入 Gateway 镜像
- Gateway 容器通过 `entrypoint-gateway.sh` 同时启动 Router 和 Caddy
- Router 通过 Docker API 扫描 `TARGET_CONTAINER`（`opencode-backend`）内运行的服务
- Router 通过 Caddy Admin API 动态更新 `routes.conf` 和 `preview.conf`
- 路由状态持久化到 `opencode-router-data` 卷的 `/data/routes.json`

Router 的 Caddy 集成方式：

1. Router 检测到新端口后，生成 Caddy 配置片段
2. 通过 Caddy Admin API（`localhost:2019/load`）动态加载配置
3. 配置写入 `/etc/caddy/routes.conf` 和 `/etc/caddy/preview.conf`
4. Caddy 的 `import` 指令在启动时读取这些文件，动态加载后通过 API 热更新

---

## 10. 总结

015-docker-deployment 模块已完整实现，提供两种部署模式：

1. **完整部署**：Gateway（Caddy + Router）+ Frontend + Backend，通过 `docker-compose.yml` 编排，一条命令启动
2. **独立前端**：仅 Frontend 容器，通过 `docker-compose.standalone.yml` 部署，连接已有后端

关键设计决策均已落地：网关双进程模型、后端工具链自愈、单一持久化卷、预览端口分离、127.0.0.1 绑定策略、SSE 无损代理。所有实现与 spec.md 中的功能需求和验收场景一一对应，与 constitution.md 中的原则和约束保持一致。
