# 技能安装 Sandbox Bootstrap、目录重置与文件种子阅读指南

这一篇接着 [36-code-reading-skill-install-lifecycle-lock-heartbeat-ready.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/36-code-reading-skill-install-lifecycle-lock-heartbeat-ready.md) 往后读，专门收拢技能安装真正进入远端 sandbox 之前和刚进入之后的那几步：

- 怎么确认当前 image 还能安全增量构建
- 怎么创建一条“隐藏但可追溯”的 maintenance session
- 为什么先 `reset` 目录再 `seed` 文件
- 为什么 seed 走服务端 tar，而不是让 installer agent 自己把源码写回去

这一段的核心不是 agent，而是：

- **先把安装现场准备正确**
- **再让 installer 在正确的 skill tree 上工作**

## 推荐阅读顺序

1. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L442-L626)
2. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L732-L980)
3. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1712-L1798)
4. [33-code-reading-skill-install-transcript-progress-timeline.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/33-code-reading-skill-install-transcript-progress-timeline.md)
5. [21-code-reading-skill-snapshot-ledger-prune-reconcile.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/21-code-reading-skill-snapshot-ledger-prune-reconcile.md)

## 1. 为什么安装前先过 `ensureUsableImage`

[ensureUsableImage](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L732-L760) 不是在检查“当前配置有没有 image”，而是在检查：

- 当前配置记录的 `SkillImage`
- 是否仍然属于现在这套 provider 凭据

如果凭据已经轮换，session 层实际会回退到 base template。

这时如果仍继续做 install：

- sandbox 里其实没有旧技能树
- 但新 snapshot 会被当成“在旧 image 基础上增量构建”的结果
- 最后 pointer 一切，row 还都显示 `ready`

结果就是：

- 看起来像一次正常升级
- 实际上却把整套旧技能悄悄丢掉了

所以这里宁可提前拒绝，也不能在错误账号上继续 grow image。

## 2. `startMaintenanceSession` 为什么要创建隐藏 session

[startMaintenanceSession](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1740-L1791) 同时做两件事：

1. 通过统一的 `resolveTenantSandboxForConfig(...)` 走 workspace 的 sandbox 开关与策略检查
2. 创建一条 maintenance session

这条 session 有三个特点：

- `Title` 会带上 `Skill install/remove`
- `Description` 会带 `SkillMaintenanceSessionMarker`
- `UserID` 仍然是发起操作的管理员

这三个点组合起来的效果是：

- 它不会出现在普通聊天会话列表里
- 但出问题时又能明确看出是谁发起、做的是 install 还是 remove
- transcript 和 message history 也都有真实归属

所以 maintenance session 不是“临时匿名容器”，而是：

- **隐藏的运行会话**
- **可追溯的故障现场**

## 3. 为什么 `sandbox_ready` 之后第一步是 `resetSkillDir`

[resetSkillDir](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L789-L817) 做的命令非常直接：

- `rm -rf skillDir`
- `mkdir -p skillsRoot skillDir`
- `chmod 755`

这一步的语义也非常直接：

- **这次安装只允许以 bundle 当前声明的文件树为准**

如果不先删目录，会出现两个问题：

### 第一，升级时会残留旧版本删掉的文件

当前 sandbox 是从已有 image 启动的。

所以旧版本带过来的文件本来就已经存在。如果新版本不再包含某个文件，但 reset 没做，那个文件会继续留在目录里，并最终进入新 snapshot。

### 第二，首次安装时目录可能根本不存在

`skills` 根目录不是 base template 的固定内容，所以首装时必须显式创建。

## 4. `guardSkillDir` 为什么是不可恢复的防线

[guardSkillDir](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L819-L828) 只防一件事：

- 目标目录绝不能等于 `sandbox.SkillsImageRoot`

因为 `resetSkillDir` 的本质是 destructive command。

一旦 `SkillDirFor("")` 或其它异常路径坍缩到 skills 根目录，`rm -rf` 就不再是清空单个技能，而会把整棵 skills image 清掉。

所以这里的意义不是“普通输入校验”，而是：

- **给 destructive install command 加最后一道不可恢复的保险**

## 5. 为什么 `seedSkillFiles` 走服务端 tar，而不是 agent 写文件

[seedSkillFiles](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L830-L855) 有一个非常明确的立场：

