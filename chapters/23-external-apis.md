# 第 23 章 对外检索接口：REST、OpenAI 兼容、Dify 外部知识库、MCP

> 本章目标：让知识库被别的系统用起来。实现三种"别人家的协议"：OpenAI Chat Completions、Dify 外部知识库、MCP，并用真实的客户端验证。
> 前置：第 13、16、22 章。对应代码：`v1.0` 的 `backend/internal/api/openai.go`。

## 23.1 为什么要兼容别人的协议

前面所有的接口都是 RAGFlow 自己的格式（`{code, message, data}`）。可调用方往往已经有了自己的生态：

| 调用方 | 它会说的协议 | 我们提供 |
|---|---|---|
| 任何用 OpenAI SDK 写的应用 | `POST /chat/completions` | 把一个聊天助手伪装成一个"模型" |
| Dify 应用 | Dify 外部知识库 API | 把一个知识库伪装成 Dify 的知识库 |
| Claude Code、Cursor 等智能体 | MCP（JSON-RPC） | 把检索暴露成一个工具 |

三者共用同一套认证（第 22 章的 API Key）和同一条检索链路（第 13 章），只是"外壳"不同。

```mermaid
flowchart LR
    SDK[OpenAI SDK] -->|/openai/{chat_id}/chat/completions| A[聊天管线 第15章]
    Dify[Dify 应用] -->|/dify/retrieval| R[检索 第13章]
    Agent[Claude Code / Cursor] -->|/mcp JSON-RPC| R
    A --> R
```

## 23.2 RAGFlow 的做法

### OpenAI 兼容

路由 `POST /api/v1/openai/:chat_id/chat/completions`（`internal/router/router.go:330`）。URL 里的 `chat_id` 决定用哪个助手，请求体里的 `model` 字段只是原样写回响应。RAGFlow 专有的参数放在请求体的 `extra_body` 对象里（`service/openai_chat.go:126` 的 `OpenAIChatRequest`）：

- `reference: true`：返回引用。非流式放在 `choices[0].message.reference`（`:488`），流式放在最后一帧的 `delta.reference`，同时附上完整答案 `delta.final_content`（`:818`）；
- `reference_metadata`：引用里附带文档元数据；
- `metadata_condition`：第 19 章的元数据过滤。

处理函数还检查了"最后一条消息必须来自 user"。会话由客户端每次完整地发过来，服务端不保存。

#### 踩坑记录：官方 Python 示例的两个问题

用 OpenAI 官方 Python SDK 时，`extra_body=` 参数的含义是"把这些字段**合并到请求体顶层**"，而不是"发送一个叫 `extra_body` 的字段"。RAGFlow 的 `docs/references/python_api_reference.md:100` 示例是这样写的：

```python
client = OpenAI(api_key="ragflow-api-key", base_url="http://ragflow_address/api/v1/openai/<chat_id>/chat")
...
    extra_body={
        "reference": reference,
        ...
```

作者用一个本地的回显服务器看了看 SDK（openai 3.26.1）实际发出的请求（真实输出）：

```text
base_url=.../api/v1/openai/CHAT/chat extra_body={'reference': True}
  -> path /api/v1/openai/CHAT/chat/chat/completions keys ['messages', 'model', 'reference'] reference=True extra_body=None
base_url=.../api/v1/openai/CHAT extra_body={'extra_body': {'reference': True}}
  -> path /api/v1/openai/CHAT/chat/completions keys ['extra_body', 'messages', 'model'] reference=None extra_body={'reference': True}
```

1. SDK 会在 `base_url` 后面拼上 `/chat/completions`，示例的 `base_url` 以 `/chat` 结尾，路径就变成了 `.../chat/chat/completions`，Go 路由里没有这条路径；
2. `reference` 被放到了请求体顶层，而 Go 的 `OpenAIChatRequest` 只读取 `extra_body` 字段里的 `reference`。

