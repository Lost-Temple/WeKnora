# 回答引用清单组装与传递阅读指南

这一篇接着 [40-code-reading-locator-stale-edit-merge-boundaries.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/40-code-reading-locator-stale-edit-merge-boundaries.md) 往外层走，专门回答一个产品层的问题：

- 回答下面显示的“关联文档 / 网页引用 / tool 引用”清单，到底是怎么从检索结果一路变成前端消息字段的？
- 为什么回答正文还没出来时，引用抽屉就可以先打开？
- 为什么历史重开后，引用清单还能继续用，而不是只在 live stream 那一刻存在？

这条链的核心结论是：

- **回答引用清单是独立于正文 token 流的一条结构化侧链**
- **它的主口径是 `knowledge_references`，不是临时的 `data.references`**
- **前端引用抽屉优先读取消息上的聚合清单，只有缺失时才回退到 tool event 兜底**

## 推荐阅读顺序

1. [internal/application/service/session_knowledge_qa.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_knowledge_qa.go)
2. [internal/types/chat.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/chat.go)
3. [internal/types/message.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/message.go)
4. [internal/handler/session/helpers.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/helpers.go)
5. [frontend/src/composables/useChatStreamHandler.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useChatStreamHandler.ts)
6. [frontend/src/views/chat/components/docInfo.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/views/chat/components/docInfo.vue)
7. [frontend/src/views/chat/components/AgentStreamDisplay.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/views/chat/components/AgentStreamDisplay.vue)
8. [frontend/src/components/ChatReferencesDrawer.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/ChatReferencesDrawer.vue)

## 1. 回答引用清单最早来自哪里

