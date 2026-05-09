# Z-Powers

Java/Spring Boot 接口测试工作流技能包。基于 Superpowers 设计思想，提供通用流程层（brainstorm → spec → plan → env-config → execute）。

## 技能列表

| 技能 | 用途 |
|------|------|
| `tester/orchestrator` | 入口：编排两阶段测试流程 |
| `shared/brainstorm` | 需求澄清与方案设计 |
| `shared/spec` | 设计文档编写 |
| `shared/plan` | 执行计划编写 |
| `shared/env-config` | 运行时环境配置管理（多项目） |
| `shared/session` | 会话日志记录 |
| `shared/task-runner` | 任务执行管理 |

## 安装

### Claude Code

将 `z-powers` 目录复制到你的项目目录下，或配置 Claude Code 的插件路径使其发现 `.claude-plugin/plugin.json`。

使用方式：

```
Skill("tester/orchestrator")
```

### OpenCode

将 `z-powers/skills/` 复制或链接到 OpenCode 可发现的技能路径（如 `~/.agents/skills/` 或项目 `skills/` 目录），然后通过 `skill` 工具加载：

```
skill("tester/orchestrator")
```

## 工作流

```
用户触发测试 → tester/orchestrator
  ├── tester/design (注入测试上下文 + shared/brainstorm)
  │    输出: test-cases.md
  └── tester/execute
        ├── Step 1: shared/spec (前置 brainstorm → 编写 spec.md)
        ├── Step 2: shared/plan (前置 brainstorm → 编写 plan.md)
        ├── Step 3: shared/env-config (多项目环境配置)
        └── Step 4: shared/task-runner (执行测试)
```
