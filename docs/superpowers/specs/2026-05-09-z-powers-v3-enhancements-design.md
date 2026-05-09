---
title: Z-Powers V3 增强设计
subtitle: spec/plan 引入 brainstorm 前置阶段 + env-context 多项目结构 + session 完整对话日志
version: V3
author: superpowers
date: 2026-05-09
---

# Z-Powers V3 增强设计

# 1. 版本历史

| 版本 | 作者 | 日期 | 描述 |
|------|------|------|------|
| V3 | superpowers | 2026-05-09 | 基于 V2 设计的三项增强：spec/plan 前置 brainstorm、env-context 多项目结构、session 完整对话日志 |

# 2. 目标

基于 Z-Powers V2（设计文档 `z-powers/design.md`）进行三项增强：

1. **shared/spec 和 shared/plan 新增 brainstorm 前置阶段** — 在编写文档前调用 shared/brainstorm 澄清 scope 和执行策略
2. **shared/env-config 重构为多项目结构** — env-context.json 支持多独立项目配置，新增鉴权采集
3. **shared/session 扩展为完整对话日志** — 记录每个 AI↔User 来回，满足审计追溯需求

# 3. 设计原则

- **复用现有 shared/brainstorm** — 不新增 skill，只在 spec/plan 入口处复用已有流程
- **env-context 集中管理多项目** — 单文件多项目结构，避免开发者维护多个配置文件
- **session 增量不破坏** — 在现有阶段确认记录基础上追加对话日志，不改变已有格式

# 4. 设计详述

## 4.1 shared/spec 新增 brainstorm 前置阶段

### 当前流程

```
输入设计构思 → 编写 spec.md → 自检 → 用户确认
```

### 目标流程

```
输入设计构思
  → 前置 brainstorm：调用 shared/brainstorm
     注入上下文："本次要编写一份设计文档"
     shared/brainstorm 通用流程：
       逐轮提问澄清需求 → 2-3 种方案 → 用户确认
  → 编写 spec.md
  → 自检
  → 用户确认
```

### 关键决策

- **brainstorm 聚焦于"怎么写 spec"** — 不重新讨论设计方案本身，只澄清 spec 的受众、覆盖维度、结构偏好
- **使用 shared/brainstorm 通用流程** — 问题由运行时动态生成，不在 skill 中硬编码
- **注入上下文由调用方提供** — 如 tester/execute 调用时注入"测试领域 spec 需要包含接口列表和用例表格"
- **HARD-GATE** — 同现有：未经用户确认不得进入编写

### 协作关系变化

```
shared/spec（更新后）
  uses:
    shared/brainstorm  → 新增：编写前澄清 spec scope
    shared/session     → 记录确认
```

## 4.2 shared/plan 新增 brainstorm 前置阶段

### 当前流程

```
输入 spec.md → 分解任务 → 编写 plan.md → 自检 → 用户确认
```

### 目标流程

```
输入 spec.md
  → 前置 brainstorm：调用 shared/brainstorm
     注入上下文："本次要将已确认的 spec 分解为可执行的计划"
     shared/brainstorm 通用流程：
       逐轮提问澄清需求 → 2-3 种方案 → 用户确认
  → 分解任务 → 编写 plan.md → 自检 → 用户确认
```

### 关键决策

- **brainstorm 聚焦于"怎么分解计划"** — 澄清技术栈确认、实现顺序、风险点
- 与 spec 的区别仅在于注入的上下文不同：spec 注入"编写设计文档"，plan 注入"分解为执行计划"
- **HARD-GATE** — 同现有

### 协作关系变化

```
shared/plan（更新后）
  uses:
    shared/brainstorm  → 新增：编写前澄清执行策略
    shared/session     → 记录确认
```

## 4.3 env-context.json 多项目结构重构

### 文件位置

`.zion-powers/env-context.json`（不变，作为当前项目通用环境配置）

### JSON 结构

