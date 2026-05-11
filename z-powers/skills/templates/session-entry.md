# session 条目格式

## 阶段确认条目

```markdown
## [yyyy-MM-dd HH:mm] 阶段名
- 字段：值
```

## 对话条目

```markdown
### [yyyy-MM-dd HH:mm] 对话
**AI→User**
消息内容

**User→AI**
消息内容
```

## 流程状态条目

```markdown
## [yyyy-MM-dd HH:mm] 流程状态
- current_step: [当前步骤]
- pipeline: [design, spec, plan, env-config, task-runner]
- completed: [已完成步骤列表]
- resume_context: [恢复上下文]
```
