# 第 12 章 查询理解：从问题到检索表达式

> 本章目标：把用户的一句自然语言变成一个"带权重的全文查询"：清洗、去疑问词、分词、给每个词算权重、扩展同义词、决定哪些词必须命中。
> 前置：第 10、11 章。对应代码：`v0.3` 的 `backend/internal/retrieval/query.go`。

## 12.1 问题

用户问"咖啡机**怎么**除垢**？**"。如果原样拿去做全文检索：

- "怎么"出现在很多 chunk 里，却和答案毫无关系；
- "咖啡机"和"除垢"同等重要吗？如果数据集里每个 chunk 都带着"咖啡机"（因为文件名就叫《咖啡机手册》），那"除垢"才是区分度最高的词；
- 文档里写的是"去水垢"，用户说的是"除垢"，字面上对不上。

查询构造要解决的就是这三件事：**去噪、加权、扩展**。

## 12.2 RAGFlow 的 Question()

`QueryBuilder.Question`（`internal/service/nlp/query_builder.go:232`）的开头是一串清洗步骤：

```go
txtWithSpaces := qb.AddSpaceBetweenEngZh(txt)        // 中英文之间补空格："RAGFlow怎么用" → "RAGFlow 怎么用"
txtLower := strings.ToLower(txtWithSpaces)
txtHalfWidth := qb.StrFullWidth2HalfWidth(txtLower)  // 全角转半角
txtSimplified := qb.Traditional2Simplified(txtHalfWidth) // 繁转简
txtCleaned := rePunct.ReplaceAllString(txtSimplified, " ")
txtNoStopWords := qb.RmWWW(txtCleaned)               // 去疑问词
```

`RmWWW` 用的正则在 `:55`，中文部分是一长串疑问词：

```go
reStopWordsZH = regexp.MustCompile(`(?i)是*(怎么办|什么样的|哪家|一下|那家|请问|啥样|咋样了|什么时候|何时|何地|何人|是否|是不是|多少|哪里|怎么|哪儿|怎么样|如何|哪些|是啥|啥是|啊|吗|呢|吧|咋|什么|有没有|呀|谁|哪位|哪个)是*`)
```

接下来分词，给每个词算权重，再把它们拼成 ES 的 `query_string`。中文分支（`:430` 起）给每个词构造了这样一个子句：

```go
if len(fineGrainedSyns) > 0 {
	term = fmt.Sprintf("(%s OR (%s)^0.2)", term, strings.Join(fineGrainedSyns, " "))
}
if len(sm) > 0 {
	smStr := strings.Join(sm, " ")
	term = fmt.Sprintf(`%s OR "%s" OR ("%s"~2)^0.5`, term, smStr, smStr)
}
...
termParts = append(termParts, fmt.Sprintf("(%s)^%v", termWeight.term, termWeight.weight))
```

翻译成白话：一个词 `w` 可以通过下面任意一种方式命中，命中后的贡献乘以它的权重 `^weight`：

- 原词；
- 同义词（打 0.2 折）；
- 它的细粒度拆分作为一个短语（`"知识 库"`）；
- 细粒度拆分在距离 2 以内邻近出现（`~2`，打 0.5 折）。

整句还要再加一个短语查询：

```go
tmsStr += fmt.Sprintf(` ("%s"~2)^1.5`, tokenized)
```

整句的同义词改写则以 `(原查询)^5 OR (同义改写)^0.7` 的方式并入。字段权重在 `:79`：

```go
"title_tks^10",
"title_sm_tks^5",
"important_kwd^30",
"important_tks^20",
"question_tks^20",
"content_ltks^2",
```

人工标注的关键词（`important_kwd`）权重最高，是正文的 15 倍；问题和标题其次。这组数字说明，RAGFlow 认为"chunk 讲什么"比"chunk 里写了什么"更能代表相关性。

### 词的权重

`TermWeightDealer.Weights`（`internal/service/nlp/term_weight.go:311`）的公式是：

```text
w = (0.3·idf(词频) + 0.7·idf(文档频率)) · ner(t) · pos(t)
idf(s, N) = log10(10 + (N − s + 0.5) / (s + 0.5))
```

