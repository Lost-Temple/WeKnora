# 技能 Source、Host Staging 与 Prompt 注入阅读指南

这一篇接着 [24-code-reading-skill-manifest-env-runtime-injection.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/24-code-reading-skill-manifest-env-runtime-injection.md) 往下读，专门把技能系统里另一条很容易绕晕的支线讲清楚：

- agent 到底是从哪里“看见”一个 skill 的
- 为什么 installed skill 和 host skill 要走两套 source / staging 逻辑
- 用户 `@技能` 以后，prompt 里到底多了什么约束

如果说 `24` 号文档解决的是：

- skill 已经进镜像后，运行时 env、工具面和持久化值怎么闭环

那这一篇解决的是：

- 这些 skill 说明、资源文件和 prompt 约束，本轮是从哪套 source 暴露出来的

## 推荐阅读顺序

1. [agent_service.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L688-L777)
2. [manager.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/manager.go#L95-L391)
3. [source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/source.go)
4. [tenant_source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go)
5. [shell_staging.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/shell_staging.go)
6. [shell_environment.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/shell_environment.go)
7. [observe.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/observe.go#L685-L744)
8. [agent_service.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L1703-L1736)

## 1. 先分清“技能定义从哪里来”和“技能运行时从哪里来”

技能系统里至少有两类 source：

### 第一类：host skill 目录

也就是 WeKnora 宿主机上本地目录树里的 skill。

### 第二类：installed skill image

也就是管理员安装进某套 sandbox config 镜像里的 skill。

两者都能表现成 `SkillSource`，但语义不一样：

- host skill 是“平台机器上有一份源码/资源”
- installed skill 是“当前 session 真会在 sandbox 里执行的一份运行时副本”

这两者不能混着告诉模型，否则模型看到的路径和资源，可能根本不是当前 sandbox 能执行到的那份。

## 2. `SkillSource` 这个接口真正统一了什么

[source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/source.go) 里定义的 `SkillSource` 统一的是“逐级披露 skill 信息”的接口：

1. `DiscoverSkills`
2. `LoadSkillInstructions`
3. `LoadSkillFile`
4. `ListSkillFiles`
5. `GetSkillBasePath`

这五层刚好对应 agent 逐层需要的视图：

- 系统提示词先只要 Level 1 元数据
- 用户/模型进一步追问时再拿 `SKILL.md`
- 再往下才会读资源文件或列文件

所以 `SkillSource` 统一的是“怎么说出一个 skill”，而不是“怎么执行一个 skill”。

## 3. `tenantSkillSource` 为什么不再重新判断 skill 可用性

[tenantSkillSource](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L737-L768) 很克制。

它做的是：

- 把 `AgentConfig.TenantSkills` 转成 `skills.TenantSkillSource`

而不是：

- 再跑一轮启用性判断
- 再去数据库重查一遍哪些 skill 可用

这是因为在更早的构建链上：

- `skillsForRun`
- `effectiveTenantSkills`

已经把“这轮实际可调用的 skill 集合”收窄好了。

`tenantSkillSource` 只是把这份结果接到 skill manager，不再改真值。

## 4. 为什么 skill manager 一旦有 `tenantSource`，就不再看 host loader

[manager.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/manager.go#L153-L191) 里最重要的一个决定就是：

- 如果有 `tenantSource`
- 那它就是唯一的 discover/load source

原因很直接：

- 远端 sandbox 真能执行的是 image 里的那份 skill
- 宿主机同名目录即使存在，也不是当前 session 里的真运行时

如果此时还把 host loader 混进来，模型就可能看到：

- 一个宿主机目录里的 skill 说明
- 但当前 sandbox 里根本没有它

这类“能读到但不能跑”的能力，对 agent 来说几乎总是负资产。

## 5. `GetAllMetadata` 为什么是“模型能看见谁”的真源

[GetAllMetadata](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/manager.go#L236-L251) 返回的是 skill manager 初始化后缓存下来的 metadata。

这份列表会进系统提示词，因此它就是：

- 模型这轮“知道自己有哪些技能可读/可用”的直接来源

所以技能系统里有个非常重要的区分：

- skill row 存不存在，不等于模型知道不知道
- 模型知道不知道，取决于 manager 初始化时 discover 到了谁

## 6. `TenantSkillSource` 为什么优先用 row 提供说明，用 bundle 提供资源

[tenant_source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go) 里有个很清晰的分层：

### name / description / instructions

直接来自 row。

### 资源文件

来自 archive bundle。

这样分层有两个好处：

1. 告诉模型“这 skill 是什么”时不需要等 object storage
2. 真要读资源文件时，又能回到安装时那份原始 bundle，而不是要求当前 turn 已经起了 sandbox

换句话说：

- row 负责便宜、稳定的说明层
- bundle 负责按需展开的资源层

## 7. 为什么 `usableSkillRow` 是 installed-skill source 的最后一道闸门

[usableSkillRow](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L110-L135) 统一过滤掉这些 row：

- 未 ready
- 被禁用
- name 非法

特别是最后那个 name 校验，非常重要。

因为 `TenantSkillSource` 往外暴露的路径最终都会走：

- `skillDir(row.Name)`

如果 name 不是单一路径段，就可能在“模型可见路径”层面越出 skill 目录甚至越出 root。

所以这里不是“多一道保险”，而是 installed-skill source 能不能安全说出路径的前提。

## 8. 为什么 `LoadSkillInstructions` 不回头下载 bundle

[LoadSkillInstructions](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L157-L179) 直接返回 row 上的 `Instructions`。

这意味着：

- 模型第一次了解这个 skill 时，不依赖 bundle 下载成功
- object storage 短暂抖动时，skill 说明层仍然可读

项目明确把“技能主说明”当成控制面数据，而不是资源文件的一部分。

## 9. 为什么 `LoadSkillFile` 明明读的是 bundle，却给模型返回 image 内路径

这是这条链里最值得注意的一个设计。

[LoadSkillFile](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L181-L220) 读文件内容时用的是 bundle，但返回给模型的 `Path` 却是：

- image 内真实运行路径

这是因为：

- 读内容时，bundle 是最稳定、最不依赖 live sandbox 的来源
- 但模型后续如果要基于这个路径采取行动，真正存在于 sandbox 的是 image 内路径

所以它故意把“内容来源”和“面向模型的路径语义”拆开处理。

## 10. 为什么 bundle 还要再做一层 per-turn 缓存

[bundleArchive](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L300-L365) 会把 zip archive 做一层按 turn 生命周期的 LRU 缓存。

原因是：

- 一个回合里 `read_file` / `list_skill_files` 很可能连续多次打到同一个 skill
- 每次都重新下载同一个 zip 太浪费

但它缓存的仍然是压缩包，而不是整包解压结果。

这又是一个很刻意的平衡：

- 保住重复读取的效率
- 但不让一个大 skill 的所有文件长期以解压状态常驻内存

## 11. `host skill` 为什么不能直接把宿主机目录暴露给 sandbox 执行

host skill 是存在宿主机上的，而普通 remote sandbox shell 根本看不到宿主机目录。

所以当 `PrepareShellEnvironment` 遇到 host skill 时，不能直接说：

- `WEKNORA_SKILL_DIR=/Users/.../skills/foo`

这对 sandbox 来说是个不存在的路径。

因此 host skill 必须先走 staging。

## 12. `stageShellSkill` 真正在做什么

[stageShellSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/shell_staging.go#L20-L100) 做的是：

1. 确认这个 skill 已经对当前 agent 可见
2. 从 source 列出资源文件
3. 过滤掉 `.venv`、`node_modules` 等不该直接复制的大目录
4. 把资源写进当前 session workspace 下的 `.skills/<name>/<digest>`
5. 用 `sessionID + skillName` 缓存这个 staged 目录

它不做的事情也要记清：

- 不执行 skill
- 不在这里安装依赖
- 不在宿主机上直接跑代码

所以 staging 的语义是：

- “把一份可读资源复制进这次会话”

而不是：

- “把 host skill 变成 fully installed skill”

## 13. 为什么 staged 目录要按内容摘要命名

如果目录名只按 skillName 固定，那么模型或用户在 session 里改过文件之后：

- 下次同 session 再调用这个 skill
- 很可能继续复用一份已经被改脏的目录

按 digest 命名后：

- 只要源内容变了，就会换一条新路径
- 旧缓存也更容易和新内容区分开

再配合 `stagedSkillStillIntact` 校验，就能避免把脏树错当成干净 staging 复用。

## 14. `stagedSkillStillIntact` 为什么先盯 `SKILL.md`

[stagedSkillStillIntact](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/shell_staging.go#L112-L176) 会优先检查：

- staged 目录里的 `SKILL.md` 还在不在
- 内容是不是还和 source 相同

因为 `SKILL.md` 是最核心、最稳定也最容易被编辑的一份文件。

只要它被改过，就几乎已经可以确定：

- 这棵 staged tree 不是干净副本了

然后它再补查其它资源是否仍然存在。

## 15. `PrepareShellEnvironment` 为什么统一走同一个 shell primitive

[PrepareShellEnvironment](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/shell_environment.go#L10-L49) 的设计重点不是“新增一种 skill shell”，而是：

- 继续复用普通 `shell_exec`
- 只在执行前把 skill runtime 所需的环境附着上去

这样 installed skill 和 host skill 最终都可以复用同一套 shell 调用：

- installed skill 直接用 image 内目录
- host skill 先 stage，再把 staged 目录当成 `WEKNORA_SKILL_DIR`

模型侧不需要知道背后到底是哪种 source，只需要知道：

- 带 `skill_name`
- shell runtime 就会被准备好

## 16. 为什么 `SandboxSkillDir` 只对 installed skill 有意义

[SandboxSkillDir](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/manager.go#L308-L327) 只会对 installed skill 返回一个可对外讲的路径。

原因很简单：

- installed skill 的 base path 就在 image 里
- host skill 的原始 base path 在宿主机上，sandbox 够不到

这也是为什么：

- 读取/展示 skill 资源时，可以同时支持两类 source
- 但直接说“skill 在 sandbox 里的目录是哪里”时，只能对 installed skill 成立

## 17. `@技能` 到底怎样进入 prompt

这一步有两层：

### 第一层：`resolvePinnedSkillInfos`

[resolvePinnedSkillInfos](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L1703-L1736) 把 `PinnedSkillNames` 翻译成：

- name
- 可选 description

这里不做复杂解析，只补 prompt 真正需要的最小字段。

### 第二层：`buildMustUseBlock`

[buildMustUseBlock](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/observe.go#L685-L744) 把这些 pinned skills 变成：

- `Must call read_file(path="skill://<name>/SKILL.md") before answering.`

也就是说，`@技能` 的 prompt 约束不是：

- 自动把整份 skill 说明塞进 prompt

而是：

- 强制模型先显式读取这份技能说明

## 18. 为什么 `@技能` 只要求先读 `skill://.../SKILL.md`

这是一个很好的平衡点。

如果直接把完整 `SKILL.md` 注入系统提示词，会有几个问题：

- prompt 膨胀
- 无论当前任务是否真需要这个 skill，都先消耗上下文
- 技能说明变更后也更难保持“显式读取”的审计痕迹

而让模型先调用：

- `read_file(path="skill://.../SKILL.md")`

则有几个好处：

- 使用痕迹明确
- 只在本轮真需要时才读
- 同时保留“用户点名了这个 skill，必须先看说明”的刚性约束

## 19. 为什么 must-use 提示还要强调“不替代其它研究”

`buildMustUseBlock` 的结尾专门补了一句：

- 这些 selections 不替代对任务事实内容的其它研究

这是因为 `@技能` / `@MCP` 很容易被误读成：

- 只允许用这个 skill
- 或者不需要别的检索了

但大部分时候，用户只是说：

- 这个资源和当前任务强相关，请务必先用它

并不是在说：

- 其它来源都禁止使用

所以这里明确把“优先使用”和“唯一允许”分开。

## 20. 这篇和前后文怎么接

建议按下面顺序连读：

1. [23-code-reading-skill-image-effective-config-session.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/23-code-reading-skill-image-effective-config-session.md)
2. [24-code-reading-skill-manifest-env-runtime-injection.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/24-code-reading-skill-manifest-env-runtime-injection.md)
3. 当前这篇 `25`

这样三篇刚好能把技能运行时另一端也补齐：

- session 怎么选到 skill image
- env / shell / 持久化值怎么闭环
- source / staging / prompt 约束又是怎么把技能暴露给模型的

## 21. 二开时最容易踩的坑

### 坑一：远端 sandbox 有 installed skill 时，仍然把 host loader 暴露给模型

这会让模型读到一份当前 sandbox 根本执行不到的技能定义。

### 坑二：host skill 直接把宿主机目录路径告诉 sandbox shell

remote sandbox 根本看不到这个路径，模型只会得到一条无法复现的命令。

### 坑三：把 staging 当成安装

staging 只是把资源复制到本 session，不负责装依赖，也不改变镜像。

### 坑四：`@技能` 时直接把整份 SKILL.md 永久塞进系统提示词

这会让 prompt 膨胀，而且失去“模型显式读取过技能说明”的可观察性。

### 坑五：must-use 提示里把 `@技能` 理解成“排斥其它来源”

用户通常是在加优先级，不是在取消其它必要的研究动作。
