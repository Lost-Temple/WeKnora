# 技能安装 Stop、Stuck-Run Reaper 与恢复路径阅读指南

这篇文档专门整理技能安装/删除卡住时的恢复链路，重点回答三个问题：

1. 操作者点击“停止安装”时，系统到底做了什么
2. 如果进程已经死了、没人能点停止，后台怎么把残留 `installing/removing` 行自愈掉
3. 前端的 `retry / remove / stop` 按钮为什么能在这些恢复路径上继续工作

如果你准备改技能安装失败恢复、卡住任务自愈、stop 按钮行为，这一篇就是主入口。

## 推荐阅读顺序

1. [tenant_skill_stop.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_stop.go)
2. [sandbox_skill.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L427-L453)
3. [tenant_skill_reaper.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L72-L189)
4. [tenant_skill_reaper.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L708-L738)
5. [tenant_skill_reaper.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L846-L890)
6. [SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1361-L1460)
7. [19-code-reading-skill-progress-state-transitions.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/19-code-reading-skill-progress-state-transitions.md)

## 1. 先分清三种“停下来”

技能安装/删除相关的“停止”其实有三种完全不同的语义：

### 人工 stop

由 [StopSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_stop.go#L99-L128) 处理。

语义是：

- 操作者现在就想打断当前安装
- 目的是尽快解除 `installing`，让 UI 可以 retry / uninstall

### 后台 reaper 自愈

由 [ReapStuckRuns](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L72-L189) 处理。

语义是：

- 这条 run 已经没人能 stop 了
- 但数据库还残留着 `installing/removing`
- 需要定时扫尾，把 durable row 恢复到合理状态

### 前端“停止跟随”

这不是任务停了，而只是：

- 某条 SSE / transcript 连接不再继续跟随

比如 [install-events](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L644-L746) 的 `detached`，只是连接结束，不是任务 verdict。

## 2. `StopSkill` 的目标不是“优雅收尾”，而是“立刻解锁 UI”

读 [StopSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_stop.go#L99-L128) 时，最重要的是理解它的优先级：

- 第一优先级：让操作者现在就能从 `installing` 脱身
- 不是：完整做完后台清理

它会做两件事：

1. 如果当前进程里真有这条 skill 的 run 在跑，就 cancel 那个 goroutine
2. 无论能不能 cancel 到，都立刻把 skill row 改成失败态

第二步很关键，因为：

- 进程刚重启过时，往往已经没有 goroutine 可取消
- 但前端不能继续等下一次 reaper 五分钟 sweep 才从 `installing` 解开

所以 `StopSkill` 是一个“同步止血”动作。

## 3. `bindSkillRun` / `withSkillRunLock` 在解决什么

核心在 [bindSkillRun](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_stop.go#L26-L49) 和 [withSkillRunLock](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_stop.go#L89-L97)。

这里把一条 install/remove run 的取消句柄，按：

- `tenantID + configID + skillID`

登记到一把业务键上。

这样带来的效果是：

1. 同一个 skill 同时只有一条真正运行中的 install/remove
2. `StopSkill` 可以不认识 job id，只按 skill 维度打断当前 run
3. retry/upgrade/new remove 开始时，旧 run 的 cancel 句柄会被替换掉

也因此 `unbind` 必须在 goroutine 退出时始终执行，否则下一轮运行会误命中上一轮残留的 cancel。

## 4. 为什么 stop 只支持 installing，不支持 removing

看 [sandbox_skill.go#Stop](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L427-L453) 和 [StopSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_stop.go#L111-L121)。

删除路径没有做“前端直接 stop removing”的语义，原因是它和安装不一样：

- 安装失败了，通常还能 retry 或重装
- 删除一旦走到中途，很多步骤已经不可逆

所以 removing 的恢复不是“人工回滚”，而是由 reaper 按事实判断：

- live image 里技能文件还在吗
- row 还该恢复成 ready，还是应该彻底删掉

## 5. `ReapStuckRuns` 处理的是哪类问题

[ReapStuckRuns](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L72-L189) 不是在处理用户主动点击 stop，而是在处理：

- 进程死了
- goroutine 消失了
- row 还留在 `installing/removing`
- 但当前已经没人能对这条 run 发 stop

它的核心问题只有一个：

> 现在每个新 session 实际启动出来的 live image，到底还带不带这个技能的文件？

这个问题决定 row 应该恢复成什么。

## 6. stuck installing 为什么有时恢复成 ready，有时恢复成 failed

对于 `status=installing` 的残留行，reaper 会先判断：

- live image 里是否仍然带着这个技能

如果答案是“带着”，说明发生的是：

- 安装其实已经成功切到 live image
- 只是最后的 terminal write / row update 丢了

这时 row 应该被修回 `ready`，否则 UI 会一直把一个实际上可用的技能藏在 `installing` 里。

如果答案是“不带”，那就说明：

- 这轮安装真的没落地

这时才改成 `failed`。

## 7. stuck removing 为什么有时恢复成 ready，有时直接删行

对于 `status=removing` 的残留行，逻辑和安装相反：

### live image 里还带着技能文件

说明删除没真正完成，row 应该恢复成 `ready`，让操作者还能 retry。

### live image 里已经没有技能文件

说明删除其实完成了，只是最后删 row 的动作没落地。

这时 reaper 会直接删掉 skill row，因为继续保留一条技能记录只会误导 agent 和 UI。

## 8. reaper 为什么必须看 live image，而不是只看 row 自己

这点在 [skillFilesInLiveImage](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L252-L280) 附近的注释里讲得很透。

不能只比较：

- row 上的 `InstalledSnapshotID`
- 当前 config 的 live snapshot id

因为一个 skill 可能是几代之前装进去的，而后面其他 skill 的安装又不断推进了 snapshot 链。

真正该问的是：

- **当前 live snapshot chain 上，是否还能追溯到这个技能的安装结果**

所以 reaper 必须读：

- config 当前 live pointer
- snapshot ledger parent chain

而不是只看单个 row 的快照 id。

## 9. `buildStillRunning` 在保护什么

看 [buildStillRunning](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L721-L738)。

它不是在问“这条 row 旧不旧”，而是在问：

- **那个生成当前 snapshot 的 run 现在是不是真的还活着**

原因是：

- 很大的镜像在慢 provider 上可能真的会跑很久
- 如果 reaper 只按时间阈值就把它当尸体清掉，甚至删掉 provider snapshot，反而会破坏一条其实快要成功的真实运行

所以这里还要结合 `InstallingSince` 心跳，只有长时间不再刷新，才当 stuck。

## 10. `Start()` 和 `Stop()` 在启动什么后台机制

看 [Start](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L846-L872) 和 [Stop](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L874-L890)。

`Start()` 会注册一个每 5 分钟跑一次的后台 sweep，内容包括：

- stuck install/remove reaper
- superseded snapshot 清理
- snapshot reconcile
- host skill tree sweep

也就是说，reaper 不是单次补救函数，而是技能系统的长期 housekeeping。

`Stop()` 则只负责停 cron 和等待当前 sweep 收尾，不会主动改写任何 skill row。

## 11. 前端的 `stopSkill()` 为什么要先忘掉旧进度

看 [SandboxSkillsPanel.vue#stopSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1415-L1433)。

一上来先：

- `stoppingId = skill.id`
- `forgetProgress(skill.id)`

原因是 stop 成功后，旧的 install-events 进度已经不再可靠：

- 可能最后一帧还停在 `seeded`
- 但 row 已经被同步改成 failed/stopped

所以前端先丢掉旧进度缓存，再合并 stop 接口返回的新 row，并 reload durable 列表收口。

## 12. 为什么 stop、retry、remove 都是“先解锁 durable row，再继续操作”

看 [retrySkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1361-L1383)、[stopSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1415-L1433)、[removeSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1435-L1460)。

这三条前端动作虽然目标不同，但都共享一个原则：

- **先让 durable row 回到一个可继续决策的状态**

例如：

- stop：把 `installing` 立刻改写掉
- retry：重新发起一轮新的 install
- remove：把 row 切到 `removing`，再跟随新的 progress

这样 UI 不会卡在“上一轮半死不活”的状态上。

## 13. 这篇和前后文怎么接

建议这样连读：

1. 先看 [19-code-reading-skill-progress-state-transitions.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/19-code-reading-skill-progress-state-transitions.md)，理解 install-events 和 durable row 是怎么配合的
2. 再看这一篇，理解当 durable row 卡在 installing/removing 时，stop 和 reaper 分别怎么救
3. 如果还继续往下补，可以再整理一篇“snapshot ledger / prune / reconcile”专题，把 reaper 依赖的 live image 判断链单独展开

## 14. 二开时最容易踩的坑

### 坑一：把 StopSkill 当成完整后台清理

它的核心目标是立刻解锁 UI，不是把所有 provider/sandbox 资源都同步回收完。

### 坑二：reaper 看见 stuck 就一律改成 failed

不行。必须先判断 live image 里到底还有没有这个技能。

### 坑三：允许 removing 也走和 installing 一样的 stop 逻辑

删除路径的恢复语义不同，很多步骤不可逆，应该交给 reaper 按事实判断来恢复。

### 坑四：stop 成功后继续沿用旧进度缓存

那会让 UI 一边显示“已停止/已失败”，一边还挂着旧的 `seeded/verified` 百分比，看起来像状态打架。
