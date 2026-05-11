# Z-Powers V5 流程保真度增强 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 z-powers 增加 6 项运行时约束（状态恢复、选择题模板、build/run 分离、MCP 指令、执行交接、对话自检），根治 AI 遗忘流程、掉环节问题。

**Architecture:** 在 shared/ 层注入 ACTIVATION 块和状态读取逻辑；新增 `templates/` 目录抽取内联模板；env-config 拆分 build/run 命令；plan 增加执行交接；task-runner 增加 mode 参数。

**Tech Stack:** Markdown (SKILL.md), JSON (env-context.json)

---

### Task 1: 模板目录 + 文件创建

**Files:**
- Create: `z-powers/skills/templates/brainstorm-question.md`
- Create: `z-powers/skills/templates/test-cases.md`
- Create: `z-powers/skills/templates/spec.md`
- Create: `z-powers/skills/templates/plan.md`
- Create: `z-powers/skills/templates/session-entry.md`

- [ ] **Step 1: 创建 templates/brainstorm-question.md**

```markdown
# 选择题提问模板（强制）

每次向用户提澄清问题时，必须使用以下格式：

**Question (N/总数):** [问题]

Options:
A. [选项A — 推荐时标注 (Recommended)]
B. [选项B]
C. [选项C]
D. 自定义 — 输入你的答案

**约束**
- 绝对不允许只问开放性问题而不给选项。即使选项是推测的，也比完全开放更好——用户可以用 D 修正。
- 优先提供 2-3 个选项。
- 推荐项标注 (Recommended) 并排在第一位。
```

- [ ] **Step 2: 创建 templates/test-cases.md**

```markdown
# test-cases.md 结构模板

# [接口名] — 测试用例

## 元信息
- 目标系统：
- 测试范围：
- 验证方式：
- 数据策略：

## 测试对象

### A 类：Starter 同步接口
| 编号 | 方法 | 路径 | 说明 |
|------|------|------|------|

### B 类：业务接口增量同步触发
| 编号 | 业务操作 | 方法 | 路径 | 同步类型 |
|------|----------|------|------|----------|

### C 类：异常场景
| 编号 | 场景 | 说明 |
|------|------|------|

## 用例

### A1-1：[用例名]
- 请求：
- 预期：
- 中间表验证：

## 执行顺序
1. A 类 → B 类 → C 类
2. 清理测试数据

## 中间表验证 SQL
\`\`\`sql
...
\`\`\`
```

- [ ] **Step 3: 创建 templates/spec.md**

```markdown
# spec.md 结构模板

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

- [ ] **Step 4: 创建 templates/plan.md**

```markdown
# plan.md 结构模板

# [功能名称] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development or executing-plans.

**Goal:** [一句话描述]

**Architecture:** [2-3 句]

**Tech Stack:** [关键技术栈]

---

### Task N: [组件名]

**Files:**
- Create: `path/to/file`
- Modify: `path/to/file:123-145`
- Test: `tests/path/to/test.py`

- [ ] **Step 1: [具体操作]**
  \`\`\`
  具体代码或命令
  \`\`\`
- [ ] **Step 2: [具体操作]**
```

- [ ] **Step 5: 创建 templates/session-entry.md**

```markdown
# session 条目格式

## 阶段确认条目

\`\`\`markdown
## [yyyy-MM-dd HH:mm] 阶段名
- 字段：值
\`\`\`

## 对话条目

\`\`\`markdown
### [yyyy-MM-dd HH:mm] 对话
**AI→User**
消息内容

**User→AI**
消息内容
\`\`\`

## 流程状态条目

\`\`\`markdown
## [yyyy-MM-dd HH:mm] 流程状态
- current_step: [当前步骤]
- pipeline: [design, spec, plan, env-config, task-runner]
- completed: [已完成步骤列表]
- resume_context: [恢复上下文]
\`\`\`
```

---

### Task 2: shared/session 增强 — 流程状态 + 对话后自检 + ACTIVATION

**Files:**
- Modify: `z-powers/skills/shared/session/SKILL.md`

- [ ] **Step 1: 在文件顶部插入 ACTIVATION 自检块**

在 `---` frontmatter 之后、`# shared/session` 之前插入：

```markdown
<ACTIVATION>
激活后首先执行以下步骤：

1. 读取 session 文件中最近一条"流程状态"记录
2. 如果有流程状态，将其作为当前上下文加载
3. 后续所有操作基于此状态继续
4. 如果无流程状态，视为首次激活
</ACTIVATION>
```

- [ ] **Step 2: 在"记录时机与内容"表格中增加"流程状态"行**

在表格末尾追加：

