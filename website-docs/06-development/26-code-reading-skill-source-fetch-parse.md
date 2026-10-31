# 技能来源解析、远程获取与安全校验阅读指南

这一篇接着 [25-code-reading-skill-source-host-staging-prompt.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/25-code-reading-skill-source-host-staging-prompt.md) 往前补，把技能“还没开始安装之前”的来源解析链单独收成一篇。

也就是回答这些问题：

- 用户贴进来的 `source` 字符串到底支持哪些形态
- 为什么像 `owner/slug` 这种看似简单的输入会被拒绝为歧义
- registry / skills.sh / GitHub / direct URL 最终怎样统一落成一份标准 bundle
- 为什么这条链对 SSRF、跳转次数、响应大小和 handoff 结构这么苛刻

## 推荐阅读顺序

1. [tenant_skill_source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L1-L155)
2. [tenant_skill_source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L156-L387)
3. [tenant_skill_source.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L620-L920)

## 1. 这条链解决的不是“下载文件”，而是“把来源变成可信 bundle”

从用户视角看，安装来源可能只是一个字符串：

- `@owner/skill`
- `my-skill`
- `skills-sh:owner/repo/slug`
- GitHub URL
- GitLab URL
- 直接 zip URL
- 甚至一份裸 `SKILL.md`

但安装流程后半段真正想要的并不是“某个 URL”，而是：

- 一个已经过结构校验的 `SkillBundle`
- 一份规范化后的 zip archive

所以这条链的真正目标不是“下载远端内容”，而是：

- **把各种来源字符串收敛成可安装、可审计、可复现的标准 bundle**

## 2. `InstallSkillFromSource` 为什么要先校验 config

[InstallSkillFromSource](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L79-L103) 一开始先做的是：

- `requireSkillTarget`

这一步是在防一种很容易忽略的资源浪费：

- 如果 configID 本身就是未授权或不存在的
- 就不应该先拿这个坏请求去发一轮外网请求、下载一大包数据

所以这里先做 target 授权，后做远程 fetch，是有明确顺序意义的。

## 3. `parsedSkillSource` 为什么存在

[parsedSkillSource](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L28-L49) 的作用，是把所有输入形态先抽象成统一的“未下载来源描述”。

它里面放的是：

- `Kind`
- `Registry`
- `Slug`
- `Version`
- `Owner`
- `Repo`
- `Ref`
- `Subdir`
- `DirectURL`

也就是说，在真正下载前，系统已经先把“你这串输入到底代表什么”说清楚了。

这是后面能统一处理 handoff、GitHub zipball、registry download API 的前提。

## 4. 为什么 `parseSkillSource` 宁可拒绝歧义，也不联网试探

[parseSkillSource](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L122-L155) 有个很重要的原则：

- 不靠联网试探来猜来源种类

比如：

- `owner/slug`

它既可能像：

- ClawHub locator

也可能像：

- GitHub repo

如果此时系统采取“我都试试”的策略，就会带来几个问题：

1. 请求次数翻倍
2. 失败行为不稳定
3. 安全边界模糊
4. 同一个输入在不同网络环境下表现不一致

所以它明确要求：

- 纯 registry slug 用 `@...` 或单段 slug
- git 仓库必须给出明确 URL

这条规则看起来严格，但换来的是可预测性。

## 5. `parseSkillSourceURL` 为什么只做静态归类

[parseSkillSourceURL](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L157-L183) 只根据：

- scheme
- hostname
- path

来决定来源属于：

- GitHub
- GitLab
- skills.sh
- registry
- direct archive

它不会在这里就发请求确认。

这是因为 URL 的“语义归属”应该是一个纯本地决定，而不是和网络状态绑定。

如果把解析和请求混在一起：

- 解析阶段就会产生副作用
- 单元测试也会变重
- 某些恶意 URL 甚至可能在“只是想看看它属于哪类”时就触发外连

## 6. 为什么 SkillHub / ClawHub / 自托管 registry 还要再单独映射

