---
name: shared-spec
description: "Use to formalize a confirmed design into a specification document. Calls shared/brainstorm internally to clarify document scope and structure before writing."
---

<ACTIVATION>
1. 调用 shared/session.get("流程状态") 获取最新流程状态
2. 如果当前步骤不是 spec 且 spec 已在 completed 中，向上报告 spec 已完成
3. 如果存在 resume_context，展示给用户确认后继续
</ACTIVATION>

# shared/spec — 设计文档编写

将经 brainstorm 确认的设计方案，编写为规范的 spec.md，经用户确认后交付 plan 阶段。

## 协作关系

```
uses:
  shared/brainstorm  → 编写前澄清 spec scope 和结构偏好
  shared/session     → 记录确认结果
```

<HARD-GATE>
未经用户确认 spec 结构方案（brainstorm 输出），不得进入编写阶段。
未经用户确认 spec 内容，不得进入 plan 编写阶段。
自检未通过的 spec 不得提交用户审阅。
</HARD-GATE>

## 流程

1. **前置 brainstorm**：调用 `shared/brainstorm`，注入上下文：
   - "本次要编写一份设计文档，需要澄清文档范围和结构偏好"
   - 聚焦于：哪些模块需要覆盖、文档粒度和结构偏好
   - 产出：经确认的文档范围与结构方案（大纲或章节列表）
2. **编写文档**：基于确认结果，按规范结构编写 spec.md，保存到调用方指定路径
3. **自检**：
   - 占位符扫描：是否有 TBD、TODO、不完整章节？
   - 内部一致性：架构是否与功能描述矛盾？
   - 范围检查：聚焦于单个 plan 可完成的范围？
   - 歧义检查：是否有两种解读方式的模糊需求？
4. **用户审阅**：展示完整文档
5. **用户确认**：确认后记录 session
6. **下一步**：由调用方决定（典型场景下调用 shared/plan）

## 自检清单

发现问题立即修复，修复完成后重新自检，通过后才能提交用户审阅。

- [ ] 占位符 — 无 TBD、TODO、不完整章节
- [ ] 一致性 — 架构描述与功能描述无矛盾
- [ ] 范围 — 聚焦于单个 plan 可完成的范围
- [ ] 歧义 — 无两种解读方式的模糊需求

## 文档结构

读取并遵循 `templates/spec.md` 的文档结构。调用方注入的具体领域上下文可在模板基础上扩展。

## 输出契约

1. **spec.md** — 保存到调用方指定的路径
2. **session 记录** — `shared/session.record("Spec 确认", {路径, 确认时间})`

## 后续流程

本 skill 不指定下一步。典型的后续动作由调用方决定：

| 调用方 | 后续动作 |
|--------|----------|
| `tester/design` | 确认后调用 `shared/plan` 编写执行计划 |
| 其他 Z-* | 由调用方自行编排 |
