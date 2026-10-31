# Agent 会话、Artifact 与续播阅读指南

这份指南接在“会话问答、引用与时间线阅读指南”后面，专门解释三类很容易混在一起的问题：

1. Agent 回答里引用的文件为什么能在刷新后继续打开
2. `continue-stream` 为什么不是简单“从断点往后读”
3. 为什么答案文字已经结束了，文件按钮还要再等一下才可下载

如果你后面准备改 Agent 会话、技能产物、续播体验，这一篇就是最直接的入口。

## 推荐阅读顺序

1. `internal/handler/session/agent_stream_handler.go`
2. `internal/handler/session/artifact_reference.go`
3. `internal/handler/session/stream.go`
4. `internal/handler/session/continue_stream_coalesce.go`
5. `frontend/src/composables/useChatStreamHandler.ts`
6. `frontend/src/utils/sessionArtifacts.ts`
7. `frontend/src/utils/skillArtifacts.ts`
8. `frontend/src/views/chat/components/ChatArtifactsDrawer.vue`

## 1. 先看 Agent 完成时到底会落什么

文件：

- `internal/handler/session/agent_stream_handler.go`

重点看：

- `handleReferences`
- `handleFinalAnswer`
- `handleComplete`

### `handleComplete` 在做什么

它不是单纯发一个 `complete` 事件，而是在 Agent turn 真正收尾：

- 标记 assistant message 完成
- 挂最终 usage / references / agent steps
- checkpoint `/workspace`
- 收集 skill 产物
- 把正文里的 artifact 引用改写成稳定引用
- 最后才把完整消息持久化完成

### 为什么答案结束了，artifact 还没马上出现

因为 `answer.done` 只代表“模型不再继续吐字”。

后面服务端还要继续做：

- checkpoint
- 扫描输出目录
- 上传文件
- 绑定 artifact ownership
- 更新消息里的 artifacts JSON

所以前端会单独显示一个 `artifactsCollecting` 状态。

## 2. 再看 artifact 引用为什么能跨刷新存活

文件：

- `internal/handler/session/artifact_reference.go`

重点看：

- `rewriteArtifactReferences`
- `artifactReference`
- `mergeArtifactLists`
- `historyOnlyArtifacts`

### 这层在解决什么问题

模型在 sandbox 里写答案时，常常引用的是：

- `report.html`
- `sandbox:report.html`
- `./output/report.html`

这些名字一旦离开当前 sandbox，就没有稳定意义了。

所以在消息最终落库前，后端会把它们改写成稳定引用：

- 有资源目录时：`resource://<handle>`
- 没资源目录时：`sandbox:<name>`

### 为什么 `mergeArtifactLists` 的顺序很重要

因为正文里的引用最终会绑定到 `message.Artifacts` 里的某个索引上。

也就是说：

- 同名文件谁排在前面
- 新生成版本是否覆盖旧版本
- fork 后引用旧文件时该指向哪一个

都取决于合并顺序。

### `historyOnlyArtifacts` 在干什么

有些回答会重新引用历史 turn 里已经生成过的文件，但这次并没有重新生成。

这种文件虽然不是“本轮新产物”，仍然要重新绑定到当前消息上。否则删掉旧消息时，会把新回答还在引用的 blob 一起删掉。

## 3. 看 continue-stream 为什么是“重放 + 续播”

文件：

- `internal/handler/session/stream.go`
- `internal/handler/session/continue_stream_coalesce.go`

### 先记住结论

`continue-stream` 不是从某个 offset 直接接着往后播给客户端。

它会先：

1. 从 0 取回整条 event log
2. replay 给客户端
3. 然后才进入 live polling，继续推后面的新事件

### 为什么要这么做

因为页面刷新后，前端内存里的 stream 状态已经没了。

要恢复这些状态，必须重新给客户端一遍完整事件序列，让它重建：

- thinking / answer segment
- references
- tool call / tool result
- artifacts pending
- complete

### 为什么 replay 要先 coalesce

历史事件是按模型原始粒度存的，一长段答案可能拆成成千上万个小 delta。

如果刷新后把它们原样 replay：

