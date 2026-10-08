# 《从 RAGFlow 到 Mini-RAGFlow：从零构建知识库 RAG 引擎》大纲

> **状态（2026-10-08）**：全部 27 章、6 个 Lab、4 个附录已完成；配套代码 `code/mini-ragflow` 的 v0.1 ~ v1.0 标签在成书时逐个重新检出并通过了 `go test ./...`（v0.5 需以 `core.autocrlf=false` 检出，见第 19 章）。目录见 [README.md](./README.md)。

> 参照仓库：`infiniflow/ragflow`（main @ `11e00194b`），以 Go 实现（`internal/`）为主；Python 旧实现（`rag/`、`deepdoc/`、`api/`）只在对照时提及。
> 读者目标：读完后能从零写出一个前后端可运行的知识库 RAG 系统（mini-ragflow）：上传文档 → 解析 → 切分 → 向量化 → 建索引 → 混合检索与重排 → 带引用的流式问答。
> 写作方式：每章都按“概念 → RAGFlow 源码导读（`path:line`）→ 从零实现 → 运行验证 → 练习”的顺序展开。

## 贯穿全书的项目：mini-ragflow

| 版本 | 能力 | 对应 Part |
|---|---|---|
| v0.1 | 知识库与文档 CRUD、文件存储、异步解析任务（状态机 + worker 池）、纯文本分块、前端文档列表和进度 | Part II |
| v0.2 | 入库流水线 DSL（File → Parser → Chunker → Extractor → Tokenizer）、多种解析器、多种切分策略、分词 + 向量化 | Part III |
| v0.3 | 自己写的文档引擎：倒排索引（BM25 + 字段权重）、向量检索、混合融合、RAGFlow 式重排、检索测试页 | Part IV |
| v0.4 | 知识库问答：kb prompt、SSE 流式、`[ID:n]` 引用与向量回填引用、会话、引用悬浮 UI | Part V |
| v0.5 | LLM 增强（关键词 / 问题 / 摘要 / 元数据）、元数据过滤、RAPTOR 递归摘要树 | Part VI |
| v1.0 | 多租户与 API Key、对外检索 API（REST / OpenAI 兼容 / Dify 外部知识库）、独立 ingestor 进程（租约 + 心跳 + 重投递）、Docker Compose | Part VII |

技术栈（和 RAGFlow 概念对齐，复杂度降一档）：
- 后端：Go 1.2x，标准库 `net/http` 路由，SQLite（纯 Go 驱动，免 CGO），本地磁盘代替 MinIO，SQLite 任务表代替 NATS JetStream，自研内存 + SQLite 持久化索引代替 ES / Infinity
- 前端：React + Vite + TypeScript
- 模型：内置确定性 mock（哈希 embedding + 会流式输出、会引用的假 LLM），离线也能跑通全部测试；同时支持任意 OpenAI 兼容接口，并用本机 Ollama（`nomic-embed-text` + `qwen2.5:0.5b`）实测一遍

---

## Part 0 导读
### 第 0 章 本书怎么读
- RAG 的四段：入库、索引、检索、生成；RAGFlow 的口号“Quality in, quality out”
- mini-ragflow 的最终效果和代码结构；环境准备（Go、Node、可选 Ollama / Docker）
- 怎么对照着读 RAGFlow 源码（附录 A 是源码地图）

## Part I 全景（浅）
### 第 1 章 RAG 要解决什么问题，RAGFlow 押注在哪
- naive RAG 的三类失败：切坏了、找不到、答不准
- RAGFlow 的三个押注：深度文档理解（DeepDoc）、模板化切分、可溯源引用
### 第 2 章 RAGFlow 架构鸟瞰
- 一个二进制、四种模式：`cmd/ragflow_server.go` 的 `--api / --ingestor / --syncer / --admin`
- 运行时拓扑：MySQL、MinIO、NATS JetStream、Kvrocks、ES / Infinity / OceanBase（`docker/docker-compose-base.yml`）
- 目录地图：`router → handler → service → dao/engine`、`ingestion/`、`parser/`、`deepdoc/`、`web/`
### 第 3 章 一份文档的一生：从上传到被引用 ★
- 上传 `handler/document.go: UploadDocuments` → 解析 `StartIngestionTask` → `IngestionTaskService.CreateForDocuments` → NATS
- `ingestion/service: Ingestor.consumeLoop / handleAndExecute` → `task/pipeline_executor.go` → 组件图 → `chunkIndexWriter` → DocEngine
- 提问 `chat/completions` → `ChatPipelineService.AsyncChat` → `nlp.RetrievalService.Retrieval` → `KbPrompt` → LLM → `InsertCitations`
- 产出：一张时序图，后面所有章节都在拆它

