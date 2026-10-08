# 附录 D 练习参考答案

> "巩固"和"扩展"题给出思路和要点，不预先给出运行结果：数字请以你自己的运行为准。"读源码"题的答案都在写作时对照 RAGFlow 源码（提交 `11e00194b`）核对过，引用了相应的位置。

## 第 0 章

1. **巩固**：v0.1 只有通用切分，三份文档的块数取决于 `chunk_token_num`，在文档页面能看到每份文档的块数。
2. **扩展**：常见的结构有：带编号的标题层级（手册、书籍）、"第 X 条"式条款（制度、法规）、一问一答（FAQ）、行列（表格、CSV）、整篇不宜切开（合同、短公告）。分别对应第 9 章的 title、laws、qa、table、one。
3. **读源码**：`cmd/ragflow_server.go:165-180` 有 `--admin`、`--ingestor`、`--api`、`--syncer`、`--deepdoc`、`--migrate` 六个模式开关：管理服务、入库执行、对外 API、数据源同步、DeepDoc 推理服务、一次性数据库迁移。

## 第 1 章

1. **巩固**：标题路径既在 `title_tks` 等字段里贡献关键词命中（"S1""除垢"这样的词出现在标题里），也被拼进向量化的文本，让一个只有步骤、没有主语的块在语义上"知道自己属于哪台机器的哪一节"。
2. **扩展**：两份结构相同、只有型号和个别数字不同的手册，问"S1 的 X 是多少"。除了元数据过滤，还可以：把型号写进每个块的标题路径；用文档名参与向量化（第 10 章的 `filename_embd_weight`）；在提示词里要求模型核对型号；拆成两个知识库。
3. **读源码**：两个模板的组件序列相同（File → Parser → 切分器 → Tokenizer → Extractor），区别在切分器：`general` 用 `GeneralChunker`（`chunk_token_size` 512，按换行和中英文句末标点切）；`laws` 用 `TitleChunker`，`method: "hierarchy"`、`hierarchy: 2`、`include_heading_content: false`，并给出 5 组标题正则（Markdown 的 `#`、"第X编/章/节/条"、数字编号等）。

## 第 2 章

1. **巩固**：MySQL 的 `document` 表（以及 `file`、`file2document`）、MinIO 中以知识库 id 为桶的对象；检索引擎只在解析完成、写入 chunk 时才写入。
2. **扩展**：`RETURNING` PostgreSQL 支持（MySQL 不支持，需要改成"先 `SELECT … FOR UPDATE SKIP LOCKED` 再 `UPDATE`"）；`INSERT OR REPLACE` 换成 `INSERT … ON CONFLICT … DO UPDATE`；`pragma_table_info` 换成查询 `information_schema.columns`；`?` 占位符换成 `$1`；还要去掉 `SetMaxOpenConns(1)` 这类为 SQLite 做的设置。
3. **读源码**：见第 26 章 26.1 节：几个进程同时执行迁移会互相冲突，单独执行一次 `--migrate` 可以保证迁移在任何服务启动之前完成，而且只执行一次。

## 第 3 章

1. **巩固**：上传、创建任务、检索、问答的入口都是同步的；任务的执行（解析、切块、向量化、写索引）是异步的。解析可能要几分钟，而且可以重试，不能占着 HTTP 连接；检索是用户在等结果，必须同步返回。
2. **扩展**：消息变大（文件可能上百 MB）；重投递时要重复传输；任务在队列里等待时用户改了配置，消息里的快照就过期了。只放 id，执行时再从数据库读取最新状态，这些问题都不存在。
3. **读源码**：`createAndEnqueueWithKBCache`（`internal/service/ingestion_task_service.go:573`）先用 `GetByDocumentID` 查这个文档已有的任务：`CREATED` 的直接重新投递；`FAILED`/`STOPPED` 的转回 `CREATED`、清掉上一次的运行标识后重新投递；其他状态（正在运行）直接报错。只有不存在时才新建一行。mini-ragflow 每次新建一行的好处是保留了每次执行的历史；坏处是要额外保证"同一个文档同时只有一个活跃任务"（`CreateTask` 在同一个事务里检查该文档是否已有 `CREATED`/`RUNNING`/`STOPPING` 的任务），并且任务表会持续增长。

