---
name: tester-design
description: "Use when tester/orchestrator dispatches phase 1+2. Injects test context into shared/brainstorm and shared/spec to clarify test cases and produce spec.md."
---

# tester/design — 测试用例澄清与 Spec 编写

薄封装层。不包含通用流程逻辑，核心工作是注入测试领域上下文后委托 shared skill。

## 协作关系

```
uses:
  shared/brainstorm  → 逐轮提问 + 方案设计
  shared/spec        → spec.md 编写 + 自检
  shared/session     → 确认记录
```

## 流程

1. **接收** 来自 orchestrator 的测试请求（如"测试登录接口"）

2. **委托 shared/brainstorm**
   - 注入领域上下文：
     - "本次编写 HTTP 接口测试用例"
     - 确认事项：接口地址、请求方法、参数、预期状态码
   - shared/brainstorm 执行通用流程：逐轮提问 → 方案 → 用户确认

3. **委托 shared/spec**
   - 注入测试 Spec 模板（见下方模板）
   - shared/spec 编写文档 → 自检 → 用户确认

4. **记录 session**
   - `shared/session.record("Spec 确认", {路径, 确认时间})`

5. **返回** orchestrator，阶段一+二完成

## 注入模板：测试 Spec 结构

```markdown
# [接口名] - 测试设计文档

## 1. 测试范围
## 2. 测试策略
| 策略 | 说明 |
|------|------|
| 正常流程 | 验证接口在有效输入下正确响应 |
| 异常流程 | 验证接口在无效/边界输入下正确拒绝 |
| 安全场景 | 越权、未认证、参数篡改 |

## 3. 用例清单
| ID | 场景 | 请求方法 | 路径 | 参数 | 预期状态码 | 预期响应 |
|----|------|----------|------|------|-----------|---------|
| TC-001 | ... | POST | /api/... | ... | 200 | ... |

## 4. 用例详情
### TC-001: [场景名]
- 前置条件：
- 请求：
- 断言：
- 清理：
```
