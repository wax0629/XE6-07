# 架构：Workflow Engine、Policy Validator 与 TaskLog

## 目标

建立主链的确定性执行层：Workflow Engine 保存并推进真实状态，Policy Validator 负责执行前安全门禁，TaskLog 与 WorkflowEvent 提供可追溯记录。

该执行层必须保证 Agent 或 API 只能提出命令，不能绕过状态机、审计前置条件、用户确认和设备能力校验直接触发关键动作。

## 架构边界

本期做：

- 定义 `WorkflowCommand`、`WorkflowResult` 和 `WorkflowEvent` 的职责与契约。
- 只允许 Workflow Engine 通过领域状态机改变真实状态。
- 在执行前校验当前状态、目标 revision、审计前置条件、用户确认、设备能力和幂等键。
- 强制执行切片和真实打印门禁。
- 写入包含 `traceId`、输入/输出 hash、状态和错误码的 `TaskLog`。
- 定义失败、重复提交和取消的最小行为。

本期明确不做：

- 不实现真实任务队列。
- 不接入真实设备。
- 不实现 Agent 推理。
- 不允许 Agent、API 或 Adapter 直接修改领域状态。

## 模块职责

### Workflow Engine

- 保存工作流的真实状态。
- 根据领域状态机判断命令是否允许执行。
- 推进合法状态迁移。
- 生成成功、失败、拒绝和取消事件。
- 保证关键记录与物理动作的幂等性。

### Policy Validator

- 校验当前状态和目标 revision 是否仍有效。
- 校验可打印审计等前置条件。
- 校验用户确认和设备能力。
- 校验幂等键和权限边界。
- 返回结构化拒绝原因，不直接改变状态。

### TaskLog

- 记录命令、状态、`traceId`、输入/输出 hash 和错误码。
- 同时覆盖成功、失败和被门禁拒绝的请求。
- 支持按任务、工作流和 trace 查询。

### WorkflowEvent

- 表达已经发生的领域状态变化和执行结果。
- 作为状态追踪、恢复和外部解释的稳定记录。
- 不暴露供应商或执行工具的原始响应。

## 核心契约

### WorkflowCommand

表示一次请求执行的领域命令，至少需要承载：

- 命令类型。
- 目标对象或 revision。
- 当前工作流上下文。
- 幂等键。
- 用户确认或权限上下文。
- `traceId`。

字段名称、版本化方式和错误码集合待评审确认。

### WorkflowResult

表示命令执行后的结构化结果，区分：

- 执行成功。
- 执行失败。
- 门禁拒绝。
- 重复提交命中已有结果。
- 取消或不可取消。

### WorkflowEvent

表示执行过程中已经确认发生的事件。事件必须可以与命令、结果、目标 revision 和 TaskLog 对应。

## 执行流程

```text
Agent 或 API 提交 WorkflowCommand
-> Policy Validator 校验状态、revision、审计、确认、设备和幂等键
-> 校验失败：返回结构化拒绝并写入 TaskLog / WorkflowEvent
-> 校验通过：Workflow Engine 按状态机执行状态迁移
-> 调用已经授权的执行能力
-> 写入 WorkflowResult、TaskLog 和 WorkflowEvent
```

## 强制门禁

- 没有有效审计的 revision 不得进入切片。
- 目标 revision 已失效时不得继续执行后续动作。
- 没有有效 `PrintChecklist.userConfirmedAt` 时不得启动真实打印。
- 设备能力不满足命令要求时不得执行物理动作。
- 相同幂等键不得重复创建关键记录或物理动作。

## 失败、重复和取消

- 成功、失败和拒绝都必须有可查询记录。
- 重复 Command 返回已有结果或稳定的幂等响应。
- 取消只能发生在状态机允许的阶段。
- 已完成或已产生不可逆物理动作的任务不能伪装成已取消。
- 取消语义、可取消状态和补偿策略待评审确认。

## 风险与待评审问题

- `WorkflowCommand`、`WorkflowResult` 和 `WorkflowEvent` 的版本化 Schema。
- 状态机的完整状态集合和合法迁移表。
- 幂等键的作用域、有效期和结果缓存规则。
- 取消与不可逆物理动作的边界。
- TaskLog 与 WorkflowEvent 的持久化、一致性和查询方式。
- 同步 mock 到异步执行的接口稳定性。

这些问题可以在 PR 评论或逐行 Review 中讨论；形成结论后必须回写本文件。

## 验收标准

### 验收 1：所有命令经过 Validator

Agent 或 API 提交的 Command 必须经过 Policy Validator 后才能执行；任何调用方都不能直接推进领域状态。

### 验收 2：安全门禁有效

无有效审计、目标 revision 失效、用户未确认打印或设备能力不满足时，命令被阻断并返回稳定错误码。

### 验收 3：执行记录可追溯

成功、失败和拒绝均可通过 TaskLog 与 WorkflowEvent 查询，并能关联同一 `traceId`、目标对象和输入/输出 hash。

### 验收 4：重复提交保持幂等

相同幂等键的重复 Command 不会重复创建关键记录、状态迁移或物理动作。

### 验收 5：失败与取消行为明确

失败和取消遵守状态机，返回结构化结果；非法取消不会改变任务终态。

### 验收 6：测试覆盖关键路径

测试覆盖状态门禁、revision 失效、用户确认、设备能力、幂等、失败、拒绝和取消路径。
