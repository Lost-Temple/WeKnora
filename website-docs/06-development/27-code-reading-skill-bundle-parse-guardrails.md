# 技能 Bundle 解析、文件裁剪与校验闸门阅读指南

这一篇接着 [26-code-reading-skill-source-fetch-parse.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/26-code-reading-skill-source-fetch-parse.md) 往下读，专门讲“远程内容已经拿回来了以后，系统到底怎样把它收敛成一个可信的 `SkillBundle`”。

它要回答的不是：

- 去哪里下载

而是：

- zip 里哪些文件算这个 skill
- `SKILL.md` 到底怎么决定 skill root
- 什么情况下允许把仓库外层目录、README、LICENSE 扔掉
- 为什么还没进 sandbox，就已经做了这么多大小、路径和结构上的硬限制

## 推荐阅读顺序

1. [tenant_skill_bundle.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L1-L118)
2. [tenant_skill_bundle.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L119-L275)
3. [tenant_skill_bundle.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L276-L493)

## 1. 这条链的目标不是“成功解压”，而是“得到标准 SkillBundle”

安装后半段真正想消费的是：

- `Name`
- `Version`
- `Description`
- `Instructions`
- `Files`
- `SHA256`

也就是 [SkillBundle](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L36-L57) 这份标准结构。

所以 zip 解析链的目标并不是：

- 把用户给的压缩包都尽量解开

而是：

- 在明确结构、安全边界和 skill root 之后，生成一份可信的 `SkillBundle`

凡是不满足这个目标的 archive，宁可直接拒绝。

## 2. 为什么上传安装和远程安装复用同一套解析器

这条设计非常关键。

无论 bundle 最初来自：

- 用户手动上传 zip
- GitHub/GitLab zipball
- registry handoff
- 单个 `SKILL.md`

最后都尽量走到：

- `ParseSkillBundle`
- 或 `ParseSkillBundleWithOptions`

原因是只要把“来源解析”和“bundle 结构校验”拆开，安装系统后半段就不用再区分：

- 这是本地上传
- 还是远程下载

这让后面的 install、verify、manifest、hash 逻辑都能共享同一套 bundle 语义。

## 3. `SkillBundleParseOptions` 在放宽什么

[SkillBundleParseOptions](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L59-L75) 只给远程 archive 放宽三个点：

1. `Subdir`
2. `AllowExtraFiles`
3. `AllowNestedSkill`

它不是在放宽“安全检查”，而是在放宽“目录包装形式”。

也就是说：

- 远程仓库 zip 可以在 skill 外面多套一层 repo 根目录
- skill 旁边可以有 README / LICENSE / 其它仓库文件
- monorepo 可以把 skill 放在更深的子目录里

但这些放宽并不意味着：

- 可以越界路径
- 可以放大到无限制
- 可以带 symlink

安全边界还是原来的。

## 4. `ParseSkillBundle` 和 `ParseSkillBundleWithOptions` 的分工

[ParseSkillBundle](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L77-L85) 是上传路径的默认入口，规则更严格。

[ParseSkillBundleWithOptions](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L87-L99) 则允许远程来源传入扩展选项。

可以把两者理解成：

- `ParseSkillBundle` = 标准单-skill 上传格式
- `ParseSkillBundleWithOptions` = 远程来源适配层

但两者最后都会落到同一份 `SkillBundle` 结果上。

## 5. 解析链其实分成三道闸门

从代码结构上看，这条链大致分三步：

1. `skillZipEntries`
2. `stripSkillRootPrefix`
3. `skillBundleFromFiles`

它们分别解决不同问题：

### 第一步：zip 层的结构和资源限制

- 路径是否合法
- 条目数是否过多
- 单文件和总量是否超限

### 第二步：skill root 识别与裁剪

- 哪个目录才是 skill 本体
- skill 外面的文件要不要丢掉

### 第三步：SKILL.md 元数据解析

- `name`
- `description`
- `instructions`
- `version`

把这三层分开，是为了让错误边界清晰。

## 6. 为什么 `skillZipEntries` 先看 central directory，而不是先解压全部文件

[skillZipEntries](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L131-L180) 做的第一件事是：

- 遍历 zip central directory

它先得到：

- entry 名称
- 目录项
- FileInfo 声称的大小

而不是一上来就把所有文件内容都解出来。

这样做的好处是：

- 能先用很低成本筛掉目录项和越界路径
- 能先做 zip entry 数量限制
- 能先判断 skill root，再决定哪些文件值得真的解压

