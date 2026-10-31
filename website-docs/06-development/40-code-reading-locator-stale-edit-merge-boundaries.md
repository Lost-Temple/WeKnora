# Locator 失效、编辑继承与检索合并边界阅读指南

这一篇接着 [39-code-reading-source-locator-generation-remap.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/39-code-reading-source-locator-generation-remap.md) 和 [38-code-reading-chat-reference-source-location.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/38-code-reading-chat-reference-source-location.md) 中间最容易读岔的一段，专门回答三个问题：

- 为什么一个 chunk 被手工编辑后，旧 locator 要直接失效？
- 为什么检索结果合并时，不能继续信 edited / rewritten 结果的 `StartAt/EndAt`？
- 为什么 citation source 必须退回到原始 chunk 自己的正文和 locator，而不能沿用扩展上下文？

这条链的核心结论是：

- **检索可继续，旧原文坐标未必可继续**
- **stale 的本质不是“这条知识没了”，而是“旧坐标不再足以证明原文位置”**
- **合并和引用展示都必须把“召回上下文”和“证据原文”严格分开**

## 推荐阅读顺序

1. [internal/application/service/chunk.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chunk.go)
2. [internal/application/service/chat_pipeline/search_entity.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chat_pipeline/search_entity.go)
3. [internal/application/service/chat_pipeline/merge_overlap.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chat_pipeline/merge_overlap.go)
4. [internal/application/service/chat_pipeline/citation_sources.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chat_pipeline/citation_sources.go)
5. [frontend/src/components/ChatReferenceSourceView.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/ChatReferenceSourceView.vue)

## 1. 编辑 chunk 后，为什么不是“重新算页码”，而是先清空 locator

入口在 [UpdateDocumentChunk](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chunk.go)。

这里有个很容易误解的点：

- 用户手工编辑正文
- 不代表这条知识不能继续参与检索
- 但通常代表“旧的 parser 坐标已经不能继续证明这段话来自上传原文的这一处”

所以系统的处理不是：

- 硬保留旧 locator
- 或试图在编辑后的正文上强行猜一个新页码

而是更保守地做两件事：

1. `chunk.SourceLocators = nil`
2. 继续保存当前正文并重新索引

也就是说：

- **知识正文继续可搜**
- **旧引用回原文要降级成 stale**

这套策略的好处是很明确的：

- 不会把用户带到一个“看起来像，但其实已经不是原始出处”的位置

## 2. `ContentRevision` 和 `SourceLocators` 这两个字段要一起看

从 [search_entity.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chat_pipeline/search_entity.go) 往下看，`SearchResult` 会直接带上：

- `ContentRevision`
- `SourceLocators`
- `StartAt/EndAt`

这几个字段后面分别承担不同职责：

- `ContentRevision`：告诉前端“这段正文是不是后来被编辑过”
- `SourceLocators`：告诉前端“当前这一版正文还有没有原文定位证据”
- `StartAt/EndAt`：只在坐标仍可信时，参与结果合并

所以不能把 `ContentRevision > 0` 简单理解成：

- 一定不能打开原文

真正更可靠的判断是：

- **正文已编辑，且当前 locator 已经清空或失配**

这时前端才会把它标成 `stale`。

## 3. parent_text 为什么也会跟着失效

[rebuildParentContent](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chunk.go) 处理的是：

- 某个 text child 被手工编辑后
- 如何把修改叠回它所属的 `parent_text`

这里的关键区分是：

- `parent.SourceContent`：始终保留最初 parser 看到的整段原文
- `parent.Content`：把若干 child 的用户编辑叠上去后的“现行父正文”

只要 child 有编辑，`parent.Content` 就不再是上传原文的逐字拷贝。

所以这里会明确执行：

- `parent.SourceLocators = nil`

这表示：

- 原始父 locator 作为历史事实仍然存在于 `SourceContent` 的语义侧
- 但不能继续对着现在这份 parent.Content 直接宣称“原文就在这里”

这也是前端 parent fallback 那么保守的原因之一。

## 4. 为什么检索合并不能继续信 stale 坐标

[merge_overlap.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chat_pipeline/merge_overlap.go) 是另一层很重要的边界。

它做的是：

- 把顺序相邻的 chunk 合并成更完整的回答上下文

但这里首先要判定：

- `StartAt/EndAt` 还能不能继续拿来算 overlap

这个判断落在：

- `chunkTrusted`

条件很严格：

