# 技能文件浏览、Catalog 文件查看与 `skill://` 读取阅读指南

这一篇接着 [29-code-reading-skill-remove-archive-release.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/29-code-reading-skill-remove-archive-release.md) 往后读，专门讲技能系统里另一条很容易被误解的链：

- 设置页里查看技能文件
- catalog 里查看定义层 bundle 文件
- agent 用 `read_file(path="skill://...")` 读取技能资源
- host skill 在 Lite 模式下又为什么看起来像另一套实现

这几个入口看起来分散，底层其实围绕同一个原则：

- **读技能资源时，优先回答“安装时那份稳定包里有什么”，而不是临时去翻 live sandbox 文件系统。**

## 推荐阅读顺序

1. [tenant_skill_files.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_files.go#L42-L362)
2. [tenant_skill_catalog.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L629-L715)
3. [tenant_source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L39-L319)
4. [read_file.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/read_file.go#L13-L230)
5. [host_skills.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/host_skills.go#L1-L29)
6. [session_agent_qa.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_agent_qa.go#L672-L690)

## 1. 为什么技能文件浏览默认不读 live sandbox

最直接的入口是：

- [ListSkillFiles](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_files.go#L62-L70)
- [ReadSkillFile](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_files.go#L76-L98)

它们都不是去读 live image 里的技能目录，而是先去找：

- 这条 skill install 对应的 archive

原因有三个：

### 第一，设置页必须在没有 sandbox 的时候也能看文件

技能可能还在 `installing`、已经 `failed`，或者当前根本没有可复用的 sandbox session。

### 第二，设置页读取不该依赖运行时环境

管理后台的文件查看是“控制面上的说明能力”，不是“会话执行能力”。

### 第三，读 image 会把“可查看”和“可执行”混成一件事

技能资源查看只需要稳定包内容，不需要真的进到容器或 VM 里。

## 2. `skillBundleArchive` 真正在回答什么

[skillBundleArchive](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_files.go#L101-L136) 回答的是：

- 这条 install row 当前最可信的技能 archive 是哪一份

它有三个层级：

### 第一层：row 自己 pin 的 `BundleRef`

这是最直接的“这条 sandbox install 当时就是用它装出来的”证据。

### 第二层：catalog definition 持有的 archive，但必须 digest 一致

这一步很关键。因为 catalog 重新注册技能时，会把定义层 archive 原地切到新版本；但某个旧 sandbox 仍可能跑着旧 image。

如果这里不校验 digest，就会出现：

- 设置页里看到的是新版文件树
- 实际 image 里跑的还是旧版技能

### 第三层：历史 row 的兼容兜底

更早的数据可能既没有 `BundleRef`，也没有 `BundleSHA256`，这时只能退回 definition 当前还能拿到的 archive。

## 3. 为什么 digest 校验不是可有可无

[sameDigestCatalogArchive](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_files.go#L151-L166) 里的 digest 校验，本质上是在保证：

- 文件浏览看到的 bundle
- 这条 install row 真正代表的 image 版本

还是同一份东西。

这条约束也直接保护了：

- 设置页里的文件浏览
- agent 侧的 `skill://` 读取

因为这两个入口都不能把新 catalog 的文件误投影给旧 install。

## 4. 为什么 catalog 文件查看和 install 文件查看不是一回事

catalog 那边的入口是：

- [ListCatalogFiles](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L629-L637)
- [ReadCatalogFile](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L639-L657)

它回答的是：

- **定义层当前保存的 bundle 里有什么**

而 install 文件查看回答的是：

- **某个 sandbox install 当时到底是从哪份 bundle 装出来的**

这两者在稳定状态下常常一样，但升级中或 catalog 已经前进时并不一定一样。

## 5. `catalogBundleArchive` 为什么要四处借引用

[catalogBundleArchive](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_catalog.go#L670-L714) 的顺序很有代表性：

1. 先看 catalog 自己是否持有 `BundleRef`
2. 再看挂着这个 `CatalogID` 的 install rows
3. 最后按老 ID / name 做兼容兜底

这说明项目的 archive 持有模型是演进过的：

- 新模型下，definition 自己持有 archive
- 老模型或迁移窗口里，install row 也可能暂时是唯一读者

所以 catalog 文件查看必须兼容这些历史状态，不能假设“archive 一定只在 catalog 上”。

## 6. agent 的 `skill://` 为什么也优先读 archive

agent 这一侧真正的入口不再是老的 `read_skill(...)`，而是：

- [read_file.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/read_file.go#L160-L230)

当路径是：

- `skill://<name>/SKILL.md`
- `skill://<name>/<relative-path>`

它就会转给技能管理器去读技能资源。

对 installed skill 来说，这条链最终会走到：

- [TenantSkillSource.LoadSkillFile](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L189-L232)
- [TenantSkillSource.ListSkillFiles](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L234-L251)

这两处同样是：

- 读 archive
- 不直接读 live sandbox

## 7. 为什么 `skill://.../SKILL.md` 不是简单读压缩包里的 `SKILL.md`

这一点很容易误判。

当 agent 读：

- `skill://<name>/SKILL.md`

时，`read_file` 不是简单把 zip 里的 `SKILL.md` 原文吐出来，而是拼了一页增强视图：

- 技能名
- 描述
- 执行提示
- 是否有 shell
- `skill://` 下还能继续读哪些 bundled files

也就是说，`skill://.../SKILL.md` 更像是：

- 技能说明主页

而不是：

- archive 里的原始 markdown 字节流

## 8. 为什么普通 `skill://<name>/<file>` 返回的路径看起来像 image 内路径

在 [TenantSkillSource.LoadSkillFile](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L213-L228) 里，返回给模型的 `SkillFile.Path` 是：

- image 内技能目录下的绝对路径

但内容本身又是从 archive 读出来的。

这是一个刻意的“双层语义”：

### 内容来源

- archive

### 后续可执行位置

- image 里的技能目录

这样模型在看完内容以后，下一步要 `shell_exec` 跑脚本时，看到的路径仍然和执行环境一致。

## 9. 为什么 `GetSkillBasePath` / `RemoteScriptPath` 又回到 image 语义

[GetSkillBasePath](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L253-L261) 和 [RemoteScriptPath](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L263-L283) 讲的是：

- 技能脚本真正会在 image 里的哪里执行

它们故意不依赖 archive 当前是不是还能下载到，因为运行时真相始终是：

- 已经写进 image 的技能目录

所以这条链和文件浏览链是互补关系：

- 看内容时优先 archive
- 真执行时认 image

## 10. 为什么 `read_file` 要把 bundled file tree 一起拼进 `SKILL.md`

在 [read_file.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/read_file.go#L183-L211) 里，读 `skill://.../SKILL.md` 时会顺手调用：

- `ListSkillFiles`

把文件树也列出来。

这一步的意义是减少模型试错：

- 模型先知道还有哪些 `reference/`、`scripts/`、`assets/`
- 再决定要不要继续读具体文件

这样就不用靠猜文件名去反复撞。

## 11. host skill 为什么看起来像另一套实现

host / Lite 模式下，技能不是通过远端 image snapshot 管理，而是落在本地版本树里。

可以从 [host_skills.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/host_skills.go#L1-L29) 看出来：

- `HostSkillTree` 负责本地版本目录
- `HostSkillInstaller` 负责把 bundle materialize 到本地 skill tree

这时技能资源不一定需要 object storage 上那份 archive 才能被读取，因为本地已经有落盘副本。

但对 agent 来说，`read_file(skill://...)` 仍然走同一个技能管理器抽象，所以：

- UI/模型入口是一致的
- 背后的资源来源可以因 skill source 不同而不同

## 12. 为什么会特别强调“host skill directory 不是 sandbox image 真相”

在 [session_agent_qa.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/session_agent_qa.go#L672-L690) 已经有一句关键说明：

- host skill directory 不是 shell_exec 在 sandbox 里真正看到的那份 image

这句话的意义是提醒二开时别把两种路径混掉：

### Host skill staging 路径

- 为了把 host skill 暂时带进 session

### Installed skill image 路径

- sandbox 里真正执行脚本、读取资源的目录

文件读取抽象可以统一，但执行真相不能混。

## 13. 为什么这条链里也有缓存

无论是后台文件浏览，还是 agent 连续几次 `read_file(skill://...)`，典型模式都是：

1. 先列文件树
2. 再开 `SKILL.md`
3. 再开 `scripts/...`
4. 再开 `reference/...`

如果每一步都重新下载整份 zip，会很浪费。

所以两边都做了缓存：

- 后台侧在 [tenant_skill_files.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_files.go#L29-L39) 和 `skillBundleArchiveCache`
- agent 侧在 [tenant_source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L27-L37) 和 `cachedBundle`

而且缓存的是：

- **压缩包本体**

不是整份解压树。

这样读取单个文件时只按需 inflate 对应条目，不会把大 skill 整包解压常驻内存。

## 14. 为什么路径校验在后台和 agent 两边都要做

后台这边有：

- [safeSkillFilePath](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_files.go#L324-L347)

agent 这边有：

- [safeSkillRelPath](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/skills/tenant_source.go#L292-L305)

两边都在做：

- 禁绝绝对路径
- 禁绝 `..`
- 禁绝逃出 skill 根目录

虽然读的是 zip 条目而不是磁盘路径，但这层校验依旧必要，因为：

- 这些相对路径最后会出现在模型可见路径和调试提示里
- 不应该让越界路径语义混进系统提示和工具返回

## 15. 这篇和前后文怎么接

建议按下面顺序连读：

1. [25-code-reading-skill-source-host-staging-prompt.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/25-code-reading-skill-source-host-staging-prompt.md)
2. [28-code-reading-skill-served-version-catalog-reuse.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/28-code-reading-skill-served-version-catalog-reuse.md)
3. [29-code-reading-skill-remove-archive-release.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/29-code-reading-skill-remove-archive-release.md)
4. 当前这篇 `30`

这样能把技能资源从“谁拥有它”一路读到“谁来展示它、谁来执行它”：

- source / host staging
- served / catalog / archive 引用
- remove 时何时释放 archive
- 最后 skill file browser 和 `skill://` 又怎样消费这些事实

## 16. 二开时最容易踩的坑

### 坑一：把设置页文件浏览当成 live sandbox 文件系统浏览

它默认读的是 archive，不是容器里的实时目录。

### 坑二：把 catalog 文件查看和 install 文件查看当成同一件事

definition 当前 bundle 和某个 sandbox 真正在跑的 bundle 可能已经分叉。

### 坑三：觉得 `skill://.../SKILL.md` 就是 zip 里原始 `SKILL.md`

实际上它是拼接过执行提示和文件树的增强视图。

### 坑四：看到 `SkillFile.Path` 是 image 路径，就误以为内容也是实时从 image 读出来的

这里的路径语义和内容来源是故意分开的。

### 坑五：把 host skill 本地目录直接当成 installed skill image 真相

host staging 只是某些模式下的来源形态，不等于远端 sandbox image 的统一真相。
