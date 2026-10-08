# Lab 3：mini-ragflow v0.3——自研混合检索引擎 + 检索测试页

> 目标：让 chunk 真正能被检索。写一个"倒排 + 向量"的引擎，复刻 RAGFlow 的查询构造与两阶段打分，再做一个能看清楚每一步的检索测试页。
> 前置：第 11~14 章。

## 1. 这一版改了什么

```bash
git diff v0.2 v0.3 --stat | tail -1
```

```text
 24 files changed, 1786 insertions(+), 23 deletions(-)
```

| 新增/改动 | 文件 | 章节 |
|---|---|---|
| 倒排索引、BM25、向量扫描、两种融合、版本号失效 | `internal/docengine/search.go` | 11 |
| chunk 新增 `important_tks / question_tks` 列 | `internal/docengine/engine.go` | 11 |
| 查询构造、term 权重、同义词、伪词降权 | `internal/retrieval/query.go` | 12 |
| 词相似度（单词 0.4 + 词对 0.6） | `internal/retrieval/similarity.go` | 13 |
| 三次检索、两阶段打分、阈值、分页、聚合、高亮 | `internal/retrieval/retrieval.go` | 13 |
| rerank 模型：mock / `POST /rerank` | `internal/llm/rerank.go` | 13 |
| `POST /api/v1/retrieval`；数据集的 `pagerank`；store 自动加列 | `internal/api/retrieval.go`、`internal/store/` | 13 |
| 检索测试页 | `web/src/pages/Testing.tsx` | 14 |

## 2. 实现顺序

1. `docengine.SearchRequest / Hit / Searcher` 接口，先写 `Search` 的纯全文路径，再加向量和两种融合；
2. `retrieval.TermWeights` 和 `Build`；
3. `TokenSimilarity`，用 `TestTokenSimilarity` 里手算的 4/7 来校验；
4. `Retrieve`，测试依次覆盖：五个问题的第一名、`sim` 公式、阈值、三次检索、融合方式、文档过滤、分页约束、rerank；
5. API 和前端。

## 3. 接口

```bash
curl -s -XPOST localhost:9380/api/v1/retrieval -H 'Content-Type: application/json' -d '{
  "question": "咖啡机怎么除垢？",
  "dataset_ids": ["<kb_id>"],
  "similarity_threshold": 0.2,
  "vector_similarity_weight": 0.3,
  "highlight": true,
  "page_size": 3
}'
```

响应字段沿用 RAGFlow 改名之后的那一套（`internal/service/dataset/search.go:332` 的 `keyMapping`）：`content`、`document_id`、`document_keyword`、`dataset_id`、`similarity`、`term_similarity`、`vector_similarity`、`highlight`、`important_keywords`、`questions`，再加上 mini-ragflow 自己的 `trace`。

多个数据集一起检索时，要求它们的 embedding 模型相同：一个查询向量只能和同一个向量空间里的向量比较。

## 4. 跑起来

```bash
cd backend && go test ./... && cd ../web && npm run build
cd ../backend && go run ./cmd/mini-ragflow --data /tmp/v03 --static ../web/dist &
cd ../web && node e2e/ui-v03.mjs http://localhost:9380 /tmp/shots ../samples              # mock
node e2e/ui-v03.mjs http://localhost:9380 /tmp/shots ../samples bge-m3@openai             # 真实模型
```

mock 模型下的真实输出（数值未改，排版做了整理）：

```text
Q: 咖啡机怎么除垢？  共 6 个结果 · 关键词：咖啡 啡机 机除 除垢  pass 1: kNN filtered by full-text, min_match=0.3, 6 hits
  top: 星河咖啡机用户手册.md · 综合0.484 关键词0.576 向量0.269  ## 5. 清洁与除垢 ……
Q: 工作满10年年假有几天  共 1 个结果
  top: 员工休假制度.txt · 综合0.274 关键词0.254 向量0.321  第二章 年假 第三条 ……
Q: 牛奶要加热到多少度  共 0 个结果 · pass 1 … 0 hits | pass 2 (relaxed) … 0 hits | pass 3 (dense only): 0 hits
Q: descaling steps  共 0 个结果 · …
```

mock 的哈希向量对长句子几乎没有区分度，后两个问题在 mock 下搜不到是预期之中的；换成 `bge-m3`，"牛奶要加热到多少度"的第一名就是"打奶泡"（第 13 章）。

## 5. 检查点

- [ ] `TestTopHits` 的五个问题第一名都正确；
- [ ] 检索测试页上，第一条结果的综合分 = `0.7 × 关键词 + 0.3 × 向量`；
- [ ] 把向量权重拖到 0，阈值不再起作用；
- [ ] 给一个数据集设置 `pagerank = 50`，两个数据集一起检索时它的结果全部排在前面（`TestRetrievalAPIAndPagerank`）；
- [ ] 用 v0.1 的数据目录启动，旧 chunk 在重新解析前搜不到，重新解析后能搜到（第 11 章）。

## 6. 本版的踩坑汇总

| 现象 | 原因 | 处理 | 章节 |
|---|---|---|---|
| "工作满10年年假有几天"搜不到 | 二元组切出的伪词拿到最高 idf | 未见过的二元组降权并设为可选 | 12 |
| 纯向量兜底召回 26 条，最终 0 条 | 兜底结果仍按 `(1-w)·tsim + w·vsim` 打分 | 兜底结果按向量分本身打分，并加回归测试 | 13 |
| nomic 把"保修"排在"清洁与除垢"前面 | 模型对中文没有区分度 | 换 bge-m3 | 13 |
| bge-m3 返回 NaN | GPU 上 fp16 溢出 | 换纯 CPU 的 Ollama 实例 | 13 |
| UI 脚本读到上一个问题的结果 | 等待了旧的元素 | 改为等待接口响应 | 14 |
| `*_tks` 的评分和 RAGFlow 不一致（写书时才发现） | ES 里用的是 `scripted_sim` | v1.0 修正 | 11 |

## 7. 下一版

能检索了，下一步就是问答：把检索到的 chunk 拼进提示词，流式生成，并给回答标上引用。

```bash
git diff v0.3 v0.4 --stat | tail -1
```

```text
 25 files changed, 2244 insertions(+), 38 deletions(-)
```
