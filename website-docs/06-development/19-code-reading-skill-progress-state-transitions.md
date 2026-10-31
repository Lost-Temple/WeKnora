# 技能安装/删除进度流与状态迁移阅读指南

这篇文档专门解释技能安装/删除里最容易混淆的两层状态：

1. **durable 状态**：数据库里 `tenant_skills.status / error / installing_since`
2. **transient 进度**：Redis 里的 `SkillProgress` 和 `install-events` SSE

如果你准备改技能进度条、删除/安装卡片、detached 提示、失败收口，这一篇就是主入口。

## 推荐阅读顺序

1. [tenant_skill_progress.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go)
2. [tenant_skill_admin.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_admin.go#L153-L184)
3. [sandbox_skill.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L576-L737)
4. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L193-L205)
5. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L541-L595)
6. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1495-L1504)
7. [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L35-L48)
8. [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L207-L220)
9. [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L438-L470)
10. [SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1072-L1218)

## 1. 先分清：数据库状态不是进度条

技能安装/删除有两个层次的状态：

### durable truth

在数据库 `tenant_skills` 行里，最重要的是：

- `status`
- `error`
- `installing_since`

这些字段回答的是：

- 这个技能现在是 `ready / failed / installing / removing`
- 上次失败原因是什么
- 当前这轮是否还在跑、跑了多久

### transient progress

