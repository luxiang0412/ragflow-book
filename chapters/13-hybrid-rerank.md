# 第 13 章 混合检索与重排 ★★

> 本章目标：彻底弄懂 RAGFlow 一次检索的两个阶段：引擎召回候选，内存里重新打分；理解 `similarity_threshold`、`vector_similarity_weight`、rerank 模型、pagerank 各自在哪一步起作用。然后实现它，并修复一处对照时发现的问题。
> 前置：第 11、12 章。对应代码：`v0.3` 的 `backend/internal/retrieval/retrieval.go`、`similarity.go`。

## 13.1 两个阶段

```mermaid
flowchart TB
    Q[问题] --> QB[查询构造<br/>带权重的词 + 查询向量]
    QB --> S1{第一阶段：引擎召回<br/>取 64 个候选}
    S1 -->|0 个| S2[放宽：min_match 0.3→0.1<br/>余弦下限→0.17]
    S2 -->|0 个| S3[只用向量]
    S1 --> R
    S2 --> R
    S3 --> R
    R[第二阶段：重新打分<br/>sim = (1-w)·tsim + w·vsim + rank特征]
    R --> T[阈值过滤 sim ≥ 0.2]
    T --> P[排序、分页、按文档聚合]
```

为什么要分两个阶段？引擎擅长**在大量数据里快速找候选**，但它的打分（BM25、余弦）不一定符合业务需要；内存里的重排只处理几十个候选，可以用更精细、更"贵"的打分方式，比如一个 cross-encoder 模型。

## 13.2 默认值

`RetrievalService.Retrieval`（`internal/service/nlp/retrieval.go:94`）开头先补齐默认值：

| 参数 | 默认 | 位置 | 作用 |
|---|---|---|---|
| `KNNTopK` | 1024 | `:102` | kNN 取多少个近邻 |
| `KNNNumCandidates` | 2048 | `:105` | HNSW 搜索的候选池 |
| `SimilarityThreshold` | 0.2 | `:108` | 第二阶段的分数下限 |
| `VectorSimilarityWeight` | 0.3（纯向量检索时为 1.0） | `:110` | 第二阶段里向量相似度的权重 |
| `RankFeature` | `{"pagerank_fea": 10}` | `:122` | 排序特征 |
| `RerankCandidatesCount` | 64 | `:135` | 第一阶段取多少个候选 |

默认的向量权重是 **0.3**，也就是说关键词相似度占 70%。这是一个偏保守的选择：在中文和专业术语场景下，字面命中往往比语义相似更可靠。

还有一条分页约束：`page × page_size` 不能超过候选数 64。因为第二阶段只在这 64 个候选里排序，第 65 名以后的结果根本不存在。配置了 rerank 模型时只允许取第一页。

## 13.3 第一阶段：引擎召回

`RetrievalService.Search`（`:671`）构造请求：全文表达式 + 向量表达式 + 融合表达式。融合方式取决于引擎（`buildRetrievalFusionExpr`，`:617`）：

```go
func buildRetrievalFusionExpr(docEngineType string, topn int, vectorSimilarityWeight *float64) *types.FusionExpr {
	if docEngineType == string(engine.EngineInfinity) {
		return buildInfinityFusionExpr(topn, vectorSimilarityWeight)
	}

	// Matches rag/nlp/search.py:331 — every backend but Infinity takes this fixed
	// pair, so the first search is a vector recall pass and the caller's
	// vector_similarity_weight is applied afterwards, by RerankWithKNN
	// (tkWeight = 1-vw, vtWeight = vw). Feeding it in here instead ranks the
	// window by BM25 and cuts a different top-N than the reference does.
	return &types.FusionExpr{
		Method: "weighted_sum",
		TopN:   topn,
		FusionParams: map[string]interface{}{
			"weights": esFusionWeights,
		},
	}
}

const esFusionWeights = "0.001,1"
```

