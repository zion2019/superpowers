---
name: shared-session
description: "Use when any Z-* skill needs to persist phase confirmation records or retrieve historical session data. Referenced via `uses:` by other shared/ and domain/ skills."
---

<ACTIVATION>
激活后首先执行以下步骤：

1. 读取 session 文件中最近一条"流程状态"记录
2. 如果有流程状态，将其作为当前上下文加载
3. 后续所有操作基于此状态继续
4. 如果无流程状态，视为首次激活
</ACTIVATION>

# shared/session — 会话管理

以增量追加方式记录 Z-Powers 各阶段的关键节点，供追溯和上下文恢复。

## 接口契约

```
session.record(phase, content)
  - phase: 阶段标识
  - content: 记录内容（结构化数据）

session.get(phase)
  - 返回指定阶段的历史记录
```

## 记录时机与内容

由调用方在对应阶段完成后调用 `record`。记录内容根据阶段不同：

| 时机 | phase 值 | 记录内容 |
|------|----------|----------|
| 每次 AI 输出 + 用户回复后 | `对话` | ai_msg、user_msg、时间戳 |
| 任务开始 | `任务开始` | 功能名称、用户原始输入、时间戳 |
| 设计方案确认 | `设计确认` | 方案概要、用户确认意见 |
| env 配置确认 | `环境配置确认` | 配置项摘要、环境列表 |
| Spec 确认 | `Spec 确认` | spec.md 路径、确认时间 |
| Plan 确认 | `Plan 确认` | plan.md 路径、任务总数 |
| 执行完成 | `执行完成` | 通过/失败统计、报告路径 |
| 流程状态变更 | `流程状态` | current_step、pipeline 数组、completed 数组、resume_context |

调用方可根据自身需要扩展 phase 值和记录内容。

## 文件位置

session 文件路径由 **调用方（orchestrator）** 指定，session skill 不决定路径。典型路径：

```
.zion-powers/tester/2026-05-07_login-function/session.md
```

## 操作规则

### record(phase, content)

追加新条目到 session 文件末尾。**永不覆盖已有内容。**

1. 文件不存在则创建
2. 追加格式化的条目
3. 每一条目包含时间戳 + phase + 内容

### get(phase)

1. 读取 session 文件全部内容
2. 按 phase 值筛选出匹配的条目
3. 返回匹配的条目列表（时间正序）

## 条目格式

每个条目是一个 H2 标题块：

```markdown
## [2026-05-07 10:30] 设计确认
- 用户输入：测试登录接口
- AI 执行：逐轮提问，确认测试范围
- 关键决策：专注正常/异常/锁定三种场景
- 下一步：等待用户确认设计方案

## [2026-05-07 10:35] Spec 确认
- spec.md 已确认
- 下一步：编写执行计划
```

格式规范：
- H2 标题 `## [时间戳] phase名`
- 条目内容使用无序列表 `- 键：值`
- 条目间空一行分隔
- 时间戳格式 `yyyy-MM-dd HH:mm`

### 对话条目格式

对话条目记录每个 AI↔User 的交互来回：

```markdown
### [2026-05-09 10:01] 对话
**AI→User**
逐轮提问澄清测试范围：你希望覆盖哪些场景？

**User→AI**
正常登录、密码错误、账号锁定三种
```

格式规范：
- H3 标题 `### [时间戳] 对话`
- 对话内容使用 `**AI→User**` 和 `**User→AI**` 标记方向
- AI 消息和用户消息之间空一行

## 文件结构示例

一个完整的 session.md 包含对话日志和阶段确认记录：

```markdown
# 任务会话日志
- 功能：测试登录接口
- 任务目录：`.zion-powers/tester/2026-05-09_login/session.md`
- 开始时间：2026-05-09 10:00

### [2026-05-09 10:01] 对话
**AI→User**
你希望覆盖哪些场景？

**User→AI**
正常登录、密码错误、账号锁定

### [2026-05-09 10:02] 对话
**AI→User**
账号锁定期望行为？

**User→AI**
5次错误锁定30分钟

## [2026-05-09 10:05] 设计确认
- 方案：正常/异常/锁定三种场景
- 用户确认：同意

### [2026-05-09 10:06] 对话
**AI→User**
现在开始编写 spec？

## [2026-05-11 14:00] 流程状态
- current_step: env-config
- pipeline: [design, spec, plan, env-config, task-runner]
- completed: [design, spec, plan]
- resume_context: 用户刚确认 plan.md，准备进入 env-config
```


## 对话后自检（强制）

每次用户回复后、执行任何操作前，以最高优先级自检：

1. 当前处于 pipeline 哪个步骤？（从 session 读取最近一条"流程状态"）
2. 用户的回复是流程内确认，还是拓展讨论？
3. 如果是拓展讨论 → 处理完后必须：
   a. 回顾 session 中的"流程状态"
   b. 告知用户："回到 z-powers 流程，继续 [当前步骤]"
   c. 回写当前步骤的产出到对应文件
4. 如果上一步有未完成的 session 记录 → 补写 `session.record()`
