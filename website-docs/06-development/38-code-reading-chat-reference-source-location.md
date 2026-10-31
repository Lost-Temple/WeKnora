# 问答引用原文定位阅读指南

这一篇接着 [11-code-reading-session-rag-references.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/11-code-reading-session-rag-references.md) 往下读，专门回答一个更细的问题：

- 回答里出现的“关联文档 / 引用片段”，到底能不能回到原文位置？
- 页码、段落、表格行、文本偏移这些信息存在哪里？
- 为什么有的引用能精确定位，有的只能打开原文但不能直接落到一句话？

这条链的核心结论是：

- **后端确实保存了原文定位信息**
- **前端也已经有“打开来源文档并定位”的能力**
- **但定位是分层降级的，不是所有引用都能稳定精确到一句**

## 推荐阅读顺序

1. [internal/types/search.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/search.go)
2. [internal/types/source_locator.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/source_locator.go)
3. [frontend/src/utils/referenceSources.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/utils/referenceSources.ts)
4. [frontend/src/composables/useChatReferencesDrawer.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useChatReferencesDrawer.ts)
5. [frontend/src/components/ChatReferencesDrawer.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/ChatReferencesDrawer.vue)
6. [frontend/src/components/ChatReferenceSourceView.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/ChatReferenceSourceView.vue)
7. [frontend/src/utils/sourceLocator.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/utils/sourceLocator.ts)
8. [frontend/src/components/document-preview.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/document-preview.vue)

## 1. 定位信息最早挂在哪里

入口在 [SearchResult](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/search.go)：

- `KnowledgeReferences` 里的每一条结果，本质上都是一个 `SearchResult`
- `SearchResult.SourceLocators` 就是“这条引用如何回到原始文件”的定位载体

也就是说，引用抽屉看到的不只是：

- 文档标题
- 摘录片段

还可能带着：

- PDF 页码和框选区域
- Word 第几个段落或表格块
- Excel 第几行到第几行
- 纯文本的字符偏移

所以“能不能回原文”不是前端临时猜出来的，而是检索结果里本来就可能带着坐标。

## 2. `SourceLocator` 真正在描述什么

[source_locator.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/source_locator.go) 里把定位分成了几类：

- `pdf`
- `docx`
- `slide`
- `sheet`
- `text`
- `time`
- `section`

最重要的几个字段是：

- `mapping`
- `partial`
- `source_hash`
- 以及各文件类型自己的结构坐标

### `mapping`

`mapping=exact` 才代表这条定位已经过“正文 -> 原始文件”的严格对齐，可以直接拿来做精确落位。

如果这个字段为空，通常意味着：

- 这是历史遗留 locator
- 或者只保留了结构级位置，没有足够强的逐字对齐保证

### `partial`

`partial=true` 不表示 locator 失效，而是表示：

- 只能把用户带到一个大致区域
- 不能承诺整个 chunk 都有完整映射

典型场景是：

- OCR
- 跨页内容
- 解析过程中只有一部分正文能和原始文件稳定对齐

### `source_hash`

这让前端能判断：

- 当前引用里的 locator
- 还是不是针对现在这份原始文件生成的

如果文件后来被重新解析、重建或编辑，旧 hash 对不上时，前端会把这次定位降级成 `stale`，而不是硬跳。

## 3. 为什么引用抽屉按文档聚合后，仍然能打开到具体位置

[referenceSources.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/utils/referenceSources.ts) 解决的是一个常见矛盾：

- 后端引用常常是 chunk 级别的
- 但抽屉展示更适合按“文档”聚合

它的做法是：

1. 把同一文档的多个 chunk 聚成一张文档卡片
2. 同时保留：
   - `chunk_ids`
   - `source_chunk_id`

这里的 `source_chunk_id` 很关键：

- 抽屉层是文档卡片
- 但“打开来源文档”时，仍然需要回到一个真的带 locator 的 chunk

所以它保留的是：

- **聚合展示**
- **但不丢原文定位入口**

## 4. 为什么有时点 citation 直接进原文，有时只是打开引用列表

[useChatReferencesDrawer.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useChatReferencesDrawer.ts) 把抽屉分成两种模式：

- 引用列表模式
- 原文预览模式

规则并不复杂：

- 普通打开抽屉：先看列表
- 正文里的 citation 点击：如果能解析出具体 `ReferenceSourceTarget`，就直接进原文预览

但这个解析有边界。

[resolveReferenceSource](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/utils/referenceSources.ts) 会直接拒绝这些类型：

- `web_search`
- `tool_result`
- `faq`

因为它们没有“原始上传文件里的定位点”。

所以它们最多只能：

- 高亮抽屉卡片
- 或打开网页

不能走本地文档预览器那条“页内定位”链路。

## 5. `ChatReferenceSourceView` 在做什么

[ChatReferenceSourceView.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/ChatReferenceSourceView.vue) 是“引用卡片”到“真正原文预览”的桥。

它做了四件关键事：

### 第一，补齐当前文件元信息

- 文件名
- 文件类型
- 来源
- 当前 `file_hash`

这一步是为后面的 stale 判断做准备。

### 第二，尽量拿到“当前 chunk 的最新 locator”

即使历史消息里已经带了引用快照，它仍然会优先再读一次当前 chunk。

原因是：

- 历史消息只是当时的快照
- 当前知识内容可能已经被重新解析或编辑

所以这层的原则是：

- **能用当前 chunk，就不用历史快照**

