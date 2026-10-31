# 技能 Snapshot Commit 与 Pointer Switch 阅读指南

这一篇接着上一轮的 [21-code-reading-skill-snapshot-ledger-prune-reconcile.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/21-code-reading-skill-snapshot-ledger-prune-reconcile.md) 往前读，专门看快照链是怎么被“写出来”的。

换句话说，`21` 号文档回答的是：

- reaper 后来怎么根据 ledger 和 live pointer 判断事实

这一篇回答的是：

- install/remove 当时是怎么把这份事实写进 ledger 和 config 的

## 推荐阅读顺序

1. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L620-L706)
2. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1700-L1848)
3. [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L150-L281)
4. [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L365-L430)
5. [21-code-reading-skill-snapshot-ledger-prune-reconcile.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/21-code-reading-skill-snapshot-ledger-prune-reconcile.md)

## 1. 先记住这条时序

无论是 install 还是 remove，涉及技能镜像变更时都遵循同一条主链：

1. 在 ledger 里先写一条 `building` row
2. 去 provider 创建真实 snapshot
3. 拿到 provider 返回的 snapshot id 后，把 ledger row 改成 `active`
4. 调 `switchImagePointer` 把 config 的 live pointer 切到新 snapshot
5. 把之前仍是 `active` 的旧快照标成 `superseded`
6. 最后再收口 skill row、progress 和 stale sandboxes

这个顺序不是实现巧合，而是故意设计的保护带。

## 2. 为什么 ledger row 必须先于 provider snapshot

在 [runInstall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L620-L706) 和 [runRemove](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L232-L281) 里，都能看到这一步：

- 先算 `generation`
- 先生成 `snapshotName`
- 先 `CreateSnapshotRow(... StateBuilding, PlannedName=...)`
- 然后才 `createSnapshot(...)`

原因很直接：

- provider snapshot 是真实资产，会计费
- 如果先 commit，后写 ledger
- 进程死在中间时，就会留下“provider 上存在，但系统里没人知道它是谁”的孤儿快照

而先写 `building + PlannedName` 至少能保证：

- 这轮在造什么快照，ledger 里有记录
- 后面的 `reapAbandonedBuilds` 还有机会凭 `PlannedName` 把它找回来

## 3. 为什么 generation 要取 `max(live pointer, ledger) + 1`

这一点在 [nextSnapshotGeneration](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1842-L1848) 上最容易看出来。

它不是简单做：

- `currentGeneration + 1`

而是同时考虑：

- config 当前 live pointer 的 generation
- ledger 里所有历史 row 的最大 generation

这是在防一个经典崩溃窗口：

- 上一轮已经把 snapshot commit 出来了
- 但还没来得及切 pointer
- 于是 live pointer 看起来还是旧 generation

如果下一轮只按 live pointer 递增，就可能复用同一个 generation，进一步复用同一个 planned name。那后面的 reaper 在按名字清理 abandoned build 时，就可能误伤后一轮真正的新快照。

## 4. `snapshotName` 为什么要带 row id

[skillSnapshotBuildName](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1859-L1866) 的命名规则不是只靠：

- tenant
- config
- generation

还会再带一个 row token，也就是 ledger row id 的压缩片段。

这是为了防止：

- 同 generation 下出现多条 building row
- 或历史异常状态导致名字撞车

只看 generation 不足以把一次 commit 唯一标识出来，尤其是在崩溃恢复路径里。

## 5. `createSnapshot` 只是 provider commit，不负责“让系统可见”

[createSnapshot](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1700-L1717) 做的事情很窄：

- 调 provider 的 `CreateSnapshot`
- 确认返回了非空 snapshot id

它并不意味着：

- 当前 skill 已经安装完成
- 新 session 已经会从这个 snapshot 启动

因为这时 live pointer 还没切过去。

所以 provider commit 成功之后，系统还得继续完成两件事：

1. 把 ledger row 从 `building` 改到 `active`
2. 调 `switchImagePointer` 完成 live image 切换

## 6. 为什么 `MarkSnapshotState(active)` 失败时要立刻 `abandonSnapshot`

这一步在 install/remove 两条路径里都是同样处理。

场景是：

- provider 已经返回了 snapshot id
- 但这次 snapshot id 没能成功写回 ledger

此时最危险的地方在于：

- snapshot 真的存在
- 但除了当前函数局部变量，没有任何地方记住了这个 id

这相当于一张已经会计费、却再也找不回来的 provider 快照。

所以这里必须立刻走 [abandonSnapshot](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1740-L1749)：

- best effort 删除 provider snapshot
- 同时把 ledger row 标成 deleted

这也是正常 install/remove 流程里极少数会主动删“刚创建的新 snapshot”的地方。

## 7. `switchImagePointer` 为什么是整个链路的真源写入

[switchImagePointer](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1750-L1787) 可以看作这条链的分水岭。

在它成功之前：

- 新 snapshot 只是 provider 上一份已存在但还未生效的候选镜像

在它成功之后：

