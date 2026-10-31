# Host Skill 本地版本树、Activate 与 Sweep 阅读指南

这一篇接着 [30-code-reading-skill-file-browser-and-skill-read.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/30-code-reading-skill-file-browser-and-skill-read.md) 往后读，专门讲 Lite 模式下 host skill 的本地版本树机制。

这一套实现表面上和远端 `snapshot -> live pointer` 完全不同，但它想解决的其实是同一件事：

- 新版本先在隔离空间里准备好
- 真正确认可用后，再原子切换成后续会话可见版本
- 失败时能回滚
- 崩溃后有后台清理残留

## 推荐阅读顺序

1. [tenant_skill_host.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L18-L320)
2. [host_skills.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/host_skills.go#L5-L39)
3. [tree.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/tree.go#L1-L295)
4. [lock_unix.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/lock_unix.go#L1-L40)
5. [install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/adapter/install.go#L18-L190)
6. [tenant_skill_reaper.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L903-L939)

## 1. host skill 和远端 skill 只是“提交介质”不同

远端 skill install 的提交介质是：

- provider snapshot
- config 上的 `SkillImage` pointer

host skill install 的提交介质是：

- 本地版本目录
- 技能名对应的活动 symlink

所以两者看起来像两套系统，但它们的语义对位其实很整齐：

| 远端 | Host/Lite |
| --- | --- |
| seed 到临时 sandbox | 写到新的版本目录 |
| verify 成功 | verify 成功 |
| 切 `SkillImage` pointer | 切活动 symlink |
| superseded snapshot | 历史版本目录 |
| prune / reaper | `Prune` / `Sweep` |

## 2. `requireSkillTarget` 为什么先把 host target 和普通 config 分开

[requireSkillTarget](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L18-L37) 是所有技能入口前面的第一道分流。

它明确规定：

- Lite 模式只接受 host target
- 标准版绝不接受 host target

这意味着 host skill 不是某个“普通 sandbox type 的特殊值”，而是一条独立部署形态。

这样后面的 install/remove/reaper 才能大胆假设：

- host 路径上没有 provider snapshot
- 普通 config 路径上没有本地版本树

## 3. `runHostInstall` 和远端 install 到哪里开始分叉

[runHostInstall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L51-L136) 的注释已经说明了关键边界：

- 抢 row
- bind run ownership
- heartbeat
- transcript
- verify

这些前半段和远端 install 是共用的。

真正分叉的是提交半段：

### 远端

- build snapshot
- switch image pointer

### host

- `NewVersion`
- 写文件与依赖
- `Activate`

这也是为什么 host skill 不是“另一套业务逻辑”，而是“同一套安装状态机换了一种物理提交方式”。

## 4. 为什么每次 host install 都要新建一个版本目录

[tree.NewVersion](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/tree.go#L105-L121) 每次都会创建：

- `<root>/.versions/<name>-<n>`

而不是复用旧目录。

原因非常现实：

- Python venv 会把绝对路径写进环境
- Node 生态里也常出现基于实际目录的缓存和软链接

如果把旧目录原地覆盖，最容易出现的就是：

- 新 install 看似成功
- 运行时却掺着旧环境残留

所以版本目录必须“只增不改”，这和远端 snapshot 不可变是同一个思路。

## 5. `HostSkillInstaller.Bind` 绑定的到底是什么

[InstallManager.Bind](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/adapter/install.go#L49-L64) 会把一个维护 session 绑定到：

- 这次 install 专属的版本目录

这意味着安装 agent 后面所有能力：

- `shell_exec`
- `write_skill_file`
- `edit_skill_file`
- `ReadSessionFile`
- `WriteSessionFile`

最终都只能落在这个目录里。

所以 Bind 的作用不是“记个上下文”，而是：

- **给一次 host install 建立唯一写入作用域**

release 之后，这个 session 立刻失去本地技能树访问权。

## 6. 为什么 `writeHostSkillFiles` 默认把文件都写成可执行

[writeHostSkillFiles](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L176-L198) 做了两件事：

1. 拒绝任何逃出 skill 根目录的路径
2. 所有文件都写成 `0o755`

第二点看起来激进，但它和远端 seed tar 的语义是一致的：

- skill 里的脚本文件应当开箱可执行

这样安装 agent 后面在 verify 阶段跑脚本时，不需要再逐个补权限位。

## 7. 为什么 host install 也要经过一个“安装专用沙箱”

host skill 并不是主进程直接在宿主机上随意执行安装命令。

它仍然通过：

- [InstallManager](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/adapter/install.go#L18-L44)

向本地策略沙箱发命令。

`ExecShellCommandWithOptions` 会把命令限制在：

- 绑定好的版本目录
- 本地 install policy

而且 [installEnv](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/adapter/install.go#L115-L128) 会强制把：

- `UV_CACHE_DIR`
- `PIP_CACHE_DIR`
- `npm_config_cache`

都指向版本目录下的 `.weknora/cache`

这样安装期产生的临时缓存也跟着版本目录生命周期一起走。

## 8. `Activate` 为什么相当于 host 版的 live pointer 切换

[tree.Activate](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/tree.go#L143-L173) 的做法是：

1. 先创建 `.next-<name>` 临时 symlink
2. 再 `rename` 覆盖 `<root>/<name>`

这一步的重要性非常像远端的 `switchImagePointer`：

- 新版本目录写完不等于已生效
- 真正对未来聊天可见的是活动链接切换成功

只要 `Activate` 还没发生，后续 session 仍然只会看到旧版本目录。

## 9. 为什么 `writeReadySkillState` 失败时还要 `restoreHostSkillLink`

在 [runHostInstall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L108-L123) 里，`Activate` 之后还会写 ready row。

如果 ready 写回失败，就会调用：

- [restoreHostSkillLink](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L254-L262)

把链接切回 previous。

这说明 host install 也坚持同一个原则：

- 控制面 row 和运行时可见版本必须一致

不能出现：

- 本地链接已经切到新版本
- 数据库却还把这项技能记成 installing/failed

## 10. 为什么 prune 只在 ready state 写回后才发生

`runHostInstall` 最后调用：

- [tree.Prune](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/tree.go#L181-L202)

但只会在 ready state 已经稳定写回之后才做。

而且它通常保留：

- `versionDir`
- `previous`

这背后的语义非常明确：

- current 当然要保留
- previous 是回滚的兜底

只有当 ready 写回、回滚窗口过去后，剩余更老的版本才算真正进入可清理状态。

## 11. `runHostRemove` 为什么先删物理版本树再删 row

[runHostRemove](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L138-L171) 先做：

- `tree.Remove(existing.Name)`

再调用：

- `finishRemoval(...)`

这和远端 remove 的顺序完全一致：

- 先让运行时事实消失
- 再让数据库关系收尾

否则会出现一种明显不一致：

- row 已删
- 但本地技能链接和版本目录还在

## 12. `tree.Remove` 真正删除的是什么

[tree.Remove](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/tree.go#L204-L219) 会做两件事：

1. 删除 `<root>/<name>` 活动链接
2. `Prune(name)` 删除全部版本目录

它不是简单的“unlink 当前版本”，而是：

- **把这项 host skill 从本地彻底移除**

之后 `Installed(name)` 也会立刻返回 false。

## 13. 为什么 `Installed` 只看活动链接下有没有 `SKILL.md`

[Installed](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/tree.go#L221-L229) 不是看：

- `.versions` 下有没有目录

而是看：

- `<root>/<name>/SKILL.md` 是否存在

也就是说，历史目录不代表“技能当前可服务”。

这和远端逻辑里“ledger 里有 superseded snapshot，不代表 live image 正在跑它”是同一套判断方式。

## 14. `Sweep` 为什么不是另一个 `Prune`

[Sweep](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/tree.go#L231-L282) 和 `Prune` 容易看混，但两者职责完全不同：

### `Prune`

- install 成功后的正常退休
- 调用方明确告诉它 keep 哪些版本

### `Sweep`

- 崩溃恢复
- 清理 `.next-<name>` 临时链接
- 清理“比 live 更新却从未激活”的孤儿版本
- 对没有 live link 的技能，清掉整棵残留版本目录

所以 `Sweep` 更像远端的 reaper，而不是 install 成功路径里的收尾。

## 15. 为什么 `Sweep` 故意不删 `<= live` 的版本

`Sweep` 在计算 live map 之后，只会清：

- 没有 live link 的整棵残留
- 或者版本号大于当前 live 的目录

它不会删：

- live 本身
- 小于等于 live 的历史版本

原因是这些目录可能正是：

- rollback 还要用的 previous

清得太激进，就会把“正常安装后的回滚余地”误当成崩溃残留。

## 16. 为什么 `Sweep` 也要先拿 per-skill lock

[sweepLocked](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/tree.go#L284-L293) 会先尝试拿锁，再删目录或 `.next-` 链接。

锁的实现见：

- [lock_unix.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/localsandbox/skilltree/lock_unix.go#L1-L38)

这意味着 sweep 不会清掉：

- 另一 Lite 进程里当前正安装/删除的技能版本树

它宁可跳过一次，也不冒并发误删的风险。

## 17. `Start` 里为什么会顺手跑一次 `SkillTree.Sweep`

在 [tenant_skill_reaper.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_reaper.go#L915-L936) 里，技能服务启动时会：

- 注册远端 stuck-run sweep
- 如果 host skills 可用，再立即跑一次 `SkillTree.Sweep()`

这说明 host 版 housekeeping 被有意收进了同一套技能后台维护节奏里。

也就是说，虽然远端和 host 的物理提交介质不同，但：

- 它们的恢复和清理都由同一个技能服务入口统一发起

## 18. `noSymlinkPath` 为什么是 host 模式下特别关键的防线

host 模式还有一个远端模式没有的风险：

- 主进程自己就在用户机器上

所以 [noSymlinkPath](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L214-L236) / [readNoSymlinkFile](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L238-L252) 会在安装后读 `skill.json`、清缓存这类动作前再次确认：

- 路径里没有符号链接

这层保护是为了防止：

- 安装 agent 在受限环境里种下一条链接
- 主进程后续无意中跟着它读到了 skill 目录之外的用户文件

## 19. 这篇和前后文怎么接

建议按下面顺序连读：

1. [25-code-reading-skill-source-host-staging-prompt.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/25-code-reading-skill-source-host-staging-prompt.md)
2. [30-code-reading-skill-file-browser-and-skill-read.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/30-code-reading-skill-file-browser-and-skill-read.md)
3. 当前这篇 `31`
4. [20-code-reading-skill-stop-reaper-recovery.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/20-code-reading-skill-stop-reaper-recovery.md)
5. [21-code-reading-skill-snapshot-ledger-prune-reconcile.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/21-code-reading-skill-snapshot-ledger-prune-reconcile.md)

这样能把 host / remote 两条技能安装链对着看：

- host source 与 staging 怎么建立
- 文件读取怎么统一抽象
- host 版版本树怎么完成 install / activate / remove / sweep
- 再回头看 remote 那边的 stop / reaper / prune，会更容易对位

## 20. 二开时最容易踩的坑

### 坑一：把 host skill 当成“没有镜像概念的简化版”

它只是把 snapshot 换成了版本目录和活动链接，提交语义并没有变简单。

### 坑二：复用旧版本目录继续装

这会把 Python venv、依赖缓存和脚本产物混成一锅。

### 坑三：版本目录一写好就当 install 成功

真正生效的是 `Activate` 和后续 ready state 写回，不是目录存在本身。

### 坑四：把 `Sweep` 当成 install 成功后的正常清理

`Sweep` 是崩溃恢复；正常退休应该走 `Prune`。

### 坑五：host 主进程后续读文件时不重新做反链接检查

安装 agent 的沙箱边界不等于主进程自己的边界，两者不能混为一谈。
