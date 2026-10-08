# 第 18 章 Extractor：用 LLM 给 chunk 补关键词、问题、元数据

> 本章目标：理解"入库时多花一点 LLM 成本，换检索时更准"这类增强手段：自动关键词、自动问题、元数据抽取，以及它们分别影响检索的哪一环。
> 前置：第 12、13 章。对应代码：`v0.5` 的 `backend/internal/pipeline/extractor.go`。

## 18.1 为什么要在入库时调 LLM

第 12 章的字段权重告诉我们：`important_kwd` 的权重是正文的 15 倍，`question_tks` 是 10 倍。可这两个字段从哪里来？人工标注太贵。Extractor 组件让 LLM 替我们写：

| 增强 | 写进哪个字段 | 帮了谁 |
|---|---|---|
| 自动关键词（每块 N 个） | `important_kwd` / `important_tks` | 全文检索的高权重字段；`tsim` |
| 自动问题（每块 N 个） | `question_kwd` / `question_tks` | 用户提问和"假想问题"措辞接近，匹配更容易 |
| 元数据（按 schema） | 文档的 `meta_fields` | 按元数据过滤（第 19 章） |
| 摘要 | chunk 的摘要字段 | 拼提示词时更省 token |

"自动问题"背后的思路常被称为 *HyDE 的反向版本*（HyDE 是"为问题生成假想答案"，这里是"为答案生成假想问题"）：用户问的是问题，chunk 是答案，两者的措辞天然不同；预先给每个答案配几个问题，相当于在两种措辞之间搭了一座桥。

## 18.2 RAGFlow 的 Extractor

`internal/ingestion/component/extractor.go`。默认模板里的配置（第 7 章）：

```json
"Extractor:AutoExtractDefault": {"obj": {"component_name": "Extractor", "params": {
  "llm_id": "", "keywords": {"top_n": 0}, "questions": {"top_n": 0},
  "tags": {"top_n": 0, "tag_file_id": ""}, "summary": {"enabled": false},
  "metadata": {"enabled": false, "metadata": [], "built_in_metadata": []}}}}
```

默认全部关闭（`top_n: 0`），用户按需打开。

### 提示词

提示词作为常量写在同一个文件里（`:141` 起），关键词的提示词是：

```go
autoKeywordPrompt = `## Role
You are a text analyzer.

## Task
Extract the most important keywords/phrases of a given piece of text content.

## Requirements
- Summarize the text content, and give the top {{ topn }} important keywords/phrases.
- The keywords MUST be in the same language as the given piece of text content.
- The keywords are delimited by ENGLISH COMMA.
- Output keywords ONLY.`
```

"用英文逗号分隔"和第 10 章 Tokenizer 里"按英文逗号拆分关键词"是配套的。（`rag/prompts/keyword_prompt.md` 里有同样内容的一份，供 `generator.go` 里聊天时的关键词扩展通过 `LoadPrompt` 读取。）

元数据的提示词（`autoMetadataPrompt`，`:167`）规则非常严格：

```text
 - Fixed Key Set: Output ONLY the keys that appear in the Schema. Never add, rename, translate, pluralize or re-case a key, ...
 - Strict Evidence Only: Extract a value ONLY if it is explicitly mentioned in the Content.
 - Enum Filter: For any field with an 'enum' list, the list acts as a strict filter. ...
 - No Meta-Inference: Do not infer values based on the document's nature, format, or category. ...
 - Zero-Hallucination: Never invent information or pick a "likely" value from the enum to fill a field.
 - Empty Result: If no matches are found for any field, or if the content is irrelevant, output ONLY {}.
```

这么严格，是因为元数据会被用来**过滤**：一个错误的值会让文档在过滤时被错误地排除或纳入，后果比一个错误的关键词严重得多。

### 元数据是逐块抽、再合并的

`runEnableMetadata`（`extractor.go:1012`）按 chunk 逐个抽取元数据：

```go
schemaMap := common.Turn2JSONSchema(c.Param.Metadata.Metadata)
...
if cached, hit := getMetadataLLMCache(ctx, in.cache, modelID, schemaStr, chunkID); hit {
	parsed = cached
} else {
	...
	systemPrompt := fmt.Sprintf(autoMetadataPrompt, schemaStr)
	parsed, err = c.callStructured(ctx, db, metaIn, systemPrompt, chunkText)
	...
	parsed = filterMetadataToDeclaredKeys(parsed, c.Param.Metadata.Metadata)
```

