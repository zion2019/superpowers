---
name: shared-task-runner
description: "Use when a confirmed plan.md needs to be executed task by task. Reads plan, tracks progress with checkboxes, manages checkpoints. Domain executors (e.g. tester/execute) reference this via uses:."
---

# shared/task-runner — 计划执行管理

读取 plan.md，按序执行任务，实时更新进度，控制检查点审核。被 domain executor（如 `tester/execute`）通过 `uses:` 引用。

<HARD-GATE>
未经用户确认 plan，task-runner 不得开始执行。
plan.md 文件不存在或含未确认标记时，不得执行。
</HARD-GATE>

## 协作关系

```
uses:
  shared/session     → 记录执行结果
```

domain executor 注入具体执行逻辑（如 HTTP 请求、编译命令、DB 查询），task-runner 管理执行流程和进度。

## 流程

```mermaid
flowchart TD
    START([接收 plan.md 路径]) --> LOAD[解析 plan.md\n提取全部 Task]
    LOAD --> NEXT{还有未完成任务?}
    NEXT -->|是| TASK[取下一个 Task]
    TASK --> MARK_START[标记 [~] 执行中]
    MARK_START --> STEP{还有未完成 Step?}
    STEP -->|是| EXEC[执行当前 Step]
    EXEC --> OK{成功?}
    OK -->|是| MARK_DONE[标记 Step 完成]
    MARK_DONE --> STEP
    OK -->|否| RETRY{已重试过?}
    RETRY -->|否| FIX[自动修复后重试]
    FIX --> EXEC
    RETRY -->|是| FAIL[标记 [✗] 失败]
    FAIL --> REPORT[暂停，报告根因给用户]
    REPORT --> DECIDE{用户决定}
    DECIDE -->|继续| STEP
    DECIDE -->|跳过| MARK_SKIP[标记跳过]
    MARK_SKIP --> NEXT
    STEP -->|全部完成| MARK_TASK_DONE[标记 [✓] 完成]
    MARK_TASK_DONE --> CHECKPOINT{达到检查点?\n每2-3个Task}
    CHECKPOINT -->|是| REVIEW[暂停，展示进展\n请求用户审核]
    REVIEW --> NEXT
    CHECKPOINT -->|否| NEXT
    NEXT -->|全部完成| DONE[记录 session\n返回调用方]
```

## 进度更新

实时修改 plan.md 中的复选框：

```
- [ ]  未开始
- [~]  执行中
- [✓]  完成
- [✗]  失败（需人工介入）
```

每次状态变更立即写回 plan.md，确保断点可恢复。

## 检查点规则

- 每完成 2-3 个 Task 暂停一次
- 展示已完成/失败/剩余的任务概览
- 用户审核通过后继续
- 若用户要求调整代码，在当前 session 修复后继续后续 Task

## 重试策略

默认策略（domain executor 可通过注入覆盖）：

| 失败类型 | 默认行为 |
|----------|----------|
| 代码/测试失败 | 修复后重试 1 次 |
| 环境/基础设施问题 | 不重试，立即暂停报告根因 |
| 断言不符合预期 | 分析根因，暂停报告 |

重试 1 次后仍失败 → 暂停，向用户报告上下文和根因，由用户决定：继续 / 跳过 / 终止。

## 输出契约

1. **更新后的 plan.md** — 包含最终进度标记
2. **session 记录** — `shared/session.record("执行完成", {通过/失败统计, 报告路径})`
3. **执行报告** — 返回调用方，包含各 Task 结果摘要

## 回调契约

task-runner 驱动 Step 循环，每个 Step 通过回调执行实际操作：

```
task-runner 执行循环（对每个 Step）:
  1. 从 plan.md 读取当前 Step 的原文内容
  2. 回调 domain executor: "执行这个 Step"
     → domain executor 解析 Step 原文，执行具体操作
     → 返回 成功 / 失败（含错误信息）
  3. 根据回调结果更新 plan.md 复选框
  4. 如失败，按重试策略处理
```

domain executor 不需要关心循环控制、进度写入、检查点调度——只负责"收到一个 Step，执行它，返回结果"。

## 被 domain executor 使用

```
domain executor（如 tester/execute）在 execute 阶段：

1. 引用 shared/env-config → 获取 MCP/Maven/JVM/日志配置
2. 引用 shared/task-runner → 传入 plan.md 路径
3. task-runner 开始循环，逐个 Step 回调 domain executor
4. domain executor 解析 Step 原文，执行测试操作
5. 重复直到全部完成
```