这是对 zip bomb 最基础也最重要的一层防线。

## 7. 为什么有两套数量限制：`maxSkillBundleZipEntries` 和 `maxSkillBundleFiles`

这个设计很值得注意。

在 [tenant_skill_bundle.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L22-L34) 里有两套上限：

- `maxSkillBundleZipEntries`
- `maxSkillBundleFiles`

前者限制的是：

- 原始 archive 的中央目录总条目数

后者限制的是：

- 经过 skill root 裁剪后，真正属于 skill 的文件数

两者不能合并，因为像 GitHub zipball 这种远程 archive：

- 整个仓库可能比 skill 子目录大很多

所以必须既防：

- 原始 zip 太夸张

也防：

- 最终 skill 自己的文件树太夸张

## 8. `inspectSkillZipPath` 在防什么

[inspectSkillZipPath](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L182-L193) 负责最底层的路径规范化。

它主要防的是：

- `..`
- 绝对路径
- 通过清理后仍能跳出 archive root 的名字

也就是说，它解决的是：

- “这个条目名能不能被当作 archive 内的相对路径”

而不是：

- 这个文件内容是否可信

路径边界必须在最早阶段处理，因为后面所有 skill root 判断都建立在这些名字不会越界的前提上。

## 9. 为什么 symlink 检查要放在“保留下来的 entry”这一层

[inspectKeptSkillZipEntry](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L195-L205) 只检查那些已经被认定属于 skill root 的 entry。

原因是：

- 对 skill 外面的多余文件，我们本来就可能直接丢掉
- 没必要为所有被丢弃的 entry 都付完整检查成本

而真正保留下来的 entry，才必须继续做：

- symlink 拦截
- 非法字符检查
- 单文件大小检查

## 10. 为什么 `readLimitedZipEntry` 还要再做一次大小上限

[readLimitedZipEntry](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L207-L221) 体现了一个很稳的策略：

- **相信 central directory，但绝不只相信 central directory**

前面虽然已经看过 entry 声称的 size 了，但真正读取时仍然再卡一次：

- `maxSkillBundleFileBytes + 1`

这是为了防止：

- zip 头信息和实际解压结果不一致
- 某些构造 archive 试图绕过声明大小

也就是说：

- “宣称大小”只够做预筛
- 真正落地前还要做实读校验

## 11. 为什么总量限制在两个阶段都要累计

这一点和上面同理。

`unzipSkillArchive` 在真正读取 entry 内容时会累计总字节数，哪怕前面在 `skillZipEntries` 已经累计过一遍了。

原因是：

- 前面累计的是 entry 声称大小
- 这里累计的是实际读出来的大小

只要两者存在偏差，就要用真实读取结果再兜一层上限。

这也是为什么代码里会出现：

- `actualExcess`

这类补差逻辑。

## 12. `skillRootPrefix` 为什么是整个解析链的核心

[skillRootPrefix](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L461-L493) 真正回答的是：

- “这个 archive 里，哪一层目录才算 skill 本体”

这一步几乎决定了后面一切：

- 哪些文件会被保留
- 哪些文件会被丢弃
- archive 是否会因为“多于一个 skill”而被拒绝

它的规则可以概括成三条：

1. 根目录直接有 `SKILL.md`，那根就是 skill
2. 指定了 `Subdir`，就只认那条路径命中的 `SKILL.md`
3. 否则只能唯一命中一个 skill root

## 13. 为什么“根目录有 SKILL.md”会压过嵌套的 SKILL.md

这条规则特别重要。

代码里明确写了：

- 如果 zip 根目录直接有 `SKILL.md`
- 那这个 archive 的 skill 就是根目录

更深层的 `SKILL.md` 会被当作普通额外文件，而不是第二个 skill。

这是因为真实 bundle 里完全可能 vendor 别人的 skill 文档、示例或模板。如果根目录已经明确表态“我就是一个 skill”，就没必要再被深层的 `SKILL.md` 干扰。

## 14. 为什么默认不接受更深层的嵌套 skill

默认上传路径里，如果 `SKILL.md` 藏在很深的目录里，通常意味着：

- 这不是一个直接可安装的单-skill bundle
- 而是整个仓库被拿来上传了

所以默认规则只接受：

- 根目录 skill
- 或单层包裹目录里的 skill

更深层的嵌套只有在远程来源明确传了 `AllowNestedSkill` 时才放行。

这能减少本地上传时“我把整个 monorepo 压了就传”的误用。

## 15. `stripSkillRootPrefix` 为什么是“重定根”而不是“复制一份新 skill”

