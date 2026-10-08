# Lab 6：mini-ragflow v1.0——多进程部署、对外 API、端到端验证

> 目标：把 v0.5 变成一个可以部署给别人用的小平台：账号与 API Key、OpenAI/Dify/MCP 三种对外协议、API/Ingestor/Syncer 三种进程角色、文件夹同步、Docker Compose 一键部署。
> 前置：第 22~26 章。

## 1. 这一版改了什么

```bash
git diff v0.5 v1.0 --stat | tail -1
git log --oneline v0.5..v1.0
```

```text
 56 files changed, 3004 insertions(+), 179 deletions(-)
2f4e6f7 Folder connector: do not echo the server path for a missing folder
87a8694 Folder sync: stream fingerprints, load content lazily, never prune an unreadable file
bb44bcc Adopted chats: move the tenant inside the JSON body too; read the tenant from the column
d820c51 Dify retrieval: authenticate in-route and use Dify's 1001/1002 error codes
1bbde78 Normalize CRLF in the embedded meta_filter prompt; .gitattributes keeps LF; fix extractor prompt comment
dc646b4 docengine: score *_tks fields like RAGFlow's scripted_sim (presence, no length norm)
f1697e0 v1.0: accounts and API keys, OpenAI/Dify/MCP endpoints, api/ingestor/syncer modes, folder sync, Docker Compose
```

第一个提交是 v1.0 本身，后面 6 个都是**写书时**对照 RAGFlow 源码或者重新运行时发现的问题（分别在第 11、19、23、25 章讲过），`v1.0` 标签随之移动到最新的修复上。如果你在本书完成之前检出过 v1.0，请重新拉取标签：`git fetch --tags --force`。

| 新增/改动 | 文件 | 章节 |
|---|---|---|
| 密码哈希、令牌 | `internal/auth/` | 22 |
| 用户、会话令牌、API Key；第一个用户过继旧数据 | `internal/store/user.go` | 22 |
| 认证中间件、注册/登录/退出、API Key 管理 | `internal/api/auth.go` | 22 |
| OpenAI 兼容、Dify 外部知识库、MCP | `internal/api/openai.go` | 23 |
| 任务归还（`ReleaseTask`）、毒消息（`FailExhausted`） | `internal/store/task.go`、`internal/ingest/worker.go` | 24 |
| 进程心跳表、系统状态接口 | `internal/store/worker.go`、`internal/api/system.go` | 24、26 |
| 模型调用限流 | `internal/llm/llm.go` | 24 |
| 连接器、文件夹同步 | `internal/store/connector.go`、`internal/syncer/` | 25 |
| 文档的创建/替换/删除抽成服务，API 和 Syncer 共用 | `internal/docs/` | 25 |
| `--mode api\|ingestor\|syncer\|all` 等启动参数 | `cmd/mini-ragflow/main.go` | 26 |
| 镜像、Compose、部署验证脚本 | `Dockerfile`、`docker-compose.yml`、`web/e2e/deploy-e2e.mjs` | 26 |
| 前端：登录页、数据源页、系统页；所有请求带令牌 | `web/src/` | 22、25、26 |

## 2. 接口变化

| 方法 | 路径 | 认证 | 说明 |
|---|---|---|---|
| POST | `/api/v1/user/register`、`/user/login` | 无 | 返回 `access_token` |
| POST | `/api/v1/user/logout`；GET `/user/info` | 令牌 | |
| GET/POST/DELETE | `/api/v1/system/tokens[/{id}]` | 令牌 | API Key，明文只在创建时返回一次 |
| GET | `/api/v1/system/status` | 令牌 | 进程心跳 + 本租户的任务数 |
| GET/POST/DELETE | `/api/v1/connectors[/{id}]`；POST `/connectors/{id}/sync` | 令牌 | 文件夹连接器 |
| POST | `/api/v1/openai/{chat_id}/chat/completions` | 令牌 | OpenAI Chat Completions |
| POST | `/api/v1/dify/retrieval` | 令牌（路由自己认证） | Dify 外部知识库，错误码 1001/1002/2001 |
| POST | `/api/v1/mcp` | 令牌 | MCP（JSON-RPC 2.0，Streamable HTTP） |
| GET | `/healthz` | 无 | 健康检查 |

表中的"令牌"指登录得到的会话令牌或 API Key，两者都可以。其余 `/api/v1/*` 接口不变，但都需要 `Authorization: Bearer <令牌或 API Key>`，并且只能访问自己租户的数据。

## 3. 跑起来

本地，一个进程三合一：

```bash
git checkout v1.0
cd backend && go test ./...
cd ../web && npm run build
cd ../backend && go run ./cmd/mini-ragflow --static ../web/dist --sync-root /tmp/nas
```

