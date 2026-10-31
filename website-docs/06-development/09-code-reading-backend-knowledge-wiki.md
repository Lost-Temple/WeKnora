# 后端知识库与 Wiki 链路阅读指南

这份指南接在“后端认证与租户链路阅读指南”后面，目标是帮你把知识库、文档处理、Wiki 页面这条后端主链快速串起来。

如果你正在读前端知识库模块，这篇文档就是它在后端的对应地图。

## 先建立一个总图

后端这条链大致可以拆成四层：

1. `internal/router/routes_knowledge.go`
2. `internal/handler/knowledgebase.go` / `knowledge.go` / `wiki_page.go`
3. `internal/application/service/knowledgebase.go` / `knowledge_process.go` / `wiki_page.go` / `wiki_ingest*.go`
4. `internal/application/repository/knowledge*.go` / `wiki_page.go`

### 可以先这样理解分工

- `router`：声明这条 API 归谁管、需要什么 guard、API key 有没有资格进来
- `handler`：把 HTTP 请求翻译成业务调用，补齐上下文、鉴权、参数整理
- `service`：真正的业务规则和异步流水线
- `repository`：数据库读写与持久化细节

## 推荐阅读顺序

### 第一步：先看路由声明

文件：

- `internal/router/routes_knowledge.go`

这里建议重点看四组路由：

- `RegisterKnowledgeBaseRoutes`
- `RegisterKnowledgeTagRoutes`
- `RegisterWikiPageRoutes`
- `RegisterKnowledgeBaseActivityRoutes`

### 这一层最值得看什么

- 哪些接口是 `Viewer` 就能读
- 哪些接口需要 `OwnedKBOrAdmin`
- 哪些接口额外挂了 `KBAccessRead/Write`
- API key 是 `retrieve`、`ingest`，还是 `manage_kbs`

这一步先不要急着读实现，先把“访问边界”看出来。

## 第二步：看知识库 handler

文件：

- `internal/handler/knowledgebase.go`

### 先看这几个入口

1. `validateAndGetKnowledgeBase`
2. `GetKnowledgeBase`
3. `ListKnowledgeBases`
4. `UpdateKnowledgeBase`
5. `DeleteKnowledgeBase`

### 为什么先看 `validateAndGetKnowledgeBase`

因为它是知识库 handler 的权限入口枢纽，会把这些问题一次性解出来：

- 这是自有 KB、共享 KB，还是通过共享 Agent 暴露出来的 KB
- 当前请求对这个 KB 的有效 tenant 是谁
- 当前调用者对它的权限级别是什么
- API key allow-list 有没有把它挡住

后面很多接口不是“重新做一遍权限判断”，而是在这个结果上继续加更细的业务限制。

### `ListKnowledgeBases` 读什么

这里要分清两条支线：

- 默认分支：列当前空间能看到的 KB
- `agent_id` 分支：列某个共享 Agent 能带出来的 KB

前端知识库列表页、Agent @ 提及、共享 Agent 可见 KB，这几种场景最后都在这里汇合。

## 第三步：看知识处理 handler

文件：

- `internal/handler/knowledge.go`

这份文件很大，不建议通读。先抓几个高价值入口：

- `ReparseKnowledge`
- `BatchReparseKnowledge`
- `MoveKnowledge`
- `MoveKnowledgeToFolder`
- 相关上传/删除入口

### `ReparseKnowledge` 在解决什么问题

它不是简单“再跑一次解析”，而是：

- 校验这条知识是否允许重新进入写路径
- 可选接收一次性的 `process_config` 覆盖
- 把知识重新送回异步处理流水线

这和前端上传确认弹窗里的“本次任务参数”是直接对应的。

### `MoveKnowledge` 为什么值得读

因为它把很多边界都压在一起了：

- 源 KB 和目标 KB 必须分别校验
- 两边都要满足 owner/admin 级限制
- API key allow-list 要同时覆盖两边
- 只能移动已完成解析的知识
- 真正执行移动是异步任务，不是同步改几行记录

如果你后面想改“跨知识库迁移”体验，这个入口几乎必看。

## 第四步：看 Wiki handler

文件：

- `internal/handler/wiki_page.go`

建议先看：

- `ListPages`
- `ListFolders`
- `MovePage`
- `RevertPage`

### `ListPages` 读什么

前端 WikiBrowser 里有两种常见调用方式：

- 目录/侧栏懒加载：更依赖 `page_type`、`folder_id`、`category_path`
- 主内容区筛选：更依赖 `query`、`status`、`sort_by`

handler 会把这些不同来源的参数统一压成 `WikiPageListRequest` 再给 service。

