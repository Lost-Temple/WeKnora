# 聊天会话状态、Steer、Stop 与 Resume 阅读指南

这篇文档把聊天前端里最容易读散的几条线放到一起：

1. 页面重新打开后，怎么判断“这条回答还在跑”
2. 为什么 `continue-stream`、停止生成、artifact 下载都不能直接拿当前行 `id`
3. steer 为什么会同时存在队列项和一条 transcript 预览行
4. 新问题发出前，为什么前端还要先把旧回答“本地封口”

如果你准备改聊天页的流式体验、停止生成、续播恢复、steer 交互，这一篇可以作为会话前端的主入口。

## 推荐阅读顺序

1. `frontend/src/views/chat/index.vue`
2. `frontend/src/composables/useChatStreamHandler.ts`
3. `frontend/src/utils/steerStreamFork.ts`
4. `frontend/src/utils/steerId.ts`
5. `frontend/src/utils/sessionArtifacts.ts`
6. `website-docs/06-development/12-code-reading-agent-artifacts-continue-stream.md`
7. `website-docs/06-development/14-code-reading-session-fork-rewind.md`

## 1. 先记住这四个状态真源

聊天页读起来容易乱，是因为同一轮回答会在几个地方同时留状态。先记住下面四个“谁负责什么”：

### `messagesList`

负责 transcript 展示本身：

- 页面上有哪些 user / assistant 气泡
- 当前回答有没有被 steer 拆成多段 assistant segment
- preview 行长什么样
- artifacts、references、thinking 等已经落到消息对象上的展示数据

### `currentAssistantMessageId`

负责“当前这轮运行中的真实 assistant message id”。

这个字段不是当前显示行的 Vue key，而是后端真正认识的持久消息 id。凡是会打到后端或需要对准真实 turn 的动作，都应该优先经过它：

- `continue-stream`
- stop
- artifacts
- steer 的 `expected_assistant_message_id`

### `steerQueue`

负责 steer 队列的生命周期。

它记录的是：

- 哪条 steer 还在排队
- 期待挂到哪条 assistant 上
- 是否 pending / failed / promoting / awaitingIdleSend

只要还在 `steerQueue` 里，这条 steer 就仍然可能被发送、认领、重试。反过来，只在 transcript 里有一条 preview 行，不代表它还在排队。

### `fullContent`

负责当前非 agent 流的增量文本拼接缓存。

它只是前端暂存，不是真正消息真值。每次新消息发送前或 turn 完成后都会清空，避免新旧回答串到一起。

## 2. 为什么“当前显示行 id”经常不能直接用

聊天页最大的坑，是 UI 行 id 和真实 assistant message id 经常不是一回事。

### steer 会把一条 assistant 拆成多段

当回答进行中插入一条用户 steer 时，前端会把同一条持久 assistant message 拆成：

1. 回答前半段 assistant
2. 中途插入的 user 气泡
3. 回答后半段 assistant

这时第 3 段 assistant 常常只有一个 synthetic 行 id，目的是给 Vue 列表做 key。

### 真正该拿什么去打后端

真正能让服务端定位这轮回答的，始终是：

- `assistant_message_id`
- 或通过 `persistedAssistantId(message)` 解析出来的持久 id

所以你会看到聊天页里这些动作都在绕开“当前行 id”：

- fork / rewind 会同时匹配 `message.id` 和 `persistedAssistantId(message)`
- stop 最终会调用 `markInFlightAssistantStopped(currentAssistantMessageId.value)`
- `continue-stream` 恢复用的是 `persistedAssistantId(lastMessage)`
- session artifact 也会回退到持久 assistant message id

结论很简单：

**UI 可以拆段，但后端只认识一条真实 assistant message。**

## 3. 重新进入会话页时，怎么恢复一条还在运行的回答

这段主逻辑在 `chat/index.vue` 的历史 reload 完成后。

### 第一步：不是找最后一行，而是找最后一条未完成的 assistant

会话尾部不一定是 assistant：

- steer 注入后，尾部可能是 user 气泡
- 某些回放顺序下，列表最后一行也不一定是当前真实正在跑的那条回答

所以页面恢复不是看 `messagesList[messagesList.length - 1]`，而是：

- 倒序找 `role === 'assistant' && !is_completed`

### 第二步：恢复时强制规整到持久 id

一旦找到了 `lastMessage`，恢复逻辑会把：

- `resumeId = persistedAssistantId(lastMessage)`

写进 `currentAssistantMessageId`，然后再拿这个 id 去调 `continue-stream`。

这样即使当前命中的是某个 steer 分段 assistant，也仍然会恢复到同一条真实 message 上。

### 第三步：IM 回复单独走 quiet recover

IM 来源的回答不会经过当前 Web 端的 SSE 链路，所以直接打 `continue-stream` 往往会 404。

但这不表示回答不存在，而是它在另外一端继续生成。因此前端会：

- 给 IM turn 打上 `isAttachingImStream`
- 失败时进入更保守的 quiet poll-to-recover

Web / API 场景则继续保留原始报错语义。

## 4. stop 为什么分“本地封口”和“真正停掉”两层

停止生成不是一个单点动作，而是两层：

### 第一层：本地状态先收口

用户点击 stop 后，前端先：

- 断开当前 SSE
- 把当前运行中的 assistant 标成 stopped
- 清理必要的 loading / recover 标记