`parseSkillHubCNURL` 和 `parseRegistryURL` 做的事情，都不是下载，而是把“面向人类的页面路径”转换成“面向安装链的 API 语义”。

原因是：

- 页面 URL 往往是给人浏览、SEO 或 SPA 路由用的
- 安装流程真正要的是稳定下载协议

比如 SkillHub.cn 页面看起来像：

- `/skills/{slug}`
- `/skills/{publisher}/{slug}`

但下载最终要落到：

- `api.skillhub.cn/api/v1/download?...`

这一步如果不显式做，就会把页面层和下载层混在一起，导致后面的 handoff 逻辑很难统一。

## 7. `skills.sh` 为什么要多一层 handoff

[parseSkillsShLocator](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L363-L385) 只接受固定三段式：

- `owner/repo/slug`

而且它故意不假设：

- `slug == repo 里的目录名`

这是因为 skills.sh / registry 常常需要告诉你：

- 实际 skill 位于 monorepo 的哪个子目录
- 应该钉死到哪个 commit

这些信息只有 registry handoff 才知道，用户本地字符串并不足以推出。

所以 skills.sh 的设计本质上是：

- 用户输入一个 catalog locator
- registry 再 handoff 成一个确定的 git/archive 安装目标

## 8. `fetchURL` 只是“生成下一跳目标”，不是下载

[fetchURL](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L566-L618) 负责把 `parsedSkillSource` 翻译成这一次 hop 真正要访问的 URL。

比如：

- registry slug -> `/api/v1/download?...`
- skills.sh locator -> `/api/v1/skills/{slug}/install?...`
- GitHub repo/ref -> `codeload.github.com/...`

这一步故意保持纯函数化：

- 不请求网络
- 不解析响应

这样“生成目标地址”和“实际信不信这个目标”就被拆成了两个阶段。

## 9. `fetchSkillSourceBytes` 为什么要允许多跳，但又只允许很少的跳数

[fetchSkillSourceBytes](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L630-L662) 是整条链真正执行联网的地方。

它的流程是：

1. 根据当前 source 生成 URL
2. 下载 body
3. 如果 body 已经是 zip 或 `SKILL.md`，结束
4. 如果 body 是 JSON handoff，转成下一跳 source
5. 继续递归 fetch

之所以要支持多跳，是因为像：

- registry -> GitHub archive
- registry -> archive proxy -> 最终 zip

这类链路是现实存在的。

但它又用 `skillSourceMaxHops = 3` 限得很死，是为了防：

- 无限 handoff
- 循环 handoff
- 被恶意 registry 拖进长链请求

这也是一种“允许必要灵活性，但严控资源边界”的设计。

## 10. 为什么 handoff JSON 不能“看起来差不多就接受”

[sourceFromHandoff](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L664-L698) 和它后面的 `sourceFromArchiveHandoff`、`sourceFromGitHubHandoff` 对 handoff 结构要求非常严格。

它只接受两大类结果：

1. 明确的 archive URL
2. 明确的 GitHub repo/commit/path

其它“差不多像下载提示”的 JSON，一律拒绝。

原因是：

- handoff 是跨系统信任边界
- 一旦这里放松，后面就等于让 registry 响应随意决定下一跳目标

所以这里的严格，并不是“代码洁癖”，而是显式地把可接受协议收窄。

## 11. 为什么 `sourceFromArchiveHandoff` 还要把 archive URL 再解析一遍

[sourceFromArchiveHandoff](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L717-L748) 没有直接把 handoff 里的 `archiveUrl` 当成 trusted target。

它会再做一轮：

- `parseSkillSourceURL(archiveURL)`

这是为了防两类问题：

### 问题一：scheme 不合法

比如 handoff 塞一个非 `http(s)` 的地址。

### 问题二：语义不清

比如一个格式残缺的 URL，若直接喂给后续下载层，可能会在某些库里被自动补全成意料之外的地址。

所以 archive handoff 也必须重新经过来源解析，不享受“因为是 registry 说的所以天然可信”的待遇。

