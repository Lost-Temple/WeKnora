# 技能安装 Transcript、Guidance 与 Repair Round 阅读指南

这篇文档专门整理“技能安装”这一条线里最容易混掉的三个概念：

1. `install-events` 是进度流，不是完整对话记录
2. `transcript` 是安装器 agent 自己的一条迷你聊天 transcript
3. `guidance` 是发给当前 installer run 的中途 steer，不是普通备注

如果你准备改技能安装进度、安装弹层、指导语输入、重装修复，这一篇就是主入口。

## 推荐阅读顺序

1. [sandbox_skill.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L635-L930)
2. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L874-L914)
3. [tenant_skill_transcript.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L1-L259)
4. [tenant_skill_steer.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go#L1-L211)
5. [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue)
6. [SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L259-L275)
7. [api/system/index.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/api/system/index.ts#L1148-L1262)
8. [17-code-reading-session-qa-skill-guidance.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/17-code-reading-session-qa-skill-guidance.md)

## 1. 先把技能安装看成“两条并行流”

技能安装弹层里，前端其实同时在读两种不同信息：

### `install-events`

对应 [InstallEvents](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L635-L737)。

它负责：

- 百分比
- 当前阶段
- 一行简短日志
- detached / done 这些进度终态

它的目标是给卡片和进度圈一个轻量、稳定、可长时间轮询的状态来源。

### `transcript`

对应 [InstallTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L763-L852)。

它负责：

- opening prompt
- thinking
- tool call / tool result
- command output
- 最终 answer
- repair round 的后续对话

它的目标不是百分比，而是“把安装器 agent 到底做了什么完整重放出来”。

一句话记忆：

- `install-events` 解决“现在进度到哪了”
- `transcript` 解决“它刚才到底做了什么”

## 2. 为什么 `install-events` 和 `transcript` 不能合成一个接口

这两个接口看起来都在跟踪安装过程，但职责完全不同。

### `install-events` 追求轻量和稳态

进度接口会：

- 优先读 Redis 进度广播
- 断订阅后回退到 skill 行状态轮询
- 期限到了发 detached frame

它允许只看最终状态，不需要完整历史。

### `transcript` 追求完整可重放

transcript 的语义和聊天里的 `continue-stream` 一样：

1. 先 replay 整条 event log
2. 再继续 tail live 事件

它必须保留历史，才能让管理员刷新弹层后还看见：

- 安装 prompt
- 某轮 thinking
- 某个命令为什么失败
- repair round 又追加了什么指令

如果把两者揉成一个接口，要么进度流太重，要么 transcript 丢历史。

## 3. `InstallTranscript` 为什么不能复用聊天的 `/sessions/continue-stream`

这点在 [InstallTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L763-L779) 的注释里已经说得很直白。

核心原因是**授权模型不同**：

- 聊天的 `/sessions/continue-stream` 是按“这条 chat session 属不属于你”授权
- 技能安装 transcript 是按“这个 sandbox config 里的 skill 归不归你这个工作区管理员看”授权

安装 session 本来就故意藏在普通会话列表之外，所以不能直接拿 chat session 端点来暴露它。

因此技能安装单独走自己的 handler：

1. 先 `resolveSkill`
2. 再从 skill 行拿 `install_session_id / install_message_id`
3. 再围绕这对 locator 读 transcript log

## 4. 为什么 transcript 要先写 locator，再让前端 follow

这一层在 [beginInstallTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L874-L914)。

安装一被接受，skill 行就已经存在，但 transcript 的 locator 还没有：

- `install_session_id`
- `install_message_id`

如果前端立刻 follow transcript，就会持续得到 404。

所以后端先：

1. 创建一条专用安装 session/message
2. 调 [newInstallTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L70-L87)
3. `Create(prompt)` 先落 user/assistant 消息壳
4. 把 locator 写回 skill 行

这样前端 [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L220-L274) 就能等 locator 就绪后再 follow，不用在安装前几分钟里疯狂 404 轮询。

## 5. `installTranscript` 为什么本质上是“安装器版 AgentStreamHandler”

[installTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L15-L64) 的职责，可以理解为服务层里的“精简版 AgentStreamHandler”。

它会把安装器 bus 上的事件投影成聊天兼容的流事件：

- `EventAgentThought` -> `thinking`
- `EventAgentToolCall` -> `tool_call`
- `EventAgentCommandOutput` -> `install_output`
- `EventAgentToolResult` -> `tool_result`
- `EventAgentFinalAnswer` -> `answer`
- `EventAgentComplete` -> 收尾到内存快照

所以前端能直接复用：

- [AgentStreamDisplay.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/views/chat/components/AgentStreamDisplay.vue)
- [useChatStreamHandler.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useChatStreamHandler.ts)

而不用再专门做一套“安装日志 UI”。

## 6. 为什么 `Create()` 要先落两条消息壳

看 [Create](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L89-L137)。

这和聊天发送主链路是同一条原则：

- transcript 还在运行时，前端也可能刷新或重新打开弹层
- 如果 assistant message 直到安装结束才创建，live follow 会因为 message 不存在而失败

所以先创建：

1. user prompt 行
2. assistant 壳

再把 opening prompt 也 append 到 event log 里，保证“一次 replay 就是整段完整对话”。

## 7. 为什么 `Finish()` 才是真正的终态关闭

[Finish](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L156-L199) 的语义很重要。

安装流程可能不是一轮就结束：

1. installer round 先跑一轮
2. verification 发现环境不对
3. 再带 guidance 开 repair round
4. repair 完了再验证

所以：

- `EventAgentComplete` 只能代表“某一轮 engine turn 结束了”
- `Finish(runErr)` 才代表“整个安装 verdict 已经定了”

这也是为什么 [onComplete](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L244-L269) 只更新内存快照，不直接发 terminal complete。

## 8. 为什么 `onToolCall` 会把之前的答案段标成 superseded

看 [onToolCall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L220-L242)。

安装器每一轮都可能先说一句类似：

- “我先检查一下环境”
- “看起来缺少 uv，我来安装”

这些其实只是**这一轮真正执行命令前的前导说明**。

如果这一轮后面真的调用了 tool，那这段 prose 就不是最终答案，不能永久粘到最后的 assistant 内容里。

所以 transcript 会把这些段标记成 `superseded`，只保留最终没有再触发工具调用的那一轮说明作为最后总结。

## 9. `guidance` 为什么本质上是 installer run 的 steer

核心服务在 [SteerInstall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go#L85-L145)。

这里有几个关键约束：

1. 必须带 `expected_message_id`
2. 必须带唯一的 `steer_id`
3. 必须通过独立的 `installSteerSession(...)` live-run 命名空间
4. 必须在分布式 lock 里再次确认当前 install run 没切走

这说明它和聊天 steer 是同一种模型：

- 发给“当前正在运行的那条 message”
- 支持 HTTP retry 幂等
- run 切了就必须拒绝

因此前端 live guidance 输入框其实不是“给 skill 留备注”，而是“往当前 installer agent 里插一条中途指导语”。

## 10. 为什么 guidance 要分 `pending / injected / unprocessed`

[InstallGuidance](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go#L41-L83) 会把 steer 事件翻译成 3 个前端状态：

- `pending`：已经接收，但当前 run 还没真正消费
- `injected`：installer 已经把它持久化成一条 user message 并写回 consumed
- `unprocessed`：run 已结束，这条指导不会再被当前安装执行

前端 [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L89-L150) 靠这份快照同时决定：

- 输入框还能不能继续发送
- 每条历史 guidance 该展示什么状态

## 11. 为什么 `PersistSteerMessage` 要用确定性 ID

看 [PersistSteerMessage](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go#L172-L206)。

这里把一条已消费 guidance 写成 transcript 里的 user 行时，不是随机生成 message id，而是：

- `sessionID + messageID + steerID` 做一个确定性 SHA1 UUID

原因是它要覆盖一种半成功场景：

1. user message 已经写成功
2. 但 steer event 的 `consumed=true` 回写失败

如果重试时再随机造一个新 id，历史里就会出现两条一模一样的 guidance user 行。

确定性 id 可以让重试安全落到同一行上。

## 12. 前端为什么分 `live` 和历史两种模式

前端 [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue#L220-L274) 明确拆成两种读取语义：

### `live=true`

- 优先 follow transcript
- guidance 输入框走 `steerConfigSkill(...)`
- 若 transcript 一时还没就绪，等 locator 落地后再 follow

### `live=false`

- 优先读 durable message history
- guidance 输入框改走 `reinstallConfigSkill(...)`

这样做是为了符合用户预期：

- 正在安装时，我是在看一个 live run
- 安装结束后，我是在查看一段历史记录；重装应该是新开一轮，而不是继续往旧 run 塞指导语

## 13. `SandboxSkillsPanel` 在做什么切换

[SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L259-L275) 是这套模式切换的入口。

它把：

- `session-id`
- `message-id`
- `live`
- `can-retry`

传给 [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue)。

因此这个面板并不自己理解 transcript/guidance 的细节，它只是根据当前 skill 行状态告诉时间线组件：

- 现在是 live 安装
- 还是已经结束，只允许 retry/reinstall

## 14. 这篇和前后文怎么接

建议这样连读：

1. 先看 [17-code-reading-session-qa-skill-guidance.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/17-code-reading-session-qa-skill-guidance.md)，理解“聊天发送入口”和“技能安装时间线为什么复用会话流模型”
2. 再看这一篇，把技能安装后端自己的 transcript / guidance / repair round 机制补齐
3. 如果要继续往深处读，再回到 [16-code-reading-backend-session-stream.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/16-code-reading-backend-session-stream.md)，对照聊天 continue-stream 看两者共享了哪些流协议约束

## 15. 二开时最容易踩的坑

### 坑一：把 `install-events` 当成 transcript

它只能告诉你进度，不能回答“安装器为什么这么做”。

### 坑二：每轮 `AgentComplete` 都直接关闭 transcript

repair round 会被截断，管理员正是为了看修复过程才打开这个面板。

### 坑三：guidance 只是普通备注

不是。live 阶段它本质上是 steer，必须对准当前 installer message。

### 坑四：run 切走后还允许把 guidance 发给旧 message

这会把指导语写进一条已经没有引擎会再消费的旧安装 run，前端看起来像“发成功了”，实际上永远不会执行。
