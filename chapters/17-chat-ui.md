# 第 17 章 聊天 UI：引用悬浮与原文定位

> 本章目标：做一个能流式显示回答、把 `[ID:n]` 变成可悬停引用、能列出来源文档的聊天界面。
> 前置：第 16 章。对应代码：`v0.4` 的 `web/src/pages/{Chats,ChatSettings,ChatView}.tsx`、`web/src/api.ts`。

## 17.1 界面要做的四件事

1. **读流**：用 `fetch` 读 SSE，按 `\n\n` 切帧，把增量拼到最后一条消息上；
2. **渲染引用**：把回答里的 `[ID:n]` 替换成一个小徽标，鼠标悬停时显示第 n 块的原文；
3. **列出来源**：回答下面列出被引用的文档，点击可以查看原文件；
4. **管理会话**：左侧是会话列表，新建、切换、删除。

## 17.2 RAGFlow 前端怎么做

### 读流：限频合并写入

`runChatCompletionStream`（`web/src/pages/next-chats/chat-stream/run-stream.ts:31`）负责读流。有一个细节：增量来得比界面刷新快得多，每来一个就写一次状态，React 会疲于奔命。所以它把增量先放进一个缓冲，**最多每 60 毫秒**往 store 里写一次（`AnswerFlushIntervalMs = 60`，`chat-stream/constants.ts:14`）：

```ts
// Chunks arrive faster than the UI can use them, so they're coalesced into a
// single store write per AnswerFlushIntervalMs. Fields are merged rather than
// replaced so a `reference` or `prompt` that arrived on a skipped chunk isn't
// lost; ...
```

这和后端"进度每秒最多写一次库"（第 6 章）是同一个思路：**高频事件和低频消费之间，需要一层合并**。

### 渲染引用

`components/markdown-content/index.tsx` 的 `renderReference`（`:333`）用 `reactStringReplace` 把引用标记替换成 HoverCard：

```tsx
const chunkIndex = getChunkIndex(match);
...
const hasReference = !!reference?.chunks?.[chunkIndex];
// Explicit markers can render while their sources are still arriving.
if (!hasReference && !(loading && match.startsWith('[ID:'))) {
  return match;
}

return (
  <HoverCard key={i}>
    <HoverCardTrigger>
      <bdi className="...">
        [{chunkIndex + 1}]
      </bdi>
    </HoverCardTrigger>
    {hasReference && (
      <HoverCardContent className="max-w-3xl">
        {getPopoverContent(chunkIndex)}
      </HoverCardContent>
    )}
  </HoverCard>
);
```

三个细节：

- **显示的编号是 n+1**：`[ID:0]` 显示成 `[1]`，用户习惯从 1 开始数；
- **流式过程中**，引用列表要到最后一帧才会到达，这时标记先渲染成徽标，悬停内容等引用到了再出现；
- **`<bdi>`**（双向隔离）：阿拉伯语、希伯来语这类从右往左书写的文本里混入 `[3]`，`<bdi>` 能防止数字把文字方向搅乱。引用规则里专门提到了 RTL 语言（第 16 章）。

## 17.3 从零实现

### 读流

`web/src/api.ts` 的 `streamCompletion`：

```ts
const reader = res.body.getReader()
const decoder = new TextDecoder()
let buf = ''
for (;;) {
  const { value, done } = await reader.read()
  if (done) break
  buf += decoder.decode(value, { stream: true })
  let i
  while ((i = buf.indexOf('\n\n')) >= 0) {
    const frame = buf.slice(0, i)
    buf = buf.slice(i + 2)
    if (!frame.startsWith('data:')) continue
    const env = JSON.parse(frame.slice(5))
    if (env.code !== 0) throw new ApiError(env.code, env.message)
    if (env.data === true) return
    onEvent(env.data as CompletionEvent)
  }
}
```

`decoder.decode(value, { stream: true })` 里的 `stream: true` 不能少：一个汉字是 3 个字节，可能被拆到两次 `read()` 里，不加这个参数会解码出乱码。为什么不用浏览器自带的 `EventSource`？因为它只支持 GET，不能带请求体，也不能自定义请求头（v1.0 要带 `Authorization`）。

mini-ragflow 没有做 60 毫秒的合并，每帧直接更新一次状态。本书场景下的增量频率不高（mock 模型每 4 个字一帧），作者没有专门测量这样做的渲染开销，留作练习 1。

### 渲染引用

`ChatView.tsx` 的 `Answer` 组件用一个正则切分回答，把标记换成徽标：

```tsx
const parts = text.split(/(\[ID:\d+\])/g)
...
const m = p.match(/^\[ID:(\d+)\]$/)
if (!m) return <Fragment key={i}>{p}</Fragment>
const n = Number(m[1])
const chunk = reference?.chunks[n]
return (
  <span key={i} className="cite">
    {n + 1}
    {chunk && (
      <span className="cite-pop">
        <b>{chunk.document_name}</b> · 相似度 {chunk.similarity.toFixed(3)}
        <pre>{chunk.content}</pre>
      </span>
    )}
  </span>
)
```