所以，按 Go 后端的代码，正确的写法是 `base_url` 到 `<chat_id>` 为止，并且把参数**再包一层**：`extra_body={"extra_body": {"reference": True}}`。`http_api_reference.md` 里用 curl 的示例本来就是嵌套的 `"extra_body": {...}`，那份是对的。（作者没有启动完整的 RAGFlow 去请求这两条路径，上面的结论来自 SDK 的实际请求和 Go 路由、请求结构的代码。）

### Dify 外部知识库

Dify 允许把"外部知识库"接到自己的应用里：Dify 调用你提供的 `POST /retrieval`，请求体里有 `knowledge_id`、`query`、`retrieval_setting {top_k, score_threshold}`，以及可选的 `metadata_condition`，返回 `{"records": [{content, score, title, metadata}]}`。

RAGFlow 的实现是 `DifyRetrievalHandler.Retrieval`（`internal/handler/dify_retrieval_handler.go:165`，路由 `router.go:734`）。它额外支持 GET 方式（参数放在查询字符串里），也支持 `use_kg`（叠加知识图谱检索的结果）。Dify 的元数据条件用的是 `name`/`comparison_operator`，handler 会转换成内部的 `key`/`op`（`toMetaFilterConditions`，`:103`）。

### MCP

MCP（Model Context Protocol）是智能体调用外部工具的协议，底层是 JSON-RPC 2.0。RAGFlow 的 MCP 服务在 API 进程内部（`docker-compose.yml` 的注释："MCP is served in-process by the API server (POST /mcp on the 9380 port)"），路由是 `router.go:234`。支持的协议版本（`internal/mcp/server.go:47`）：

```go
		SupportedProtocolVersions: []string{"2025-11-25", "2025-06-18", "2025-03-26", "2024-11-05"},
```

工具清单在 `internal/mcp/tools.json`，共 3 个：`ragflow_retrieval`（参数和 REST 检索接口几乎一一对应）、`ragflow_list_datasets`、`ragflow_list_chats`。

## 23.3 从零实现

三个处理函数都在 `backend/internal/api/openai.go`，一共约 330 行。

### OpenAI 兼容

```go
cfg := *c
cfg.PromptConfig.Quote = req.ExtraBody.Reference
...
final, err := s.chatPipeline(r).Run(r.Context(), &cfg, history, func(e chat.Event) error {
	if !req.Stream || e.Final {
		return nil
	}
	delta := map[string]any{"content": e.Delta}
	if first {
		delta["role"], first = "assistant", false
	}
	return send(delta, nil, nil)
})
```

直接复用第 15 章的聊天管线，只改了"外壳"：

- **是否要引用**由 `extra_body.reference` 决定，并且同时控制提示词里的引用规则（`Quote`）。不要引用时，模型就不会输出 `[ID:n]`，回答对普通 OpenAI 客户端来说是干净的文本；
- 流式输出是 OpenAI 的 `chat.completion.chunk` 格式，第一帧带 `role`，最后一帧带 `finish_reason: "stop"` 和 `usage`，然后是 `data: [DONE]`；
- 与 RAGFlow 一样，不保存会话。

### Dify

请求转换成第 13 章的 `retrievalReq`：`top_k` → `PageSize`，`score_threshold` → `SimilarityThreshold`，Dify 的运算符（`is`、`is not`、`before`、`after`）映射成第 19 章的 `=`、`≠`、`<`、`>`。

#### 踩坑记录：错误码

写这一章时对照 Dify 的文档，发现 mini-ragflow 的错误处理有两处不对。Dify 文档建议的错误码是：1001 表示 Authorization 头格式不对，1002 表示认证失败，2001 表示知识库不存在；Dify 根据非 200 的 HTTP 状态判断失败。而 mini-ragflow 原来：

- 缺少参数时返回了 `error_code: 1001`，但 1001 的含义是"认证头格式错误"；
- `/dify/retrieval` 挂在 `requireAuth` 后面，认证失败返回的是 mini-ragflow 自己的格式。先写测试，修复前的真实输出：