## 第 4 章

1. **巩固**：两个 worker 同时读到 `chunk_num = 10`，各自加 5 后写回 15，实际应为 20。`chunk_num = chunk_num + ?` 让数据库在一条语句里完成读-改-写。
2. **扩展**：放在 `document` 表适合"按标签筛文档"、修改代价低；写进每个 chunk 适合在检索引擎里直接过滤或加权，但修改标签要更新所有 chunk。RAGFlow 的元数据走第一条路（单独的元数据索引），标签（`tag_feas`）走第二条路。
3. **读源码**：`conf/infinity_mapping.json` 是一张固定的表结构：一个物理列配多个分词器，例如 `content` 列配置了 `"analyzer": ["rag-coarse", "rag-fine"]`，注释写着它对应逻辑字段 `content_with_weight, content_ltks, content_sm_ltks`。向量列不在这个文件里，而是建表时按维度动态加上：类型 `vector,<dim>,float`，列名 `q_<dim>_vec`，并建索引 `q_<dim>_vec_idx`（`internal/engine/infinity/chunk.go:99-190`）。ES 则用动态模板按字段名后缀决定类型（第 11 章）。

## 第 5 章

1. **巩固**：`filepath.Join(root, "../../evil")` 会被清理成 `root` 的上两级目录下的 `evil`，测试断言文件出现在 `root` 外面即可。
2. **扩展**：同步器用 `connector_doc` 记录"源 id → 文档 id"，自动改名后文档名和源文件名不再一致，所以不能再用名字匹配（mini-ragflow 本来也没有用名字）；但第一次同步时的重名冲突需要处理：要么改名，要么跳过并记为失败。
3. **读源码**：`internal/handler/document.go:1035` 起，只放行 `table_column_mode` 和 `table_column_roles` 两个键，格式不对时静默忽略。`parser_config` 里有影响资源消耗（如 LLM 调用、RAPTOR）和安全（如外部服务地址）的参数，这些应该由有权限修改知识库配置的人在知识库层面设置，而不是任何能上传文件的人在上传时覆盖。

## 第 6 章

1. **巩固**：A 认领：`owner=A, attempts=1`；A 卡住，租约过期；B 认领：`owner=B, attempts=2`；A 醒来调用 `FinishTask(id, A, …)`，`WHERE lease_owner = 'A'` 不匹配，更新 0 行，结算失败。A 的心跳也会失败，触发 `errLeaseLost`，A 放弃这个任务。
2. **扩展**：同进程时，`RequestStop` 直接取消对应任务的 context（`map[taskID]cancel`）；跨进程时，数据库里的 `STOPPING` 仍由心跳读取，把心跳中的状态检查拆成一个更快的轮询（比如 500 毫秒只读不写）即可。
3. **读源码**：函数的返回值只代表"这段代码认为自己做完了"。数据库写入可能失败、租约可能已被他人接手、panic 可能打断了一半，`settleMessage` 注释里写明了以数据库为准（"DB truth is authoritative"）。第 24 章练习 3 是同一个问题。

## 第 7 章

