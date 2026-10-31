# 技能 Remove、Served 回落与 Archive 回收阅读指南

这一篇接着 [28-code-reading-skill-served-version-catalog-reuse.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/28-code-reading-skill-served-version-catalog-reuse.md) 往后读，专门讲技能删除这条链里最容易混掉的两件事：

- 技能什么时候才算真的从 live image 里消失
- archive object 又是什么时候才能真的物理删除

这两件事在实现里是刻意分开的。

## 推荐阅读顺序

1. [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L1-L283)
2. [tenant_skill_remove.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L304-L590)
3. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L310-L381)
4. [tenant_skill_catalog.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L455-L560)

## 1. Remove 删的首先不是 row，而是 image

从入口 [RemoveSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L17-L56) 就能看出来，删除技能不是“数据库删一行”这么简单。

真正要做的是：

- 把技能目录从 image 里拿掉
- 写一张新的 snapshot，或者回退到 base template
- 让未来 session 不再从带这个技能的 image 启动

所以 row 删除只是最后一步，而且必须放到 image 已经发生变化之后。

## 2. 为什么 remove 也要拿 config 级串行锁

install 和 remove 本质上都在做同一类事情：

- 从当前 image 派生一代新 image
- 再把 config 的 live pointer 切过去

如果两者并发：

- 一个删了目录
- 另一个装了新文件

最后谁先写 pointer，谁就会把另一方的改动整块覆盖掉。

所以 remove 和 install 共用同一把 per-config lock，不是实现巧合，而是同一个资源竞争面。

## 3. `runRemove` 真正分成哪几段

[runRemove](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L58-L283) 大致可以分成四段：

1. 确认当前 run 仍拥有这条 row
2. 在临时 sandbox 里删除技能目录与 manifest
3. 把 image 变化提交成新 snapshot，或者直接清 pointer
4. 做 row 删除、archive 释放和 stale sandbox 标记

这四段分别对应：

- 运行权确认
- 文件层变化
- image 层变化
- 控制面与存储层收口

## 4. 为什么 `removeStillOwnsTheRow` 和 install 那条 guarding 逻辑同样重要

删除也是异步 run，也会和更新/重试/stop 并发。

[removeStillOwnsTheRow](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L530-L543) 只要发现：

- row 已经不再是 `removing`

就必须立刻放弃。

因为这通常意味着：

- 同名 skill 的新 install 已经接管了 row

如果还继续删目录、删 row，就会直接把那次新 install 的结果抹掉。

## 5. 为什么“最后一个技能删除”要走 `clearImagePointer`

如果删掉这条 skill 后 config 已经没有任何技能要继续携带，[clearImagePointer](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L372-L404) 会直接：

- 把 `SkillImage.SnapshotID` 清空
- 指回 base template
- 继续推进 generation

它不会再额外做一张“空技能 image”。

原因很直接：

- base template 天然就不带这些技能
- 再做一张空 snapshot 只会多一张 billable 资源

所以这是一条很刻意的快路径。

## 6. 为什么 `countRemainingSkills` 连 failed/installing rows 也算

[countRemainingSkills](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L353-L370) 不是只数 ready rows。

因为：

- 某条 row 虽然现在是 failed
- 但它的 `Served` 旧版可能仍然在 image 里

如果此时把“不是 ready 的 skill 都当成不存在”，就可能在删除另一个 skill 时误判：

- “这是最后一个技能了，可以直接退回 base template”

结果把 image 里仍在服务的旧版技能一起带没了。

## 7. `removeManifestEntry` 为什么失败也不终止 removal

[removeManifestEntry](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L304-L351) 的注释已经把边界说得很清楚：

- manifest 只是镜像内自述
- 不是运行时真源

所以 manifest 读不出来或 JSON 坏了，最多是：

- 失去一层排查辅助

并不等于：

- 这个技能实际上删不掉

如果因为 manifest 坏了就让整个 removal 失败，等于把“调试元数据不完整”和“真实 image 无法删除”混成了一件事。

## 8. 为什么 `finishRemoval` 要在 row 删除前先记下 pinned archive refs

[finishRemoval](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L406-L452) 先做了一步很关键的读：

- `installBundleRefs(row)`

然后才删 row。

原因是 row 一旦被删掉，这条 install 当前通过哪些 archive refs 还在读文件，就再也没人能告诉你了。

而 archive 回收恰恰依赖这份信息。

所以这里的顺序必须是：

1. 先记下 row 仍引用的 refs
2. 再删 row
3. 再尝试释放这些 refs

## 9. 为什么 row 删除和 archive 删除要分开

这条设计是整个 remove 链里最重要的一点。

从语义上看：

- row 删除回答的是“这个 sandbox config 还装没装这项技能”
- archive 删除回答的是“工作区里还有没有任何读者需要这份 bytes”

这两个问题根本不是同一个维度。

一个 archive 可能仍被这些地方读着：

- catalog definition
- 另一条 install row 的 `BundleRef`
- 某条 upgrade 中 `e.Served.BundleRef`

所以 row 删掉后，只能说：

- 这条安装关系结束了

不能直接推出：

- 这份 archive 从世界上消失了