- 服务端会写出大量 SSE 帧
- 前端要重复处理极多微小更新
- 用户体感会像“整条历史又卡顿地重新生成一遍”

所以 replay 阶段会把同一个 `event_id`、同一种 delta 类型、且尚未 `done` 的连续片段合并成一个大帧。

### 为什么 live tail 不再 coalesce

因为 live 阶段要尽量保持和首次实时流一致的语义：

- `references`、`complete` 这类边界事件仍要原样送达
- 实时增量更适合按原始事件推进前端状态机

所以这里只压缩历史 replay，不压缩后面的 live tail。

再配合 [会话 Fork、Rewind 与分叉状态阅读指南](14-code-reading-session-fork-rewind.md) 一起看，会更容易理解为什么 rewind 删除消息时还要同步 drop replay stream：

- `continue-stream` 是按 `session_id + message_id` 去重放 event log
- 如果消息历史已经被 rewind 删除，但 replay stream 还留着，前端就可能重新接上一条本不该存在的回答

## 4. 看前端怎么承接 artifactsCollecting

文件：

- `frontend/src/composables/useChatStreamHandler.ts`
- `frontend/src/utils/steerStreamFork.ts`
- `frontend/src/utils/skillArtifacts.ts`

### 关键事件

- `artifacts_pending`
- `complete`

### `artifacts_pending` 的作用

当答案正文已经结束、但产物还在上传时，前端会把消息标成：

- `artifactsCollecting = true`

这样工具栏不会出现“答案写完了但按钮空空如也”的错觉。

### `complete` 为什么还带 artifacts

因为前端不能等下次整段历史 refetch 才知道这条消息有哪些文件。

所以后端会把最终 artifacts 一起带在 `complete` 里，前端当场写回：

- `message.artifacts`

这样下载按钮和抽屉能立刻出现。

### `isCollectingSkillArtifacts` 为什么要看 `execute_skill_script`

有时 `artifacts_pending` 事件还没到，但这一轮已经明显执行过会产文件的技能。

这时前端会先基于 `agentEventStream` 给出一个“应该还在收集”的兜底判断，减少状态抖动。

### 为什么 steer 后一条回答会显示成多段 assistant

这不是后端真的落了多条 assistant message，而是前端为了把：

- 原回答前半段
- 中途插入的用户消息
- 回答后半段

按时间顺序展示出来，临时把同一条持久 assistant message 拆成多个 segment。

这里最关键的约束是：

- 后续 segment 可以有 synthetic `id` 作为 Vue 列表 key
- 但 `assistant_message_id` 必须继续指向同一条真实消息

否则 `continue-stream`、停止生成、artifact 下载这些依赖真实 message id 的能力都会串线。

### 停止生成为什么也不能直接拿行 id

`frontend/src/views/chat/index.vue` 里的停止生成流程，最终会调用：

- `stopStream()` 断 SSE
- `markInFlightAssistantStopped(currentAssistantMessageId.value)` 标记当前 turn 已停止

这里的 `currentAssistantMessageId`，在页面刷新后 resume 时，是通过 `persistedAssistantId(lastMessage)` 重新取出来的。

为什么不能直接用 `lastMessage.id`：

1. 如果此刻正在展示的是 steer 拆开后的后半段 assistant，`lastMessage.id` 只是一个 synthetic key
2. 用这个 id 去做 `markInFlightAssistantStopped`，页面状态机里不会命中真实正在运行的 turn
3. 视觉上就会出现：停止按钮已经按了，但页面还以为回答在继续运行，继续显示 loading 或续播入口

因此“停止生成”和“continue-stream”一样，寻址对象都必须是后端那条持久 assistant message。

### turn complete / steer follow-up 为什么也要走 persistedAssistantId

`useChatStreamHandler` 的 `onTurnComplete` 回调会触发：

- `loadFollowUpSuggestions(message, true)`
- `flushSteerAfterTurn(persistedAssistantId(message))`

`flushSteerAfterTurn` 要回答的问题是：“刚结束的到底是哪条 assistant turn？它后面可以接着发哪批 steer？”

如果这里直接用 `message.id`，会碰到 steer 拆分场景：