[stripSkillRootPrefix](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L433-L459) 做的不是逻辑上的二次安装，而只是：

- 把 skill root 外面的前缀剥掉

举例来说，如果 archive 里是：

- `repo-main/skills/pdf/SKILL.md`
- `repo-main/skills/pdf/scripts/run.py`

重定根之后会变成：

- `SKILL.md`
- `scripts/run.py`

这样后面的 install 流程就能一直用“skill root 相对路径”这套内部语义，而不用一路背着 repo 外层路径。

## 16. 为什么 `AllowExtraFiles` 是“丢掉噪音”，不是“保留整仓库”

这点很容易误会。

`AllowExtraFiles` 的语义不是：

- skill 之外的文件也一起保留

而是：

- skill root 外面的文件不再触发报错，直接忽略

所以它解决的是：

- GitHub zipball 把 README、LICENSE、其它目录一起打进来了

并不是：

- 允许整个仓库都进入安装产物

真正留下来的，仍然只有 skill root 里的文件。

## 17. `skillBundleFromFiles` 才真正进入 SKILL.md 语义层

[skillBundleFromFiles](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L306-L327) 是结构校验之后，第一次真正关心：

- skill 名称
- 描述
- instructions
- version

在这之前，系统只是在处理：

- zip 结构
- 文件树
- 路径
- 大小

也就是说：

- **SKILL.md 语义层永远建立在 bundle 结构已经安全可接受的前提上**

这能避免一份结构有问题的 archive 先把技能文案读进来，再在更晚的阶段才报错。

## 18. 为什么 version 解析是“补字段”，不是“硬要求”

[parseSkillBundleVersion](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L386-L419) 只尝试从 frontmatter 中读 `version`。

几个很重要的点：

- 没有 frontmatter，不报错
- 有 frontmatter 但没有 version，也不报错
- 真正的前端/后端主语义校验，还是由 `skills.ParseSkillFile` 负责

这说明版本号在当前系统里是：

- 一个尽量提取的补充字段

而不是：

- 缺了就不能安装的绝对门槛

## 19. 为什么最终还要重新生成规范化 zip

[zipSkillFiles](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L347-L367) 会把 skill root 文件集重新按确定性顺序写成 zip。

这一步非常重要，因为经过：

- root 裁剪
- 额外文件剥离
- 单文件/总量检查

以后，系统真正信任的已经不是最初那份原始 archive，而是：

- 这份被规范化后的 skill-root zip

它后面会继续参与：

- SHA256 识别
- install 路径
- 文件浏览
- 缓存与复用

## 20. 为什么 `validateSkillEntryName` 只禁控制字符

[validateSkillEntryName](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_bundle.go#L369-L384) 的注释写得很透：

- 它不负责 shell 安全
- 它负责日志、错误消息和 manifest 的可打印性

也就是说，它要防的是：

- 换行伪造日志
- 控制字符污染输出

而不是：

- 一切特殊字符都不许出现

这也是为什么 `@`、`+` 这类真实依赖树里很常见的字符会被保留。

## 21. 这篇和前后文怎么接

建议按下面顺序连读：

1. [26-code-reading-skill-source-fetch-parse.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/26-code-reading-skill-source-fetch-parse.md)
2. 当前这篇 `27`
3. [22-code-reading-skill-snapshot-commit-pointer-switch.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/22-code-reading-skill-snapshot-commit-pointer-switch.md)

这样刚好串成一条完整安装前半段：

- 先把 source 字符串安全地拉成原始内容
- 再把原始内容收敛成可信 `SkillBundle`
- 然后才进入 install / snapshot / pointer switch 后半段

## 22. 二开时最容易踩的坑

### 坑一：把 `AllowExtraFiles` 理解成“整个仓库都保留”

它只是允许 skill root 外的文件被忽略，不是让它们进入安装产物。

### 坑二：只看 zip 声称大小，不看实际读出的大小

这样很容易被构造 archive 绕过上限。

### 坑三：一看到深层 `SKILL.md` 就默认挑一个继续装

没有明确 `Subdir` 或 `AllowNestedSkill` 的情况下，这会把 monorepo 误装成随机 skill。

### 坑四：把结构校验和 `SKILL.md` 语义解析揉在一起

这样错误边界会非常乱，既难调试也难给用户解释。

### 坑五：沿用原始 archive 继续往后传

一旦不把最终信任对象收敛成规范化 skill-root zip，后面的 hash、缓存和文件浏览语义都会飘。
