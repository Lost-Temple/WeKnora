# 会话问答、引用与时间线阅读指南

这份指南解决的是另一类很常见的阅读痛点：

- 为什么知识问答接口一边流式输出答案，一边还能展示引用和时间线
- 为什么前端会有 citation 抽屉、引用聚合、等待行、断流后的活动恢复
- 为什么后端还要单独处理 `resource_urls=public`

如果你已经看过前面的检索与知识库阅读指南，这一篇就是“检索结果如何进入对话 UI”的下一站。

## 推荐阅读顺序

1. `internal/handler/session/qa.go`
2. `internal/application/service/session_knowledge_qa.go`
3. `internal/handler/session/resource_urls.go`
4. `internal/handler/session/quick_answer_timeline.go`
5. `frontend/src/utils/rag-pipeline-state.ts`
6. `frontend/src/utils/referenceSources.ts`
7. `frontend/src/utils/citationMarkdown.ts`
8. `frontend/src/stores/sessionActivityState.ts`

## 1. 先看会话入口

文件：

- `internal/handler/session/qa.go`

建议先看两个入口：

- `SearchKnowledge`
- `KnowledgeQA`

### 它们的区别

#### `SearchKnowledge`

这是“只要检索结果，不要 LLM 总结”的接口。

它会：

- 解析 KB / knowledge / tag scope
- 校验 API key / 租户可见性
- 调用统一检索服务
- 把结果直接返回给客户端

不会做：

- prompt 组装
- 流式回答
- SSE 事件
- 完整时间线持久化

#### `KnowledgeQA`

这是产品里的真正问答入口。它会继续往后走：

- 检索
- rerank
- 合并上下文
- 拼装 chat message
- 流式输出答案
- 同步发引用和时间线事件

## 2. 再看问答服务如何接检索

文件：

- `internal/application/service/session_knowledge_qa.go`

重点看：

- `KnowledgeQA`
- `runKnowledgeSearchPipeline`
- `applyKnowledgeSearchOverrides`

### 这一层最重要的理解

检索在问答里不是一个独立接口调用，而是整个 RAG pipeline 的中间阶段。

可以先把典型路径记成：

1. `QUERY_UNDERSTAND`
2. `CHUNK_SEARCH_PARALLEL`
3. `CHUNK_RERANK`
4. `CHUNK_MERGE`
5. `FILTER_TOP_K`
6. `INTO_CHAT_MESSAGE`
7. `CHAT_COMPLETION_STREAM`

也就是说：

- 底层检索只是“找到候选”
- 真正进入模型前，还会再经过筛选和上下文组织

## 3. 看资源链接为什么要单独改写

文件：

- `internal/handler/session/resource_urls.go`

### 这个文件在解决什么问题

模型答案、引用列表、工具结果里都可能带有内部存储引用。客户端有两种消费方式：

- `handle`：保留内部 handle，前端后续再按需解析
- `public`：后端直接改写成可加载直链

### 为什么 SSE 里要有 holdback buffer

因为流式输出时，一个 storage handle 可能被拆在两段 delta 里。

如果后端每收到一段就立刻改写：

- 前半段还不是完整 handle
- 结果既改不出来，也可能把文本截坏

所以这里会先把 delta 放进 buffer，等引用完整后再释放。

### 阅读时重点看

- `resolveResourceRewriter`
- `resolveStreamRewriter`
- `buildStreamResponseFor`
- `flushHeldStreamContent`

## 4. 看时间线为什么能在刷新后还原

文件：

- `internal/handler/session/quick_answer_timeline.go`

### 背景

如果只靠 `knowledge_references` 重建时间线，会漏掉很多“搜过但没命中”的情况。

比如：

- 搜索执行了，但没有引用结果
- 附件解析做了，但不是知识引用
- 图片分析做了，但最终没产出 citation

所以 quick-answer 会把这些工具事件单独持久化成 timeline step。

### 这里最值得记住

fast-answer 没有 agent 的多轮 ReAct 结构，所以它把这些工具调用都收敛进一个固定 step 里，给前端回放。

## 5. 再看前端等待行状态

文件：

- `frontend/src/utils/rag-pipeline-state.ts`

### 它在解决什么问题

用户能看到的 RAG 时间线，往往会有一个“检索都做完了，但模型还没开始吐字”的静默窗口。

这个工具就是专门描述那段窗口的：

- 是否显示等待行
- 先显示 `preparing` 还是 `model`
- 多久之后判定为 stalled

### 为什么重要

它直接决定了“这个问答是在正常等待，还是流已经断了但 UI 还在假装有进度”。

## 6. 看前端引用为什么要按文档聚合

文件：

- `frontend/src/utils/referenceSources.ts`

重点看：

- `mergeDocumentReferences`
- `buildDocumentItem`

### 这里在解决什么问题

后端返回的引用很多时候是 chunk 级别的，但用户在引用抽屉里更关心：

- 这是哪篇文档
- 摘要片段是什么
- 点开后跳到原文哪里

所以前端会把同一文档的多个 chunk 引用聚成一个文档项，同时保留：

- `chunk_ids`
- 第一个可定位的 `source_chunk_id`

这样展示和跳转都能兼顾。

## 7. 看引用标签为什么要先预处理

文件：

- `frontend/src/utils/citationMarkdown.ts`

### 这一层主要处理什么

- `<kb/>` / `<web/>` 标签
- `[[wiki-link]]`
- 不完整 citation tag
- citation HTML placeholder

### 最容易忽略的点

模型是流式吐字的，所以 citation tag 可能只吐出一半。

如果前端直接把半截标签交给 markdown：

- 会在页面上看到裸 `<`
- 还可能把后续结构解析乱

所以这里会先把不完整 citation tag 隐去，等完整后再进入正常渲染流程。

## 8. 看断流后的会话活动恢复

文件：

- `frontend/src/stores/sessionActivityState.ts`

### 它在解决什么问题

有些场景里，页面上的 SSE 流断了，但服务端那条 assistant message 还在继续生成。

前端为了不丢状态，会维护一个 detached activity：

- 记住 sessionId
- 尽量锁定原 messageId
- 定期轮询 messages
- 判断这条消息是否已经完成

### 为什么优先按 messageId 找

因为同一个 session 里可能同时存在别的 assistant message，直接找“第一个未完成 assistant”很容易串线。

## 一个实用阅读路径

### 路径一：改“引用抽屉 / citation 展示”

1. `referenceSources.ts`
2. `citationMarkdown.ts`
3. 相关 citation popover / markdown 组件
4. `resource_urls.go`

### 路径二：改“问答时间线”

1. `session_knowledge_qa.go`
2. `quick_answer_timeline.go`
3. `rag-pipeline-state.ts`
4. 时间线组件

### 路径三：改“知识搜索接口和会话问答之间的关系”

1. `session/qa.go`
2. `session_knowledge_qa.go`
3. 前面的检索阅读指南

## 当前最值得记住的几个边界

- `SearchKnowledge` 是检索接口，`KnowledgeQA` 是完整问答接口。
- 引用结果能展示，不代表一定有答案；答案能生成，也不代表一定有 citation。
- SSE 流里的资源链接改写要考虑“引用被拆成多段 delta”的情况。
- 快速问答时间线不能只靠 references 重建。
- 前端引用抽屉是“按文档/网页/工具”聚合展示，不是原样照抄后端 chunk 列表。