- 回调 `message` 可能只是后半段 assistant segment
- 它的 `id` 是 synthetic
- 后面 `attachSteerFollowUp` 里拿“新 assistant !== 已完成 id”做比对时，就会把旧 turn 当成还没完成，或者把当前 turn 当成新 turn

所以 steer 队列从“排队”到“挂到某条运行中的 assistant turn”，整条链路的 `completedAssistantId / expected_assistant_message_id` 都必须是后端那条持久消息 id。相应地，`hydrateSteerQueue` 从服务端拉回来的 `assistant_message_id`，也是同一个口径。

### steer 入队失败时为什么要特别处理 409

`handleSteerMsg` 失败时会做两件 UI 相关的状态设置：

1. 把 `messagesList` 里那条“pending 预览行”标 `_steerFailed`，让用户一眼看到是哪条 steer 失败
2. 如果错误是 `409`，把该 steer 的 `expected_assistant_message_id` 改成 `currentAssistantMessageId`

这里 `409` 的语义一般是：“用户以为这条 steer 要挂到的那条 assistant turn 还在，但服务端视角里它已经结束/不存在/不能再追加了”。

如果不把 `expected_assistant_message_id` 修正到页面当前正在跑的那条，用户点“重试”时会：

- 继续把请求打到旧的 expected id
- 服务端继续 409
- 用户看到“永远重试不成功”的错觉

因此 409 的处理本质是一次“本地 turn 对齐”：把 steer 的预期运行时，从已经失效的旧 turn，重新对齐到页面当前真实在跑的新 turn。

### flushSteerAfterTurn 的两条路径

`flushSteerAfterTurn(completedAssistantId)` 里有两条互斥路径，分别对应两种不同的 steer 来源：

#### 路径 1：`awaitingIdleSend` 的 promote 升级项

如果用户在“回答正在生成”时把某条 after steer 手动 promote 成 inject，服务端经常会返回 `new_run` 而不是立刻插入当前回答：

- 这表示这条 steer 已经不是“插入当前 turn”了，而是“开启一次全新的独立问答”
- 前端不能再对它走 inject 语义，只能等“当前 turn 结束”再真正发射
- 所以会先打上 `awaitingIdleSend = true`

等到 `flushSteerAfterTurn` 触发时：

- 第一条 `awaitingIdleSend` 直接当普通 `sendMsg`
- 后面其余的仍然走 `handleSteerMsg(... after)`

这样保证“新一轮问答”从第一个 user 问题开始，后面的 steer 仍然保持“排队在某条 assistant 后面”的顺序感。

#### 路径 2：走 `attachSteerFollowUp` 的排队 follow-up

没有 `awaitingIdleSend` 时，就进入真正把 steer “接上新 run”的流程：`attachSteerFollowUp(completedAssistantId)`。

### attachSteerFollowUp 在做什么

`attachSteerFollowUp` 本质是在回答三个问题：

1. **服务端把 steer 接上新 run 了吗？**
   - 不是同步完成的，所以前端要做最多 `40 次 × 200ms` 的轮询
   - 判断依据是：消息列表里出现一条“还没完成、且 id 不等于 `completedAssistantId`”的 assistant

2. **新 run 里的 user 问题，应该认领前端本地队列里的哪条 steer？**
   - 不能只看消息文本，否则同一问句连排两次会错位
   - 正确做法是：先按 `request_id` 把“这个新 assistant 对应的 user 行”挑出来，再按 content 做一一匹配，并用 `claimed: Set<steer_id>` 去重
   - 如果有 pending 预览行，就原地把预览行改造为真实 user 行，避免气泡跳动

3. **认领完成后，怎么把 SSE 画面跟上？**
   - 直接走 `/api/v1/sessions/continue-stream`
   - 参数里的 `query: newAssistant.id` 就是“从哪条消息继续流”，和 rewind / fork 后的续播共用同一条链路
   - 认领成功后还要 `hydrateSteerQueue({ onlyWhenLive: true })`，把服务端真实 backlog 同步回前端，避免本地队列和服务端口径分叉

### 为什么 finally 里要尾递归再 attach 一次

`attachSteerFollowUp` 内部用 `await startStream(...)`，它会一直等到整段 SSE 结束才 resolve。