```markdown
| 流程状态变更 | `流程状态` | current_step、pipeline 数组、completed 数组、resume_context |
```

- [ ] **Step 3: 在文件末尾追加"对话后自检"规则**

```markdown
## 对话后自检（强制）

每次用户回复后、执行任何操作前，以最高优先级自检：

1. 当前处于 pipeline 哪个步骤？（从 session 读取最近一条"流程状态"）
2. 用户的回复是流程内确认，还是拓展讨论？
3. 如果是拓展讨论 → 处理完后必须：
   a. 回顾 session 中的"流程状态"
   b. 告知用户："回到 z-powers 流程，继续 [当前步骤]"
   c. 回写当前步骤的产出到对应文件
4. 如果上一步有未完成的 session 记录 → 补写 `session.record()`
```

- [ ] **Step 4: 在"文件结构示例"中追加"流程状态"示例**

```markdown
## [2026-05-11 14:00] 流程状态
- current_step: env-config
- pipeline: [design, spec, plan, env-config, task-runner]
- completed: [design, spec, plan]
- resume_context: 用户刚确认 plan.md，准备进入 env-config
```

---

### Task 3: shared/brainstorm 增强 — 模板引用

**Files:**
- Modify: `z-powers/skills/shared/brainstorm/SKILL.md`

- [ ] **Step 1: 替换"关键原则"列表（约第 80-87 行）为模板引用**

找到：
```markdown
## 关键原则

- **一次一问** — 不一次抛出多个问题
- **选择题优先** — 比开放题更容易回答
...
```

替换为：

```markdown
## 提问格式（强制）

读取并严格遵循 `templates/brainstorm-question.md` 的格式。

## 关键原则

- **一次一问** — 不一次抛出多个问题
- **YAGNI 无情** — 从所有设计中移除不必要的功能
- **探索替代方案** — 在定案前始终提出 2-3 种方案
- **增量验证** — 逐段呈现设计，确认后再继续
- **灵活调整** — 遇到不合理时回溯澄清
```

---

### Task 4: shared/spec 增强 — 模板引用 + ACTIVATION

**Files:**
- Modify: `z-powers/skills/shared/spec/SKILL.md`

- [ ] **Step 1: 在 frontmatter 后插入 ACTIVATION 块**

```markdown
<ACTIVATION>
1. 调用 shared/session.get("流程状态") 获取最新流程状态
2. 如果当前步骤不是 spec 且 spec 已在 completed 中，向上报告 spec 已完成
3. 如果存在 resume_context，展示给用户确认后继续
</ACTIVATION>
```

- [ ] **Step 2: 替换内联"文档结构"（约第 50-62 行）为模板引用**

找到：
```markdown
## 文档结构

\`\`\`markdown
# [功能名称] - 设计文档
...
\`\`\`
```

替换为：

```markdown
## 文档结构

读取并遵循 `templates/spec.md` 的文档结构。调用方注入的具体领域上下文可在模板基础上扩展。
```

---

### Task 5: shared/plan 增强 — 执行交接 + 模板引用 + ACTIVATION

**Files:**
- Modify: `z-powers/skills/shared/plan/SKILL.md`

- [ ] **Step 1: 在 frontmatter 后插入 ACTIVATION 块**

```markdown
<ACTIVATION>
1. 调用 shared/session.get("流程状态") 获取最新流程状态
2. 如果当前步骤不是 plan 且 plan 已在 completed 中，向上报告 plan 已完成
3. 如果存在 resume_context，展示给用户确认后继续
</ACTIVATION>
```

- [ ] **Step 2: 替换内联"文档结构"（约第 76-102 行）为模板引用**

```markdown
## 文档结构

读取并遵循 `templates/plan.md` 的文档结构。
```

- [ ] **Step 3: 在"后续流程"段落后追加"执行交接"段**

```markdown
## 执行交接

Plan 确认后，必须询问用户选择执行方式：

> "Plan 已保存到 `<路径>`。两种执行方式：
>
> **1. Subagent 执行（推荐）** — 每个 Task 派发独立子代理，Task 间自动审查
> **2. Inline 执行** — 在当前会话中按序执行，每 2-3 个 Task 设检查点
>
> **选择哪种？**"

### 若选 Subagent
- 调用 shared/task-runner 时传入 mode: subagent
- task-runner 为每个 Task 派发独立子代理，做两阶段审查

### 若选 Inline
- 调用 shared/task-runner 时传入 mode: inline
- 在当前会话中按序执行，每 2-3 个 Task 暂停展示进展
```

---

### Task 6: shared/env-config 增强 — build/run 分离 + 鉴权反模式 + 选择题