这里有一个很重要的函数：

- `markInFlightAssistantStopped(messageId?)`

它只负责把 transcript 里那条真正正在飞的 assistant 标成已停止，不立即清掉 stop 请求还要用到的真实 id。

### 第二层：服务端 stop / stop 事件再决定最终结果

后续如果服务端真的发来 `stop` 事件，`useChatStreamHandler` 会再次按：

- request id
- assistant message id
- trailing incomplete assistant

去 resolve 同一条消息，并把 stop 正确落到这条 assistant 上。

### 为什么 stop 失败时反而要继续显示“在生成”

`handleStopFailed` 会把：

- `isReplying = true`
- `loading = true`

重新设回去。

原因不是奇怪，而是准确：stop 接口失败只说明“没停下来”，不代表服务端已经停止生成。若前端此时把加载态关掉，用户会看到页面说没在生成，但答案还在继续冒出来。

## 5. 新问题发出前，为什么还要先“封口旧回答”

这段逻辑在：

- `prepareForNewOutgoingMessage()`

它会遍历 transcript，把所有未完成的 assistant turn 先本地封口，再清：

- `fullContent`
- `currentAssistantMessageId`

### 它不是 stop API

这里做的不是向服务端发停止请求，而是前端状态重置。

核心目的有三个：

1. 防止旧回答继续以“未完成态”挂在 transcript 上，和新 user 问题并存
2. 给 agent message 补 stop event，让 UI 和阅读工具知道这轮已经被本地切断
3. 把下一次发送需要的输出缓冲清干净，避免新旧回答文本串到同一个 `fullContent`

### 为什么这是必要的

聊天页允许很多“边生成边操作”的路径：

- stop 后立刻再问
- steer promote 失败后改成新一轮
- fork / rewind 后马上在新会话继续输入

如果没有这个“新外发前先封口”的动作，页面很容易同时保留两条看起来都还在生成的 assistant。

## 6. steer 为什么有“队列项”和“preview 行”两套状态

这是第二个很容易误判的点。

### `steerQueue` 是生命周期真源

`findSteerQueueItem` / `dropSteerQueueItem` 操作的都是队列真源。

它决定：

- 这条 steer 还在不在
- 还要不要发送
- 还能不能重试
- 是否期待挂到某条新的 assistant turn 上

### preview 行只是 UI 占位

`previewSteerMessage` 会立即在 transcript 里插一条 user 假行，目的是让用户看到“我刚点了 inject”。

这条行有两个特点：

- `isSteer = true`
- `_steerPending = true`

只要它还是 pending，就说明它只是乐观 UI，并不是真实持久消息。

### 为什么删除时经常要两边一起删

如果一条 steer 被取消、失败回滚或已经被别处消费，前端通常既要：

- `dropSteerQueueItem(steerId)`，停止生命周期驱动
- `discardSteerPreview(messagesList, steerId)`，移除 UI 假行

只做其中一个，就会出现：

- 队列没了，但页面还留着一条假气泡
- 页面没了，但队列里还在偷偷等待 attach / retry

## 7. 为什么 steer id 有 clientId / serverId 两套口径

为了让 steer 可以立即排队、立即画 preview，又兼容老后端不回显客户端 id，前端采用了两套 id：

### `clientId`

由 `makeSteerClientId()` 在本地立即生成。

作用：

- 先插 preview
- 先入 `steerQueue`
- 不必等后端回包才能在 UI 上看到结果

### `serverId`

服务端真正接受这条 steer 后，可能：

- echo 回 clientId
- 或老版本后端重新生成一条自己的 id

这时前端要调用：

- `reconcileSteerMessageId(list, clientId, serverId)`

把 transcript 和 queue 的 steer_id 统一到 serverId。

### 为什么不用 `crypto.randomUUID()`

因为纯 HTTP 部署下，`randomUUID` 并不总是可用；`crypto.getRandomValues` 更稳，所以前端手工拼一个 UUID v4 形状的字符串。

## 8. 这一模块和其它几篇文档怎么衔接

如果你要继续往深处读，建议按下面顺序接着看：

1. 先回到 [会话问答、引用与时间线阅读指南](11-code-reading-session-rag-references.md)，把回答为什么会落出 references / timeline 补齐
2. 再看 [Agent 会话、Artifact 与续播阅读指南](12-code-reading-agent-artifacts-continue-stream.md)，理解 artifact collecting、continue-stream replay/coalesce
3. 再看 [会话 Fork、Rewind 与分叉状态阅读指南](14-code-reading-session-fork-rewind.md)，理解为什么 fork / rewind 也都必须绕过 synthetic 行 id

## 9. 二开时最容易踩的坑

### 坑一：把 transcript 行 id 当成真实 assistant id

这会直接影响：

- stop
- continue-stream
- fork / rewind
- artifact 下载/删除

### 坑二：只删 preview，不删 steerQueue

页面看起来像删掉了，但后面 attach / retry 还会继续驱动这条 steer。

### 坑三：stop 失败时把 loading 关掉

用户会看到回答仍在继续生成，但页面已经说“没在生成”。

### 坑四：新问题发送前不先封口旧 turn

很容易得到两个同时未完成的 assistant，后续 complete/stop 事件也会串到错误的 turn 上。