```text
    api_test.go:617: dify auth "": 401 {"code":109,"message":"Authentication error: API key is invalid!","data":null}
    api_test.go:617: dify auth "Token abc": 401 {"code":109,"message":"Authentication error: API key is invalid!","data":null}
    api_test.go:617: dify auth "Bearer ragflow-wrong": 401 {"code":109,"message":"Authentication error: API key is invalid!","data":null}
```

修复：Dify 的路由不经过 `requireAuth`，改用自己的 `difyAuth` 认证，按 Dify 的约定返回 403 和 1001/1002；缺少参数时只返回 400 和 `error_msg`。（RAGFlow 的 handler 用的也是它自己的错误码，比如认证失败返回 HTTP 401 和 RAGFlow 的 code，并没有使用 Dify 建议的这几个码。）

### MCP

MCP 的 Streamable HTTP 传输允许服务端直接用一个 JSON 回答 POST 请求，不需要 SSE。所以一个 `switch` 就够了：

| 方法 | 回答 |
|---|---|
| `initialize` | 协议版本 `2025-06-18`、能力 `{tools: {}}`、服务端信息 |
| `notifications/initialized` 等通知（没有 `id`） | HTTP 202，没有响应体 |
| `ping` | `{}` |
| `tools/list` | 一个工具 `ragflow_retrieval` |
| `tools/call` | 检索，结果作为一段 JSON 文本放进 `content` |

一个小设计：`tools/list` 返回的工具描述里，**动态列出了调用者能访问的知识库**（名字、id、描述）。智能体看了描述就知道有哪些知识库可以搜，不需要再调用一个"列出知识库"的工具，这正是 RAGFlow 的 `ragflow_list_datasets` 要解决的问题，这里换了一种更省一轮调用的方式。

不传 `dataset_ids` 时搜索全部知识库。不同的知识库可能使用不同的 embedding 模型，向量不能放在一起比较，所以 `retrieveAcrossModels` 按模型分组分别检索，再按分数合并。代码注释也写明了，不同模型的分数并不严格可比，这只是一个近似。

## 23.4 运行与验证

`TestExternalAPIs`（`internal/api/api_test.go`）覆盖了三个协议。下面用真实的客户端和真实模型（bge-m3 + qwen2.5:3b）验证，数据是第 22 章那份升级上来的数据。

### 又一个踩坑：过继来的助手打不开

第一次运行这个脚本时，OpenAI SDK 的调用直接失败了：

```text
{"code":102,"message":"You don't own the chat 9b151ac6908219e0d4b7e5642d385985.","data":null}
```

`GET /chats` 能列出这个助手，打开它却说"不属于你"。原因在第 22 章的"从 v0.5 升级"：`chat` 表把整个助手存成一个 JSON `body`，`body` 里**也有一个** `tenant_id`。过继数据时只更新了 `tenant_id` 列，`body` 里还是 `default`；列表按列查询，加载时却读的是 `body`。修复：过继时用 `json_set` 同时改 `body`，读取时以列为准（这样已经升级过的数据也能自愈）。回归测试 `TestFirstUserAdoptsDefaultData`，修复前的输出：

```text
    user_test.go:30: chat tenant: listed 1, body "default", want "3f4a875658499a7f55b749f8ba4740ba"
```

教训：**同一个事实存了两份，迟早会不一致**。第 22 章的 API 测试都是先注册再建助手，从来没有覆盖"升级"这条路径。

### 真实输出

修复后（`ext.py` 的输出；长 id 已截短）：

