---
title: Z-Powers V5 流程保真度增强设计
subtitle: 6 项缺陷修复，根治 AI 遗忘流程、掉环节、乱执行
version: V5
date: 2026-05-11
---

# Z-Powers V5 流程保真度增强设计

## 1. 背景

Z-Powers V4 的 shared/ + tester/ 分层架构在文档层面完整，但实际使用时 AI 频繁出现以下问题：

1. spec 和 plan 阶段之间的 brainstorm 被跳过
2. brainstorm 只用开放性问题，不给选择题
3. 中途拓展讨论后 AI 遗忘 pipeline 位置，不回写 spec、不更新 plan、不记录 session
4. env-config 跳过 Maven/JDK 采集，不主动使用 MCP，浪费时间探索鉴权
5. plan 完成后不询问 subagent vs inline 执行方式
6. 编译启动命令混乱（`mvn spring-boot:run` 而非 `mvn clean package` → `java -jar`）

根因：文档声明 ≠ 运行时约束。缺少状态持久化、恢复机制、强制模板和反模式声明。

## 2. 改动总览

| 修复项 | 涉及文件 | 新增/修改 |
|--------|----------|-----------|
| 1. session 状态恢复 | `shared/session`, `tester/orchestrator`, `tester/execute` | 改 |
| 2. 选择题模板 | `shared/brainstorm`, `shared/env-config`, `templates/brainstorm-question.md` | 新增 + 改 |
| 3. build/run 分离 + 日志重定向 | `shared/env-config`, `design.md` | 改 |
| 4. MCP 指令 + 鉴权反模式 | `shared/env-config`, `tester/execute`, `shared/task-runner` | 改 |
| 5. 执行交接 | `shared/plan`, `shared/task-runner` | 改 |
| 6. 对话后自检 | `shared/session` | 改 |

## 3. 详细设计

### 3.1 session 状态恢复

**问题**：AI 中途讨论后丢失 pipeline 上下文。

**方案**：复用 `shared/session`，增加"流程状态"为内置 phase。每个 skill 激活时首先读取 session 恢复位置。

#### session 状态记录格式

```markdown
## [2026-05-11 14:00] 流程状态
- current_step: env-config
- pipeline: [design, spec, plan, env-config, task-runner]
- completed: [design, spec, plan]
- resume_context: 用户刚确认 plan.md
```

#### shared/session 改动

- "流程状态"加入内置 phase 列表
- 增加 `<ACTIVATION>` 自检块（见 3.6）

#### tester/orchestrator 改动

在每个阶段完成后调用：
`session.record("流程状态", {current_step, pipeline, completed, resume_context})`

#### tester/execute 改动

激活时第一步：读取 session 中"流程状态"，跳到 `current_step` 继续执行。

---

### 3.2 选择题模板化

**问题**：brainstorm 声明"优先选择题"但无强制模板，AI 仍用开放问题。

**方案**：新建 `templates/brainstorm-question.md`，`shared/brainstorm` 和 `shared/env-config` 引用此模板。

#### 目录结构

```
z-powers/skills/
├── templates/                           ← 新增
│   ├── brainstorm-question.md           # 选择题模板
│   ├── test-cases.md                    # 从 tester/design 抽出
│   ├── spec.md                          # 从 shared/spec 抽出
│   ├── plan.md                          # 从 shared/plan 抽出
│   └── session-entry.md                 # session 条目格式
```

#### templates/brainstorm-question.md 内容

```
**Question (N/总数):** [问题]

Options:
A. [选项A — 推荐时标注 (Recommended)]
B. [选项B]
C. [选项C]
D. 自定义 — 输入你的答案

**绝对不允许只问开放性问题而不给选项。** 即使选项是推测的，也比完全开放更好。
```

#### shared/brainstorm 改动

第 48-51 行"关键原则"替换为：
> 读取并严格遵循 `templates/brainstorm-question.md` 中的格式。

#### shared/env-config 改动

鉴权方式、DB 环境选择等分支项使用选择题模板。
确定性采集（路径、端口）仍可直接问。

---

### 3.3 Build / Run 命令分离 + 日志重定向

**问题**：`env-context.json` 只有一个 `start_command`，AI 乱用。

**方案**：拆为 `build.command` + `run.command`，并自动追加日志重定向。

#### env-context.json 格式变更

```json
"build": {
  "command": "mvn clean package -DskipTests",
  "project_dir": "E:/project/my-app"
},
"run": {
  "command": "java -jar target/my-app-1.0.0.jar",
  "project_dir": "E:/project/my-app",
  "log_file": ".zion-powers/tester/{{session_dir}}/log/app.log"
}
```

移除原有的 `run.start_command` 和 `jvm.start_command`。

#### shared/env-config 采集流程变更

- ⑤ 编译命令："项目的完整编译命令是什么？（如 `mvn clean package -DskipTests`）工作目录？"
- ⑥ 启动命令："编译产物的启动命令是什么？（如 `java -jar target/xxx.jar`）"

启动命令采集后自动拼接日志重定向：
> `java -jar target/xxx.jar > .zion-powers/tester/{{session_dir}}/log/app.log 2>&1`