结合第 11 章 ES 的查询构造（全文查询放进 `knn.filter`，`boost = 1 − 向量权重`），在 ES 上第一阶段做的事情是：

> **在满足全文条件（`minimum_should_match`）的 chunk 中，按向量余弦取前 N 个。**

此时全文的权重被压到 0.001，几乎只起"过滤"作用；用户设置的 `vector_similarity_weight` 要到第二阶段才生效。

### 搜不到就放宽

`Search` 在结果为空时会重试（`:842` 起）：

```go
// No doc_id filter — retry with lower min_match (0.1 vs default 0.3)
// and lower vector similarity threshold (0.17 vs default 0.1-0.2).
// This provides a second chance for queries that were too strict
// on the first attempt.
matchText, _ := GetQueryBuilder().Question(req.Question, "qa", minMatch(req.VectorSimilarityWeight, 0.1))
matchDense = cloneDenseExpr(denseTemplate)
matchDense.ExtraOptions["similarity"] = 0.17
...
// Zero-only by design: any lexical hit keeps the existing hybrid candidate semantics.
if engineResult.Total == 0 && matchText != nil && *req.AllowDenseFallback {
	common.Debug("Retrieval dense-only fallback after empty hybrid retries")
	...
	searchRequest.MatchExprs = []any{matchDense}
```

第一次：`min_match = 0.3`；第二次：`min_match = 0.1`，余弦下限 0.17；第三次：完全不要全文条件，只按向量找。

## 13.4 第二阶段：重新打分

`scoreSearchResult`（`:450`）按引擎和配置分成几条路：

| 情况 | 打分方式 |
|---|---|
| 配置了 rerank 模型 | `RerankByModel`：`sim = (1-w)·tsim + w·rerank分数` |
| Infinity | 直接用引擎的融合分数 |
| OceanBase | `RerankStandard`：本地算 `tsim` 和余弦 |
| 纯向量请求 | 直接用引擎的余弦 |
| ES（默认） | `KNNScores` 让引擎重算余弦，再 `RerankWithKNN` |

最常见的 ES 路径是 `RerankWithKNN`（`internal/service/nlp/reranker.go:913`）：

```go
// Combine tokens with weights: content + title*2 + important_kwd*5 + question_tks*6
// This matches Python: tks = content_ltks + title_tks * 2 + important_kwd * 5 + question_tks * 6
tks := make([]string, 0, len(contentLtks)+len(titleTks)*2+len(importantKwd)*5+len(questionTks)*6)
tks = append(tks, contentLtks...)
for i := 0; i < 2; i++ {
	tks = append(tks, titleTks...)
}
...
tsim = TokenSimilarity(keywords, insTw, qb)
...
for i := range tsim {
	sim[i] = tkWeight*tsim[i] + vtWeight*vsim[i]
}
// Apply rank feature scores (tag_score * 10 + pagerank)
sim = applyRankFeatureScoresForIDs(ids, field, sim, rankFeature)
```

### 词相似度 tsim

`tsim` 不是 BM25，而是一个更简单的"覆盖率"。先把查询词变成一个带权字典（`tokensToDict`，`:429`）：

```go
for i, tw := range wts {
	t := tw.Term
	c := tw.Weight
	...
	d[t] += c * 0.4                 // 单个词：40% 的权重
	if i+1 < len(wts) {
		_t := wts[i+1].Term
		_c := wts[i+1].Weight
		k := t + _t
		...
		d[k] += math.Max(c, _c) * 0.6   // 相邻词对：60% 的权重
	}
}
```

再算"查询字典里有多大比例的权重出现在 chunk 里"（`tokenDictSimilarity`，`:461`）：

```go
s := 1e-9
for _, t := range keys {
	if _, ok := dtwt[t]; ok {
		s += qtwt[t]
	}
}
q := 1e-9
for _, t := range keys {
	q += qtwt[t]
}
result := s / q
```

