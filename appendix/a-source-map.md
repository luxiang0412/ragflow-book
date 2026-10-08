# 附录 A 源码地图：概念 → 路径

> RAGFlow 路径相对于仓库根目录，基于提交 `11e00194b`；mini-ragflow 路径相对于 `code/mini-ragflow/`，基于 `v1.0` 标签。带行号的条目都在写作时核对过；RAGFlow 是一个活跃的仓库，行号会漂移，请以符号名为准。

## 进程与入口

| 概念 | RAGFlow | mini-ragflow | 章 |
|---|---|---|---|
| 单二进制、多种模式 | `cmd/ragflow_server.go`（`--api/--ingestor/--syncer/--admin/--migrate`） | `backend/cmd/mini-ragflow/main.go`（`--mode api\|ingestor\|syncer\|all`） | 2、26 |
| 路由注册 | `internal/router/router.go` | `backend/internal/api/server.go` | 2 |
| 容器入口与进程编排 | `docker/entrypoint.sh` | `docker-compose.yml` | 26 |
| 一次性迁移 | `ragflow_server --migrate`（`cmd/ragflow_server.go:180`） | `backend/internal/store/store.go` 的 `Open` / `addColumn` | 26 |

## 数据模型与存储

| 概念 | RAGFlow | mini-ragflow | 章 |
|---|---|---|---|
| 知识库实体 | `internal/entity/dataset.go:99` `Knowledgebase` | `backend/internal/model/model.go` | 4 |
| 文档实体 | `internal/entity/document.go:22` `Document` | 同上 | 4 |
| 入库任务实体 | `internal/entity/ingestion_task.go:19` `IngestionTask` | 同上，`backend/internal/store/task.go` | 4、6 |
| 计数原子更新 | `internal/dao/document.go:90` `IncrementCounts`、`internal/dao/kb.go:451` | `backend/internal/store/document.go` | 4 |
| 共享表结构的迁移 | `internal/dao/migration.go` | `backend/internal/store/store.go` | 4 |
| 对象存储接口 | `internal/storage/types.go:59` `Storage` | `backend/internal/storage/` | 5 |
| 上传 | `internal/handler/document.go:982`、`internal/service/document/document_upload.go:29` | `backend/internal/api/documents.go`、`backend/internal/docs/docs.go` | 5 |

## 入库：任务、流水线、解析、切块、向量化