这会导致一个“锁把后续回调吃掉”的问题：

- 中间 onTurnComplete 被触发时，`attachingSteerFollowUp` 还没释放
- onTurnComplete 会因为这个 guard 直接 no-op
- steer 队列里如果还有剩余 after-items，就不会被驱动

因此 finally 里有一段尾递归：

```ts
if (steerQueue.value.some(item => !item.failed)) {
  void attachSteerFollowUp(attachedAssistantId);
}
```

语义是：**“我刚把一段 SSE 完整跑完了，如果队列里还有没失败的 steer，就再自动驱动下一轮 attach，直到排完为止。”**

这也是为什么 `attachingSteerFollowUp` 这把锁很关键：没有它，尾递归 + onTurnComplete 会同时跑多个 attach，造成同一条 steer 被认领多次、同一条 continue-stream 被重复打开。

### promote 为什么要先乐观修改，失败了再回滚

`handlePromoteSteer` 做的是“把一条 after steer 升级成 inject，插到当前回答中间”。

它的状态机是：

1. **乐观更新（请求前）**
   - `item.promoting = true`
   - `item.delivery = 'inject'`
   - 立刻调 `previewSteerMessage` 在 transcript 里插一条 pending 预览行
   - 目的：给用户即时反馈，不要等服务端回包才动 UI

2. **服务端回来后的三个成功分支**
   - `already_injected`：服务端说“它已经进当前 turn 了”，前端直接 drop 队列项和 pending 标记，避免后续重复认领
   - `new_run`：服务端说“插不进当前回答，只能作为新一轮问答”，再分两种：
     - 当前还在生成：打 `awaitingIdleSend = true`，等 `flushSteerAfterTurn` 时真正发送
     - 当前已空闲：立刻 drop 预览和队列，直接 `sendMsg` 开启新一轮
   - 默认成功：后端确认了 inject，保持本地 delivery=inject

3. **失败回滚**
   - `item.delivery` 从 `inject` 退回 `after`
   - `discardSteerPreview` 把刚才乐观插入的假行删掉
   - 再提示“提升失败”

为什么一定要回滚：如果只报错不改本地状态，用户会看到“UI 上明明有一条 inject 预览行，但答案永远不会把这句话插进去”，形成长期视觉欺骗。

### remove 为什么分成 already_injected / gone / removed=false 三条路径

`handleRemoveSteer` 的核心原则是：**后端没删掉的，前端也不要自作主张删掉；后端已经不存在了的，前端不要继续留着显示。**

对应三种情况：

#### 情况 1：`status === 'already_injected'`

- 含义：这条 steer 已经被服务端并入当前 turn，用户实际上已经删不掉了
- 前端动作：提示“已经注入了”，把本地队列项清掉
- 原因：再保留本地行只会误导用户，仿佛“还有一条没消费的 steer”

#### 情况 2：`status === 'gone'`

- 含义：服务端已经没有这条 steer（可能被别的端消费、回收、或同步脚本删掉了）
- 前端动作：直接 `discardSteerPreview + dropSteerQueueItem`，不报错
- 原因：这是“后端已经找不到了，前端跟它保持一致就好”，不该让用户感觉删除失败

#### 情况 3：`res.removed === false` 且不是 failed steer

- 含义：后端明确表态“我没删”
- 前端动作：只报错，不删本地行/预览
- 原因：这里的语义等价于“删除被服务端拒绝了”，如果前端强行清空，就会出现“提示删失败，但列表里东西没了”的不一致
- 注意：只有 `failed` 类型的 steer 会继续走到后面的本地强制删除分支——因为它本来就是死项，用户希望把它清出视线

#### 情况 4：其它成功路径 / failed steer

- 走到 `discardSteerPreview + dropSteerQueueItem`
- 覆盖两种语义：
  1. 后端确认删成功了（没显式返回 gone/already_injected，但也没说 removed=false）
  2. 本地是 failed steer，用户想强制清理显示

最后 finally 里只在队列还存在该 steer 时才把 promoting 复位，也是同样的一致性原则：如果前面已经真正 drop 了，再访问 item 字段会把 promoting 复位到一个不存在的对象上，语义不正确。

