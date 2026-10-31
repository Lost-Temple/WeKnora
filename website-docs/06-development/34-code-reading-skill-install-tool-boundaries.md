# 技能 Install Shell、Tool 白名单与 Skill File Tools 阅读指南

这一篇接着 [33-code-reading-skill-install-transcript-progress-timeline.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/33-code-reading-skill-install-transcript-progress-timeline.md) 往后读，专门讲技能安装期的工具边界：

- installer agent 到底被允许用哪些工具
- install mode 的 `shell_exec` 为什么和普通会话 shell 不是同一种能力
- `write_skill_file` / `edit_skill_file` 为什么是“更窄的写能力”而不是“更强的写能力”

这条链最核心的设计是：

- **安装期并不是把普通 agent 工具全开**
- **而是换上一套专门面向 skill tree 的受控工具面**

## 推荐阅读顺序

1. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2169-L2295)
2. [agent_service.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L416-L535)
3. [agent_service.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L965-L1015)
4. [shell_exec.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L207-L373)
5. [skill_file.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/skill_file.go#L1-L407)
6. [capabilities.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sandbox/capabilities.go#L102-L139)
7. [tenant_skill_install.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2018-L2050)

## 1. 为什么 installer agent 只拿固定三件工具

从 [installerAgentConfig](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2202-L2260) 可以直接看到，安装期平台层固定塞进去的工具只有：

- `shell_exec`
- `write_skill_file`
- `edit_skill_file`

这不是一个“默认列表”，而是平台主动定义的 install contract。

也就是说，installer 并不会像普通聊天 agent 那样拿到：

- `write_sandbox_file`
- `edit_sandbox_file`
- `list_sandbox_files`
- 各种知识库 / wiki / MCP / Web 搜索工具

因为它的任务非常单一：

- 在 skill tree 里装依赖
- 修 skill tree 文件
- 最后把 image 或本地版本目录提交出去

## 2. 为什么工具白名单要在平台层硬并进去

[installerAgentConfig](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2241-L2253) 不是简单读取 builtin agent 的 registry 配置，而是把 install tools 强制 union 进去。

原因有两个：

### 第一，installer 是平台安全面的一部分

如果它少了 `write_skill_file` / `edit_skill_file`，模型就很容易回退到：

- `shell_exec` + heredoc

也就是项目过去明确想替换掉的那条脆弱路线。

### 第二，registry 配置可能滞后

平台 YAML 早于这些新工具存在，如果完全信 registry，就会让老部署静默退回旧行为。

所以这里的取舍是：

- 工具白名单由平台兜底定义
- registry 只是在这个基础上提供默认 prompt / 迭代上限等配置

## 3. `SkillInstallMode` 才是安装期工具面的总开关

在 [installerAgentConfig](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2234-L2240) 和 [registerSandboxShellTool](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L987-L1008) 里可以看到：

- 只有 `EnableSkillInstallMode(...)` 打开了 install mode
- 后面才会注册 install shell 和 skill file tools

这意味着：

- “把某个 agent 命名成 installer”
- “在数据库里给 agent 配一些工具”

都不等于它真能拿到安装期特权能力。

真正能进入这条分支的，仍然只有：

- 内置 skill installer agent

## 4. 为什么 install mode 不能复用普通 `/workspace` 文件工具

[registerSandboxFileTools](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L416-L473) 已经把这件事讲得很直白：

- 普通会话文件工具写的是 `/workspace`
- 安装期真正要写的是 skill tree

而且 `/workspace` 在技能安装语义里还有一个额外问题：

- 它不是最终 snapshot / activate 要保留的那棵树

所以安装期如果继续给：

- `write_sandbox_file`
- `edit_sandbox_file`

最常见的结果就是：

- agent 先往 `/workspace` 写文件
- 后来发现根本不会进 image
- 再退回 `shell_exec` heredoc

这正是项目明确想避免的路径。

## 5. 为什么 `write_skill_file` / `edit_skill_file` 是“更窄”而不是“更强”

很多人第一眼会以为：

- install shell 已经是 root
- 再给 skill file tools，岂不是更危险

其实 [registerSkillFileTools](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L476-L523) 和 [resolveSkillFilePath](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/skill_file.go#L383-L407) 表达的恰好相反：

- install shell 的 reach 确实更大
- skill file tools 反而把写入收窄成“只准写这一个 skillDir”

所以它们的定位不是扩权，而是：

- **给模型一条比 heredoc 更稳、比 root shell 更窄的写文件通道**

## 6. `SkillFileStore` 的两层边界是什么

[SkillFileStore](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/skill_file.go#L34-L40) 的注释很关键：

### 第一层：底层 manager 边界

- 只能写 install 根

对 remote 来说是：

- skills image root

对 host 来说是：

- 当前 install 绑定的版本目录根

### 第二层：工具层边界

- 再进一步只允许写当前 skillDir

这样就算 install shell 本身能接触更大的安装根，`write_skill_file` 也不能越过自己的 skill 边界。

## 7. 为什么 `resolveSkillFilePath` 是 skill file tools 的核心防线

[resolveSkillFilePath](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/skill_file.go#L383-L407) 做了这几件事：

1. skillDir 必须先是合法绑定目录
2. 空路径直接拒绝
3. 相对路径统一按 skillDir 解析
4. `..`、绝对路径逃逸、邻居 skill 路径全部拒绝
5. skillDir 自己这个目录也拒绝，当成“不是文件”

这使得 skill file tools 的 contract 非常明确：

- 你可以写“这个技能树里的某个文件”
- 不能把它们当成任意文件 API

## 8. 为什么 `write_skill_file` / `edit_skill_file` 都只接受小文本

在 [WriteSkillFileTool.Execute](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/skill_file.go#L98-L180) 和 [EditSkillFileTool.Execute](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/skill_file.go#L242-L379) 里，都有这些限制：

- 单次大小上限 262144 bytes
- 拒绝二进制内容
- `edit_skill_file` 只做精确文本替换

这也是刻意的：

- 它们的目标是修声明文件、补 wrapper 脚本、改几行配置
- 不是拿来传大文件或构造通用二进制写接口

这和普通 `write_sandbox_file` 的定位也不一样：

- 安装期工具写的是将被 snapshot 的 skill tree
- 出错代价更高，所以 contract 更窄

## 9. 为什么 syntax hint 不是“拒绝写入”

`write_skill_file` / `edit_skill_file` 在写完 Python 文件后，会顺手给一个：

- `pythonScriptSyntaxHint(...)`

如果有问题，返回的是：

- 文件已经写进去
- 但 ToolResult 标成失败并提示语法问题

这背后反映的设计是：

- 工具负责把 agent 想写的文本稳定落下去
- 不负责替 agent 再决定要不要保留这次修改

也就是说，syntax hint 更像：

- “写入成功，但下一步你应该继续修”

而不是：

- “我替你偷偷回滚这次写入”

## 10. install shell 为什么要用单独的执行器接口

[SandboxInstallCommandExecutor](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L207-L215) 故意和普通 shell executor 分开命名。

目的就是避免这种误接：

- 某个只支持普通 session shell 的对象
- 因为恰好实现了类似方法
- 被错误接到 install shell 分支上

安装期的 root/skills-root 写权限是高敏感能力，所以项目这里要求：

- 类型名就先把语义分开

## 11. `installShellExecutor` 真正盖的章是什么

[installShellExecutor](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L217-L240) 不是普通适配层，它统一给每次执行盖了两个关键选项：

- `AsRoot = true`
- `AllowSkillsRoot = true`

这就是 install shell 和普通 shell 的根本分水岭：

### 普通会话 shell

- 主要面向 session workspace
- 不允许随便写 image root

### 安装期 shell

- 允许在安装根里装依赖、改 skill tree

所以 install shell 的特权不是“模型传参自己决定的”，而是：

- 平台在这一层统一强制附加的

## 12. remote install shell 和 host install shell 为什么又不完全一样

[NewInstallShellExecTool](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L323-L345) 和 [NewHostInstallShellExecTool](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L347-L362) 体现了两种 install contract：

### remote

- 默认 work_dir 是 image 内 skillDir
- 允许受控地离开到 workspace/image 根
- 以 root 身份运行

### host

- 默认 work_dir 是本地版本目录
- 写范围只限当前版本目录
- 以桌面用户身份运行，不是 root

所以它们共享“安装期 shell”这个概念，但安全边界并不完全相同。

## 13. prompt 为什么要反复强调“不要用 heredoc / cat”

在 [buildInstallPrompt](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/tenant_skill_install.go#L2018-L2050) 和 [installShellExecDescription](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/shell_exec.go#L366-L398) 里，都反复写了：

- 用 `write_skill_file` / `edit_skill_file`
- 不要用 heredoc
- 不要用 `cat > file`
- 不要用 `write_sandbox_file`

这是因为这几种错误路径都曾经有现实问题：

### heredoc / `cat`

- 命令长度上限
- 双层 quoting
- 文件内容容易截断或转义错乱

### `write_sandbox_file`

- 只能写 `/workspace`
- skill tree 不在那
- snapshot 不会保留它

换句话说，这不是 prompt 在“偏好推荐”，而是在强行把模型推离历史上最容易坏的几条路。

## 14. `SessionInstallShellExecutor` 为什么要和普通 session shell capability 并列

[SessionInstallShellExecutor](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sandbox/capabilities.go#L102-L112) 和 [SessionInstallCapabilityProvider](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sandbox/capabilities.go#L126-L133) 说明，安装期 shell 在 capability 层就是一个单独接口。

这意味着 backend 可以显式表达：

- 我支持普通 shell
- 但我不支持 install-mode shell

或反过来。

上层据此决定：

- 该不该把 install `shell_exec` 工具注册出来

这比“拿到一个 manager 就假设它什么 shell 都能跑”要安全很多。

## 15. 为什么普通 shell 还要继续保留另一套 env contract

在 [registerSandboxShellTool](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/agent_service.go#L965-L1015) 里，普通 shell 分支会挂：

- `userEnvResolver`
- `skillEnvCapture`

而 install mode 不会。

原因很简单：

- 普通 shell 是给聊天 agent 跑技能/runtime 命令的
- install shell 是给 installer 改 image/版本树的

两者虽然都叫 `shell_exec`，但上层语义完全不同：

### 普通 shell

- 可选选择某个 installed skill 的 runtime
- 使用和回填用户 skill env

### install shell

- 不消费这些 runtime skill env
- 只负责把 skill 安装到将来会被运行的位置

## 16. 为什么 `write_skill_file` / `edit_skill_file` 不出现在普通可选工具列表里

在 [definitions.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/agent/tools/definitions.go#L48-L67) 已经写明：

- 这两个工具故意不放进 `AvailableToolDefinitions`

原因是它们写的是：

- 共享 snapshot image / host 版本树

如果把它们暴露成 tenant-editable agent 的普通可选工具，就等于把安装期写能力泄露给了日常 agent 配置面。

所以它们只能通过：

- install mode

被动注册，而不是主动勾选。

## 17. 这篇和前后文怎么接

建议按下面顺序连读：

1. [31-code-reading-host-skill-version-tree.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/31-code-reading-host-skill-version-tree.md)
2. [32-code-reading-skill-install-verify-runtime-gates.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/32-code-reading-skill-install-verify-runtime-gates.md)
3. [33-code-reading-skill-install-transcript-progress-timeline.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/33-code-reading-skill-install-transcript-progress-timeline.md)
4. 当前这篇 `34`

这样能把安装链读成完整闭环：

- host / remote 最终写到哪里
- install 如何验收
- transcript / progress 如何展示
- 最后 install agent 具体被允许拿哪些工具、这些工具各自受什么约束

## 18. 二开时最容易踩的坑

### 坑一：觉得 install mode 只是普通 agent 多开几个工具

其实它是另一套单独的 shell 和文件写入 contract。

### 坑二：继续让 installer 用 `write_sandbox_file`

它写的是 `/workspace`，不是 skill tree。

### 坑三：以为 `write_skill_file` / `edit_skill_file` 比 install shell 更危险

恰好相反，它们是更窄的受控写接口。

### 坑四：把 install shell 的 `AsRoot` / `AllowSkillsRoot` 交给模型自己决定

项目这里是平台层统一盖章，不接受模型请求特权。

### 坑五：把这些工具暴露进 tenant 可编辑的普通 agent 工具面

那会把安装期写共享 image 的能力泄露到日常聊天 agent。
