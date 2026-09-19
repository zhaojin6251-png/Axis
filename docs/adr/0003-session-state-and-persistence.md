# ADR-0003：Session 状态、持久化与恢复模型

- 状态：已接受
- 日期：2026-09-19
- 决策范围：领域状态机、SQLite、事件、Artifact、恢复
- 关联方案：`technical-proposal-v2.md` 第 4、5、6、9、12 节

## 背景

一次回归定位可能持续较长时间，期间会发生模型失败、测试超时、应用退出和用户取消。只依赖内存状态或 Git 内部 Bisect 文件，无法完整恢复 UI、Evidence 与业务判断；只保存最终结果，又无法审计每一步是如何得出的。

Session 需要明确的状态机、持久化顺序和证据模型。

## 决策

每次回归定位对应一个独立的 Regression Session。Rust Host 在 SQLite 中保存产品状态，并在 Artifact Store 中保存大型内容。SQLite 由 Rust Host 单独写入。

### 主流程状态

```text
CREATED
  -> PREPARING_WORKTREE
  -> PREPARING_ORACLE
  -> VALIDATING_ENDPOINTS
  -> BISECTING
  -> VERIFYING_CULPRIT
  -> ANALYZING
  -> COMPLETED
```

该顺序以可执行依赖为准：Endpoint Validation 需要已存在的 Worktree 和待验证的 OracleSpec。它修正了技术方案第 4.1 节状态示意图与第 6 节 Workflow 之间的顺序矛盾；技术方案后续应同步更新。

旁路状态为：

- `WAITING_USER`：需要人工提供或确认信息，可回到记录的来源状态。
- `CANCELLED`：用户主动终止的终态。
- `FAILED`：系统错误导致无法继续的终态。
- `AMBIGUOUS`：无法得到唯一 culprit 的业务终态。
- `INTERRUPTED`：进程或应用异常结束，恢复前的持久化状态。

实现状态机时，每个状态必须声明：

- 合法的进入来源和下一状态。
- 允许执行的领域命令。
- 进入状态前必须满足的不变量。
- Cancel 和异常发生时的清理行为。
- 可恢复的稳定检查点。

### 持久化顺序

领域状态变化遵循以下原则：

1. 校验命令在当前状态是否合法。
2. 执行产生外部事实的操作，并记录结构化结果。
3. 在 SQLite 事务中写入领域记录和事件。
4. 事务提交后更新内存投影并向 UI 广播。

若步骤涉及 SQLite 与 Git 两种无法共享事务的状态，必须记录足以在重启后 reconcile 的意图、执行结果和检查点。不得将“UI 已收到事件”视为操作成功。

### Evidence 与 Verdict 分离

Execution 记录客观执行事实，例如退出码、超时、命令不存在、持续时间和输出引用。Commit Verdict 是根据事实做出的领域判断，两者分开保存。

每个生效的 Verdict 必须包含：

- `bug_absent`、`bug_present` 或 `untestable` 之一；`inconclusive` 不推进 Bisect。
- Evidence 等级：`deterministic`、`strong` 或 `weak`。
- 稳定的 `reason_code` 和面向用户的摘要。
- 至少一个 Evidence ID。
- Verdict 来源：规则、Agent 或用户。
- Oracle revision 和执行 attempt。

非零退出码不能自动等同于 BAD。超时、工具缺失和基础设施错误应由 Oracle classifier 明确分类，否则为 `inconclusive` 或 `untestable`。

### Oracle revision

OracleSpec 在通过 Good/Bad Endpoint Validation 后冻结。所有 Execution 和 Verdict 都引用明确的 Oracle revision。

修改 OracleSpec 必须：

1. 创建新 revision。
2. 重新验证 Good 与 Bad 两端。
3. 明确旧 evaluation 是否失效。
4. 从一致的稳定检查点重新开始或继续。

不允许在同一 Bisect Path 中静默改变判断标准。

### 存储边界

SQLite 至少保存：

- Session 元数据与当前状态。
- Commit evaluations。
- Execution 元数据。
- 顺序事件。
- Artifact 元数据与校验值。
- 结构化报告及其版本。

Artifact Store 保存：

- 完整 stdout 与 stderr。
- Diff、补丁和大型代码片段。
- 报告附件及其他超出数据库合理大小的内容。

API Key、Token 和常见 Secret 不得进入 SQLite、事件、Artifact 或 Report。路径在持久化前必须规范化，Artifact 通过 ID 访问，不能接受模型提供的任意路径。

## 恢复与清理

应用启动时，Host 检查所有非终态 Session：

1. 检查对应 Worktree 的 registry、目录和 HEAD。
2. 检查是否遗留受管进程；不能仅凭 PID 假设进程身份。
3. 对比最近稳定检查点、Git Bisect 状态和持久化事件。
4. 将状态标记为可恢复的 `INTERRUPTED`，或记录不可恢复原因。
5. 由用户选择继续、保留现场或清理。

清理必须先停止活动进程，再检查 worktree dirty、lock 和 registry 状态。异常目录通过 reconcile 处理，不能直接递归删除未经验证的路径。

## 结果模型

Session 结束时返回以下结果之一：

- `exact`：确认唯一 culprit，并完成必要复验。
- `ambiguous`：由于 SKIP 或证据不足，只能给出候选集合。
- `aborted`：用户取消、策略终止或前置条件无法满足。

报告从结构化状态、Evaluation、Execution 和 Artifact 渲染，不把 Agent 自由文本作为唯一事实来源。

## 后果

### 正面影响

- 应用崩溃后可以解释已完成到哪一步，并安全决定恢复方式。
- Verdict、执行事实和 Agent 推理之间可追溯。
- UI、报告和审计可以从同一持久化数据生成。
- Oracle 变更不会悄悄污染已有 Bisect 结果。

### 代价

- 需要设计状态迁移、数据库迁移、事件序列和 reconcile 流程。
- SQLite 与 Git 操作之间不存在天然原子事务，需要显式恢复协议。
- Artifact 生命周期、容量限制和清理策略需要额外实现。

## 未采用的方案

### 只依赖 Git Bisect 状态

Git 不保存产品需要的 Execution、Evidence、审批和报告信息，不能作为完整 Session 存储，因此不采用。

### 所有输出直接存入 SQLite

大型 stdout/stderr 会放大数据库和 UI 压力，因此数据库只保存元数据、摘要与 Artifact 引用。

### 用模型置信度小数决定是否推进

精确小数缺少可校准依据。系统使用证据等级、来源和策略阈值，不采用模型自报概率作为真相。

## 未解决问题

- 事件表是否承担完整 event sourcing，还是仅作为审计日志。
- Git 与 SQLite reconcile 的具体状态标记和幂等键。
- Artifact 配额、保留时间和用户导出格式。
- `WAITING_USER` 恢复到来源状态的数据库表示方式。
