---
name: tester-design
description: "Use when tester/orchestrator dispatches phase 1. Injects test context into shared/brainstorm to clarify test cases and produce test-cases.md."
---

# tester/design — 测试用例澄清

薄封装层。不包含通用流程逻辑，核心工作是注入测试领域上下文后委托 shared skill，产出 test-cases.md。

## 协作关系

```
uses:
  shared/brainstorm  → 逐轮提问 + 方案设计
  shared/session     → 确认记录
```

## 流程

1. **接收** 来自 orchestrator 的测试请求（如"测试登录接口"）

2. **委托 shared/brainstorm**
   - 注入领域上下文：
     - "本次编写 HTTP 接口测试用例"
     - 确认事项：接口地址、请求方法、参数、预期状态码
   - shared/brainstorm 执行通用流程：逐轮提问 → 方案 → 用户确认

3. **编写 test-cases.md**
   - 基于确认的设计方案，编写测试用例文档
   - 保存到 `.zion-powers/tester/[yyyy-MM-dd]_[功能名]/test-cases.md`

4. **记录 session**
   - `shared/session.record("Test-Cases 确认", {路径, 确认时间})`

5. **返回** orchestrator，阶段一完成

## 文档结构

读取并遵循 `templates/test-cases.md` 的结构。具体测试内容由 shared/brainstorm 确认后填充。
```