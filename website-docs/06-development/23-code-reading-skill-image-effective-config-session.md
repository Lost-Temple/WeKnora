# 技能 SkillImage、Effective Config 与 Session 启动阅读指南

这一篇接着 [22-code-reading-skill-snapshot-commit-pointer-switch.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/22-code-reading-skill-snapshot-commit-pointer-switch.md) 往后读，专门回答一个很关键的问题：

- 技能镜像既然已经写进了 `SkillImage`，后面的 session、agent、artifact、teardown 到底是怎么消费这份信息的

前两篇主要讲的是：

- 快照链如何被写出来
- ledger / prune / reconcile 后来如何维护它

这一篇讲的是：

- 这些 snapshot 信息最后怎样影响一次真实运行

## 推荐阅读顺序

1. [tenant_skill_effective.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_effective.go)
2. [skill_image.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sandbox/skill_image.go)
3. [tenant_config.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sandbox/tenant_config.go#L38-L176)
4. [session_sandbox_pin.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_sandbox_pin.go#L222-L329)
5. [session.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session.go#L694-L756)

## 1. 先分清两个容易混掉的问题

技能系统里有两个经常被误当成同一个的问题：

### 问题一：数据库里有哪些 skill row

这是安装/删除管理视角。

### 问题二：这次 session 真能调用哪些技能

这是运行时视角。

这两者不能简单画等号。因为 skill row 只是记录“系统认为这项技能处于什么状态”，而运行时真正要看的是：

- 当前 session 启动的是哪套 sandbox config
- 这套 config 当前是不是还真的 boot 到了 `SkillImage`
- 这张 image 里是否仍然携带这个技能的文件
- 这个技能是否还启用、是否存在 served version

`tenant_skill_effective.go` 做的，就是把问题一收敛成问题二。

## 2. `skillsForRun` 为什么先看 pin，而不是先看 agent 当前配置

[skillsForRun](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_effective.go) 一上来先做的不是直接读 `agentConfigID`，而是：

- 去查这个 session 有没有已经存在的 sandbox pin

原因是远端 sandbox 是长寿命资源：

- 一个 session 的 sandbox 创建出来以后可能会暂停、恢复、继续跑很多轮
- 中途管理员可能已经把 agent 改指向了另一套 sandbox config

如果这时还按 agent 当前配置来给模型注入技能，就会出现严重错位：

- 模型看到的是新 config 的技能集合
- 实际运行的却还是旧 sandbox 里那张老 image

这样模型就会反复调用实际上不存在的技能，浪费回合还难排查。

所以 `skillsForRun` 的策略是：

1. 有 pin 时，以 pin 为准
2. 没 pin 时，才回退到 agent 当前配置

## 3. `effectiveTenantSkills` 为什么先问 `SkillImageActive`

[effectiveTenantSkills](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_effective.go) 里最关键的一步是：

- 先判断 `sandbox.SkillImageActive(cfgEntity.Config)`

它不是先把所有 ready skill row 列出来，而是先判断：

- 当前 config 记录的 `SkillImage`，是不是 session 现在真的会启动的那张镜像

如果答案是否定的，它会直接返回零技能，而不是“尽量返回一个差不多的子集”。

因为一旦 session 实际 boot 回了 base template：

- skill files 一个都不在
- 再怎么从 row 里挑状态“看起来健康”的技能，也只是给模型制造假能力

这也是为什么这里的失败策略是“返回空”，而不是“尽量保留部分技能”。

## 4. `ServedView()` 为什么重要

即便 `SkillImageActive` 为真，也不是所有 row 都能直接暴露。

`effectiveTenantSkills` 还会继续过滤：

- row 必须启用
- row 必须存在 `ServedView()`

这一步解决的是安装中、失败升级、旧版本继续 serving 的情况。

换句话说，运行时应该看到的是：

- 当前 image 里实际有文件、且允许被调用的那一版 skill

而不一定是 row 表面上的最新 version。

## 5. `SkillImageActive` 到底在判断什么

[SkillImageActive](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sandbox/skill_image.go) 是整个消费链的核心事实判断之一。

它不是只看：

- `SkillImage != nil`
- `SkillImage.SnapshotID != ""`

而是同时判断：

1. 当前 backend 是否支持 snapshot override
2. `OwnerFingerprint` 是否存在
3. `OwnerFingerprint` 是否仍与当前 provider credentials 匹配

只有这些都满足时，它才承认：

- 这条 `SkillImage` 现在仍然是 session 真会启动的镜像

否则就要退回 base template。

## 6. 为什么要有 `OwnerFingerprint`

[SkillOwnerFingerprint](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sandbox/skill_image.go) 记录的是：

- 这张 skill snapshot 属于哪个 provider 账号/daemon

它的核心作用不是给人看，而是防 credential rotation。

典型场景是：

- 之前的 snapshot 用旧的 Cube/E2B 账号创建
- 现在 config 上的 API key 已经换成另一个账号

这时数据库里虽然还留着同一个 `SnapshotID`，但当前账号其实根本解析不到它。

如果没有 fingerprint 保护，session 启动链就会把一个“当前账号打不开的 snapshot id”继续当成 live image 使用。

## 7. `ResolveEffectiveConfig` 为什么是 session 侧真正消费 `SkillImage` 的地方

[ResolveEffectiveConfig](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sandbox/tenant_config.go#L38-L176) 做的是：

- 把存储态的 tenant sandbox config 转成真正用来建 manager 的运行时 config

对技能镜像来说，它最重要的逻辑就是这一步：

- 在 provider block、默认值和校验都准备好之后
- 根据 `SkillImage` 决定要不要覆盖模板字段

具体体现为：

- Cube 覆盖 `CubeTemplate`
- E2B 覆盖 `E2BTemplate`
- Docker 覆盖 `DockerImage`

这意味着 session 侧其实不需要知道“技能系统”本身的很多细节。对下游来说，它只是在启动 sandbox 时拿到了一个已经被 skill snapshot 替换过模板的有效配置。

## 8. 为什么 skill snapshot 的覆盖只改模板字段

这一点非常关键。

`ResolveEffectiveConfig` 并不会因为有了 skill image 就改一大堆其他字段，它只改：

- 模板/镜像字段

原因是 skill snapshot 的语义本来就是：

- 在同一套 provider、同一套连接信息、同一套网络策略上
- 把“从哪个基础模板启动”替换成“从哪张 skill image 启动”

这样下游 manager 的逻辑可以保持稳定：

- 它始终只认模板/镜像字段
- 不需要知道这次模板值来自 base template 还是 skill snapshot

## 9. 为什么 session pin 是这条链的另一半真源

如果只有 `SkillImageActive` 和 `ResolveEffectiveConfig`，还不够。

还必须回答另一个问题：

- 这次运行到底该读取哪条 config

这就是 [resolveSandboxForExecution](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_sandbox_pin.go#L222-L316) 和 [sandboxConfigForExistingSandbox](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_sandbox_pin.go#L323-L329) 的职责。

它们保证：

- 新 session 的第一次创建，可以跟随 agent 当前配置
- 已经存在的 sandbox，后续所有操作都继续沿着旧 pin 走

也就是说，运行时真实 backend 的选择，首先由 pin 决定；而在选定 config 之后，真正启动哪张 image，再由 `ResolveEffectiveConfig` 里的 `SkillImage` override 决定。

## 10. 为什么 `destroyBoundSandbox` 也必须走 pin

[destroyBoundSandbox](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session.go#L694-L756) 是这条链很容易被忽略的一环。

它清理 session sandbox 时不会问：

- agent 现在在哪套 config 上

而是会先去读这个 session 当年的 pin。

原因很现实：

- teardown 是个“事后动作”
- 它经常发生在聊天请求上下文之外
- 此时 agent 当前配置可能早就变了

如果 teardown 不沿 pin 找回原来的 backend，就会把真正还在计费的 paused sandbox 永远留在 provider 上。

## 11. 为什么 agent 侧技能注入和 session 侧镜像选择必须共用同一套判断

这条链最核心的设计目标，其实是避免下面这类断层：

### 断层一：模型觉得有技能，session 实际 boot 的 base template 里没有

模型连续调用失败。

### 断层二：session 其实已经 boot 进了 skill image，但 agent 侧把技能全隐藏了

技能明明可用，却永远不会被调用。

所以项目里特意把判断拆成了两层、但保持一致：

- `SkillImageActive` 负责判断 `SkillImage` 现在是否还是真实 live image
- `ResolveEffectiveConfig` 负责在 session 启动时用同样的条件去做模板覆盖

agent 侧和 session 侧只是在不同位置消费这套结论，而不是各自发明一套判断规则。

## 12. 这篇和前后文怎么接

建议按下面顺序连读：

1. [21-code-reading-skill-snapshot-ledger-prune-reconcile.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/21-code-reading-skill-snapshot-ledger-prune-reconcile.md)
2. [22-code-reading-skill-snapshot-commit-pointer-switch.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/22-code-reading-skill-snapshot-commit-pointer-switch.md)
3. 当前这篇 `23`

这样三篇合起来刚好形成闭环：

- snapshot ledger 后来怎样维护
- install/remove 当时怎样写 pointer 和 generation
- session 与 agent 最终怎样消费 SkillImage

## 13. 二开时最容易踩的坑

### 坑一：直接按 skill row 给模型注入技能，不看 `SkillImageActive`

这会让模型看到一批当前 image 根本不存在的能力。

### 坑二：session 后续动作回头看 agent 当前配置，不看 pin

这会把 artifact 收集、terminal、teardown 全部导向错误 backend。

### 坑三：在 `ResolveEffectiveConfig` 里为技能镜像改动过多 provider 字段

skill snapshot 的语义只是替换模板，不是替换账号、网络和连接身份。

### 坑四：把 `OwnerFingerprint` 当成装饰字段

它其实是 skill image 是否仍然可启动的关键保护条件。