- 两个 idf 的统计量都来自分词器自带的**语料词典**（`GetTermFreq` 和 `df` 表），不是当前数据集的统计；
- `ner(t)`：数字 ×2，一两个字母的英文 ×0.01（几乎忽略），公司名、地名、学校 ×3；
- `pos(t)`：代词、连词、副词 ×0.3，名词 ×2，地名机构名 ×3；
- 最后归一化，所有词的权重加起来等于 1。

这就是"咖啡机"和"除垢"谁更重要的答案：越少见的名词越重要。

### minimum_should_match

`Question` 的第三个参数是 `minMatch`，对应 `query_string` 的 `minimum_should_match`：一个 chunk 至少要命中多少比例的查询词，才算"全文命中"。检索时先用 0.3，搜不到再放宽到 0.1（第 13 章）。`minMatch` 函数（`retrieval.go:645`）还有一个规则：向量权重 ≥ 0.8 时，全文这一路"随便命中一个词就行"（`min_match = 0`）。

## 12.3 从零实现

`backend/internal/retrieval/query.go` 的 `Build`：

```go
func Build(ctx context.Context, eng docengine.Searcher, kbIDs []string, question string, minMatch float64) (*Query, error) {
	cleaned := RemoveQuestionWords(tokenize.Normalize(strings.ToLower(question)))
	coarse := dedupe(tokenize.Terms(cleaned))
	weights, unseen, err := TermWeights(ctx, eng, kbIDs, coarse)
	...
	mt := &docengine.MatchText{Fields: DefaultFields(), MinShouldMatch: minMatch}
	for _, t := range coarse {
		_, junk := unseen[t]
		mt.Terms = append(mt.Terms, docengine.QueryTerm{Term: t, Weight: weights[t], Optional: junk})
	}
	for _, t := range tokenize.FineTerms(cleaned) {        // 细粒度：可选、低权重
		...Optional: true
	}
	for _, t := range coarse {                              // 同义词：可选、×0.25
		for _, syn := range Synonyms[t] {
			...Optional: true
		}
	}
```

和 RAGFlow 对应的地方：

| RAGFlow | mini-ragflow |
|---|---|
| 清洗：全角半角、繁简、`RmWWW` | `Normalize` + `RemoveQuestionWords`（同一组疑问词正则），没有繁简转换 |
| 词典分词 | 二元组（第 10 章） |
| 细粒度、同义词作为 `OR` 分支 | 细粒度、同义词作为**可选词**（`Optional: true`），不计入 `minimum_should_match` |
| idf 来自语料词典 | idf 公式相同，统计量来自**当前数据集**（`Searcher.DocFreq`） |
| 字段 boost | 相同的五个字段和权重 |

同义词表是一个很小的内置 map，只用来演示机制：

```go
var Synonyms = map[string][]string{
	"除垢": {"去水垢", "清洗"},
	"保修": {"质保", "维修"},
	"休假": {"请假", "假期"},
	"价格": {"售价", "多少钱"},
}
```

### 踩坑记录：二元组切出的伪词毁掉了查询

v0.3 第一次跑检索测试时，有一个问题怎么也搜不出来：

```text
retrieval_test.go:84: 工作满10年年假有几天: top = "<none>", want it to contain "第三条" (trace [pass 1: kNN filtered by full-text, min_match=0.3, 1 hits])
```

拆开来看：二元组把"工作满10年年假有几天"切成了"工作 / 作满 / 10 / 年年 / 年假 / 假有 / 有几 / 几天"。其中"作满""年年""假有""有几""几天"是**跨词的伪词**，语料里一次都没出现过。按 idf 公式，文档频率越低权重越高，于是这些伪词拿到了**最高**的权重。后果有两个：

1. 词相似度（第 13 章）的分母被伪词撑大，真正的关键词"年假"占比很小，相似度过不了阈值；
2. `minimum_should_match = 30%` 是按 8 个词算的，至少要命中 3 个，可是"真词"只有 3 个（工作、10、年假），稍有偏差就整体落空。

