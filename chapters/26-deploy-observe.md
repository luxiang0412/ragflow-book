# 第 26 章 部署与可观测：Docker Compose 一键启动

> 本章目标：把 mini-ragflow 打包成镜像，用 Docker Compose 起 1 个 API、2 个 Ingestor、1 个 Syncer，并且能看出它们是不是都活着、在干什么。最后用一个脚本对部署做端到端验证。
> 前置：第 22~25 章。对应代码：`v1.0` 的 `Dockerfile`、`docker-compose.yml`、`web/e2e/deploy-e2e.mjs`。

## 26.1 RAGFlow 怎么部署

### 一个应用容器 + 一组基础设施

`docker/docker-compose.yml` 里只有一个应用服务 `ragflow-cpu`，基础设施在 `docker-compose-base.yml`：MySQL、MinIO、Kvrocks、NATS、ClickHouse，以及按需选择的检索引擎（Elasticsearch / OpenSearch / Infinity / OceanBase / SeekDB / SereneDB）。选哪些服务由 `.env` 里的 `COMPOSE_PROFILES` 控制：

```text
COMPOSE_PROFILES=${DOC_ENGINE},${DEVICE},metadata-${METADATA_DB_PROFILE},ragflow-go,clickhouse
```

`DOC_ENGINE` 默认是 `elasticsearch`，`DEVICE` 默认是 `cpu`。Kibana、Jaeger、TEI（本地 embedding 服务）、沙箱这些可选组件，也都是"往 `COMPOSE_PROFILES` 里加一个名字"就启用。

### 一个容器里的多个进程

应用容器的入口 `docker/entrypoint.sh` 根据环境变量，在**同一个容器**里启动多个 `ragflow_server` 进程：`--syncer`、`--admin`、`--api`（前面还有 nginx），以及 `WORKERS` 个 `--ingestor`。每个进程都包在一个"退出就 1 秒后重启"的循环里：

```bash
run_with_restart() {
  local process_name="$1"
  shift

  while true; do
    echo "Attempt to start ${process_name}..."
    set +e
    "$@"
    local exit_code=$?
    set -e
    echo "${process_name} exited with code ${exit_code}. Restarting in 1 second..."
    sleep 1
  done
}
```

这是一个务实的选择：单机部署只需要管理一个容器。代价是 Docker 看到的只有一个容器，某个进程反复崩溃重启，在 `docker ps` 里是看不出来的，只能看日志。要扩容 Ingestor，可以在另一台机器上起一个只开 `ENABLE_TASKEXECUTOR` 的容器，连同一套 MySQL/NATS。

### 迁移只跑一次

数据库迁移是一个单独的模式：`ragflow_server --migrate`（`cmd/ragflow_server.go:180`）只做迁移，然后退出。入口脚本在启动任何服务进程**之前**执行一次（`entrypoint.sh` 的 `run_go_migrations`），注释里说明了为什么："--migrate is a standalone action, so it is no longer attached to --api/--admin/--syncer"。如果每个进程启动时各自迁移，几个进程同时执行 `ALTER TABLE` 就会互相冲突。

### 可观测

- **日志**：每个进程写自己的日志，`run_with_restart` 会打印退出码；
- **追踪**：`jaeger` profile 提供一个 OTLP 收集器（4317/4318 端口）和 UI（16686 端口）。代码里的 OpenTelemetry 埋点主要在 `internal/harness/graph/pregel/`（Agent 运行时的图执行）；
- **管理端**：`--admin` 模式是单独的管理服务，第 22 章提到普通接口会拒绝超级管理员，管理功能都在这里。

## 26.2 从零实现

### 镜像：三阶段构建

```dockerfile
FROM node:22-alpine AS web
...
RUN npm run build

FROM golang:1.26-bookworm AS go
...
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/mini-ragflow ./cmd/mini-ragflow

FROM debian:bookworm-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends poppler-utils ca-certificates curl \
 && rm -rf /var/lib/apt/lists/* \
 && useradd -r -u 10001 app && mkdir -p /data /sync && chown app /data
COPY --from=go /out/mini-ragflow /usr/local/bin/mini-ragflow
COPY --from=web /web/dist /app/web
USER app
```