1. **巩固**：模板是用 `go:embed` 嵌入、启动时遍历 `templates/` 目录加载的（`internal/pipeline/pipeline.go:192`），`chunk_method` 只是模板名，所以新增一个 JSON 文件、重新构建即可。
2. **扩展**：`Order` 改为返回一张 DAG（或分层的拓扑序）；`State` 里的 `Chunks` 改成按组件 id 存放，合并组件再把多个上游的 chunk 拼起来，并处理 id 冲突。
3. **读源码**：`PipelineExecutor` 默认的 `loadDSLFunc` 是 `loadDSLFromCanvas`（`internal/ingestion/task/pipeline_executor.go:1015`）：用 `UserCanvasDAO.GetByID` 从 `user_canvas` 表读出一行，DSL 存在 `dsl` 列（`longtext`，JSON），格式是一个"信封"：`{"dsl": {"components": …}}`，执行前要先 `UnwrapCanvasDSL`。入库流水线和 Agent 画布存在同一张表里，用 `canvas_category` 区分。

## 第 8 章

1. **巩固**：不需要改切分器：通用切分器只看文本，title 切分器只认 `title` 版面，`quote` 会被当作普通正文。
2. **扩展**：取每行的字高，算出全文的中位数，明显大于中位数（比如 1.3 倍以上）的短行判为标题，再根据字号的档次给出 `#`、`##` 级别。
3. **读源码**：`enrichOnePageWithDeepDoc`（`internal/deepdoc/parser/pdf/table_extract.go:68`）先对页面图片跑 DLA 得到版面区域，再调用 `AnnotateBoxLayouts`（`internal/deepdoc/parser/pdf/table/table_annotate.go:327`）把标签写到文本框上：区域先经过置信度过滤、排序和清理，并从图片像素坐标换算到 PDF 坐标；然后**按类型的优先级**（页脚、页眉、参考文献、图注、表注、标题、表格、正文、图、公式）依次处理，对每个文本框，在同一类型的区域中找覆盖它比例最大的那个（相同时取区域被覆盖比例更大的），覆盖比例达到 0.4 才贴上这个标签。所以一个跨两个区域的文本框，会被优先级更高、且覆盖它至少 40% 的那个类型的区域认领；页眉页脚类的框如果不在页面边缘，会被直接丢弃。

## 第 9 章

1. **巩固**：重叠比例变大后，每块能容纳的"新内容"变少，块数会增加；具体数字以运行结果为准，并对照第 9 章"缩放阈值"的规则检查。
2. **扩展**：`include_heading_content = true` 时，"总则"下的说明文字会成为单独的块（或并入第一条）；为 false 时只保留标题作为路径，说明文字的去向取决于实现。对照第 9 章的 `getPaths` 讲解检查你的结果。
3. **读源码**：`ManualChunker`（`internal/ingestion/component/chunker/manual.go`）和按标题分组的切分器用同样的分组逻辑，区别是它会先按 `(页码, top, left)` **重新排序**。多栏或手工排版的 PDF，解析器按阅读顺序（逐栏）输出，直接分组会把左栏后面的内容和右栏前面的内容交错在一起。没有坐标的输入（比如 docx）不排序，输出与按标题分组完全一致。

## 第 10 章

1. **巩固**：权重越大，文件名在向量里占的比重越大，同一份文档的所有块向量会越来越像，块之间的区分度下降。
2. **扩展**：要点是只替换 `Terms` 的实现，保证索引和查询用同一套分词；改完后重建索引。
3. **读源码**：`internal/tokenizer/embedlimit.go` 的文件注释给了一个真实例子：cl100k 和模型自己的分词器（如 XLM-R SentencePiece）的计数大约相差 ±2%，方向取决于内容。重复的数字表格在 cl100k 下压缩得更狠，一个 25000 字符的块 cl100k 计 8143 个 token（低于 8182 的上限），在模型自己的分词器里却是 8250 个，于是整个请求被服务商拒绝，整份文档失败。所以现在的做法是：用模型自己的分词器（或校准过的上界）计数，安全余量按比例取 2%（至少 32 个 token）。`truncateForEmbedding` 只是在 embedder 不能自己截断时的旧路径。

## 第 11 章

