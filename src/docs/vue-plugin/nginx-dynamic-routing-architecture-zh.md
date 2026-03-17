# Plugin Nginx 动态路由 — 架构设计与实现文档

## 1. 背景与动机

此前，plugin 前端通过**直连 IP + 端口**（`VITE_PLUGIN_API_URL:VITE_PLUGIN_API_PORT`）与各自的后端通信。该方案存在以下问题：

- 每个 plugin 后端需要一个独立的对外暴露端口（如 `8002:8082`）
- 前端在构建时必须知道后端的确切 IP 和端口
- Dashboard 与 plugin 后端之间存在跨域请求（CORS）问题
- 随着 plugin 数量增加，端口管理复杂度持续上升

新架构采用 **Nginx 反向代理**，将所有流量通过单一入口（80/443 端口）路由，每个 plugin 分配唯一的 URL 路径前缀。

## 2. 架构概览

```
浏览器
  │
  ├── /                        → Nginx 提供 mainapp SPA（index.html）
  ├── /api/...                 → Nginx 代理到 portal-backend:8000
  ├── /plugin/<name>/api/...   → Nginx 代理到 plugin 容器:<port>
  └── /plugin/<name>/ws/...    → Nginx 代理 WebSocket 到 plugin 容器:<port>
```

### 关键设计决定

| 决定 | 值 |
|------|-----|
| `<plugin-id>` | `expose_name`（来自 `PluginBuild.expose_name`，由 `unique_name()` 生成） |
| 路由前缀 | `/plugin/<expose_name>`（构建时作为 `VITE_PLUGIN_ROUTE_PREFIX` 注入） |
| Nginx 配置共享 | Docker 命名卷 `nginx_plugin_configs`，在 portal-backend 和 portal-frontend 之间共享 |
| 配置生成 | portal-backend 在部署时生成 `.conf` 文件 |
| 配置重载 | `docker exec portal-frontend nginx -s reload` |

### Docker Volume 架构

```
portal-backend 容器                portal-frontend (nginx) 容器
┌─────────────────────┐           ┌──────────────────────────────┐
│ /nginx-plugins-conf/│◄─────────►│ /etc/nginx/conf.d/plugins/   │
│   annotator.conf    │  共享卷   │   annotator.conf             │
│   other-plugin.conf │           │   other-plugin.conf          │
└─────────────────────┘           └──────────────────────────────┘
     （写入配置）                      （nginx 读取并生效）
```

`docker-compose.yml` 中的卷定义：
```yaml
volumes:
  nginx_plugin_configs:
    name: digitaltwins_nginx_plugin_configs
```

## 3. 请求流转

### 3.1 Mainapp 前端 API 请求（Phase 6）

```
浏览器 → GET /api/workflow-tools/...
       → Nginx（location /api/）
       → proxy_pass http://portal-backend:8000/api/
```

`http.ts` 设置 `axios.defaults.baseURL = "/api"` — 同源请求，无 CORS。

### 3.2 Plugin 前端 API 请求

```
浏览器 → GET /plugin/annotator/api/cases
       → Nginx（location /plugin/annotator/）
       → proxy_pass http://annotator-backend-app-1:8082/
```

`getBaseUrl.ts` 在 `__IS_PLUGIN__` 为 true 时返回 `${VITE_PLUGIN_ROUTE_PREFIX}/api`。

### 3.3 Plugin WebSocket 连接

```
浏览器 → ws://host/plugin/annotator/ws/<caseId>
       → Nginx（通过 $connection_upgrade 进行 WebSocket 升级）
       → proxy_pass ws://annotator-backend-app-1:8082/ws/<caseId>
```

### 3.4 MinIO 文件访问（未改动）

```
浏览器 → http://<PORTAL_BACKEND_HOST_IP>:<MINIO_PORT>/workflow-tools/...
       → 直连 MinIO（不经过 nginx）
```

MinIO URL 由 plugin 后端的 `rewrite_url_for_docker()` 重写为外部可访问的 host/port。

## 4. 实现细节

### 4.1 Phase 1：Plugin 前端 API Client 改造（`medical-image-annotator-dev`）

**新增文件：`annotator-frontend/src/plugins/api/getBaseUrl.ts`**

双模式 URL 解析：
- **Plugin 模式**（`__IS_PLUGIN__ = true`）：HTTP 和 WebSocket 均使用 `VITE_PLUGIN_ROUTE_PREFIX`
- **开发模式**（`__IS_PLUGIN__ = false`）：使用 `VITE_PLUGIN_API_URL` + `VITE_PLUGIN_API_PORT` 直连

