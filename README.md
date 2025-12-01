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

## Kubernetes 部署示例

以下示例默认集群已存在可访问的 MySQL（服务名 `mysql-service`、端口 3306）与 Redis（`redis-service`、6379），并使用命名空间 `apo-sandbox`。如果集群中暂不存在 MySQL/Redis，可以先按下面的“补充数据库与缓存”小节创建：

### 补充数据库与缓存（集群无现成 MySQL/Redis 时）

1. 创建命名空间与 MySQL 初始化脚本 ConfigMap（复用仓库的 `mysql-init-script/sandbox.sql`）：
   ```bash
   kubectl create ns apo-sandbox
   kubectl -n apo-sandbox create configmap mysql-init --from-file=mysql-init-script/sandbox.sql
   ```

   如工作节点上尚未存在 `mysql:8.0` 与 `redis:7` 镜像，可依赖 Kubernetes 默认的镜像拉取流程（`imagePullPolicy: IfNotPresent`）在首次启动 Pod 时自动从镜像仓库下载；若集群离线或需要提前预热，可在各节点手工 `docker pull mysql:8.0 redis:7`，或临时创建一个预拉取 DaemonSet（拉取完成后可删除）：

   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: pre-pull-mysql-redis
     namespace: apo-sandbox
   spec:
     selector:
       matchLabels:
         app: pre-pull-mysql-redis
     template:
       metadata:
         labels:
           app: pre-pull-mysql-redis
       spec:
         containers:
           - name: mysql
             image: mysql:8.0
             command: ["sleep", "3600"]
           - name: redis
             image: redis:7
             command: ["sleep", "3600"]
   ```

   预拉取完毕后执行 `kubectl -n apo-sandbox delete ds pre-pull-mysql-redis` 即可回收。

2. 部署 MySQL（演示用 emptyDir，仅作示例，生产应改为 PVC）：
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: mysql-service
     namespace: apo-sandbox
   spec:
     ports:
       - port: 3306
     selector:
       app: mysql
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mysql
     namespace: apo-sandbox
   spec:
     selector:
       matchLabels:
         app: mysql
     template:
       metadata:
         labels:
           app: mysql
       spec:
         containers:
           - name: mysql
             image: mysql:8.0
             env:
               - name: MYSQL_ROOT_PASSWORD
                 value: "root"
               - name: MYSQL_DATABASE
                 value: "sandbox"
             ports:
               - containerPort: 3306
             volumeMounts:
               - name: data
                 mountPath: /var/lib/mysql
               - name: init-sql
                 mountPath: /docker-entrypoint-initdb.d
         volumes:
           - name: data
             emptyDir: {}
           - name: init-sql
             configMap:
               name: mysql-init
   ```

3. 部署 Redis（ClusterIP 暴露 6379）：
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: redis-service
     namespace: apo-sandbox
   spec:
     ports:
       - port: 6379
     selector:
       app: redis
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: redis
     namespace: apo-sandbox
   spec:
     selector:
       matchLabels:
         app: redis
     template:
       metadata:
         labels:
           app: redis
       spec:
         containers:
           - name: redis
             image: redis:7
             ports:
               - containerPort: 6379
   ```

   将 MySQL 与 Redis 的 YAML 片段保存为同一文件（如 `mysql-redis.yaml`），执行 `kubectl -n apo-sandbox apply -f mysql-redis.yaml` 即会在集群内拉取镜像并启动 Pod；可使用 `kubectl -n apo-sandbox rollout status deploy/mysql` 与 `kubectl -n apo-sandbox rollout status deploy/redis` 查看启动进度。

### 主业务组件部署

1. 创建命名空间与数据库/Redis 访问信息：
   ```bash
   kubectl create ns apo-sandbox
   kubectl -n apo-sandbox create secret generic db-secret \
     --from-literal=DB_USER="root" --from-literal=DB_PASSWORD="root"
   kubectl -n apo-sandbox create configmap db-config \
     --from-literal=DB_HOST="mysql-service" --from-literal=DB_PORT="3306" \
     --from-literal=REDIS_HOST="redis-service" --from-literal=REDIS_PORT="6379"
   ```

2. 部署任意语言的业务服务（以下以 Golang 为例，可替换镜像）：
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: go-service
     namespace: apo-sandbox
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: go-service
     template:
       metadata:
         labels:
           app: go-service
       spec:
         containers:
           - name: app
             image: apo-sandbox-go:latest
             imagePullPolicy: IfNotPresent
             env:
               - name: DB_HOST
                 valueFrom:
                   configMapKeyRef:
                     name: db-config
                     key: DB_HOST
               - name: DB_PORT
                 valueFrom:
                   configMapKeyRef:
                     name: db-config
                     key: DB_PORT
               - name: DB_USER
                 valueFrom:
                   secretKeyRef:
                     name: db-secret
                     key: DB_USER
               - name: DB_PASSWORD
                 valueFrom:
                   secretKeyRef:
                     name: db-secret
                     key: DB_PASSWORD
               - name: REDIS_HOST
                 valueFrom:
                   configMapKeyRef:
                     name: db-config
                     key: REDIS_HOST
               - name: REDIS_PORT
                 valueFrom:
                   configMapKeyRef:
                     name: db-config
                     key: REDIS_PORT
             ports:
               - containerPort: 3500
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: go-service
     namespace: apo-sandbox
   spec:
     selector:
       app: go-service
     ports:
       - port: 3500
         targetPort: 3500
   ```

3. 部署网关，指向目标服务（将 `TARGET_SERVICE_URL` 改成需要的语言服务地址）：
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: api-gateway
     namespace: apo-sandbox
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: api-gateway
     template:
       metadata:
         labels:
           app: api-gateway
       spec:
         containers:
           - name: gateway
             image: apo-sandbox-gateway:latest
             imagePullPolicy: IfNotPresent
             env:
               - name: TARGET_SERVICE_URL
                 value: "http://go-service:3500"
             ports:
               - containerPort: 8000
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: api-gateway
     namespace: apo-sandbox
   spec:
     type: ClusterIP
     selector:
       app: api-gateway
     ports:
       - port: 8000
         targetPort: 8000
   ```

4. 如需在集群内复用 Redis 延迟代理，可部署 `proxy/` 镜像并让应用将 `REDIS_HOST` 指向代理 Service：
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: redis-proxy
     namespace: apo-sandbox
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: redis-proxy
     template:
       metadata:
         labels:
           app: redis-proxy
       spec:
         containers:
           - name: proxy
             image: apo-sandbox-redis-proxy:latest
             env:
               - name: SERVER_ADDR
                 value: ":20000"
               - name: REDIS_ADDR
                 value: "redis-service:6379"
             ports:
               - containerPort: 20000
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: redis-proxy
     namespace: apo-sandbox
   spec:
     selector:
       app: redis-proxy
     ports:
       - port: 20000
         targetPort: 20000
   ```

5. 按需将网关通过 Ingress 或 `Service.type=LoadBalancer` 暴露给集群外部：
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: api-gateway
     namespace: apo-sandbox
     annotations:
       kubernetes.io/ingress.class: nginx
   spec:
     rules:
       - http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: api-gateway
                   port:
                     number: 8000
   ```

将以上 YAML 保存后执行 `kubectl apply -f <file>`，即可在集群中运行。可根据需要切换镜像或扩缩容，或通过 Helm/Kustomize 进行复用和参数化。
