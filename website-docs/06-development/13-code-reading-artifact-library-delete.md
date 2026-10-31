# Artifact 下载、删除与产物库阅读指南

这份指南继续接在 Agent artifact/续播之后，专门讲“文件已经产出了，后面怎么列出来、怎么下载、怎么删”。

它适合覆盖这几类二开场景：

1. 你想改聊天里的文件抽屉
2. 你想改全局产物库页面
3. 你想改 artifact 删除和回收逻辑

## 推荐阅读顺序

1. `internal/handler/session/artifact_download.go`
2. `internal/handler/session/artifact_library.go`
3. `internal/handler/session/artifact_delete.go`
4. `internal/application/service/message.go`
5. `internal/application/service/message_artifact_delete.go`
6. `internal/application/repository/message_artifact.go`
7. `internal/handler/session/artifact_storage.go`
8. `frontend/src/api/chat/index.ts`
9. `frontend/src/api/artifacts.ts`
10. `frontend/src/views/artifacts/ArtifactLibrary.vue`
11. `frontend/src/utils/sessionArtifacts.ts`
12. `frontend/src/utils/artifactLibrary.ts`

## 1. 先看下载与列表 handler

文件：

- `internal/handler/session/artifact_download.go`

重点看：

- `ListSessionArtifacts`
- `ListMessageArtifacts`
- `DownloadMessageArtifact`

### 三个接口的关系

#### `ListMessageArtifacts`

返回某条 assistant message 自己挂着的 artifacts。

典型场景：

- 回答工具栏里的“下载本轮文件”
- 某条回答点开后的 artifact 抽屉

#### `ListSessionArtifacts`

返回整个 session 范围内所有 assistant message 产生的 artifacts。

典型场景：

- 聊天侧边的 sandbox/产物面板

#### `DownloadMessageArtifact`

真正把文件流出来的下载接口。

注意它的寻址方式始终是：

- `session_id`
- `message_id`
- `index`

而不是 URL、句柄、文件名。

### 为什么列表接口不直接返回存储 URL

因为 artifact 的底层存储路径可能是：

- 资源目录 handle
- provider:// 路径
- tenant 绑定的独立 backend

这些都不应该直接暴露给前端。前端只能拿到元数据，再通过下载接口走一次权限校验。

## 2. 再看产物库列表 handler

文件：

- `internal/handler/session/artifact_library.go`

### 这层要记住什么

产物库不是“所有 artifact 行的原始平铺”，而是：

- 跨会话
- 按用户可见 session 范围
- 每个文件只显示当前最新版本

所以前端看到的一行，背后可能对应同一个 `source_path` 的多次生成版本。

### 为什么一行还要带 `session_id + message_id + index`

因为产物库只是“跨会话的文件视图”，不负责重新定义下载寻址。

真正的下载、删除仍然要回到原消息坐标上执行，这样后端权限模型和会话绑定关系都不用另起一套。

## 3. 看删除 handler 的职责边界

文件：

- `internal/handler/session/artifact_delete.go`

重点看：

- `DeleteMessageArtifact`
- `DeleteLibraryArtifact`
- `deleteArtifact`

### 两个删除入口的差别

#### 聊天内删除

`DeleteMessageArtifact`

- 默认只删当前这一行
- 因为聊天抽屉是一版一版展开看的

#### 产物库删除

`DeleteLibraryArtifact`

- 默认 `all_versions = true`
- 因为产物库一行代表“这个文件”的当前视图，不是某次单独生成记录

### 为什么删除分两段

`deleteArtifact` 实际上做的是两件事：

1. 先把数据库行 tombstone
2. 再 best-effort 回收底层 blob

用户可见成功条件主要看第一步。第二步失败只会留下可回收的孤儿对象，不会把界面上的“已删除”反转回去。

## 4. 再看 service：产物库与删除规则

文件：

- `internal/application/service/message.go`
- `internal/application/service/message_artifact_delete.go`

### `ListArtifactLibrary`

这里主要做：

- 把查询范围限定到当前 tenant / user
- 标准化分页
- 规范化 fileTypes 扩展名
- 把内部 URL 转成对前端有意义的 `handle`

### `DeleteSessionArtifact`

这层负责的不是文件 IO，而是业务规则：

- 调用者是否拥有该 session
- 要删的是一行还是一整个版本组
- 这次真正 tombstone 了哪些行
- 哪些 blob 候选可以进入后续 reclaim

### `AllVersions` 的真正含义

