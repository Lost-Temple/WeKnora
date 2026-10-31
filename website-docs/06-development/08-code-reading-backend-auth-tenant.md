# 后端认证与租户链路阅读指南

这份指南面向“准备读后端认证、空间、RBAC、API key 路由控制”的场景。它的目标是先帮你把后端入口和权限边界串起来，再去看具体 handler / service。

## 为什么先读这一块

前端很多行为看起来像页面逻辑，实际上都受后端这条链路影响，例如：

- 为什么同一个用户切换空间后，很多接口返回会变；
- 为什么某些页面只是隐藏按钮，但后端仍然会 403；
- 为什么 API key 和 JWT 登录态走的是两套授权逻辑；
- 为什么某些公开路由必须注册在全局 Auth 前面。

如果这层没理顺，后面看知识库、Wiki、Agent 的后端路由会一直觉得“为什么这里又多一层判断”。

## 推荐阅读顺序

### 1. 先看总路由装配

文件：

- `internal/router/router.go`

这一层先回答两个问题：

1. 整个 Gin 路由是怎么分层注册的？
2. 哪些路由必须在全局 Auth 之前注册？

### 建议重点看

- `NewRouter()` 的注册顺序
- 全局中间件
- `middleware.Auth(...)` 挂载点
- `/api/v1` 下面统一挂载的 API key gate

这里最重要的不是某个 handler 做了什么，而是“路由注册顺序本身就是安全边界”。

## 2. 再看 RBAC 守卫怎么分类

文件：

- `internal/router/rbac.go`

这是后端权限模型的阅读入口。建议先把文件开头那段大注释完整读掉。

### 先理解两个维度

#### 1. 按角色判断

- `Viewer`
- `Contributor`
- `Admin`
- `Owner`

这类 guard 问的是：调用者在当前空间里是什么角色。

#### 2. 按资源归属判断

- `OwnedKBOrAdmin`
- `OwnedAgentOrAdmin`
- 以及 Knowledge / Chunk / WikiPage 这些子资源对应的 lookup 变体

这类 guard 问的是：调用者是不是这个资源的创建者，否则是否至少 Admin+。

### 最容易误判的地方

不要把“我是 Contributor”理解成“我能改所有 KB”。在这个项目里更接近下面这条规则：

- Contributor 可以创建自己的资源；
- 之后能改“自己创建的资源”；
- 改别人创建的资源，通常仍然不行，除非自己是 Admin+。

## 3. 再看认证与空间路由

文件：

- `internal/router/routes_auth_tenant.go`

这份文件建议按三段读：

### 第一段：`RegisterAuthRoutes`

重点看：

- `/auth/login`
- `/auth/refresh`
- `/auth/switch-tenant`
- `/auth/me`

其中最重要的是 `/auth/me`。前端那边把它当作“会话真值来源”，不是偶然设计，而是后端这里明确提供了一份“当前调用者是谁、当前空间是什么、当前 memberships 是什么”的权威视图。

### 第二段：`RegisterTenantRoutes`

重点看：

- `/tenants`
- `/tenants/:id`
- `/tenants/:id/members`
- `/tenants/:id/invitations`
- `/tenants/:id/audit-log`

这里最关键的是 `PathTenantMatch`。它决定了：

- URL 里的 `:id` 必须和当前激活空间一致；
- 不能仅仅通过改 URL，就去操作别的空间；
- 跨空间超管能力是显式豁免，不是默认开放。

### 第三段：`RegisterMyInvitationRoutes`

这块主要是“当前用户自己的邀请收件箱”，和 `/:id/invitations` 的“某个空间的全部邀请列表”是两层不同语义。

## 4. 再看组织 / 分享 / 知识库横向访问

文件：

- `internal/router/routes_agent.go`

虽然文件名叫 `routes_agent.go`，但这里其实也承接了很多组织和共享相关路由。

建议重点看：

- `/organizations/*`
- `/knowledge-bases/:id/shares`
- `/shared-knowledge-bases`
- `/shared-agents`

### 这层在解决什么问题

项目里不只有“当前空间里的自有资源”，还有：

- 组织共享过来的 KB
- Agent 间接可见的 KB
- 共享 Agent

所以后端不能只按“当前 tenant + 当前角色”简单判断，还得把“资源是如何暴露给你的”考虑进去。

## 一个推荐的真实阅读路径

如果你想搞清楚“前端切换空间后，为什么知识库列表和详情会变化”，建议这样读：

1. `frontend/src/stores/auth.ts`
2. `frontend/src/utils/request.ts`
3. `internal/router/routes_auth_tenant.go`
4. `internal/router/rbac.go`
5. `internal/router/routes_agent.go`
6. 对应的 handler / service

如果你想搞清楚“为什么某个 API key 能调这个接口，另一个不能”，建议这样读：

1. `internal/router/router.go`
2. `internal/router/rbac.go`
3. 具体 `Register*Routes` 里该路由的 `apiKeyRoute` / `apiKeyGroup` 声明

## 阅读时最该盯住的几个概念

### 1. 当前激活空间

很多接口虽然路径里没有 tenant id，但实际上是从认证上下文里取当前空间。

### 2. 角色和资源归属是两套判断

不要把它们混成一套：

- 角色决定你在这个空间里的基础权限级别；
- 归属决定这个具体资源是不是“你的”。

### 3. API key 能力和 JWT 角色不是一回事

JWT 用户是“人”，有空间角色；
API key 是“程序身份”，走 capability allow-list。

### 4. 路由声明本身就是权限文档

这个项目把很多原先藏在 handler 里的权限逻辑提升到了 route layer，所以读路由注册代码本身，就能看出大部分访问边界。

## 建议下一步接着读什么

当你把这一层读顺以后，下一步比较适合接：

1. 知识库后端路由与 handler
2. Wiki 后端路由与版本管理
3. 文件访问与跨空间资源代理
4. 数据源同步链路

这几块和前端知识库阅读指南能正好接上。