### 重新进入会话页：怎么恢复“正在生成、可以 stop/continue”的状态

文件：
- `frontend/src/views/chat/index.vue`
- `frontend/src/composables/useChatStreamHandler.ts`

关键流程在 history reload 之后那一段：

1. **找“最后一条未完成的 assistant”，而不是简单取最后一行**
   - 使用 `findLastMessage(m => m.role === 'assistant' && !m.is_completed)`
   - 原因：steer inject 后，transcript 尾部可能是 user bubble，不一定是 assistant；如果只取 last row，会漏掉正在跑的 turn
   - 这样即使“最新一行是用户消息”，也能继续接上那条没完成的 assistant

2. **恢复时走 `persistedAssistantId(lastMessage)`**
   - 因为这条 assistant 在 UI 上可能已经被拆成多段（前半段 + 用户注入 + 后半段）
   - 直接拿 `lastMessage.id` 可能命中某个 synthetic 分段 id
   - 真正能让 `continue-stream / stop / artifacts` 寻址到后端消息的，只能是 persisted id

3. **IM 回复 vs Web/API 回复的两种恢复策略**
   - `lastMessage.channel === 'im'`：打 `isAttachingImStream = true`，后续失败会走“安静轮询恢复”，不把 404 当真实错误
   - 其它渠道：保持原始报错，真实失败就真实露出
   - 原因：IM 渠道的回答不会走本服务 SSE，直接 `continue-stream` 会 404，但答案实际上还在来，所以需要一个单独的安静 recover 路径

4. **stop 成功 / stop 失败的状态设置故意相反**
   - stop 成功：清 loading/isReplying，清 recover 定时器，然后 `markInFlightAssistantStopped(currentAssistantMessageId)`
   - stop 失败（`handleStopFailed`）：把 `isReplying / loading` 重新设为 true

为什么 stop 失败反而要“继续认为在生成”：
- stop 接口失败只说明“没停下来”，不代表服务端停止生成
- 如果这里把 loading/isReplying 清掉，用户会看到“按钮点了，页面说没在生成，但答案还在继续往外蹦”的视觉错位

### steer 的两个真源：steerQueue 与 messagesList preview 行要分开读

steer 在页面上实际会存两份东西，角色完全不同：

1. **真源一：`steerQueue`（队列真源）**
   - `dropSteerQueueItem(steerId)` / `findSteerQueueItem(steerId)` 只操作这份
   - 这里真的删了，后续 `handleSteerMsg / flushSteerAfterTurn / attachSteerFollowUp` 就不会再把它当成待处理项
   - `expected_assistant_message_id / pending / failed / awaitingIdleSend / promoting` 这些生命周期字段，全部只存在于 steerQueue item 上

2. **真源二：`messagesList` 里的 preview 行（UI 真源）**
   - 由 `previewSteerMessage` 插入，由 `discardSteerPreview` 删除
   - 特点是带 `isSteer: true` + `_steerPending: true`，role 先占成 user
   - 它不是真实消息，只是让用户立刻看到“我点了 inject，这句话会插到这里”
   - 真正被消费后，要么原地被改造成 persisted user 行，要么被丢弃，永远不会作为真实消息继续参与后端寻址

因此阅读代码时要特别小心：
- 判断“这条 steer 还在不在”要用 `findSteerQueueItem`，不能搜 transcript preview
- 删除 steer 时，经常需要 `dropSteerQueueItem + discardSteerPreview` 配套调用
- 只调一个不调另一个，就会出现“队列里没了但页面还有假行”或“队列里还在但 UI 看不见”的不一致

### steerId 为什么有 clientId / serverId 两套口径

文件：
- `frontend/src/utils/steerId.ts` → `makeSteerClientId`
- `frontend/src/utils/steerStreamFork.ts` → `reconcileSteerMessageId`

设计思路是：
- 前端排队时先生成一个 UUID 形状的 `clientId`（`makeSteerClientId`）
- 调 `steerSession` 时把它作为客户端标识传出去
- 后端可能直接 echo 回来，也可能老版本后端生成自己的 `serverId`
- 一旦 HTTP 回包回来，前端要调用 `reconcileSteerMessageId` 把 transcript / steerQueue 口径统一到 serverId