```text
chats: [('售后助手', 'qwen2.5:3b@openai')]
[openai non-stream] model: any | finish: stop
  content: 星河S2咖啡机的除垢步骤如下：  1. 每天清空渣盒和滴水盘。 2. 每500杯或每3个月进行一次除垢。 3. 使用1包S2专用除垢粉，加温水（1.5升），倒入水箱。 4. 在设置菜单中选择"维护 → 自动除垢"。 5. 机器自动循环约35分钟，期间请勿关机。 6. 完成后清洗水箱，按提示自动冲洗两次。  这些步骤可以在[ID:0]和[ID:1]中找到详细说明。
  reference: 6 chunks; first keys: ['content', 'dataset_id', 'document_id', 'document_name', 'id', 'similarity']
  usage: 935 157 1092
[openai stream] 星河S1咖啡机的保修信息如下：

- 整机保修 2 年。
- 加热组件保修 3 年。

因此，星河S1咖啡机的整机保修期为2年，加热组件的保修期为3年。
  (54 content chunks)
[dify] 200 {"records": [{"content": "# 星河 S2 全自动咖啡机用户手册\n## 3. 清洁与除垢…", "metadata": {"chunk_id": "4c3af158…", "document_id": "e01d5478…"}, "score": 0.6193829096636667, "title": "星河S2咖啡机用户手册.md"}, {"content": "星河S2用户手册：每日清空渣盒和滴水盘，每500杯或3个月除…", ..., "score": 0.608270986318429, ...}]}
[dify auth] (none) 403 {'error_code': 1001, 'error_msg': "Invalid Authorization header format. Expected 'Bearer <api-key>' format."}
[dify auth] ['Token ragflow-95b3…'] 403 {'error_code': 1001, ...}
[dify auth] ['Bearer ragflow-wrong'] 403 {'error_code': 1002, 'error_msg': 'Authorization failed'}
[dify] missing kb 404 {'error_code': 2001, 'error_msg': 'The knowledge does not exist'}
[mcp initialize] {'capabilities': {'tools': {}}, 'protocolVersion': '2025-06-18', 'serverInfo': {'name': 'mini-ragflow', 'version': '1.0'}}
[mcp tools/list] ragflow_retrieval | Retrieve relevant chunks from the knowledge bases. Omit dataset_ids to search all of them. Available datasets: / - 咖啡机手册 (id be38ebfc…): 
[mcp tools/call] 0.619 星河S2咖啡机用户手册.md | # 星河 S2 全自动咖啡机用户手册 ## 3. 清洁与除垢 每天清空渣盒和滴水
[mcp tools/call] 0.608 星河S2咖啡机用户手册.md | 星河S2用户手册：每日清空渣盒和滴水盘，每500杯或3个月除垢一次，用1包S2专
```

几点观察：

- 非流式回答里带 `[ID:0]`、`[ID:1]`，因为请求了 `reference`；流式请求没有带，回答里就没有引用标记；
- qwen2.5:3b 把"每天清空渣盒"也列成了除垢的第 1 步，原文里这是日常清洁，不属于除垢流程。引用是对的，归纳得不够准确；
- Dify 的第二条记录是 RAPTOR 摘要块（第 20 章），对外部调用方来说，它和普通 chunk 没有区别。

另外两个客户端在 v1.0 开发时验证过：MCP 官方的 Inspector（CLI 模式，0.17.2）对 `tools/list`、`tools/call` 都能正常工作；Dify 只用 `httptest` 模拟验证过，作者没有部署一个真的 Dify。

## 23.5 练习

1. **巩固**：用 curl 发一个流式的 OpenAI 请求，带上 `"extra_body": {"reference": true}`，找到最后一帧里的 `reference`。
2. **扩展**：给 MCP 加上 RAGFlow 的另外两个工具 `ragflow_list_datasets` 和 `ragflow_list_chats`，然后在 Claude Code 里用 `claude mcp add --transport http` 接入，让它回答一个手册里的问题。
3. **读源码**：RAGFlow 的 OpenAI 兼容接口在流式的最后一帧里同时返回了 `reference` 和 `final_content`。既然内容已经一帧一帧发过了，为什么还要再发一次完整的答案？（提示：第 16 章，引用修复会改写答案。）

## 23.6 延伸阅读

- `internal/service/openai_chat.go`、`internal/handler/dify_retrieval_handler.go`、`internal/mcp/`
- OpenAI API 文档：Chat Completions、Streaming
- Dify 文档：External Knowledge API
- MCP 规范：Transports（Streamable HTTP）、Tools
