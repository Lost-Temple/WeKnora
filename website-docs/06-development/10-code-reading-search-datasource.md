# 检索与数据源同步阅读指南

这份指南面向两类常见二开场景：

1. 你想改“知识库怎么检索、怎么混合召回、怎么 rerank”
2. 你想改“外部数据源怎么同步进知识库、怎么调度、怎么查看同步结果”

这两条线都属于“运行时链路”，读起来比普通 CRUD 更容易迷路，所以建议先按本页顺序建立地图。

## 一、知识库检索链路怎么读

### 数据源推荐阅读顺序

1. `internal/handler/knowledgebase.go`
2. `internal/application/service/knowledgebase_search.go`
3. `internal/application/service/knowledgebase_search_fanout.go`
4. `internal/application/service/knowledgebase_search_fusion.go`
5. `internal/application/service/knowledgebase_search_rerank.go`
6. `internal/application/service/knowledgebase_search_results.go`
7. `internal/application/service/session_knowledge_qa.go`

## 1. 先看 handler 入口

文件：

- `internal/handler/knowledgebase.go`

重点函数：

- `HybridSearch`

### 这一层在做什么

- 校验调用者对主 KB 的访问权限
- 解析 `SearchParams`
- 校验 rerank 参数是否合法
- 解析资源 URL 改写模式
- 选择调用“纯召回”还是“召回 + rerank”

### 先记住一个边界

`/knowledge-bases/:id/hybrid-search` 是底层召回接口，不是完整问答接口。

它返回的是检索结果，不负责：

- 组织自然语言答案
- 多轮对话上下文拼装
- 工具调用
- 最终回答生成

这些事情在 session / QA 那条链上。

## 2. 再看检索主入口

文件：

- `internal/application/service/knowledgebase_search.go`

重点函数：

- `HybridSearch`
- `hybridSearchCandidates`
- `GetQueryEmbedding`

### 读这层时重点看什么

#### 多 KB 检索不是简单并集

当 `params.KnowledgeBaseIDs` 带多个 KB 时，service 会先做三件事：

1. 批量加载这些 KB
2. 逐个校验调用者是否真的有权访问
3. 校验它们是否处在兼容的 embedding 空间

第三点非常关键。不同 embedding 模型打出来的向量分数不能直接混在一起比较，所以这里会先挡掉“不兼容的多 KB 混搜”。

#### 查询向量为什么尽量只算一次

多 store / 多 group 搜索时，如果每个 group 都自己重新 embed 一次同一个 query，开销会成倍放大。所以这里会尽量先算出一份 query embedding，再往后复用。

## 3. 看多 store fan-out

文件：

- `internal/application/service/knowledgebase_search_fanout.go`

重点函数：

- `retrieveFromStores`
- `retainPartialResults`
- `paramsWithTopK`

### 这一层在解决什么问题

当一个检索请求需要跨多个 store group 执行时，它会：

- 按 group 并发 fan-out
- 给每个 group 单独设置 timeout
- 汇总结果
- 在必要时做跨引擎分数归一化

### 什么是 store group

可以先粗略理解为：

- 一组共享同一底层检索引擎 / store 绑定方式
- 且在当前调用里可以一起发起 retrieve 的 KB

fan-out 的最小单位不是单个 KB，而是这种 group。

### 这里最值得记住的点

- 一个 group 失败，会让整个搜索失败
- 但如果某个 group 返回了“部分结果 + 部分错误”，当前实现会保留结果，并把错误记日志
- 只有跨不同引擎类型混检时，才需要做 score normalization

## 4. 看向量 + 关键词如何融合

文件：

- `internal/application/service/knowledgebase_search_fusion.go`

重点函数：

- `fuseOrDeduplicate`
- `fuseWithRRF`
- `deduplicateByScore`

### 为什么不能直接比较原始 score

因为：

- 向量分数和 BM25 分数不是同一个量纲
- 不同引擎的 score 含义也不完全一致
- 多路结果拼接后，单纯按原始分数排序会失真

所以这里真正做的是：

- 单路时，做去重并保留本路的相对高低
- 混合检索时，用 RRF 这种“按名次而不是按裸分值”融合

这也是为什么混合检索链路要拆成 `fanout -> fusion -> rerank` 三层。

## 5. 再看 rerank

文件：

- `internal/application/service/knowledgebase_search_rerank.go`

建议重点看：

- rerank 什么时候触发
- rerank 之前结果集怎么裁剪
- rerank 后分数如何和前面的召回结果衔接

### 一个实用理解

前面的 hybrid search 更像“尽量把可能相关的候选找全”，
rerank 更像“在候选里重新排一遍最像用户问题的顺序”。

## 6. 最后把它接到问答链

文件：

- `internal/application/service/session_knowledge_qa.go`
- `internal/handler/session/qa.go`

如果你只看 `/hybrid-search`，会漏掉产品里真正的知识问答路径。问答链会在检索之上继续做：

- 检索范围组织
- prompt 拼装
- 答案生成
- 引用和流式输出

所以“改检索”时，最好也顺手看一眼这条调用链到底怎么消费检索结果。

