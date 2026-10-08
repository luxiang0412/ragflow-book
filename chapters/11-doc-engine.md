# 第 11 章 文档引擎抽象：DocEngine 与索引 schema

> 本章目标：理解 RAGFlow 怎么把"检索引擎"抽象成一个接口，从而能在 Elasticsearch、Infinity、OceanBase 之间切换；然后实现 mini-ragflow 自己的引擎：内存倒排索引加向量扫描。
> 前置：第 4、10 章。对应代码：`v0.3` 的 `backend/internal/docengine/search.go`（`v1.0` 有一处评分修正）。

## 11.1 为什么要抽象

RAGFlow 支持的检索引擎有 Elasticsearch、OpenSearch、Infinity（InfiniFlow 自家的数据库）、OceanBase、SeekDB 等等（第 2 章）。它们的查询语言完全不同：ES 用 JSON DSL，Infinity 用自己的 SDK 和表达式，OceanBase 走 SQL。如果检索代码里到处都是 ES 的 JSON，换引擎就等于重写。

## 11.2 RAGFlow 的 DocEngine

`internal/engine/engine.go:41`：

```go
type DocEngine interface {
	// Chunk operations
	CreateChunkStore(ctx context.Context, baseName, datasetID string, vectorSize int, parserID string) error
	InsertChunks(ctx context.Context, chunks []map[string]interface{}, baseName string, datasetID string) ([]string, error)
	UpdateChunks(ctx context.Context, condition map[string]interface{}, newValue map[string]interface{}, baseName string, datasetID string) error
	DeleteChunks(ctx context.Context, condition map[string]interface{}, baseName string, datasetID string) (int64, error)
	Search(ctx context.Context, req *types.SearchRequest) (*types.SearchResult, error)
	GetChunk(ctx context.Context, baseName, chunkID string, datasetIDs []string) (interface{}, error)
	DropChunkStore(ctx context.Context, baseName, datasetID string) error
	ChunkStoreExists(ctx context.Context, baseName, datasetID string) (bool, error)
	...
	KNNScores(ctx context.Context, chunks []map[string]interface{}, queryVector []float64, topK int) (map[string]interface{}, error)
	GetScores(searchResult map[string]interface{}) map[string]float64
	...
```

几个要点：

- **chunk 是 `map[string]interface{}`**。字段名约定（第 4 章的后缀规则）就是"schema"，每个引擎各自把它翻译成自己的类型系统；
- **`baseName` 是租户级的索引名**。`buildIndexNames`（`internal/service/nlp/retrieval.go:1287`）把租户 id 变成 `ragflow_<tenant_id>`。一个租户的所有数据集共用一个索引，用 `kb_id` 字段区分；
- **`Search` 接收一个与引擎无关的 `SearchRequest`**，里面是几种"匹配表达式"：`MatchTextExpr`（全文）、`MatchDenseExpr`（向量）、`FusionExpr`（融合方式），再加上过滤条件；
- **`KNNScores`** 用来对一批已经召回的 chunk 重新计算向量分数（第 13 章的第二阶段打分会用到）。

### 写入：refresh 的时机

ES 写入的文档默认要等 refresh（`conf/mapping.json` 设置为 1 秒）之后才能被搜到。入库时如果每批都强制 refresh，会很慢。`insertChunksForIngestion`（`internal/ingestion/task/pipeline_executor.go:119`）的做法是：**前面的批次走不 refresh 的写入路径，最后一批正常写入并等待 refresh**。这样，当任务被 ack、文档状态变成"完成"时，它的所有 chunk 都已经可以被搜到：

```go
// Refreshing every batch adds unnecessary latency. The writer uses the regular
// inserter for its final batch, so all preceding batches become searchable
// before the document task is acknowledged and its completion event is sent.
func insertChunksForIngestion(eng engine.DocEngine) InsertFunc {
	return func(ctx context.Context, chunks []map[string]any, baseName string, datasetID string) ([]string, error) {
		if bulk, ok := eng.(noRefreshChunkInserter); ok {
			return bulk.InsertChunksNoRefresh(ctx, chunks, baseName, datasetID)
		}
		return eng.InsertChunks(ctx, chunks, baseName, datasetID)
	}
}
```

`noRefreshChunkInserter` 是一个**可选接口**：支持的引擎实现它，不支持的照常写入。用 Go 的接口断言给抽象层"打补丁"，是 RAGFlow 代码里反复出现的手法。

### ES 怎么执行混合查询

ES 版的 `Search`（`internal/engine/elasticsearch/chunk.go`）把三种表达式拼成一个查询。最关键的一段从 `:1249` 开始：全文查询放进 bool query，然后把**整个 bool query 当成 kNN 的 `filter`**：

