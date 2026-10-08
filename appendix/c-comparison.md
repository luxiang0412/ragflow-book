# 附录 C 同类系统对比

> 说明：RAGFlow 和 mini-ragflow 的内容来自本书对源码的阅读和实际运行；Dify 的内容来自作者此前对 Dify 源码的阅读（《从 Dify 到 Mini-Dify》第 22、23 章）；LlamaIndex、Haystack、Weaviate、Qdrant 只列出公开文档中广为人知的设计，作者没有在本书中运行它们，细节以各自的最新文档为准。

## C.1 定位

| | 是什么 | 谁来用 |
|---|---|---|
| **RAGFlow** | 一个完整的知识库平台：上传/同步 → 深度解析 → 切块 → 检索 → 问答，带界面和多租户 | 想直接部署一套知识库服务的团队 |
| **Dify** | LLM 应用平台，知识库是其中一个模块，主要服务于工作流和 Agent | 想搭建各种 LLM 应用、知识库只是其中一环的团队 |
| **LlamaIndex / Haystack** | Python 库/框架，提供构建 RAG 的组件（加载器、切分器、检索器、管线） | 写代码自己组装 RAG 的开发者 |
| **Weaviate / Qdrant** | 向量数据库，原生支持混合检索 | 作为 RAG 系统的存储与检索层 |
| **mini-ragflow** | 本书的教学实现，结构照着 RAGFlow，规模小两个数量级 | 读者 |

## C.2 入库

| | 解析 | 切块 | 入库增强 |
|---|---|---|---|
| RAGFlow | DeepDoc：版面分析、表格结构识别、OCR；也可接 MinerU、PaddleOCR 等（第 8 章） | 按文档类型选模板：general/book/laws/qa/table/one……，可视化的流水线（第 7、9 章） | Extractor（关键词、问题、元数据）、RAPTOR、思维导图/结构图/Wiki 等知识编译（第 18、20 章） |
| Dify | 按文件类型分派的 extractor，可接 Unstructured 服务 | 分隔符 + 递归切分 + 重叠；父子分段 | 问答模式（为分段生成问题） |
| LlamaIndex / Haystack | 各种 Reader/Converter，第三方解析服务 | 多种 splitter / node parser | 由用户自己用组件组合 |
| mini-ragflow | Markdown/HTML/文本/DOCX/CSV/XLSX 自己解析，PDF 用 `pdftotext` | 6 个模板，title/general/qa/table/one | Extractor、RAPTOR（watershed） |

RAGFlow 的特点是"**解析优先**"：它投入最多的是把 PDF、扫描件、表格**读对**，以及针对不同文档类型的切块模板。它的口号是 "Quality in, quality out"。

## C.3 检索

| | 全文 | 向量 | 融合 | 重排 |
|---|---|---|---|---|
| RAGFlow | ES/Infinity/OpenSearch 的全文检索，多字段加权（`title_tks^10`、`important_kwd^30`……），`*_tks` 字段只看是否出现（第 11、12 章） | 引擎内 kNN，全文查询作为 kNN 的过滤条件（ES） | 应用内二次打分：`(1-w)·tsim + w·vsim`，w 默认 0.3；空结果时放宽 `min_match`、降低阈值，最后退回纯向量（第 13 章） | 可选重排模型；还有 pagerank 和标签特征 |
| Dify | 由向量库提供（或经济模式下的 jieba 关键词表） | 由向量库提供 | 加权融合（关键词分数用 TF-IDF 余弦），或交给重排模型 | 重排模型 |
| Weaviate | BM25 | HNSW 等 | `hybrid` 查询，`alpha` 控制两路权重，提供 ranked / relative-score 两种融合 | 可接重排模块 |
| Qdrant | 稀疏向量 | 稠密向量 | Query API 用 prefetch 组合多路结果，支持 RRF 等融合 | 可在外部做 |
| mini-ragflow | 内存倒排 + BM25（`*_tks` 只看是否出现） | 暴力余弦 | 照搬 RAGFlow 的二次打分和重试 | 可选 rerank 模型 |

RAGFlow 的检索比"向量库 + 加权融合"多了一层**应用内的二次打分**：引擎负责召回，最终的分数在应用里重新计算。这让它能把检索词权重、标签、pagerank 这些引擎不知道的信号加进来，也让检索测试页能展示"全文分 / 向量分 / 总分"（第 14 章）。代价是每次检索都要把一批候选拉回应用。

## C.4 问答与引用

| | 引用 | 不知道时 | 多步检索 |
|---|---|---|---|
| RAGFlow | 提示词要求 `[ID:n]`；生成后修复格式、校验编号；模型没有引用时按向量相似度回填（第 16 章） | 检索为空且配置了 `empty_response` 时直接返回它，不调用 LLM；没配置则仍调用 LLM | 思考等级 1~4 的研究循环、等级 5 的 smart-reasoning 智能体（第 21 章） |
| Dify | 返回检索到的分段作为"引用与归属"，回答文本中一般不插入编号 | 由提示词和工作流决定 | 用工作流或 Agent 自己编排 |
| mini-ragflow | 同 RAGFlow，另外能识别"拒答"并去掉引用 | 同 RAGFlow | 无 |

## C.5 平台

| | 多租户 | 数据源同步 | 对外接口 | 任务系统 |
|---|---|---|---|---|
| RAGFlow | 用户即租户，团队成员通过 `user_tenant` 共享 | 41 个连接器，增量窗口 + 断点续传 + 完整快照删除（第 25 章） | REST、OpenAI 兼容、Dify 外部知识库、MCP（第 23 章） | NATS JetStream + 数据库状态（第 24 章） |
| Dify | 工作空间（workspace） | 数据源插件 | 应用 API、知识库 API | Celery + Redis |
| mini-ragflow | 用户即租户，无团队 | 一个文件夹连接器 | 同 RAGFlow 的三种 | SQLite 任务表 + 租约 |

## C.6 mini-ragflow 与 RAGFlow 的差距一览

| 方面 | RAGFlow | mini-ragflow |
|---|---|---|
| 规模 | `internal/` 和 `cmd/` 下非测试 Go 代码约 49 万行，外加 Python 遗留代码和前端 | 后端约 9700 行 Go（不含测试），前端十几个页面 |
| 存储 | MySQL/PostgreSQL + MinIO + ES/Infinity/OpenSearch + Kvrocks + NATS | 一个 SQLite 文件 + 本地目录 |
| 解析 | DeepDoc（ONNX 模型、版面、表格、OCR） | 文本类格式 + `pdftotext` |
| 检索 | 引擎内全文 + kNN，多种引擎 | 内存倒排 + 暴力余弦 |
| 进阶 | 知识图谱、知识编译（树/图/思维导图/Wiki）、Agentic RAG、Agent 画布 | Extractor、RAPTOR |
| 部署 | 一个应用容器内多进程 + 一组基础设施，可多机扩展 | Compose 中按角色拆分，单机 |

mini-ragflow 的价值不在于替代 RAGFlow，而在于让每一个机制都能在几百行代码里被看清、被改动、被测试。读完本书再回到 RAGFlow，你应该能在它的源码里认出这些机制，知道每一层复杂度是为了解决什么问题而存在的。
