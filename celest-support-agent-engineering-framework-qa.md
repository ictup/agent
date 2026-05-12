# Celest Support Agent 工程框架搭建与专业问答手册

版本：2026-05-12  
定位：专门回答“这个 Agent 系统在工程上是如何一步步搭建起来的，每一步使用了什么技术，以及如何在面试中专业解释”的文档  
适用场景：

- 简历项目深挖
- AI Agent 工程架构面试
- 垂直领域 Agent 系统设计答辩
- 从 0 到 1 复刻 Celest Support 类 Agent 平台
- 将该工程框架迁移到金融、保险、医疗、运维、客服等其他领域

---

## 目录

1. [核心结论](#1-核心结论)
2. [整体工程技术栈](#2-整体工程技术栈)
3. [工程框架总览](#3-工程框架总览)
4. [从 0 到 1 的工程搭建步骤](#4-从-0-到-1-的工程搭建步骤)
5. [每一层为什么这样选技术](#5-每一层为什么这样选技术)
6. [关键源码路径对应关系](#6-关键源码路径对应关系)
7. [通用迁移模板](#7-通用迁移模板)
8. [专业面试 Q&A](#8-专业面试-qa)
9. [最终总结话术](#9-最终总结话术)

---

# 1. 核心结论

Celest Support Agent 不是用一个单独的 Agent 框架搭起来的，也不是简单的 LangChain Demo。

它本质上是一个由多层工程模块组成的生产级垂直领域 Agent 系统：

```text
TypeScript / Bun 工程
+ Hono 后端 Gateway
+ SolidJS 前端工作台
+ OpenCode / Codex / Claude Agent Runtime
+ Markdown Agent 指令与业务知识
+ TypeScript Skill Scripts
+ Proposal 人审系统
+ SSE 状态同步
+ Optool 内部后台适配
+ ClickHouse / Graylog / Splunk Evidence 系统
+ Docker Compose 部署
+ 测试与审计体系
```

一句话概括工程架构：

```text
这是一个 TypeScript + Bun + Hono + SolidJS + OpenCode Runtime 搭建的垂直领域 Agent 工作流平台，通过 Skill 工具层、Backend Gateway、Proposal 审批、SSE 状态同步和 Docker 部署，把 LLM 安全嵌入真实客服业务流程。
```

最重要的工程思想：

```text
LLM 不是直接接生产系统；
LLM 通过 Skill 调 Backend Gateway；
写操作不直接执行，而是生成 Proposal；
人类在前端审批 Proposal；
后端再校验、执行、审计。
```

---

# 2. 整体工程技术栈

## 2.1 技术栈总表

| 工程层 | 使用技术 | 作用 |
|---|---|---|
| 语言 | TypeScript | 前后端、Skill、Proposal 类型统一 |
| 运行时 | Bun | 本地开发、脚本执行、后端运行 |
| 后端框架 | Hono | 实现 `/support/*` API Gateway |
| 前端框架 | SolidJS | 构建 Support Case 工作台 |
| 前端构建 | Vite | 开发服务器与前端构建 |
| Agent Runtime | OpenCode / Codex / Claude Session | 启动和运行 Agent 会话 |
| Skill 工具 | TypeScript CLI scripts | 封装 Agent 可调用工具 |
| 实时同步 | SSE / EventSource | 后端向前端推送 case/proposal/session 状态 |
| Live Store | 内存 Map | 保存实时 proposal/session 状态 |
| 历史存储 | SQLite side-channel | 保存 proposal 历史和 timeline |
| 内部后台 | Optool HTTP/XML | 真实客服、玩家、账号、物品操作系统 |
| 数据仓库 | ClickHouse | 历史经济、物品、活动等证据 |
| 日志系统 | Graylog | 服务端日志证据 |
| 崩溃系统 | Splunk | 客户端 crash evidence |
| Bug 系统 | Jira | 明确要求时创建 SBR bug/task |
| 部署 | Docker Compose | 多服务编排 |
| 本地命令 | justfile | 安装、构建、运行、Docker 操作 |
| 插件注册 | `.codex-plugin/plugin.json` | 注册 Celest Support skill plugin |
| Agent 配置 | `opencode.json` | 注册模型和 skill paths |

## 2.2 为什么不是单一 Agent 框架

很多面试官会问：

```text
你是不是用了 LangChain / LangGraph？
```

这个项目的重点不是某个 Agent 编排框架，而是生产工程闭环。

即使换成 LangGraph，也仍然需要：

```text
Backend Gateway
Skill Scripts
Proposal Store
Proposal Executor
Allowlist
Script Contract
Auth Gate
Frontend Workbench
SSE State
Audit
Tests
Deployment
```

所以 Celest Support 的核心能力不是来自某个 Agent 框架，而是来自：

```text
业务流程建模 + 工具封装 + 后端权限 + 人审执行 + 审计追踪
```

---

# 3. 工程框架总览

## 3.1 总体架构图

```text
客服浏览器
  |
  v
SolidJS Support UI
  |
  | HTTP + SSE
  v
Hono / Bun Support Backend Gateway
  |
  +--> Optool Session / Operator Auth
  +--> Agent Token Store
  +--> Read Allowlist
  +--> Proposal Store
  +--> Proposal Executor
  +--> Support State Store + SSE
  +--> Agent Session Launcher
  |
  v
OpenCode / Codex / Claude Agent Runtime
  |
  +--> agent.md
  +--> policy/playbook markdown
  +--> TypeScript Skill Scripts
  +--> Evidence Subagents
  |
  v
业务系统 / 数据系统
  +--> Optool
  +--> ClickHouse
  +--> Graylog
  +--> Splunk
  +--> Jira
```

## 3.2 请求链路

以“客服点击 Solve”为例：

```text
1. 客服在 SolidJS UI 打开 case。
2. 前端调用 Hono backend 的 `/support/cases/:id`。
3. Backend 从 Optool 拉 case detail。
4. 客服点击 Solve。
5. 前端 POST `/support/cases/:id/solve`。
6. Backend 创建 scoped agent token。
7. Backend 构造 Agent prompt，注入 case JSON。
8. Backend 启动 OpenCode/Codex/Claude session。
9. Agent 读取 agent.md、policy、playbook。
10. Agent 调用 TypeScript skill scripts。
11. Skill scripts 调 Backend `/support/*` API。
12. Backend 按 allowlist 查询 Optool / 数据源。
13. Agent 基于证据创建 proposal。
14. Backend 校验并保存 proposal。
15. Backend 通过 SSE 推送给前端。
16. 客服在 UI approve / deny。
17. Backend 二次校验。
18. Proposal Executor 执行 Optool command。
19. Backend 写审计并 SSE 更新状态。
```

---

# 4. 从 0 到 1 的工程搭建步骤

## Step 1：搭建 Monorepo / 工程骨架

### 使用技术

```text
Bun workspace
TypeScript
justfile
Docker Compose
```

### 当前对应文件

```text
package.json
opencode/package.json
justfile
docker/docker-compose.yaml
```

### 搭建内容

建立一个多模块工程：

```text
repo/
  agents/
    celest-support/
  apps/
    terminal-webui/
  opencode/
    packages/
      app/
  docker/
  docs/
  justfile
  opencode.json
```

### 每个目录的职责

| 目录 | 职责 |
|---|---|
| `agents/celest-support` | Agent 本体、skills、policy、playbook |
| `apps/terminal-webui` | Hono 后端、terminal/session backend |
| `opencode/packages/app` | SolidJS Support UI |
| `docker` | Docker Compose 和镜像配置 |
| `docs` | 架构、复刻、面试文档 |
| `justfile` | 本地开发命令 |
| `opencode.json` | Agent / skill 注册 |

### 常用命令

```powershell
just install
just build
```

### 这一步的意义

把 Agent 系统拆成工程模块，而不是把所有代码塞进一个脚本。

---

## Step 2：配置 Agent Runtime 与 Skill Discovery

### 使用技术

```text
OpenCode config
Codex plugin manifest
JSON 配置
本地 skill path discovery
```

### 当前对应文件

```text
opencode.json
agents/celest-support/.codex-plugin/plugin.json
```

### `opencode.json` 的作用

注册默认 agent、模型和 skill path：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "default_agent": "celest",
  "model": "anthropic/claude-sonnet-4-6",
  "skills": {
    "paths": [
      "agents/celest-support/skills"
    ]
  }
}
```

### `.codex-plugin/plugin.json` 的作用

声明 Celest Support 插件：

```json
{
  "name": "celest-support",
  "version": "0.1.0",
  "skills": "./skills/"
}
```

### 这一步的意义

让 Agent Runtime 能发现并加载：

```text
Celest Support 的 Skill 工具目录
```

---

## Step 3：创建 Agent 本体

### 使用技术

```text
Markdown prompt
文件系统上下文
Agent instruction loading
```

### 当前对应文件

```text
agents/celest-support/agent.md
```

### 这个文件定义

```text
Agent 身份
Operating Contract
Canonical Sources
Support Knowledge Boundary
Proposal 规则
玩家回复规则
Subagent 使用规则
Truth & Source Discipline
```

### 工程意义

`agent.md` 不是业务说明文档，而是 Agent 的运行规约。

它约束 Agent：

```text
先读政策；
必须检查附件；
把玩家说法拆成 claims；
使用 approved skills；
不直接执行 mutation；
所有写操作走 proposal；
玩家回复必须政策安全。
```

### 易错点

错误写法：

```text
你是一个客服助手，请帮助玩家解决问题。
```

正确写法：

```text
你是内部客服调查员。你必须按 policy/playbook 调查，只能用 approved tools，所有写操作必须创建 proposal。
```

---

## Step 4：创建业务知识层

### 使用技术

```text
Markdown
文件系统读取
rg 搜索
Agent context
```

### 当前对应目录

```text
agents/celest-support/resources/
```

### 核心文件

```text
RULES-POLICIES.md
DECISION_FRAMEWORK.md
CASE_PLAYBOOK.md
case_playbook_library/
```

### 每个文件作用

| 文件 | 作用 |
|---|---|
| `RULES-POLICIES.md` | 绑定客服政策 |
| `DECISION_FRAMEWORK.md` | 证据充分性和 actionability 判断 |
| `CASE_PLAYBOOK.md` | case 类型路由 |
| `case_playbook_library/*.md` | 具体场景调查 SOP |

### 工程意义

这层让 Agent 从：

```text
通用语言模型
```

变成：

```text
被业务政策和 SOP 约束的垂类 Agent
```

---

## Step 5：搭建 Skill / Tool 目录

### 使用技术

```text
TypeScript CLI scripts
Bun 执行
SKILL.md 描述文件
fetch 调 Backend API
JSON 输入输出
```

### 当前对应目录

```text
agents/celest-support/skills/
```

### 当前 Skill

```text
support-cases
player-information
player-actions
player-account
jira-bug-report
self-improvement
```

### 每个 Skill 结构

```text
skills/<skill-name>/
  SKILL.md
  scripts/
    *.ts
```

### Skill 分类

| Skill | 类型 | 职责 |
|---|---|---|
| `support-cases` | read + proposal | 工单读取、回复、关闭、升级 |
| `player-information` | read | 玩家只读调查 |
| `player-actions` | proposal | 玩家状态/补偿/物品等操作提案 |
| `player-account` | read + proposal | 账号、支付、安全、银行等 |
| `jira-bug-report` | direct special flow | 明确要求时创建 Jira bug |
| `self-improvement` | proposal | 支持流程改进提案 |

### 工程原则

```text
Read scripts 查询证据。
Proposal scripts 只创建 proposal，不直接写生产系统。
```

---

## Step 6：搭建 Skill 公共库

### 使用技术

```text
TypeScript helper functions
fetch
环境变量
共享 wire contract
```

### 当前对应文件

```text
agents/celest-support/lib/support-read.ts
agents/celest-support/lib/support-proposal.ts
```

### `support-read.ts` 负责

```text
读取 support backend base URL
读取 agent token
构造请求 headers
调用 /support/* read endpoints
统一处理错误
```

### `support-proposal.ts` 负责

```text
定义 SupportProposalInput
定义 ProposalExecution
定义 ProposalOptoolStep
构造 proposal presentation
校验 player_message
提交 POST /support/proposals
```

### 工程意义

避免每个 skill script 重复实现：

```text
认证
请求
proposal schema
错误处理
消息格式校验
```

---

## Step 7：搭建 Backend Gateway

### 使用技术

```text
TypeScript
Bun
Hono
hono/cookie
hono/streaming
Node crypto
Node path/fs
```

### 当前对应文件

```text
apps/terminal-webui/src/server/support-routes.ts
```

### 主要职责

```text
Optool 登录
Agent token
Support case API
Attachment API
Read proxy
Player/account report
Proposal create
Proposal approve/deny
SSE stream
Agent session launcher
Jira bug report
```

### 主要路由

```text
GET  /support/status
GET  /support/optool-status
POST /support/optool-login
POST /support/agent-token
GET  /support/cases
GET  /support/cases/:id
GET  /support/cases/:id/attachments
POST /support/player-view
GET  /support/account-report/:account_id
POST /support/proposals
POST /support/proposals/:id/approve
POST /support/proposals/:id/deny
GET  /support/state
GET  /support/stream
POST /support/cases/:id/analyze
POST /support/cases/:id/solve
```

### 工程意义

Backend Gateway 是 Agent 和内部业务系统之间的安全隔离层。

---

## Step 8：接入 Optool / 内部后台

### 使用技术

```text
HTTP
XML / structured response parsing
Cookie/session
Optool command abstraction
```

### 当前实现位置

```text
apps/terminal-webui/src/server/support-routes.ts
```

### Optool 提供

```text
support case
player data
account data
item data
payment data
security data
write commands
```

### 工程原则

```text
Agent 不直接访问 Optool。
Agent 不拿 Optool credentials。
Backend 使用 operator session 执行 approved proposal。
```

---

## Step 9：实现 Operator Auth / Optool Session

### 使用技术

```text
Hono cookie
in-memory Map
session expiration
decision cookie
```

### 当前实现概念

```text
optoolSessions: Map<email, OptoolCreds>
optoolDecisionSessions: Map<sessionId, OptoolDecisionSession>
```

### 作用

```text
客服在浏览器登录 Optool。
后端保存 operator session。
approve proposal 时用 operator session 执行。
```

### 为什么这样做

```text
Agent token 只用于调查；
operator session 才用于真实执行；
审批责任归属于人类操作员。
```

---

## Step 10：实现 Agent Token

### 使用技术

```text
randomUUID / random token
HTTP header
in-memory session store
expiration
scope validation
```

### 当前请求头

```text
x-celest-support-agent-token
```

### Token 限制

```text
绑定 case_id
绑定 account_id / planet_id
限制 proposal:create
限制 related accounts 数量
限制过期时间
```

### 工程意义

Agent 不是登录用户，不应该拥有全局权限。  
它只能拥有当前 case 任务范围内的最小权限。

---

## Step 11：实现 Read Allowlist

### 使用技术

```text
TypeScript Set
TypeScript Map
route validation
view template
field whitelist
```

### 当前实现概念

```text
SUPPORT_READ_COMMANDS
SUPPORT_SEARCH_VIEWS
PLAYER_VIEW_COLUMNS
```

### 作用

限制 Agent 只能执行被允许的只读查询。

例如：

```text
Account.Find
Player.Stats
Player.LocateAvatar
Item.Find
Item.Data
Quest.ShowPlayerData
StandardReply.List
```

### 为什么只读也要限制

因为只读也可能泄露：

```text
账号隐私
支付信息
安全设备
内部字段
无关玩家数据
风控信息
```

---

## Step 12：定义 Proposal Wire Contract

### 使用技术

```text
TypeScript interfaces
JSON payload
frontend/backend shared shape
```

### Agent 侧文件

```text
agents/celest-support/lib/support-proposal.ts
```

### Backend 侧文件

```text
apps/terminal-webui/src/server/support/proposal-store.ts
```

### 核心结构

```ts
interface Proposal {
  id: string
  case_id: number
  account_id: number
  operation: string
  planet_id: number
  params: Record<string, unknown>
  reason: string
  player_message?: string
  status: "pending" | "approved" | "executed" | "failed" | "denied"
  presentation: ProposalPresentation
  execution: ProposalExecution
}
```

### 关键设计

```text
operation 是展示和审计标签。
execution.steps 是真正执行计划。
```

---

## Step 13：实现 Proposal Store

### 使用技术

```text
TypeScript Map
Subscriber pattern
SQLite side-channel
Event callbacks
```

### 当前对应文件

```text
apps/terminal-webui/src/server/support/proposal-store.ts
```

### 职责

```text
create
get
list
mark approved
mark denied
mark executed
mark failed
evict by case
subscribe
mirror to SQLite
```

### 当前设计

```text
内存 Map 是 live approval flow 的 source of truth。
SQLite 用于 durable timeline / historical rendering。
```

---

## Step 14：实现 Proposal Allowlist

### 使用技术

```text
TypeScript Set
validation function
command allowlist
```

### 当前对应文件

```text
apps/terminal-webui/src/server/support/proposal-allowlist.ts
```

### 作用

限制 proposal approval 后能执行的命令。

### 示例命令类型

```text
SupportCase.AddReply
SupportCase.Close
Player.Place
Player.Logout
Account.ChangeEmail
Item.Recreate
Vehicle.Recall
__support_noop__
__support_create_improvement_manifest__
```

### 工程意义

即使 Agent 创建了 proposal，后端也只执行 allowlist 中的命令。

---

## Step 15：实现 Script Contract

### 使用技术

```text
TypeScript Map
source validation
filesystem path validation
operation-command mapping
```

### 当前实现位置

```text
apps/terminal-webui/src/server/support-routes.ts
```

### 校验内容

```text
created_by_skill
created_by_script
operation
execution command
script path 是否存在
script 是否允许创建该 operation
script 是否允许使用该 command
```

### 解决的问题

防止低风险脚本伪造成高风险 proposal。

例如：

```text
create-reply-to-support-case-proposal.ts
不能伪造 compensate_player_ped。
```

---

## Step 16：实现 Proposal Executor

### 使用技术

```text
TypeScript
async functions
dependency injection
data-driven execution
structured error handling
```

### 当前对应文件

```text
apps/terminal-webui/src/server/support/proposal-executor.ts
```

### 核心设计

Executor 不根据 `operation` switch。

它做：

```text
读取 proposal.execution.steps
校验 step
替换 player_message
调用 executeOptool
处理失败
mark executed / failed
emit state
```

### 好处

```text
新增操作不需要改 executor 主逻辑；
proposal 自带执行计划；
执行统一可审计；
更适合迁移到其他领域。
```

---

## Step 17：实现 Support State Store + SSE

### 使用技术

```text
Backend in-memory state
Hono streamSSE
EventSource
JSON delta events
```

### Backend 文件

```text
apps/terminal-webui/src/server/support/support-state-store.ts
```

### Frontend 文件

```text
opencode/packages/app/src/context/support-store.tsx
```

### 流程

```text
GET /support/state
EventSource /support/stream
backend emits delta
frontend updates Solid store
```

### 工程意义

让前端状态由后端统一驱动，避免轮询和状态竞争。

---

## Step 18：实现前端 Support Store

### 使用技术

```text
SolidJS createSignal
SolidJS createStore
SolidJS createMemo
EventSource
TypeScript types
```

### 当前对应文件

```text
opencode/packages/app/src/context/support-store.tsx
```

### 职责

```text
bootstrap state
open SSE stream
store cases
store case details
store proposals
store sessions
store feedback
provide approve/deny/open session actions
```

### 设计原则

```text
UI 不直接猜状态。
UI 发 action POST。
状态由 backend SSE 返回。
```

---

## Step 19：实现 Support Case List UI

### 使用技术

```text
SolidJS
@solidjs/router
createMemo
Tailwind / design tokens
```

### 当前对应文件

```text
opencode/packages/app/src/pages/support-cases.tsx
```

### 功能

```text
展示 case 队列
展示 AI 状态
过滤 case
显示 pending / executed / failed
点击进入 case detail
```

### 工程意义

这是客服处理队列和发现 AI 待审任务的入口。

---

## Step 20：实现 Support Case Detail UI

### 使用技术

```text
SolidJS
router params
reactive memo
SupportStore
proposal card components
embedded session panel
```

### 当前对应文件

```text
opencode/packages/app/src/pages/support-case-detail.tsx
```

### 功能

```text
展示 case thread
展示附件
展示 Agent session
展示 proposals
approve / deny
编辑 player message
查看反馈和执行状态
```

### 工程意义

这是 human-in-the-loop 的主界面。

---

## Step 21：实现 Optool Auth Gate

### 使用技术

```text
SolidJS component
HTTP fetch
Cookie/session check
```

### 当前对应文件

```text
opencode/packages/app/src/components/optool-auth-gate.tsx
```

### 功能

```text
检查 /support/optool-status
未登录时提示登录
登录后才能审批执行
```

### 工程意义

保证执行 proposal 的是已认证 operator，而不是 Agent。

---

## Step 22：实现 Agent Session Launcher

### 使用技术

```text
Hono route
OpenCode / Codex / Claude session API
environment variable injection
workspace session
model config
```

### 当前相关文件

```text
apps/terminal-webui/src/server/index.ts
apps/terminal-webui/src/server/support-routes.ts
```

### 入口

```text
POST /support/cases/:id/analyze
POST /support/cases/:id/solve
```

### 注入环境变量

```text
CELEST_SUPPORT_AGENT_TOKEN
CELEST_SUPPORT_CASE_ID
CELEST_SUPPORT_MODE
CELEST_SUPPORT_OPTOOL_TARGET
CELEST_SUPPORT_OPTOOL_SERVER_URL
```

### 注入 Prompt 内容

```text
case JSON
mode-specific instructions
agent.md path
policy path
decision framework path
skill path
subagent roles
proposal rules
```

---

## Step 23：实现 Workspace Verification

### 使用技术

```text
Node fs/path
TypeScript validation
startup checks
```

### 当前对应文件

```text
apps/terminal-webui/src/server/support/support-workspace.ts
```

### 检查内容

```text
agent.md
resources/
skills/
subagents/
关键 scripts
knowledge root
```

### 工程意义

Agent 所需文件缺失时应 fail fast，而不是凭记忆继续处理真实 case。

---

## Step 24：实现 Evidence Subagents

### 使用技术

```text
Markdown subagent prompt
专用 skill/tool access
ClickHouse
Graylog
Splunk
Knowledge base
Jira
```

### 当前对应目录

```text
agents/celest-support/subagents/
```

### 当前子 Agent

```text
data-investigator.md
log-investigator.md
crash-investigator.md
knowledge-retriever.md
bug-reporter.md
```

### 工程意义

把复杂证据调查分给专项角色，主 Agent 负责 case 总控和最终 proposal。

---

## Step 25：实现测试体系

### 使用技术

```text
Bun scripts
TypeScript test runners
PowerShell scenario runner
batch scripts
Playwright 可选
```

### 当前对应目录

```text
agents/celest-support/tests/
```

### 测试类型

```text
skill script tests
proposal creation tests
backend validation tests
approval execution tests
negative security tests
scenario e2e tests
```

### 示例命令

```powershell
bun agents/celest-support/tests/run-all-skill-tests.ts --account-id 65625 --case-id <case_id>
```

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File agents\celest-support\tests\run-scenario-end-to-end.ps1 -Scenario player_misconduct_report -AccountId 65625
```

---

## Step 26：实现 Docker 部署

### 使用技术

```text
Docker
Docker Compose
Nginx
Bun containers
workspace image
Docker socket
volume mounts
env files
```

### 当前对应文件

```text
docker/docker-compose.yaml
```

### 主要服务

```text
frontend
celest-core
terminal-pty
opencode
celest-workspace
slack-bot-router
scheduler
agent-orchestrator
dashboard-api
artifacts
admin
```

### 常用命令

```powershell
just docker-build
just docker-up
```

### 部署所需配置

```text
MODEL API keys
Optool URL
ClickHouse credentials
Graylog credentials
Splunk credentials
Jira credentials
Docker socket
/data volume
workspace image
auth secrets
```

---

# 5. 每一层为什么这样选技术

## 5.1 为什么用 TypeScript

因为这个系统有大量结构化协议：

```text
Proposal
ProposalExecution
ProposalPresentation
CaseDetail
SupportState
AgentSession
Tool parameters
```

TypeScript 能让这些数据结构在前端、后端、Skill scripts 中保持一致。

## 5.2 为什么用 Bun

Bun 适合：

```text
快速运行 TypeScript scripts
本地开发
CLI 工具执行
启动 Hono server
monorepo 包管理
```

Celest Support 的 skill scripts 用 Bun 执行很自然。

## 5.3 为什么用 Hono

Hono 轻量，适合构建：

```text
API Gateway
大量 REST-like routes
Cookie/session
SSE stream
```

Support backend 更多是 gateway 和 workflow orchestrator，不是传统复杂业务单体。

## 5.4 为什么用 SolidJS

SolidJS 的 fine-grained reactivity 适合工作台类 UI：

```text
case list
case detail
proposal state
session state
SSE delta updates
```

状态变化细粒度，UI 响应快。

## 5.5 为什么用 SSE

因为状态更新是服务端到客户端单向推送：

```text
proposal created
proposal updated
session updated
case detail updated
```

SSE 比轮询实时，比 WebSocket 简单。

## 5.6 为什么用内存 Map + SQLite

内存 Map 适合 live approval flow：

```text
快
简单
进程内状态
实时订阅
```

SQLite side-channel 适合：

```text
历史 timeline
admin rendering
重启后部分历史可查
```

## 5.7 为什么不用 Agent 直接访问数据库

因为：

```text
权限不可控
数据泄露风险
缺少业务语义
难以审计
容易越权
```

所以用 Skill + Gateway + Allowlist 包装。

---

# 6. 关键源码路径对应关系

| 模块 | 路径 |
|---|---|
| Agent 本体 | `agents/celest-support/agent.md` |
| Policy / Playbook | `agents/celest-support/resources/` |
| Skills | `agents/celest-support/skills/` |
| Skill 公共读库 | `agents/celest-support/lib/support-read.ts` |
| Skill Proposal 库 | `agents/celest-support/lib/support-proposal.ts` |
| Backend Routes | `apps/terminal-webui/src/server/support-routes.ts` |
| Proposal Store | `apps/terminal-webui/src/server/support/proposal-store.ts` |
| Proposal Executor | `apps/terminal-webui/src/server/support/proposal-executor.ts` |
| Proposal Allowlist | `apps/terminal-webui/src/server/support/proposal-allowlist.ts` |
| Support DB | `apps/terminal-webui/src/server/support/support-db.ts` |
| State Store | `apps/terminal-webui/src/server/support/support-state-store.ts` |
| Workspace Verify | `apps/terminal-webui/src/server/support/support-workspace.ts` |
| Frontend Store | `opencode/packages/app/src/context/support-store.tsx` |
| Case List UI | `opencode/packages/app/src/pages/support-cases.tsx` |
| Case Detail UI | `opencode/packages/app/src/pages/support-case-detail.tsx` |
| Auth Gate | `opencode/packages/app/src/components/optool-auth-gate.tsx` |
| Agent config | `opencode.json` |
| Plugin manifest | `agents/celest-support/.codex-plugin/plugin.json` |
| Docker Compose | `docker/docker-compose.yaml` |
| Tests | `agents/celest-support/tests/` |

---

# 7. 通用迁移模板

如果要迁移到其他垂直领域，工程结构可以保持不变。

## 7.1 通用工程结构

```text
agents/<domain-agent>/
  agent.md
  resources/
    POLICIES.md
    DECISION_FRAMEWORK.md
    CASE_PLAYBOOK.md
  skills/
    cases/
    entity-information/
    entity-actions/
    account-operations/
  subagents/
  lib/
  tests/

apps/<domain-workbench>/src/server/
  routes.ts
  proposal-store.ts
  proposal-executor.ts
  proposal-allowlist.ts
  state-store.ts
  workspace.ts

apps/<domain-workbench>/src/frontend/
  store.tsx
  case-list.tsx
  case-detail.tsx
  proposal-card.tsx
```

## 7.2 技术替代关系

| Celest 技术 | 可替代技术 |
|---|---|
| Bun | Node.js / Deno |
| Hono | Express / Fastify / NestJS / FastAPI |
| SolidJS | React / Vue / Svelte |
| SSE | WebSocket / polling |
| SQLite | Postgres / MySQL |
| OpenCode Runtime | LangGraph / AutoGen / CrewAI / 自研 Runtime |
| Optool | 任意内部 Admin Backend |

## 7.3 不建议替代的架构原则

```text
Read / Write 分离
Backend Gateway
Scoped Agent Token
Command Allowlist
Script Contract
Proposal Human Review
Audit Trail
Scenario Tests
```

---

# 8. 专业面试 Q&A

## Q1：这个 Agent 系统的工程框架是什么？

**答：**

它是一个基于 TypeScript 的多模块工程系统，而不是单一 Agent 脚本。

核心工程框架包括：

```text
Bun workspace 管理依赖和脚本；
Hono 构建 /support/* Backend Gateway；
SolidJS + Vite 构建客服工作台；
OpenCode/Codex/Claude Runtime 启动 Agent 会话；
TypeScript CLI scripts 封装 Agent tools；
Proposal Store + Executor 实现人审执行；
SSE 实现状态同步；
Docker Compose 实现多服务部署。
```

它的核心思想是：

```text
Agent Runtime 只负责推理；
Skill Scripts 负责受控工具调用；
Backend Gateway 负责权限和系统集成；
Proposal System 负责人审和安全执行；
Frontend Workbench 负责人机协作；
Audit/Test 负责可追踪和可靠性。
```

---

## Q2：为什么说这不是 LangChain Demo？

**答：**

因为这个系统的核心不是链式调用模型，而是工程化业务闭环。

LangChain / LangGraph 这类框架可以解决：

```text
Agent 编排
工具调用
节点流转
```

但 Celest Support 还需要解决：

```text
客服登录态；
内部后台代理；
case/account scope；
只读白名单；
proposal 人审；
后端执行；
前端审批；
SSE 状态；
审计；
测试；
Docker 部署。
```

所以即使用 LangGraph，也只是替换 Agent Runtime 编排层，不能替代整套工程架构。

---

## Q3：为什么后端用 Hono？

**答：**

Hono 轻量、路由清晰，适合做 API Gateway。

Celest Support backend 的职责不是复杂业务计算，而是：

```text
提供 /support/* routes；
做认证和 session；
代理 Optool；
校验 proposal；
推送 SSE；
启动 Agent session。
```

Hono 很适合这种：

```text
轻量 HTTP router + middleware + streaming
```

的场景。

---

## Q4：为什么前端用 SolidJS？

**答：**

Support UI 是一个实时工作台，需要频繁响应后端状态变化：

```text
case 更新；
proposal 创建；
proposal 执行；
Agent session 状态变化；
queue 更新。
```

SolidJS 的细粒度响应式适合这种场景。  
配合 `EventSource` 和 `createStore`，前端可以很轻量地维护实时状态。

核心不是 SolidJS 本身，而是：

```text
前端用统一 SupportStore 订阅后端 SSE；
页面只渲染状态；
状态真源在后端。
```

---

## Q5：为什么用 TypeScript 写 Skill scripts？

**答：**

因为 Skill scripts 需要和后端共享结构化协议，例如：

```text
Proposal
ProposalExecution
ProposalOptoolStep
ProposalPresentation
```

TypeScript 可以保证：

```text
参数结构清晰；
proposal schema 一致；
脚本更容易测试；
公共 helper 可复用；
前后端类型更容易对齐。
```

同时 Bun 可以直接运行 TypeScript 脚本，适合 CLI-style tools。

---

## Q6：Backend Gateway 的核心作用是什么？

**答：**

Backend Gateway 是 Agent 和内部生产系统之间的安全隔离层。

它负责：

```text
认证；
授权；
session；
agent token；
只读白名单；
proposal 校验；
script contract；
scope validation；
Optool 代理；
SSE；
审计。
```

如果没有 Gateway，Agent 就会直接接触内部系统和高权限 API。  
这在客服、账号、经济、支付等高风险场景不可接受。

所以我会说：

```text
Gateway 是生产级 Agent 的硬安全边界。
```

---

## Q7：为什么要做 Agent Token，而不是直接复用客服登录态？

**答：**

因为 Agent 和客服是不同安全主体。

客服登录态代表人类操作员，可以审批和执行。  
Agent token 只代表一个当前 case 的自动化调查会话。

Agent token 应该限制：

```text
只能访问当前 case；
只能创建当前 case proposal；
不能直接执行 mutation；
有过期时间；
有 related account 限制。
```

如果让 Agent 复用客服登录态，就会让 Agent 拥有人类的完整执行权限，这是危险的。

---

## Q8：Proposal Store 为什么用内存 Map + SQLite？

**答：**

这是为了区分 live state 和 historical state。

内存 Map 适合：

```text
实时 proposal 审批；
快速状态更新；
SSE 推送；
进程内协作。
```

SQLite side-channel 适合：

```text
历史 timeline；
admin view；
审计查看；
重启后历史可查。
```

这不是最终唯一方案。大型生产系统可以换成 Postgres，但当前设计体现了：

```text
live approval flow 和 durable audit 的职责分离。
```

---

## Q9：为什么用 SSE 而不是轮询？

**答：**

因为 support UI 有很多实时事件：

```text
proposal.created
proposal.updated
case.detail
session.updated
feedback.updated
```

轮询会造成：

```text
延迟；
重复请求；
状态竞争；
数据库压力；
前端状态不一致。
```

SSE 更适合服务端单向推送，足够轻量，不需要 WebSocket 的复杂双向连接。

---

## Q10：Proposal Executor 为什么不 switch operation？

**答：**

因为 switch operation 会让执行器变成业务分支堆叠：

```ts
switch(operation) {
  case "move_player":
  case "close_case":
  case "change_email":
}
```

扩展性差，也容易让展示标签和真实执行不一致。

Celest 使用：

```text
data-driven execution
```

即：

```text
proposal.execution.steps 定义真正执行计划；
executor 只负责校验和执行 steps。
```

这样新增动作主要改：

```text
skill script
allowlist
script contract
test
```

而不是大改 executor。

---

## Q11：Script Contract 解决什么问题？

**答：**

它解决“谁有资格创建什么 proposal”的问题。

只有 allowlist 不够。  
因为 command 在 allowlist 中，不代表任何脚本都能创建它。

Script Contract 校验：

```text
created_by_skill
created_by_script
operation
execution command
```

确保：

```text
create-reply-to-support-case-proposal.ts
只能创建 reply proposal；
不能伪造 account mutation proposal。
```

这是防止工具供应链越权的重要机制。

---

## Q12：为什么 Read Allowlist 也很重要？

**答：**

只读也可能泄露隐私和内部信息。

如果 Agent 能任意查询：

```text
账号数据；
支付数据；
安全设备；
其他玩家信息；
风控字段；
日志细节；
```

就可能产生数据泄露。

所以 read command 和 view template 必须白名单化。  
Agent 只能查当前 case 所需的最小信息。

---

## Q13：前端工作台为什么是工程架构的一部分？

**答：**

因为这个系统采用 human-in-the-loop。

人类审批不是一句概念，它需要 UI 支撑：

```text
展示 proposal；
展示原因；
展示参数；
展示玩家回复；
允许编辑；
approve / deny；
显示执行状态。
```

没有前端工作台，Proposal 机制无法真正落地。

---

## Q14：Workspace Verification 为什么重要？

**答：**

Agent 运行依赖很多文件：

```text
agent.md
resources
skills
subagents
proposal helpers
knowledge root
```

如果缺文件，Agent 可能凭记忆瞎跑，风险很高。

Workspace Verification 在启动前检查这些依赖，缺失就报错。  
这是 fail-fast 思路：

```text
生产级 Agent 宁可启动失败，也不要在错误环境中处理真实 case。
```

---

## Q15：如果迁移到其他领域，工程框架怎么改？

**答：**

保留工程架构，替换业务模块。

保留：

```text
Backend Gateway
Agent Token
Skill Script 模式
Proposal schema
Proposal Store
Proposal Executor
Allowlist
Script Contract
SSE State
Frontend Workbench
Audit
Tests
```

替换：

```text
业务对象
Policy
Playbook
Skills
Internal backend adapter
Proposal operations
UI labels
Scenario tests
```

例如金融领域：

```text
SupportCase -> RiskAlert
Player -> Customer
Optool -> Risk Ops Backend
Proposal -> Risk Action Proposal
```

---

## Q16：这个工程框架和普通 CRUD 后台有什么区别？

**答：**

普通 CRUD 后台是人直接操作数据。

这个系统多了一层 Agent workflow：

```text
Agent 读取业务上下文；
Agent 调用工具调查；
Agent 生成结构化 proposal；
人类审批；
后端执行。
```

它不是简单 CRUD，而是：

```text
AI-assisted decision preparation + human-approved execution
```

所以它需要额外的：

```text
Agent token
tool contract
proposal store
execution plan
SSE session state
prompt/policy/playbook
negative tests
```

---

## Q17：这个工程里最关键的数据结构是什么？

**答：**

最关键的是 Proposal。

因为它连接：

```text
Agent 建议
前端展示
人工审批
后端执行
审计记录
```

Proposal 中最重要的是：

```text
reason
presentation
execution.steps
status
approved_by
result
```

如果没有 Proposal，AI 输出就只是聊天内容；  
有了 Proposal，AI 输出变成可执行、可审批、可审计的业务对象。

---

## Q18：这个工程里最关键的安全机制是什么？

**答：**

不是单一机制，而是组合：

```text
Agent token scope
Read allowlist
Proposal allowlist
Script contract
Case/account scope validation
Operator session approval
Backend execution
Audit trail
```

其中最核心的是：

```text
Agent 不直接执行 mutation。
```

所有写动作必须通过 proposal。

---

## Q19：如何判断这个 Agent 系统是生产级，而不是 Demo？

**答：**

看它是否具备：

```text
1. 权限边界；
2. 工具封装；
3. 人审流程；
4. 审计记录；
5. 状态同步；
6. 错误处理；
7. 负面测试；
8. 部署方案；
9. 业务 playbook；
10. 可迁移架构。
```

Celest Support 具备这些模块，所以它是工程化系统，而不是 prompt demo。

---

## Q20：面试时如何一句话说明工程框架？

**答：**

```text
这个系统是用 TypeScript + Bun + Hono 搭 Backend Gateway，用 SolidJS + Vite 搭客服工作台，用 OpenCode/Codex Runtime 启动 Agent 会话，用 TypeScript Skill Scripts 封装工具，通过 Proposal Store/Executor、Allowlist、Script Contract 和 SSE 状态同步实现人审、安全执行和审计闭环的垂直领域 Agent 工程框架。
```

---

# 9. 最终总结话术

## 9.1 30 秒版本

```text
工程上，这套 Agent 系统不是单独接一个 LLM，而是一个完整的 TypeScript/Bun 工程。后端用 Hono 搭 `/support/*` Gateway，负责 Optool 登录、Agent token、只读白名单、Proposal 创建和审批执行；前端用 SolidJS + Vite 搭客服工作台，通过 SSE 实时同步 case、session 和 proposal 状态；Agent Runtime 用 OpenCode/Codex/Claude session，工具层用 TypeScript Skill Scripts 封装业务能力。所有写操作不直接执行，而是通过 Proposal Store、Allowlist、Script Contract 和 Proposal Executor 进入人工审批和后端安全执行链路。
```

## 9.2 2 分钟版本

```text
这个框架从工程上分成几层。第一层是 monorepo 和运行环境，用 Bun workspace、TypeScript、justfile 和 Docker Compose 管理。第二层是 Agent 配置，通过 opencode.json 和 .codex-plugin/plugin.json 注册 Celest Support skills。第三层是 Agent 本体和业务知识，用 agent.md、RULES-POLICIES.md、DECISION_FRAMEWORK.md、CASE_PLAYBOOK.md 定义 Agent 的行为和业务 SOP。第四层是工具层，用 TypeScript CLI scripts 封装 support case、player information、player actions、account operations 等能力，并通过 support-read 和 support-proposal 公共库统一调用后端。

第五层是后端 Gateway，用 Hono + Bun 实现 /support/* API，负责 Optool 登录态、Agent token、只读白名单、proposal 校验、approve/deny、SSE 状态流和 Agent session 启动。第六层是 Proposal 系统，用 TypeScript interface 定义 proposal wire contract，用内存 Map 做 live store，用 SQLite 做历史 side-channel，用 allowlist 和 script contract 做安全校验，用 generic executor 根据 execution.steps 执行后端命令。第七层是前端工作台，用 SolidJS + Vite + EventSource 展示工单、AI 会话、proposal card，并让客服 approve/deny。第八层是证据和部署，包括 ClickHouse、Graylog、Splunk、Jira 以及 Docker Compose。

所以它不是一个 prompt demo，而是一个完整的 Agent 工程系统：LLM 负责推理，Skill 负责工具调用，Gateway 负责权限，Proposal 负责人审，Executor 负责安全执行，前端负责协作，审计和测试负责可靠性。
```

## 9.3 一句话核心理解

```text
这个工程框架的本质，是用常规 Web 工程能力把 LLM 包进一个可控业务执行系统：后端控权限，工具控能力，Proposal 控写操作，前端控人审，审计控追踪。
```