### 第三，处理旧历史消息的 parent chunk 回补

这是最容易误判的一段。

旧版扩展上下文里，citation 可能绑在子 chunk 上，但真正稳定的精确 locator 在 `parent_text`。

这里不是无条件回补，而是很严格地要求：

- 父 chunk 与当前 knowledge 对得上
- 父 chunk 没被编辑过
- 答案里的原句能在父 chunk 正文里逐字找到
- 父 locator 全是 `mapping=exact`，且带 `source_hash`

只有这样才允许把定位切回父 chunk。

换句话说，这里追求的不是“尽量命中”，而是：

- **宁可不跳，也不能跳错**

### 第四，构造统一的 `SourceLocateRequest`

`buildLocate()` 会把三层信息一起交给预览器：

- `locators`：结构化坐标
- `quotes`：文本证据
- `scope/sourceMarkdown`：失败时的兜底语料

同时还会先做一次 `unavailable` 判断：

- 当前 chunk 已编辑且没有可靠 locator
- 或 `source_hash` 对不上

这种情况就不再尝试“旧文本猜位置”，而是直接告诉 UI 这是 stale。

## 6. `sourceLocator.ts` 为什么这么重视“精确原句”

[sourceLocator.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/utils/sourceLocator.ts) 不是简单的字符串搜索工具，它在做的是：

- 规范化文本
- 用答案句子挑 locator
- 用 locator.quote 再限制可接受的匹配范围

这里最关键的两个函数是：

- `selectLocatorsForSentence`
- `selectQuoteForSentence`

它们的共同原则是：

- 只有当答案句子能在来源文本里找到明确原句时，才允许继续缩小

否则就退回更宽的范围。

这能避免一种很常见的假命中：

- 答案里是一句模型改写后的总结
- 文档里有很多相似句
- 如果直接全文模糊搜，就可能跳到另一个“看起来也差不多”的段落

## 7. `document-preview.vue` 怎样真正执行落位

[document-preview.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/document-preview.vue) 是最后执行定位的地方。

它遵循统一优先级：

1. 先吃结构坐标
2. 再在结构范围内用 quote / sentence 缩小
3. 最后才退回全文文本匹配

但不同文件类型的实现各不一样。

### PDF

PDF 走自己的预览器逻辑，`document-preview.vue` 只负责把 locate 请求转进去，并通过事件拿回结果。

### DOCX

先用：

- `block`
- `quote`
- `sentence`

在 Word 段落/表格块里逐步缩小。

如果是图片引用，还会优先尝试：

- `imageDigest`
- `imageContext`

### Excel

按：

- `sheet`
- `row_start`
- `row_end`

映射到具体表格行。

### Text / Markdown / HTML

优先相信：

- `start/end` 偏移

但也会再次校验：

- 偏移截出来的文本里是否真的包含 `quote`

如果不成立，就说明旧 offset 不能直接复用，只能退回文本搜索。

### Audio

走：

- `start_ms`
- `end_ms`

本质上不是高亮文本，而是把播放器 seek 到对应时间段。

## 8. 为什么有些引用只能到“页”或“块”，不能到“句”

前端定位结果里有几种 UI 状态：

- `found`
- `page`
- `block`
- `partial`
- `stale`
- `missing`
- `ambiguous`

它们不是“前端渲染样式”，而是在告诉你：

- 这次定位的可信度到底有多高

最常见的几种情况是：

### `found`

说明：

- 结构坐标和文本证据都对上了

这是最理想的“精确到原句”。

### `page` / `block`

说明：

- 能确定页、段、表格行或某个结构区域
- 但没法稳定缩到一句

这通常不是 bug，而是证据粒度本来就只到这一层。

### `stale`

说明：

- locator 仍然像是真的
- 但来源文件 hash 或 chunk 修订已经变了

系统这时刻意不继续猜，避免旧坐标误伤新文档。

### `missing` / `ambiguous`

说明：

- 要么完全找不到
- 要么能找到不止一个候选

这两种都不能被当成精确定位。

## 9. 这条链最容易看错的地方

### 第一，别把“有摘录”当成“必然可精确定位”

有摘录只说明：

- 引用里保留了证据文本

不说明一定有：

- 精确结构坐标
- 未过期的 source hash

### 第二，别把“打开原文成功”当成“定位精确成功”

能打开原文只是第一层。

真正是否精确，要看最后是：

- `found`
- 还是 `page/block/partial`

### 第三，历史消息回补不是随便向上找 parent chunk

`ChatReferenceSourceView` 那段 parent 回补非常保守。

它不是为了提高命中率，而是为了只在“证据仍然逐字成立”时，恢复旧历史消息的精确定位能力。

## 10. 推荐连读方式

如果你要完整读懂“回答里的引用是怎么落到原文上的”，建议按这个顺序连读：

1. 先看 [11-code-reading-session-rag-references.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/11-code-reading-session-rag-references.md)，确认引用是怎么从问答结果流到前端抽屉的
2. 再看 `search.go` 和 `source_locator.go`，确认 locator 数据到底长什么样
3. 再看 `referenceSources.ts` 和 `useChatReferencesDrawer.ts`，确认抽屉怎么从“文档卡片”切到“原文预览”
4. 最后看 `ChatReferenceSourceView.vue`、`sourceLocator.ts`、`document-preview.vue`，确认真正的原文落位和降级规则

这样读下来，你会把“引用展示”和“原文定位”这两层彻底分开，不容易把它们混成一件事。