```typescript
export function getApiBaseUrl(): string {
  if (__IS_PLUGIN__) {
    const routePrefix = import.meta.env.VITE_PLUGIN_ROUTE_PREFIX;
    return `${routePrefix}/api`;
  }
  const base_url = import.meta.env.VITE_PLUGIN_API_URL || "http://localhost";
  const port = import.meta.env.VITE_PLUGIN_API_PORT || "8082";
  return `${base_url}:${port}/api`;
}

export function getWsBaseUrl(): string {
  if (__IS_PLUGIN__) {
    const routePrefix = import.meta.env.VITE_PLUGIN_ROUTE_PREFIX;
    const protocol = location.protocol === "https:" ? "wss:" : "ws:";
    return `${protocol}//${location.host}${routePrefix}/ws`;
  }
  const base_url = import.meta.env.VITE_PLUGIN_API_URL || "http://localhost";
  const port = import.meta.env.VITE_PLUGIN_API_PORT || "8082";
  const { hostname } = new URL(base_url);
  return `ws://${hostname}:${port}/ws`;
}
```

**修改的文件：**
- `client.ts` — 使用 `getApiBaseUrl()` 替代硬编码的 URL
- `useWebSocketSync.ts` — 使用 `getWsBaseUrl()` 替代硬编码的 WS 地址
- `vite-env.d.ts` — 添加 `__IS_PLUGIN__` 和环境变量的 TypeScript 类型声明

### 4.2 Phase 2：Nginx 配置改造（`clinical-dashboard/frontend`）

**`nginx.conf` 新增内容：**
```nginx
# WebSocket 升级映射
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    # 代理 mainapp API 请求到 portal-backend
    location /api/ {
        proxy_pass http://portal-backend:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 动态加载的 plugin 路由配置（部署时生成）
    include /etc/nginx/conf.d/plugins/*.conf;
}
```

**`Dockerfile`** — 创建 `/etc/nginx/conf.d/plugins/` 目录
**`entry.sh`** — 启动时确保目录存在（volume 挂载场景）

### 4.3 Phase 3：Portal Backend 部署流程改造（`clinical-dashboard/backend`）

**`build_tool.py` — `_create_env_file()`**

构建时写入 `VITE_PLUGIN_ROUTE_PREFIX=/plugin/<expose_name>` 到 `.env`，并移除旧的 `VITE_PLUGIN_API_URL` / `VITE_PLUGIN_API_PORT`。

```python
@staticmethod
def _create_env_file(project_dir: Path, expose_name: str):
    config = {
        "VITE_PLUGIN_ROUTE_PREFIX": f"/plugin/{expose_name}"
    }
    # ... 移除 VITE_PLUGIN_API_URL 和 VITE_PLUGIN_API_PORT
```

**`deploy_tool.py` — Nginx 辅助方法**

在 `PluginDeployer` 类中新增三个静态方法：

| 方法 | 功能 |
|------|------|
| `generate_nginx_conf()` | 写入 `<expose_name>.conf`，包含 proxy_pass + WebSocket 请求头 |
| `remove_nginx_conf()` | 删除 `<expose_name>.conf` |
| `reload_nginx()` | 执行 `docker exec portal-frontend nginx -s reload` |

**生成的 nginx location 配置示例：**
```nginx
location /plugin/annotator/ {
    proxy_pass http://annotator-backend-app-1:8082/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 86400;
}
```

**`db_model.py` — 数据库表结构变更**

`PluginDeployment` 表新增四列：
- `route_prefix` — 如 `/plugin/annotator`
- `internal_host` — Docker 容器名
- `internal_port` — 容器内部端口
- `has_websocket` — 是否使用 WebSocket

**`workflow_tool_plugin.py` — 路由处理变更**

| 函数 | 变更 |
|------|------|
| `_parse_docker_compose_routing()` | 新增辅助函数：解析 plugin 的 `docker-compose.yml`，提取容器名和端口 |
| `get_plugin_deploy()` → `run_deploy()` | 部署成功后：生成 nginx 配置 → 重载 → 保存路由元数据到数据库 |
| `execute_plugin_backend_by_docker()` | `up`：重新生成配置 + 重载；`down`：移除配置 + 重载 |
| `delete_plugin()` | 删除该 plugin 所有的 nginx 配置 → 重载后再关停容器 |

### 4.4 Phase 4：Docker Compose 配置

**`docker-compose.yml` 变更：**

| 服务 | 变更 |
|------|------|
| portal-backend | 挂载 `nginx_plugin_configs:/nginx-plugins-conf` |
| portal-backend | 添加 `NGINX_PLUGINS_CONF_DIR=/nginx-plugins-conf` |
| portal-backend | 添加 `NGINX_CONTAINER_NAME=portal-frontend` |
| portal-frontend | 挂载 `nginx_plugin_configs:/etc/nginx/conf.d/plugins` |
| portal-frontend | 添加 `container_name: portal-frontend` |

### 4.5 Phase 5：验证与测试

- Dev 模式测试：`yarn dev` + `.env` 仍然可以直连 plugin 后端 ✅
- Plugin Build 测试：`yarn build:plugin` 产物使用 `VITE_PLUGIN_ROUTE_PREFIX` ✅
- Deploy 测试：部署 plugin 后端后 nginx 配置自动生成 ✅
- 端到端测试：通过 nginx 代理访问 plugin HTTP API ✅
- WebSocket 测试：通过 nginx 代理建立 WS 连接 ✅

### 4.6 Phase 6：Mainapp 前端也走 Nginx 代理

**`http.ts`** — 从异步 IIFE 初始化简化为同步初始化：
```typescript
// 改造前：从 runtime-config.json 获取 IP:PORT，拼接 URL
// 改造后：
axios.defaults.baseURL = "/api";
```

**删除的文件：**
- `runtime.ts` — 不再需要（仅被 `http.ts` 使用）

**`entry.sh`** — 移除 `runtime-config.json` 生成逻辑

**`docker-compose.yml`** — 从 portal-frontend 移除：
- `PORTAL_BACKEND_HOST_IP`
- `BACKEND_PORT`
- `SSL`
- `KEYCLOAK_*`（nginx/entry.sh 中未使用；前端使用编译时的 `VITE_KEYCLOAK_*`）

从 portal-backend 移除：
- `PLUGIN_PORT=8002`（代码中无任何引用）

## 5. 环境变量汇总

### portal-backend（保留）

| 变量 | 用途 |
|------|------|
| `PORTAL_BACKEND_HOST_IP` | 用于 CORS 来源白名单和 MinIO URL 重写 |
| `USE_SSL` | SSL 协议检测（http/https） |
| `MINIO_PORT` | MinIO 外部端口，用于 URL 重写 |
| `NGINX_PLUGINS_CONF_DIR` | plugin nginx 配置文件的写入路径 |
| `NGINX_CONTAINER_NAME` | 用于 `docker exec nginx -s reload` 的容器名 |

### portal-frontend（所有环境变量已移除）

Nginx 配置现在完全静态化。Plugin 配置通过共享卷动态加载。

### Plugin 前端（构建时 .env）

| 变量 | 模式 | 用途 |
|------|------|------|
| `VITE_PLUGIN_ROUTE_PREFIX` | Plugin 构建 | 如 `/plugin/annotator` |
| `VITE_PLUGIN_API_URL` | 仅开发模式 | 如 `http://localhost` |
| `VITE_PLUGIN_API_PORT` | 仅开发模式 | 如 `8082` |

## 6. 调试接口

新增了一个调试接口用于验证 nginx 配置：

```
GET /api/workflow-tools/debug/nginx-config
```

返回内容：
- `plugin_configs` — 共享卷中所有 `.conf` 文件的内容
- `nginx_main_conf` — portal-frontend 容器中的主 `nginx.conf`
- `nginx_plugins_in_container` — nginx 容器中 plugin 配置的目录列表
- `nginx_test` — `nginx -t` 配置校验结果

## 7. 开发模式兼容性

所有改动对本地开发保持向后兼容：

- `yarn dev` + `.env`（含 `VITE_PLUGIN_API_URL/PORT`）仍可直连后端
- 开发模式下 `__IS_PLUGIN__` 为 `false`，`getBaseUrl.ts` 使用直连 URL 路径
- Mainapp 前端本地开发仍可通过 Vite 开发服务器代理或直连后端

## 8. 修改文件清单

### `clinical-dashboard/`（mainapp）

| 文件 | 操作 | 阶段 |
|------|------|------|
| `frontend/nginx.conf` | 修改 | Phase 2 |
| `frontend/Dockerfile` | 修改 | Phase 2 |
| `frontend/entry.sh` | 修改 | Phase 2, 6 |
| `frontend/src/plugins/http.ts` | 修改 | Phase 6 |
| `frontend/src/plugins/runtime.ts` | 删除 | Phase 6 |
| `backend/app/builder/build_tool.py` | 修改 | Phase 3 |
| `backend/app/builder/deploy_tool.py` | 修改 | Phase 3 |
| `backend/app/models/db_model.py` | 修改 | Phase 3 |
| `backend/app/router/workflow_tool_plugin.py` | 修改 | Phase 3 |
| `backend/pyproject.toml` | 修改 | Phase 3 |
| `docker-compose.yml` | 修改 | Phase 4, 6 |
| `.env.template` | 修改 | Phase 6 |

### `medical-image-annotator-dev/`（plugin）

| 文件 | 操作 | 阶段 |
|------|------|------|
| `annotator-frontend/src/plugins/api/getBaseUrl.ts` | 新增 | Phase 1 |
| `annotator-frontend/src/plugins/api/client.ts` | 修改 | Phase 1 |
| `annotator-frontend/src/composables/right-panel/useWebSocketSync.ts` | 修改 | Phase 1 |
| `annotator-frontend/src/vite-env.d.ts` | 修改 | Phase 1 |
