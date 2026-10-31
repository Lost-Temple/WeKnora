# 代码阅读指南总览

这组文档是给“准备二次开发、先读懂代码”的场景准备的。目标不是替代源码，而是帮你更快建立全局地图：先看哪几层、各层职责是什么、哪些文件是状态真值来源、哪些地方最容易误判。

当前第一批已经覆盖知识库前端主链路，后续可以继续按模块补充。

## 阅读建议

建议按下面顺序读：

1. 先看[架构总览](../02-architecture/01-overview.md)和[开发指南](01-dev-guide.md)，确认仓库分层和本地运行方式。
2. 再看本页后面的“推荐阅读顺序”，知道某个模块的入口文件和核心状态文件在哪里。
3. 最后配合源码中的中文注释阅读具体实现，重点看“为什么这么做”和“哪些状态不能混用”。

## 推荐阅读顺序

### 1. 先建立全局入口感

- 前端入口与路由：
  - `frontend/src/router/index.ts`
  - `frontend/src/utils/request.ts`
  - `frontend/src/stores/auth.ts`
- 推荐先看这三处，因为登录恢复、租户切换、路由守卫、请求头注入都在这里收口。

### 2. 再看业务主页面

- 知识库列表页：`frontend/src/views/knowledge/KnowledgeBaseList.vue`
- 知识库详情页：`frontend/src/views/knowledge/KnowledgeBase.vue`
- 这两页决定了大部分“用户在页面上做的操作，最后怎么落到接口和状态里”。

### 3. 再看弹窗与设置页

- 知识库编辑弹窗：`frontend/src/views/knowledge/KnowledgeBaseEditorModal.vue`
- 上传确认弹窗：`frontend/src/views/knowledge/components/UploadConfirmDialog.vue`
- 数据源弹窗：`frontend/src/views/knowledge/settings/DataSourceEditorDialog.vue`
- 这些文件通常承接“表单状态 -> API 请求”的映射，二开时最容易改到。

### 4. 最后看纯逻辑工具层

- 目录树：`frontend/src/views/knowledge/folderTree.ts`
- 列表合并：`frontend/src/views/knowledge/kbListMerge.ts`
- 文档排序：`frontend/src/views/knowledge/documentSorting.ts`
- Wiki 目录状态：`frontend/src/views/knowledge/wiki/wikiDirectoryState.ts`
- Wiki 状态轮询：`frontend/src/views/knowledge/wikiStatusRefresh.ts`
- 这些文件通常最适合快速理解规则，因为不依赖具体 UI。

## 当前已整理的阅读指南