## 二、数据源同步链路怎么读

### 推荐阅读顺序

1. `internal/handler/datasource.go`
2. `internal/handler/datasource_credentials.go`
3. `internal/application/service/datasource_service.go`
4. `internal/datasource/scheduler.go`
5. `internal/datasource/*connector*`
6. 与知识入库相关的 `knowledge` service

## 1. 先看 data source handler

文件：

- `internal/handler/datasource.go`
- `internal/handler/datasource_credentials.go`

### 这层最重要的边界

数据源权限跟着所属 KB 走，不单独做共享模型。

也就是说：

- 只有 owning tenant 能管理数据源配置
- 凭据、资源范围、同步日志都锁在这个租户里
- 数据源编辑不是共享 KB editor 能直接越权管理的一层

### 建议先看这些入口

- `CreateDataSource`
- `UpdateDataSource`
- `ValidateCredentials`
- `ManualSync`
- `GetSyncLogs`

#### `UpdateDataSource` 要特别看什么

它明确把“非凭据配置更新”和“凭据更新”拆开了：

- 普通 `PUT /datasource/:id` 不处理 credentials
- credentials 只能走单独的 `/credentials` 子资源

这和前端数据源弹窗的交互是直接对应的。

## 2. 再看 data source service

文件：

- `internal/application/service/datasource_service.go`

建议先看：

- `CreateDataSource`
- `UpdateDataSource`
- `UpdateDataSourceCredentials`
- `ManualSync`
- `ProcessSync`
- `processSyncStreaming`
- `updateSyncRunResult`

### `ManualSync` 的职责

它不是直接执行同步，而是：

1. 校验数据源当前状态是否允许同步
2. 先创建一条 `sync_log`
3. 再把真正的同步任务丢进 asynq

这样前端一点击“立即同步”，就能马上拿到一条可以轮询的运行记录。

### `ProcessSync` 的职责

这是异步 worker 的真正主入口。它会负责：

- 加载 data source、KB、sync log
- 把写权限上下文切到目标 KB
- 加载 connector 和配置
- 决定走流式同步还是普通同步
- 最终更新 cursor、sync result、data source 状态、审计日志

### 这一层最容易误判的地方

同步不是“单纯把外部文件拉下来”。

它还涉及：

- 资源筛选
- 增量 cursor
- 删除同步
- 知识入库
- 标签附着
- 同步结果采样和错误归档

所以很多“为什么这次同步没有删掉旧文档 / 为什么状态变 partial”都要回到这里看。

## 3. 看 scheduler

文件：

- `internal/datasource/scheduler.go`

重点函数：

- `Start`
- `AddOrUpdate`
- `triggerSync`

### 调度器在做什么

调度器只负责“按 cron 触发”和“防止重复排队”，真正的同步仍然走和手动同步相同的 asynq worker。

### 这里有两层防重

#### 第一层：数据库运行中检查

- 如果这个 data source 还有 running sync，就跳过这次 cron tick

#### 第二层：asynq TaskID 去重

- 同一分钟内，不同实例为同一个 data source 生成相同的 TaskID
- 第一个 enqueue 成功，其他实例拿到 `ErrTaskIDConflict`

这两层组合起来，能减少分布式场景下的重复同步。

## 4. 看流式同步和 cursor

文件：

- `internal/application/service/datasource_service.go`

重点函数：

- `processSyncStreaming`
- `streamStartCursor`
- `streamSyncHandler.Emit`
- `streamSyncHandler.Checkpoint`

### 为什么这里重要

大数据源如果每次都“先全量抓完，再统一入库”，会遇到：

- 内存占用高
- 超时后全量重来
- 大批量失败后定位困难

所以支持流式 connector 时，这里会改成：

- 边 fetch
- 边 ingest
- 边推进 cursor/checkpoint

这样失败后可以从最近 checkpoint 继续，而不是从头开始。

## 5. 一个实用阅读路径

### 如果你准备改“搜索结果排序”

1. `internal/handler/knowledgebase.go`
2. `knowledgebase_search.go`
3. `knowledgebase_search_fusion.go`
4. `knowledgebase_search_rerank.go`
5. 前端引用检索结果展示组件

### 如果你准备改“多 KB 混合检索”

1. `knowledgebase_search.go`
2. `knowledgebase_search_fanout.go`
3. `knowledgebase_search_shared.go`
4. `session_knowledge_qa.go`

### 如果你准备改“数据源立即同步 / 定时同步”

1. `internal/handler/datasource.go`
2. `datasource_service.go`
3. `scheduler.go`
4. 对应 connector 实现
5. 前端数据源设置和同步日志组件

## 当前最值得记住的几个边界

- `/hybrid-search` 是底层召回，不是最终问答。
- 多 KB 检索前必须先校验 embedding 模型兼容性。
- fan-out 的单位是 store group，不是单个 KB。
- 数据源普通更新接口不处理 credentials。
- 手动同步和定时同步最终都会汇合到同一个异步同步 worker。
- 同步日志是前端观察同步过程的真入口，不只是事后报表。