- skill tree 的源码真相来自上传 bundle
- installer agent 不负责把源码“重新写一遍”

因此它的路径是：

1. 服务端把 `bundle.Files` 打成 tar
2. 直接写进当前 session 的 image 可写区
3. 在 sandbox 内部一次性解开

这样做有两个收益：

### 第一，源码仍然和上传 bytes 严格一致

agent 后面只负责：

- 安装依赖
- 修安装期问题
- 响应 verify / guidance

而不是“根据理解重建 skill 原始文件”。

### 第二，远端 I/O 成本低得多

如果逐文件写：

- 每个文件至少是 `MakeDir + WriteFile`
- 文件一多，网络往返会把 `seeding` 阶段拖得很长

改成 tar 后，远端只需要：

- 一次上传
- 一次本地解包

## 6. 为什么 `installFileStore` 要单独卡 capability

[installFileStore](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2548-L2562) 检查的是：

- 当前 sandbox manager
- 有没有“把文件写进 skills image 根目录”的能力

这和普通会话里的 `/workspace` 文件工具完全不是一回事。

它的意义是：

- 如果 backend 根本不支持 image write，就在 seed 前立刻失败
- 而不是等流程跑到一半，才发现当前 backend 连把 bundle 落进 image 都做不到

换句话说，这一步是在保护 install flow 不出现“半准备状态”。

## 7. `packSkillTar` 为什么还要再做一遍路径约束

[packSkillTar](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L862-L895) 虽然输入已经是解析后的 `bundle.Files`，但这里仍然会再检查：

- `.` / `..`
- `../`
- 绝对路径

原因是：

- zip 解析阶段防的是 bundle 本身的越界
- tar 打包阶段防的是“最终将被远端解开的文件系统输入”越界

也就是说，哪怕前面已经做过 bundle 校验，这里仍然把 tar entry 当成另一层边界重新守一次。

## 8. `beginInstallTranscript` 为什么放在 seed 前后这个位置

[beginInstallTranscript](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L897-L948) 被放在：

- `resetSkillDir` 之后
- `seedSkillFiles` 之前

这是一个很讲究的位置。

### 为什么不更早

如果连 sandbox/session 都还没准备好，就还没有真实可 follow 的对象。

### 为什么不更晚

如果等到 installer 第一帧输出再写 locator，前端在文件种子这段时间里只能反复 404 或空白等待。

所以这里的目标是：

- sandbox 已经确定
- row 已经可写入 locator
- installer 虽然还没开始真正执行
- 但前端已经能订阅这条 transcript

这也是为什么 `SkillInstallTimeline` 能在安装很早期就有东西可跟。

## 9. 为什么 prompt 要在 transcript 和 engine 之间共用同一份

`beginInstallTranscript` 里先生成 prompt，再同时用于：

- transcript 的 opening user row
- 后续 installer engine 的首轮输入

这样才能保证：

- UI 看到的“安装开场白”
- installer 实际接收到的 prompt

是同一份内容。

否则就会出现非常难排查的偏差：

- transcript 里记录的是一段话
- 实际运行时收到的是另一段 prompt

尤其在附带管理员 instructions 时，这种偏差会直接误导后续排查。

## 10. `releaseSandbox` 为什么只销毁 sandbox，不删 session

[releaseSandbox](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1793-L1802) 只做：

- 销毁临时 sandbox

不会删掉：

- session row
- transcript message

这是故意的。

因为 install 出问题时，最有价值的现场通常不是临时 sandbox 本身，而是：

- transcript 里 agent 做过什么
- verify 到底报了什么
- guidance/repair round 是怎么串起来的

所以 sandbox 可以释放以停止计费，但 session 必须保留以供事后阅读。

## 11. 阅读这条链时最该盯住什么

### 第一，区分“安装现场准备”和“安装动作本身”

`reset / seed / transcript locator / maintenance session` 都还属于：

- 让安装现场变得可执行、可观察、可追溯

不是 installer agent 已经开始干活。

### 第二，区分“源码真相”和“安装期修改”

源码真相来自 bundle bytes；
installer 的职责是围绕这份源码完成依赖安装和修复，而不是重建原始 skill tree。

### 第三，别把 maintenance session 当成普通聊天会话

它是隐藏的，但不是匿名的；
它不进会话列表，但必须保留 transcript 供后续排查。
