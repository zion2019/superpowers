---
name: shared-plan
description: "Use to decompose a confirmed spec into bite-sized execution tasks. Calls shared/brainstorm internally to clarify execution strategy before writing the plan."
---

<ACTIVATION>
1. 调用 shared/session.get("流程状态") 获取最新流程状态
2. 如果当前步骤不是 plan 且 plan 已在 completed 中，向上报告 plan 已完成
3. 如果存在 resume_context，展示给用户确认后继续
</ACTIVATION>

# shared/plan — 执行计划编写

将经确认的 spec 分解为可独立执行的 bite-size 任务，输出 plan.md，经用户确认后交付。

## 协作关系

```
uses:
  shared/brainstorm  → 编写前澄清执行策略和分解方式
  shared/session     → 记录确认结果
```

<HARD-GATE>
未经用户确认执行策略（brainstorm 输出），不得进入分解阶段。
未经用户确认 plan，不得交付。
含占位符的 plan 不得提交用户审阅。
</HARD-GATE>

## 流程

1. **前置 brainstorm**：调用 `shared/brainstorm`，注入上下文：
   - "本次要将已确认的 spec 分解为可执行的计划，需要澄清执行策略和分解方式"
   - 聚焦于：技术栈确认、实现顺序、风险点、任务粒度偏好
   - 产出：经确认的执行策略与分解方案
2. **文件结构先行**：列出所有需要创建/修改的文件及其职责
3. **分解任务**：每步 2-5 分钟，独立可执行
4. **编写计划文档**：按规范结构编写 plan.md，保存到调用方指定路径
5. **自检**：
   - Spec 覆盖率：每个 spec 需求是否有对应任务？
   - 占位符扫描：是否有 TBD、TODO、"类似任务 N"？
   - 类型一致性：任务间函数签名、类型、属性名是否一致？
6. **用户确认**：确认后记录 session
7. **输出**：将 plan.md 路径返回调用方，由调用方决定下一步（典型场景下调用 shared/task-runner）

## 文件结构

定义任务前，先映射出所有要创建/修改的文件及其职责。这步锁定分解决策。

- 每个文件应有单一清晰职责
- 文件越小越聚焦，编辑越可靠
- 一起变更的文件放在一起，按职责拆分而非按技术层拆分
- 在现有代码库中遵循已有模式

## 任务粒度

每个步骤是一个可独立执行的动作（2-5 分钟）：

- "编写失败测试" — 步骤
- "运行确认失败" — 步骤
- "编写最小实现" — 步骤
- "运行确认通过" — 步骤

## 无占位符

以下均为 plan 的失败模式，写入即 violation：

- TBD、TODO、"implement later"
- "添加适当的错误处理"（没有具体代码）
- "类似任务 N"（重复完整代码）
- "编写测试验证以上功能"（没有具体测试代码）
- 步骤描述怎么做但不展示代码（代码步骤必须含代码块）

## 自检清单

编写完成后对照检查，发现问题立即修复：

- [ ] Spec 覆盖率：每条 spec 需求都有对应任务
- [ ] 占位符扫描：无 TBD、TODO、"类似任务 N"
- [ ] 类型一致性：任务间的函数签名、类型、属性名一致

## 文档结构

读取并遵循 `templates/plan.md` 的文档结构。

## 输出契约

1. **plan.md** — 保存到调用方指定的路径
2. **session 记录** — `shared/session.record("Plan 确认", {路径, 任务总数})`

## 后续流程

本 skill 不指定下一步。典型的后续动作由调用方决定：

| 调用方 | 后续动作 |
|--------|----------|
| `tester/plan` | 确认后调用 `shared/task-runner` 执行任务 |
| 其他 Z-* | 由调用方自行编排 |

## 执行交接

Plan 确认后，必须询问用户选择执行方式：

> "Plan 已保存到 `<路径>`。两种执行方式：
>
> **1. Subagent 执行（推荐）** — 每个 Task 派发独立子代理，Task 间自动审查
> **2. Inline 执行** — 在当前会话中按序执行，每 2-3 个 Task 设检查点
>
> **选择哪种？**"

### 若选 Subagent
- 调用 shared/task-runner 时传入 mode: subagent
- task-runner 为每个 Task 派发独立子代理，做两阶段审查

### 若选 Inline
- 调用 shared/task-runner 时传入 mode: inline
- 在当前会话中按序执行，每 2-3 个 Task 暂停展示进展