```json
{
  "version": 2,
  "updated_at": "2026-05-09 10:00",
  "current_project": "project-a",
  "projects": {
    "project-a": {
      "auth": {
        "method": "direct_token",
        "token": "eyJhbGciOiJIUzI1NiIs...",
        "acquire": null,
        "acquire_credentials": null
      },
      "build": {
        "jdk_home": "C:/Program Files/Java/jdk-17",
        "maven_home": "C:/tools/maven",
        "maven_settings": "C:/Users/xxx/.m2/settings.xml"
      },
      "run": {
        "jdk_home": "C:/Program Files/Java/jdk-17",
        "start_command": "mvn spring-boot:run",
        "project_dir": "E:/project-a"
      },
      "db": {
        "environments": [
          {
            "name": "dev",
            "host": "localhost",
            "port": 3306,
            "database": "app_dev",
            "username": "app_user",
            "constraints": ["禁止使用 root 账号"]
          }
        ]
      },
      "mcp_servers": {
        "db_mcp": "dev-db-mcp",
        "http_mcp": "local-http-mcp",
        "base_url": "http://localhost:8080"
      },
      "logs": {
        "directory": "E:/project-a/logs"
      }
    },
    "project-b": { }
  }
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | int | 结构版本，递增为 2 |
| `current_project` | string | 当前激活的项目 key |
| `projects` | object | key 为用户自定义项目名称，value 为项目配置 |
| `auth.method` | enum | `"direct_token"` 或 `"acquire"` |
| `auth.token` | string | method=direct_token 时，token 值直接存储 |
| `auth.acquire` | string | method=acquire 时，自由文本描述获取方式 |
| `auth.acquire_credentials` | object | method=acquire 时，获取 token 所需的凭据 |
| `build` | object | 编译配置：jdk、maven、maven settings |
| `run` | object | 启动配置：jdk、启动命令、工作目录 |
| `db` | object | 数据库连接信息，多环境支持，禁止 root |
| `mcp_servers` | object | MCP server 名称和 base_url，已从顶层移到项目级 |
| `logs` | object | 日志目录，已从顶层移到项目级 |

### 与原结构对比变化

| 变化项 | V1 | V3 |
|--------|----|-----|
| 项目支持 | 单项目，配置在顶层 | 多项目，配置在 `projects.<name>` 下 |
| 鉴权 | 无 | 新增 auth 字段 |
| MCP | 顶层 | 移入项目级 |
| 日志 | 顶层 | 移入项目级 |
| 构建配置 | jvm 对象 | 改为 build 对象，语义更清晰 |

## 4.4 shared/env-config 流程调整

### 流程变化

```mermaid
flowchart TD
    START([被 Z-* 引用]) --> CHECK{env-context.json\n是否存在?}
    CHECK -->|是| SHOW_PROJECTS[展示已配置项目列表\n含 current_project 标记]
    SHOW_PROJECTS --> CHOOSE[询问: 本次为哪个项目工作?\n选择已有 / 新增项目]
    CHOOSE -->|选择已有| SUMMARY[展示配置摘要]
    SUMMARY --> CONFIRM{用户确认?}
    CONFIRM -->|否| RECONFIG[标记为需重配\n进入逐项引导]
    CONFIRM -->|是| RECORD[shared/session\nrecord 环境配置确认]
    RECORD --> RETURN[返回配置数据\n给调用方]
    RECONFIG --> Q_NAME
    CHOOSE -->|新增项目| Q_NAME[① 项目名称]
    CHECK -->|否| Q_NAME
    
    Q_NAME --> Q_AUTH[② 鉴权方式\ndirect_token 或 acquire]
    Q_AUTH --> Q_BUILD_JDK[③ JDK 路径]
    Q_BUILD_JDK --> Q_BUILD_MVN[④ Maven 路径 + settings]
    Q_BUILD_MVN --> Q_RUN_CMD[⑤ 启动命令 + 工作目录\n采集后显式确认:"是这个命令?"]
    Q_RUN_CMD --> Q_DB[⑥ DB 环境信息]
    Q_DB --> Q_MCP[⑦ MCP server + base_url]
    Q_MCP --> Q_LOG[⑧ 日志目录]
    Q_LOG --> SAVE[写入 env-context.json\n保存到对应项目下]
    SAVE --> SHOW_FULL[展示完整配置\n让用户最终确认]
    SHOW_FULL --> RECORD
```

### 新增/变更的采集步骤

| 顺序 | 配置项 | 变化说明 |
|------|--------|----------|
| ① | 项目名称 | 新增 — 用户输入简短名称作为 projects key |
| ② | 鉴权 | 新增 — 询问 direct_token 还是 acquire，分别采集 |
| ③④ | 编译配置 | 同现有，但写入项目级 |
| ⑤ | 启动命令 | **显式确认** — 采集后回显命令，用户必须确认"是，正确"才能写入 |
| ⑥⑦⑧ | DB/MCP/日志 | 同现有，但写入项目级 |

### 启动命令显式确认规则

```
采集后回显：
  "确认启动命令：mvn spring-boot:run  (工作目录: E:/project-a)
  是这个命令没错？(是/否)"