```go
if matchText != nil {
	textQuery := buildQueryStringQuery(matchText, isSkillIndex, isMemoryIndex)
	...
			if hasVectorMatch {
				boolMap["boost"] = 1.0 - vectorSimilarityWeight
			}
	...
}

if hasVectorMatch {
	...
	knnQuery := map[string]interface{}{
		"field":          vectorFieldName,
		"query_vector":   matchDense.EmbeddingData,
		"k":              k,
		"num_candidates": numCandidates,
		"similarity":     similarity,
		"filter":         boolQuery,
	}
```

意思是：**在满足全文条件的 chunk 里，找向量最近的 k 个**。这是理解 RAGFlow 检索的一把钥匙，第 13 章会细讲它带来的后果。

### Infinity 怎么做

Infinity 原生支持"全文 + 向量 + 融合"一体的查询，RAGFlow 直接把 `FusionExpr{Method: "weighted_sum", weights: "0.7,0.3"}` 交给它，由引擎在内部完成两路召回和加权求和（`buildInfinityFusionExpr`，`retrieval.go:601`）。所以同一个 `SearchRequest`，在 ES 上是"全文过滤 + kNN"，在 Infinity 上是"两路召回 + 加权融合"。抽象层统一了**接口**，但没有统一**语义**。

### 字段的评分方式

第 4 章提到，ES 的 mapping 里 `*_tks` 字段（标题、关键词、问题）用的是自定义相似度 `scripted_sim`：**归一化 IDF × min(词频, 1)**。只看词在不在，不看出现几次，也不做长度归一。`*_ltks`（正文）用 ES 默认的 BM25。

## 11.3 从零实现

### 接口

mini-ragflow 的 `docengine` 包分成两个接口。`Engine`（v0.1 起就有）负责增删改查；`Searcher`（v0.3 新增）负责检索：

```go
type Searcher interface {
	Search(ctx context.Context, req *SearchRequest) ([]Hit, error)
	// DocFreq returns how many chunks in the KBs contain term in any
	// full-text field, and the total chunk count — the statistics the query
	// side needs for IDF term weights.
	DocFreq(ctx context.Context, kbIDs []string, terms []string) (map[string]int, int, error)
}

type SearchRequest struct {
	KbIDs  []string
	DocIDs []string        // optional scope
	Text   *MatchText      // 全文：带权重的词 + 字段 boost + minimum_should_match
	Dense  *MatchDense     // 向量：查询向量 + topK + 余弦下限
	Fusion string          // "filter_knn"（ES 式）或 "weighted_sum"（Infinity 式）
	TextWeight, VectorWeight float64
	Limit                    int
	IncludeUnavailable       bool
}
```

两种融合方式都实现了，并且可以在检索测试页上切换对比（第 14 章）。

### 倒排索引

每个数据集一个内存索引，第一次检索时从 chunk 表加载：

```go
type kbIndex struct {
	version  int64
	docs     []*docEntry
	df       map[string]int            // term → 含该词的 chunk 数（任意字段）
	fieldDF  map[string]map[string]int // field → term → chunk 数
	avgLen   map[string]float64        // field → 平均长度（BM25 用）
	postings map[string][]int          // term → chunk 下标
}
```

可检索的字段有五个：`title_tks / content_ltks / content_sm_ltks / important_tks / question_tks`。

### 什么时候重建：版本号

API 进程里的内存索引怎么知道 ingestor 进程刚写了新 chunk？mini-ragflow 用一张 `chunk_version(kb_id, version)` 表：每次 `InsertChunks / DeleteChunks / DropKB` 都把版本号加 1，检索前先读一下版本号，和内存里的不一样就重建。读一行整数的开销，比每次检索都扫一遍全表小得多。这和 ES 的 refresh 是同一个问题：**写入之后，读取端什么时候能看见**。

### 评分

每个字段单独计算分数，取**最好的那个字段**乘以它的 boost（相当于 ES 的 `best_fields` / `dis_max`）：

```go
func bm25(idx *kbIndex, d *docEntry, q *MatchText) (score float64, matched int) {
	...
	for f, boost := range q.Fields {
		...
		for _, qt := range q.Terms {
			c := float64(tf[qt.Term])
			if c == 0 {
				continue
			}
			df := float64(idx.fieldDF[f][qt.Term])
			idf := math.Log(1 + (n-df+0.5)/(df+0.5))
			if presenceField(f) {
				// idf / idf_max · min(freq, 1), the scripted_sim formula.
				s += qt.Weight * idf / math.Log(1+(n-0.5)/1.5)
				continue
			}
			norm := c * (bm25K1 + 1) / (c + bm25K1*(1-bm25B+bm25B*float64(d.len[f])/idx.avgLen[f]))
			s += qt.Weight * idf * norm
		}
		if s*boost > score {
			score = s * boost
		}
	}
	return score, matched
}
```

