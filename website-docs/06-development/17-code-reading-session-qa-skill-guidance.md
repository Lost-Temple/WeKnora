# 会话 QA 入口、配置型技能引导与安装时间线阅读指南

这篇文档把两条容易分开读散的链路放到一起：

1. 后端 [qa.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go) 里“发送一轮消息”到底经历了什么
2. 前端 [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue) 为什么看起来像聊天窗口，其实是在复用会话流模型展示技能安装

如果你准备改聊天发送入口、技能安装引导、配置型 steer 或安装 transcript，这一篇就是新的主入口。

## 推荐阅读顺序

1. [qa.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go)
2. [useChatStreamHandler.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useChatStreamHandler.ts)
3. [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue)
4. [SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue)
5. [api/system/index.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/api/system/index.ts#L1209-L1262)
6. [15-code-reading-chat-session-steer-stop-resume.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/15-code-reading-chat-session-steer-stop-resume.md)
7. [16-code-reading-backend-session-stream.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/16-code-reading-backend-session-stream.md)

## 1. 先把 `qa.go` 看成“发送一轮消息的总装配函数”

后端会话入口不是 service 层，而是 [executeQA](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go) 这一段 handler 流程。

它负责的不是某一个动作，而是一整轮发送的装配顺序：

1. 校验当前 session 还能不能再开一轮
2. 落 user / assistant 两条消息壳
3. 建 live run、SSE、stream handler
4. 异步调用 `KnowledgeQA` 或 `AgentQA`
5. 在 defer 里收尾：持久化 assistant、交接 steer backlog、发布 complete

如果只看 service，很容易误以为模型事件一出来就直接推给前端。实际上这里先把“这轮回答属于哪条真实消息、当前会话有没有 live run、历史刷新增量该挂到哪条 assistant”全部搭好，再放 service 执行。

## 2. `buildQARequest` 在切什么边界

[buildQARequest](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L101-L131) 是 handler 和 service 的交界点。

这一步最重要的不是字段搬运，而是把 handler 已经确定好的运行边界传给 service：

- `AssistantMessageID` / `UserMessageID`：流事件最终要落到哪两条已持久化消息
- `TurnLeaseHeld`：发送路径已经拿过 sandbox turn lease，service 不要重复持有
- `SteerSink`：这轮若是 agent run，引擎需要在 round boundary drain 中途 steer

结论可以记成一句话：

**handler 先把“这一轮消息的容器和占位”准备好，service 只负责把内容填进去。**

## 3. `executeQA` 的顺序为什么这么严格

读 [executeQA](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L1246-L1598) 时，最关键的是顺序。

### 先拒绝不该发送的场景

一开始先做两种互斥检查：

- `rejectIfOtherAgentRunLive`
- `rejectIfSessionRewinding`

也就是：

- 同一个 session 已经有另一条 live agent turn，就不能再发
- session 正在 rewind，也不能发

这两类都必须在任何消息持久化前挡掉，否则会制造出“数据库里有新消息，但实际没有运行”的假历史。

### 再持久化 user / assistant 消息壳

[persistTurnMessages](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L1086-L1130) 的职责是先把这轮 user 和 assistant 行落出来。

这样做的意义有两个：

1. 页面刷新、`continue-stream`、历史重建时始终有真实 message id 可以挂
2. agent run 一开始就能把 `assistant_message_id` 发给前端，后续所有流事件都围绕同一条消息对齐

如果 `SetLiveRun` 等后续动作失败，[rollbackTurnMessages](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L1132-L1150) 会删掉刚创建的空壳，避免历史里留下孤儿 turn。

### Agent 模式先发 `agent_query`

`agent_query` 在 [executeQA](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L1283-L1296) 里先被 emit。

它的语义是：

- 告诉前端“这一轮 assistant turn 开始了”

前端收到后可以先进入运行态，再随着后续事件逐步对齐真实 `assistant_message_id`、thinking、tool call、answer 和 complete。

## 4. 为什么 steer carry-over 要先搬，再 publish live run

这一段在 [executeQA](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L1321-L1331)。

上一轮若还有没来得及消费的 steer backlog，需要先 append 到这一轮 assistant 的 steer 子列表，再让新 run 对外表现成 live。

不然会出现一个刷新窗口：

1. 客户端看见新 assistant 已经 live
2. 但 steer 队列暂时还是空的
3. 用户会误以为自己还看得到的 steer 已经丢了

所以这一步其实是在维持“live run 指针”和“队列真相”之间的一致性。

## 5. Agent 和 QuickAnswer 为什么共用同一套 complete 语义

表面看 `KnowledgeQA` 是快答，`AgentQA` 是完整 agent run，但收尾时两者都要落到同一套 complete 语义上。

### Agent 路径

Agent 路径在异步 goroutine 的 defer 里收尾，最终会走：

- [completeStreamAssistantMessage](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L1942-L1958)

### QuickAnswer 路径

QuickAnswer 在 [completeQuickAnswerTurn](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L1890-L1915) 里收尾。

这里会先补发 `EventAgentComplete`，让 [handleComplete](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/agent_stream_handler.go#L715-L931) 把：

- checkpoint
- artifacts
- final metadata

都挂到内存里的 `assistantMessage` 上，然后再持久化。

这一步很关键，因为 QuickAnswer 刷新历史时也必须看到和 AgentQA 一样的最终消息快照。

## 6. `completeStreamAssistantMessage` 在守什么时序

[completeStreamAssistantMessage](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/qa.go#L1942-L1958) 和上一轮补过的 [publishCompletion](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/session/agent_stream_handler.go#L933-L950) 是一组。

它守住的是这个顺序：

1. `completeAssistantMessage` 先更新数据库
2. 成功后再 `publishCompletion`

原因很简单：

- 客户端一旦收到 complete，就会把这条消息视为最终稳定状态
- artifacts / final_content / usage 都会开始依赖这条已持久化消息

若顺序反过来，就会出现“流里看起来完成了，刷新历史又回退”的错位。

## 7. 配置型技能引导为什么看起来像聊天

前端 [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue) 复用了聊天流处理器 [useChatStreamHandler](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useChatStreamHandler.ts)。

这不是巧合，而是因为“技能安装 transcript”本质上也是一条会话流：

- 有一条安装 prompt
- 有 assistant thinking / tool call / answer
- 有命令输出
- 有继续 follow live tail 的需求

所以这个组件不是另写一套安装日志渲染，而是把安装器 transcript 重新包装成一条迷你聊天时间线。

## 8. 为什么 `SkillInstallTimeline` 有 live 和 persisted 两套读取路径

这段逻辑在 [open](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L220-L274)。

### 已结束安装：优先读 durable rows

如果 `live=false`，组件先读：

- [getMessageList](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/api/chat/index.ts)

原因是“查看历史安装记录”不应该把旧 thinking/tool call 再动画回放一遍。

只有当 durable history 为空时，才退回 transcript replay 补一次内容。

### 正在安装：优先 follow transcript

如果 `live=true`，组件会走 [follow](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L162-L218) 订阅 transcript。

而且它不会在 `sessionId / messageId` 还没落好时就去打接口，因为安装初期 transcript locator 还不存在，过早 follow 只会持续得到 404。

## 9. `follow()` 为什么和 `continue-stream` 是同一模型

[configSkillTranscriptUrl](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/api/system/index.ts#L1209-L1216) 提供的是安装器自己的 transcript SSE。

[follow](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L162-L218) 读取它时，语义和聊天的 `continue-stream` 是一致的：

- 服务端先 replay 历史事件
- 再接 live tail
- 前端统一交给 `processStreamChunk` 重建消息状态机

这里返回 `served`，是为了区分：

- transcript 还在，可以继续靠事件流重建
- transcript 已过期，只能退回 durable 历史消息

## 10. 配置型技能 guidance 为什么本质上也是 steer

看 [sendGuidance](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L71-L107) 和 [steerConfigSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/api/system/index.ts#L1259-L1262)。

当安装仍在进行中时，guidance 输入框并不是“再开一轮安装”，而是把一条指导语 steer 给当前 installer run：

- `expected_message_id`：对准当前 live installer message
- `steer_id`：本地先生成一个稳定 id
- `content`：指导语正文

这和聊天里的 steer 是同一条规则：

- 必须对准当前真实 live assistant/message
- 不能只靠 UI 上当前看到的最后一行

## 11. 为什么 guidance 要维护 `pendingSend`

这一点在 [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L47-L91)。

场景是这样的：

1. 用户发了一条 guidance
2. POST 其实成功了
3. 但后面的 guidance 列表刷新失败了

如果此时 Retry 重新生成一个新的 `steer_id`，后端就会把它当成第二条新指导重复入队。

所以组件保留 `pendingSend`：

- messageId 一样
- content 一样
- steer_id 继续复用

并在本地先插一条 `pending` guidance 回执，防止“服务器已接收，但 UI 因轮询失败看起来像没发出去”。

## 12. 为什么历史安装完成后 guidance 会切成“重装并附带说明”

还是 [sendGuidance](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L71-L107)。

当 `live=false` 时，guidance 输入框的语义就变了：

- 不再 steer 给当前 run
- 而是调用 [reinstallConfigSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/api/system/index.ts#L1148-L1154)

也就是：

- live 安装中：这是“给当前安装 agent 的中途指导”
- 安装结束后：这是“带着新指导重新跑一轮安装”

[SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L259-L275) 通过 `live` 和 `canRetry` 两个 props 把这两种模式切开。

## 13. 这篇文档和前后文怎么接

建议按下面顺序连读：

1. 先看 [15-code-reading-chat-session-steer-stop-resume.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/15-code-reading-chat-session-steer-stop-resume.md)，理解“前端一轮回答的真实消息 id、steer 队列和 preview 行”
2. 再看 [16-code-reading-backend-session-stream.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/16-code-reading-backend-session-stream.md)，理解“后端 continue-stream / complete 的时序”
3. 再回到这一篇，把“发送入口如何搭好消息壳”和“技能安装如何复用同一套会话流模型”串起来

## 14. 二开时最容易踩的坑

### 坑一：以为 `qa.go` 只是简单转发到 service

实际上 live run、消息壳、rollback、steer carry-over、complete 时序都在这里装配。

### 坑二：AgentQA 和 QuickAnswer 收尾可以各走各的

不行。QuickAnswer 也必须补 `EventAgentComplete`，否则刷新历史会丢 checkpoint / artifacts 等最终状态。

### 坑三：技能安装 guidance 只是普通重试输入框

live 阶段它本质上是 steer，不是重新安装。

### 坑四：历史安装也应该继续 follow transcript

会把旧 thinking / tool call 全部重新动画回放，违背“查看一次安装记录”的阅读预期。
