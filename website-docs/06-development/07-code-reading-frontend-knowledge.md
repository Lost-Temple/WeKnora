# 前端知识库模块阅读指南

这份指南面向“准备改知识库前端，但还没完全熟悉代码”的场景。它不追求把每个组件都解释一遍，而是帮你先抓住主干：入口、页面、弹窗、工具层分别负责什么，状态是怎么流动的。

## 适合先知道的事实

- 知识库前端不是一个孤立页面，而是建立在全局认证、租户切换、请求封装之上。
- “当前空间”“当前知识库”“当前筛选条件”“当前上传任务配置”分别属于不同层级的状态。
- 很多地方为了避免旧请求回写，用了 generation、requestSeq、缓存失效、定时器清理这类保护。

如果一开始就只盯着某个组件细节，很容易读着读着丢失全局上下文。

## 推荐阅读顺序

### 第一步：先看全局前置状态

1. `frontend/src/router/index.ts`
2. `frontend/src/stores/auth.ts`
3. `frontend/src/utils/request.ts`

建议先搞清楚下面几件事：

- 登录态如何从本地恢复；
- `/auth/me` 为什么是会话真值来源；
- 当前租户如何影响请求头里的 `X-Tenant-ID`；
- 路由守卫为什么要为 OIDC 回跳、Lite 模式、初始化页面留特判。

如果这三处没看明白，后面很多“为什么这个页面一进来就跳转/刷新/重拉数据”都会显得很突然。

## 第二步：看知识库列表页

核心文件：

- `frontend/src/views/knowledge/KnowledgeBaseList.vue`
- `frontend/src/views/knowledge/kbListMerge.ts`

这一步建议重点看：

- “全部 / 我的 / 收藏 / 最近 / 某个组织空间”这几种视角怎么切；
- owned 和 shared 的知识库是怎么合并、去重、排序的；
- 高亮跳转参数 `highlightKbId` 为什么是一次性消费；
- 上传完成后为什么用延迟合并刷新，而不是每个上传事件都强刷一次列表。

### 这层的职责

- 拉取知识库列表和共享列表；
- 组织卡片分组与展示；
- 处理从列表跳详情、跳编辑、删除、收藏、最近访问；
- 接住上传完成后的全局刷新事件。

## 第三步：看知识库详情页

核心文件：

- `frontend/src/views/knowledge/KnowledgeBase.vue`
- `frontend/src/views/knowledge/documentSorting.ts`
- `frontend/src/views/knowledge/folderTree.ts`
- `frontend/src/views/knowledge/wikiStatusRefresh.ts`

这里是最值得花时间读的一层，因为它把页面局部状态几乎都串起来了。

### 先抓几个关键问题

#### 1. 切换知识库时，哪些状态必须整体清空

例如：

- 文档列表
- 目录树
- 已选标签
- trace 可见性缓存
- Wiki 状态轮询

这里如果没清干净，就会出现“上一份 KB 的 UI 状态残留到下一份 KB”。

#### 2. 文档列表为什么经常回第一页

因为下面这些行为都会改变结果集：

- 搜索关键词变化
- 标签筛选变化
- 目录变化
- 上传完成
- 排序变化

如果还保留旧分页游标，用户看到的内容很容易和预期不一致。

#### 3. Wiki 状态为什么单独轮询

文档处理完成，不代表 Wiki 整理、摘要生成、图谱抽取都结束了。知识库详情页把这块状态单独管理，是为了让文档列表和 Wiki 状态各自按自己的节奏刷新，而不是互相绑死。

## 第四步：看文档列表和上传弹窗

核心文件：

- `frontend/src/views/knowledge/components/DocumentListView.vue`
- `frontend/src/views/knowledge/components/UploadConfirmDialog.vue`
- `frontend/src/views/knowledge/components/KbFolderTree.vue`

### `DocumentListView.vue`

建议重点看：