两个观察：

1. **相邻词对占 60%**。"年假 天数"和"天数 年假"的单词覆盖率一样，但只有前者能命中"年假天数"这个词对。所以 `tsim` 对词序敏感，可以算作一种廉价的短语匹配；
2. **chunk 这一侧只看 key 在不在**（`if _, ok := dtwt[t]`），不看权重。所以上面 `title_tks * 2`、`important_kwd * 5`、`question_tks * 6` 的重复，**对结果几乎没有影响**：重复一个列表并不会让它的 key 变多，唯一的作用是在拼接处多出几个"跨字段的词对"。字段的重要性真正起作用的地方，是第一阶段 `query_string` 里的字段 boost（第 12 章）。

### 排序特征

`applyRankFeatureScores`（`:660`）在 `sim` 上**直接加**：

```text
sim += tag_score × 10 + pagerank
```

`pagerank` 是数据集上配置的一个整数（0~100），写进每个 chunk 的 `pagerank_fea` 字段。它的量级远大于 `sim`（0~1），所以效果是"pagerank 高的数据集的结果整体排在前面"。多个数据集一起检索时，可以用它表达"官方文档优先于论坛帖子"。`tag_score` 来自标签特征（第 19 章）。

### 阈值

```go
// When vector_similarity_weight is 0, similarity_threshold is not meaningful for term-only scores
postThreshold := *req.SimilarityThreshold
if *req.VectorSimilarityWeight <= 0 {
	postThreshold = 0.0
}
...
if req.VectorOnly {
	postThreshold = 0.0
}
```

阈值作用在**第二阶段的综合分数**上。纯关键词（向量权重为 0）和显式的纯向量请求不设阈值。

### rerank 模型

配置了 rerank 模型（cross-encoder）时，`RerankByModel`（`:92`）把"问题 + 每个 chunk 的原文"一起送进模型打分，用这个分数**代替**向量相似度：

```go
modelSim = NormalizeRerankScores(modelSim)

sim = make([]float64, len(insTw))
for i := range tsim {
	sim[i] = tkWeight*tsim[i] + vtWeight*modelSim[i]
}
```

`NormalizeRerankScores`（`:225`）保留已经在 [0,1] 区间里的分数（很多 rerank 服务输出的是校准过的概率），只对超出范围的分数做 min-max 归一化。

## 13.5 从零实现

`backend/internal/retrieval/retrieval.go` 的 `Retrieve` 按同样的结构写，并在每一步留下 `Trace`，检索测试页会把它显示出来：

```go
hits, err := r.Engine.Search(ctx, sreq)
res.Trace = append(res.Trace, fmt.Sprintf("pass 1: %s, min_match=%.1f, %d hits", fusionName(sreq), q.Text.MinShouldMatch, len(hits)))
if len(hits) == 0 && len(req.DocIDs) == 0 && qvec != nil {
	q2, err := Build(ctx, r.Engine, req.KbIDs, req.Question, minMatch(0.1))
	...
	sreq.Dense.Similarity = 0.17
	...
	if len(hits) == 0 {
		sreq.Text = nil
		...
		res.Trace = append(res.Trace, fmt.Sprintf("pass 3 (dense only): %d hits", len(hits)))
		denseOnly = true
	}
}
```

第二阶段：

```go
for i, h := range hits {
	tsim[i] = TokenSimilarity(q.Keywords, q.Weights, chunkTerms(h.Chunk))
	if qvec != nil {
		vsim[i] = docengine.Cosine(qvec, h.Chunk.Vector)
	}
}
...
sim[i] = (1-vw)*tsim[i] + vw*vsim[i]
sim[i] += req.Pagerank[hits[i].Chunk.KbID]
```

`TokenSimilarity`（`similarity.go`）是 `tokensToDict + tokenDictSimilarity` 的移植。`chunkTerms` 把四个字段各拼一次，没有重复，理由见 13.4 节的第二个观察。

