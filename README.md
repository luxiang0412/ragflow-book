# 从 RAGFlow 到 Mini-RAGFlow

> 读 RAGFlow 源码，从零写一个能跑的知识库 RAG 引擎。
> 参照仓库：`infiniflow/ragflow` @ `11e00194b`（Go 实现为主）；配套代码：[github.com/luxiang0412/mini-ragflow](https://github.com/luxiang0412/mini-ragflow)（每个 Part 一个标签；本地写作时位于 `code/mini-ragflow`）。在线阅读：[luxiang0412.github.io/ragflow-book](https://luxiang0412.github.io/ragflow-book/)。

读完这本书，你能从零写出一个前后端都能运行的知识库系统：上传文档 → 解析 → 切分 → 向量化 → 建索引 → 混合检索与重排 → 带引用的流式问答，再加上 LLM 增强、元数据过滤、RAPTOR、多租户、对外 API（OpenAI 兼容 / Dify / MCP）、多进程部署和数据源同步。每一个机制都先读 RAGFlow 怎么做，再自己实现一遍，最后用真实的运行结果验证。

## 目录

**Part 0 导读**
- [第 0 章 本书怎么读](./chapters/00-how-to-read.md)

**Part I 全景**
- [第 1 章 RAG 要解决什么问题，RAGFlow 押注在哪](./chapters/01-why-rag.md)
- [第 2 章 RAGFlow 架构鸟瞰](./chapters/02-architecture.md)
- [第 3 章 一份文档的一生：从上传到被引用 ★](./chapters/03-one-document-life.md)

**Part II 地基：知识库、文件与任务 → v0.1**
- [第 4 章 数据模型：Tenant / Knowledgebase / Document / Chunk](./chapters/04-data-model.md)
- [第 5 章 文件存储与上传](./chapters/05-file-storage-upload.md)
- [第 6 章 异步入库任务：状态机与工作分发 ★](./chapters/06-ingestion-tasks.md)
- [Lab 1：v0.1 建知识库、上传、后台解析、前端看进度](./chapters/lab-1-v01.md)

**Part III 解析与切分 → v0.2**
- [第 7 章 入库流水线：把入库写成一张 DSL 图 ★](./chapters/07-pipeline-dsl.md)
- [第 8 章 解析器家族与 DeepDoc ★](./chapters/08-parsers-deepdoc.md)
- [第 9 章 切分策略 ★★](./chapters/09-chunking.md)
- [第 10 章 分词与向量化：Tokenizer 组件](./chapters/10-tokenizer-embedding.md)
- [Lab 2：v0.2 可配置的入库流水线](./chapters/lab-2-v02.md)

**Part IV 检索 → v0.3**
- [第 11 章 文档引擎抽象：DocEngine 与索引 schema](./chapters/11-doc-engine.md)
- [第 12 章 查询理解：从问题到检索表达式](./chapters/12-query-builder.md)
- [第 13 章 混合检索与重排 ★★](./chapters/13-hybrid-rerank.md)
- [第 14 章 检索测试页（前端）](./chapters/14-retrieval-testing-ui.md)
- [Lab 3：v0.3 自研混合检索引擎 + 检索测试页](./chapters/lab-3-v03.md)

**Part V 知识库问答 → v0.4**
- [第 15 章 Chat 管线：从问题到 prompt](./chapters/15-chat-pipeline.md)
- [第 16 章 流式输出与引用 ★](./chapters/16-streaming-citations.md)
- [第 17 章 聊天 UI：引用悬浮与原文定位](./chapters/17-chat-ui.md)
- [Lab 4：v0.4 带引用的流式知识库问答](./chapters/lab-4-v04.md)

**Part VI 进阶 → v0.5**
- [第 18 章 Extractor：用 LLM 给 chunk 补关键词、问题、元数据](./chapters/18-extractor.md)
- [第 19 章 元数据过滤与标签](./chapters/19-metadata-filter.md)
- [第 20 章 RAPTOR 与知识编译](./chapters/20-raptor-compile.md)
- [第 21 章 Agentic RAG 一瞥：多跳检索与 Deep Research](./chapters/21-agentic-rag.md)
- [Lab 5：v0.5 LLM 增强、元数据过滤、RAPTOR](./chapters/lab-5-v05.md)

**Part VII 平台化 → v1.0**
- [第 22 章 多租户、权限与 API Key](./chapters/22-tenancy-auth.md)
- [第 23 章 对外检索接口：REST、OpenAI 兼容、Dify 外部知识库、MCP](./chapters/23-external-apis.md)
- [第 24 章 Ingestor 扩容：租约、心跳、重投递、并发与限流](./chapters/24-ingestor-scaling.md)
- [第 25 章 数据源同步：Syncer 与增量入库](./chapters/25-syncer.md)
- [第 26 章 部署与可观测：Docker Compose 一键启动](./chapters/26-deploy-observe.md)
- [Lab 6：v1.0 多进程部署、对外 API、端到端验证](./chapters/lab-6-v10.md)

**附录**
- [附录 A 源码地图：概念 → 路径](./appendix/a-source-map.md)
- [附录 B 术语表](./appendix/b-glossary.md)
- [附录 C 同类系统对比](./appendix/c-comparison.md)
- [附录 D 练习参考答案](./appendix/d-answers.md)

★ 表示核心章节。全书大纲见 [OUTLINE.md](./OUTLINE.md)。

## 快速开始

```bash
git clone https://github.com/luxiang0412/mini-ragflow.git && cd mini-ragflow
git checkout v1.0                     # 或 v0.1 … v0.5，跟着对应的 Part 读
cd backend && go test ./...
cd ../web && npm install && npm run build
cd ../backend && go run ./cmd/mini-ragflow --static ../web/dist --sync-root ../samples
# 打开 http://localhost:9380 ，注册一个账号
```

不配置任何模型也能运行：内置的 mock embedding（特征哈希）和 mock LLM 是确定性的，所有测试离线可跑。接真实模型（任何 OpenAI 兼容的服务，比如 Ollama）：

```bash
go run ./cmd/mini-ragflow --static ../web/dist --openai-base-url http://localhost:11434/v1
```

一键部署（1 个 API + 2 个 Ingestor + 1 个 Syncer）：

```bash
cd mini-ragflow && docker compose up -d --build
node web/e2e/deploy-e2e.mjs http://localhost:9380
```

| 版本 | 内容 | 对应 |
|---|---|---|
| v0.1 | 知识库、上传、异步解析任务、前端进度 | Part II |
| v0.2 | 流水线 DSL、解析器、6 种切分模板、分词与向量 | Part III |
| v0.3 | 混合检索引擎、查询构造、两阶段打分、检索测试页 | Part IV |
| v0.4 | 聊天助手、SSE 流式、引用修复与回填、聊天 UI | Part V |
| v0.5 | LLM 抽取、元数据过滤、RAPTOR | Part VI |
| v1.0 | 账号与 API Key、OpenAI/Dify/MCP、多进程、同步、Compose | Part VII |

`git diff v0.4 v0.5` 就是 Part VI 新增的全部代码。v1.0 共约 9800 行 Go（不含测试；测试约 2200 行、72 个测试函数）和 1500 行 TypeScript。

## 这本书是怎么验证的

- **源码引用**：所有 `path:line` 都对照 RAGFlow 提交 `11e00194b` 检查过；成书前又用脚本把全书的引用逐条取出原文复核了一遍，修正了两处行号；
- **运行结果**：书中的输出都是真实运行的结果。命令在对应的版本标签上执行，没有"预期输出"；
- **真实模型**：除了 mock，还用本地 Ollama（CPU）上的 bge-m3、qwen2.5:0.5b、qwen2.5:3b、nomic-embed-text 验证过检索、问答、抽取、元数据过滤、RAPTOR；用 OpenAI 官方 Python SDK 和 MCP Inspector 验证过对外接口；
- **界面**：每个版本都有一个 headless Chrome 脚本（`web/e2e/ui-v*.mjs`）走一遍界面并截图，书中的截图都来自这些脚本；
- **部署**：最终版本用 Docker Compose 部署，并用 `deploy-e2e.mjs` 做了端到端验证；多进程故障转移（`kill -9` 与 SIGTERM）用真实进程实验过；
- **踩坑记录**：各章的"踩坑记录"只写实际遇到的问题，自查发现但未实际观测到的会注明。开发和写作中发现并修复的问题汇总在各个 Lab 的"踩坑汇总"表格里；其中一部分是成书时对照 RAGFlow 源码或重新运行才发现的，修复后 `v1.0` 标签随之移动过几次。

## 已知的局限

- mini-ragflow 是教学实现：SQLite + 本地文件，只能单机部署；没有 DeepDoc 的版面分析和 OCR；没有团队权限、知识图谱、Agentic RAG 和 Agent 画布（见附录 C）；
- 第 21 章（Agentic RAG）和 RAGFlow 的知识编译中除 RAPTOR 以外的部分只读源码，没有实现；
- Dify 外部知识库接口只用 `httptest` 和协议文档验证过，没有接入真实的 Dify；
- 第 23 章关于 RAGFlow 官方 Python 示例的两个问题，是根据 SDK 的实际请求和 Go 路由/请求结构的代码得出的，没有启动完整的 RAGFlow 复现；
- RAGFlow 正在快速演进（Python → Go 迁移中），行号和部分行为会随版本变化，请以符号名为准。
