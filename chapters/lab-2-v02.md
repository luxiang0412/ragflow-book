# Lab 2：mini-ragflow v0.2——可配置的入库流水线

> 目标：把 v0.1 写死的 `Runner.Run` 换成"模板 → 组件图 → 执行器"，支持 6 种格式、5 种切分算法，并为每个 chunk 算出分词和向量。
> 前置：第 7~10 章。

## 1. 这一版改了什么

```bash
git diff v0.1 v0.2 --stat -- backend | tail -1
```

```text
 47 files changed, 3562 insertions(+), 277 deletions(-)
```

| 新增/改动 | 文件 | 章节 |
|---|---|---|
| 解析器家族：md/html/docx/pdf/csv/xlsx/txt | `internal/parser/*.go` | 8 |
| 切分算法：General/Title/QA/Table/One | `internal/chunker/{general,title,others}.go` | 9 |
| 全文分词（二元组/单字） | `internal/tokenize/terms.go` | 10 |
| 模型注册表：mock / OpenAI 兼容 embedding | `internal/llm/` | 10 |
| 流水线：DSL、模板、执行器、组件、缓存 | `internal/pipeline/` | 7、10 |
| chunk 表新增分词和向量列，旧库自动加列 | `internal/docengine/engine.go` | 4、11 |
| `Runner` 改为执行流水线；进度写库限频 | `internal/ingest/runner.go` | 6、7 |
| 接口：模板列表、模型列表；编辑 chunk 后重新分词和向量化 | `internal/api/` | |
| 前端：数据集配置面板、组件图、每个文档单独选模板、显示索引字段 | `web/src/pages/{Settings,PipelineView}.tsx` | |

依赖只多了一个 `golang.org/x/net`（解析 HTML）。PDF 依赖系统里的 `pdftotext`。

## 2. 实现顺序

1. **parser**：先定 `Section` 结构和"标题一律写成 markdown 形式"这条约定，再逐个格式实现。测试用 pandoc 和 headless Chrome 从同一份 markdown 生成的真实 docx/html/pdf（第 8 章）。
2. **chunker**：纯函数，按 `MergeUnits → General → Title → QA/Table/One` 的顺序写，每个配一个基于 `samples/` 的测试。
3. **tokenize/terms.go** 和 **llm**：mock embedder 先行，OpenAI 兼容的实现用 `MR_OLLAMA=1` 才会运行的测试来验证。
4. **pipeline**：DSL 类型 → `Order` → `WithOverrides` → `Run` → 模板 JSON → 组件注册 → Tokenizer → 缓存。
5. 把 **ingest.Runner** 改成调用 `pipeline.Run`，chunk id 的去重逻辑保留。
6. **docengine** 加列，并实现"旧库自动迁移"：`pragma_table_info` 查到缺哪一列，就 `ALTER TABLE ADD COLUMN` 哪一列。

## 3. 跑起来

```bash
cd backend && go test ./... && cd ../web && npm run build
cd ../backend && go run ./cmd/mini-ragflow --data /tmp/v02 --static ../web/dist &
cd ../web && node e2e/ui-v02.mjs http://localhost:9380 /tmp/shots ../samples
```

脚本做了两件事：

1. 新建"公司制度"数据集，在配置面板里把模板改成"法规"，上传《员工休假制度》，解析，打开分块页（第 9 章图 9-1）；
2. 新建"产品资料"数据集，上传 `samples/formats/` 下的 PDF、DOCX、CSV，把 DOCX 改成"书籍"、CSV 改成"问答"，然后全部解析。

真实输出：

```text
laws chunks: 共 14 个分块
--- .pdf log:
Task claimed by ingestor-a4a985c2/1.
Start general pipeline for 星河咖啡机用户手册.pdf (attempt 1).
Parsed as pdf: 2 sections map[text:2].
Chunked by general into 2 chunks.
Embedded 2 chunks with mock-embedding@mock (dim 256, 0 cache hits).
Indexed 2 chunks, 750 tokens in 0.02s.
Done.
--- .docx log:
Task claimed by ingestor-a4a985c2/1.
Start book pipeline for 星河咖啡机用户手册.docx (attempt 1).
Parsed as docx: 23 sections map[list:7 table:1 text:7 title:8].
Chunked by title into 7 chunks.
Embedded 7 chunks with mock-embedding@mock (dim 256, 0 cache hits).
Indexed 7 chunks, 830 tokens in 0.00s.
Done.
--- .csv log:
Task claimed by ingestor-a4a985c2/1.
Start qa pipeline for 售后问答.csv (attempt 1).
Parsed as spreadsheet: 3 sections map[table_row:3].
Chunked by qa into 3 chunks.
Embedded 3 chunks with mock-embedding@mock (dim 256, 0 cache hits).
Indexed 3 chunks, 86 tokens in 0.00s.
Done.
```

### 换成真实的 embedding

```bash
go run ./cmd/mini-ragflow --data /tmp/v02 --openai-base-url http://localhost:11434/v1
```

在数据集配置里选 `nomic-embed-text@openai`，解析两次，第二次全部命中缓存（第 10 章）。

注意：mini-ragflow 不允许已经有 chunk 的数据集更换 embedding 模型，接口会返回 `If chunk count is not 0, embedding model cannot be changed.`。不同模型的向量维度和语义空间都不同，混在一起就没法比较了。RAGFlow 当前的做法更灵活：它允许更换，但提供了一个检查接口 `POST /datasets/{id}/embedding/check`（`router.go:341`）。`DatasetService.CheckEmbedding`（`internal/service/dataset/embedding.go:38`）随机抽取一批已有 chunk，用新模型重新编码（包括"0.1×标题 + 0.9×正文"的混合方式），再和库里已有的向量算余弦，以此判断新模型是否和原模型兼容（例如同一个模型换了一个提供商）。维度不一样时直接报错。

## 4. 检查点

- [ ] 同一份手册：`general` 4 块，`book` 7 块，`chunk_token_num=64` 的 `general` 14 块；
- [ ] 法规模板把《员工休假制度》切成 14 块，每块以"第X章 …\n第Y条"开头；
- [ ] 问答模板下，FAQ 的 `content_ltks` 只有问题的分词；
- [ ] 在 chunk 页编辑一块内容，保存后 `content_ltks` 随之更新；
- [ ] 用 v0.1 的数据目录启动 v0.2，旧 chunk 表会自动加列，服务正常启动。

## 5. 本版的踩坑汇总

| 现象 | 原因 | 处理 | 章节 |
|---|---|---|---|
| PDF 中文全是乱码 | 纯 Go 库不支持 CID 字体 | 改用 `pdftotext` | 8 |
| PDF 句子被拆成两行 | 文本层的硬换行 | "满行续接"启发式 | 8 |

## 6. 局限（下一版解决）

chunk 已经带上了分词和向量，但还**没有检索**。v0.3 会写一个检索引擎：倒排索引、向量扫描、两种融合方式，以及 RAGFlow 式的两阶段重排。

```bash
git diff v0.2 v0.3 --stat | tail -1
```

```text
 24 files changed, 1786 insertions(+), 23 deletions(-)
```