RAGFlow 不会遇到这个问题，因为它的词典分词根本不会切出"假有"。mini-ragflow 的修法是：**数据集里从没出现过的中文二元组，权重打一折，并且不计入 `minimum_should_match`**。思路是把数据集自己的词表当成词典用：

```go
case df[t] == 0 && isCJKBigram(t):
	v *= 0.1
	unseen[t] = struct{}{}
```

修复后，同一个问题构造出的查询（v0.3，真实输出）：

```text
Q: 工作满10年年假有几天
  cleaned: "工作满10年年假有几天"
  工作     w=0.201 optional=false
  作满     w=0.031 optional=true
  10     w=0.444 optional=false
  年年     w=0.031 optional=true
  年假     w=0.201 optional=false
  假有     w=0.031 optional=true
  有几     w=0.031 optional=true
  几天     w=0.031 optional=true
  工      w=0.022 optional=true
  ...
```

"10" 的权重最高：它是数字（×2），而且在语料里很少见。这正是 RAGFlow `ner` 规则想要的效果，在"年假多少天"这类问题里，数字往往是关键。

另一个问题"咖啡机怎么除垢？"：

```text
Q: 咖啡机怎么除垢？
  cleaned: "咖啡机除垢?"
  咖啡     w=0.298 optional=false
  啡机     w=0.298 optional=false
  机除     w=0.049 optional=true
  除垢     w=0.355 optional=false
  咖      w=0.040 optional=true
  ...
  去水     w=0.089 optional=true
  水垢     w=0.089 optional=true
  清洗     w=0.089 optional=true
```

"怎么"被去掉了；"机除"是伪词，被降权；"除垢"的权重最高；同义词"去水垢""清洗"以可选词的身份加入了查询。

## 12.4 运行与验证

```bash
git checkout v0.3 && cd backend && go test ./internal/retrieval/ -run TopHits -v | grep -E 'retrieval_test|ok'
```

```text
    retrieval_test.go:88: 咖啡机怎么除垢？       sim=0.484 tsim=0.576 vsim=0.269  [pass 1: kNN filtered by full-text, min_match=0.3, 6 hits]
    retrieval_test.go:88: 工作满10年年假有几天    sim=0.274 tsim=0.254 vsim=0.321  [pass 1: kNN filtered by full-text, min_match=0.3, 2 hits]
    retrieval_test.go:88: 能用胶囊吗          sim=0.564 tsim=0.683 vsim=0.285  [pass 1: kNN filtered by full-text, min_match=0.3, 1 hits]
    retrieval_test.go:88: 蒸汽棒不出汽怎么办      sim=0.774 tsim=1.000 vsim=0.246  [pass 1: kNN filtered by full-text, min_match=0.3, 2 hits]
    retrieval_test.go:88: 陪产假            sim=0.867 tsim=1.000 vsim=0.556  [pass 1: kNN filtered by full-text, min_match=0.3, 3 hits]
ok  	mini-ragflow/internal/retrieval	0.013s
```

五个问题的第一名都是正确的 chunk（`TestTopHits` 断言了这一点）。`sim / tsim / vsim` 是什么，下一章讲。

## 12.5 练习

1. **巩固**：在 `RemoveQuestionWords` 里去掉"多少"，然后问"保修多少年"，对比去掉前后的查询词。
2. **扩展**：mini-ragflow 的 idf 用的是当前数据集的统计。一个只有 3 份文档的小数据集，统计会有什么偏差？如果混合"语料词典 idf"和"数据集 idf"，你会怎么设计？
3. **读源码**：RAGFlow `Question()` 的英文分支（`!qb.IsChinese(...)`）和中文分支有什么不同？为什么英文不需要细粒度拆分？

## 12.6 延伸阅读

- `internal/service/nlp/query_builder.go`、`term_weight.go`、`synonym.go`、`wordnet.go`
- `internal/rag/res/synonym.json`：RAGFlow 的同义词表
- Elasticsearch 文档：`query_string` 的 boost（`^`）、邻近（`~`）和 `minimum_should_match`