三个要点：

1. **schema 先转成 JSON Schema**（`Turn2JSONSchema`，`internal/common/metadata_utils.go:391`），渲染成 `{"type":"object","properties":{...},"additionalProperties":false}`。这个细节在 mini-ragflow 里直接决定了成败，见 18.3 节；
2. **只保留声明过的键**（`filterMetadataToDeclaredKeys`）：就算模型自作主张多吐了一个键，也会被丢掉；
3. **结果按 (模型, schema, chunk) 缓存**。

每个 chunk 抽出来的元数据，最后由 `mergeChunkMetadata`（`internal/ingestion/task/indexdoc/process.go:173`）合并到文档级别。

### 并发

LLM 调用是入库流程里最慢的一步。Extractor 用一个进程级的工作池并发处理多个 chunk，默认 10 个并发（`defaultExtractorConcurrency`，`:88`），注释里说这和 Python 版的 `chat_limiter` 一致。温度固定为 0.2，单次调用超时 600 秒。

## 18.3 从零实现

`backend/internal/pipeline/extractor.go`，参数用扁平键：`auto_keywords`、`auto_questions`、`metadata`（字段列表）、`llm_id`。处理流程：

```go
for i := range st.Chunks {
	...
	sem <- struct{}{}                          // 并发上限 4
	go func(i int) {
		...
		if c.keywords > 0 {
			if out, err = ask(fmt.Sprintf(keywordPrompt, c.keywords), ch.Content); err == nil {
				kws = splitKeywords(out, c.keywords)
			}
		}
		if err == nil && c.questions > 0 {
			if out, err = ask(fmt.Sprintf(questionPrompt, c.questions, c.questions), ch.Content); err == nil {
				qs = splitLines(out, c.questions)
			}
		}
		...
```

`ask` 先查 LLM 缓存（`llm_cache` 表，键是 `sha1(模型 + 提示词 + 正文)`），没命中才调模型。`splitKeywords` 除了英文逗号，也接受中文逗号和顿号，并去掉 `1.`、`-` 这类列表前缀，因为小模型经常不听"用英文逗号"的指令。

元数据和 RAGFlow 不同，**按文档抽取一次**：取前 1500 个 token 左右的 chunk 拼起来，一次调用抽出整个文档的元数据。这省掉了逐块调用的成本；代价是，只出现在文档后半部分的信息抽不到。

### 踩坑记录 1：JSON Schema 不是可有可无

v0.5 第一次用真实模型（qwen2.5:3b）跑元数据抽取，两份手册的结果都是空的：

```text
Extracted metadata map[].
```

翻 LLM 缓存，模型的原始输出就是 `{}`。最初的提示词里，schema 是原样塞进去的字段列表：

```json
[{"key": "产品型号", "description": "咖啡机型号，如 S1、S2", "enum": ["S1", "S2"]}]
```

把提示词单独拿出来，在同一个模型上做了对照实验（真实输出）：

```text
A (system 放 schema 列表，user 放正文):           '{}'
B (schema 和正文都放 system，user 写 "Output: "):  '{}'
C (schema 换成 JSON Schema 对象):                 '{\n  "产品型号": "S2"\n}'
D (JSON Schema + 一个输出示例):                    '{"产品型号": "S2"}'
```

只改了 schema 的写法，结果就从"什么都没抽到"变成了正确答案。小模型显然更熟悉 JSON Schema 这种形式。回头看 RAGFlow，它正是先用 `Turn2JSONSchema` 转换过的。mini-ragflow 改用同样的格式：

```go
// jsonSchema renders the field list as a JSON Schema object, like RAGFlow's
// common.Turn2JSONSchema. The form matters: given the raw list,
// qwen2.5:3b answered {} for a manual that names its model on line one;
// given this schema it answered {"产品型号": "S2"}.
```