其中 `{{session_dir}}` 由 orchestrator 创建运行时目录时注入。

两个命令均走「采集后显式确认」流程。

#### task-runner / tester/execute 改动

执行失败时，自动读取 `run.log_file` 内容排查。

---

### 3.4 MCP 使用指令 + 鉴权反模式

**问题 A**：AI 收集了 MCP server 名但不知道要实际使用。

**方案**：在 `tester/execute` Step 4 和 `shared/task-runner` 增加：

```markdown
<MCP-MANDATE>
任何 HTTP 请求操作必须通过 [http_mcp_server_name] MCP 工具发起。
数据库查询操作必须通过 [db_mcp_server_name] MCP 工具执行。
禁止自行构造 curl、HttpClient、RestTemplate 等本地 HTTP 调用。
</MCP-MANDATE>
```

执行时，`http_mcp_server_name` 和 `db_mcp_server_name` 从 `env-context.json` 读取后替换。

**问题 B**：AI 探索项目代码找鉴权方式，浪费时间。

**方案**：`shared/env-config` 鉴权采集增加：

```markdown
<ANTI-PATTERN>
禁止通过读取项目代码、配置文件、或调用任何工具来推测鉴权方式。
只能通过询问用户获取。
</ANTI-PATTERN>
```

鉴权采集使用选择题模板：

> A. 直接提供 token
> B. 描述获取步骤
> C. 自定义

---

### 3.5 执行交接（subagent vs inline）

**问题**：plan 完成后无执行方式选择。

**方案**：`shared/plan` 末尾增加执行交接段落。

```markdown
## 执行交接

Plan 确认后，必须询问：

> "Plan 已保存。两种执行方式：
> **1. Subagent 执行（推荐）** — 每个 Task 派发独立子代理，Task 间自动审查
> **2. Inline 执行** — 在当前会话中按序执行，每 2-3 个 Task 设检查点
> **选择哪种？**"

### 若选 Subagent
- 调用 shared/task-runner 时 mode: subagent
- task-runner 为每个 Task 派发独立子代理

### 若选 Inline
- 调用 shared/task-runner 时 mode: inline
- 在当前会话中按序执行
```

`shared/task-runner` 增加 `mode` 参数，分两条执行路径。

---

### 3.6 对话后自检（流程闭环）

**问题**：用户拓展讨论后 AI 不回写产出、不更新状态。

**方案**：在 `shared/session` 顶部增加自检规则，由于 session 被所有 skill 通过 `uses:` 引用，此规则带入每个技能上下文。

```markdown
## 对话后自检（强制）

每次用户回复后、执行任何操作前，以最高优先级自检：

1. 当前处于 pipeline 哪个步骤？（从 session 读取"流程状态"）
2. 用户的回复是流程内确认，还是拓展讨论？
3. 如果是拓展讨论 → 处理完后必须：
   a. 回顾 session 中的"流程状态"
   b. 告知用户："回到 z-powers 流程，继续 [当前步骤]"
   c. 回写当前步骤的产出到对应文件
4. 如果上一步有未完成的 session 记录 → 补写 session.record()
```

---

## 4. 改动文件清单

| 文件 | 操作 | 内容 |
|------|------|------|
| `skills/templates/brainstorm-question.md` | **新增** | 选择题强制模板 |
| `skills/templates/test-cases.md` | **新增** | 从 tester/design 内联抽出 |
| `skills/templates/spec.md` | **新增** | 从 shared/spec 内联抽出 |
| `skills/templates/plan.md` | **新增** | 从 shared/plan 内联抽出 |
| `skills/templates/session-entry.md` | **新增** | session 条目格式模板 |
| `skills/shared/session/SKILL.md` | **改** | +流程状态 phase, +对话后自检, +ACTIVATION |
| `skills/shared/brainstorm/SKILL.md` | **改** | 提问格式引用模板 |
| `skills/shared/spec/SKILL.md` | **改** | 文档结构引用模板 + ACTIVATION |
| `skills/shared/plan/SKILL.md` | **改** | +执行交接, 文档结构引用模板 + ACTIVATION |
| `skills/shared/env-config/SKILL.md` | **改** | build/run分离, 日志重定向, +鉴权反模式, +选择题格式 |
| `skills/shared/task-runner/SKILL.md` | **改** | +mode参数, +MCP强制规则 |
| `skills/tester/orchestrator/SKILL.md` | **改** | +流程状态记录 |
| `skills/tester/design/SKILL.md` | **改** | test-cases结构引用模板 |
| `skills/tester/execute/SKILL.md` | **改** | +MCP强制规则, +ACTIVATION状态恢复 |
| `design.md` | **改** | 同步 env-context.json 格式变更 |

## 5. 保持不变的部分

- 四层架构（Plugin / Shared / Domain / Runtime）
- skill 引用关系（`uses:` 声明）
- HARD-GATE 门禁机制
- 两阶段门禁流程（Design → Execute）
- 铁律约束（禁止跳过门禁、失败快速反馈等）

## 6. 未完成事项

无。
