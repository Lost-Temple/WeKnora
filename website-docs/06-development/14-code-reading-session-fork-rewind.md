# 会话 Fork、Rewind 与分叉状态阅读指南

这份指南继续沿着会话主线往下，专门解释：

1. 会话为什么可以从某条消息分叉出新会话
2. 为什么 rewind 不只是“删掉后面的消息”
3. fork / rewind 后 sandbox、checkpoint、artifact、stream 这些状态怎么一起对齐

如果你准备改聊天里的“分叉”“回滚”“从某条消息继续编辑”这类能力，这一篇就是最直接的入口。

## 推荐阅读顺序

1. `frontend/src/views/chat/forkPoint.ts`
2. `frontend/src/views/chat/rewindView.ts`
3. `frontend/src/views/chat/index.vue`
4. `internal/handler/session/fork.go`
5. `internal/handler/session/rewind.go`
6. `internal/application/service/session_fork.go`
7. `internal/application/service/session_rewind.go`
8. `internal/handler/session/artifact_reference.go`
9. `internal/handler/session/artifact_storage.go`

## 1. 先看前端按钮怎么决定能不能点

文件：

- `frontend/src/views/chat/forkPoint.ts`
- `frontend/src/views/chat/rewindView.ts`

### fork 按钮规则

前端只做很轻的可用性判断：

- 目标消息必须是 `user` 或 `assistant`
- 当前 transcript 里不能有还没完成的 assistant turn

真正是否允许 fork，后端还会再重新校验。

### rewind 按钮规则

rewind 比 fork 更保守：

- embedded 模式不允许
- 只要当前有 outgoing work / streaming / recovering，就不允许
- 只要 transcript 里存在未完成的 assistant turn，就整段消息都不提供 rewind

原因很简单：rewind 是原地改源会话，不能和正在写入的 turn 并发。

## 2. 看前端点击后做了什么

文件：

- `frontend/src/views/chat/index.vue`

建议重点看：

- `handleFork`
- `handleRewind`

### `handleFork`

这里有两件事很关键：

#### 一：fork 点也必须先对到持久消息 id

和 rewind 一样，fork 点可能落在 steer 拆开的后半段 assistant 上。这行 assistant 的 `message.id` 只是前端列表用的 synthetic key，如果直接用它去找 `messagesList.find((m) => m.id === messageId)`：

- 会直接找不到源消息
- 分叉按钮点了就静默失败

所以前端找 fork 目标时要同时匹配：
- `message.id === 目标 id`（普通行、未被拆分的 assistant）
- `persistedAssistantId(message) === 目标 id`（steer 拆分出来的后半段 assistant）

并且一旦确认是 assistant 行，最终发往服务端的 `message_id` 会被规整成 `persistedAssistantId(source)`，避免把 synthetic row id 真的传去后端。

#### 二：用户消息 fork 时的文本暂存

另一个细节是：

- 如果 fork 点是用户消息，新会话历史里不会包含这条问题
- 但用户界面上又需要“从这条问题继续改写”

所以前端会把问题文本先存进 `sessionStorage`，等跳到新会话后再预填回 composer。

### `handleRewind`

这里除了“发请求 + 刷新历史”，还要特别注意一件事：**rewind 点不能直接拿 UI 行 id 去对。**

因为 steer 会把一条持久 assistant message 拆成“前半段回答 -> 插入用户消息 -> 后半段回答”来渲染。后半段那条 assistant 行只有一个 synthetic id，并不是后端真正保存的 message id。如果直接按那条行 id 发 rewind，服务端会定位不到真实消息。

所以前端在找 rewind 目标时，会同时匹配：
- `message.id === 目标 id`（普通行、未拆分行）
- `persistedAssistantId(message) === 目标 id`（steer 拆分后的后半段 assistant）

这样无论用户点的是哪个分段，最终发给后端的都是同一条真实持久消息 id。

#### 路径一：服务端 rewind 成功，历史 reload 也成功

那就直接用服务端返回的新历史替换本地 transcript。

#### 路径二：服务端 rewind 成功，但 reload 失败

前端会先本地裁掉确定应该消失的后缀，至少不继续展示已被服务端删除的消息。

这里的本地裁剪同样要走 `persistedAssistantId`，不能只看行 id，否则裁剪到 steer 后半段时会漏删真实 turn 的后半部分。

如果 rewind 点是用户消息，还会把那条问题重新塞回输入框，方便用户继续改。

## 3. 再看 handler：只是做协议翻译，不做业务推断

文件：

- `internal/handler/session/fork.go`
- `internal/handler/session/rewind.go`

### `ForkSession`

handler 的重点是：

- 读取 `session_id` / `message_id`
- 转成 `tenantID + userID + sessionID + messageID`
- 把 service 返回的 degrade / reason 原样交回前端

特别要注意：

fork 成功时，并不代表新会话已经有了 sandbox。

它只是把：

- 新 session
- 复制后的消息
- 可能的 `ForkBootstrap`

写好了，真正起 sandbox 仍然是 lazy provisioning。

### `RewindSession`

rewind handler 只负责把结果交给前端：

- 历史已经截断
- `workspace_reset` 是否真的发生
- 如果没 reset，是因为 `NO_SANDBOX` 还是 `NO_CHECKPOINT`

它自己不会根据 session 状态做额外猜测。