改完之后再跑，S2 的手册抽出了 `{"产品型号": "S2"}`，但 S1 的手册**仍然是 `{}`**。两份手册的开头分别是"# 星河 S1 咖啡机用户手册"和"# 星河 S2 全自动咖啡机用户手册"，结构几乎一样。这是小模型输出的不稳定性，书里如实记录，不粉饰：生产环境里，元数据抽取的结果需要能被人工查看和修改（mini-ragflow 的文档表上有"元数据"按钮）。

### 踩坑记录 2：一次输出里有两个 JSON

缓存里还有这样一条输出：

```text
"{}\n```json\n{}\n```"
```

模型先输出了一个 `{}`，又在代码块里输出了一遍。最初的解析用的是贪婪正则 `\{.*\}`，它会匹配从第一个 `{` 到最后一个 `}` 的全部内容，也就是 `{}\n```json\n{}`，这不是合法的 JSON。改成"从每个 `{` 开始尝试用 JSON 解码器解码第一个完整值"：

```go
func FirstJSONObject(s string, v any) error {
	for i := strings.IndexByte(s, '{'); i >= 0; {
		if err := json.NewDecoder(strings.NewReader(s[i:])).Decode(v); err == nil {
			return nil
		}
		...
```

### 观察：关键词的语言

真实模型给 S2 手册"制作咖啡"那一块抽出的关键词是 `coffee | espresso | drip`。提示词明确要求"MUST be in the same language as the given piece of text content"，原文是中文，模型还是给了英文。英文关键词写进 `important_tks` 以后，中文提问基本不会命中它们。这种情况在 mock 测试里永远不会出现，只有接真实模型才能看到。

![图 18-1：真实模型（qwen2.5:3b）给 S2 手册补的关键词和问题。](./images/v05-chunks-enriched-real.png)

*图 18-1：真实模型给 S2 手册补的关键词和问题。第 2 块的关键词变成了英文。*

## 18.4 运行与验证

```bash
git checkout v0.5 && cd backend && go test ./internal/pipeline/ -run 'Extractor|FirstJSON' -v | grep -E '^(---|ok)|pipeline_test'
```

```text
    pipeline_test.go:152: book/星河S2咖啡机用户手册.md: Parsed as markdown: 10 sections map[list:1 text:4 title:5]. | Chunked by title into 4 chunks. | Extractor: 3 keywords / 1 questions per chunk with mock-chat@mock. | Extracted metadata map[产品型号:S2]. | Embedded 4 chunks with mock-embedding@mock (dim 256, 0 cache hits). | RAPTOR level 1: 4 nodes → 2 clusters, 2 summaries. | RAPTOR level 2: 2 nodes → 1 clusters, 1 summaries.
--- PASS: TestExtractorAndRaptor (0.01s)
--- PASS: TestFirstJSONObject (0.00s)
ok  	mini-ragflow/internal/pipeline	0.010s
```

在数据集配置面板里打开增强（下图），再上传两份手册：

![图 18-2：v0.5 的数据集配置：自动关键词、自动问题、抽取用的 LLM、RAPTOR 开关和元数据 schema。](./images/v05-settings.png)

*图 18-2：v0.5 的数据集配置。*

成本是实打实的（真实输出）。同样两份手册（11 个 chunk），每块 3 个关键词 + 2 个问题，加上元数据和 RAPTOR：

| 模型 | 解析耗时 |
|---|---|
| mock | 1.7 秒 |
| qwen2.5:3b + bge-m3（CPU 上的 Ollama） | 78.6 秒 |

## 18.5 练习

1. **巩固**：在 mini-ragflow 的关键词提示词里加一个中文示例，观察 qwen2.5:3b 输出英文关键词的情况是否减少。
2. **扩展**：把元数据抽取改成 RAGFlow 那样"逐块抽取、再合并"，并想清楚合并冲突怎么处理（同一个键，不同的块抽出了不同的值）。
3. **读源码**：RAGFlow Extractor 的 `tags` 功能（`extractor_tag.go`）怎么用一个"标签文件"给 chunk 打标签？这些标签后来在检索里起什么作用？（第 19 章会用到）

## 18.6 延伸阅读

- `internal/ingestion/component/extractor.go`、`extractor_tag.go`
- Gao et al., *Precise Zero-Shot Dense Retrieval without Relevance Labels*（HyDE）, 2022
- JSON Schema 规范：object、properties、enum、additionalProperties
