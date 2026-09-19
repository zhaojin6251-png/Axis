# ADR-0001：Rust Host 拥有工作流控制权

- 状态：已接受
- 日期：2026-09-19
- 决策范围：Session、Git Bisect、Worktree、进程、持久化
- 关联方案：`technical-proposal-v2.md` 第 2、3、6、7 节

## 背景

Regression Agent 同时包含确定性的产品流程和非确定性的模型推理。如果 Rust Host、Python Sidecar 和 Git 自身分别维护一部分流程状态，可能出现以下问题：

- UI 显示的当前 commit 与实际 Bisect 状态不一致。
- Agent 超时或崩溃后，无法判断某次 GOOD、BAD 或 SKIP 是否已经生效。
- Manual、Assisted 和 Autonomous 模式各自形成不同流程，增加取消、恢复和测试成本。
- 模型输出错误时直接改变 Git 状态，导致结果难以重放和审计。

因此需要明确唯一的流程控制者和状态真相源。

## 决策

Rust Host 是 Regression Session 的唯一流程控制者，也是运行时状态的唯一真相源。

Rust Host 负责：

- 创建、校验、推进、取消和恢复 Session。
- 校验 Good Commit、Bad Commit 及其可达关系。
- 创建、检查、保留和清理 Session Worktree。
- 选择待测 commit，并应用 GOOD、BAD、SKIP。
- 执行 Oracle 和其他受控进程，管理超时与进程树。
- 执行权限策略和审批结果。
- 持久化状态、事件、Execution、Evidence 与 Report。
- 启动、监控和终止 Python Sidecar。

Python Sidecar 可以生成建议或结构化结果，但不能直接：

- 推进 Session 状态。
- 执行 `git bisect good/bad/skip/reset`。
- 创建、切换或删除 Worktree。
- 写入产品状态数据库。
- 把模型建议视为已经生效的领域事件。

所有会改变流程的操作必须先由 Rust Host 校验，并以领域事件的形式持久化。状态持久化成功后，Host 才向 UI 广播新状态并继续下一步。

Manual、Assisted 和 Autonomous 使用同一个 Session 状态机。三种模式只改变 Verdict 的来源和是否需要人工确认，不改变状态定义与推进规则。

## 状态所有权

| 状态或资源 | 所有者 | 其他组件的权限 |
| --- | --- | --- |
| Session phase | Rust Host | UI 与 Sidecar 只读 |
| 当前 Bisect 区间 | Rust Host | UI 展示；Sidecar 接收必要摘要 |
| Commit Verdict | Rust Host 接受并生效 | 用户、规则或 Agent 可以提供候选 Verdict |
| Worktree 生命周期 | Rust Host | Sidecar 只能通过受控工具读取或执行 |
| ExecutionResult | Rust Host | Sidecar 和 UI 读取结构化结果 |
| Agent 推理上下文 | Python Sidecar | Host 仅提供任务输入并接收输出 |
| UI 临时交互状态 | React UI | 不作为产品状态真相源 |

## 原子性要求

以下操作应被视为一个可恢复的领域步骤：

1. 接收并验证候选 Verdict。
2. 记录 Verdict、Evidence 引用和来源。
3. 应用对应的 Git Bisect 操作。
4. 记录新的 Bisect 区间与 Session 状态。
5. 发布 UI 事件。

具体事务边界由实现阶段确定，但崩溃恢复必须能够区分“尚未应用”“已经应用”和“结果未知”，不能依赖 UI 或 Sidecar 猜测。

## 后果

### 正面影响

- 三种运行模式可以复用同一套流程、测试和恢复逻辑。
- Agent 崩溃时，Manual 模式和 Git 状态仍可恢复。
- Git 写操作集中，权限校验和审计更明确。
- 相同事件序列可以用于重建 UI 和生成报告。

### 代价

- Rust Host 需要实现较完整的领域状态机和 Sidecar 协议。
- Tool Request 必须在 Host 与 Sidecar 间往返，不能直接调用本地函数。
- 状态推进需要设计幂等、事务和崩溃恢复语义。

## 未采用的方案

### 由 Python Agent 驱动 `git bisect`

实现初期较快，但会使 Agent 同时承担推理和全局编排，难以支持可靠的 Manual 模式、取消、恢复与审计，因此不采用。

### Rust 与 Python 共同维护 Session 状态

会形成双重真相源，需要复杂的一致性协议，收益不足，因此不采用。

### 为三种模式分别实现工作流

会导致状态语义逐渐分叉，并显著扩大测试矩阵，因此不采用。

## 未解决问题

- Verdict 持久化与 Git Bisect 写操作之间的崩溃一致性方案。
- 是否完全依赖 `git bisect` 内部状态，或由领域模型独立计算候选区间。
- 单个应用实例允许同时运行多少个活动 Session。
