# 技能 Served Version、Catalog Archive 复用与 Ready 切换阅读指南

这一篇接着 [27-code-reading-skill-bundle-parse-guardrails.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/27-code-reading-skill-bundle-parse-guardrails.md) 往后读，专门把技能安装里最容易混掉的三层事实拆开：

- row 当前描述的是哪一版
- live image 现在真正还在服务哪一版
- workspace catalog 当前把哪份 archive 当成定义层真源

这三者在“升级进行中”或“升级失败后”并不总是同一个东西。

## 推荐阅读顺序

1. [tenant_skill.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/tenant_skill.go#L42-L179)
2. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L120-L390)
3. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1495-L1588)
4. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1372-L1408)
5. [tenant_skill_catalog.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L312-L575)

## 1. 先分清三种“当前版本”

技能升级过程中，至少有三种“当前版本”：

### 第一种：row 上层字段当前写的版本

也就是 `TenantSkillEntity.Version / Description / Instructions / BundleSHA256` 这些字段。

### 第二种：live image 现在真正还在服务的版本

也就是 session / agent 运行时真正能调用到的那版 skill。

### 第三种：workspace catalog 当前把哪份 archive 视为定义层最新版本

也就是 catalog row 上的 `BundleRef / BundleSHA256 / Version`。

平时三者常常一致，所以很容易被误当成同一个概念。真正复杂的是升级途中：

- row 先切到新安装尝试
- image 还没切过去
- catalog 可能已经被新 archive 替换

这时三者会故意分离。

## 2. 为什么 `ServedView` 是 agent 侧看到的真相

