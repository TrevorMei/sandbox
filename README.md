# APO Sandbox 部署概览

## 拓扑与端口
仓库中包含同一个业务 API 的多语言实现（具备故障注入能力），以及网关与支撑组件：

- 各语言服务默认在 **3500** 端口提供 `/api/users` 业务接口。
- FastAPI 网关监听 **8000** 端口，转发到指定的后端服务。
- Redis 故障代理默认监听 **20000** 端口，并转发到实际的 Redis 服务（默认 6379）。

## 组件说明

### Golang 服务（`Golang/`）
- 使用 Gorilla Mux 提供 `/api/users/{1,2,3}` 等接口，并带有请求日志中间件。
- 通过环境变量配置端口、超时、MySQL/Redis 连接参数；默认值包括 MySQL 主机 `mysql-service`、Redis 主机 `redis-service` 等。
- 当设置 `DEPLOY_PROXY=true` 时可启用 Redis 故障代理（如 Toxiproxy）。
- Dockerfile 生成静态二进制，安装 `iptables`/`iproute2` 以支持网络故障注入，并暴露 3500 端口。

### Java 服务（`Java/`）
- Maven 构建的 Spring Boot 应用，打包成 fat JAR。
- Dockerfile 采用多阶段构建，安装 `iproute2` 便于网络故障模拟，并内置 OpenTelemetry Java Agent。
- 容器通过 `java -javaagent ... -jar app.jar --server.port=3500` 启动并暴露 3500 端口。

### Node.js 服务（`node.js/`）
- 基于 Express，包含 CPU 压力、Redis 延迟和网络延迟等故障注入接口（使用示例见该目录下 `README.md`）。
- `config.js` 从环境变量读取 MySQL/Redis 主机、端口、凭据和故障参数。
- Dockerfile 安装生产依赖与 OpenTelemetry 自动探针，包含 `iproute2`，以 `npm run start:otel` 在 3500 端口启动。

### Python 服务（`python/`）
- Flask 应用，提供 `/api/users` 接口和请求日志中间件。
- `config.py` 读取端口、超时、MySQL/Redis 主机等环境变量；当 `DEPLOY_PROXY=true` 时可挂载 Redis 故障代理。
- Dockerfile 安装 `iproute2`、`procps`、`curl`，拉取 Python 自动探针，并在 3500 端口运行 `main.py`。

### API 网关（`gateway/`）
- FastAPI 网关，将 `/api/users` 流量转发到配置的目标服务（通过 `.env` 中的 `TARGET_SERVICE_URL` 设置）。
- 提供请求日志、宽松的 CORS、健康检查 `/health` 以及根信息接口。
- Dockerfile 以非 root 用户安装依赖，并使用 Uvicorn 在 8000 端口运行。

### Redis 延迟代理（`proxy/`）
- 轻量级 Go TCP 代理，接受 `FAULT.START <delay_ms>` 与 `FAULT.STOP` 指令，为上游 Redis 注入人工延迟。
- 监听 `SERVER_ADDR`（默认 `localhost:20000`），转发到 `REDIS_ADDR`（默认 `localhost:6379`），镜像暴露 20000 端口。

### MySQL 初始化脚本（`mysql-init-script/`）
- `sandbox.sql` 创建 `sandbox` 数据库与 `users` 表，并写入示例数据供各服务查询。

## Docker 构建速查
在各自目录执行镜像构建：

```bash
# 核心服务（端口 3500）
docker build -t apo-sandbox-go ./Golang
docker build -t apo-sandbox-java ./Java
docker build -t apo-sandbox-node ./node.js
docker build -t apo-sandbox-python ./python

# 支撑组件
docker build -t apo-sandbox-gateway ./gateway
docker build -t apo-sandbox-redis-proxy ./proxy
```

运行容器时，请补充 MySQL/Redis 连接所需的环境变量；网关需将 `TARGET_SERVICE_URL` 指向目标语言服务（例如 `http://go-service:3500`）。