为什么这样设计：
- 本地先有 id，才能立刻在 UI 上插 preview、在队列里加 item，不必等后端往返
- 兼容老后端不用改前端逻辑，一次 reconcile 就把口径拉齐
- SSE 可能比 HTTP 响应更快落库，reconcile 里还负责“发现 receipt 已经插了真实 user 行时，保留真实行并迁走 preview 的 mentioned_items，再删 preview”，避免气泡重复

为什么 `makeSteerClientId` 不直接用 `crypto.randomUUID`：
- 纯 HTTP 部署（非 Secure Context）下，某些浏览器里 `randomUUID` 不可用
- `crypto.getRandomValues` 仍然可用，所以手动拼一个 UUID v4 形状的字符串
- 最终 id 只是 steer 的排队标识，不涉及数据库主键安全约束

### 为什么 continue-stream 前要先 reset steer 分段

`continue-stream` 是从 event log 头部整段重放，不是只补最后几帧。

如果本地已经把这轮回答拆成多段 assistant，又直接把 replay 追加上去，就会出现：

- 老分段还在
- 新 replay 又再切一次段
- 内容重复
- 某些 segment 被旧的 `is_completed` 提前封口

所以前端会先把这轮 turn 的 segment 清空回“待 replay 重建”的状态，再让事件日志重新把它切开。

## 5. 看会话级 artifact 面板为什么还保留 messageId

文件：

- `frontend/src/utils/sessionArtifacts.ts`
- `frontend/src/views/chat/components/ChatArtifactsDrawer.vue`

### 关键点

会话级 artifact 面板虽然把所有 assistant message 的 artifacts 摊平展示了，但下载地址并不是“会话里的全局序号”，而是：

- `sessionId`
- `messageId`
- `artifact index`

所以前端聚合后仍然必须保留每个 artifact 的 owning `messageId`。

### `messageId` 为什么也要走 persistedAssistantId

在 `frontend/src/utils/sessionArtifacts.ts` 里，每个 artifact 的 owning `messageId` 不是直接读 `message.id`，而是先过：

- `persistedAssistantId(message)`
- 取不到才回退到 `request_id`

原因和 steer 分段一致：

- 如果某条回答在页面上已经被拆成多段 assistant，后面几段的 `id` 是 synthetic
- 直接用它们去拼 `/messages/<id>/artifacts/<index>` 这种地址，后端会认为这是不存在的消息
- 结果就是：文件按钮能显示出来，但下载/删除全 404

所以 session 级 artifact 聚合里，真正的寻址键始终是“后端落库的那条持久消息 id”，不能和前端列表渲染用的行 id 混为一谈。

### 为什么删除后不直接 splice

因为服务端把 artifact 的位置本身当成下载地址。

如果前端把中间一项删掉并重排：

- 之后的 index 就会错位
- 下载地址会对不上真实文件

所以已删除 artifact 只打 tombstone，不重排。

## 一个实用阅读路径

### 路径一：改“技能生成文件并在回答中引用”

1. `agent_stream_handler.go`
2. `artifact_reference.go`
3. `ChatArtifactsDrawer.vue`
4. `sessionArtifacts.ts`

### 路径二：改“页面刷新后继续显示正在生成的回答”

1. `stream.go`
2. `continue_stream_coalesce.go`
3. `useChatStreamHandler.ts`
4. `steerStreamFork.ts`
5. `views/chat/index.vue`

### 路径三：改“artifact 下载 / 列表 / 删除”

1. `artifact_download.go`
2. `artifact_delete.go`
3. `frontend/src/api/chat/index.ts`
4. `ChatArtifactsDrawer.vue`
5. `ArtifactLibrary.vue`

## 当前最值得记住的几个边界

- `answer.done` 不等于整条 Agent turn 已经完全落库。
- artifact 引用最终要绑定到消息自己的 artifact 列表，而不是 sandbox 临时路径。
- `continue-stream` 是“重放整条事件日志 + 接上 live tail”，不是纯断点续传。
- replay 阶段会 coalesce，live tail 不会。
- 会话级 artifact 列表只是展示聚合，下载寻址仍然以 message 维度为准。