而 [SkillProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go#L19-L24) 回答的是：

- 现在大概到百分之多少
- 当前处于哪个细阶段
- 当前阶段想展示的一行日志是什么

它只服务“当前操作者正在看的 UI”，不是长期事实。

一句话记忆：

- `status` 决定技能最终是什么
- `progress` 只描述这一轮运行现在走到哪

## 2. 为什么 `SkillProgress` 故意不入库

看 [tenant_skill_progress.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go#L16-L46)。

这里的设计原则很明确：

- durable truth 必须可长期依赖
- percentage/stage 这种过程态不该长久存在

如果一条安装早就失败或完成了，数据库里却还长期保留着：

- `35% seeded`
- `60% removed`

那比完全没有进度条更误导，因为它看起来像“任务卡在半路”。

所以 `SkillProgress` 只写 Redis，并带短 TTL：

- 新打开的 SSE 还能看到最近一次进度
- 放一段时间后自动消失

## 3. `publishProgress` 同时做哪两件事

[publishProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go#L30-L46) 每次发布都会同时：

1. `SET` 到 Redis key
2. `PUBLISH` 到 Redis pub/sub

这两步分别服务两类连接：

### 已经在看的连接

靠 pub/sub 直接收到最新阶段，进度条实时跳。

### 刚打开的新连接

先通过 [LastProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go#L48-L66) 补一帧最近值，避免先空白等待下一次广播。

## 4. 没有 Redis 时为什么系统仍然能工作

这一点是整个设计里很关键的降级策略。

### service 层

[SubscribeProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_admin.go#L153-L177) 在没有 Redis 时返回：

- `nil` channel
- 一个空 `release`

这不是错误，只是“当前没有 live 进度流”。

### handler 层

[InstallEvents](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L620-L737) 看到 `nil` channel 后，并不会直接失败，而是退化成：

- 定时轮询 skill row
- 用 durable status 合成 terminal frame
- 用注释帧保持连接活着

所以没有 Redis 时，系统少的是“实时细粒度百分比”，不是安装/删除能力本身。

## 5. `InstallEvents` 为什么是“三层收口”

读 [InstallEvents](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L620-L737) 时，可以把它理解成三层收口：

### 第一层：补最近一次进度

用 [LastProgress](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_progress.go#L48-L66) 先补首帧。

这样新连接打开时，不用等下一次 pub/sub 才看到百分比。

### 第二层：读 live 广播

有 Redis 时，`SubscribeProgress(...)` 提供真正的实时阶段推进。

### 第三层：durable 状态兜底

即使：

- 没有 Redis
- pub/sub 断了
- 进程在发布 terminal frame 前崩了

handler 仍然会通过 [terminalSkillEvent](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L593-L616) 从 skill row 合成最后一帧。

因此 install-events 这条 SSE 不会因为 transient progress 丢了就永远悬挂。

## 6. `terminalSkillEvent` 在解决什么

[terminalSkillEvent](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L593-L616) 的意义是：

> “只要 durable 状态已经表明任务结束，SSE 就必须能结束。”

它覆盖三类情况：

### 删除结束后 skill 行已经没了

`skill == nil` 时，直接合成：

- `stage=done`
- `status=removed`

因为删除的最后一步就是删行本身。

### 安装/删除失败

如果 `skill.status == failed`，就把：

- `stage=failed`
- `log=skill.error`

补成终态帧。

### 其它已结束状态

例如 `ready`，则统一翻成：

- `stage=done`
- `status=ready`

## 7. `detached` 是什么，为什么它不是失败

handler 里还有一个特殊阶段：

- `detached`

它出现在 [InstallEvents](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/handler/sandbox_skill.go#L725-L734) 的 deadline 分支。

语义不是：

- 安装失败了

而是：

- “这条 SSE 连接不再继续跟随这轮任务了”

也就是说任务本身可能还在跑，只是这次跟随已经到达最大陪跑时间，需要：

- 重新连接
- 或回到技能列表看 durable 状态

所以前端必须保留最后一个百分比，而不是把它误显示成失败。

## 8. 安装进度的主要阶段是怎么定义的

核心发布点在 [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L193-L205)、[tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L541-L595) 和 [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1495-L1504)。

### `accepted` 10%

表示：

- skill row 已切到 `installing`
- 后台 goroutine 已经排起

但这时 sandbox 还没 ready。

### `sandbox_ready` 25%

表示：

- 已拿到这轮安装要用的临时 sandbox

从这里开始，后续阶段才是在描述“真正执行安装工作”的进度。

### `seeding` 28%

大技能包复制文件本身就可能很慢，所以单独占一个阶段，避免用户误以为卡住。

### `seeded` 35%

表示：

- 技能源码已经复制进 sandbox

### `verified` 90%

表示：

- agent 安装动作完成
- gate 校验也通过

后面剩下的是 snapshot / 指针切换 / 落最终 row。

### `done` 100%

表示：

- durable row 已切到 `ready`
- install-events 可以真正终止

## 9. 删除进度为什么和安装类似但不完全一样

看 [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L35-L48)、[tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L207-L220) 和 [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L438-L470)。

删除也复用同一套进度协议，但阶段含义不同：

### `accepted` 5%

row 已切到 `removing`，真正的 sandbox 删除工作还没开始。

### `sandbox_ready` 30%

表示已经拿到用于拆技能目录的临时 sandbox。

### `removed` 60%

表示：

- 技能目录已删
- manifest 已删
- scratch 已清

但最终 `done` 还要等 snapshot / ledger / row 清理。

### `failed` 100%

删除失败时会把失败原因直接带进 progress log，方便前端直接展示。

## 10. 前端为什么不能只看 `skill.status`

前端面板逻辑主要在 [SandboxSkillsPanel.vue](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1072-L1218)。

如果只看 `skill.status`，最多只能知道：

- `installing`
- `removing`
- `ready`
- `failed`

但不知道：

- 现在是 `sandbox_ready` 还是 `seeding`
- 失败前到底卡在哪一步
- detached 时最后停在多少

所以面板优先读 `progressEvent(skill.id)`，再回退到 durable row。

## 11. `progressOf()` 为什么要有保底值

[progressOf](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1072-L1079) 的逻辑是：

1. 先读 live progress percent
2. 没有的话：
   - `removing` 返回 5
   - `ready / failed` 返回 100
   - 其它返回 0

这样做是因为：

- 有些环境没有 Redis，永远拿不到 live percent
- 但 UI 仍然需要一个不至于完全空白的保底显示

## 12. 为什么 `progressStageText()` 和 `progressLog()` 要分开

[progressStageText](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1087-L1098) 主要负责：

- 把阶段枚举翻译成可读文案

[progressLog](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1100-L1105) 主要负责：

- 显示服务端额外带的一行日志

两者分开后：

- 有阶段但没日志时，仍有稳定文案
- 有失败日志时，能优先展示原因

## 13. `followBusySkills()` 在解决什么

看 [followBusySkills](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1144-L1150)。

skill 列表每次 reload 后，前端都会重新同步当前 busy 集合到全局 progress follow。

原因是 install-events 是短期流，不是 durable 订阅：

- 关闭抽屉
- 切 skill
- reload 列表

都会让原来的跟随关系失效。

如果不每次按 busy skill 重建，进度条会停在旧快照上不再动。

## 14. `overlayUninstallStatus()` 为什么要乐观叠 removing

看 [overlayUninstallStatus](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/frontend/src/components/SandboxSkillsPanel.vue#L1179-L1190)。

删除按钮点下去后，会有一个小窗口期：

1. 本地已经知道用户开始卸载
2. 列表接口还没刷新回来
3. progress SSE 也还没到

这时若仍显示 `ready`，用户会觉得按钮没生效。

所以这里先本地乐观叠成 `removing`，直到真正的列表状态或 progress 事件接管。

## 15. 这篇和前后文怎么接

建议这样连读：

1. 先看 [18-code-reading-skill-install-transcript-guidance.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/18-code-reading-skill-install-transcript-guidance.md)，理解 transcript/guidance 是如何描述“安装器到底做了什么”
2. 再看这一篇，理解 install-events 和进度条为什么只负责“现在走到哪”
3. 如果继续往后补，可以再回到删除/重装的主 service，整理“状态迁移 + stuck-run reaper”那条恢复链

## 16. 二开时最容易踩的坑

### 坑一：把进度百分比当 durable truth

百分比只是临时 UI 快照，真正可信的是 skill row。

### 坑二：没有 Redis 就让 install-events 直接报错

正确做法是退回 durable 状态轮询，而不是把功能做成强依赖。

### 坑三：收到 `detached` 就当安装失败

`detached` 只是“不再跟这条连接了”，不是 verdict。

### 坑四：只看 `skill.status`，完全忽略 progress event

那你会丢掉 `sandbox_ready / seeding / removed / failed(log)` 这些用户真正关心的细节。