## 4. 看 fork service 的真正分支逻辑

文件：

- `internal/application/service/session_fork.go`

建议重点看：

- `Fork`
- `prepareBootstrap`
- `historyThroughForkPoint`
- `copyMessagesInto`

### fork 的三层结果

可以把 fork 的结果理解成三档：

#### 1. 完整 fork

历史复制成功，并且带着可用的 sandbox bootstrap：

- snapshot id
- commit sha
- source sandbox id

这样新会话第一次真正起 sandbox 时，就能落到 fork 点对应的工作区状态。

#### 2. degrade fork

历史复制成功，但 sandbox 状态带不过去，比如：

- 没 checkpoint
- sandbox 已替换
- sandbox 已不存在
- provider 不支持 snapshot

这时 fork 仍然成功，只是新会话会从一个全新的 sandbox 开始。

#### 3. 失败 fork

只有这些“用户必须处理”的情况才会直接失败：

- session 不存在 / 不属于当前用户
- fork point 消息不存在
- fork 点不是 user/assistant
- 源会话正忙

### `historyThroughForkPoint`

这是 fork 里最容易搞混的一条规则：

#### fork 点是用户消息

- 历史不包含这条用户消息
- 前端会把它预填回输入框

也就是“从这个问题重新开始问”

#### fork 点是助手消息

- 历史包含这条助手消息

也就是“以这条回答为结尾，再往后走一条支线”

### `copyMessagesInto`

复制消息时有两个关键细节：

#### request_id 要按原值映射重写

不能原样照搬父会话的 `request_id`，否则父子会话里同一轮 user/assistant 配对关系会串线。

#### KnowledgeID 要清空

因为父会话的消息早就已经被聊天历史知识库索引过了，fork copy 不能再把同一内容重复索引一遍。

## 5. 看 rewind service：原地修改要比 fork 更严格

文件：

- `internal/application/service/session_rewind.go`

建议重点看：

- `Rewind`
- `resetWorkspaceIfPossible`
- `syncPendingForkBootstrap`
- `cleanupDeleted`

### rewind 的核心难点

fork 是“复制一份新分支”，失败大不了不创建新会话。

rewind 是“原地回退当前会话”，所以必须保证：

- 消息历史
- sandbox workspace
- lazy bootstrap
- stream replay
- artifact 列表

这些状态不能互相打架。

### `resetWorkspaceIfPossible`

这一步决定会不会真的去 reset `/workspace`。

#### kept history 里有可达 checkpoint

就 reset 到那个 commit。

#### kept history 里有 assistant 消息，但没有可达 checkpoint

直接报 `ErrRewindNoCheckpoint`。

因为如果消息历史回退了、workspace 却留在更后面的代码状态，用户会看到明显错位。

#### kept history 连 assistant checkpoint 都没有

就不是 reset 到某个 SHA，而是把 workspace 还原成“空对话起点”的样子。

### `syncPendingForkBootstrap`

这是 rewind 里非常容易漏看的点。

有些 session 是 fork 出来的，但还从没真正起过 sandbox。它们只有一个待消费的 `ForkBootstrap`。

如果这时对它做 rewind，而 kept history 更早：

- 旧 bootstrap 的 `CommitSHA` 就可能已经超前

所以这里要么：

- 把 bootstrap 的 `CommitSHA` 改到 kept history 的最新 checkpoint

要么：

- 直接清掉 bootstrap，让它下次像普通新会话一样起 sandbox

### `cleanupDeleted`

rewind 删除消息后，不能只停在消息表：

- suggestions 要删
- stream replay 缓冲要删
- message knowledge 要删
- artifacts 要 tombstone

否则你会看到各种“消息没了，但旁边状态还活着”的残留。

## 6. 为什么 rewind 还要清 stream 和 artifact

### 清 stream

如果不把已删除消息的 replay stream 一起删掉：

- `continue-stream` 还能继续请求这些 message id
- 前端可能重新收到一段已经不该存在的 SSE 历史

### 清 artifact

如果已删除消息挂着的 artifact 不 tombstone：

- 会话文件面板还会列出它们
- 但正文历史里已经没有那条消息了

这就会让“聊天内容”和“会话产物”两套视图产生分叉。

## 一个实用阅读路径

### 路径一：改“从某条消息分叉新会话”

1. `forkPoint.ts`
2. `chat/index.vue` 的 `handleFork`
3. `fork.go`
4. `session_fork.go`

### 路径二：改“回滚到某条消息”

1. `rewindView.ts`
2. `chat/index.vue` 的 `handleRewind`
3. `rewind.go`
4. `session_rewind.go`

### 路径三：改“分叉/回滚后的 sandbox 一致性”

1. `session_fork.go` 的 `prepareBootstrap`
2. `session_rewind.go` 的 `resetWorkspaceIfPossible`
3. `session_rewind.go` 的 `syncPendingForkBootstrap`

## 当前最值得记住的几个边界

- fork 成功不等于已经创建了新 sandbox。
- 用户消息 fork 不包含该问题本身；助手消息 fork 包含该回答。
- rewind 成功不等于一定执行了 workspace reset，要看 `workspace_reset` 和 `reason`。
- rewind 删除消息后，stream / suggestions / artifacts 也要跟着清。
- 一个尚未消费的 `ForkBootstrap` 也可能因为 rewind 而被重定向或清空。