- 前端用 Node 构建，后端用 Go 构建，最终镜像里两者都不需要；
- SQLite 驱动是纯 Go 的（`modernc.org/sqlite`），所以可以 `CGO_ENABLED=0`。RAGFlow 的 Go 服务需要 CGO 和几个原生静态库（PDF、Office 解析），构建要复杂得多，这也是它的仓库里有 `build.sh` 的原因；
- 运行时只装了 `poppler-utils`（第 8 章的 `pdftotext`）和健康检查用的 `curl`；
- 不用 root 运行。

### Compose：按角色拆成服务

```yaml
services:
  api:
    <<: *common
    command: ["--mode", "api", "--data", "/data", "--static", "/app/web"]
    healthcheck:
      test: ["CMD", "curl", "-fs", "http://localhost:9380/healthz"]
  ingestor:
    <<: *common
    command: ["--mode", "ingestor", "--data", "/data", "--workers", "2"]
    deploy:
      replicas: 2
    # SIGTERM releases running tasks back to the queue; give them time.
    stop_grace_period: 20s
    depends_on:
      api:
        condition: service_healthy
  syncer:
    <<: *common
    command: ["--mode", "syncer", "--data", "/data"]
```

（节选。）和 RAGFlow 不同，mini-ragflow 每个角色一个服务，进程的重启交给 Docker（`restart: unless-stopped`），`docker compose ps` 能直接看到每个进程的状态。几个细节：

- **共享数据卷**：四个容器挂载同一个 `data` 卷，SQLite 数据库和上传的文件都在里面。SQLite 依赖文件锁，同一台机器的本地卷没有问题，**不要**把它放在 NFS 这类网络文件系统上，多机部署需要换成真正的数据库服务器。这是 mini-ragflow 和 RAGFlow 在部署上最大的差别；
- **`stop_grace_period: 20s`**：`docker compose stop` 或滚动更新时，Ingestor 收到 SIGTERM 后要把正在运行的任务归还给队列（第 24 章），给它留出时间；
- **`depends_on: service_healthy`**：等 API 的健康检查通过后，再启动 Ingestor 和 Syncer；
- **同步目录**：`./samples` 以只读方式挂载到 `/sync/samples`，`MR_SYNC_ROOT=/sync`（第 25 章）；
- **接真实模型**：在 `.env` 里设置 `MR_OPENAI_BASE_URL=http://host.docker.internal:11434/v1`，`extra_hosts` 让容器能访问宿主机上的 Ollama。

### 迁移

mini-ragflow 没有单独的迁移步骤：每个进程打开数据库时执行 `CREATE TABLE IF NOT EXISTS`，再补齐后来新增的列。四个容器几乎同时启动，两个进程可能同时发现"缺少某一列"并都去执行 `ALTER TABLE`，后执行的那个会报 `duplicate column name`。`addColumn` 把这个错误当作成功处理：

```go
	_, err := db.Exec(`ALTER TABLE ` + table + ` ADD COLUMN ` + name + ` ` + ddl)
	if err != nil && strings.Contains(err.Error(), "duplicate column name") {
		// Another process (a second ingestor container starting at the same
		// moment) added it between our check and our ALTER.
		return nil
	}
```

这是写 Compose 文件时预先做的处理，作者并没有实际观测到这个冲突。RAGFlow 的做法（单独执行一次 `--migrate`）更彻底：迁移逻辑一复杂（比如要改数据、建索引），"容忍并发"就不够用了。

### 可观测：进程心跳

第 24 章提到，每个 Ingestor 和 Syncer 每 5 秒往 `worker` 表写一次心跳。`GET /api/v1/system/status` 返回所有进程，以及当前租户各状态的任务数。超过 30 秒没有心跳的进程标记为离线。系统页（图 22-2）把这些显示成一张表。

## 26.3 运行与验证

```bash
cd code/mini-ragflow
docker compose up -d --build
node web/e2e/deploy-e2e.mjs http://localhost:9380
```

作者在完成全部修复后，用最终的 v1.0 重新部署了一次（使用另一个端口和项目名，避免和本机的其他服务冲突：`MR_PORT=9390 docker compose -p minirag up -d --build`）。容器状态：

```text
minirag-api-1	Up 8 seconds (healthy)
minirag-ingestor-1	Up 2 seconds
minirag-ingestor-2	Up 2 seconds
minirag-syncer-1	Up 2 seconds
```