1. **巩固**：一个租户一个索引，索引和分片的数量不会随数据集数量增长（ES 每个分片都有固定开销），跨数据集检索就是一次查询加 `kb_id` 过滤；代价是删除一个数据集要按条件删除而不是删索引，不同数据集也无法使用不同的映射。
2. **扩展**：要点：`MatchText` → `query_string`（带字段权重和 `minimum_should_match`），`MatchDense` → 顶层 `knn`，`filter_knn` 时把全文查询放进 `knn.filter`；返回 `_score` 后仍然在应用里做第二阶段打分。
3. **读源码**：Infinity 的 `Search`（`internal/engine/infinity/chunk.go:780`）依次调用 `MatchText`、`MatchDense`，最后用 `Fusion("weighted_sum", …)` 合并。权重来自 `buildInfinityFusionExpr`（`internal/service/nlp/retrieval.go:601`）：`"(1-w),w"`，也就是说在 Infinity 上 `vector_similarity_weight` **直接参与引擎内的融合**。其他引擎用固定的 `"0.001,1"`（`esFusionWeights`），第一次检索实质上是向量召回，`vector_similarity_weight` 在之后的 `RerankWithKNN` 里才起作用。

## 第 12 章

1. **巩固**：观察 `TermWeights` 的输出：去掉疑问词之前"多少"会作为一个检索词出现，并分走一部分权重。
2. **扩展**：小数据集里几乎每个词的文档频率都很低或很高，idf 的区分度很差。可以用"语料词典 idf 为主、数据集 idf 平滑修正"的方式，比如 `idf = α·idf_corpus + (1-α)·idf_dataset`，α 随数据集规模减小。
3. **读源码**：英文分支（`internal/service/nlp/query_builder.go` 中 `!qb.IsChinese(...)` 之后）：分词、算权重、查 WordNet 同义词（每个词最多 8 个），然后为相邻两个词再加一个短语查询 `"左 右"^(2·max权重)`。中文分支按片段处理，对每个词还要做**细粒度切分**（`sm`）并把细粒度词也加进查询。因为英文单词之间有空格，切出来的就是最小单位；中文的粗粒度分词可能切出"咖啡机除垢剂"这样的长词，不拆开就匹配不到只写了"除垢剂"的文本。

## 第 13 章

1. **巩固**：按 `sim = (1-w)·tsim + w·vsim` 计算，w 变大后，向量相似度高但关键词命中少的块会往前排。
2. **扩展**：RRF 只用排名，不需要两路分数可比；但它丢掉了分数的绝对大小，"两路都很弱"和"两路都很强"在排名上可能一样，所以通常还要配合阈值。
3. **读源码**：当通用切分器配置了 `children_delimiters` 时，每个块会再按这些分隔符切成更小的"子块"写入索引，并通过 `mom_id` 指向父块。检索命中的是子块（更精确），`RetrievalByChildren`（`retrieval.go:1025`）按"父块 + 文档"分组，取回父块的完整内容，相似度取子块的平均值，并把子块的分词和关键词汇总到父块上，最终返回父块（上下文更完整）。

## 第 14 章

1. **巩固**：向量权重为 0 时 `sim = tsim`，而全文召回本身已经要求命中 `min_match` 比例的词；阈值仍然会过滤，但能通过全文召回的结果往往 `tsim` 不低。请以你的运行结果和第 13 章 13.4 节的说明为准。
2. **扩展**：后端需要返回每个结果命中的查询词及其权重（检索时已经算出 `TermWeights`，只需和块的分词求交集）。
3. **读源码**：`sanitizeHtmlWithImagesAsText`（`web/src/utils/dom-util.ts:32`）没有自定义白名单，用的是 DOMPurify 的默认配置（会去掉脚本和 `onerror` 这类事件属性），然后把所有 `<img>` 替换成它自身 HTML 的**文本**。注释说明了原因：防止通过外部图片 URL 进行追踪或数据外泄，以及图片打乱布局；同时用户仍能看到原文里有这样一个标签。