它不是“同名文件都删掉”，而是：

- 同一个 session
- 同一个 `source_path`
- 且仍然存活的所有再生成版本

所以 library 的“版本数”其实更接近“同一 sandbox 输出文件的历史再生成次数”。

## 5. 看 repository：版本折叠和事务 tombstone

文件：

- `internal/application/repository/message_artifact.go`

重点看：

- `ListArtifactLibrary`
- `FindSessionArtifact`
- `FindSessionArtifactVersions`
- `SoftDeleteSessionArtifacts`

### `ListArtifactLibrary` 是怎么折叠版本的

它会先按：

- `session_id`
- `source_path`（为空时退回单条 artifact 自己）

构造一个 version group，然后：

- 用 `ROW_NUMBER` 选出最新版本
- 用 `DENSE_RANK/MAX` 近似算出 `version_count`

所以前端看到的：

- 一行 = 一个 version group 的最新版本
- `version_count` = 这个 group 里一共有多少个不同持久化版本

### 为什么 `SoftDeleteSessionArtifacts` 要事务执行

删除一个文件的“所有版本”往往不止一行。

如果中间失败：

- 一部分版本会消失
- 另一部分还在
- blob reclaim 也可能只回收了一半的前置条件

所以这里必须 all-or-nothing。

### 为什么只返回本次真正标记成功的行

因为并发删除时，只有先 tombstone 成功的那个请求，才有资格去 reclaim 对应 blob。

另一个慢到的请求应该看到“已经删过了”，而不是再去删一次对象。

## 6. 看 blob 回收为什么要额外守护

文件：

- `internal/handler/session/artifact_storage.go`

重点看：

- `reclaimArtifactBlobs`
- `locateArtifactBlob`

### 这里最重要的边界

数据库行删掉，不等于底层 blob 一定能删。

还要继续确认：

- 资源目录里是否还有别的 owner binding
- 是否还有别的 live message row 指向同一 URL
- 这个 blob 是否真的属于当前 tenant

### 为什么这么谨慎

因为同一个 artifact blob 可能仍然被这些东西引用：

- 另一条消息重新挂载
- fork 出来的会话
- 保存进知识库后的引用

误删 blob 的代价比留一个孤儿对象高得多。

## 7. 再看前端下载与产物库页面

文件：

- `frontend/src/api/chat/index.ts`
- `frontend/src/api/artifacts.ts`
- `frontend/src/views/artifacts/ArtifactLibrary.vue`

### 前端下载为什么用 `getDown`

因为下载仍然要带 Bearer token。

如果直接用裸 `<a href>`：

- 浏览器不会自动复用前端 axios 那套认证
- 很容易变成 401

所以这里统一走 blob 下载，再自己触发浏览器保存。

### 产物库页面最值得看的点

#### 分页追加为什么要按业务键去重

删除一行后，后续页的数据会整体前移。

如果还按“第 N 页就是全新的一段”去拼接，就会把某些行重复渲染一遍。

#### `loadMore` 为什么按当前条数反推页码

因为删除之后：

- `page.value` 还是旧值
- 但真实已展示条数已经变了

这时继续 `page + 1` 会直接跳过一行。

#### 删除确认为什么要说版本数

因为 library 里的删除默认会带走整个 version group。

对用户来说，它删的不只是当前看到的这一条记录，而是这个文件在该 session 里的所有历史版本。

## 8. 一个实用阅读路径

### 路径一：改“回答里的下载按钮”

1. `frontend/src/api/chat/index.ts`
2. `artifact_download.go`
3. `ChatArtifactsDrawer.vue`

### 路径二：改“产物库页面”

1. `frontend/src/views/artifacts/ArtifactLibrary.vue`
2. `frontend/src/utils/artifactLibrary.ts`
3. `frontend/src/api/artifacts.ts`
4. `artifact_library.go`
5. `message_artifact.go`

### 路径三：改“删除和回收”

1. `artifact_delete.go`
2. `message_artifact_delete.go`
3. `message_artifact.go`
4. `artifact_storage.go`

## 当前最值得记住的几个边界

- artifact 列表接口返回的是元数据，不是底层存储 URL。
- 下载和删除的真正坐标始终是 `session_id + message_id + index`。
- 聊天内删除默认只删当前行；产物库删除默认删整个版本组。
- tombstone 成功后，用户视角的删除就已经成立；blob reclaim 是后续清理。
- 产物库一行代表“最新版本视图”，不是某次单独生成记录。
