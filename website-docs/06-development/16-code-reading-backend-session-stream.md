# 后端 Session Stream、Continue-Stream 与 Complete 阅读指南

这篇文档专门补聊天后端这一段：

1. `continue-stream` 为什么是“整段 replay + live tail”
2. 为什么 replay 可以 coalesce，但 live tail 不行
3. `answer.done` 为什么不等于一条消息真正完成
4. `handleComplete` 和 `publishCompletion` 为什么要拆成两段

如果你准备改聊天流式输出、续播恢复、artifact 收尾、SSE 协议，这一篇就是后端主入口。

## 推荐阅读顺序

1. `internal/handler/session/stream.go`
2. `internal/handler/session/continue_stream_coalesce.go`
3. `internal/handler/session/resource_urls.go`
4. `internal/handler/session/helpers.go`
5. `internal/handler/session/agent_stream_handler.go`
6. `frontend/src/composables/useChatStreamHandler.ts`
7. `website-docs/06-development/15-code-reading-chat-session-steer-stop-resume.md`

## 1. 先记住两层数据形态

后端这一段至少有两层数据形态：

### `interfaces.StreamEvent`

这是内部事件日志形态。

它被：

- event bus 产生
- stream manager 持久化
- continue-stream 读出来 replay / 轮询

它更像“服务端内部事件记录”，保留了：

- `Type`
- `Content`
- `Done`
- `Data`
- `Usage`

### `types.StreamResponse`

这是真正发给前端的 SSE payload。