**Files:**
- Modify: `z-powers/skills/shared/env-config/SKILL.md`

- [ ] **Step 1: 修改 mermaid 流程图中的⑤⑥节点**

将原 `Q_MVN`、`Q_JDK`、`Q_START` 三个节点替换为：
```
Q_JDK → Q_MVN → Q_BUILD → Q_RUN → Q_DB
```

- [ ] **Step 2: 替换"配置项明细"表格（约第 50-61 行）**

将 ③④⑤ 改为：

| 顺序 | 配置项 | 引导词 | 约束 |
|------|--------|--------|------|
| ① | 项目名称 | "请给这个项目一个简短名称（如 my-app）？" | 唯一标识 |
| ② | 鉴权 | 读取 `templates/brainstorm-question.md` 格式："鉴权 token 如何获取？" A. 直接提供 token B. 描述获取步骤 C. 自定义 | **禁止探索鉴权** |
| ③ | JDK 路径 | "项目使用的 JDK 安装路径是？" | 确认路径存在 |
| ④ | Maven 路径 | "Maven 安装目录和 settings.xml 文件路径是？" | 确认路径存在 |
| ⑤ | 编译命令 | "项目的完整编译命令是什么？（如 `mvn clean package -DskipTests`）工作目录在哪？" | 采集后显式确认 |
| ⑥ | 启动命令 | "编译产物的启动命令是什么？（如 `java -jar target/xxx.jar`）" | 采集后自动拼接日志重定向：`> .zion-powers/tester/{{session_dir}}/log/app.log 2>&1`，显式确认 |
| ⑦ | DB 信息 | 使用 `templates/brainstorm-question.md` 格式 | 禁止 root；密码不记录 |
| ⑧ | MCP server | "本地 HTTP MCP server 名和 base_url？" | 确认端口可访问 |
| ⑨ | 日志目录 | "项目运行时日志输出到哪个目录？" | 确认目录存在 |

- [ ] **Step 3: 在鉴权行下方插入反模式声明**

```markdown
<ANTI-PATTERN>
禁止通过读取项目代码、配置文件、或调用任何工具来推测鉴权方式。
只能通过询问用户获取。
</ANTI-PATTERN>
```

- [ ] **Step 4: 替换"文件格式"中的 JSON 示例（约第 65-111 行）**

将 `build` 和 `run` 字段改为：

```json
{
  "version": 2,
  "updated_at": "2026-05-09 10:00",
  "current_project": "my-app",
  "projects": {
    "my-app": {
      "auth": {
        "method": "direct_token",
        "token": "eyJhbGciOiJIUzI1NiIs...",
        "acquire": null,
        "acquire_credentials": null
      },
      "build": {
        "command": "mvn clean package -DskipTests",
        "project_dir": "E:/project/my-app"
      },
      "run": {
        "command": "java -jar target/my-app-1.0.0.jar",
        "project_dir": "E:/project/my-app",
        "log_file": ".zion-powers/tester/2026-05-09_login/log/app.log"
      },
      "db": { ... },
      "mcp_servers": {
        "db_mcp": "dev-db-mcp",
        "http_mcp": "local-http-mcp",
        "base_url": "http://localhost:8080"
      },
      "logs": {
        "directory": "E:/project/my-app/logs"
      }
    }
  }
}
```

移除原来的 `jvm` 和 `maven` 顶层字段（已合并到 `build`）。

- [ ] **Step 5: 更新"启动命令确认说明"段**

将标题改为"编译/启动命令确认说明"，增加：
```markdown
### 编译命令确认
1. 回显完整命令 + 工作目录
2. 用户必须明确回答"是，正确"

### 启动命令确认
1. 回显完整命令（含自动追加的日志重定向）
2. 确认日志目录存在（不存在则自动创建）
3. 用户必须明确回答"是，正确"
4. 未确认时返回"未确认"状态给调用方
```

---

### Task 7: shared/task-runner 增强 — mode 参数 + MCP 强制规则

**Files:**
- Modify: `z-powers/skills/shared/task-runner/SKILL.md`

- [ ] **Step 1: 在"流程"段上面增加 mode 参数说明**

```markdown
## 执行模式

调用方通过 mode 参数指定：

| mode | 行为 |
|------|------|
| `subagent` | 每个 Task 派发独立子代理，Task 间做两阶段审查（spec 合规 + 代码质量） |
| `inline` (默认) | 在当前会话中按序执行，每 2-3 个 Task 暂停检查点 |
```

- [ ] **Step 2: 在"被 domain executor 使用"段中增加 MCP 强制规则**

在末尾追加：

