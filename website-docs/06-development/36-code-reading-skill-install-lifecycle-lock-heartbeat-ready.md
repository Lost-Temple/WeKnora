# 技能安装 Lifecycle、Run Lock、Heartbeat 与终态落盘阅读指南

这一篇接着 [35-code-reading-skill-install-guidance-steer-repair-boundaries.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/35-code-reading-skill-install-guidance-steer-repair-boundaries.md) 往后读，专门解释技能安装从“接单”到“真正 ready/failed”这条执行链。

这一轮最值得盯住的不是 agent，而是安装流程自己的执行边界：

- 请求何时算真正接单
- 哪个 run 还拥有这条 row
- 心跳为什么必须在 ready/fail 前停掉
- 指针切换后为什么只能补账，不能再把安装判失败

## 推荐阅读顺序

1. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L64-L207)
2. [tenant_skill_stop.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_stop.go#L13-L107)
3. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L442-L731)
4. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1490-L1712)
5. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2612-L2660)
6. [19-code-reading-skill-progress-state-transitions.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/19-code-reading-skill-progress-state-transitions.md)
7. [22-code-reading-skill-snapshot-commit-pointer-switch.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/22-code-reading-skill-snapshot-commit-pointer-switch.md)

## 1. `accepted` 不等于 sandbox 已经起来

[installParsedSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L84-L207) 在真正启动后台 goroutine 之前，就已经完成了几件很关键的事情：

- 解析并校验 bundle
- 决定这次 install 是更新已有 skill 还是创建新 row
- 把 row 切到 `installing`
- 把 archive/catalog 绑定好

这就是进度条里 `accepted` 的真实含义：

- **安装请求已经被正式接单**

但这时仍然还没：

- 拿到临时 sandbox
- reset skill 目录
- seed 文件
- 跑 installer agent

所以 `accepted` 只是“后台任务已经排起”，不是“远端环境已经可执行”。

## 2. 为什么后台 goroutine 不继承 HTTP 请求取消

同样在 [installParsedSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L195-L207) 可以看到，后台安装是用 `context.WithoutCancel(ctx)` 启动的。

原因很直接：

- 安装通常比当前 HTTP 请求活得久
- 浏览器断开、页面切走，不应该直接把 install 判失败

但它也不是可持久恢复的 job 系统：

- 进程内 run 靠 `StopSkill` 直接取消
- 进程外僵尸态靠 stuck-run reaper 清理

也就是说，这里追求的是“请求级别不绑死”，不是“进程重启后自动续跑”。

## 3. `withSkillRunLock` = config 级互斥 + 进程内可取消 run

