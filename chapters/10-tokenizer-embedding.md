# 第 10 章 分词与向量化：Tokenizer 组件

> 本章目标：弄清一个 chunk 入库前要算出哪些"检索字段"，包括全文检索用的分词串和向量检索用的 embedding；理解标题向量加权、截断和缓存，并实现 mini-ragflow 的 Tokenizer 组件。
> 前置：第 9 章。对应代码：`v0.2` 的 `backend/internal/tokenize/terms.go`、`backend/internal/llm/`、`backend/internal/pipeline/tokenizer.go`。

## 10.1 一个 chunk 入库前长什么样

切分器交给 Tokenizer 的只有一段文本。Tokenizer 交给检索引擎时，这段文本已经变成了一组字段：

| 字段 | 内容 | 给谁用 |
|---|---|---|
| `content_with_weight` | 原文 | 展示、拼提示词 |
| `content_ltks` | 正文分词（粗粒度），空格连接 | 全文检索（BM25） |
| `content_sm_ltks` | 正文分词（细粒度） | 全文检索，召回单字查询 |
| `title_tks` / `title_sm_tks` | 文件名分词 | 全文检索，权重很高（×10） |
| `important_kwd` / `important_tks` | 关键词（原文 / 分词） | 全文检索（×30 / ×20） |
| `question_kwd` / `question_tks` | 问题（原文 / 分词） | 全文检索（×20） |
| `q_<维度>_vec` | 向量 | kNN 检索、重排 |

字段名的后缀决定了它在 ES 里的类型（第 4 章 4.2 节）。

## 10.2 分词：RAGFlow 的做法

### 两种粒度

`tokenizeChunks`（`internal/ingestion/component/tokenizer.go:912`）对每个 chunk 依次生成上表里的字段：

```go
tok := tokenizer.New(language)
for i := range chunks {
	ck := &chunks[i]
	titleTk, err := tok.Tokenize(titleStem)
	...
	titleSmTk, err := tok.FineGrainedTokenize(titleTk)
	...
	ck.TitleTks = titleTk
	ck.TitleSmTks = titleSmTk

	if q := ck.Questions; q != "" {
		if err = ck.SetExtraValue("question_kwd", strings.Split(q, "\n")); err != nil {
		...
		qt, err := tok.Tokenize(q)
		...
		if err = ck.SetExtraValue("question_tks", qt); err != nil {
	...
	if kw := ck.Keywords; kw != "" {
		rawParts := strings.Split(kw, ",")
		...
		if err = ck.SetExtraValue("important_kwd", kwdParts); err != nil {
		...
		it, err := tok.Tokenize(kw)
		...
		if err = ck.SetExtraValue("important_tks", it); err != nil {
```

注意关键词是按**英文逗号**拆分的。第 18 章的抽取提示词要求模型"用英文逗号分隔"，两边是配套的。

`Tokenize` 和 `FineGrainedTokenize`（`internal/tokenizer/tokenizer.go:453`、`:506`）背后是一个 C++ 分词器（`internal/binding/cpp/`），构建时由 `build.sh` 编译并静态链接。目录里有 `darts_trie.cpp`（双数组 trie，存放词典）、`opencc/`（繁简转换）、`stemmer/` 和 `wordnet_lemmatizer.cpp`（英文词干与词形还原），以及 `thinc_ner.cpp`（命名实体识别）。中文分词基于词典和词频（`tokenizer.GetTermFreq` 能查到词频），先切出"知识库"这样的粗粒度词（`ltks`），再把它细分成"知识 / 库"这样的细粒度词（`sm_ltks`）。

为什么要两种粒度？用户搜"知识"时，只有粗粒度"知识库"的 chunk 搜不到，有了细粒度"知识"就能搜到。用户搜"知识库"时，粗粒度命中的分数更高，更精确。

## 10.3 向量化：RAGFlow 的做法

### 标题向量加权

`embedChunks`（`tokenizer.go:430`）为每个 chunk 计算向量，并且**混进一点文件名的向量**。`mergeEmbeddingVectors`（`:640`）：