[ServedView](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/types/tenant_skill.go#L120-L146) 的作用，是把 install row 投影成：

- agent 运行时当前应该看到哪一版

规则很克制：

- row 是 `ready`，就直接看 row
- row 是 `installing/failed`，但 `Served` 还在，就看 `Served`
- 否则什么也不服务

也就是说，agent 根本不需要知道“这是不是升级中的 skill”。它只需要看到一份已经被投影成 ready 的视图。

## 3. 为什么 row 一开始就会切到新版本字段

[takeSkillRowForInstall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L370-L381) 会在 install 刚开始时，就把 row 上层字段改成：

- 新 `Version`
- 新 `Description`
- 新 `Instructions`
- 新 `BundleSHA256`
- `Status=installing`

这意味着 row 很早就开始描述“最新尝试”，而不是“当前 image”。

这样做的好处是：

- 后续进度、错误、引导、安装 transcript 都围绕当前 attempt 展开

但代价是：

- 运行时真相不能再直接看 row 本体

这也是 `Served` 字段存在的根本原因。

## 4. `servedVersionOf` 为什么只在接管 row 的那一刻转抄一次

[servedVersionOf](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L294-L316) 会在 row 从 `ready` 切到 `installing` 时，把旧版转存到 `e.Served`。

它的关键点是：

- ready row 才能代表当前 image
- 已经处于 `installing/failed` 的 row，继续沿用先前记录的 `Served`

这条规则非常重要，因为否则：

- 重试中的 upgrade 可能会把“上一轮失败 attempt”误当成 served 真相

项目这里明确区分：

- 当前 live image 里真的还在跑的旧版
- 最近一次没成功的新尝试

## 5. 为什么 `servedBundleRef` 要把 archive object 也一起钉住

[servedBundleRef](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L318-L337) 看起来只是个小辅助函数，但它解决的是一个很实在的问题：

- row 可能并不自己持有 `BundleRef`
- 它可能只是通过 `CatalogID` 跟随 definition 读 archive

如果这时 catalog 被重注册到新 archive，而旧版仍在服务：

- `read_skill`
- 文件浏览
- 调试用的资源读取

都会突然找不到旧版 bytes。

所以 served 版本在被记录下来时，不只保存 digest，还尽量保存：

- **它当时读的具体 object ref**

## 6. 为什么 `restoreServedVersion` 是“承认指针根本没切过去”

[restoreServedVersion](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L339-L359) 不是普通失败回滚。

它只用于那些最终确认：

- live image pointer 根本没有切到新版本

的路径。

这时正确做法不是让 row 留在“新 attempt 失败，但还带着新 bundle 的描述”状态，而是：

- 直接把旧 served 版本还原回 row 本体

这样 row、served truth 和 archive pin 就重新归一了。

## 7. 为什么 `canSkipInstall` 只在“同 digest 且 live image 里真的有”时才成立

[canSkipInstall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1546-L1573) 不是简单比较 `BundleSHA256`。

它还要看：

- row 当前是不是 `ready`
- 或者 install heartbeat 是否仍在活着
- 尤其是 `ready` 时，`skillFilesInLiveImage` 是否确认这版 bytes 真的还在 live chain 上

原因是 row 上层字段可能已经先被新 install 改过了，单看 digest 很容易把：

- “row 当前写的是 B，但 image 里还跑着 A”

误判成：

- “B 已经装好了，可以跳过”

所以 skip 的真条件是：

- **同一份 bytes 已经真实存在于 live image**

## 8. `installStillOwnsTheRow` 为什么是并发 install 的最后一道闸门

[installStillOwnsTheRow](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1515-L1544) 处理的是 install 开始后那些慢路径里的竞态：

- remove 已经接管
- StopSkill 改成 failed
- 更晚的 upload 把 `BundleSHA256` 换掉了
- 同 digest 的另一条 run 已经真正装进 live image

只要命中这些情况，当前 run 就必须立刻放弃，因为继续 snapshot 会造成：

- row 已经不再属于它
- ledger 却多出一份没人能正确认领的 snapshot

这条检查的本质不是“防重复工作”，而是“防孤儿快照和错误终态写回”。

## 9. 为什么 `skillArchiveOrigin` 要单独区分 uploaded 和 stored

[skillArchiveOrigin](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L58-L69) 看起来只是两个枚举值，但它解决的是 catalog 回滚问题。

它区分：

### `uploaded`

- 新上传
- 新来源拉取
- 本质上是在定义层引入新版本

### `stored`

- 重试已有 archive
- 从 catalog 向某个 sandbox 复制安装
- 本质上不应改写 workspace definition

如果不区分这两类，最常见的坏结果就是：

- catalog 已经到 v2
- 某个还在 v1 的 sandbox 做一次重试
- 结果把 definition 又写回 v1

## 10. `catalogForArchive` 真正回答的是什么

[catalogForArchive](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L214-L233) 回答的不是：

- 这个 skill 叫什么

而是：

- 这份 archive 在 definition 层应该跟随谁

它的策略是：

- `stored` 且 workspace 已经有同 digest catalog -> 直接复用
- 否则走 `upsertCatalogFromBundle`

所以 definition 的推进只和“是不是新定义的 bytes”有关，不和“哪个 sandbox 此刻在装它”绑定。

## 11. 为什么 `storedArchiveKeepsItsPin` 是防止 catalog 回滚的关键

[storedArchiveKeepsItsPin](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L255-L291) 解决的正是上面的回滚窗口。

它会在这种情况下返回 `true`：

- workspace catalog 已经前进到新 digest
- 当前 sandbox row 仍然 pin 着旧 digest
- 这次操作只是重放旧 archive

这意味着：

- row 继续保留自己的旧 archive pin
- catalog 不被改写

这条规则看似绕，但它保证了：

- sandbox 的重试是 sandbox 级事件
- definition 的前进是 workspace 级事件

两者不会互相踩踏。

## 12. 为什么 `pointInstallAtCatalog` 要主动清掉 row 的 `BundleRef`

[pointInstallAtCatalog](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1372-L1396) 会把 install row 改成：

- `CatalogID = catalog.ID`
- `BundleRef = ""`

这一步的意义是把“谁拥有 archive object”讲清楚：

- install row 不再拥有 archive
- definition 才拥有 archive

这样后面做 uninstall 时就不会因为删了某个 sandbox 的 skill，就把整个工作区定义层的 archive 一起删掉。

## 13. 为什么 catalog 替换 archive 时不能立刻删旧 object

在 [storeCatalogBundle](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L396-L433) 之后，旧 archive 还不能立刻删。

因为此时可能还有两类读者：

1. install row 自己 pin 着旧 object
2. upgrade 中的 `Served` 版本仍然在读旧 object

所以 catalog archive 的替换顺序必须是：

1. 先写新 object
2. 先给旧读者补 pin
3. 再提交 catalog row
4. 确认没人需要旧 object 后再回收

## 14. `pinReplacedCatalogBundle` 在补什么 pin

[pinReplacedCatalogBundle](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L455-L469) 补的是：

- definition 刚换下来的旧 archive object

它要把这个 object 显式交给那些仍然在服务旧版的 installs。

如果不做这一步，就会出现一种很隐蔽的问题：

- row 上的 `BundleSHA256` 还是旧版
- 但真正解析 archive 时，由于 row 没有 `BundleRef`，会顺着 `CatalogID` 读到新版 object

这会让“served 旧版”在文件读取层突然读到一份新版 zip。

## 15. 为什么 `pinServedVersionToReplacedBundle` 要特别照顾 `e.Served`

[pinServedVersionToReplacedBundle](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L523-L546) 处理的是最容易被忽略的一种读者：

- row 本体已经切到 installing
- 但 `e.Served` 仍代表 live image 里的旧版

而且老数据里，`e.Served` 可能只有：

- `BundleSHA256`

没有：

- `BundleRef`

这时就必须在 catalog 发生替换时，趁机把旧 object 补 pin 到 `e.Served` 上，否则它之后就失去对旧 bytes 的稳定引用。

## 16. `catalogBundleArchive` 为什么还要到处兜底找 archive

[catalogBundleArchive](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L654-L704) 会按好几层兜底去找 archive：

1. catalog 自己的 `BundleRef`
2. 直接挂着 `CatalogID` 的 installs
3. 同名或历史兼容 row

这说明 archive 持有关系经历过演进：

- 新模型是 definition 持有 archive
- 旧模型里 install row 也可能自己 pin 过 object

所以读取路径必须兼容这几种历史状态，不能假设“一定只有 catalog 持有”。

## 17. 为什么 catalog 列表里的 `Served` 只是提示，不是另一套真源

在 [tenant_skill_catalog.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L17-L46) 和 `ServedInfoOf` 里，catalog 视图会把：

- “row 当前在升级中，但旧版仍在服务”

投影成一个简单的 `ServedInfo`。

这只是控制台可读性的提示层，不是新的真源。

真正决定运行时可见性的仍然是：

- `ServedView`
- live image
- archive pins

Catalog 这里只是把这种状态说明给管理员看。

## 18. 这篇和前后文怎么接

建议按下面顺序连读：

1. [26-code-reading-skill-source-fetch-parse.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/26-code-reading-skill-source-fetch-parse.md)
2. [27-code-reading-skill-bundle-parse-guardrails.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/27-code-reading-skill-bundle-parse-guardrails.md)
3. 当前这篇 `28`
4. [22-code-reading-skill-snapshot-commit-pointer-switch.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/22-code-reading-skill-snapshot-commit-pointer-switch.md)

这样能把安装链前后半段接起来：

- source 怎么变成 bundle
- bundle 怎么变成标准 archive
- archive 怎么在 served version / catalog / ready row 之间安全切换
- 最后 snapshot / pointer 又怎样把新版本真正变成 live image

## 19. 二开时最容易踩的坑

### 坑一：把 row 上层字段直接当成运行时真相

升级中的 row 早就切到新尝试了，live image 可能还在跑旧版。

### 坑二：重试旧 archive 时顺手把 catalog 也写回旧版

这会把 workspace definition 倒退。

### 坑三：catalog 替换 archive 后立刻删旧 object

仍在服务旧版的 installs 和 `e.Served` 可能马上读不到文件。

### 坑四：只给 row 本体补 pin，忘了给 `e.Served` 补 pin

中途 upgrade 的旧版读者并不总在 row 本体上。

### 坑五：skip install 只比较 digest，不确认 live image 里真的有它

这样很容易把“row 先写成新版本，但 image 还没切过去”的状态误判成安装成功。
