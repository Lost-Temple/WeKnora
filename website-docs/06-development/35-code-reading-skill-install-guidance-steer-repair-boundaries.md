# 技能安装 Guidance、Steer 与 Repair Round 边界阅读指南

这一篇接着 [34-code-reading-skill-install-tool-boundaries.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/34-code-reading-skill-install-tool-boundaries.md) 往后读，专门收拢技能安装里最容易混淆的三条线：

- 管理员在安装进行中追加的 `guidance`
- 后端把 guidance 投递给当前 installer run 的 `steer`
- verify 失败后由系统触发的 `repair round`

这条链最核心的结论是：

- **live guidance 只属于当前这条 installer run**
- **repair round 是同一条 run 的继续，不是重新安装**
- **安装结束后再输入指导语，语义就切换成“带说明重装一轮”**

## 推荐阅读顺序

1. [tenant_skill_steer.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go)
2. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go)
3. [tenant_skill_transcript.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go)
4. [sandbox_skill.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go)
5. [SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue)
6. [steerId.ts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/utils/steerId.ts)

## 1. 为什么技能安装 guidance 不能复用聊天 steer 命名空间

[installSteerSession](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go) 把安装 guidance 放进 `skill-install:` 前缀的独立 live-run 命名空间。

这样做是因为：

- 聊天会话也有 `/steer`
- 技能安装也要支持“中途插话”
- 但两者绝不能共享同一条 live run 标记

否则普通聊天的 steer 队列就可能误打到 installer shell，上下文会彻底串台。

## 2. `InstallGuidance` 返回的不是历史总账，而是“当前 run 快照”

[InstallGuidance](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go) 和 [SandboxSkillHandler.InstallGuidance](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go) 只读取：

- 当前 `tenant/config/skill`
- 当前 `install_session_id`
- 当前 `install_message_id`

然后给前端两个结果：

- `accepting`：当前 run 还接不接受新 guidance
- `messages`：当前 run 下 guidance 的可视状态

状态只分三种：

- `pending`：已经入队，但当前 run 还没消费
- `injected`：已经持久化成 transcript 里的 user message，并被这一 run 视为已知约束
- `unprocessed`：run 已结束或切换，这条指导不会再被当前安装消费

这也是为什么前端抽屉里看到的 guidance 列表，并不是“这个技能历史上所有管理员备注”。

## 3. `SteerInstall` 的真正边界：发给这一轮，而且只能发一次

[SteerInstall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go) 里有三层保护：

1. `expected_message_id` 必须等于技能记录里的当前 `InstallMessageID`
2. 进入 `withInstallSteerLock(...)` 后再重读一次技能行，防止和 run 关闭/切换竞争
3. `steer_id` 必须幂等，HTTP retry 不能重复 append 同一条 guidance

这里的判断结果非常重要：

- 当前 run 还活着：允许 append 到 steer 队列
- 当前 run 已结束或切走：直接返回冲突，让前端改走 reinstall

所以“我刚才输的这段指导语”到底是：

- 发给当前安装器继续执行

还是：

- 作为下一轮重新安装的开场说明

不是由前端文案决定，而是由 `expected_message_id + live marker + lock` 共同决定。

## 4. `PersistSteerMessage` 为什么一定要补 user message

[installSteerSink.PersistSteerMessage](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_steer.go) 做了两件事：

1. 把已消费的 guidance 持久化成 transcript 里的 user message
2. 把内容追加到 `sink.guidance`

前者解决“历史可读性”问题：

- 后面打开 transcript 时，能看见管理员究竟在什么时候插入了什么要求

后者解决“后续轮次继承上下文”问题：

- repair round
- 同一 run 的 continuation

都需要继续带上已经消费过的 guidance，否则 installer 下一轮会把管理员刚说过的话忘掉。

这里还专门用了确定性 message ID。原因是半成功场景必须可重试：