- 所有未来新建 session 都会从这张 snapshot 启动
- 后面的工作都变成“收尾”，而不是“是否生效”

所以代码里才会有那句很重要的语义：

- `One DB write; everything after this is cleanup.`

## 8. 为什么 `switchImagePointer` 必须重新读 config

它不能直接复用 `runInstall` / `runRemove` 开头拿到的 `cfgEntity`，因为这中间可能已经过了几分钟，尤其 install 还夹着一轮完整的 installer agent 对话。

如果直接把旧实体写回去，会把期间管理员修改过的内容一并覆盖掉，比如：

- sandbox 名称
- 描述
- 凭据
- 其他 config blob 字段

因此它必须：

1. 重新 `GetByID`
2. 只更新 `SkillImage`
3. 其余字段以最新读到的 config 为准

## 9. 为什么 `builtFingerprint` 不匹配时要拒绝切针

`switchImagePointer` 还会校验：

- 这张 snapshot 当时是用哪个 provider 账号构建出来的
- 现在 config 上 live credentials 对应的是不是同一个账号

如果这两者不一致，说明：

- 运行期间发生了 credential rotation
- 新 snapshot 属于旧账号
- 当前 live credentials 根本解析不了它

这时如果还强行切针，就会把一个“当前账号根本 boot 不起来的 snapshot id”写成 live image，结果是：

- session 层只能退回 base template
- 但 skill row 可能还显示 ready

所以这里宁可失败，也不能把 pointer 写成伪真值。

## 10. 为什么 `markPreviousSnapshotsSuperseded` 必须放在切针之后

[markPreviousSnapshotsSuperseded](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1807-L1823) 的职责是：

- 把旧的 active rows 退休成 superseded

但这件事只能在新 pointer 已经落稳之后做。

因为如果顺序反过来：

- 旧快照先被标成 superseded
- 新 pointer 又切失败

系统就会把真正的 live image 提前降级，后面的 prune 甚至可能开始把它当退休快照处理。

所以“先切针，再退休旧快照”是必须保持的时序。

## 11. 为什么 remove 的最后一个技能分支不再创建空 snapshot

在 [clearImagePointer](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L365-L394) 里，可以看到最后一个技能被删掉时走的是另一条快路径：

- 不再启动 sandbox 去构造一张“空技能镜像”
- 直接把 config 指回 base template

因为按系统约束：

- base template 天生不带任何 skill
- 当 config 下已经没有其他 skill，需要表达的事实就是“回到无技能状态”

这时再造一张空 snapshot 没有增量价值，反而只会多一张新的 provider 资产。

## 12. 为什么 `clearImagePointer` 也要继续推进 generation

虽然它没有新的 snapshot id，但仍然会更新：

- `Generation`
- `BuiltAt`
- `BaseTemplateID`
- `OwnerFingerprint`

原因是这次 remove 仍然是快照链上的一次真实状态变更，只不过它的结果不是“指向另一张 snapshot”，而是“显式回到 base template”。

保留 generation 的连续性，后面的 ledger / reaper / 人工排查才能看明白：

- 到底是哪一代把镜像清空了

## 13. `finishRemoval` 为什么必须等到 imageChanged 之后

[finishRemoval](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L396-L430) 不是 remove 一开始就能做的。

它只能在这件事已经成立时运行：

- live image 确实不再携带这个技能

这时删除 skill row 才安全。否则 row 先删掉而 image 里还留着技能，会让系统进入一种更难解释的状态：

- 技能文件还在镜像里
- 但数据库里已经没有这条技能记录

所以 remove 的真实“点火成功”不是 row 进入 removing，而是 imageChanged 已经发生。

## 14. 这一篇和前后文怎么接

建议按下面顺序连读：

1. [18-code-reading-skill-install-transcript-guidance.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/18-code-reading-skill-install-transcript-guidance.md)
2. [19-code-reading-skill-progress-state-transitions.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/19-code-reading-skill-progress-state-transitions.md)
3. [20-code-reading-skill-stop-reaper-recovery.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/20-code-reading-skill-stop-reaper-recovery.md)
4. [21-code-reading-skill-snapshot-ledger-prune-reconcile.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/21-code-reading-skill-snapshot-ledger-prune-reconcile.md)
5. 当前这篇 `22`

这样能把技能系统整条链串起来：

- 安装对话怎么跑
- 进度和 durable row 怎么协同
- 卡住后怎么恢复
- reaper 后来怎么判断 live image
- install/remove 一开始又是怎么把 snapshot ledger 和 live pointer 写出来的

## 15. 二开时最容易踩的坑

### 坑一：provider snapshot 创建成功就当安装完成

不对。没切 pointer 之前，它还不是 live image。

### 坑二：先切旧快照状态，再切新 pointer

顺序反了会把真实 live image 提前退休。

### 坑三：最后一个技能删除时也坚持生成一张空 snapshot

这只会制造新的 provider 资产，没有额外业务价值。

### 坑四：generation 只按 live pointer 自增

这样会忽略 ledger 中的 abandoned build，给 planned name 冲突和误删埋雷。
