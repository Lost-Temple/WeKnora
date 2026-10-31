# 技能 Snapshot Ledger、Prune 与 Reconcile 阅读指南

这篇文档把技能安装恢复链里更底层的一层单独拎出来：`snapshot ledger / live image 判断 / superseded prune / reconcile 告警 / abandoned build 回收`。

如果说上一篇 [20-code-reading-skill-stop-reaper-recovery.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/20-code-reading-skill-stop-reaper-recovery.md) 解释的是“卡住后怎么恢复 skill row”，这一篇讲的就是：

1. reaper 凭什么判断 live image 里还带不带某个技能
2. 哪些 provider snapshot 允许删，哪些只能告警不能删
3. 为什么还需要一条专门处理 abandoned build 的回收路径

## 推荐阅读顺序

1. [tenant_skill_reaper.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go)
2. [20-code-reading-skill-stop-reaper-recovery.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/20-code-reading-skill-stop-reaper-recovery.md)
3. [19-code-reading-skill-progress-state-transitions.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/19-code-reading-skill-progress-state-transitions.md)

建议先把 `StopSkill / ReapStuckRuns` 的恢复语义看懂，再回到这一篇看它们依赖的快照事实判断。

## 1. 先建立一个正确心智模型

一套 sandbox config 在任意时刻只有一个 live pointer，也就是当前给新 session 启动的 `SkillImage`。

但技能安装不是“每个 skill 各管各的 snapshot”，而是：

- 任意一次 install/remove 都会推进整套 config 的快照链
- 新快照是从当前 live snapshot 长出来的
- 其他技能后续继续安装时，并不会回写旧 skill row 的 `InstalledSnapshotID`

所以不能问：

- `row.InstalledSnapshotID == currentSnapshotID ?`

真正该问的是：

- **当前 live pointer 沿着 snapshot ledger parent chain 往回追，最近一次提到这个 skill 的节点是什么**

这就是 [skillFilesInLiveImage](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go) 的职责。

## 2. `skillFilesInLiveImage` 为什么一定要走 parent chain

`skillFilesInLiveImage` 的返回值可以理解成三件事：

- `snapshotID`：这条 live chain 上最近一次决定该技能状态的快照
- `serving`：当前 live image 是否仍然带着这个技能
- `known`：这个结论是否可信

这里最容易误判的地方是把它写成“比较两个 snapshot id 是否相等”。

那样会错判两类情况：

- 一个 skill 是两代之前装进去的，但后面其他 skill 又继续推进了 snapshot 链，此时它仍然在 live image 里
- 一次 stuck remove 其实还没真正删掉技能文件，但 row 上的 `InstalledSnapshotID` 早已不是 live pointer

因此这里必须：

1. 先读当前 config 的 live pointer
2. 再读该 config 的 snapshot ledger
3. 从 live snapshot 沿 `ParentSnapshotID` 一路往回走
4. 找到最近一次提到该 skill 的 install/remove 节点

结论也正是靠这个最近节点来定：

- 最近节点是 `install`，说明 live image 里还带着它
- 最近节点是 `remove`，说明 live image 已经把它拿掉
- 链读不全时，`known=false`，调用方必须拒绝做不可逆动作

## 3. `liveSnapshotID` 只是读指针，不负责解释事实

[liveSnapshotID](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go) 做的事情很克制：

- 只从 config 里读当前 `SkillImage` pointer
- 告诉调用方这次读取是否可信

它不尝试回答“这个 skill 是否还在镜像里”。因为那个问题本质上不是配置读取问题，而是快照链追溯问题。

## 4. `ReconcileSnapshots` 为什么只告警，不删除

[ReconcileSnapshots](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go) 做的是：

- 列出 provider 上当前可见的 snapshots
- 过滤掉明显属于其他 config 的命名空间
- 再和 ledger 里认账的 snapshot 集合做比对

如果 provider 上出现了 ledger 不认识的 snapshot，它只会：

- `logger.Warnf(...)`

不会调用 delete。

原因不是“以后再说”，而是这就是它的职责边界：

- provider 账号可能是共享的
- 这里的 extra 很可能是别的环境的 live image
- reconcile 是诊断路径，不是清理路径

所以这一步的意义是：

- 帮你发现 ledger 与 provider 之间的异常漂移
- 但不擅自做不可逆清理

## 5. 真正删除 provider snapshot 的是 `PruneSupersededSnapshots`

[PruneSupersededSnapshots](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go) 才会真正去调 provider delete。

它的删除前提很严格：

- 这条 snapshot 必须是 ledger 里认得的
- 不能是当前 live pointer
- 状态必须已经退休，或者属于崩溃窗口遗留的非 live active row
- 年龄已经超过 retention window

也就是说，prune 清理的是：

- **ledger 已退休、超过保留期、并且确认不是 live image 的 provider snapshot**

它明确不会碰两类对象：

- live pointer 当前正在指向的快照
- ledger 没记账的 extra snapshot

前者是当前业务真值，后者属于共享 provider 账号上的未知资产，不能贸然删。

## 6. retention 为什么不能只看一个固定常量

[snapshotRetentionFor](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go) 的逻辑很值得注意。

它不是简单返回一个统一常量，而是取：

- 全局 `snapshotRetentionWindow`
- 当前 config 的 sandbox TTL + 一个安全边际

中的较大值。

这样做是为了防止：

- 新 live pointer 已经切走
- 但旧 session 还可能基于上一代模板活着
- 结果 provider 上的旧 snapshot 被提前删掉

不同 provider 的 TTL 口径还不完全一样：

- Cube / E2B 看显式 sandbox TTL
- Docker 没有独立 sandbox 生命周期，只能退回 idle TTL

