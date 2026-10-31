# 技能 Install Verify、Runtime Verify 与 Repair Round 阅读指南

这一篇接着 [31-code-reading-host-skill-version-tree.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/31-code-reading-host-skill-version-tree.md) 往后读，专门讲技能安装里最核心的一条验收链：

- installer agent 负责把依赖装起来、把运行时前置条件识别出来
- 服务端 gate 负责独立复核“这棵树现在是否真的可以被接受为 ready”
- 如果失败但仍可修，repair round 会继续在同一安装上下文里补齐

这条链的设计目标很明确：

- **不相信“agent 说装好了”**
- **也不把所有判断都压给静态 gate**

## 推荐阅读顺序

1. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L951-L1215)
2. [tenant_skill_verify.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_verify.go#L14-L520)
3. [tenant_skill_runtime_verify.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_runtime_verify.go#L10-L111)
4. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1991-L2076)
5. [tenant_skill_host.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L350-L392)

## 1. 为什么 install 不是 agent 结束就算成功

从 [installDependenciesAndVerify](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L981-L1035) 可以看出，安装 verdict 分成两段：

1. installer agent 跑完一轮
2. 服务端自己执行 `verifySkill`

只有两者都通过，install 才会继续往下走：

- `verified`
- snapshot / activate
- write ready row

所以项目这里明确拒绝一种常见偷懒设计：

- “模型说它装好了，就直接把技能标成 ready”

## 2. 为什么 install prompt 本身就写进了验收标准

[buildInstallPrompt](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1991-L2076) 和 [buildHostInstallPrompt](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_host.go#L350-L392) 都不是普通“请帮我安装一下”。

它们在 installer 开始前就把标准固定死了：

- Python 依赖必须进 `.venv`
- Node 依赖必须进 `node_modules`
- 入口导入必须由 agent 亲自跑起来证明
- 运行时前置条件必须写成 `.weknora/install-report.json`
- `requirements.json` 只能声明运行时 env，不能写值

这意味着 installer agent 并不是自由决定“算不算成功”，它只是：

- **先按照平台写死的验收协议去准备环境**

## 3. 为什么验收要分成“静态 gate”和“运行时前置条件 gate”

[verifySkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_verify.go#L66-L99) 的结构很清楚：

1. `verifySkillTree`
2. `verifyDeclaredDependencies`
3. `verifyScriptsParse`
4. `verifyRuntimePrerequisites`

前 3 项主要是：

- 文件树结构
- 依赖安装位置
- 语法和 manifest 对账

最后一项则是：

- CLI 是否真的可执行
- 是否仍存在 blocker

也就是说，服务端把“静态可判定的底线”和“agent 运行时收集的事实”拆成了两层 gate。

## 4. `verifySkillTree` 守的是什么底线

[verifySkillTree](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_verify.go#L101-L139) 解决的是最朴素但也最关键的问题：

- agent 最后留下的 skill tree 还是不是当初那份 bundle 应有的树

因为 installer 拿着 root shell 在技能目录里跑：

- 不是“服务端刚 seed 进去过”就能说明文件还在

这一步会统一检查：

- `SKILL.md`
- 所有 bundled script

如果连树本身都不完整，后面的 dependency / runtime 验收都没意义。

## 5. 为什么 `verifyDeclaredDependencies` 只检查依赖树“存在”，不检查每个包内容

[verifyDeclaredDependencies](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_verify.go#L165-L186) 看起来很轻，只检查：

- `.venv/bin/python`
- `node_modules/`

这是故意的。

它回答的是：

- installer 是否至少把依赖装进了正确隔离位置

而不是：

- 每个 package 是否都完整、正确

后者交给语言级 verify 和 runtime verify 去兜，因为它们更接近实际运行语义。

## 6. `verifyScriptsParse` 为什么只做 parse，不做 import 执行

[verifyScriptsParse](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_verify.go#L188-L222) 的三个 pass 分别是：

- Python
- Node
- Shell

它们只做：

- `ast.parse`
- `node --check`
- `bash -n`

而不真正执行 skill 代码。

原因是 import 解析属于动态运行事实：

- Python 的 `sys.path`
- wrapper 脚本的入口逻辑
- runtime 时才出现的 CLI

这些都不是静态 parse 能可靠决定的。

所以项目这里非常克制：

- 静态 gate 只守“语法和 manifest 能复核的事”
- 真运行证明留给 installer agent

## 7. 为什么 Python pass 会区分 entry 和 auxiliary scripts

`verifyScriptsParse` 里对 Python 调了：

- [splitAuxiliaryScripts](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_verify.go#L430-L459)

原因是：

- `tests/`
- `examples/`
- `docs/`

这些文件经常 import 一堆开发时依赖，但它们并不代表技能真正提供给用户的入口。

所以这类文件出问题，项目的处理是：

- 记 note
- 不直接拒绝 install

这体现了一个很明确的价值判断：

- 验收的是“技能能不能服务”
- 不是“仓库里的每一份辅助文件都要像生产入口一样完美”

## 8. `execVerify` 为什么必须走普通执行路径

[execVerify](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_verify.go#L224-L270) 故意不走一条“特权验证捷径”。

它仍然使用：

- 普通 shell exec 路径
- 同样的工作目录
- 同样的 skill 环境变量

这样验证看到的环境，才尽量接近后续 session 真实会看到的环境。

否则就会出现一种很糟的假阳性：

- “验证环境里能过”
- “真实运行环境里却不行”

## 9. `skillInstallRuntimeInstructions` 为什么要强制写 `install-report.json`

[skillInstallRuntimeInstructions](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_runtime_verify.go#L12-L39) 要求 installer agent 最终写出：

- `commands`
- `blockers`

这份文件的价值不是留日志，而是把 agent 收集到的运行时事实变成：

- **服务端可独立复核的声明**

这样服务端后面就能明确问两件事：

1. 你说需要的命令，真的存在吗？
2. 你承认还有 blocker，那为什么要把这个技能标成 ready？

## 10. `verifyRuntimePrerequisites` 真正兜的是什么

[verifyRuntimePrerequisites](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_runtime_verify.go#L51-L111) 主要做三类检查：

### 第一类：报告格式是否合法

- JSON 是否可读
- `commands` / `blockers` 是否是数组
- 数量和命名是否合理

### 第二类：是否存在未解决 blocker

只要 `blockers` 非空，就直接拒绝 ready。

### 第三类：命令是否真的可执行

对每个 `commands` 都再跑一次：

- `command -v`

这说明 runtime verify 不接受“agent 自己说已经安装了某个命令”作为最终证据。

## 11. 为什么 repair round 只能围着 gate 打转

当 `verifySkill` 失败但返回的是 repairable finding 时，[installDependenciesAndVerify](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1010-L1035) 会：

1. 记录当前失败
2. 发布 `repairing`
3. 用 [buildRepairPrompt](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1182-L1215) 生成下一轮 prompt

这里很关键的一点是：

- repair prompt 直接复用 gate 原始 findings
- 不重新写一份解释
- 不让 installer 自己再总结一遍需求

这样是为了避免：

- gate 真正要修的是 A
- 中间 prompt 又把它转述成 B

导致 repair round 朝偏了方向继续折腾。

## 12. 为什么只有“repairable”失败才给第二轮机会

`skillVerificationError.Repairable` 只在一种情况下为真：

- 这次失败看起来像“再装点缺失依赖就能补齐”

如果失败属于这些类型，就不会继续 repair：

- 语法错误
- skill tree 已损坏
- runtime blocker 仍存在
- 文件/目录权限或结构已经不对

因为这些不是“再 pip/npm 一轮”能自动修复的，继续 repair 只是在浪费时间。

## 13. 为什么 `verified` 阶段在 snapshot / activate 之前

在远端 install 和 host install 两条链里，`verified` 都出现在：

- 所有 gate 都已经通过
- 但真正的 snapshot / activate / ready row 还没发生

这说明 `verified` 的语义是：

- **安装内容已经被接受**

而不是：

- **运行时指针已经切过去**

后者还属于提交链，而不是验收链。

## 14. 为什么 verification notes 不会被丢掉

[reportVerificationNotes](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L1162-L1178) 会把 note：

- 写日志
- 发进度事件

但不会把 install 判失败。

这些 note 通常是：

- 辅助文件有问题
- 某些 marker 在当前环境无法严格判断
- 某个 import 只在辅助目录里出现

它们不构成“技能不能服务”的证据，但对二开排查很有帮助，所以项目选择：

- 不拿它们卡 ready
- 也不静默吞掉

## 15. 远端与 host 的 verify 链为什么其实是同一套

虽然 host install 和远端 install 的提交介质不同，但在验收这一层它们复用的是同一套主链：

- 都用 install prompt 先写死安装与证明义务
- 都走 `installDependenciesAndVerify`
- 都经过 `verifySkill`
- 都把 runtime prerequisites 写进报告

真正不同的只是：

- host 用本地版本目录与 activate
- remote 用 snapshot 与 image pointer

也就是说，项目把“如何验收一棵技能树”尽量做成了提交介质无关的逻辑。

## 16. 这篇和前后文怎么接

建议按下面顺序连读：

1. [24-code-reading-skill-manifest-env-runtime-injection.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/24-code-reading-skill-manifest-env-runtime-injection.md)
2. [30-code-reading-skill-file-browser-and-skill-read.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/30-code-reading-skill-file-browser-and-skill-read.md)
3. [31-code-reading-host-skill-version-tree.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/31-code-reading-host-skill-version-tree.md)
4. 当前这篇 `32`
5. [22-code-reading-skill-snapshot-commit-pointer-switch.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/22-code-reading-skill-snapshot-commit-pointer-switch.md)

这样能把技能安装从“文件与环境如何长出来”一路读到“安装完成到底凭什么被接受”：

- 运行时 env 和工具注入
- 技能资源如何被读取
- host 版如何提交
- install / runtime verify 如何验收
- 最后再切回 snapshot / pointer 提交链

## 17. 二开时最容易踩的坑

### 坑一：把 installer agent 的自然语言总结当成最终成功证据

项目这里从设计上就不接受这种证据。

### 坑二：把所有检查都塞进静态 gate

真正的入口导入、CLI 可用性、外部 blocker 属于运行时事实，不该假装静态可判。

### 坑三：repair round 自己重新总结“应该修什么”

最可靠的 brief 就是 gate 原始 findings，本地再翻译一层很容易跑偏。

### 坑四：verification 通过就立刻视作 ready 已生效

verify 只是验收通过，真正生效还要经过 snapshot / activate / ready row 写回。

### 坑五：把 note 当成必须拒绝 install 的错误

很多 note 是辅助文件或环境 marker 的提示，不构成服务失败证据。