悬停卡片用纯 CSS（`.cite:hover .cite-pop { display: block }`）实现，不依赖组件库。`Sources` 组件只列出**真正被引用了的**文档，而不是检索到的全部文档，这和 RAGFlow 的行为一致。

### 踩坑记录 1：页面加载就崩溃

聊天页第一次跑 UI 脚本时，页面一直停在"生成中…"，按钮再也没恢复。直接用 curl 请求后端，流是完整的。在 headless Chrome 里打印页面错误，并用不压缩的构建看调用栈：

```text
pageerror: TypeError: destroy_ is not a function |     at commitHookEffectListUnmount (…)
```

React 在卸载时，把一个 effect 的返回值当作清理函数调用，而那个返回值不是函数。罪魁祸首是这一行：

```tsx
useEffect(() => bottom.current?.scrollIntoView({ block: 'end' }), [current])
```

箭头函数没有花括号，于是 `scrollIntoView()` 的返回值成了 effect 的返回值。在 MDN 的文档里，`scrollIntoView` 返回 `undefined`；但在作者这台机器的 Chrome 152 上：

```text
$ node e2e/_probe.mjs     # document.getElementById('x').scrollIntoView() 的返回值
[object Promise]
```

它返回了一个 Promise。修复很简单，加上花括号，让 effect 不返回任何东西：

```tsx
useEffect(() => {
  bottom.current?.scrollIntoView({ block: 'end' })
}, [current])
```

教训：effect 里只写副作用，不要写成表达式箭头函数。浏览器 API 的返回值也是会变的。

### 踩坑记录 2：悬浮卡片被裁掉

第一版的引用卡片向上弹出（`bottom: 120%`）。第一条消息的引用一弹出，就被消息列表容器的顶部裁掉了（容器设了 `overflow: auto`）。改成向下弹出（`top: 140%`）后正常。

## 17.4 运行与验证

```bash
git checkout v0.4 && cd web && npm run build
cd ../backend && go run ./cmd/mini-ragflow --data /tmp/v04 --static ../web/dist --mock-delay 5ms &
cd ../web && node e2e/ui-v04.mjs http://localhost:9380 /tmp/shots ../samples
```

`--mock-delay 5ms` 让 mock 模型每帧之间停 5 毫秒，在界面上能看到"打字"的效果。mock 模型的真实输出：

```text
Q: 咖啡机怎么除垢？
A: 根据知识库：当除垢指示灯亮起时（通常每 200 杯或每 2 个月），请使用专用除垢剂按以下步骤除垢： 1。1. 将除垢剂与 1 升清水混合后倒入水箱 1。
   来源：
星河咖啡机用户手册.md
Q: 年假没休完可以折现吗？
A: 根据知识库：按制度应在次年 3 月 31 日前休完，逾期作废，不折现 1。
   来源：
faq.md
Q: 量子计算机多少钱？
A: Sorry! No relevant content was found in the knowledge base!
   (no sources)
```

（`innerText` 里的 `1` 就是渲染出来的引用徽标。）

换成真实模型（`bge-m3` + `qwen2.5:3b`，跑在 CPU 上的 Ollama）：

```bash
node e2e/ui-v04.mjs http://localhost:9380 /tmp/shots ../samples bge-m3@openai qwen2.5:3b@openai
```

![图 17-1：真实模型下的聊天页。回答里的数字是引用，悬停可以看到“清洁与除垢”那一块的原文。](./images/v04-real-qwen2.5-3b-chat.png)

*图 17-1：真实模型（bge-m3 + qwen2.5:3b）。第一条回答引用了"清洁与除垢"那一块（悬停卡片），第三个问题检索不到任何内容，直接返回了空回复。*

一次运行中，qwen2.5:3b 对第一个问题的回答是（来自 UI 脚本输出）：

```text
A: 星河 S1 咖啡机的除垢步骤如下：  1 1. 将除垢剂与1升清水混合后倒入水箱。 1 2. 同时按住单杯键和双杯键5秒进入除垢模式。 1 3. 机器会自动分3次排出溶液，全程约20分钟。 4. 用清水冲洗水箱，再空冲两箱清水。  这是星河 S1 咖啡机的除垢指示灯亮起时的正确操作流程。 1
   来源：
星河咖啡机用户手册.md
```

步骤和原文一致，引用也指对了块。真实模型的输出每次都不一样，上面截图里的那次回答措辞就有所不同。

## 17.5 练习

1. **巩固**：仿照 RAGFlow，把 `streamCompletion` 的状态更新改成每 60 毫秒合并一次，用 Chrome 的 Performance 面板对比改动前后 React 的渲染次数。
2. **扩展**：点击引用徽标时，打开原文件并滚动到对应位置。后端需要提供什么？（提示：chunk 的页码 `page_num`，以及 RAGFlow 的 `positions` 坐标。）
3. **读源码**：RAGFlow 的 `getPopoverContent` 除了文本，还会显示什么？`doc_aggs` 在聊天界面里是怎么用的？

## 17.6 延伸阅读

- `web/src/components/markdown-content/index.tsx`
- `web/src/pages/next-chats/chat-stream/`
- React 文档：`useEffect` 的返回值必须是清理函数或 `undefined`
