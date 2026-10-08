# 附录 B 术语表

按拼音/字母顺序排列。括号里是英文或代码里的名字，最后是首次详细讲解的章节。

| 术语 | 解释 | 章 |
|---|---|---|
| Agentic RAG | 让模型自己决定检索什么、读什么、什么时候停的 RAG，通常是多轮检索 + 证据充分性判断 | 21 |
| API Key | 给程序调用用的长期令牌，RAGFlow 的格式是 `ragflow-` 加随机串 | 22 |
| BM25 | 经典的全文相关度算法，考虑词频、逆文档频率和文档长度 | 11 |
| chunk（块） | 文档切分后的检索单位，也是写进检索引擎的一行 | 3、9 |
| chunk_method / parser_id | 知识库或文档的"切块方法"（general、book、laws、qa、table、one……），决定用哪个流水线模板 | 7 |
| CJK 二元组（bigram） | 中日韩文本没有空格，把相邻两个字作为一个检索词 | 10 |
| DeepDoc | RAGFlow 的文档理解模块：版面分析、表格结构识别、OCR | 8 |
| DSL（流水线描述） | 用 JSON 描述的组件图：每个组件的名字、参数、下游 | 7 |
| embedding（向量） | 把文本映射成定长向量，语义相近的文本向量也相近 | 10 |
| extra_body | OpenAI 兼容接口里放 RAGFlow 专有参数（如 `reference`）的对象 | 23 |
| Extractor | 用 LLM 给 chunk 补充关键词、问题、元数据的流水线组件 | 18 |
| fingerprint（指纹） | 内容或版本的摘要值，用来判断源文档有没有变 | 25 |
| FetchRef | 连接器给出的"如何下载"的引用，用于懒下载 | 25 |
| fusion（融合） | 把全文检索和向量检索的结果合成一个排序，如加权求和或 RRF | 13 |
| hybrid search（混合检索） | 同时使用全文检索和向量检索 | 13 |
| Ingestor | 执行入库任务（解析、切块、向量化、写索引）的进程 | 2、6 |
| JetStream | NATS 的持久化消息流，RAGFlow 用它投递入库和同步任务 | 24 |
| kNN | k 近邻：找与查询向量最相近的 k 个向量 | 11 |
| knowledge compilation（知识编译） | 入库时把内容组织成摘要树、图谱、思维导图、Wiki 等更高层的产物 | 20 |
| lease（租约） | 认领任务时写入的"我在处理，有效期到某时刻"，过期即可被他人接手 | 24 |
| MCP | Model Context Protocol，智能体调用外部工具的协议，基于 JSON-RPC 2.0 | 23 |
| metadata filter（元数据过滤） | 按文档属性（如型号、日期）缩小检索范围的硬条件 | 19 |
| min_match / minimum_should_match | 全文查询中至少要命中的检索词比例，RAGFlow 默认 0.3，重试时降到 0.1 | 12 |
| mock provider | 本书中确定性的假模型（假 embedding 用特征哈希，假 LLM 从上下文里摘句子），让一切离线可跑 | 0 |
| pagerank（文档权重） | RAGFlow 里人为设置的知识库权重，加到排序分数上 | 13 |
| pipeline（流水线） | 入库的一串组件：File → Parser → Chunker → Extractor → Tokenizer → … | 7 |
| poison message（毒消息） | 每次处理都会让 worker 失败或崩溃的任务，需要投递次数上限 | 24 |
| prune（删除同步） | 根据源的完整清单，删除知识库里源已不存在的文档 | 25 |
| RAG | Retrieval-Augmented Generation，先检索再让模型基于检索结果生成 | 1 |
| RAPTOR | 递归摘要树：聚类 → 摘要 → 再聚类，所有层都进入索引 | 20 |
| reasoning level（思考等级） | RAGFlow 聊天的检索策略开关：0 普通、1~4 研究循环、5 smart-reasoning | 21 |
| reference（引用） | 回答所依据的 chunk 列表，回答中用 `[ID:n]` 指向它们 | 16 |
| rerank（重排） | 用专门的重排模型对候选重新打分 | 13 |
| SCA | Sufficient Context Agent，RAGFlow 研究循环里判断"证据是否足够"的审查步骤 | 21 |
| scripted_sim | RAGFlow 在 ES 映射里给 `*_tks` 字段定义的打分：只看是否出现，不做长度归一 | 11 |
| SourceID | 连接器内稳定、唯一的源文档 id，同步与删除的依据 | 25 |
| SSE | Server-Sent Events，基于 HTTP 的单向流，用来流式返回回答 | 16 |
| Syncer | 执行数据源同步的进程 | 25 |
| tag（标签） | RAGFlow 的软排序特征：问题与 chunk 标签越相似，分数加得越多 | 19 |
| tenant（租户） | 数据的归属单位；RAGFlow 中租户 id 等于其所有者的用户 id | 22 |
| term weight（词权重） | 每个检索词的重要性，基于 IDF 和词性等 | 12 |
| threshold（相似度阈值） | 低于它的 chunk 不返回，RAGFlow 默认 0.2 | 13 |
| tokenizer（分词） | 把文本切成检索词；RAGFlow 生成粗粒度 `*_ltks` 和细粒度 `*_sm_ltks` | 10 |
| top_k / top_n | 向量检索的候选数 / 最终返回或送入提示词的块数 | 13、15 |
| vector_similarity_weight | 混合打分中向量相似度的权重，全文权重为 1 减去它，默认 0.3 | 13 |
| watershed | RAGFlow RAPTOR 的分段算法：按文档顺序，在相邻向量相似度低的地方切开 | 20 |
| WAL | SQLite 的预写日志模式，允许读写并发 | 24、26 |
| 两阶段检索 | 第一阶段由引擎召回较多候选，第二阶段在应用里重新打分、过滤、分页 | 13 |
| 多轮改写（refine） | 用 LLM 把依赖上下文的追问改写成独立的问题 | 15 |
| 断点续传（checkpoint） | 同步任务每批提交后保存的位置，失败重跑时从这里继续 | 25 |
| 哨兵值（sentinel） | 一个不可能存在的值（如文档 id `"-999"`），用来表达"空集合"而不被当成"不限制" | 19 |
| 重投递（redelivery） | 任务没有被确认（进程死了或主动放弃）时，再交给其他 worker | 24 |