只有到 [buildStreamResponse](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/helpers.go#L190-L238) 这一层，后端才把内部事件翻译成前端协议，并顺手把：

- `references`
- `agent_query` 的 `session_id / assistant_message_id`
- `usage`

这些前端常读字段提升出来。

理解这一点很重要：**continue-stream replay 的是 StreamEvent，但浏览器真正消费的是 StreamResponse。**

## 2. `continue-stream` 的真实语义不是“从断点继续”

核心入口在 [ContinueStream](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/stream.go#L35-L185)。

### 它先做什么

在写任何 SSE header 之前，它会先做几件验证：

1. 校验 `session_id`
2. 校验 `message_id`
3. 解析 `resource_urls` 模式
4. 确认 session 属于当前用户
5. 确认 message 真存在

这里故意把 `resolveStreamRewriter` 放在最前面，是为了保证：

- 参数非法时还能返回正常 JSON 400/403
- 不会因为 SSE header 已写出而把一个普通参数错误变成半截流响应

### 然后它真正做的事情

`continue-stream` 的主逻辑不是“从客户端给的 offset 接着播”，而是：

1. 从 offset 0 读回这条消息的整条 event log
2. 先 replay 一遍历史
3. 再进入 ticker poll，持续拉新事件

这就是为什么页面刷新后还能重建同一条回答的状态机：

- thinking
- answer
- references
- tool call / tool result
- artifacts pending
- complete / stop

前端不是靠“记住上次看到哪一帧”恢复，而是靠“重新看完整事件历史，再接后面的新事件”恢复。

## 3. 为什么 replay 要 coalesce，live tail 却不能

这一层在 [coalesceReplayEvents](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/continue_stream_coalesce.go#L9-L58)。

### replay 阶段的问题

模型原始输出粒度很细，一条很长的回答可能拆成成千上万个 delta。

如果 continue-stream 原样 replay：

- 后端要写大量 SSE 帧
- 浏览器要处理大量微小更新
- 用户体感像“整条历史重新慢慢打一遍”

### coalesce 的边界非常保守

它只合并：

- 同一个 `event_id`
- 同一种 `response_type`
- 且 `Done == false`
- 并且必须是连续片段

因此这些边界仍然会保留原样：

- `references`
- `complete`
- `stop`
- `duration`
- 其它类型切换点

所以它优化的是 replay 体积，不改事件边界语义。

### 为什么 live tail 不 coalesce

进入 poll loop 后，后端故意按原始事件粒度推：

- 首次实时流是什么顺序，续播时 live tail 也是什么顺序
- 前端状态机不需要在 replay 和 live 两段切换不同语义

一句话概括：

- replay：可以更粗，只要不破坏边界
- live：必须更真，尽量和首次实时流一致

## 4. `resource_urls` 为什么需要专门的 stream rewriter

这一层在 [resource_urls.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/resource_urls.go#L15-L159)。

### 普通响应和流式响应的不同

普通 HTTP 响应可以整段内容一次性改写：

- `resource://handle`
- storage handle
- public URL

但 SSE delta 流不一样，一个引用很可能被拆成两帧：

- 第一帧只有前半截
- 第二帧才补齐后半截

如果每帧都立刻改写，就可能把半个引用当普通文本放出去。

### `StreamRewriter` 的职责

[resolveStreamRewriter](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/resource_urls.go#L36-L49) 返回的不是普通 rewriter，而是带 holdback buffer 的版本。

对 `answer / thinking / reflection` 这类 delta 事件：

- 先 push 进 holdback
- 等引用完整后再输出

对非 delta 事件：

- 直接整段改写

### 为什么 terminal 事件前要 `flushHeldStreamContent`

[emitStreamEvent](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/resource_urls.go#L106-L124) 在 `complete / error` 前会先 flush holdback。

原因很直接：

- terminal 事件一到，客户端就把这轮流视为结束
- 若最后半个引用还在 holdback 里没吐出来，就永远丢了

所以这一步是在保证“流结束时，最后尾巴不会 silently lost”。

## 5. `answer.done` 为什么不等于 turn 已完成

这一点最容易在前后端都读混。

### `handleFinalAnswer` 在做什么

入口在 [handleFinalAnswer](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/agent_stream_handler.go#L527-L609)。

它负责：

- 累积最终答案文本
- 记录每个 `event_id` 的段内容
- 在 stream log 里追加 `answer` delta
- 在必要时记录 `duration / truncated / is_fallback`

但它**不负责宣布这条 assistant message 真正完成**。

### 为什么不能在这里就算 complete

因为 answer token 流停止后，后面通常还要继续做：

- checkpoint `/workspace`
- collect artifact
- merge artifact ownership
- rewrite artifact references
- 挂 usage / steps / final_content
- 准备 complete 事件

所以 `answer.done` 只代表：

- “这一段 token 结束了”

而不是：

- “这条持久 assistant message 已完整落库”

## 6. `handleComplete` 为什么是真正的收尾总线

入口在 [handleComplete](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/agent_stream_handler.go#L715-L931)。

可以把它理解成“turn 最终快照组装器”。

### 它收口的东西

在这一层，后端把同一条 `assistantMessage` 的最终状态一次性补全：

- `IsCompleted`
- `AgentDurationMs`
- `KnowledgeReferences`
- `Content += FinalAnswer`
- `AgentSteps`
- `Usage`
- `SandboxCheckpoint`
- `Artifacts`
- `rewriteArtifactReferences`
- `ClarifyArtifactVersions`

这里真正组出来的，是“准备持久化进数据库”的最终消息快照。

### artifact 为什么也必须在这里定型

因为正文里的文件引用最后要依赖：

- 最终 attached artifact 列表
- 最终索引顺序

只有 artifact 列表真正定型后，`sandbox:name` / `./output/file.html` 才能被安全改写成稳定引用。

### `previous` / `historyOnlyArtifacts` 在解决什么

一条新回答可能引用的是历史文件，而不是这轮重新生成的文件。

这时即便 artifact 不是“本轮新产物”，也必须重新绑定到当前消息上，否则旧消息删除后，这条新回答正文里的文件引用会悬空。

## 7. 为什么 `handleComplete` 和 `publishCompletion` 必须拆开

这一点是整个后端流处理里最关键的时序约束。

### `handleComplete` 只准备 `completionEvent`

在 [handleComplete](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/agent_stream_handler.go#L894-L931) 结尾，它只是把最终的 complete 事件放进：

- `h.completionEvent`

并没有立刻 append 到 stream。

### 真正发 complete 的是 `publishCompletion`

入口在 [publishCompletion](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/agent_stream_handler.go#L933-L950)。

它只在：

- `UpdateMessage` 成功之后

才把 complete 事件真正写进 stream log。

### 为什么一定要这样

因为前端一旦收到 `complete`，就会立刻认为这条消息已经是“最终稳定状态”：

- `final_content` 可以读
- artifact 按钮可以点
- usage 可以显示

如果 complete 先发了，但数据库更新随后失败，就会出现：

- 流里看着已完成
- 一刷新历史又回不到同样状态

所以这里的真实时序是：

1. `handleFinalAnswer` 处理正文流
2. `handleComplete` 组装最终消息快照
3. 消息持久化成功
4. `publishCompletion` 再把 complete 发给客户端

## 8. 前后端怎么接上

这篇后端文档最好和前端那篇一起看：

- [聊天会话状态、Steer、Stop 与 Resume 阅读指南](15-code-reading-chat-session-steer-stop-resume.md)

对应关系可以这样记：

### 后端 `agent_query`

对应前端：

- 创建或对齐当前 assistant 壳
- 设置 `currentAssistantMessageId`

### 后端 replay + poll

对应前端：

- `continue-stream` 重建消息状态机
- replay 时 reset steer 分段
- live tail 继续增量推进

### 后端 `stop`

对应前端：

- `markInFlightAssistantStopped`
- `onGenerationStopped`

### 后端 `complete`

对应前端：

- 清 loading / isReplying
- 落 artifacts / usage / final_content
- 触发 `onTurnComplete`

## 9. 二开时最容易踩的坑

### 坑一：把 replay 当成“只补最后几帧”

这样会误解很多前端 reset / rebuild 行为，尤其是 steer 分段和 references 恢复。

### 坑二：在 live tail 里也做 coalesce

会破坏首次实时流和续播流的一致性，边界事件顺序也更难保证。

### 坑三：answer.done 就直接当 turn 完成

artifact、usage、final_content、checkpoint 都可能还没准备好。

### 坑四：在消息持久化前就发 complete

这会把“流完成”和“数据库已稳定可读”混成一件事，刷新后最容易出现状态回退。
