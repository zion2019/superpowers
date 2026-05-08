---
name: shared-spec
description: "Use when a confirmed design needs to be formalized into a specification document. Called after shared/brainstorm completes, before shared/plan."
---

# shared/spec — 设计文档编写

将经 brainstorm 确认的设计方案，编写为规范的 spec.md，经用户确认后交付 plan 阶段。

<HARD-GATE>
未经用户确认 spec，不得进入 plan 编写阶段。
自检未通过的 spec 不得提交用户审阅。
</HARD-GATE>

## 流程

1. **输入**：经 brainstorm 确认的设计方案（架构、组件、数据流等）
2. **编写文档**：按规范结构编写 spec.md，保存到调用方指定路径
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

```markdown
# [功能名称] - 设计文档

## 1. 目标
## 2. 架构
### 2.1 组件
### 2.2 数据流
## 3. 详细设计
## 4. 错误处理
## 5. 测试策略
## 6. 未完成事项
```

此为通用模板。调用方（如 tester/design）可在注入领域上下文时定制结构。

## 输出契约

1. **spec.md** — 保存到调用方指定的路径
2. **session 记录** — `shared/session.record("Spec 确认", {路径, 确认时间})`

## 后续流程

本 skill 不指定下一步。典型的后续动作由调用方决定：

| 调用方 | 后续动作 |
|--------|----------|
| `tester/design` | 确认后调用 `shared/plan` 编写执行计划 |
| 其他 Z-* | 由调用方自行编排 |