- [前端知识库模块阅读指南](07-code-reading-frontend-knowledge.md)
- [后端认证与租户链路阅读指南](08-code-reading-backend-auth-tenant.md)
- [后端知识库与 Wiki 链路阅读指南](09-code-reading-backend-knowledge-wiki.md)
- [检索与数据源同步阅读指南](10-code-reading-search-datasource.md)
- [会话问答、引用与时间线阅读指南](11-code-reading-session-rag-references.md)
- [Agent 会话、Artifact 与续播阅读指南](12-code-reading-agent-artifacts-continue-stream.md)
- [Artifact 下载、删除与产物库阅读指南](13-code-reading-artifact-library-delete.md)
- [会话 Fork、Rewind 与分叉状态阅读指南](14-code-reading-session-fork-rewind.md)
- [聊天会话状态、Steer、Stop 与 Resume 阅读指南](15-code-reading-chat-session-steer-stop-resume.md)
- [后端 Session Stream、Continue-Stream 与 Complete 阅读指南](16-code-reading-backend-session-stream.md)
- [会话 QA 入口、配置型技能引导与安装时间线阅读指南](17-code-reading-session-qa-skill-guidance.md)
- [技能安装 Transcript、Guidance 与 Repair Round 阅读指南](18-code-reading-skill-install-transcript-guidance.md)
- [技能安装/删除进度流与状态迁移阅读指南](19-code-reading-skill-progress-state-transitions.md)
- [技能安装 Stop、Stuck-Run Reaper 与恢复路径阅读指南](20-code-reading-skill-stop-reaper-recovery.md)
- [技能 Snapshot Ledger、Prune 与 Reconcile 阅读指南](21-code-reading-skill-snapshot-ledger-prune-reconcile.md)
- [技能 Snapshot Commit 与 Pointer Switch 阅读指南](22-code-reading-skill-snapshot-commit-pointer-switch.md)
- [技能 SkillImage、Effective Config 与 Session 启动阅读指南](23-code-reading-skill-image-effective-config-session.md)
- [技能 Manifest、环境变量声明与运行时注入阅读指南](24-code-reading-skill-manifest-env-runtime-injection.md)
- [技能 Source、Host Staging 与 Prompt 注入阅读指南](25-code-reading-skill-source-host-staging-prompt.md)
- [技能来源解析、远程获取与安全校验阅读指南](26-code-reading-skill-source-fetch-parse.md)
- [技能 Bundle 解析、文件裁剪与校验闸门阅读指南](27-code-reading-skill-bundle-parse-guardrails.md)
- [技能 Served Version、Catalog Archive 复用与 Ready 切换阅读指南](28-code-reading-skill-served-version-catalog-reuse.md)
- [技能 Remove、Served 回落与 Archive 回收阅读指南](29-code-reading-skill-remove-archive-release.md)
- [技能文件浏览、Catalog 文件查看与 `skill://` 读取阅读指南](30-code-reading-skill-file-browser-and-skill-read.md)
- [Host Skill 本地版本树、Activate 与 Sweep 阅读指南](31-code-reading-host-skill-version-tree.md)
- [技能 Install Verify、Runtime Verify 与 Repair Round 阅读指南](32-code-reading-skill-install-verify-runtime-gates.md)
- [技能 Install Transcript、Progress 与阶段时序阅读指南](33-code-reading-skill-install-transcript-progress-timeline.md)
- [技能 Install Shell、Tool 白名单与 Skill File Tools 阅读指南](34-code-reading-skill-install-tool-boundaries.md)
- [技能安装 Guidance、Steer 与 Repair Round 边界阅读指南](35-code-reading-skill-install-guidance-steer-repair-boundaries.md)
- [技能安装 Lifecycle、Run Lock、Heartbeat 与终态落盘阅读指南](36-code-reading-skill-install-lifecycle-lock-heartbeat-ready.md)
- [技能安装 Sandbox Bootstrap、目录重置与文件种子阅读指南](37-code-reading-skill-install-sandbox-bootstrap-seeding.md)
- [问答引用原文定位阅读指南](38-code-reading-chat-reference-source-location.md)
- [Source Locator 生成、重映射与补强阅读指南](39-code-reading-source-locator-generation-remap.md)
- [Locator 失效、编辑继承与检索合并边界阅读指南](40-code-reading-locator-stale-edit-merge-boundaries.md)
- [回答引用清单组装与传递阅读指南](41-code-reading-chat-reference-list-pipeline.md)

## 阅读时重点关注什么

### 状态真值来源

这个项目里有不少“本地缓存 + 接口回填 + 页面局部状态”共同存在的场景。阅读时优先判断：

- 谁是真值来源；
- 哪些只是首屏快照；
- 哪些只用于 UI 显示，不能作为安全判断；
- 哪些状态切换后必须清理旧缓存。

### 并发与旧请求回写

知识库和 Wiki 相关页面里，有很多“切页、切 KB、切版本、关闭重开弹窗”的高频操作。阅读时要特别看：

- 是否用了 generation / requestSeq / in-flight 复用；
- 旧请求返回后会不会覆盖新页面；
- 定时器和轮询在切对象时有没有清干净。

### URL、筛选和局部 UI 状态

很多页面会把状态拆成三层：

- 路由参数：便于分享链接、刷新恢复；
- 页面状态：当前列表、分页、弹窗、菜单；
- 本地缓存：用户习惯或跨页面上下文。

读代码时不要把这三层混成一层来看。

## 后续扩展建议

如果你后面还想继续补阅读指南，比较值得按下面顺序往下扩：

1. 后端认证与租户流
2. 知识库后端处理流水线
3. Wiki 后端生成与版本管理
4. Agent 会话与资源选择
5. 数据源同步链路

## 会话模块推荐连读顺序

如果你准备集中读聊天会话这条线，建议直接按下面顺序串起来：