```go
func mergeEmbeddingVectors(titleVec, contentVec []float64, titleWeight float64) ([]float64, error) {
	if len(titleVec) == 0 || len(contentVec) == 0 {
		return nil, fmt.Errorf("empty embedding vector")
	}
	if len(titleVec) != len(contentVec) {
		return nil, fmt.Errorf("unexpected embedding dimensions")
	}
	merged := make([]float64, len(titleVec))
	for i := range titleVec {
		merged[i] = titleWeight*titleVec[i] + (1-titleWeight)*contentVec[i]
	}
	return merged, nil
}
```

`titleWeight` 就是模板里的 `filename_embd_weight`，默认 0.1。最终向量 = 0.1 × 文件名向量 + 0.9 × 正文向量。这样，《星河咖啡机用户手册》里讲"除垢"的那一块，即使正文里没出现"咖啡机"，向量上也带了一点"咖啡机手册"的方向。合并后的向量写进 `q_<维度>_vec`：

```go
if err := chunks[i].SetExtraValue(fmt.Sprintf("q_%d_vec", len(merged)), merged); err != nil {
```

### 截断

embedding 模型都有输入长度上限，超了会直接报错。`truncateForEmbedding`（`:634`）的注释记录了一次事故：最初用"固定留 10 个 token 余量"的办法，对 8192 窗口来说只有 0.12%，比两种分词器之间的差异还小一个数量级，结果还是超限；修复后改成按比例留余量，并优先使用模型自己的分词器来计数。

### 缓存

重新解析一份文档时，大部分 chunk 的内容其实没变。`embedChunks` 先查 Kvrocks 里的 chunk 缓存（`internal/ingestion/chunkcache`），只对没命中的 chunk 调模型。缓存键（`chunkcache.go:98` 的 `Key`）由 `类型 + 模型id + chunk id + 实际送去 embedding 的文本` 组成，每个字段都加了长度前缀，防止 `"ab","c"` 和 `"a","bc"` 撞出同一个键。注释里还说明了一个细节："the content vector is title-weight independent"：缓存里存的是**合并标题之前**的正文向量，所以只改 `filename_embd_weight` 时，缓存依然有效。

## 10.4 从零实现

### 分词：没有词典怎么办

mini-ragflow 不带中文词典。它用 n-gram 来近似两种粒度（`backend/internal/tokenize/terms.go`）：

```text
粗粒度 Terms:     英文单词 + 中文二元组   "知识库问答" → 知识 识库 库问 问答
细粒度 FineTerms: 英文单词 + 中文单字     "知识库问答" → 知 识 库 问 答
```

另外做了三件小事：全角转半角（`ＲＡＧ１２３` → `RAG123`，和 RAGFlow 查询端的 `StrFullWidth2HalfWidth` 一样）、英文小写和复数还原（`systems` → `system`）、去停用词（"的、了、是"，"the、of、is"）。

二元组的问题是会切出"跨词的伪词"："年假有几天"会切出"假有"、"有几"。这个问题会在第 12 章造成真实的检索失败。这就是不用词典的代价。

### Embedding 模型

`backend/internal/llm/` 用 RAGFlow 的 `模型名@厂商` 命名约定（`BAAI/bge-large-zh-v1.5@BAAI`），支持两个"厂商"：

- `mock-embedding@mock`：**特征哈希**。把每个词哈希到 256 维中的某一维，加 ±1，最后做 L2 归一化。共享词越多，余弦越高；完全没有共享词，余弦接近 0。它不懂同义词，但结果确定、不需要网络，所有测试都能离线运行；
- `<model>@openai`：任意 OpenAI 兼容的 `/embeddings` 接口（OpenAI、Ollama、vLLM……）。

```go
func (m *MockEmbedder) vector(text string) []float64 {
	v := make([]float64, m.Dim)
	add := func(term string, w float64) {
		h := fnv.New64a()
		h.Write([]byte(term))
		x := h.Sum64()
		sign := 1.0
		if x>>63 == 1 {
			sign = -1
		}
		v[x%uint64(m.Dim)] += sign * w
	}
	for _, t := range tokenize.Terms(text) {
		add(t, 1)
	}
	for _, t := range tokenize.FineTerms(text) {
		add("#"+t, 0.5)
	}
	... // L2 normalize
}
```

