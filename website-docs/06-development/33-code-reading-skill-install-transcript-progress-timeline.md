# 技能 Install Transcript、Progress 与阶段时序阅读指南

这一篇接着 [32-code-reading-skill-install-verify-runtime-gates.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/32-code-reading-skill-install-verify-runtime-gates.md) 往后读，专门讲技能安装里另一条很容易看混的链：

- transcript 负责完整回放安装器的对话和工具调用
- progress 负责给列表卡片和抽屉头部一个低噪声阶段感
- 前端同时消费这两条流，但不会让它们互相替代

一句话概括就是：

- **transcript 是完整时间线**
- **progress 是阶段摘要**

## 推荐阅读顺序

1. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L904-L1215)
2. [tenant_skill_transcript.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L18-L597)
3. [tenant_skill_progress.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go#L10-L92)
4. [sandbox_skill.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L586-L910)
5. [useConfigSkillInstallProgress.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useConfigSkillInstallProgress.ts#L1-L154)
6. [SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L717-L1231)

## 1. 为什么技能安装要同时维护两条流

如果只看进度条，用户能知道：

- 正在安装
- 已经到哪个阶段
- 有没有失败

但不知道：

- installer 到底跑了哪些命令
- verification 到底发现了什么
- repair round 为什么会突然开始补依赖

如果只看 transcript，又会有另一个问题：

- 列表卡片和抽屉头部只是想显示一个稳定的阶段摘要
- 没必要把每条 thought、tool_call、tool_result 都塞进去

所以项目把它拆成两层：

### transcript

- 完整时间线
- 面向“打开日志看细节”的人

### progress

- 阶段摘要
- 面向“当前只想知道到哪一步了”的人

## 2. `beginInstallTranscript` 为什么要尽早写 locator

[beginInstallTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L904-L950) 在真正 installer 开跑前就先做了三件事：

1. 生成 `assistantMessageID`
2. `Create` transcript 的 user / assistant 消息壳
3. 把 `install_session_id` / `install_message_id` 写回 skill row

这样前端在安装初期就已经能拿到：

- transcript 要 follow 的 session/message locator

否则会出现一个很差的体验：

- skill row 已经是 `installing`
- 但 transcript 还没有对象可跟
- 抽屉一开就一直 404 / 空白

## 3. transcript 为什么和 `continue-stream` 很像，但又不是同一个接口

install transcript 在 handler 侧由：

- [InstallTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L756-L859)

提供。

它的跟随语义和聊天 `continue-stream` 很像：

1. 先 replay 现有事件
2. 再持续 tail 新事件

但它没有直接复用聊天接口，因为权限语义不同：

- 聊天流按 session 属主授权
- 技能安装 transcript 按 workspace / sandbox config 授权

安装 session 本身又是故意隐藏在普通会话列表之外的。

## 4. `installTranscript` 真正在记录什么

[installTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L18-L79) 只订阅安装路径实际可能产生的事件：

- thought
- tool call
- install output
- tool result
- final answer
- error
- complete

它不是聊天 `AgentStreamHandler` 的完整拷贝，而是安装模式的精简投影。

这说明设计上有一个明确边界：

- transcript 的目标是让前端复用聊天渲染组件
- 不是把聊天世界里所有事件类型都拖进安装服务层

## 5. 为什么 transcript 会“撤回”前导 prose

[onToolCall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L263-L303) 有一个非常关键的动作：

- 这一轮一旦开始调用工具，就把之前尚未 supersede 的 answer segment 标成 superseded

原因是 installer 常常会先说一段：

- “我准备先创建 venv，再安装依赖……”

如果把每轮这种前导 prose 全都保留到最终 `Message.Content`，最后消息会变成：

- 一堆中间态推理 + 最后结论 全部粘在一起

所以 transcript 这里选择：

- 日志流里仍然保留每轮的流式片段
- 但最终 assistant message 只保留真正收束成 verdict 的那轮 prose

## 6. 为什么 transcript 会反向驱动 progress 的 35→79 区间

在 [beginInstallTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L926-L935) 里，`newInstallTranscript` 接了一个 `onActivity` 回调。

它做的是：

- 每次 tool call 来了
- 就发布一帧 `Stage=agent`
- 百分比走 [asymptoticInstallPercent](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L576-L597)

这一步很巧：

- progress 本身不知道 installer 最终会跑多少条命令
- 但 transcript 知道“刚刚又发生了一次工具调用”

于是项目把 transcript 当成 progress 的活动信号来源，用来填满：

- `seeded` 35%
- 到 `agent_done` 80%

之间那段最容易“看起来卡死”的时间。

## 7. 为什么 `muteActivityProgress` 要在 `agent_done` 后立刻触发

[installDependenciesAndVerify](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1000-L1009) 在每轮 agent 停下后，会先发：

- `Stage=agent_done`
- `Percent=80`

然后立刻调用：

- [muteActivityProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L559-L567)

原因是后面即使进入 repair round，也不能让新的 tool call 把百分比从 82 又拉回 70 多。

也就是说：

- transcript 驱动的活动进度，只负责“agent 正在忙”
- 一旦进入 verify / repairing 这些显式阶段，进度锚点改由 install path 明确发布

## 8. 为什么 `agent_done` 不是“安装完成”

这点非常容易被误解。

在 [installDependenciesAndVerify](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1000-L1035) 里：

- `agent_done = 80`

只表示：

- **这一轮 installer agent 已经停止输出**

它不表示：

- verify 已通过
- repair 已结束
- ready row 已写回

所以 `agent_done` 更像是：

- transcript 活动段落的分界点

而不是：

- 整个 install 生命周期的终点

## 9. 为什么 `verify_note` 只发一帧聚合日志

[reportVerificationNotes](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1162-L1190) 会把多个 notes 合成一帧：

- `Stage=verify_note`
- `Log=多行文本`

而不是一条 note 发一帧。

原因是 progress 只存最新值：

- 如果逐条发，晚来的订阅者只能看到最后一条 note

所以这里选择：

- progress 只保留这一阶段的聚合摘要
- 想看更完整上下文的人打开 transcript

## 10. `SkillProgress` 为什么故意只存最后一帧

[SkillProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go#L16-L34) 和 [publishProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go#L40-L66) 有一个非常明确的边界：

- 只存最新值
- 不入库
- 只放 Redis
- TTL 只有 30 分钟

这说明 progress 从设计上就不是历史系统。

它的职责只有两个：

1. 给新 SSE 连接补首帧
2. 给已经跟随中的界面持续推最新阶段

长期要追责、排查、回放时，项目期待你去看的不是它，而是：

- transcript
- durable skill row

## 11. 为什么 install-events 首屏要先播 `LastProgress`，再播 durable terminal

[InstallEvents](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L647-L754) 的首屏顺序是：

1. `LastProgress`
2. `terminalSkillEvent`

这个顺序很重要。

如果先看 durable row：

- 新连接可能先看到空白
- 然后才等到下一次阶段广播

如果只看 `LastProgress`：

- 又可能把一条过期的 60%/80% 误当成当前仍在运行

所以这里采用了“两步收口”：

- 先用 Redis 最近一帧点亮界面
- 再用数据库终态兜底确认这条流是否其实已经结束

## 12. 为什么 `terminalSkillEvent` 是 install-events 必不可少的补尾

[terminalSkillEvent](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L598-L628) 是 install-events 最大的可靠性补丁。

因为 progress 流本身可能中断于：

- Redis 不可用
- 进程重启
- terminal frame 还没发出去进程就死了

这时如果只依赖 pub/sub，前端会永远等不到“done / failed / removed”。

所以 install-events 最终一定还要回到：

- durable skill row

来判断：

- 这条流现在到底是不是已经结束

## 13. 为什么 transcript 完成与 progress 完成并不是一回事

在 transcript 侧：

- [Finish](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go#L184-L232) 才会真正写 `ResponseTypeComplete`

在 progress 侧：

- 终止条件来自 `done/failed/removed/detached`

两者并不要求同时发生。

比如：

- transcript 可能已经写完最终 error 和 complete
- 但前端列表还要等下一次 skill row reload 才把 status 刷成 failed

或者：

- progress 因为 `detached` 停止跟随
- transcript 仍然能在下次打开时从事件日志完整 replay

这也是为什么前端要同时消费：

- 一条阶段流
- 一条时间线流

## 14. 前端为什么要把 progress 缓存在 composable 里

[useConfigSkillInstallProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useConfigSkillInstallProgress.ts#L26-L154) 维护了一份：

- `progressByKey`

它的定位不是“最终状态存储”，而是：

- 当前页面作用域内的阶段缓存

这样：

- 列表 reload 不会把进度条瞬间抹掉
- 详情抽屉重开时还能先看到上一帧 progress
- 多个技能卡片能共享同一份 follow 状态

## 15. 为什么 `done` 了还可能继续重连

在 [useConfigSkillInstallProgress.follow](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/composables/useConfigSkillInstallProgress.ts#L61-L126) 里，`parsed.done` 以后分两种：

### 真 finished

- `installRunFinished(parsed)` 为真
- 触发 `onDone`

### 只是 `detached`

- 说明服务器停止陪跑，但任务仍可能在继续
- 前端会再次 `follow`

所以 `done` 这个布尔值在 install-events 语义里并不总等于：

- 任务结束

它也可能只是：

- 这条 SSE 连接的当前生命周期结束

## 16. `SandboxSkillsPanel` 为什么同时保留 progress 和 transcript 的消费逻辑

[SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L717-L1231) 里：

- `progressOf` / `progressStageText` / `progressLog` 消费 progress
- `hasTranscript` / `onTranscriptVisible` / `transcriptEpoch` 消费 transcript

这两组逻辑没有合并，是因为目标不同：

### progress 消费链

- 列表卡片
- 抽屉头部
- 删除/安装中的即时阶段提示

### transcript 消费链

- 详情弹窗里的完整安装时间线

如果把 transcript 直接拿来填卡片阶段文案，界面会太嘈杂；如果只靠 progress，详情弹窗又失去排障价值。

## 17. 为什么 `onTranscriptVisible` 重开时要 bump `transcriptEpoch`

[onTranscriptVisible](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L926-L939) 在弹窗重新打开时会：

- `transcriptEpoch += 1`

它的作用是强制时间线组件重新从头 replay。

因为 transcript 的价值就在于：

- 重建完整上下文

不是只从“上次你关掉弹窗时的 offset”继续往后追。

而 progress 缓存则可以继续沿用，因为它只负责：

- 当前阶段摘要

## 18. 为什么 `followBusySkills` 要在每次列表 reload 后重建

[followBusySkills](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1167-L1175) 会按当前 busy skills 重建 follow 集合。

原因是 install-events 是：

- 短期连接
- 会断开
- 会因为抽屉切换、列表刷新、组件卸载而重建

如果不在 reload 后同步 busy 集合，最常见的问题就是：

- 技能其实还在安装
- 但前端已经不再跟随
- 进度条停在旧快照

## 19. 这篇和前后文怎么接

建议按下面顺序连读：

1. [18-code-reading-skill-install-transcript-guidance.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/18-code-reading-skill-install-transcript-guidance.md)
2. [19-code-reading-skill-progress-state-transitions.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/19-code-reading-skill-progress-state-transitions.md)
3. [32-code-reading-skill-install-verify-runtime-gates.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/32-code-reading-skill-install-verify-runtime-gates.md)
4. 当前这篇 `33`

这样能把技能安装的“会话化表现层”按层读完：

- transcript 如何把安装器过程投成聊天时间线
- progress 如何提供列表阶段感
- verify / repairing 如何插进这两条流
- 前端最终又如何把它们拼成“卡片摘要 + 弹窗细节”

## 20. 二开时最容易踩的坑

### 坑一：把 progress 当成完整历史日志

它从设计上就只保留最后一帧。

### 坑二：把 transcript 当成最终状态真相

真正的终态兜底仍来自 durable skill row。

### 坑三：看到 `agent_done` 就当 install 已经成功

它只是 agent 这一轮停下了，verify 和 repair 还可能继续。

### 坑四：让 repair round 的 tool call 继续驱动活动百分比

这样会把 82 的 repairing 又拖回 70 多，进度条会看起来倒退。

### 坑五：重开时间线弹窗时只从旧 offset 往后追

transcript 的价值恰恰是“能从头回放完整上下文”，不是只看最新几帧。