## 10. `restoreSkillAfterFailedRemoval` 为什么有时回 ready，有时回 failed

[restoreSkillAfterFailedRemoval](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L455-L491) 的策略很克制：

### 如果 skill 曾经真的进过 image

- 回 ready，或者还原成 served 旧版

因为 live image 里实际上还带着它。

### 如果 skill 从未真正进过 image

- 保持 failed

因为此时把它叫 ready，会让 agent 误以为某个 image 里有它的文件。

这就是“row 状态必须服从 image 事实”的一个典型例子。

## 11. 为什么失败 remove 也会调用 `restoreServedVersion`

这和上一轮 upgrade 文档是连着的。

当 remove 失败时，row 可能已经借用了新状态；而真正还在 image 里服务的，仍然是旧版 skill。

所以它会调用 [restoreServedVersion](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L361-L377)：

- 把 row 上层字段还原成原来的 served 事实
- 顺手返回这次失败路径暂时 pin 下来的 archive

这说明 remove 和 install 虽然方向相反，但它们共享同一套“served 版本真相优先于 row 当前尝试”的原则。

## 12. `installBundleRefs` 为什么要同时看 row 本体和 `e.Served`

[installBundleRefs](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L506-L519) 列出一条 install row 当前所有仍具读取语义的 archive refs。

它不只看：

- `row.BundleRef`

还看：

- `row.Served.BundleRef`

这是因为 upgrade 中间态里，真正仍在服务的旧版 archive 往往并不在 row 本体上，而在 `e.Served` 上。

如果 remove 或 catalog replacement 忘了看这层，就会把仍在读的旧 archive 提前删掉。

## 13. `releaseInstallBundle` 为什么要同时检查 catalog 和所有 installs

[releaseInstallBundle](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L537-L573) 的问题不是：

- “这条 row 还引用吗”

而是：

- “这个 tenant 下还有任何地方在引用吗”

所以它会同时扫：

1. 全部 catalog rows
2. 全部 install rows（含 `Served`）

只要还有任何一个读者，就绝不删 object。

这是一种典型的“宁可泄漏一点存储，也不能误删可读对象”的策略。

## 14. 为什么 `deleteBundleBestEffort` 必须是 best-effort

[deleteBundleBestEffort](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L575-L587) 体现的是很明确的风险排序：

- 删不掉，最坏是留一份孤儿 archive
- 删错了，会直接让文件浏览、重试和 served 旧版读取失去依据

所以这里绝不能反过来做成“必须删除成功，否则整个 remove 失败”。

image 变化和 row 删除已经是真正的业务结果，archive 回收只是后续清理动作。

## 15. 为什么 `finishRemoval` 只有在 imageChanged 时才标 stale sandboxes

这一点很细，但很重要。

如果这次 remove 根本没有真正改变 live image，比如：

- row 本来就没进过 image

那去标记 config 的所有 bound sandboxes stale，只会：

- 无意义地重建 session sandbox
- 丢掉 /workspace scratch

所以 `finishRemoval` 只在 `imageChanged=true` 时才做这一步。

## 16. remove 路径和 catalog replacement 路径为什么共用一套 archive 引用语义

你会发现：

- remove 用 `installBundleRefs` / `releaseInstallBundle`
- catalog replacement 用 `pinReplacedCatalogBundle` / `pinServedVersionToReplacedBundle`

这两条链虽然场景不同，但底层问题是同一个：

- 某份 archive 是否还有读者

所以项目把 archive 生命周期的判断统一建在：

- install row 本体的 `BundleRef`
- served 旧版的 `BundleRef`
- catalog definition 的 `BundleRef`

这三类引用上，而不是每条链各自发明一套“猜谁还在读”的规则。

## 17. 这篇和前后文怎么接

建议按下面顺序连读：

1. [28-code-reading-skill-served-version-catalog-reuse.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/28-code-reading-skill-served-version-catalog-reuse.md)
2. 当前这篇 `29`
3. [20-code-reading-skill-stop-reaper-recovery.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/20-code-reading-skill-stop-reaper-recovery.md)
4. [21-code-reading-skill-snapshot-ledger-prune-reconcile.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/21-code-reading-skill-snapshot-ledger-prune-reconcile.md)

这样能从三个角度把删除链看完整：

- upgrade 中 served 真相和 archive pin 怎么维护
- remove 时 image 变化和 archive 回收怎么拆开
- 删除之后 snapshot ledger 和后台回收又怎样继续接手

## 18. 二开时最容易踩的坑

### 坑一：把 remove 等同于删 row

row 删了不代表 live image 变了，更不代表未来 session 不会再从旧 image 启动。

### 坑二：删 row 之后再去想“它刚才引用了哪些 archive”

这时信息已经没了。

### 坑三：只看 row 本体，不看 `e.Served`

升级中的旧版读者经常只活在 `Served` 上。

### 坑四：manifest 删除失败就判整个 remove 失败

manifest 是自述文件，不是运行时真源。

### 坑五：archive 删除失败就把整次 remove 当失败

真正的删除结果是 image 和 row 的状态，不是对象存储清理是否 100% 成功。