- 多选状态如何和父层同步；
- 行菜单里的“移动到文件夹 / 移动到知识库”为什么是局部子流程，而不是再开全局弹窗；
- 状态标签为什么先做成 `statusByRow` 索引。

### `UploadConfirmDialog.vue`

这是“本次任务参数”的集中编辑器，不是知识库默认配置页。

重点看：

- 普通上传、手动录入、重解析三种模式如何共用一套 UI；
- `buildProcessOverrides()` 为什么只生成本次任务配置；
- reparse 为什么要把旧文档的 `processOverrides` 反向回填到 UI；
- 为什么图片/音频上传会强依赖多模态模型或 ASR 模型配置。

### `KbFolderTree.vue`

重点看：

- 展开状态为什么用 `Set<string>`；
- 重命名为什么本质上是“改最后一段路径”；
- 为什么要保证选中目录的祖先路径始终展开。

## 第五步：看编辑和设置弹窗

核心文件：

- `frontend/src/views/knowledge/KnowledgeBaseEditorModal.vue`
- `frontend/src/views/knowledge/settings/DataSourceEditorDialog.vue`
- `frontend/src/views/knowledge/settings/KB*.vue`
- `frontend/src/views/knowledge/settings/GraphSettings.vue`
- `frontend/src/views/knowledge/settings/KnowledgeBaseActivitySettings.vue`

### 阅读时重点关注

- 编辑态和创建态的 payload 为什么不同；
- 为什么有些配置是知识库持久配置，有些只是本次上传配置；
- 为什么有些弹窗在打开时要重新水合本地副本；
- 为什么数据源编辑要拆成“主配置更新”和“凭据替换”两步。

## 第六步：看 Wiki 专项组件

核心文件：

- `frontend/src/views/knowledge/wiki/WikiBrowser.vue`
- `frontend/src/views/knowledge/wiki/WikiRevisionDrawer.vue`
- `frontend/src/views/knowledge/wiki/wikiDirectoryState.ts`

### `WikiRevisionDrawer.vue`

这块建议重点看：

- 历史版本列表为什么按需分页加载；
- 点击版本后为什么正文快照懒加载；
- `detailRequestSeq` 和 `diffRequestSeq` 分别在防什么；
- 快照缺失时为什么要降级展示，而不是直接报错。

## 一个实用的阅读路径

如果你准备改“上传文档并配置处理参数”，可以这样读：

1. `KnowledgeBase.vue`
2. `UploadConfirmDialog.vue`
3. `KBParserSettings.vue`
4. `KBChunkingSettings.vue`
5. `KBAdvancedSettings.vue`
6. 后端对应上传接口与处理任务逻辑

如果你准备改“知识库列表和共享视图”，可以这样读：

1. `KnowledgeBaseList.vue`
2. `kbListMerge.ts`
3. `auth.ts`
4. 组织与共享知识库相关 API / store

如果你准备改“Wiki 版本和目录树”，可以这样读：

1. `WikiBrowser.vue`
2. `wikiDirectoryState.ts`
3. `WikiRevisionDrawer.vue`
4. `wikiStatusRefresh.ts`

## 当前最值得记住的几个边界

- `auth store` 里的角色/租户主要用于 UI 显示，不是安全真值。
- `/auth/me` 比本地缓存更接近真实会话。
- 上传确认弹窗改的是“本次任务配置”，不是“知识库永久配置”。
- 切换 KB、切版本、切数据源时，旧请求和旧定时器都必须失效。
- shared KB 和 owned KB 在列表层必须先去重，否则会出现重复卡片和错误 key。

## 和源码配合的方式

建议你后面阅读时，按下面方式一起看：

1. 先看这份指南，知道应该打开哪些文件。
2. 再看对应文件里的中文注释，理解为什么这么实现。
3. 最后再顺着事件、接口和 store 跳转到具体细节。

这样比从组件模板一路盲点进去，效率会高很多。
