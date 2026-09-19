# ADR-0004：进程运行时与安全边界

- 状态：已接受
- 日期：2026-09-19
- 决策范围：进程执行、权限、网络、环境变量、Worktree 安全声明
- 关联方案：`technical-proposal-v2.md` 第 2、5、7、8、9 节

## 背景

Regression Agent 需要在历史 commit 上运行安装、构建和测试命令。这些命令来自用户仓库，可能读取文件、访问网络、产生子进程或修改工作目录之外的资源。

Git Worktree 只能隔离 checkout 和工作副本，不能限制程序拥有的操作系统权限。简单的命令 allowlist、环境变量清理或不使用 Shell，也不能构成强安全沙箱。

第一阶段需要提供足够可控、可取消和可审计的运行时，同时准确说明无法提供的安全保证。

## 决策

所有由 Oracle 或 Agent 触发的本地进程均由 Rust Process Runtime 启动。Python Sidecar 不直接调用本地进程。

执行链为：

```text
Tool Request
  -> Rust Capability Broker
  -> Session / Path / Program / Network / Timeout Policy
  -> Process Runtime
  -> Session Worktree
  -> Structured ExecutionResult + Trace
```

### 命令表示

默认使用结构化的 `program + args`：

```text
program: string
args: string[]
cwd: relative path
timeout_ms: integer
```

MVP 不通过系统 Shell 解释命令字符串，也不把通用 Shell 作为必备能力。这样可以避免依赖 Shell quoting，并减少管道、重定向、命令替换等隐式能力。

这不意味着被执行程序是安全的；程序仍可能自行访问系统资源。

### 路径策略

- `cwd` 必须是当前 Session Worktree 内的相对路径。
- Host 对路径进行规范化，并拒绝解析后逃逸 Worktree 的路径。
- Session 与 Worktree 路径由 Host 生成，不接受模型指定的绝对路径。
- Agent 暴露的文件读取与搜索工具使用相同的路径约束。
- 原始仓库不得作为 Agent 或 Oracle 的可写工作目录。
- 符号链接、junction 和其他重解析点的逃逸行为必须纳入 Windows 测试。

### 程序与环境策略

- 工具请求必须经过基于 Session 和能力的 Policy 校验。
- 环境变量以最小 allowlist 构造，并移除常见 Token、Secret 和凭据变量。
- API Key 通过模型客户端的受控配置注入，不传给仓库进程。
- 网络策略默认关闭；需要联网的 Setup 必须产生明确的 UI 审批事件。
- Git 写操作只允许 Rust 的 BisectEngine 与 WorktreeManager 执行。
- 后台进程在 MVP 中禁止；命令结束、超时或取消时必须终止其进程树。

网络“默认关闭”是产品策略目标。只有在采用可验证的平台隔离机制后，才能宣称网络在操作系统层面被阻断；仅靠约定或环境变量不能作为强制隔离证明。

### 执行结果

每次执行返回结构化结果，至少包含：

- Execution ID 与关联的 Session、commit、Oracle revision。
- `exited`、`timed_out`、`cancelled`、`spawn_failed` 等状态。
- Exit code（存在时）。
- stdout/stderr preview、完整 Artifact 引用和截断标记。
- 开始、结束时间与持续时间。
- 实际执行的 program、args、cwd 和 Policy/审批记录。

输出 preview 有大小上限，完整内容流式写入 Artifact Store。展示、模型上下文和 JSONL 协议都不得依赖无限增长的内存缓冲区。

### 取消与进程树

Host 为每次执行维护 Cancellation Token 和受管进程身份。以下事件必须触发进程树终止：

- 用户取消 Session 或当前 Execution。
- 命令超时。
- Sidecar 退出且该执行只服务于其请求。
- App 正常退出或进入受控关闭流程。

Windows-first 阶段应选择能够管理完整子进程树的机制，并通过“命令产生孙进程”的 Fixture 验收。不能只终止直接子进程。

### 审批与 Trace

需要扩大能力的操作由 Host 发送 UI 审批事件。审批记录包括请求内容、风险说明、用户决定、时间和适用范围。Sidecar 不使用终端输入，也不能自行扩大权限。

每次 Tool Request、Policy 判定、审批、执行与结果都进入 Trace。进入持久化或 UI 前对敏感字段进行结构化脱敏。

## 安全声明

MVP 提供的是：

- Git 工作副本隔离。
- 受控的工具入口、路径和流程。
- 环境最小化、审批、超时、取消、进程树清理和审计。

MVP 不提供：

- 对恶意代码的强 OS 沙箱。
- 文件系统、注册表、凭据和网络的完整系统级隔离。
- 在当前用户权限下运行任意仓库脚本而不会影响系统的保证。

面向不可信仓库的强隔离需要后续引入容器、受限用户、Windows Sandbox 或其他平台级机制，并单独形成 ADR 与验收标准。

## 后果

### 正面影响

- 所有命令具有一致的超时、取消、输出和审计语义。
- Agent 无法绕过 Host 直接获得 Shell 或 Git 写权限。
- Worktree 不会被错误宣传为安全沙箱。
- 后续引入平台级隔离时，可以保留上层 Capability Broker 接口。

### 代价

- 不使用 Shell 会让部分现有项目命令需要拆解或包装。
- Windows 进程树、路径重解析点和网络限制实现复杂。
- 环境 allowlist 可能导致部分构建工具缺少必要配置，需要可审计地扩展。
- 在强沙箱实现前，不适合对不可信仓库提供无提示的自动执行。

## 未采用的方案

### 将 Worktree 视为沙箱

Worktree 不限制进程权限，无法满足安全定义，因此明确拒绝这一表述。

### 给 Agent 无限 Shell

难以静态校验路径、权限和副作用，且会扩大提示注入的影响范围，因此不属于 MVP。

### 仅使用命令 allowlist

同一个解释器或构建工具可以执行任意项目代码，allowlist 不能替代系统隔离，因此只能作为纵深策略的一部分。

## 未解决问题

- Windows 上进程树管理采用 Job Object 还是等价封装。
- 无强沙箱阶段，网络默认关闭策略可达到的可验证程度。
- Setup 依赖缓存的位置、隔离粒度和可信边界。
- 用户自定义环境变量的审批与脱敏规则。
- Shell 能力未来开放时的权限模型和命令展示方式。