入口在 [emitKnowledgeReferencesEvent](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_knowledge_qa.go#L1319-L1340)。

它发出的不是：

- 原始搜索候选
- rerank 前的草稿结果
- 或模型正文里内联出来的 citation tag

而是：

- **这一轮回答最终采用的引用清单**

也就是：

- `MergeResult`

这层非常关键，因为它说明引用清单在产品语义上是：

- 一份已经过 merge / top-k / 回答组装后确认下来的结构化结果

而不是“边搜边试”的中间态。

## 2. 为什么要把引用清单单独发一个 `references` 事件

`KnowledgeQA` 里这条链是：

1. 检索、rerank、merge
2. 先发 `references`
3. 再进入 `CHAT_COMPLETION_STREAM`

这么设计的原因是：

- 引用清单和正文 token 是两种完全不同的内容

正文 token 的特点是：

- 流式、增量、可能被模型截断或 fallback

引用清单的特点是：

- 结构化
- 一次性
- 可以独立展示

所以它适合单独作为一个 SSE 事件先送给前端。

这也是为什么在很多场景里：

- 回答正文还没开始输出
- 右侧引用抽屉已经可以先打开

## 3. `References` 为什么只是 `[]*SearchResult`，却又要单独起一个名字

[types/chat.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/chat.go) 里：

- `type References []*SearchResult`

看起来像只是一个别名，但它承担的是更稳定的产品语义：

- live SSE 的 `knowledge_references`
- assistant message 持久化时的 `knowledge_references`
- 历史重开时回给前端的 `knowledge_references`

都走这一份 shape。

所以它虽然底层复用了 `SearchResult`，但在系统语义里其实代表：

- **回答引用清单的统一载体**

这能避免很多层都各自定义一份“简化引用结构”，结果把 `source_locators`、`citation_sources`、`image_info` 这些后续原文定位还要用的数据抹掉。

## 4. 为什么 assistant message 也要把这份引用清单持久化

[types/message.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/message.go) 里，`Message.KnowledgeReferences` 不是临时流式字段，而是直接落表的消息字段。

这件事的重要性在于：

- live stream 结束后，原始 SSE 事件不再可靠可取
- 但用户重开历史会话时，引用抽屉仍然应该可用

所以消息层必须把这份引用清单保留下来。

换句话说：

- **引用清单不是 UI 装饰**
- **它是 assistant turn 的一部分历史事实**

这也是为什么前端很多地方优先看：

- `message.knowledge_references`

而不是再回放一次事件流。

## 5. `buildStreamResponse` 在这条链里扮演什么角色

[buildStreamResponse](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/helpers.go) 是“内部事件 -> 对外 SSE payload”的最后一层翻译。

这里对 `references` 事件做了一件很重要的事：

- 把内部 `data.references`
- 提升为对前端统一可读的顶层 `knowledge_references`

并且兼容：

- 直接的 typed struct
- Redis round-trip 后的 `[]interface{}`
- 历史回放里重新反序列化出来的 map

它的意义是：

- 前端只需要记住一个读取口径

也就是：

- `response.knowledge_references`
- 或消息上的 `message.knowledge_references`

而不必到处区分：

- 这是 live SSE
- 还是 replay 事件
- 还是历史消息行

## 6. 前端为什么会在没正文时先补一个 assistant 壳

[useChatStreamHandler.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useChatStreamHandler.ts) 里的 `applyKnowledgeReferences` 很关键。

它处理 `references` 事件时，如果当前还没有对应的 assistant 消息，会：

1. 先创建一条 assistant 壳
2. 把 `knowledge_references` 挂上去
3. 等正文 token 到来时继续复用同一条消息

这个设计是为了支持：

- RAG 时间线先出现
- 引用抽屉先可打开
- 正文稍后再流入

也就是说，这条 assistant 壳并不是“空消息”，而是：

- **这一轮回答的结构化挂载点**

## 7. 为什么 `knowledge_references` 是直接覆盖，而不是 append

`applyKnowledgeReferences` 不是把新引用 append 到旧列表后面，而是直接：

- `message.knowledge_references = refs.slice()`

这是因为后端发出的 `references` 事件已经代表：

- 当前回答最终采用的 merged result list

如果前端再做 append，就会引入几个问题：

- 历史残留 chunk 没清掉
- 同一文档被重复累加
- live 重放时重复插入

所以这里的原则是：

- **引用清单是快照替换，不是增量 patch**

## 8. `docInfo` 为什么优先跳统一抽屉，而不是展开旧内联列表

[docInfo.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/views/chat/components/docInfo.vue) 里现在的主路径是：

- 有 `referencesDrawer` 时直接打开右侧抽屉

而不是继续主要依赖它自己这套旧的内联引用块。

原因很实际：

- 右侧抽屉已经统一支持文档 / 网页 / tool 引用
- 也支持原文定位、citation 高亮和来源切换

而 `docInfo` 自己本地只适合做一个简短摘要入口。

所以它现在更像是：

- 一个“引用摘要组件”

而不是完整引用查看器。

## 9. 为什么 `AgentStreamDisplay` 还要从 tool event 里兜底恢复引用

[AgentStreamDisplay.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/views/chat/components/AgentStreamDisplay.vue) 里有个很重要的兜底逻辑：

- 优先使用 `session.knowledge_references`
- 如果没有，再去 `agentEventStream` 里找 retrieval tool event 的 source data

这层兜底是为了覆盖几种特殊情况：

- live 时正文里 citation tag 已经到了，但聚合的 `knowledge_references` 事件还没到
- 某些历史会话缺少聚合引用字段
- 某些 agent answer 只保存了 tool event，但没保存完整引用清单

注意这里的顺序不能反：

- **message.knowledge_references 才是主口径**
- tool event 只是兜底恢复

否则前端会把中间检索态和最终回答态混在一起。

## 10. 这条链和第 38/39/40 篇是什么关系

这一篇解决的是：

- 回答引用清单“怎么被装起来并传给前端”

而前几篇分别处理的是：

- 第 `39` 篇：locator 是怎么生成出来的
- 第 `40` 篇：locator 在编辑/重写后何时必须失效
- 第 `38` 篇：前端怎样拿 locator 去打开原文并做降级

所以它们的关系可以理解成：

1. `39` 生成原文定位能力
2. `40` 定义何时这份能力不能再信
3. `41` 把最终引用清单送进回答消息
4. `38` 在前端把它展示成抽屉和原文预览

## 11. 这条链最容易看错的地方

### 第一，`references` 事件不是“额外信息”，而是回答结构的一部分

它和正文 answer 不属于一类数据，但都属于同一轮 assistant turn 的可见产物。

### 第二，`knowledge_references` 不是纯 UI 字段

它会随消息持久化，服务的是：

- 历史恢复
- citation 点击
- 原文定位
- 文档 / 网页 / tool 引用抽屉

### 第三，tool event 引用不能反过来取代聚合引用清单

tool event 是兜底，不是主口径。

真正对回答负责的引用清单，仍然应该来自：

- merge/filter 之后的 `MergeResult`

## 12. 推荐连读方式

如果你要把“问答引用从检索结果变成前端抽屉”完整读顺，建议这样连：

1. 先看 [41-code-reading-chat-reference-list-pipeline.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/41-code-reading-chat-reference-list-pipeline.md)，确认清单怎么组装和传递
2. 再看 [39-code-reading-source-locator-generation-remap.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/39-code-reading-source-locator-generation-remap.md)，确认原文坐标从哪里来
3. 再看 [40-code-reading-locator-stale-edit-merge-boundaries.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/40-code-reading-locator-stale-edit-merge-boundaries.md)，确认哪些坐标不能再信
4. 最后看 [38-code-reading-chat-reference-source-location.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/38-code-reading-chat-reference-source-location.md)，确认前端最终如何打开原文

这样就能把“引用清单”和“原文定位”这两层明确拆开，再把它们重新接起来。