rerank 模型支持两种：`mock-rerank@mock`（按查询二元组的覆盖率打分），以及 OpenAI 兼容服务里常见的 `POST /rerank`（Jina、Cohere 风格，vLLM、Xinference、TEI 都支持）。

### 对照中发现的问题：纯向量兜底被阈值"吃掉"了

用真实模型测试跨语言检索时，出现了一个奇怪的结果（`nomic-embed-text`，问题是英文，语料是中文）：

```text
Q: descaling steps  total=0  pass 1: kNN filtered by full-text, min_match=0.3, 0 hits | pass 2 (relaxed): min_match=0.1, cosine>=0.17, 0 hits | pass 3 (dense only): 26 hits
```

第三阶段的纯向量兜底召回了 26 个候选，最终结果却是 0。原因是：既然没有一个 chunk 命中全文，`tsim` 就几乎全是 0，于是 `sim = 0.7 × 0 + 0.3 × vsim ≤ 0.3 × vsim`。要过 0.2 的阈值，`vsim` 必须大于 0.667，这对跨语言检索来说几乎不可能。

回到 RAGFlow 的代码核对：`Retrieval` 只对**显式的** `VectorOnly` 请求把阈值清零（13.4 节"阈值"），兜底路径打分时仍然用 `(1-w)·tsim + w·vsim`。所以 RAGFlow 当前的代码在同样的情况下，也会把兜底结果过滤掉。

mini-ragflow 的处理是：**兜底结果按向量分数本身打分**。理由是，所有候选都来自向量这一路，而且它们恰恰是因为全文没有命中才进来的，再用 `tsim` 去惩罚它们，等于把兜底这一步作废了：

```go
case denseOnly && req.Reranker == nil:
	// Every candidate came from the vector leg because no chunk matched
	// the text at all, so tsim is ~0 by construction. Weighting it would
	// cap sim at w·vsim (0.3·vsim by default) and the threshold would
	// throw the whole fallback away. Score the fallback by its own leg,
	// as RAGFlow already does for explicit vector-only requests.
	sim[i] = vsim[i]
```

回归测试 `TestDenseFallbackIsScoredByVector` 用一个"返回固定向量"的假 embedder，构造出"全文 0 命中、向量完全命中"的场景。修复前测试失败：

```text
--- FAIL: TestDenseFallbackIsScoredByVector (0.01s)
    retrieval_test.go:226: dense fallback lost: total=1 trace=[... pass 3 (dense only): 5 hits]
```

修复后通过。用真实模型复测，同一个问题不再是 0 个结果，但排序并不理想：

```text
nomic-embed-text:  total=26  0.445 ## 2. 首次使用 / 0.442 ## 5. 清洁与除垢 / 0.427 ## 6. 故障排除
bge-m3:            total=26  0.478 ## 4. 打奶泡 / 0.463 Q: 推荐使用什么研磨度？ / 0.462 第二章 年假 第五条
```

`nomic` 至少把"清洁与除垢"排到了第二；`bge-m3` 的前三名一个相关的都没有。修复让兜底结果"能出来"，但单个英文词的跨语言检索本来就很弱。RAGFlow 为此提供了 `cross_languages` 选项：先让 LLM 把问题翻译成目标语言，再去检索（`chat_pipeline.go` Phase 8）。

### 两种 embedding 模型的真实对比

同样四个问题，同样的数据集，分别用 `nomic-embed-text` 和 `bge-m3`（真实输出，每个问题列出前两名）：