## 7. `pruneConfigSnapshots` 先过 `ensureUsableImage` 是在防什么

这一步防的是 credential 轮换后的“错账号删除”问题。

如果 config 已经切到了另一个 provider 账号：

- ledger 里老的 snapshot ID 在新账号上根本不存在
- delete 返回的可能只是 `not found`

如果这时把 `not found` 直接当成“老快照已经清掉”，ledger 就会把状态记成 deleted，但真正计费的旧账号还保留着那些快照。

所以 `ensureUsableImage` 在这里的作用是：

- 一旦 provider 身份已经不可信，就先跳过 prune
- 宁可晚一点清，也不能错误地把 ledger 和 provider 真相彻底对离

## 8. 为什么还要单独有 `reapAbandonedBuilds`

`reapAbandonedBuilds` 处理的是一个非常具体的崩溃窗口：

1. ledger 先写入一条 `building` row
2. provider commit 成功，真实 snapshot 已经生成
3. 进程在把 `SnapshotID` 写回 ledger 之前崩溃

这时会出现一个麻烦的事实：

- provider 上真的有个会计费的 snapshot
- 但 ledger 里没有任何 snapshot ID 能指向它

这导致常规路径都救不到：

- `PruneSupersededSnapshots` 不会删，因为 row 还是 `building` 且 `SnapshotID` 为空
- `ReconcileSnapshots` 只会把它记成 extra，不会删
- config 删除路径看到空 `SnapshotID` 也没法 release

所以只能靠 `PlannedName` 这条在 commit 前就写下来的线索去认领。

## 9. 为什么 abandoned build 只能“正匹配后再删除”

[matchSnapshotByName](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go) 的策略很保守：

- 只有 provider 返回的 `ID` 或 `Names` 能正向匹配 `PlannedName`
- 才会把它认成那条 abandoned build 对应的 snapshot

查不到时不会做任何“推断式删除”。

因为 provider 不返回匹配结果，可能有两种完全不同的原因：

- 这次 commit 确实没成功
- provider 压根不回显 name，或者返回格式与预期不同

如果在这种不确定状态下就把 row 标成 deleted，相当于把最后一条线索也抹掉了。

## 10. `buildStillRunning` 和 `configHasInFlightSkill` 在保护什么

这两个函数是在给 abandoned build 回收加双保险。

### `buildStillRunning`

它不会只看 row 年龄，而是还会回头看 skill row 上的 `InstallingSince` 心跳。

保护的是这种情况：

- provider 很慢
- 大镜像真的还在 commit
- 只是耗时超过了平时经验值

如果只按时间阈值清理，最坏会删掉一条马上就要成功的真实 snapshot。

### `configHasInFlightSkill`

只要同一套 config 里还有别的 `installing/removing`，就先不动 abandoned build。

因为这说明：

- 当前这条快照链还在推进
- 旧快照的清理可能和新一轮 install/remove 互相干扰

所以它宁可延后一个 sweep，也不抢跑做不可逆删除。

## 11. `snapshotEligibleForPrune` 是 prune 的最后一道 ledger 侧闸门

这个函数统一回答：

- 这条 ledger row 有没有资格去尝试 provider delete

它会排掉几类绝对不能动的对象：

- 没有 `SnapshotID` 的 row
- 当前 live pointer 指向的 snapshot
- 非 `superseded` / `active` 的状态
- 年龄还没超过保留窗口的 row

这里把 `active` 也纳入候选，不是常态，而是在兼容崩溃窗口：

- pointer 已经切换走
- 但 `markPreviousSnapshotsSuperseded` 还没来得及把旧 row 改成 superseded

这类 row 虽然状态还是 active，但已经不是 live image，也仍然是计费残留。

## 12. 后台 housekeeping 的完整节奏

[runSkillReaper](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go) 把后台维护收成了一条固定顺序：

1. `ReapStuckRuns`
2. `PruneSupersededSnapshots`
3. `reconcileAllSnapshots`

这个顺序也有含义：

- 先把 stuck row 修到和 live image 一致
- 再删已经退休且超过窗口的 provider snapshot
- 最后把 provider 与 ledger 的残余漂移记到告警里

所以它不是三个孤立任务，而是一条“先修真值，再回收，再诊断”的 housekeeping 链。

## 13. 这一篇和前后文怎么连

建议按下面顺序连续读：

1. [19-code-reading-skill-progress-state-transitions.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/19-code-reading-skill-progress-state-transitions.md)
2. [20-code-reading-skill-stop-reaper-recovery.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/20-code-reading-skill-stop-reaper-recovery.md)
3. 当前这篇 `21`

这样能形成一条完整理解链：

- 前端看到的 install/remove progress 如何流动
- skill row 卡住后 stop/reaper 如何恢复
- reaper 又是凭什么判断 live image、prune 旧快照、diagnose provider extras

## 14. 二开时最容易踩的坑

### 坑一：把 `InstalledSnapshotID` 当成“这个 skill 当前是否还在 live image”的唯一证据

不对。它只记录这条 skill row 上次成功安装时的快照，不会随着别的 skill 安装被回写。

### 坑二：在 reconcile 里顺手把 extra snapshots 删掉

不行。那样会把共享 provider 账号里别的环境的 live image 一起删掉。

### 坑三：只做 superseded prune，不处理 abandoned build

这样会留下“provider 在计费，但 ledger 没人认领”的真孤儿快照。

### 坑四：abandoned build 查不到名字时，直接把 row 记成 deleted

这会抹掉最后的追踪线索，让 still-live 但不可见的 provider snapshot 永远无人认领。