## 第 15 章

1. **巩固**：截断 system 时从知识块的末尾截起，排在后面的块（相似度较低但可能仍然相关）会丢失；更糟的是 system 里的回答规则和引用规则也可能被截掉，模型就会在没有约束的情况下回答。
2. **扩展**：要点：抽取关键词的 LLM 调用要设置较短的超时，失败时退回原问题；把关键词追加到检索问题而不是替换它。
3. **读源码**：`internal/service/chat_pipeline.go` 约 512-585 行：`prompt_config.parameters` 是 `[{key, optional}]` 的数组。如果助手关联了知识库、系统提示词里又有 `{knowledge}` 却没声明这个参数，会自动补上。随后逐个检查调用方传入的参数：必需参数缺失时，直接返回一个 `**ERROR**: Miss parameter: <key>` 的最终回答；可选参数缺失时，把系统提示词里的 `{key}` 替换成空格。

## 第 16 章

1. **巩固**：帧数大约是回答长度除以 mock 每帧的字数；首帧时间主要是检索加提示词构造的耗时。
2. **扩展**：先用词重叠（比如二元组的 Jaccard 相似度）为每句话筛出 2~3 个候选块，只对有候选的句子算向量，并缓存同一回答中重复句子的向量。
3. **读源码**：`ExpandRangeCitations`（`internal/service/citation.go:455`）处理模型自作主张写出的 `[ID:1-3]`、`[ID: 1 - 3]`、`[ID:1~3]` 这类范围引用。注释说得很直接：没有任何代码会生成它，也没有块能按它解析，但模型有时会把连续的引用压缩成范围，所以必须防御性地规范化：两端都在有效范围内才展开成 `[ID:1][ID:2][ID:3]`（反了就先交换），否则直接删掉。规则禁止，不等于模型不会这么写。

## 第 17 章

1. **巩固**：在 Performance 面板里对比"Scripting"时间和 React 的提交次数，合并后提交次数应该大幅下降。
2. **扩展**：后端返回块的 `page_num` 和位置坐标（RAGFlow 的 `positions`），前端用 PDF 查看器打开文件并跳到对应页、绘制高亮框；非 PDF 文件可以按块在文档中的顺序定位。
3. **读源码**：`getPopoverContent`（`web/src/components/markdown-content/index.tsx`）除了经过清洗的正文，还会显示：块关联的图片（悬停放大）、文档元数据（`document_metadata`，如果请求了的话）、文件缩略图或文件类型图标，以及可点击打开原文的文档名。`doc_aggs` 是"被引用的文档"的汇总，聊天消息下方的 `ReferenceDocumentList`（`components/next-message-item/index.tsx`）用它列出来源文档。

## 第 18 章

1. **巩固**：观察输出关键词的语言比例；注意小模型对示例的语言很敏感，示例本身也可能被照抄。
2. **扩展**：合并冲突的常见做法：按出现次数投票；按块的位置加权（文档开头的块更可能包含型号、日期等）；或者保留多值（数组）。RAGFlow 用 `mergeChunkMetadata`（`internal/ingestion/task/indexdoc/process.go:173`）合并，可以对照阅读。
3. **读源码**：`extractor_tag.go` 的 `runAutoTags` 分两个阶段：第一阶段在本地，用标签文件的样本建立倒排索引，把 chunk 和最相似的样本匹配，继承样本的标签，若有标签权重达到 6（`isHighConfidenceMatch`）就直接采用；其余的进入第二阶段，让 LLM 参考真实样本打标签。标签及权重写入 `tag_feas`，检索时与问题的标签做相似度，乘以 10 加到分数上（第 13、19 章）。

## 第 19 章

