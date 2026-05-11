---
name: tester-execute
description: "Use when tester/orchestrator dispatches phase 2. Executes 4 steps in order: generate spec → generate plan → env check → run tests, then appends result summary to spec.md."
---

<ACTIVATION>
1. 调用 shared/session.get("流程状态") 读取最新状态
2. 根据 current_step 确定从哪个 Step 开始执行（跳过已完成的步骤）
3. 展示 resume_context 给用户确认后继续
4. 每完成一个 Step 后更新流程状态
</ACTIVATION>

# tester/execute — 测试执行

按 4 步顺序执行测试。每步调用 shared skill 完成，不包含通用流程逻辑。

## 协作关系

```
uses:
  shared/spec        → Step 1：生成 spec.md
  shared/plan        → Step 2：生成 plan.md
  shared/env-config  → Step 3：环境检查
  shared/task-runner → Step 4：执行测试
  shared/session     → 记录执行结果
```

## 流程

### Step 1：生成 spec.md

- 调用 shared/spec，输入 test-cases.md
- 从 test-cases.md 提取元信息（测试范围、测试对象、验证方式等）
- 产出：`.zion-powers/tester/[日期]_功能名/spec.md`
- 完成后：shared/session.record("流程状态", {current_step: "plan"})

### Step 2：生成 plan.md

- 调用 shared/plan，输入 test-cases.md
- 基于 test-cases.md 拆解为具体执行步骤
- 产出：`.zion-powers/tester/[日期]_功能名/plan.md`
- 完成后：shared/session.record("流程状态", {current_step: "env-config"})

### Step 3：环境检查

- 调用 shared/env-config 确认环境就绪
- 获取服务地址、Token、数据库连接等运行时信息
- 环境未就绪时立即停止并报告用户
- 完成后：shared/session.record("流程状态", {current_step: "task-runner"})

### Step 4：执行测试

<MCP-MANDATE>
所有 HTTP 请求和数据库操作必须通过 MCP 工具。详见 shared/task-runner MCP 强制规则。
</MCP-MANDATE>

- 调用 shared/task-runner，传入 plan.md 路径和 mode 参数
- task-runner 驱动循环，逐 Step 回调 tester/execute 执行具体操作
- 遇到失败立即停止并报告用户，不重复尝试
- 完成后：shared/session.record("流程状态", {current_step: "completed"})

### Step 5：追加结果摘要

- 执行完成后，将通过/失败统计追加到 spec.md 末尾
- `shared/session.record("执行完成", {通过/失败统计, 报告路径})`
- 完成后：shared/session.record("流程状态", {current_step: "completed"})

### 返回

- 返回 orchestrator，阶段二完成

## 失败处理

- 任何步骤失败，立即停止并报告用户
- 不重复尝试，不跳过步骤
- 失败信息写入 session 记录