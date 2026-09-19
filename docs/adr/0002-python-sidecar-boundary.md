# ADR-0002：Python Sidecar 的职责与通信边界

- 状态：已接受
- 日期：2026-09-19
- 决策范围：Agent Runtime、模型接入、Host/Sidecar 协议
- 关联方案：`technical-proposal-v2.md` 第 3、7、10、11 节

## 背景

项目需要复用 Python 生态中的模型客户端和 Agent Harness，同时保持桌面产品的 Git、进程和持久化流程可控。若 Sidecar 直接访问 Git、数据库和本地进程，它将变成第二个 Host；若所有推理都移入 Rust，则会提高模型能力迭代成本。

需要在产品控制面与 Agent 推理面之间建立清晰、可测试的边界。

## 决策

Python Sidecar 是短生命周期、按 Session 隔离的推理进程。第一阶段每个活动 Session 最多对应一个 Sidecar。

Sidecar 负责：

- 运行最小 Agent Loop。
- 通过 `ModelClient` 接口调用配置的模型 Provider。
- 生成 OracleSpec 候选。
- 对确定规则无法分类的结果进行分析并返回候选 Verdict。
- 在 Bisect 完成后分析 culprit、parent 和相关 Diff。
- 校验模型结构化输出，执行重试、上下文和预算限制。
- 通过 EventSink 发送状态、增量文本、工具请求和最终结果。

Sidecar 不负责：

- 直接管理 Git Bisect 或 Worktree。
- 直接写 SQLite 或 Artifact Store。
- 直接执行本地进程或任意 Shell。
- 通过终端 `input()` 获取权限。
- 保存跨 Session Memory，或把长对话作为产品状态。

## 通信协议

Host 与 Sidecar 使用 stdin/stdout 上的 JSON Lines 通信：

- stdout 只包含协议消息，日志只写 stderr。
- 每条消息包含 `protocol_version`、`session_id`、`message_id`、`seq`、`type`、`timestamp` 和 `payload`。
- 启动时必须完成 `host.hello` / `sidecar.ready` 版本握手。
- 工具调用使用唯一 `tool_call_id` 关联请求与结果。
- Host 使用 `message_id` 做幂等处理，并验证 Session、消息类型、大小与顺序。
- 大型 stdout、stderr、Diff 和报告附件通过 Artifact 引用传递，不嵌入协议消息。
- 协议必须定义最大单行大小、背压、超时、取消和异常关闭行为。

Sidecar 发出的 Tool Request 只是能力请求。Rust Capability Broker 负责参数校验、Policy、审批、执行和结果记录；Sidecar 不拥有最终执行权。

## Agent 生命周期

一个典型生命周期为：

1. Host 启动 Sidecar 并完成版本握手。
2. Host 发送包含 Session 受限上下文的任务。
3. Sidecar 按需请求只读 Git、代码搜索或进程能力。
4. Host 返回结构化 Tool Result 或结构化失败。
5. Sidecar 返回 OracleSpec、Verdict、分析报告或错误。
6. Host 决定是否接受结果以及如何推进状态。
7. Session 取消、结束或 Sidecar 失联时，Host 终止并回收 Sidecar。

每个 commit 的评价尽量使用独立、受限上下文。历史事实由 Host 从 SQLite 和 Artifact Store 提供，不要求 Agent 依赖一个持续增长的对话记住整个 Bisect 过程。

## 协议错误语义

- 非法 JSON、未知消息类型或 Schema 不匹配不得导致 Host 崩溃。
- 未知工具、非法参数和 handler 异常必须返回与 `tool_call_id` 匹配的失败结果。
- Sidecar 失联时，Host 将相关 Session 标记为 `INTERRUPTED`，且不自动应用尚未确认的 Verdict。
- 收到取消后，Sidecar 应停止模型循环；Host 仍负责强制终止 Sidecar 和 Session 进程树。
- Sidecar 重启后不得假设其内存状态仍然有效，恢复输入由 Host 重新提供。

## Harness 范围

第一阶段保留：

- Agent Loop
- ModelClient interface
- Tool Registry 与 tool-call 协议完整性
- 结构化输出校验与重试
- Context、步骤、时间、Token 和成本预算
- Cancellation Token 与 tracing events

第一阶段不接入：

- Subagent 或 Agent Teams
- Memory、Skill、Cron、MCP
- Todo 系统
- 通用文件写入和无限制 Shell

## 后果

### 正面影响

- Python 模型生态可以独立迭代，不侵入 Rust 领域状态机。
- Agent 权限集中由 Host 控制，行为可追踪并可测试。
- Sidecar 崩溃被限制在单个 Session，不应破坏 Git 状态。
- 协议 Schema 可以作为 Rust 与 Python 的契约测试边界。

### 代价

- 需要维护进程生命周期、JSONL 协议和跨语言 Schema。
- Tool 调用存在序列化与进程通信开销。
- Sidecar 打包、版本匹配和 Windows 进程管理会增加发布复杂度。

## 未采用的方案

### 将现有 `gfpl_agent` 原样嵌入产品

其全局 Workspace、直接进程执行、终端 Permission 和 stdout 日志行为不满足产品边界，因此只提取最小能力，不作为未经改造的生产依赖。

### 在 Sidecar 内直接执行工具

这会绕过 Rust Policy、Trace 和取消控制，因此不采用。

### 引入 LangGraph 或第二套 Agent 框架

第一阶段的 Agent 流程有限，自研 Harness 已覆盖所需核心机制。增加框架会引入重复抽象和额外状态，因此暂不采用。

## 未解决问题

- Sidecar 的打包形式、升级策略和二进制完整性校验。
- 协议 Schema 的代码生成方案与兼容窗口。
- 模型流式输出的背压阈值和 UI 合并策略。
- Sidecar 异常重启次数及退避策略。