| 概念 | RAGFlow | mini-ragflow | 章 |
|---|---|---|---|
| 创建任务 | `internal/service/ingestion_task_service.go:105` `CreateForDocuments` | `backend/internal/store/task.go` `CreateTask` | 6 |
| 停止任务 | `internal/service/ingestion_task_service.go:298` `RequestStop` | `backend/internal/store/task.go` `RequestStop` | 6 |
| 消费循环 | `internal/ingestion/service/ingestion_service.go:302` `consumeLoop` | `backend/internal/ingest/worker.go` `loop` | 6、24 |
| 心跳 | `ingestion_service.go:53` `defaultHeartbeatInterval`、`heartbeat.go` | `backend/internal/store/task.go` `Heartbeat` | 6、24 |
| 关闭时留给重投递 | `ingestion_service.go:808` `runTask` | `backend/internal/store/task.go` `ReleaseTask` | 24 |
| 以数据库为准结算、接住 panic | `ingestion_service.go:964` `settleMessage` | `backend/internal/ingest/worker.go` `execute`、`safeRun` | 24 |
| NATS 消费者配置 | `internal/engine/nats/nats.go:278` | （任务表即队列） | 24 |
| 流水线执行 | `internal/ingestion/task/pipeline_executor.go:204` `Execute` | `backend/internal/pipeline/pipeline.go`、`backend/internal/ingest/runner.go` | 3、7 |
| 只统计原始块 | `pipeline_executor.go:475` `countOriginalChunkIDs` | `backend/internal/ingest/runner.go` | 6、20 |
| 流水线模板（DSL） | `internal/ingestion/pipeline/`（`go:embed` 的模板） | `backend/internal/pipeline/templates/*.json` | 7 |
| 组件参数清洗 | `internal/ingestion/pipeline/pipeline_params.go:158` `CleanComponentParams` | `backend/internal/pipeline/pipeline.go` `WithOverrides` | 7 |
| 组件注册 | `internal/agent/runtime/registry.go:204` `RegisterWithMeta` | `backend/internal/pipeline/components.go` | 7 |
| 解析器分派 | `internal/ingestion/component/parser_dispatch.go` | `backend/internal/parser/` | 8 |
| DeepDoc PDF | `internal/deepdoc/parser/pdf/parser.go`（`processPage :169`、`buildLayout :521`） | `backend/internal/parser/`（调用 `pdftotext`） | 8 |
| 通用切块 | `internal/ingestion/component/chunker/token.go:1063` `mergeUnits` | `backend/internal/chunker/general.go` | 9 |
| 按标题切块 | `chunker/title.go:139`、`chunker/group.go:66`、`chunker/hierarchy.go:103` | `backend/internal/chunker/title.go` | 9 |
| 问答 / 表格切块 | `chunker/qa.go`、`chunker/table.go` | `backend/internal/chunker/others.go` | 9 |
| 分词字段 | `internal/ingestion/component/tokenizer.go:912` `tokenizeChunks` | `backend/internal/pipeline/tokenizer.go` | 10 |
| 分词器 | `internal/tokenizer/tokenizer.go:453` `Tokenize` | `backend/internal/tokenize/` | 10 |
| 向量化与缓存 | `component/tokenizer.go:430` `embedChunks`、`internal/ingestion/chunkcache/chunkcache.go:98` | `backend/internal/pipeline/cache.go` | 10 |
| LLM 抽取 | `internal/ingestion/component/extractor.go`（`runEnableMetadata :1012`） | `backend/internal/pipeline/extractor.go` | 18 |
| 元数据 Schema | `internal/common/metadata_utils.go:391` `Turn2JSONSchema` | `backend/internal/pipeline/extractor.go` | 18 |
| RAPTOR | `internal/ingestion/component/knowledge_compiler/tree/`（`watershed.go:45`、`raptor.go:181`） | `backend/internal/pipeline/raptor.go` | 20 |
| 其他知识编译 | `knowledge_compiler/{mindmap,structure,wiki}/`、`internal/ingestion/knowledge_compile/` | — | 20 |

## 检索引擎与检索

| 概念 | RAGFlow | mini-ragflow | 章 |
|---|---|---|---|
| 检索引擎接口 | `internal/engine/engine.go:41` `DocEngine` | `backend/internal/docengine/engine.go` | 11 |
| ES 映射 | `conf/mapping.json` | `backend/internal/docengine/search.go`（`presenceField`） | 11 |
| 索引名 | `internal/service/nlp/retrieval.go:1287` `buildIndexNames` | （单库） | 11 |
| 查询构造 | `internal/service/nlp/query_builder.go:232` `Question` | `backend/internal/retrieval/query.go` | 12 |
| 词权重 | `internal/service/nlp/term_weight.go:311` `Weights` | `backend/internal/retrieval/query.go` `TermWeights` | 12 |
| 检索主流程与重试 | `internal/service/nlp/retrieval.go:94` `Retrieval`、`:645` `minMatch` | `backend/internal/retrieval/retrieval.go` | 12、13 |
| 融合表达式 | `retrieval.go:601` `buildInfinityFusionExpr` | `backend/internal/docengine/search.go` `Fusion` | 11、13 |
| 二次打分与重排 | `internal/service/nlp/reranker.go:913` `RerankWithKNN` | `backend/internal/retrieval/similarity.go` | 13 |
| 父子块 | `retrieval.go:1025` `RetrievalByChildren` | — | 13 |
| 检索测试接口 | `internal/handler/dataset.go:1204` | `backend/internal/api/retrieval.go` | 14 |
| 元数据过滤 | `internal/service/metadata_filter.go`（`:91` 哨兵、`:96` 模板） | `backend/internal/metadata/` | 19 |
| 标签 | `internal/service/tag.go:99` `LabelQuestion` | — | 19 |

## 问答