### `MovePage` 和 `RevertPage` 的边界

- `MovePage` 是结构调整，只改目录归属和派生层级缓存
- `RevertPage` 是一次新的常规编辑，用旧版本内容生成新当前版本

这两个操作从 UI 上都像“修改页面”，但业务语义完全不同。

## 第五步：进入 service 层

### 1. 知识库主服务

文件：

- `internal/application/service/knowledgebase.go`

适合先看：

- `CreateKnowledgeBase`
- `UpdateKnowledgeBase`
- 搜索相关主入口

这里重点理解：

- 创建 KB 时默认值从哪里补
- storage backend / vector store 怎么校验
- CreatorID 为什么只对真实用户记录，不给 synthetic API-key user 记录

### 2. 知识处理流水线

文件：

- `internal/application/service/knowledge_process.go`
- `internal/application/service/knowledge_post_process.go`
- `internal/application/service/knowledge_process_config.go`

这组文件是“上传文档 -> 解析 -> chunk -> embedding -> summary / graph / wiki 后处理”的核心地带。

如果你准备改下面这些功能，就必须看这里：

- 文档重解析
- chunk 切分
- 自动摘要
- 问题生成
- 图谱后处理
- Wiki 跟随生成

### 这层最容易误判的点

知识进入 `processing` 并不等于只是“文档解析中”。

在这个项目里，文档主解析完成后，后面还可能挂着：

- multimodal 子任务
- summary
- question generation
- graph extraction
- wiki enqueue / wiki ingest

所以不要轻易把某个中间状态当作“全链路已结束”。

### 3. Wiki 页面服务

文件：

- `internal/application/service/wiki_page.go`

这里建议重点看：

- `CreatePage`
- `UpdatePage`
- `UpdatePageMeta`
- `UpdateAutoLinkedContent`
- `RevertPageToVersion`

#### 读这层时重点记住

- `version` 只反映“用户可见内容变化”，不是所有数据库写入都会递增
- `FolderID` 和 `CategoryPath` 不是并列真值，后者更像派生缓存
- 出链 / 入链会随正文一起重算
- 自动补链、修链这类机器写入不应该随便 bump 用户版本号

### 4. Wiki 生成流水线

文件：

- `internal/application/service/wiki_ingest.go`
- `internal/application/service/wiki_ingest_batch.go`
- `internal/application/service/wiki_ingest_followup.go`
- `internal/application/service/wiki_ingest_taxonomy.go`

这块是项目里最复杂的异步链路之一。第一次读时建议只抓主线：

- 哪些知识会进入 Wiki ingest
- 如何按 KB 和 slug 控制并发
- 为什么需要 follow-up batch
- 页面 slug / identity / reduce lock 分别在防什么竞争

别一开始就试图把所有并发常量都记住，不然很容易淹没在细节里。

## 一个实用阅读路径

### 如果你准备改“上传后自动生成 Wiki”

1. `frontend/src/views/knowledge/KnowledgeBase.vue`
2. `frontend/src/views/knowledge/components/UploadConfirmDialog.vue`
3. `internal/handler/knowledge.go`
4. `internal/application/service/knowledge_process.go`
5. `internal/application/service/knowledge_post_process.go`
6. `internal/application/service/wiki_ingest*.go`

### 如果你准备改“Wiki 页面列表、目录、版本回滚”

1. `frontend/src/views/knowledge/wiki/WikiBrowser.vue`
2. `frontend/src/views/knowledge/wiki/WikiRevisionDrawer.vue`
3. `internal/handler/wiki_page.go`
4. `internal/application/service/wiki_page.go`
5. `internal/application/repository/wiki_page.go`

### 如果你准备改“知识库列表 / 共享可见性 / 权限”

1. `frontend/src/views/knowledge/KnowledgeBaseList.vue`
2. `internal/handler/knowledgebase.go`
3. `internal/router/rbac.go`
4. `internal/router/routes_auth_tenant.go`
5. `internal/router/routes_knowledge.go`

## 当前最值得记住的几个边界

- KB 可见，不等于 KB 可写。
- KB 可写，不等于 KB 可删除。
- Wiki 页面移动，不等于 Wiki 页面正文编辑。
- 重解析可带本次任务覆盖，但不等于修改知识库默认配置。
- `version` 是用户可见修订号，不是所有内部更新次数。

## 接下来适合继续读什么

读完这篇后，下一步比较适合继续往下接：

1. 知识库搜索与混合检索
2. 数据源同步与导入链路
3. Agent 会话里的知识库检索
4. Wiki 自动修复与 lint

这样就能把“知识库作为数据源”和“知识库作为对话检索来源”两条线完全串起来。