[withSkillRunLock](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_stop.go#L90-L107) 并不只是包一层锁，它实际上组合了两件事：

1. `bindSkillRun(...)`：把当前 goroutine 的 cancel 句柄挂到 `tenant+config+skill` 这把业务键上
2. `withConfigLock(...)`：确保同一配置的 image 修改流程串行执行

所以它解决的是两个不同层面的并发问题：

- **跨请求/跨副本**：靠 config 级锁避免两个 install/remove 同时改同一镜像
- **当前进程内的人为 stop**：靠 `cancelSkillRun(...)` 直接打断正在跑的 goroutine

## 4. 为什么“run 还绑定着”要看两层所有权

这一条链有两个互补的 owner 判断：

### 第一层：`skillRunStillBound(...)`

看的是：

- 当前 goroutine 的 handle 指针
- 现在是不是还挂在 `runCancels` map 里那一把

如果不是，说明：

- 新的一轮 run 已经把旧句柄顶掉

### 第二层：`installStillOwnsTheRow(...)`

看的是数据库语义上这条 row 还属不属于当前 install：

- row 是否已经变成 `removing / failed`
- `BundleSHA256` 是否已经换成别的 archive
- skill 是否已经 `ready` 且 live image 里确认是另一轮结果

只有这两层都没失效，旧 run 才有资格继续：

- snapshot
- 写 `ready`
- 写 `failed`

## 5. `runInstall()` 里真正的分水岭是 `pointerSwitched`

[runInstall](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L442-L731) 前半段都还发生在“临时 sandbox 世界”里：

- 启动 maintenance session
- `sandbox_ready`
- reset 目录
- seed 文件
- installer + verify
- 清 scratch
- 创建 snapshot

到这里为止，失败都还是“这轮安装没有真正对外生效”。

但一旦 `switchImagePointer(...)` 成功：

- 新 snapshot 已开始 serving 新会话

这就是 `pointerSwitched = true` 的意义：

- **它之后的错误只允许视为 bookkeeping 缺口**
- **不能再把整个 install 记成 failed**

否则就会出现：

- 实际上技能已经可用
- durable row 却被写成失败

这种最难排查的假故障。

## 6. 为什么 snapshot 前还要重新检查 owner

`runInstall()` 在 agent/verify 跑完后，又重新检查了一次：

- `skillRunStillBound(...)`
- `installStillOwnsTheRow(...)`

这是因为 install 主体可能已经执行了几分钟，期间完全可能发生：

- 同名新上传
- queued remove
- StopSkill 之后的 retry

如果不在 snapshot 前再确认一次，就会把一个语义上已经被别人接管的 tree 继续提交进 ledger。

## 7. Heartbeat 为什么不是“纯保活”，而是“状态写入者”

[startInstallHeartbeat](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1657-L1712) 和 [beatInstallHeartbeat](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1688-L1712) 最大的误区是：很容易把它想成一个旁路 ping。

其实不是。

它做的是：

- 重新读取 skill row
- 更新 `InstallingSince`
- 再把整条 row 写回去

所以 heartbeat 本身也是 durable 状态写入者。

这带来两个重要后果：

### 第一，reaper 和重复上传都依赖它判断“这条 installing 还活着”

如果长时间没有心跳：

- stuck-run reaper 会认为进程已经死了
- 同 digest 的再次上传也会被允许开新 run

### 第二，它必须在 `ready/fail` 前先停掉

因为一个迟到的 heartbeat 完全可能把已经写好的：

- `ready`
- `failed`

重新覆盖回：

- `installing`

所以 `runInstall()` 在指针切换后会先 `stopHeartbeat()`，再写 `ready`；失败路径也依赖 defer 顺序，先停心跳，再执行 `failSkill(...)`。

## 8. `writeReadySkillState()` 为什么要重试，但不能乱写

[writeReadySkillState](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1490-L1528) 发生在 pointer 已切换之后。

这意味着：

- 技能已经安装成功并开始 serving
- 缺的只是 row 里的最终 bookkeeping

因此这里的失败不该触发“整轮 install 失败”，而应该做的是：

- 用 `retrySkillBookkeeping(...)` 重试

但即便是补账，也不能盲写。它仍然要先调用 `installStillOwnsTheRow(...)`，防止把 `ready` 写到一条已经被更新 install/remove 接管的 row 上。

## 9. `failSkill()` 为什么也不能想写就写

[failSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1554-L1570) 不是“任何错误都去落 failed”。

它也要先判断：

- 当前 run 是否仍然拥有这条 row

这是因为 install 失败时，外部世界可能已经发生了别的操作：

- 管理员点了 StopSkill
- 新上传接管了同名 skill
- remove 已经把 row 切成 `removing`

如果旧 run 此时仍强行写 `failed`，就会把后来者的真实状态冲掉。

## 10. `markConfigSandboxesStale()` 为什么是成功后的 best-effort

[markConfigSandboxesStale](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2612-L2660) 发生在 `ready` 之后。

它解决的问题不是“这轮 install 是否成功”，而是：

- 当前已经打开、仍绑在旧 sandbox 上的会话
- 什么时候切到新 image

所以它被明确设计成 best-effort：

- 标记成功，旧会话下次使用时会重建到新 image
- 标记失败，也只是暂时继续跑旧 image

这里绝不能反过来把整个 install 说成失败，因为 pointer 早就已经切过去了。

## 11. 阅读这一段时最该关注什么

### 第一，区分“运行 owner”和“row owner”

一个是进程内当前 goroutine 句柄是谁；
一个是数据库里这条 row 现在语义上属于谁。

两者混看，就会误判为什么某些路径明明还在执行，却已经不允许再写终态。

### 第二，区分“真正失败”和“补账失败”

在 `pointerSwitched` 之前：

- 失败就是 install 失败

在 `pointerSwitched` 之后：

- 失败大多只是 bookkeeping 不完整

这是 install 生命周期里最关键的语义分水岭。

### 第三，别把 heartbeat 当成只读保活

heartbeat 会写 row。

只要记住这一点，就会自然明白为什么它必须：

- 持续跑
- 严格停在终态写入之前
- 只给仍处于 `installing` 的 owner 续命