1. [会话问答、引用与时间线阅读指南](11-code-reading-session-rag-references.md)
2. [Agent 会话、Artifact 与续播阅读指南](12-code-reading-agent-artifacts-continue-stream.md)
3. [会话 Fork、Rewind 与分叉状态阅读指南](14-code-reading-session-fork-rewind.md)
4. [聊天会话状态、Steer、Stop 与 Resume 阅读指南](15-code-reading-chat-session-steer-stop-resume.md)
5. [后端 Session Stream、Continue-Stream 与 Complete 阅读指南](16-code-reading-backend-session-stream.md)
6. [会话 QA 入口、配置型技能引导与安装时间线阅读指南](17-code-reading-session-qa-skill-guidance.md)
7. [技能安装 Transcript、Guidance 与 Repair Round 阅读指南](18-code-reading-skill-install-transcript-guidance.md)
8. [技能安装/删除进度流与状态迁移阅读指南](19-code-reading-skill-progress-state-transitions.md)
9. [技能安装 Stop、Stuck-Run Reaper 与恢复路径阅读指南](20-code-reading-skill-stop-reaper-recovery.md)
10. [技能 Snapshot Ledger、Prune 与 Reconcile 阅读指南](21-code-reading-skill-snapshot-ledger-prune-reconcile.md)
11. [技能 Snapshot Commit 与 Pointer Switch 阅读指南](22-code-reading-skill-snapshot-commit-pointer-switch.md)
12. [技能 SkillImage、Effective Config 与 Session 启动阅读指南](23-code-reading-skill-image-effective-config-session.md)
13. [技能 Manifest、环境变量声明与运行时注入阅读指南](24-code-reading-skill-manifest-env-runtime-injection.md)
14. [技能 Source、Host Staging 与 Prompt 注入阅读指南](25-code-reading-skill-source-host-staging-prompt.md)
15. [技能来源解析、远程获取与安全校验阅读指南](26-code-reading-skill-source-fetch-parse.md)
16. [技能 Bundle 解析、文件裁剪与校验闸门阅读指南](27-code-reading-skill-bundle-parse-guardrails.md)
17. [技能 Served Version、Catalog Archive 复用与 Ready 切换阅读指南](28-code-reading-skill-served-version-catalog-reuse.md)
18. [技能 Remove、Served 回落与 Archive 回收阅读指南](29-code-reading-skill-remove-archive-release.md)
19. [技能文件浏览、Catalog 文件查看与 `skill://` 读取阅读指南](30-code-reading-skill-file-browser-and-skill-read.md)
20. [Host Skill 本地版本树、Activate 与 Sweep 阅读指南](31-code-reading-host-skill-version-tree.md)
21. [技能 Install Verify、Runtime Verify 与 Repair Round 阅读指南](32-code-reading-skill-install-verify-runtime-gates.md)
22. [技能 Install Transcript、Progress 与阶段时序阅读指南](33-code-reading-skill-install-transcript-progress-timeline.md)
23. [技能 Install Shell、Tool 白名单与 Skill File Tools 阅读指南](34-code-reading-skill-install-tool-boundaries.md)
24. [技能安装 Guidance、Steer 与 Repair Round 边界阅读指南](35-code-reading-skill-install-guidance-steer-repair-boundaries.md)
25. [技能安装 Lifecycle、Run Lock、Heartbeat 与终态落盘阅读指南](36-code-reading-skill-install-lifecycle-lock-heartbeat-ready.md)
26. [技能安装 Sandbox Bootstrap、目录重置与文件种子阅读指南](37-code-reading-skill-install-sandbox-bootstrap-seeding.md)
27. [问答引用原文定位阅读指南](38-code-reading-chat-reference-source-location.md)
28. [Source Locator 生成、重映射与补强阅读指南](39-code-reading-source-locator-generation-remap.md)
29. [Locator 失效、编辑继承与检索合并边界阅读指南](40-code-reading-locator-stale-edit-merge-boundaries.md)
30. [回答引用清单组装与传递阅读指南](41-code-reading-chat-reference-list-pipeline.md)

这样可以把“引用与时间线 -> 续播与产物 -> 分叉回滚 -> 前端会话状态机 -> 后端流处理 -> 会话发送入口/技能安装引导 -> 技能安装 transcript/guidance -> 技能进度与状态迁移 -> stop/reaper 恢复链 -> snapshot ledger/prune/reconcile 底层事实判断 -> snapshot commit/pointer switch 上游写入链 -> SkillImage/effective config/session 启动消费链 -> manifest/env 声明/runtime 注入链 -> source/host staging/prompt 约束链 -> 远程来源解析与获取链 -> bundle 解析与校验闸门链 -> served version/catalog/ready 切换链 -> remove/archive 回收链 -> skill file browser / skill:// 读取链 -> host skill 本地版本树链 -> install/runtime verify 验收链 -> install transcript/progress 时序链 -> install shell/tool 边界链 -> install guidance/steer/repair round 幂等边界链 -> install lifecycle/run lock/heartbeat/ready 落盘链 -> install sandbox bootstrap/reset/seeding 链 -> 引用原文定位链 -> source locator 生成/remap/补强链 -> locator 失效/编辑继承/检索合并边界链 -> 回答引用清单组装与传递链”连续读完，不用在源码里来回跳。