| 概念 | RAGFlow | mini-ragflow | 章 |
|---|---|---|---|
| 聊天入口 | `internal/handler/chat_session.go:160`、`internal/service/chat_session.go:1233` | `backend/internal/api/chats.go` | 15、16 |
| 聊天管线 | `internal/service/chat_pipeline.go:204` `AsyncChat` | `backend/internal/chat/pipeline.go` | 15 |
| 默认提示词 | `internal/service/chat.go:779` | `backend/internal/chat/prompt.go` | 15 |
| 知识块装填 | `internal/service/kb_prompt.go:61` `KbPrompt` | `backend/internal/chat/prompt.go` | 15 |
| 历史裁剪 | `chat_pipeline.go:2754` `messageFitIn` | `backend/internal/chat/prompt.go` `MessageFitIn` | 15 |
| 引用修复与回填 | `internal/service/citation.go`（`:68`、`:351`、`:455`）、`chat_pipeline.go:3056` | `backend/internal/chat/citation.go` | 16 |
| 前端读流 | `web/src/pages/next-chats/chat-stream/run-stream.ts:31` | `web/src/api.ts` `streamCompletion` | 17 |
| 前端引用渲染 | `web/src/components/markdown-content/index.tsx` | `web/src/pages/ChatView.tsx` | 17 |
| Agentic RAG（研究循环） | `internal/rag/agentic-rag/` | — | 21 |
| Agentic RAG（smart-reasoning） | `internal/agentic_rag/`、`conf/agentic_rag.yaml` | — | 21 |

## 平台化

| 概念 | RAGFlow | mini-ragflow | 章 |
|---|---|---|---|
| 注册与租户 | `internal/service/user.go:106` `Register` | `backend/internal/store/user.go` `CreateUser` | 22 |
| 团队关系 | `internal/entity/user_tenant.go:20` | — | 22 |
| 知识库访问控制 | `internal/dao/kb.go:344` `Accessible` | `backend/internal/api/datasets.go` `loadKB` | 22 |
| 认证中间件 | `internal/handler/auth.go:89`、`:136` | `backend/internal/api/auth.go` `requireAuth` | 22 |
| 令牌与 API Key | `internal/utility/token.go`、`internal/entity/api_token.go` | `backend/internal/auth/auth.go` | 22 |
| 密码 | `internal/common/password.go` | `backend/internal/auth/auth.go` | 22 |
| OpenAI 兼容 | `internal/service/openai_chat.go` | `backend/internal/api/openai.go` `openAIChat` | 23 |
| Dify 外部知识库 | `internal/handler/dify_retrieval_handler.go:165` | `backend/internal/api/openai.go` `difyRetrieval` | 23 |
| MCP | `internal/mcp/`（`server.go`、`tools.json`） | `backend/internal/api/openai.go` `mcp` | 23 |
| 限流 | `component/extractor.go:88` `defaultExtractorConcurrency` | `backend/internal/llm/llm.go` | 24 |
| 进程状态 | `--admin` 服务 | `backend/internal/store/worker.go`、`backend/internal/api/system.go` | 24、26 |
| 数据源同步 | `internal/syncer/`（先读 `README.md`） | `backend/internal/syncer/syncer.go` | 25 |
| 连接器接口 | `internal/syncer/connector/interface.go` | `backend/internal/syncer/syncer.go` `Connector` | 25 |
| 指纹 | `internal/syncer/connector/fingerprint.go:40` | `backend/internal/syncer/syncer.go` `fileSHA256` | 25 |

## 怎样在 RAGFlow 里找东西

1. 从路由开始：`internal/router/router.go` 里搜 URL 的一段，找到 handler；
2. handler 很薄，跟到 `internal/service/`；入库相关的在 `internal/ingestion/`；
3. 用 `grep -rn "func Name" internal/` 而不是全仓库搜索：仓库里还有前端、文档和遗留的 Python 代码，全仓库搜索又慢、结果又杂；
4. Go 代码里大量注释写着"mirrors Python …"，指向的 Python 文件很多已经删除了，看注释本身就够了；
5. 子系统的 `README.md`（比如 `internal/syncer/README.md`）通常比代码更快让你建立全局印象。