## 12. `getSkillURL` 为什么是整条链唯一的网络闸门

[getSkillURL](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L783-L848) 非常关键，因为：

- SSRF 校验
- redirect 安全
- 响应大小限制
- HTTP 状态码检查
- 错误文本裁剪

全部统一收在这里。

这样做的好处是：

- 所有来源最终都通过同一套网络安全门
- 不会出现 GitHub 分支做了一套校验，registry 分支又忘了另一套

它本质上是技能远程安装链的统一网络出口。

## 13. 为什么响应大小限制要落在“下载体”而不是“skill 子目录”

这一点在注释里讲得很明确：

- GitHub zipball 返回的往往是整个 repo
- 真正的 skill 可能只是其中一个子目录

所以下载阶段能做的上限，只能是：

- 整个 HTTP body 不能太大

后面 bundle parse 阶段才会进一步收窄成：

- 真正保留为 skill 的文件集不能太大

这两个上限解决的是不同层级的问题，不能互相替代。

## 14. 为什么 `normalizeFetchedSkill` 连裸 `SKILL.md` 也能接受

[normalizeFetchedSkill](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_source.go#L850-L878) 会把两类远端结果统一成：

- `SkillBundle`
- 规范化 archive

其中一种很有意思的情况是：

- 远端只返回了一份裸 `SKILL.md`

这时系统会把它包装成最小 zip。

这样做的意义是：

- 安装后半段不需要分支判断“这次来源是不是单文件技能”
- hash、bundle 解析、后续 install 流程都还能继续复用统一逻辑

## 15. 为什么最终还要重新 zip 一次

无论远端最初给的是：

- monorepo zipball
- 直接 zip
- 单个 markdown

走完 `normalizeFetchedSkill` 之后，系统都会重新生成一份规范化 archive。

原因是安装系统真正需要的是：

- 只包含 skill 自己文件的、结构标准化的 zip

这让后续：

- 计算 SHA256
- 落库
- 生成 manifest
- 重新读取 bundle 资源

都基于同一种内部格式，而不是背着远端各种原始形态往下传。

## 16. 这条链为什么对错误文本还要做裁剪

`truncateSkillError` 看起来很小，但它解决的是另一个现实问题：

- 上游 registry / proxy 可能返回超长 HTML、traceback、甚至整页错误正文

如果这些原样透给前端或日志：

- 噪音会非常大
- 还可能把无关内容塞进用户可见错误里

所以这里把错误压到一个短、可读、可映射的范围内，是在保证交互可控。

## 17. 这篇和前后文怎么接

建议按下面顺序连读：

1. [22-code-reading-skill-snapshot-commit-pointer-switch.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/22-code-reading-skill-snapshot-commit-pointer-switch.md)
2. [24-code-reading-skill-manifest-env-runtime-injection.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/24-code-reading-skill-manifest-env-runtime-injection.md)
3. [25-code-reading-skill-source-host-staging-prompt.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/25-code-reading-skill-source-host-staging-prompt.md)
4. 当前这篇 `26`

这样可以从两个方向把技能系统连起来：

- 一边看“安装后它怎样进入 image、进入 session、进入 prompt”
- 一边看“安装前用户给的 source 是怎样被安全地拉成 bundle”

## 18. 二开时最容易踩的坑

### 坑一：把歧义输入当成“我联网猜一下”

这会让同一个输入的行为不稳定，而且扩大网络副作用面。

### 坑二：把 registry handoff 当成完全可信的下载目标

handoff 本身就是跨系统边界，仍然必须重新解析和校验。

### 坑三：只限制最终 skill 子目录大小，不限制整个下载体大小

那大仓库 zipball 仍然可能在下载阶段把资源打爆。

### 坑四：把单文件 `SKILL.md` 当特例一路特判

这样会让安装后半段充满分支，丢掉统一的 bundle/install 流程。

### 坑五：把 registry 错误原样透传

超长错误正文既不利于用户理解，也会污染日志和接口响应。