1. **巩固**：ISO 日期按字符串比较的顺序正确，两个条件的结果应该互补。
2. **扩展**：semi_auto 把可用的键限定在用户指定的几个，提示词里的元数据也只剩这几个键。它能减少模型编造键的空间，但不能阻止模型把中文键名翻译成英文；配合"丢弃不存在的键"，最坏的情况是生成不出条件，而不是生成错误的条件。
3. **读源码**：**关键词**，不是向量。`MatchTagQuery`（`internal/ingestion/component/tag_query.go:246`）用建索引时的同一个分词器切分问题，按 idf 加权的词重叠给标签文件里的记录打分，取前 100 条（`tagQueryTopLabels`），统计这些记录的标签分布，再用 `round(0.1·(count+1)/(total+S)/背景比例)` 换算成权重：一个标签在匹配记录里出现的比例相对于它在整个标签文件里的背景比例越高，权重越大。注释说明这是对 Python 版"引擎查询"的近似。

## 第 20 章

1. **巩固**：`tree_order` 越小，切点越多、每簇越小、层数越多、LLM 调用越多。
2. **扩展**：GMM 需要先确定簇数（可用 BIC 选择），并且同一簇的块在文档里可能不相邻；比较时记录每层的簇数、LLM 调用次数和摘要长度。
3. **读源码**：`mindmap` 的包注释（`knowledge_compiler/mindmap/mindmap.go`）：把源 chunk 按 token 预算打包成批，每批调用一次 LLM，LLM 回复一个 Markdown 大纲；大纲被解析成嵌套字典、把列表转换成键值，再跨批次合并，最后整理成 `{"id", "children"}` 的树，每个节点作为一个产物输出。"Markdown source is the LLM's reply, NOT the source document Markdown"的意思是：这里要解析的 Markdown 是 **LLM 生成的大纲**，而不是被入库的那份 Markdown 文档，所以不能复用流水线里的 Parser 组件。

## 第 21 章

1. **巩固**：在 RAGFlow 界面上观察执行轨迹，等级越高，检索次数和读取的文档通常越多，耗时也越长。
2. **扩展**：要点：拆分步骤要有超时和退路（失败时退回原问题）；每个子查询单独检索后按 chunk id 去重；合并后重新编号，交给第 15 章的 `KbPrompt`。
3. **读源码**：`ExtendDeadline`（`internal/rag/agentic-rag/agentic_rag_graph.go:109`）上方的注释给出了一个测量过的理由：预算刚好够做一遍，而枚举类问题需要两遍，第一遍被截断时漏掉的成员正是第二遍能补上的。只延长一次，是为了"再看一眼漏掉的成员"，而不是让一个不断声称还有成员的槽位表无限运行下去。

## 第 22 章

1. **巩固**：每个请求都写库会把只读请求变成写请求，在 SQLite 上还会和入库任务争抢写锁。做法：在内存里记录，每分钟批量写一次；或者只有距离上次记录超过一分钟时才写。
2. **扩展**：要点：`user_tenant(user_id, tenant_id, role)`；`loadKB` 的判断改为"自己的租户，或者是我加入的租户且 `permission = team`"；删除只允许 `created_by` 等于自己（RAGFlow 的 `Accessible4Deletion`）。
3. **读源码**：签名中的时间戳让一个被截获的令牌字符串最多一天有效（`accessTokenExpireSeconds = 86400`，`internal/utility/token.go`）；用户表里的 `access_token` 让服务端可以主动吊销：退出时改成 `INVALID_…`，重新登录时生成新值（`service/user.go`）。两者结合，既有自动过期，也能立即失效。密钥泄露时，攻击者仍然需要一个有效的 `access_token` 值才能伪造令牌；但如果他截获过某个用户的旧令牌，就可以用新的时间戳重新签名，让这个已过期的令牌"复活"，直到该用户下一次登录或退出。

## 第 23 章

