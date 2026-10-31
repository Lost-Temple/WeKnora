# 技能 Manifest、环境变量声明与运行时注入阅读指南

这一篇接着 [23-code-reading-skill-image-effective-config-session.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/23-code-reading-skill-image-effective-config-session.md) 往后读，专门把技能“已经进镜像之后，为什么会在一次真实 agent 运行里可见、可调用、可复用”的最后一段收完整。

如果说 `21/22/23` 三篇已经解释了：

- snapshot ledger 怎么维护
- install/remove 怎么写 pointer
- session 怎么消费 `SkillImage`

那这一篇讲的就是：

- image 里的 skill 清单怎么自描述
- skill 需要哪些 env 变量怎么声明、怎么落库
- agent 运行时怎么把这些声明变成 tool surface 和 `shell_exec` 的真实环境
- 用户第一次在聊天里提供的 key，为什么下一轮就不用再问

## 推荐阅读顺序

1. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2288-L2450)
2. [tenant_skill_env_declare.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_env_declare.go)
3. [session_agent_qa.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_agent_qa.go#L366-L397)
4. [agent_service.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L410-L541)
5. [agent_service.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L811-L1000)
6. [user_env_resolver.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/user_env_resolver.go)
7. [user_env.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/user_env.go#L233-L286)
8. [shell_exec.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L530-L770)

## 1. 先分清三种“技能存在”

技能系统里至少有三层“存在”：

### 第一层：数据库 row 存在

表示系统记录了这个 skill 的安装状态、版本、声明等元数据。

### 第二层：镜像文件存在

表示这张 `SkillImage` 里真的带着 skill 目录、依赖和 manifest 记录。

### 第三层：本轮 agent 运行能感知并调用

表示本轮的 `AgentConfig`、tool surface、`shell_exec` 环境和用户变量解析都已经接上了。

这一篇主要讲第三层是怎么建立起来的。

## 2. `writeManifestEntry` 写的是什么

[writeManifestEntry](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2288-L2339) 会把一条 skill 记录写进 image 内的 `sandbox.SkillsManifestPath`。

写进去的字段大致是：

- `id`
- `name`
- `version`
- `sha256`
- `installed_at`

要注意，这份 manifest 不是安装真值。

真正的真值仍然是：

- skill row
- snapshot ledger
- config 的 `SkillImage`

manifest 更像是镜像内部的“自述文件”：

- operator 可以从 image 视角看当前带了哪些 skill
- remove 路径可以据此把将被移除的 skill 从镜像清单里删掉
- 调试时能快速判断“镜像自称带了什么”

所以它是镜像侧元数据，不是控制面真源。

## 3. 为什么 remove 也要改 manifest

在 [removeManifestEntry](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_remove.go#L304-L348) 里，remove 路径会把 skill 从 manifest 里删掉。

这一步很重要，因为 remove 的 commit 结果不仅是：

- 目录被删掉

还应该包括：

- image 的自描述也同步反映这次变更

否则就会出现一种很难读懂的状态：

- 镜像里的 skill 文件已经没了
- 但 manifest 还在说“我带着这个 skill”

## 4. installer 写 env 声明时，写的不是 value

[readEnvDeclaration](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2341-L2420) 读的是 installer agent 在 sandbox 里写下的 env 声明文件。

这个文件表达的不是：

- 变量的具体值

而是：

- 名字是什么
- 描述是什么
- 是否 required

项目里故意把“声明”和“值”分开处理：

- 声明来自 skill bundle / installer 判断
- 值来自管理员或用户后续填写

这样 skill 安装流程不会因为没有实际 secret 值而失败。

## 5. 为什么 env 声明失败不会让安装失败

这是这条链里非常关键的设计。

`readEnvDeclaration` 的很多失败都会被降级成：

- 记录日志
- 不保存声明
- 安装继续成功

原因是 env 声明只是一个增强体验的元数据层：

- 它能让设置页自动出现表单
- 能让 shell_exec 在 required env 缺失时给出更明确的提示

但即便没有它，skill 的文件和依赖也已经装进镜像了，不能因为一份 JSON 声明写得不好就把整次安装都判死。

## 6. `validateEnvDeclarations` 在防什么

[validateEnvDeclarations](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_env_declare.go) 是安装侧把 agent 输出收缩成可信元数据的核心闸门。

它按三层过滤：

1. 名字格式必须像 env 名
2. 这个名字必须真的出现在 bundle 某个文件里
3. 名字不能撞上 sandbox 自己保留的变量

其中第二层特别关键。

因为 installer agent 很容易“合理想象”出一个凭据名，比如：

- `OPENAI_KEY`
- `SERVICE_TOKEN`

但 skill 代码里可能从头到尾根本没读过它。

如果不做 bundle literal match，设置页就会出现一堆其实没被代码消费的变量，长期把用户往错误方向带。

## 7. 为什么 reserved env 不是安全边界，而是自保边界

保留名检查并不是在防 skill 逃逸。

skill 本来就已经能在自己的 sandbox 里执行任意代码。这里防的是另一类更难排查的问题：

- 用户把 `PATH`、`PYTHONPATH`、`WEKNORA_SKILL_OUTPUT_DIR` 之类关键名字填成自己的值
- 结果 skill 本身的运行环境被悄悄改坏

这种错误从现象上看很像“系统随机坏了”，实际却只是声明层把不该暴露的名字放进了设置页。

## 8. `storeEnvDeclaration` 真正写回 row 时在防什么

[storeEnvDeclaration](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2401-L2420) 只会在下面条件还成立时落库：

- 这条 skill row 仍然是 `ready`
- `BundleSHA256` 仍然等于本次安装的 bundle

它防的是典型并发窗口：

- 安装 A 快结束时读到了声明
- 这时管理员又上传了新版本，或者 remove 已经排队接管这条 row

如果不重验版本归属，就可能把旧 bundle 的声明写到新 bundle 的 row 上。

## 9. skill 什么时候真正进入 `AgentConfig`

这一跳发生在 [session_agent_qa.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_agent_qa.go#L366-L397)。

这里不会直接把“数据库里所有 ready skills”塞进 `AgentConfig.TenantSkills`，而是先调用：

- `skillsForRun`
- `effectiveTenantSkills`

把运行时可见集合收窄好以后，再放进 `AgentConfig`。

所以后面 agent 侧所有技能可见性逻辑，其实都建立在这份已经过筛的列表上。

## 10. 为什么运行时工具面要分成 workspace 文件工具和 skill image 文件工具

[registerSandboxFileTools](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L410-L470) 做的是普通会话的文件工具暴露：

- `list_sandbox_files`
- `read_file`
- `write_sandbox_file`
- `edit_sandbox_file`

它们面向的是：

- `/workspace`
- `/workspace/input`
- 会话自己的工作目录

而 install mode 不走这套，因为 installer 真正需要写的是：

- skills image root

所以 install mode 会切到：

- `write_skill_file`
- `edit_skill_file`

这两套工具虽然都叫“文件工具”，但改的不是同一片文件系统语义。

## 11. 为什么普通 `shell_exec` 和 install-mode `shell_exec` 必须分开

[registerSandboxShellTool](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L953-L1000) 会给不同场景挂不同 shell：

### install mode

- root 权限
- 允许写 skills image root
- 目标是把依赖安装进镜像

### 普通会话

- session shell
- 作用于当前会话的 sandbox
- 额外挂上 `userEnvResolver` 和 `skillEnvCapture`

如果把两者混成一个工具，就会把“安装时的特权能力”直接漏给普通 agent 回合。

## 12. `userEnvResolver` 解决的不是“技能筛选”，而是“值怎么叠”

[userEnvResolver](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L811-L855) 和 [NewUserEnvResolver](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/user_env_resolver.go) 容易被误解成“再筛一次哪些 skill 可见”。

其实不是。

skill 可见性在更早的：

- `skillsForRun`
- `effectiveTenantSkills`

已经决定了。

`userEnvResolver` 只负责在这份已经可见的 skill 集合上，为某一次执行解析出正确的 env 值。

## 13. `ResolveEnv` 的四层语义

[ResolveEnv](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/user_env_resolver.go#L52-L109) 真正把 skill env 拼成了一次执行可见的 map。

可以把它理解成四层语义：

1. skill row 上管理员填过的默认值
2. 当前用户在 config-wide scope 填过的值
3. 当前用户针对这个 skill 单独填过的值
4. sandbox create-time env_vars 只参与 required 检查，不回注到本次 env map

优先级是后者覆盖前者。

这保证了：

- 管理员可以给团队设置默认值
- 用户可以在个人 scope 覆盖它
- 某些变量即使已经 baked 进容器，也不会在每次执行时重复注入

## 14. 为什么 `shell_exec` 里的 `skill_name` 是必要参数

[shell_exec.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L199-L214) 定义了 `skill_name`。

它不是装饰字段，而是告诉运行时：

- 这次命令要用哪一个 skill 的运行时与声明 env

没有这个名字，系统只会注入 config-wide 变量，不会去解析 skill-specific 声明。

这也是为什么错误提示会明确告诉模型：

- 缺少变量时，要重新带着 `skill_name` 和 `env` 再调用一次

## 15. `shell_exec` 合并 env 时在做什么

[shell_exec.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L535-L576) 这一段是运行时 env 合并的交汇点。

它同时处理：

- 调用参数 `env`
- 命令里的 `NAME=value`
- resolver 解析出来的存储值
- required env 缺失检查

关键规则有两个：

### 规则一：模型本次显式传入的 env 优先

`ApplyResolvedEnv` 只会把 resolver 里的值补到空位上，不会覆盖本次调用显式传来的值。

这让“用户在聊天里临时给一个 key”成为可能。

### 规则二：已经从存储里解析出来的名字，不再参与 capture

也就是说，只有本次调用自己新带来的、且之前没人存过的值，才有机会被回填。

这能防止一条 hallucinated `export KEY=test` 把原本就好用的旧 secret 覆盖掉。

## 16. 为什么 `maybeCaptureSkillEnv` 只能在成功后执行

[maybeCaptureSkillEnv](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L734-L757) 只在这几个条件都成立时运行：

- 有 `skill_name`
- 有 capture hook
- 不是 install mode
- shell 命令 exit code 为 0

这意味着系统只会把“本次确实成功用上的值”补回存储。

如果命令失败了，它不会贸然把值记成“可用凭据”，避免把明显无效的测试值长期污染用户配置。

## 17. `CaptureSkillEnv` 真正承诺的是什么

[CaptureSkillEnv](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/user_env.go#L233-L286) 的承诺很克制：

- 只写当前 principal
- 只写该 skill 已声明的名字
- 只补 workspace/user 还没填过的空白

它明确不承诺：

- 旋转已有值
- 覆盖管理员默认值
- 接受任意自由输入名字

也就是说，它的目标不是“自动管理 secrets”，而是“把本次已经验证可用的临时输入，安全地补成下次可复用的最小状态”。

## 18. 这篇和前后文怎么接

建议按下面顺序连读：

1. [22-code-reading-skill-snapshot-commit-pointer-switch.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/22-code-reading-skill-snapshot-commit-pointer-switch.md)
2. [23-code-reading-skill-image-effective-config-session.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/23-code-reading-skill-image-effective-config-session.md)
3. 当前这篇 `24`

这样能把技能运行链完整串起来：

- 镜像怎么生成
- session 怎么选择镜像
- image 里的 skills 怎么变成 agent 真能看到、真能调用、真能复用的一组能力

## 19. 二开时最容易踩的坑

### 坑一：把 image manifest 当成安装真值

它只是镜像侧自描述，不是控制面真源。

### 坑二：让 env 声明失败直接判整次安装失败

这会把“元数据不完整”和“skill 根本没装好”混为一谈。

### 坑三：普通会话沿用 install-mode 文件/壳能力

这样会把 skills image root 的特权写能力暴露给普通 agent 回合。

### 坑四：允许 `shell_exec` 把本次显式传入的值覆盖已有存储值

这会把临时测试值或模型幻觉写成长期凭据。

### 坑五：跳过 `skill_name`，直接按命令内容猜属于哪个 skill

这会让任何提到 skill 目录的命令都有机会写进错误 skill 的凭据存储。