```markdown
## MCP 强制规则

<MCP-MANDATE>
执行过程中：
- 任何 HTTP 请求操作，必须通过 env-context.json 中 `mcp_servers.http_mcp` 指定的 MCP 工具发起
- 任何数据库查询操作，必须通过 `mcp_servers.db_mcp` 指定的 MCP 工具执行
- 禁止自行构造 curl、HttpClient、RestTemplate 等本地 HTTP 调用
- 禁止直接 JDBC 连接数据库

执行前从 env-context.json 读取 `mcp_servers.http_mcp` 和 `mcp_servers.db_mcp` 确定具体 MCP server 名。
</MCP-MANDATE>
```

- [ ] **Step 3: 在"失败处理"中增加日志排查指令**

```markdown
### HTTP 调用失败排查

1. 读取 env-context.json 中 `run.log_file` 查看应用日志
2. 将日志中相关异常信息展示给用户
3. 不要自行猜测原因——用日志说话
```

---

### Task 8: tester/orchestrator 增强 — 流程状态记录

**Files:**
- Modify: `z-powers/skills/tester/orchestrator/SKILL.md`

- [ ] **Step 1: 修改"阶段一：委托 tester/design"段**

在"等待 tester/design 返回"后追加：

```markdown
- 调用 shared/session.record("流程状态", {
    current_step: "design",
    pipeline: ["design", "spec", "plan", "env-config", "task-runner"],
    completed: <动态>根据当前完成情况填充,
    resume_context: "阶段一完成，test-cases.md 已产出"
  })
```

- [ ] **Step 2: 修改"阶段二：委托 tester/execute"段**

在开头追加：

```markdown
- 调用 shared/session.record("流程状态", {
    current_step: "spec",
    pipeline: ["design", "spec", "plan", "env-config", "task-runner"],
    completed: ["design"],
    resume_context: "准备进入阶段二：生成 spec"
  })
```

---

### Task 9: tester/design 增强 — 模板引用

**Files:**
- Modify: `z-powers/skills/tester/design/SKILL.md`

- [ ] **Step 1: 替换内联"test-cases.md 结构"（约第 39-77 行）**

将整个"## 注入模板：test-cases.md 结构"替换为：

```markdown
## 文档结构

读取并遵循 `templates/test-cases.md` 的结构。具体测试内容由 shared/brainstorm 确认后填充。
```

---

### Task 10: tester/execute 增强 — ACTIVATION + MCP 强制规则

**Files:**
- Modify: `z-powers/skills/tester/execute/SKILL.md`

- [ ] **Step 1: 在 frontmatter 后插入 ACTIVATION 块**

```markdown
<ACTIVATION>
1. 调用 shared/session.get("流程状态") 读取最新状态
2. 根据 current_step 确定从哪个 Step 开始执行（跳过已完成的步骤）
3. 展示 resume_context 给用户确认后继续
4. 每完成一个 Step 后更新流程状态
</ACTIVATION>
```

- [ ] **Step 2: 在 Step 4 "执行测试"段增加 MCP 强制规则**

```markdown
### Step 4：执行测试

<MCP-MANDATE>
所有 HTTP 请求和数据库操作必须通过 MCP 工具。详见 shared/task-runner MCP 强制规则。
</MCP-MANDATE>

- 调用 shared/task-runner，传入 plan.md 路径和 mode 参数
```

- [ ] **Step 3: 在每一步结束后增加状态记录**

在 Step 1-5 各段末尾追加：
```markdown
- 完成后：shared/session.record("流程状态", {current_step: <更新>})
```

---

### Task 11: design.md 同步更新

**Files:**
- Modify: `z-powers/design.md`

- [ ] **Step 1: 更新 env-context.json 格式示例（约第 427-467 行）**

将 `maven`、`jvm` 字段替换为 `build`、`run` 字段，与 Task 6 一致。

- [ ] **Step 2: 在"配置项说明"表格中同步更新（约第 416-425 行）**

将 Maven/JDK/JVM 相关行替换为编译/启动两步骤。

- [ ] **Step 3: 在"目录结构"中增加 templates/ 目录（约第 85-108 行）**

在 `skills/` 树中追加：
```markdown
    └── templates/
        ├── brainstorm-question.md
        ├── test-cases.md
        ├── spec.md
        ├── plan.md
        └── session-entry.md
```

---

### Task 12: 完整性验证

**Files:**
- (验证，不修改文件)

- [ ] **Step 1: 检查所有 SKILL.md 的 uses: 声明与 ACTIVATION 引用的 skill 名一致**

- [ ] **Step 2: 检查所有模板引用路径是否正确（`templates/xxx.md`）**

- [ ] **Step 3: 检查 design.md 与各 SKILL.md 之间无矛盾**