1. **巩固**：用 `curl -N`，最后一帧（`finish_reason: "stop"` 的那一帧）的 `delta.reference` 就是引用列表。
2. **扩展**：`tools/list` 增加两个工具；`tools/call` 按名字分派。接入命令形如 `claude mcp add --transport http mini-ragflow http://localhost:9380/api/v1/mcp --header "Authorization: Bearer <API Key>"`。
3. **读源码**：流式过程中发出的是模型的原始输出；生成结束后，`decorateAnswer` 还会修复引用格式、校验编号、为没有引用的句子回填引用（第 16 章），最终答案和增量拼起来的文本并不相同。`final_content` 让客户端可以用最终版本替换掉流式拼出来的版本，和 `reference` 里的编号对得上。

## 第 24 章

1. **巩固**：心跳间隔变为约 0.67 秒，CPU 繁忙时一次心跳延迟超过 2 秒就会丢失租约，任务被别人认领，原 worker 走 `errLeaseLost` 分支放弃。可以用 `stress` 或 `taskset` 限制 CPU 来复现。
2. **扩展**：认领时不调用重置进度的 `SetDocumentRun`，而是 `AppendLog("— 第 N 次执行 —")`，并把进度条的计算限定在当前这次执行内。
3. **读源码**：例如：任务函数把状态更新为 COMPLETED 的那条 SQL 因为连接中断而失败，但函数没有正确传播这个错误，仍然返回 true；或者执行期间用户点了停止，数据库里已经是 STOPPING。如果按内存里的 true 去 Ack，消息就永远消失了，数据库里却留着一个非终态的任务，再也不会被处理。以数据库为准，这种情况会 Nack，让消息重投、重新结算。

## 第 25 章

1. **巩固**：最长等待时间 ≈ 连接器的 `refresh_seconds` + Syncer 的轮询间隔（默认 5 秒）+ Ingestor 的轮询间隔 + 解析耗时。`deploy-e2e.mjs` 创建的连接器 `refresh_seconds` 是 3600，所以要么等一小时，要么在页面上点"立即同步"。
2. **扩展**：分片上传（multipart）的对象，ETag 不是内容的 MD5，而是各分片 MD5 再取 MD5 并带上分片数；用服务端加密时也可能不同。所以 ETag 只能作为"版本"指纹：内容相同但上传方式不同，ETag 可能不同，导致一次多余的重新解析；反过来，ETag 相同则内容一定没变，这对同步来说已经足够。
3. **读源码**：`internal/syncer/README.md` 的 "Anchor invalidation" 一节：检查点所指的那个文档在源里被删了，续传就失去了可信的锚点。连接器返回 `ErrSyncResumeInvalid`，`SyncRunner` 清掉检查点、重置统计，从这个固定窗口的开头重新开始；最多重来 `MaxAnchorRestartCount` 次（默认 2，`sync_runner.go:79`），之后任务失败，而不是"猜一个新的偏移量"。限制次数是因为源可能在持续变化，锚点可能一再失效，无限重来会让任务永远完不成。

## 第 26 章

1. **巩固**：`--scale ingestor=4` 后系统页出现 4 个 Ingestor；`stop` 时正在运行的任务被归还为 `CREATED`，Ingestor 全部停止期间任务会一直等待，重新启动后被认领。
2. **扩展**：要点：`--mode migrate` 打开数据库、完成迁移后退出；Compose 里 `migrate` 服务 `restart: "no"`；其他服务 `depends_on: {migrate: {condition: service_completed_successfully}}`。
3. **读源码**：`entrypoint.sh` 在这里没有注释说明原因，下面是推测：在迁移完成、Syncer 启动之后，给它几秒钟完成初始化（比如连接 NATS、创建流和消费者），再启动 Admin、nginx 和 API，错开各进程的启动。固定等待的问题是：快的时候浪费时间，慢的时候又不够。更可靠的办法是让被依赖的组件提供就绪检查（readiness），依赖方轮询它，或者让每个进程自己在启动时幂等地创建所需的资源，不依赖启动顺序。