```text
=== nomic-embed-text@openai
Q: 咖啡机怎么除垢？  total=8
  0.615 (t 0.576 v 0.708)  ## 7. 保修 整机保修 2 年，……未按说明除垢导致的水垢堵塞不在保修范围内
  0.602 (t 0.576 v 0.661)  ## 5. 清洁与除垢 每天使用后清洗手柄和滤网。……
Q: 牛奶要加热到多少度  total=2
  0.279 (t 0.136 v 0.614)  ## 7. 保修 整机保修 2 年，加热组件保修 3 年。……
  0.276 (t 0.136 v 0.604)  ## 4. 打奶泡 将蒸汽旋钮转到蒸汽档，……

=== bge-m3@openai
Q: 咖啡机怎么除垢？  total=8
  0.631 (t 0.576 v 0.759)  ## 5. 清洁与除垢 每天使用后清洗手柄和滤网。……
  0.580 (t 0.576 v 0.589)  ## 7. 保修 整机保修 2 年，……
Q: 牛奶要加热到多少度  total=2
  0.288 (t 0.136 v 0.644)  ## 4. 打奶泡 将蒸汽旋钮转到蒸汽档，……
  0.213 (t 0.136 v 0.394)  ## 7. 保修 整机保修 2 年，加热组件保修 3 年。……
```

两个问题里，两个候选的 `tsim` 都一模一样：都只命中了查询里的同一批词（`tsim` 只看词在不在，不看出现几次）。这时排序完全由 `vsim` 决定。`nomic-embed-text` 给"保修"一节打的向量分比"清洁与除垢"还高，`bge-m3` 则排对了。第 1 章的"余弦区分度"数据（0.726/0.711/0.719 对比 0.654/0.335/0.509）说的就是这件事。

> 题外话：`bge-m3` 在作者机器的 GPU（RTX 2080 Ti）上，对超过约 100 个汉字的输入返回 `HTTP 500: json: unsupported value: NaN`，换成纯 CPU 的 Ollama 实例就正常了（疑似 fp16 溢出）。mini-ragflow 当时正确地把这份文档标成了"失败"，并在日志里写出了错误原因。这件事也提醒我们：embedding 服务的失败必须让用户看得见。

## 13.6 运行与验证

```bash
git checkout v0.3 && cd backend && go test ./internal/retrieval/ -v | grep -E '^(---|ok)'
```

```text
--- PASS: TestTopHits (0.01s)
--- PASS: TestScoreFormulaAndHighlight (0.01s)
--- PASS: TestThresholdAndKeywordOnly (0.01s)
--- PASS: TestRelaxedRetryAndDenseFallback (0.01s)
--- PASS: TestWeightedSumFusionAndDocFilter (0.01s)
--- PASS: TestPaginationGuardAndRerank (0.01s)
--- PASS: TestTokenSimilarity (0.00s)
--- PASS: TestDenseFallbackIsScoredByVector (0.01s)
ok  	mini-ragflow/internal/retrieval	0.061s
```

`TestRelaxedRetryAndDenseFallback` 用一个"前两次故意返回空"的 Searcher 记录每次请求的参数，断言三次检索分别是 `(0.3, 0.2) → (0.1, 0.17) → 只有向量`。`TestTokenSimilarity` 验证了"词对占 60%"：查询"年假 天数"、chunk 里两个词都有但不相邻时，`tsim = 0.4 / 0.7 ≈ 0.571`。

## 13.7 练习

1. **巩固**：用检索测试页（下一章）把向量权重从 0.3 调到 0.9，观察"咖啡机怎么除垢"的排序变化，并用 13.4 节的公式解释。
2. **扩展**：实现 RRF（Reciprocal Rank Fusion）作为第三种融合方式：`score = Σ 1/(60 + rank_i)`。和 `weighted_sum` 相比，它不需要把两路分数归一化到同一个尺度。
3. **读源码**：`RetrievalByChildren`（`retrieval.go:1025`）做了什么？什么样的切分配置会产生"子 chunk"？

## 13.8 延伸阅读

- `internal/service/nlp/retrieval.go`、`reranker.go`
- Cormack et al., *Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods*, 2009
- BGE-M3 论文：*M3-Embedding: Multi-Linguality, Multi-Functionality, Multi-Granularity*