- `ContentRevision == 0`
- `!ContentRewritten`
- `EndAt > StartAt`
- `runeLen(Content) == EndAt-StartAt`

只要不满足，就退回文本路径。

### 为什么要这么严

如果一个 chunk：

- 手工编辑过
- 或经过 pipeline 改写
- 或正文长度已经和原坐标对不上

这时再按照旧坐标去裁 overlap，风险很大：

- 可能把真实正文截掉
- 也可能把重复表格行、日志行错当成 overlap

所以这里的原则和 locator 失效是一致的：

- **宁可退回文本合并，也不继续信旧坐标**

## 5. 合并后为什么还保留 locator 并集

`mergeSequentialChunks` 在合并内容时，仍然会执行：

- `sourceloc.MergeLocators(...)`

这里看起来像是矛盾，其实不是。

它区分了两件事：

- 这些 locator 能不能继续拿来做“位置剪裁”
- 这些 locator 要不要继续保留给 citation 打开原文

前者需要 `chunkTrusted`

后者不一定。

所以合并后的结果会：

- 保留 locator 并集，尽量让前端还能打开到相关原文范围
- 但下一次再拿这些结果做位置合并时，仍要重新经过 `chunkTrusted`

## 6. 为什么 citation source 不能沿用 expanded context

[citation_sources.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/chat_pipeline/citation_sources.go) 讲的是另一个常见误区：

- 检索排序时，用的是扩展上下文
- 但模型和前端真正需要看到的“证据原文”，不能直接把那整段扩展上下文当成 citation

原因很简单：

- 一个 parent 的邻近段落，不等于 child 自己的证据
- OCR 文本可能是 image child 自己的证据，不是 text parent 的正文

所以这里会把 expanded result 再拆回多个真正的 source：

- 自己的 ID
- 自己的正文
- 自己的 `SourceLocators`
- 自己的 `ContentRevision`

这样后面前端在做：

- stale 判断
- parent fallback
- image context

时，才不会混淆“召回上下文”和“证据来源”。

## 7. 前端为什么把“edited + no locator”当成 stale，而不是缺数据

[ChatReferenceSourceView.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/ChatReferenceSourceView.vue) 里这条判断很关键：

- `content_revision > 0 && source_locators 为空`

它表达的不是：

- 后端没返回完整字段

而是：

- 后端明确告诉前端：正文已经是人工修改后的版本，旧原文坐标不该再信

所以前端会：

- 继续展示这条引用
- 允许用户打开文档
- 但把定位能力降级成 `stale`

这和“纯粹没有 locator 的旧消息”不是一个语义。

## 8. 为什么 parent fallback 还要额外检查 exact/source_hash

即使当前 child 已 stale，前端也不会随便拿 parent 顶上去。

它还要求：

- parent 未编辑
- parent 正文里能逐字找到当前答案句子
- parent locator 全部 `mapping=exact`
- 每个 locator 都带 `source_hash`

也就是说，parent fallback 不是在“补一个看起来差不多的定位”，而是在非常保守地恢复：

- 某些旧历史消息原本存在、后来因为 citation 绑在 child 上而丢失的稳定证据

一旦这些条件不满足，就宁可维持 `stale`。

## 9. 这一段最容易看错的地方

### 第一，stale 不等于无效知识

stale 只说明：

- 旧坐标不再足以证明原文位置

不说明：

- 这条知识不能检索
- 或这段正文一定错误

### 第二，合并结果里的 locator 还在，不代表坐标还能继续拿来算 overlap

合并时保留 locator，是为了 citation 打开原文。

是否能继续做位置合并，还要重新过 `chunkTrusted`。

### 第三，expanded context 不是 citation source

expanded context 是为召回和排序服务的；

真正的 citation source 必须退回到：

- 原始命中的 chunk
- 或能证明自己就是证据体的 image/text child

## 10. 推荐连读方式

如果你要把 locator 这条链完整读顺，建议这样连：

1. 先看 [39-code-reading-source-locator-generation-remap.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/39-code-reading-source-locator-generation-remap.md)，确认 locator 是怎么生成出来的
2. 再看本篇 [40-code-reading-locator-stale-edit-merge-boundaries.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/40-code-reading-locator-stale-edit-merge-boundaries.md)，确认哪些情况下它必须失效或降级
3. 最后看 [38-code-reading-chat-reference-source-location.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/38-code-reading-chat-reference-source-location.md)，确认前端怎样消费 `stale / partial / found`

这样读下来，就能把“生成 locator -> 失效边界 -> 前端消费”这三层彻底分开。
