---
name: tester-plan
description: "Use when tester/orchestrator dispatches phase 3. Injects test task decomposition template into shared/plan to produce plan.md."
---

# tester/plan — 测试执行计划编写

薄封装层。注入测试任务分解模板到 `shared/plan`。

## 协作关系

```
uses:
  shared/plan        → 任务分解 + 自检 + 确认
  shared/session     → 确认记录
```

## 流程

1. **接收** 来自 orchestrator 的「编写测试执行计划」请求

2. **委托 shared/plan**
   - 注入测试领域上下文：
     - "本次为 HTTP 接口测试编写执行计划"
     - 用例来源：spec.md（含用例清单）
     - 环境来源：`shared/env-config`（执行前独立确认）
   - 注入测试任务分解模板（见下方模板）
   - shared/plan 分解任务 → 自检 → 用户确认

3. **记录 session**
   - `shared/session.record("Plan 确认", {路径, 任务总数})`

4. **返回** orchestrator，阶段三完成

## 注入模板：测试 Plan 结构

```markdown
# [功能名称] - 测试执行计划 (Plan)

## 前置准备
- [ ] 执行前 `shared/env-config` 确认环境就绪

## 执行任务清单

### TC-001: [场景名]
- [ ] 1. 数据准备：...
- [ ] 2. 执行请求：POST /api/...
- [ ] 3. 断言：...
- [ ] 4. 验证日志：...
- [ ] 5. 数据清理：...

### TC-002: [场景名]
...
```

## 测试任务分解规则

每个测试用例分解为 5 个 Step，每个 Step 2-5 分钟：

| Step | 内容 | 示例 |
|------|------|------|
| 1. 数据准备 | 写入测试前置数据到数据库 | `INSERT INTO users ...` |
| 2. 执行请求 | 发送 HTTP 请求到被测接口 | `POST /api/login` |
| 3. 断言 | 验证响应状态码和 body | 状态码 200，body 含 token |
| 4. 验证日志 | 查询日志确认无异常（可选） | 日志中无 ERROR |
| 5. 数据清理 | 清理测试数据 | `DELETE FROM users ...` |