本地，多进程（第 24 章的实验就是这样起的）：

```bash
mini-ragflow --mode api      --data ./data --static ../web/dist --sync-root /tmp/nas
mini-ragflow --mode ingestor --data ./data --workers 2
mini-ragflow --mode ingestor --data ./data --workers 2
mini-ragflow --mode syncer   --data ./data --sync-root /tmp/nas
```

Docker Compose：

```bash
docker compose up -d --build
node web/e2e/deploy-e2e.mjs http://localhost:9380
```

从 v0.5 升级：直接用 v1.0 打开原来的 `--data` 目录，第一个注册的用户会得到原来所有的知识库和助手（`adopted_default_data: true`）。

## 4. 检查点

- [ ] `go test ./...` 全部通过（作者的最终一次运行：14 个有测试的包全部 `ok`）；
- [ ] `TestAuthAndTenantIsolation`、`TestFirstUserAdoptsDefaultData`：租户隔离、两种登录错误同一条消息、升级后过继的助手能打开；
- [ ] `TestExternalAPIs`：OpenAI（非流式带引用、流式无引用）、Dify（含 1001/1002/2001）、MCP（initialize、通知、tools/list、tools/call）；
- [ ] `TestShutdownRequeuesRunningTask`、`TestPoisonTaskStopsAfterMaxAttempts`、`TestPanicFailsTaskNotProcess`；
- [ ] `TestFolderSyncLifecycle`、`TestFolderMustStayInsideRoot`、`TestUnreadableFileIsNotPruned`；
- [ ] 两个 Ingestor 进程时，`kill -9` 正在运行任务的那个，任务在一个租约周期内被另一个接手并完成（第 24 章：4.5 秒接手，26.6 秒完成）；
- [ ] `kill -TERM` 时任务立即回到 `CREATED`（第 24 章：1.0 秒被接手）；
- [ ] Compose 部署后 `deploy-e2e.mjs` 输出 `PASS`（第 26 章：7 个文档 3.6 秒同步并解析完）；
- [ ] UI 脚本 `ui-v10.mjs`：登录、建连接器并同步、建 API Key、系统页显示进程。

![图 L6-1：v1.0 的系统页。](./images/v10-system.png)

*图 L6-1：系统页上的 API Key 和进程列表（UI 脚本 `ui-v10.mjs` 截图）。*

## 5. 本版的踩坑汇总

| 现象 | 原因 | 处理 | 章节 |
|---|---|---|---|
| SIGTERM 后任务被标成 FAILED | 任务在进程上下文下运行 | 独立上下文 + `ReleaseTask` | 24 |
| 杀进程的脚本把自己的 shell 也杀了 | `pgrep -f` 匹配到了自己 | 用 `/proc/<pid>/cmdline` 精确匹配 | 24 |
| `*_tks` 字段的打分和 RAGFlow 不一致 | 全部用了 BM25 | 只看"出现与否"，不做长度归一 | 11 |
| 检出 v0.5 后两个测试失败 | `core.autocrlf=true` | 加载时统一换行 + `.gitattributes` | 19 |
| Dify 认证失败时返回 401 和 code 109 | 走了通用的 `requireAuth` | 路由自己认证，403 + 1001/1002 | 23 |
| 升级后的助手列得出来、打不开 | 租户存了两份，只改了一份 | `json_set` + 读取时以列为准 | 23 |
| 读不了的文件，对应文档被删了 | "没列出来"被当成"已删除" | 列出但指纹为空，记为失败 | 25 |
| 同步时整个目录的内容都在内存里 | `List` 预读了所有文件（自查） | 流式哈希 + 懒加载 | 25 |
| 错误信息里带着服务器的绝对路径 | 直接返回了 `EvalSymlinks` 的错误 | 返回通用的错误信息 | 25 |
| 并发启动时的 `duplicate column name` | 多个进程同时迁移（预防，未观测到） | 容忍该错误 | 26 |

## 6. 还缺什么

和 RAGFlow 相比，v1.0 仍然是一个教学用的"骨架"。最明显的几处差距：

- **存储**：SQLite + 本地文件，只能单机部署；RAGFlow 是 MySQL + MinIO + Elasticsearch/Infinity，可以水平扩展；
- **解析**：没有 DeepDoc 的版面分析、表格识别和 OCR（第 8 章）；
- **团队与权限**：一个用户就是一个租户，没有团队共享（第 22 章练习 2）；
- **Agentic RAG、知识图谱、Agent 画布**：只读源码，没有实现（第 21 章）；
- **连接器**：只有一个文件夹连接器，没有增量窗口和断点续传（第 25 章）；
- **可观测**：只有进程心跳和日志，没有追踪和指标。

附录 C 有一张更完整的对照表。