## Part II 地基：知识库、文件与任务 → v0.1
### 第 4 章 数据模型：Tenant / Knowledgebase / Document / Chunk
- `entity/dataset.go`、`entity/document.go`；`parser_id` 和 `parser_config`；计数字段怎么维护
- chunk 不放 MySQL，放在检索引擎里：为什么这样分
### 第 5 章 文件存储与上传
- MinIO 抽象 `internal/storage/`、`file2document`、文件名与类型校验、content hash
### 第 6 章 异步入库任务：状态机与工作分发 ★
- `IngestionTask` 状态：CREATED → RUNNING → COMPLETED / FAILED / STOPPING → STOPPED
- NATS pull consumer、按 worker 空闲拉取（不预取）、ack / nack / 心跳续租
- 进度：`progress_sink`、组件进度比例、文档 `progress / progress_msg`
### Lab 1：mini-ragflow v0.1 —— 建知识库，上传文件，后台解析，前端实时看进度

## Part III 解析与切分 → v0.2
### 第 7 章 入库流水线：把入库写成一张 DSL 图 ★
- 模板 `ingestion/pipeline/template/*.json`（general / qa / book / paper / laws / table …）
- 组件注册与上下游数据契约（`component/schema`）；`parser_id` 怎么映射到模板
### 第 8 章 解析器家族与 DeepDoc ★
- `parser/parser/`：markdown / html / docx / xlsx / csv / text&code / email / epub
- DeepDoc：版面识别（DLA）、OCR（det + rec）、表格结构识别（TSR），`deepdoc/native` 用 ONNX Runtime
- mini 版：纯 Go 的 PDF 文本提取 + 按页记位置，交代省掉了什么
### 第 9 章 切分策略 ★★
- General / Token：按分隔符切开、按 token 预算合并、overlap、超长切分（`chunker/token.go: mergeUnits`）
- Title / Hierarchy、QA、Table、One、Page、Manual：每种模板适合哪类文档
- 表格 / 图片的上下文附着（`attachMediaContext`）
### 第 10 章 分词与向量化：Tokenizer 组件
- `content_ltks` / `content_sm_ltks` / `title_tks` 是什么；中文细粒度分词
- 标题向量与正文向量按 `filename_embd_weight=0.1` 加权合并、`q_<dim>_vec` 字段、embedding 缓存、截断
### Lab 2：mini-ragflow v0.2 —— 可配置的入库流水线，可在 UI 上查看和编辑 chunk

## Part IV 检索 → v0.3
### 第 11 章 文档引擎抽象：DocEngine 与索引 schema
- `engine/engine.go: DocEngine`；ES / Infinity / OceanBase 的取舍；chunk 字段表
### 第 12 章 查询理解：从问题到检索表达式
- `nlp/query_builder.go: Question`：全角转半角、繁转简、term weight、同义词、短语查询
- 字段权重 `important_kwd^30 question_tks^20 title_tks^10 content_ltks^2`
### 第 13 章 混合检索与重排 ★★
- 第一遍召回：全文 + kNN 融合，`min_match` 与“没结果就放宽阈值重试”
- 第二遍打分：`sim = (1-w)·tsim + w·vsim`，token 相似度、rank feature（pagerank / tag）、rerank 模型
- 阈值、分页、父子 chunk 聚合、删除过滤
### 第 14 章 检索测试页（前端）
- `web/src/pages/dataset/testing`：参数面板、命中高亮、分数拆解
### Lab 3：mini-ragflow v0.3 —— 自研混合检索引擎 + 检索测试页

## Part V 知识库问答 → v0.4
### 第 15 章 Chat 管线：从问题到 prompt
- `ChatPipelineService.AsyncChat`：多轮问题改写、检索、`KbPrompt` 按 token 预算装填、`messageFitIn`、空结果回复
### 第 16 章 流式输出与引用 ★
- SSE 帧格式；`[ID:n]` 协议、坏格式修复、用向量回填引用（`citation.go: findCitations`）
### 第 17 章 聊天 UI：引用悬浮与原文定位
- `web/src/pages/next-chats`：流式渲染、引用气泡、文档预览
### Lab 4：mini-ragflow v0.4 —— 带引用的流式知识库问答

## Part VI 进阶 → v0.5
### 第 18 章 Extractor：用 LLM 给 chunk 补关键词、问题、摘要、元数据
### 第 19 章 元数据过滤与标签
### 第 20 章 RAPTOR 与知识编译：递归摘要树、思维导图、结构图、Wiki
### 第 21 章 Agentic RAG 一瞥：多跳检索与 Deep Research（以读源码为主）
### Lab 5：mini-ragflow v0.5 —— LLM 增强 + 元数据过滤 + RAPTOR

## Part VII 平台化（深）→ v1.0
### 第 22 章 多租户、权限与 API Key
### 第 23 章 对外检索接口：REST、OpenAI 兼容、Dify 外部知识库、MCP
### 第 24 章 Ingestor 扩容：租约、心跳、重投递、并发与限流
### 第 25 章 数据源同步：Syncer 与增量入库
### 第 26 章 部署与可观测：Docker Compose 一键启动
### Lab 6：mini-ragflow v1.0 —— 多进程部署 + 对外 API + 端到端验证

## 附录
- A 源码地图（概念 → 路径）
- B 术语表
- C 同类系统对比（Dify 知识库、LlamaIndex、Haystack、Weaviate/Qdrant 原生混合检索）
- D 练习参考答案