#### 一次对照中发现的差距

v0.3 写好时，所有字段都用 BM25。写本书第 4 章、对照 `conf/mapping.json` 时才发现 `*_tks` 字段用的是 `scripted_sim`。差别在于：一个标题里同一个词重复三次，BM25 会给更高的分，`scripted_sim` 不会。v1.0 补上了 `presenceField` 这个分支，并加了回归测试 `TestTitleFieldsScorePresenceOnly`：

```go
tks := score("title_tks")
if tks["once"] == 0 || tks["once"] != tks["thrice"] {
	t.Fatalf("title_tks should ignore repetition: %v", tks)
}
ltks := score("content_ltks")
if !(ltks["body3"] > ltks["body1"]) {
	t.Fatalf("content_ltks should reward frequency: %v", ltks)
}
```

### 向量与融合

向量检索就是对候选集逐个算余弦，谈不上什么 ANN 索引。数据集规模在几万个 chunk 以内时，暴力扫描完全够用。

两种融合方式：

- **filter_knn**（ES 式）：候选 = 命中了全文条件的 chunk；排序 = 余弦；余弦低于下限的丢弃；
- **weighted_sum**（Infinity 式）：候选 = 所有 chunk；分数 = `TextWeight × 归一化 BM25 + VectorWeight × 余弦`。只要全文命中或者余弦达到下限，就保留。

`MinShouldMatch` 对应 ES `query_string` 的 `minimum_should_match`：至少要命中多少比例的"必需词"。必需词和可选词的区别是第 12 章的内容。

### 与 ES 的差异

| 项 | ES | mini docengine |
|---|---|---|
| 规模 | 分片、副本 | 单进程内存 |
| 向量 | HNSW 近似最近邻 | 暴力余弦 |
| 查询语言 | `query_string` 语法 | 结构化的 `MatchText` |
| 可见性 | refresh 间隔 | 版本号比对 |
| 多字段 | `best_fields` | 取最好的字段 × boost |

## 11.4 运行与验证

```bash
git checkout v1.0 && cd backend && go test ./internal/docengine/ -v | grep -E '^(---|ok)'
```

```text
--- PASS: TestTitleFieldsScorePresenceOnly (0.01s)
ok  	mini-ragflow/internal/docengine	0.011s
```

**旧库升级**：用 v0.3 的程序打开 v0.1 留下的数据目录。chunk 表会自动补上新列，但旧 chunk 没有分词结果，所以在重新解析之前什么也搜不到（真实输出）：

```text
{"code":0,"message":"success","data":{"chunks":[],"doc_aggs":[],"total":0,"keywords":["除垢"],
 "trace":["pass 1: kNN filtered by full-text, min_match=0.3, 0 hits",
          "pass 2 (relaxed): min_match=0.1, cosine>=0.17, 0 hits",
          "pass 3 (dense only): 0 hits"]}}
```

重新解析之后：

```text
2 ['pass 1: kNN filtered by full-text, min_match=0.3, 2 hits'] 当除垢指示灯亮起时（通常每 200 杯或每 2 个月），请使用专用除垢剂……
```

这和 RAGFlow 的行为一致：更换切分或分词方式之后，旧 chunk 必须重新解析，因为索引字段是入库时算出来的。

## 11.5 练习

1. **巩固**：为什么 RAGFlow 一个租户只用一个索引，而不是一个数据集一个索引？各有什么利弊？（提示：ES 的分片数量、跨数据集检索。）
2. **扩展**：为 mini-ragflow 写一个 Elasticsearch 版的 `Searcher`：`MatchText` 翻译成 `query_string`，`MatchDense` 翻译成 `knn`，`filter_knn` 模式下把全文查询放进 `knn.filter`。
3. **读源码**：读 `internal/engine/infinity/chunk.go` 的 `Search`，找出 Infinity 怎么表达"全文 + 向量 + 加权融合"，并和 ES 版对比 `vector_similarity_weight` 在两边分别起什么作用。

## 11.6 延伸阅读

- `internal/engine/engine.go`、`internal/engine/types/`：请求与结果类型
- `internal/engine/elasticsearch/chunk.go`：ES 查询构造的全部细节
- Elasticsearch 文档：kNN search 的 `filter` 参数，以及 `query_string` 的 `minimum_should_match`