### Tokenizer 组件

`backend/internal/pipeline/tokenizer.go` 做了和 RAGFlow 一样的四件事：

1. 填 `title_tks / content_ltks / content_sm_ltks`。问答块的 `content_ltks` 由切分器预先填好，只包含问题，这里不再覆盖；
2. 截断到模型的 `MaxTokens()`；
3. 先查缓存，没命中的按 16 个一批调模型；
4. 合并标题向量：`v = 0.1·embed(文件名) + 0.9·embed(正文)`，写进 `Chunk.Vector`。

缓存（`backend/internal/pipeline/cache.go`）是 SQLite 里的一张 `embedding_cache` 表，键是 `sha1(模型id + "\x00" + 文本)`。和 RAGFlow 一样，缓存的是合并标题之前的正文向量。

向量在 chunk 表里存成 little-endian 的 float32 二进制（`docengine.EncodeVector`），比 float64 省一半空间。余弦排序对这点精度损失并不敏感。

## 10.5 运行与验证

```bash
git checkout v0.2 && cd backend
go test ./internal/tokenize/ ./internal/llm/ -v | grep -E '^(---|ok)'
```

```text
--- PASS: TestCount (0.00s)
--- PASS: TestTruncate (0.00s)
--- PASS: TestTerms (0.00s)
ok  	mini-ragflow/internal/tokenize	0.001s
--- PASS: TestMockEmbedderSimilarity (0.00s)
--- PASS: TestSplitID (0.00s)
--- SKIP: TestOllamaEmbedding (0.00s)
ok  	mini-ragflow/internal/llm	0.004s
```

接上本机 Ollama 跑真实的 embedding：

```bash
MR_OLLAMA=1 go test -run Ollama -v ./internal/llm/ | grep -E 'dim=|ok'
```

```text
    llm_test.go:49: dim=768
ok  	mini-ragflow/internal/llm	1.793s
```

用真实模型解析同一份手册两次，第二次的 7 个 chunk 全部命中缓存：

```text
Embedded 7 chunks with nomic-embed-text@openai (dim 768, 0 cache hits).
Indexed 7 chunks, 837 tokens in 1.83s.
...
Embedded 7 chunks with nomic-embed-text@openai (dim 768, 7 cache hits).
Indexed 7 chunks, 837 tokens in 0.00s.
```

在 chunk 页面勾选"显示索引字段"，可以看到每块的 `title_tks` 和 `content_ltks`（见第 9 章图 9-1）。

### 选哪个 embedding 模型

这一章只用 `nomic-embed-text` 跑通了流程，但它**不适合中文**。第 12 章会给出量化对比：同一个问题和三段中文文本的余弦分别是 0.726、0.711、0.719，几乎分不出相关和不相关。中文场景下请用 `bge-m3` 这类多语言模型。

## 10.6 练习

1. **巩固**：把 `filename_embd_weight` 改成 0 和 0.5，分别对"咖啡机怎么除垢"做检索（v0.3 的检索测试页）。向量相似度怎么变？为什么不能设得太大？
2. **扩展**：给 mini-ragflow 接一个中文分词库（比如 gse），替换 `Terms` 的二元组实现，然后重跑第 13 章的检索测试，看看"工作满10年年假有几天"的结果有没有变化。
3. **读源码**：`truncateForEmbedding` 的注释为什么说 cl100k 分词器"is not the model's tokenizer"？如果按 cl100k 计数截到 8000，真实模型的分词结果可能是多少？

## 10.7 延伸阅读

- `internal/ingestion/component/tokenizer.go`：整个 Tokenizer 组件
- `internal/ingestion/task/embedder.go`：批量、限流与重试
- Weinberger et al., *Feature Hashing for Large Scale Multitask Learning*：mock embedding 用的哈希技巧