- user message 已写入
- 但 steer event 的 `consumed` 标记回写失败

这时下一次重试必须复用同一条 user row，而不是制造重复消息。

## 5. `round()` 是 guidance continuation 和 repair round 的汇合点

[installerRun.round](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go) 要处理两类“继续跑”的来源：

### 第一类：管理员又追加了 guidance

如果当前轮 agent 自然停下，但 steer 队列里又进来了新的 guidance，`closeIfDrained()` 就不会关闭 live marker，而是让 `round()` 继续下一次 `continuation`。

这时候追加的提示词不是重新安装，而是：

- 复用现有 sandbox 状态
- 只处理新 guidance
- 不重复已经完成的安装步骤

所以 continuation 的语义是：

- **同一条 run 的续跑**

不是：

- **重新 replay 一遍旧 transcript**

### 第二类：verify gate 触发 repair round

[installDependenciesAndVerify](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go) 在每轮 agent 停下后都会跑 verify。

如果 verify 返回的是可修复错误：

- 不是直接失败
- 也不是走 steer 队列
- 而是 `buildRepairPrompt(...)` 直接生成下一轮 prompt

然后把之前确认要保留的管理员 guidance 再拼回去。

所以 repair round 的语义是：

- **系统根据 verify finding 触发的下一轮安装指令**

不是：

- **管理员 guidance 的另一种表现形式**

## 6. `Finish()` 为什么不能在单轮 `agent complete` 时就关闭 transcript

[installTranscript.Finish](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_transcript.go) 只有在整个安装 verdict 定下来后才执行 terminal close。

原因就在于技能安装不是“单轮 agent complete 就结束”：

- agent 停下后还要 verify
- verify 失败后可能还有 repair round
- repair round 里又可能继续消费 guidance

如果 transcript 在第一轮 `complete` 时就关掉，后面的 repair 过程和最终失败原因都会丢失。

## 7. 前端为什么把 live guidance 和 retry 明确分成两种请求

[SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue) 已经把两种动作拆开了：

- `live === true`：调用 `steerConfigSkill(...)`
- `live === false`：调用 `reinstallConfigSkill(...)`

这不是 UI 层的随意分支，而是后端语义的直接投影：

- live 时，输入框表示“给当前 run 插话”
- ended 时，输入框表示“带说明重新安装”

同一个文本框，背后是两种完全不同的后端边界。

## 8. 为什么前端要保留 `pendingSend.id`

[makeSteerClientId](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/utils/steerId.ts) 生成客户端 steer ID，[SkillInstallTimeline.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SkillInstallTimeline.vue) 则在 `pendingSend` 里暂存：

- `messageId`
- `content`
- `id`

这样做是为了处理一个很常见的灰区：

- POST 其实已经成功
- 但随后 guidance 列表刷新失败

如果这时用户点 Retry，前端必须继续复用同一个 `steer_id`，否则后端会把它当成两条不同 guidance。

也就是说，客户端并不是“只是生成个随机 UUID”，而是在主动配合后端完成幂等保障。

## 9. 阅读这条链时最该盯住什么

### 第一，run 绑定边界

始终看清楚当前逻辑绑定的是：

- `install_session_id`
- `install_message_id`
- 还是新的 reinstall run

这三者一旦混淆，就会误判 guidance 为什么被拒绝。

### 第二，队列消费和 transcript 持久化是不是同一件事

不是。

- steer event 被 append，只代表“进入待处理队列”
- `PersistSteerMessage` 成功，才代表“这条 guidance 进入 durable transcript 历史”

二者之间的半成功场景，正是这条链幂等设计最关键的原因。

### 第三，repair 和 reinstall 是两种不同层级的继续动作

- repair：同一条安装 run 内，由 verify gate 触发下一轮
- reinstall：当前 run 结束后，重新开一条 run

repair 关注的是“修正当前安装”，reinstall 关注的是“重新开始一轮安装流程”。