`deploy-e2e.mjs` 走一遍完整的业务：注册 → 建知识库 → 建一个指向 `samples` 的文件夹连接器 → 等所有文档同步并解析完 → 检索 → 聊天 → 用 API Key 调 OpenAI 兼容接口 → 检查进程状态。真实输出：

```text
  ok  registered and logged in
  ok  folder connector on samples
  ok  7 documents synced and parsed in 3.6s: 星河咖啡机用户手册.md(7), 员工休假制度.txt(14), 星河S2咖啡机用户手册.md(4), formats_星河咖啡机用户手册.pdf(15), formats_售后问答.csv(1), formats_星河咖啡机用户手册.docx(7), faq.md(1)
  ok  retrieval top hit: formats_星河咖啡机用户手册.docx sim=0.485
  ok  chat answer with citation: 根据知识库：Q: 年假没休完可以折现吗 [ID:0]？A: 按制度应在次年 3 月 31 日前休完，逾期作废，不折现 […
  ok  OpenAI-compatible endpoint with an API key
  ok  processes alive: ingestor@627012f89309, ingestor@f3166f093d70, syncer@f046113c0f02; tasks {"COMPLETED":7}
PASS in 3.6s
```

（这次部署没有配置真实模型，用的是 mock。）

`docker compose logs` 里有一个值得注意的现象：7 个任务**全部**被 `ingestor-1` 认领了，`ingestor-2` 一个也没拿到：

```text
ingestor-1  | ... INFO task claimed task=0f747909… owner=ingestor-77961340/0 attempt=1
ingestor-1  | ... INFO task claimed task=a3cfed43… owner=ingestor-77961340/1 attempt=1
...（共 7 行，全部是 ingestor-77961340）
```

mock 模型下每个任务几十毫秒就完成了。Syncer 直接往任务表里写入任务，`ingestor-1` 的 worker 恰好先轮询到，处理得又比任务到来还快，`ingestor-2` 就没有机会。认领是"先到先得"，没有负载均衡；在任务耗时较长时（比如第 24 章用真实 LLM 做抽取），两个进程才会真正分摊任务。这和 NATS 的 pull 消费者行为类似：谁来拉，就给谁。

系统状态接口的输出（节选）：

```json
{
    "tasks": {},
    "workers": [
        {"id": "ingestor-77961340", "mode": "ingestor", "host": "627012f89309", "running": 0, "alive": true, ...},
        {"id": "ingestor-b97dd486", "mode": "ingestor", "host": "f3166f093d70", "running": 0, "alive": true, ...},
        {"id": "syncer-03a5da45", "mode": "syncer", "host": "f046113c0f02", "running": 0, "alive": true, ...}
    ]
}
```

这是用一个新注册的账号查询的：`tasks` 是空的，因为任务数按租户统计；`workers` 却是全局的，任何登录用户都能看到所有进程的 id 和容器主机名。对一个演示项目来说可以接受，对外的多租户部署应该把进程信息只开放给管理员。

## 26.4 练习

1. **巩固**：部署好之后执行 `docker compose up -d --scale ingestor=4`，在系统页确认出现 4 个 Ingestor。再执行 `docker compose stop ingestor`，观察正在解析的文档会怎样（第 24 章）。
2. **扩展**：仿照 RAGFlow，给 mini-ragflow 加一个 `--mode migrate`，在 Compose 里加一个只运行一次的 `migrate` 服务，让其他服务 `depends_on: {migrate: {condition: service_completed_successfully}}`，然后删掉 `addColumn` 里对并发的容忍。
3. **读源码**：RAGFlow 的 `entrypoint.sh` 在启动 Syncer 之后、启动 API 之前 `sleep 5`。读一读前后的代码，猜猜这 5 秒是为了什么；有没有比固定等待更可靠的办法？

## 26.5 延伸阅读

- `docker/docker-compose.yml`、`docker/docker-compose-base.yml`、`docker/entrypoint.sh`、`docker/.env`
- SQLite 文档：WAL 模式与网络文件系统（"WAL does not work over a network filesystem"）
- Docker 文档：Compose 的 `healthcheck`、`depends_on`、`stop_grace_period`