用户必须回答"是"才能写入。
未确认时，env-config 返回"启动命令未确认"状态给调用方。
```

### 职责边界

| 职责 | 归属 |
|------|------|
| 采集配置 | shared/env-config |
| 采集后显式确认启动命令 | shared/env-config |
| 启动项目 | 调用方（如 tester/execute） |
| 验证 DB 连通性 | 调用方 |
| 验证 HTTP 接口可达性 | 调用方 |

## 4.5 shared/session 扩展完整对话日志

### 当前设计

只记录阶段确认点：

- 任务开始
- 设计确认
- Spec 确认
- Plan 确认
- 环境配置确认
- 执行完成

### 目标设计

保留现有阶段确认记录，**新增完整对话日志**：

```
session.md 结构：
┌─ # 任务会话日志（元信息头部）
├─ ## [时间戳] 对话           ← 新增：每个 AI↔User 来回
├─ ## [时间戳] 对话           ← 新增
├─ ## [时间戳] 设计确认       ← 保留原有格式
├─ ## [时间戳] 对话           ← 新增
├─ ## [时间戳] Spec 确认      ← 保留原有格式
└─ ...
```

### 新增记录格式

```markdown
### [2026-05-09 10:01] 对话
**AI→User**
逐轮提问澄清测试范围：你希望覆盖哪些场景？

**User→AI**
正常登录、密码错误、账号锁定三种

### [2026-05-09 10:02] 对话
**AI→User**
账号锁定场景的预期行为是什么？

**User→AI**
连续5次密码错误后锁定30分钟

### [2026-05-09 10:05] 设计确认
- 方案：专注正常/异常/锁定三种场景
- 用户确认：同意
```

### 记录时机

| 记录点 | 由谁调用 | 内容 |
|--------|----------|------|
| 每个 AI 输出后 | 当前执行 skill（如 tester/design） | AI 消息内容 |
| 每个 User 输入后 | 当前执行 skill | 用户消息内容 |
| 阶段确认时 | 当前执行 skill | 确认摘要（同现有格式）|

### 接口扩展

```
// 新增 phase 值
session.record("对话", {ai_msg: string, user_msg: string})
  → 写入 ### 对话 条目

// 现有不变
session.record("设计确认", {...})
session.record("Spec 确认", {...})
...
```

### 与现有记录的兼容

- 现有 `session.get(phase)` 按 phase 筛选，`"对话"` 是新增 phase 值，不影响已有查询
- 对话条目使用 `###` 三级标题，阶段确认使用 `##` 二级标题，视觉上容易区分
- 文件仍按任务存放于 `.zion-powers/tester/[日期]_[功能名]/session.md`

# 5. 协作关系变化总览

## Skill 引用关系（更新后）

```
Z-Tester Orchestrator (入口 skill)
  uses:
    ├── tester/design
    │   uses:
    │     ├── shared/brainstorm  → 不变
    │     └── shared/session     → 不变
    ├── tester/execute
    │   uses:
    │     ├── shared/spec        → 更新：前置 brainstorm
    │     │   uses:
    │     │     ├── shared/brainstorm  → 新增引用
    │     │     └── shared/session
    │     ├── shared/plan        → 更新：前置 brainstorm
    │     │   uses:
    │     │     ├── shared/brainstorm  → 新增引用
    │     │     └── shared/session
    │     ├── shared/env-config  → 更新：多项目流程
    │     │   uses:
    │     │     └── shared/session
    │     ├── shared/task-runner → 不变
    │     └── shared/session     → 扩展记录粒度
    └── shared/session           → 扩展记录粒度
```

# 6. 与 V2 的兼容性

| 变化 | 兼容性 | 说明 |
|------|--------|------|
| spec 前置 brainstorm | 向前兼容 | 现有调用方调用 shared/spec 时会自动经历新阶段 |
| plan 前置 brainstorm | 向前兼容 | 现有调用方调用 shared/plan 时会自动经历新阶段 |
| env-context 新结构 | **不兼容** | V1 格式的 env-context.json 需迁移：检测 version=1 时提示用户重新采集 |
| session 对话日志 | 向后兼容 | 追加新格式，不影响已有查询和记录 |

# 7. 未完成事项

- env-context.json 从 V1 到 V2 的自动迁移策略（当前设计为检测 version=1 时提示重新采集）
- 调用方（tester/execute）启动项目和验证连通性的具体实现
