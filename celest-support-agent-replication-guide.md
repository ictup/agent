# Celest Support 智能客服 Agent 复刻操作文档

版本：2026-05-12  
适用对象：想要完整复刻 Celest Support 内部智能客服系统的工程师 / Agent 开发者 / 平台架构师  
目标：基于当前 Celest 仓库，理解并复刻一个功能等价的内部客服 Agent 系统

---

## 目录

1. [文档目标](#1-文档目标)
2. [结论先行](#2-结论先行)
3. [Celest Support 到底是什么](#3-celest-support-到底是什么)
4. [它不是什么](#4-它不是什么)
5. [总体架构](#5-总体架构)
6. [核心设计原则](#6-核心设计原则)
7. [目录与源码地图](#7-目录与源码地图)
8. [Agent 本体结构](#8-agent-本体结构)
9. [Policy / Playbook / Resource 体系](#9-policy--playbook--resource-体系)
10. [Skill 工具体系](#10-skill-工具体系)
11. [Support Case 能力](#11-support-case-能力)
12. [Player Information 能力](#12-player-information-能力)
13. [Player Actions 能力](#13-player-actions-能力)
14. [Player Account 能力](#14-player-account-能力)
15. [Jira Bug Report 能力](#15-jira-bug-report-能力)
16. [Proposal 审批系统](#16-proposal-审批系统)
17. [后端 `/support/*` API 系统](#17-后端-support-api-系统)
18. [前端 Support UI 系统](#18-前端-support-ui-系统)
19. [Agent Session 启动流程](#19-agent-session-启动流程)
20. [Optool 集成方式](#20-optool-集成方式)
21. [安全与权限模型](#21-安全与权限模型)
22. [子 Agent / Evidence 系统](#22-子-agent--evidence-系统)
23. [本地运行方式](#23-本地运行方式)
24. [Docker 部署方式](#24-docker-部署方式)
25. [复刻实施步骤](#25-复刻实施步骤)
26. [新增一个客服动作的完整流程](#26-新增一个客服动作的完整流程)
27. [测试与验收](#27-测试与验收)
28. [常见坑点](#28-常见坑点)
29. [最小可用版本 MVP](#29-最小可用版本-mvp)
30. [完整复刻验收清单](#30-完整复刻验收清单)
31. [总结](#31-总结)

---

# 1. 文档目标

本文档的目标不是简单解释“Celest 是什么”，而是帮助你**从工程实现角度完整复刻 Celest Support 智能客服系统**。

读完后，你应该能够：

- 理解 Celest Support 的整体架构。
- 知道它由哪些模块组成。
- 知道每个模块在仓库中的位置。
- 知道 agent 是如何被启动、如何拿到 case、如何调用工具。
- 知道为什么所有高风险操作都必须走 Proposal。
- 知道 Proposal 是如何创建、展示、审批和执行的。
- 知道前端、后端、Optool、Agent、Skill 之间如何协作。
- 知道复刻时最容易踩哪些坑。
- 知道如何一步步搭建一个功能等价的系统。

---

# 2. 结论先行

Celest Support 不是一个普通聊天机器人。

它本质上是一个：

```text
内部客服 AI 调查工作台
```

它由以下部分组成：

```text
1. 客服前端 UI
2. support backend
3. Optool 登录与代理层
4. agent session launcher
5. Celest Support agent prompt
6. support skills
7. player/account investigation scripts
8. proposal creation scripts
9. proposal approval system
10. proposal executor
11. state store / SSE 推送
12. evidence subagents
13. policy / playbook / knowledge base
14. audit / history / testing system
```

最重要的安全原则：

```text
Agent 只能调查、分析、建议、创建 proposal。
Agent 不能直接修改玩家、账号、经济、物品或工单状态。
所有变更必须由人工客服审批。
后端使用人工客服的 Optool 登录态执行。
```

一句话总结：

```text
Celest Support = AI 调查员 + 人工审批 + 安全执行器 + 客服工作台
```

---

# 3. Celest Support 到底是什么

Celest Support 是给内部客服人员使用的智能客服系统。

它负责辅助客服处理 Entropia Universe 的 support cases。

它可以：

- 读取工单详情。
- 读取玩家历史工单。
- 下载并检查附件。
- 查询玩家账号资料。
- 查询玩家当前状态和位置。
- 查询玩家库存、仓库、物品、技能、任务、经济历史。
- 查询支付、银行卡、账户安全、登录、设备等信息。
- 查询 crash report。
- 查询 item history。
- 判断玩家诉求是否合理。
- 根据政策和 playbook 给出建议。
- 创建客服回复草案。
- 创建内部备注。
- 创建关闭、重开、升级、认领等工单操作提案。
- 创建玩家传送、强制登出、补偿、物品恢复等操作提案。
- 在人工确认后执行动作。
- 在明确要求下创建 Jira bug report。

它的定位是：

```text
AI internal investigator
```

不是：

```text
player-facing chatbot
```

---

# 4. 它不是什么

复刻时必须先避免几个误解。

## 4.1 它不是单 Prompt Agent

错误理解：

```text
写一个 system prompt，接几个 API，就叫 Celest Support。
```

真实情况：

```text
Celest Support 是一套完整系统。
Prompt 只是其中一部分。
```

## 4.2 它不是自动客服机器人

当前系统主要服务于内部客服人员。

玩家不是直接面对这个 agent。

玩家看到的是客服最终发送的回复，而不是 AI 的完整调查过程。

## 4.3 它不是无限权限 Agent

Celest Support 不允许 agent 直接执行危险操作。

例如：

```text
转移玩家
发钱
改邮箱
重置密码
删除安全设备
关闭工单
补偿物品
释放 IP lock
```

这些都不能由 agent 直接执行。

必须走：

```text
proposal -> human approve -> backend execute
```

## 4.4 它不是任意 SQL 查询器

Agent 不能随便查数据库。

它只能通过白名单脚本、白名单 Optool command、白名单 view template 查询。

这是安全边界之一。

---

# 5. 总体架构

## 5.1 高层架构图

```text
┌──────────────────────────────┐
│        客服浏览器 UI          │
│ support case list/detail      │
│ proposal cards                │
│ agent session iframe          │
└───────────────┬──────────────┘
                │
                v
┌──────────────────────────────┐
│ terminal-webui backend        │
│ /support/* routes             │
│ optool session                │
│ proposal store                │
│ state store                   │
│ agent launcher                │
└───────┬──────────┬───────────┘
        │          │
        │          v
        │  ┌─────────────────────┐
        │  │ Agent Session        │
        │  │ agents/celest-support│
        │  │ skills               │
        │  │ subagents            │
        │  └─────────────────────┘
        │
        v
┌──────────────────────────────┐
│ Optool / internal backend     │
│ support cases                 │
│ player/account operations     │
│ HTTP/XML commands             │
└──────────────────────────────┘
```

## 5.2 核心数据流

```text
客服打开 support case
        |
        v
前端请求 /support/cases/:id
        |
        v
后端从 Optool 拉取 case
        |
        v
客服点击 Analyze / Solve
        |
        v
后端创建 agent token
        |
        v
后端启动 agent session
        |
        v
Agent 读取 case JSON
        |
        v
Agent 调用 skill scripts 调查
        |
        v
Agent 创建 proposal
        |
        v
前端显示 proposal card
        |
        v
客服 approve / deny
        |
        v
后端二次校验
        |
        v
后端调用 Optool 执行动作
        |
        v
刷新 case / proposal / audit 状态
```

---

# 6. 核心设计原则

## 6.1 AI 负责调查，人类负责批准

Celest Support 的关键不是“让 AI 自动客服”，而是：

```text
让 AI 快速调查复杂 case，并把可执行方案交给人类批准。
```

这可以降低风险。

## 6.2 所有 mutation 都必须 proposal-gated

Mutation 指任何会改变系统状态的动作，例如：

```text
工单回复
工单关闭
工单重开
添加内部备注
玩家传送
强制登出
发放 PED
发放 Universal Ammo
恢复物品
修改账号信息
修改邮箱
移除支付卡
释放 IP lock
修改安全设备
```

这些必须走 Proposal。

## 6.3 Agent 不持有原始 Optool 凭证

Agent 只能拿到 scoped token。

它不能直接拿客服的用户名密码。

当前系统中：

```text
GET /support/optool-credentials
```

是 tombstone，返回 410。

这意味着系统明确废弃了“把 Optool 凭证暴露给 agent”的模式。

## 6.4 只读查询也要白名单

Agent 不能任意查询内部数据。

查询必须走：

```text
support skill script
optool-read whitelist
player-view templates
account-report routes
```

## 6.5 玩家回复必须安全

玩家回复不能包含：

```text
内部字段
日志细节
检测规则
风控原因
数据库输出
内部工具名
经济系统实现细节
未经确认的猜测
```

玩家回复必须以：

```text
Kind regards,
Entropia Universe Support
```

结尾。

---

# 7. 目录与源码地图

## 7.1 当前真正的 agent 根目录

```text
agents/celest-support/
```

重要内容：

```text
agents/celest-support/agent.md
agents/celest-support/resources/
agents/celest-support/skills/
agents/celest-support/subagents/
agents/celest-support/lib/
agents/celest-support/tests/
agents/celest-support/.codex-plugin/plugin.json
```

## 7.2 当前真正的 support backend

```text
apps/terminal-webui/src/server/support-routes.ts
apps/terminal-webui/src/server/support/
```

重要文件：

```text
apps/terminal-webui/src/server/support/proposal-store.ts
apps/terminal-webui/src/server/support/proposal-executor.ts
apps/terminal-webui/src/server/support/proposal-allowlist.ts
apps/terminal-webui/src/server/support/support-db.ts
apps/terminal-webui/src/server/support/support-state-store.ts
apps/terminal-webui/src/server/support/support-workspace.ts
```

## 7.3 当前前端入口

```text
opencode/packages/app/src/context/support-store.tsx
opencode/packages/app/src/pages/support-cases.tsx
opencode/packages/app/src/pages/support-case-detail.tsx
opencode/packages/app/src/pages/support-shared.ts
opencode/packages/app/src/pages/support-categories.ts
opencode/packages/app/src/components/optool-auth-gate.tsx
```

## 7.4 Agent session launcher

```text
apps/terminal-webui/src/server/index.ts
```

## 7.5 Proposal 展示逻辑

```text
opencode/packages/app/src/lib/support-proposal-presentation.ts
```

## 7.6 旧路径提醒

不要把下面这些当成当前运行系统的主入口：

```text
plugins/celest-support/
opencode/packages/opencode/src/server/routes/support.ts
plugin-sync sidecar
```

这些在文档或历史代码中可能出现，但当前主系统已经迁移。

---

# 8. Agent 本体结构

## 8.1 agent.md

核心文件：

```text
agents/celest-support/agent.md
```

它定义了 Celest Support agent 的身份和行为。

核心身份：

```text
Celest Support - Internal Investigator
```

它是：

```text
内部调查员
客服辅助者
证据收集者
proposal 创建者
```

它不是：

```text
最终执行者
无限权限操作者
玩家端聊天机器人
```

## 8.2 agent 启动后必须做什么

Agent 每次处理 case 时应该：

```text
1. 阅读 case 内容。
2. 阅读所有 case posts。
3. 下载并检查附件。
4. 找出玩家的核心诉求。
5. 把诉求拆成可验证 claim。
6. 根据 case 类型查对应 playbook。
7. 查询玩家账号与状态。
8. 查询相关历史数据。
9. 必要时调用 evidence subagents。
10. 形成判断。
11. 如果要执行动作，创建 proposal。
12. 如果要回复玩家，创建 reply proposal。
13. 如果只是分析，给出内部总结。
```

## 8.3 Agent 的硬性规则

必须遵守：

```text
- 不直接执行变更。
- 不跳过附件。
- 不编造数据。
- 不向玩家暴露内部证据。
- 不用未经验证的信息做结论。
- 不绕开 proposal。
- 不越权操作 unrelated account。
- 对老 case / 历史事实优先使用历史数据源。
```

---

# 9. Policy / Playbook / Resource 体系

资源目录：

```text
agents/celest-support/resources/
```

核心资源：

```text
RULES-POLICIES.md
DECISION_FRAMEWORK.md
CASE_PLAYBOOK.md
case_playbook_library/
```

## 9.1 RULES-POLICIES.md

定义客服政策。

例如：

```text
哪些情况可以补偿
哪些情况不能补偿
哪些情况需要升级
哪些情况需要内部备注
哪些信息不能告诉玩家
哪些操作必须谨慎
```

## 9.2 DECISION_FRAMEWORK.md

定义 agent 如何做判断。

典型判断流程：

```text
1. 玩家声称发生了什么？
2. 这个 claim 是否可验证？
3. 需要哪些数据源验证？
4. 数据是否支持玩家说法？
5. 是否符合政策？
6. 是否需要 action？
7. 是否需要 escalation？
8. 是否需要 player reply？
```

## 9.3 CASE_PLAYBOOK.md

定义不同 case 类型的处理手册。

例如：

```text
登录问题
卡住问题
物品丢失
交易争议
付款问题
崩溃问题
账号安全
行为举报
任务问题
采矿 claim 问题
```

## 9.4 case_playbook_library

更细的专项 playbook。

复刻时要注意：

```text
Agent 不是靠通用推理盲猜。
它应先查 playbook，再决定调查路径。
```

---

# 10. Skill 工具体系

Skill 是 Celest Support 的工具层。

路径：

```text
agents/celest-support/skills/
```

当前主要 skill：

```text
support-cases
player-information
player-actions
player-account
jira-bug-report
self-improvement
```

## 10.1 Skill 的角色

Skill 负责把复杂后端调用封装成 agent 可使用的脚本。

Agent 不应该直接拼 API 请求。

正确方式：

```text
Agent -> skill script -> backend /support route -> Optool/internal system
```

## 10.2 Skill 的结构

每个 skill 通常包括：

```text
SKILL.md
scripts/
```

其中：

```text
SKILL.md
```

告诉 agent：

```text
什么时候用这个 skill
有哪些脚本
脚本参数是什么
脚本输出是什么
注意事项是什么
```

---

# 11. Support Case 能力

路径：

```text
agents/celest-support/skills/support-cases/
```

## 11.1 只读能力

典型脚本：

```text
get-support-case-details-and-posts.ts
download-support-case-attachments.ts
list-player-support-case-history.ts
search-support-cases.ts
view-support-queue-status.ts
list-standard-reply-templates.ts
get-standard-reply-template.ts
```

这些用于：

```text
读取当前 case
读取帖子
读取附件
查询玩家历史 case
查询队列状态
查看标准回复模板
```

## 11.2 Proposal 能力

典型脚本：

```text
create-claim-support-case-proposal.ts
create-reply-to-support-case-proposal.ts
create-add-support-case-internal-comment-proposal.ts
create-reopen-support-case-proposal.ts
create-escalate-support-case-proposal.ts
create-modify-support-case-proposal.ts
create-close-support-case-proposal.ts
create-bulk-close-inactive-cases-proposal.ts
```

这些用于创建：

```text
认领工单 proposal
回复玩家 proposal
添加内部备注 proposal
重开工单 proposal
升级工单 proposal
修改工单 proposal
关闭工单 proposal
批量关闭 inactive case proposal
```

## 11.3 复刻要求

必须实现：

```text
1. case detail read
2. posts read
3. attachments download
4. case history
5. proposal creation for reply/comment/close/reopen/escalate
6. frontend card display
7. approval execution
```

---

# 12. Player Information 能力

路径：

```text
agents/celest-support/skills/player-information/
```

这个 skill 主要是只读调查。

## 12.1 核心约定

非常重要：

```text
account_id == player_id
```

复刻时不要额外发明一层 player id/account id 映射，除非底层系统真的需要。

## 12.2 常用调查能力

典型脚本：

```text
find-player-account-by-name-or-email.ts
register-case-related-player-account.ts
get-player-full-investigation-report.ts
get-player-account-profile-and-status.ts
get-player-current-state-and-location.ts
view-player-inventory-and-storage.ts
list-player-claimed-mining-resources.ts
list-player-claimed-mining-resource-history.ts
view-player-estates-and-properties.ts
list-player-auction-and-marketplace-history.ts
view-player-economic-balances.ts
view-player-recent-economic-history.ts
list-player-skills-and-professions.ts
list-player-recent-positions.ts
get-player-referral-and-invite-info.ts
get-player-activity-counters.ts
list-player-quest-progress.ts
find-item-by-name-or-cid.ts
get-item-instance-details.ts
trace-item-history-by-id.ts
dump-player-persistent-state-blob.ts
dump-player-character-definition-blob.ts
```

## 12.3 用途举例

### 玩家说“我卡住了”

Agent 应查：

```text
get-player-current-state-and-location.ts
list-player-recent-positions.ts
```

然后可能创建：

```text
create-teleport-player-to-safe-zone-proposal.ts
```

### 玩家说“我的物品丢了”

Agent 应查：

```text
view-player-inventory-and-storage.ts
trace-item-history-by-id.ts
view-player-recent-economic-history.ts
list-player-auction-and-marketplace-history.ts
```

然后决定：

```text
无证据 -> 回复说明
有证据 -> 创建恢复/补偿 proposal
复杂 -> escalate/internal comment
```

### 玩家说“采矿 claim 没给资源”

Agent 应查：

```text
list-player-claimed-mining-resources.ts
list-player-claimed-mining-resource-history.ts
view-player-recent-economic-history.ts
```

然后可能创建：

```text
create-credit-mining-claim-resource-proposal.ts
```

---

# 13. Player Actions 能力

路径：

```text
agents/celest-support/skills/player-actions/
```

这个 skill 主要创建玩家行为操作 proposal。

## 13.1 典型动作

```text
teleport player to safe zone
force logout
recall vehicle
remove active effect
reset settings
reset tutorial
reset character definition
reset quest
change avatar first/last name
compensate PED
compensate Universal Ammo
credit mining claim resource
collect restoration fee
move items
recreate items from transaction
```

## 13.2 安全要求

这些动作都不能直接执行。

必须：

```text
Agent creates proposal
Human approves
Backend executes
```

## 13.3 特殊限制

玩家移动类 proposal 不能带玩家消息。

错误：

```text
一个 proposal 同时 Player.Place + 给玩家回复
```

正确：

```text
proposal 1: teleport player
proposal 2: reply to support case
```

---

# 14. Player Account 能力

路径：

```text
agents/celest-support/skills/player-account/
```

这是账号层能力，风险更高。

## 14.1 只读能力

包括：

```text
account profile
operator comments
payment status
payment limits
payment cards
payment card history
Gold Card
Yubikey
LinOTP
banking transactions
pending transactions
crash reports
address history
orders
webshop purchases
offer packages
lifecycle events
activations
IP range locks
```

## 14.2 Proposal 能力

包括：

```text
add operator comment
change email
change account info
change birth date
change password
change flags
change account type / lock
set verified
set deposit limit
remove payment card
release IP lock
Gold Card add/remove/toggle/resync
Yubikey add/remove/toggle
LinOTP add/remove/toggle
accept/reject/commit pending transaction
send email
terminate account preview/commit
move all items
```

## 14.3 复刻注意

账号能力必须特别小心：

```text
1. 必须 proposal gated。
2. 必须校验 account scope。
3. related account 操作必须受限。
4. 审批时必须再次确认当前 case 与 account 仍匹配。
5. 玩家回复不能透露安全细节。
```

---

# 15. Jira Bug Report 能力

路径：

```text
agents/celest-support/skills/jira-bug-report/
```

Jira bug report 是特殊能力。

规则：

```text
只有当 operator 明确要求创建 Jira bug report 时，agent 才能提交。
```

不要让 agent 在普通 case 中自动创建 Jira。

普通客服动作仍然必须 proposal-gated。

---

# 16. Proposal 审批系统

Proposal 是整个系统的安全核心。

## 16.1 为什么需要 Proposal

因为客服系统涉及真实玩家、真实经济、真实账号安全。

如果让 AI 直接执行动作，风险过高。

Proposal 模型把流程改成：

```text
AI 建议
人类审批
后端执行
系统审计
```

## 16.2 Proposal 创建流程

```text
Agent 调用 skill script
        |
        v
script 构造 proposal JSON
        |
        v
POST /support/proposals
        |
        v
backend 校验
        |
        v
ProposalStore 保存
        |
        v
SupportStateStore 推送
        |
        v
前端显示 proposal card
```

## 16.3 Proposal 审批流程

```text
客服点击 Approve
        |
        v
POST /support/proposals/:id/approve
        |
        v
backend 校验 proposal still pending
        |
        v
校验 Optool 登录态
        |
        v
校验 case/account/planet
        |
        v
校验 execution allowlist
        |
        v
proposal-executor 执行
        |
        v
写 audit
        |
        v
刷新 case state
```

## 16.4 Proposal 拒绝流程

```text
客服点击 Deny
        |
        v
POST /support/proposals/:id/deny
        |
        v
proposal 标记 denied
        |
        v
state 推送给前端
```

## 16.5 Proposal 数据模型

核心定义在：

```text
agents/celest-support/lib/support-proposal.ts
```

概念上类似：

```ts
type SupportProposalInput = {
  case_id: number
  account_id: number
  operation: string
  planet_id: number
  params: Record<string, unknown>
  reason: string
  player_message?: string
  presentation?: ProposalPresentation
  execution: ProposalExecution
}
```

执行部分：

```ts
type ProposalExecution = {
  steps: ProposalOptoolStep[]
  audit_summary?: string
  evicts_case?: boolean
}
```

每一步：

```ts
type ProposalOptoolStep = {
  command: string
  parameters: Record<string, unknown>
  failure_message?: string
  if_player_message?: boolean
  substitute_player_message_as?: string
}
```

## 16.6 最关键的实现点

当前系统不是这样执行：

```ts
switch (proposal.operation) {
  case "move_player":
    ...
}
```

而是这样：

```text
proposal-executor 读取 proposal.execution.steps
逐步执行 command
```

也就是说：

```text
operation 是展示、审计和 contract 校验用的。
真正执行的是 execution.steps。
```

## 16.7 Proposal Allowlist

允许执行哪些 command，由这里控制：

```text
apps/terminal-webui/src/server/support/proposal-allowlist.ts
```

如果 command 不在 allowlist，后端拒绝执行。

## 16.8 Proposal Script Contract

后端还维护一张 script contract map。

它用于校验：

```text
某个 proposal script
是否只创建它被允许创建的 operation
是否只使用它被允许使用的 command
```

这样可以防止一个低风险脚本伪造高风险 proposal。

---

# 17. 后端 `/support/*` API 系统

核心文件：

```text
apps/terminal-webui/src/server/support-routes.ts
```

## 17.1 后端职责

这个文件承担大量职责：

```text
1. Optool 登录状态
2. Optool target 切换
3. support case list/detail
4. support case attachments
5. support queue
6. case history
7. player/account read proxy
8. item history
9. proposal creation
10. proposal approve/deny
11. support state stream
12. agent token
13. agent session open
14. Jira bug report
```

## 17.2 重要路由

### 状态和登录

```text
GET  /support/status
GET  /support/optool-status
POST /support/optool-target
POST /support/optool-login
POST /support/optool-logout
```

### Agent token

```text
POST /support/agent-token
```

### State / Stream

```text
GET /support/state
GET /support/stream
```

### Case

```text
GET  /support/cases
GET  /support/cases/:id
GET  /support/cases/:id/attachments
GET  /support/cases/:id/attachments/download
GET  /support/queue
GET  /support/account/:id/cases
GET  /support/players/:id/support-cases
```

### Case mutation

这些通常由 proposal 或 gated UI 操作触发：

```text
POST /support/cases/:id/checkout
POST /support/cases/:id/reply
POST /support/cases/:id/comment
POST /support/cases/:id/state
POST /support/cases/:id/release
POST /support/cases/:id/close
POST /support/cases/:id/reopen
```

### Agent session

```text
POST /support/cases/:id/analyze
POST /support/cases/:id/solve
```

### Proposal

```text
POST /support/proposals
POST /support/proposals/:id/approve
POST /support/proposals/:id/deny
```

### Read proxies

```text
POST /support/optool-read
POST /support/optool-search-view
POST /support/player-view
GET  /support/player-view/:view
GET  /support/account-report/:account_id
GET  /support/account-report/:account_id/view/:view_key
GET  /support/account-report/:account_id/related/:view_key/:related_id
GET  /support/player-report/:account_id
GET  /support/item-history/:item_id
```

### Jira

```text
POST /support/jira-bug-reports
```

## 17.3 只读 Optool command 白名单

复刻时必须实现类似白名单。

当前系统支持的典型只读命令包括：

```text
Account.Find
Account.Find.JSON
Player.Stats
Player.ViewProfessions
Player.ViewPreviousPositions
Player.ViewPersistent
Player.ViewCharDef
Player.LocateAvatar
Skills.ShowProfessions
Invites.GetInfo
Invites.GetInvitedFriends
EffectOverTime.ListActive
EntityModifiers.ListActive
ClientCrashes.URL
ClientCrashes.eupdkURL
StandardReply.List
StandardReply.Get
Quest.ShowPlayerData
Item.Find
Item.Data
```

不要开放任意 command。

---

# 18. 前端 Support UI 系统

## 18.1 核心文件

```text
opencode/packages/app/src/context/support-store.tsx
opencode/packages/app/src/pages/support-cases.tsx
opencode/packages/app/src/pages/support-case-detail.tsx
opencode/packages/app/src/components/optool-auth-gate.tsx
```

## 18.2 页面结构

### Case List 页面

功能：

```text
显示 case 队列
过滤 case
查看 AI 状态
查看 assigned 状态
搜索 case
进入 case detail
```

### Case Detail 页面

通常是双栏：

```text
左侧：
  case thread
  posts
  attachments
  reply/comment controls
  proposal cards

右侧：
  agent session
  AI investigation terminal / iframe
```

## 18.3 OptoolAuthGate

前端会先检查：

```text
GET /support/optool-status
```

如果客服未登录 Optool，显示登录界面。

登录：

```text
POST /support/optool-login
```

## 18.4 Support Store

`support-store.tsx` 做状态管理。

启动时：

```text
GET /support/state
```

然后建立 SSE：

```text
EventSource /support/stream
```

它维护：

```text
cases
case details
proposals
agent sessions
queue
feedback
optool auth status
```

## 18.5 Proposal Card

Proposal card 必须显示：

```text
动作类型
目标玩家/account
原因
风险
将执行的 command
玩家回复草案
approve 按钮
deny 按钮
```

如果包含玩家消息，客服应该可以编辑。

审批时，编辑后的消息会通过：

```text
substitute_player_message_as
```

进入执行参数。

---

# 19. Agent Session 启动流程

## 19.1 入口

当客服点击：

```text
Analyze
```

前端调用：

```text
POST /support/cases/:id/analyze
```

当客服点击：

```text
Solve
```

前端调用：

```text
POST /support/cases/:id/solve
```

## 19.2 后端做什么

后端会：

```text
1. 确认 Optool 可用。
2. 确认 case 存在。
3. 刷新 case detail。
4. 生成 agent token。
5. 构造 support agent prompt。
6. 注入 case JSON。
7. 注入环境变量。
8. 启动 agent session。
9. 返回 session 信息给前端。
```

## 19.3 Agent prompt 注入内容

Prompt 中会注入：

```text
case id
case title
case category
case posts
account id
planet id
mode: analyze / solve
optool context
evidence environment
agent instructions
```

并要求 agent 阅读：

```text
agents/celest-support/agent.md
agents/celest-support/resources/RULES-POLICIES.md
agents/celest-support/resources/DECISION_FRAMEWORK.md
```

## 19.4 环境变量

关键环境变量：

```text
CELEST_SUPPORT_AGENT_TOKEN
CELEST_SUPPORT_CASE_ID
CELEST_SUPPORT_MODE
CELEST_SUPPORT_OPTOOL_TARGET
CELEST_SUPPORT_OPTOOL_SERVER_URL
```

---

# 20. Optool 集成方式

## 20.1 Optool 是什么

Optool 是内部客服 / 运维工具后端。

Celest Support 后端通过它读取和执行很多支持操作。

默认本地地址通常是：

```text
http://localhost:5000
```

配置：

```text
OPTOOL_SERVER_URL
```

## 20.2 登录方式

客服通过浏览器登录 Optool。

后端保存客服 session。

Agent 不拿原始凭证。

审批 proposal 时，后端用客服的登录态执行 Optool command。

## 20.3 为什么这样设计

因为：

```text
1. 人工客服必须承担最终动作责任。
2. 审计应归属于审批人。
3. Agent 不应拥有长期高权限凭证。
4. 防止 prompt injection 诱导 agent 直接操作内部系统。
```

---

# 21. 安全与权限模型

## 21.1 Agent Token

后端给 agent 的不是 Optool 凭证，而是 scoped token。

Token 用途：

```text
允许 agent 访问当前 case 相关 support endpoints
允许 agent 创建当前 case 的 proposal
限制 agent 只能操作当前 scope
```

## 21.2 Case Scope

Proposal 必须匹配当前 case。

后端会拒绝：

```text
case_id 不匹配
account_id 不匹配
planet_id 不匹配
source script 不合法
command 不在 allowlist
```

## 21.3 Related Account Scope

如果 case 涉及相关账号，必须先注册 related account。

脚本：

```text
register-case-related-player-account.ts
```

但 related account 不是无限权限。

只有少数操作允许作用于 related account。

## 21.4 Proposal Source Validation

创建 proposal 时，后端会检查：

```text
created_by_skill
created_by_script
```

并确认该脚本真实存在、路径合法、contract 合法。

## 21.5 Player Message Validation

如果 proposal 包含玩家回复，必须符合格式。

特别是结尾签名：

```text
Kind regards,
Entropia Universe Support
```

---

# 22. 子 Agent / Evidence 系统

Celest Support 不是一个 agent 单干到底。

它会根据任务使用专门调查角色。

## 22.1 data-investigator

负责：

```text
ClickHouse
经济历史
item flow
auction analytics
bounded historical evidence
```

适合：

```text
物品流转
经济争议
长时间历史事实
拍卖记录
收益/损失分析
```

## 22.2 log-investigator

负责：

```text
Graylog
server logs
service logs
deployment traces
backend incidents
```

适合：

```text
服务错误
平台事件
后端异常
时间点排查
```

## 22.3 crash-investigator

负责：

```text
Splunk crash
client crash signature
ClientLoader / Entropia.exe incidents
```

适合：

```text
玩家崩溃
客户端启动失败
crash signature 归因
```

## 22.4 knowledge-retriever

负责：

```text
知识库
游戏机制
源码-backed documentation
```

适合：

```text
解释机制
判断玩家说法是否符合游戏规则
查已有内部文档
```

## 22.5 bug-reporter

负责：

```text
创建 Jira bug report
```

只在明确要求时报 bug。

---

# 23. 本地运行方式

## 23.1 安装依赖

```powershell
cd C:\Users\haoli\Perforce\haoli_workspace\tools\celest
just install
```

## 23.2 构建检查

```powershell
just build
```

## 23.3 本地服务

常用命令：

```powershell
just run-local-backend
just run-local-web
just run-terminal-pty
just run-terminal-webui-client
```

当前常见端口：

```text
local backend: 31357
local web:     31358
terminal pty:  31367
webui client:  31368
Optool:        5000
```

注意：不同文档里可能有历史端口。以当前 `justfile` 为准。

---

# 24. Docker 部署方式

常用命令：

```powershell
just docker-build
just docker-up
```

当前 docker compose 中的重要服务包括：

```text
frontend
celest-core
celest-workspace
terminal-pty
opencode
slack-bot-router
scheduler
agent-orchestrator
dashboard-api
artifacts
admin
```

复刻部署时要准备：

```text
Docker socket
/data volume
workspace image
network
API keys
P4 credentials
Optool URL
Codex / Claude auth
evidence service env vars
```

旧文档可能提到：

```text
plugin-sync
```

但当前 compose 不再以它为主。以当前 `docker/docker-compose.yaml` 为准。

---

# 25. 复刻实施步骤

## Step 1：确定你要复刻的是内部客服系统

明确范围：

```text
目标：internal support agent
不是 player-facing chatbot
```

## Step 2：复制 agent root

复制：

```text
agents/celest-support/
```

必须包含：

```text
agent.md
resources/
skills/
subagents/
lib/
tests/
.codex-plugin/plugin.json
```

## Step 3：注册 skills

确保运行时能发现：

```text
agents/celest-support/skills
```

例如在配置中：

```json
{
  "skills": {
    "paths": ["agents/celest-support/skills"]
  }
}
```

## Step 4：实现后端 support routes

至少实现：

```text
/support/status
/support/optool-status
/support/optool-login
/support/cases
/support/cases/:id
/support/cases/:id/attachments
/support/cases/:id/analyze
/support/cases/:id/solve
/support/proposals
/support/proposals/:id/approve
/support/proposals/:id/deny
/support/state
/support/stream
```

## Step 5：实现 Optool bridge

需要：

```text
OPTOOL_SERVER_URL
登录态管理
command 调用
XML/HTTP response parsing
error mapping
```

## Step 6：实现 read whitelist

不要开放任意 command。

建立：

```text
SUPPORT_READ_COMMANDS
SUPPORT_SEARCH_VIEWS
PLAYER_VIEW_COLUMNS
```

或等价结构。

## Step 7：实现 proposal store

需要支持：

```text
create
list
get
approve
deny
state update
history mirror
SSE broadcast
```

## Step 8：实现 proposal executor

必须支持：

```text
execution.steps
command allowlist
player message substitution
audit summary
case eviction
special internal commands
```

## Step 9：实现前端 UI

至少需要：

```text
Optool login gate
case list
case detail
agent session panel
proposal cards
approve / deny
editable player message
```

## Step 10：实现 agent session launcher

启动 agent 时注入：

```text
case JSON
agent token
mode
optool context
required instruction files
environment variables
```

## Step 11：接入 evidence subagents

如果暂时不能接入，至少保留接口和 prompt 规则。

否则 agent 对历史经济、日志、崩溃问题会弱很多。

## Step 12：跑测试

先跑 skill-level tests，再跑 scenario-level tests。

---

# 26. 新增一个客服动作的完整流程

假设要新增：

```text
重置玩家某个状态
```

## 26.1 新增 skill script

路径示例：

```text
agents/celest-support/skills/player-actions/scripts/create-reset-something-proposal.ts
```

## 26.2 在脚本中创建 proposal

使用：

```text
agents/celest-support/lib/support-proposal.ts
```

生成：

```text
case_id
account_id
operation
planet_id
params
reason
presentation
execution
```

## 26.3 定义 execution step

例如：

```json
{
  "command": "Player.ResetSomething",
  "parameters": {
    "account_id": 12345
  }
}
```

## 26.4 更新 allowlist

文件：

```text
apps/terminal-webui/src/server/support/proposal-allowlist.ts
```

加入：

```text
Player.ResetSomething
```

## 26.5 更新 script contract

文件：

```text
apps/terminal-webui/src/server/support-routes.ts
```

注册：

```text
script -> operation -> command
```

## 26.6 更新前端展示

如果默认展示不够，更新：

```text
opencode/packages/app/src/lib/support-proposal-presentation.ts
```

## 26.7 写测试

测试至少覆盖：

```text
script creates proposal
backend accepts proposal
invalid account rejected
invalid command rejected
UI displays proposal
approve executes command
deny blocks execution
audit written
```

---

# 27. 测试与验收

## 27.1 测试目录

```text
agents/celest-support/tests/
```

## 27.2 全量 skill 测试

```powershell
bun agents/celest-support/tests/run-all-skill-tests.ts --account-id 65625 --case-id <case_id>
```

## 27.3 单 skill 测试

示例：

```powershell
agents\celest-support\tests\player-account\create-set-deposit-limit-proposal.bat
```

## 27.4 场景测试

示例：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File agents\celest-support\tests\run-scenario-end-to-end.ps1 -Scenario player_misconduct_report -AccountId 65625
```

## 27.5 更新测试状态

```powershell
bun agents/celest-support/tests/update-skill-e2e-status.ts
```

## 27.6 验收标准

一个完整客服闭环应满足：

```text
1. 客服能登录 Optool。
2. UI 能显示 support case。
3. UI 能显示 case posts。
4. UI 能下载附件。
5. 点击 Solve 能启动 agent session。
6. Agent 能读取 case。
7. Agent 能调用 skill scripts。
8. Agent 能创建 proposal。
9. UI 能显示 proposal card。
10. 客服能 approve。
11. 后端能执行 Optool command。
12. 执行结果能刷新到 UI。
13. 审计信息存在。
14. 玩家回复格式正确。
15. 错误 case 能被安全拒绝。
```

---

# 28. 常见坑点

## 坑 1：只复制 prompt

只复制：

```text
agent.md
```

不够。

必须复制：

```text
resources
skills
lib
subagents
backend routes
frontend UI
proposal system
```

## 坑 2：让 agent 直接调用 Optool mutation

这是严重错误。

正确模式：

```text
agent -> proposal -> human approval -> backend execute
```

## 坑 3：把 operation 当执行依据

错误：

```text
根据 operation 执行具体逻辑
```

正确：

```text
执行 execution.steps
```

## 坑 4：漏掉 script contract 校验

如果没有 script contract，一个低风险脚本可能伪造高风险 proposal。

必须校验：

```text
created_by_script
operation
command
```

## 坑 5：忘记 allowlist

新增 command 不进 allowlist，审批会失败。

## 坑 6：忘记 Optool 登录态

Agent token 不等于 Optool 登录态。

审批 proposal 前，客服必须登录 Optool。

## 坑 7：把 raw credentials 暴露给 agent

不要这样做。

当前系统已经明确废弃 `/support/optool-credentials`。

## 坑 8：附件没检查

有附件的 case，agent 必须下载并检查。

否则调查不完整。

## 坑 9：历史问题只看 live state

如果玩家说的是几天前、几周前的问题，live state 不够。

要查：

```text
ClickHouse
economic history
item flow
logs
crash reports
case history
```

## 坑 10：玩家回复泄露内部信息

玩家回复只能给结论和必要说明，不能给内部证据细节。

## 坑 11：related account 越权

必须先注册 related account。

且只能执行有限操作。

## 坑 12：移动玩家时顺手回复

`move_player` proposal 不允许带玩家消息。

分两个 proposal。

## 坑 13：不签名

玩家回复必须结尾：

```text
Kind regards,
Entropia Universe Support
```

## 坑 14：用旧路径

当前主路径是：

```text
agents/celest-support
apps/terminal-webui
opencode/packages/app
```

不是旧 `plugins/celest-support`。

## 坑 15：只跑单脚本测试

必须跑场景闭环测试。

---

# 29. 最小可用版本 MVP

如果你想先做一个最小可运行版本，可以这样切：

## 29.1 MVP 必须有

```text
1. case list
2. case detail
3. Optool login
4. agent session start
5. read case skill
6. reply proposal
7. close proposal
8. proposal card
9. approve/deny
10. backend execute
```

## 29.2 MVP 可以暂缓

```text
经济历史
复杂 account operations
Jira bug report
subagents
full player report
Docker production setup
bulk close
self-improvement proposal
```

## 29.3 MVP 不可以省略

```text
proposal gate
human approval
allowlist
agent token
player reply validation
```

如果省略这些，就不再是 Celest Support 的安全模型。

---

# 30. 完整复刻验收清单

## 30.1 Agent 层

```text
[ ] agent.md 存在
[ ] RULES-POLICIES.md 存在
[ ] DECISION_FRAMEWORK.md 存在
[ ] CASE_PLAYBOOK.md 存在
[ ] skills 被运行时发现
[ ] agent 启动时能读取 case JSON
[ ] agent 能调用 support scripts
[ ] agent 不直接执行 mutation
```

## 30.2 Backend 层

```text
[ ] /support/status 可用
[ ] /support/optool-status 可用
[ ] /support/optool-login 可用
[ ] /support/cases 可用
[ ] /support/cases/:id 可用
[ ] /support/proposals 可用
[ ] /support/proposals/:id/approve 可用
[ ] /support/proposals/:id/deny 可用
[ ] /support/state 可用
[ ] /support/stream 可用
```

## 30.3 Proposal 层

```text
[ ] proposal model 支持 execution.steps
[ ] command allowlist 存在
[ ] script contract 校验存在
[ ] player message validation 存在
[ ] account/case/planet scope 校验存在
[ ] approval 前二次校验存在
[ ] audit 写入存在
```

## 30.4 Frontend 层

```text
[ ] OptoolAuthGate 存在
[ ] case list 存在
[ ] case detail 存在
[ ] agent session panel 存在
[ ] proposal card 存在
[ ] approve/deny 按钮存在
[ ] player message 可编辑
[ ] SSE 状态刷新正常
```

## 30.5 Security 层

```text
[ ] agent 无原始 Optool 凭证
[ ] mutation 都 proposal-gated
[ ] read commands 有白名单
[ ] related account 有 scope 限制
[ ] 玩家回复不泄露内部信息
[ ] 高风险动作需要人工审批
```

## 30.6 Test 层

```text
[ ] skill tests 能跑
[ ] scenario tests 能跑
[ ] invalid proposal 被拒绝
[ ] unauthorized account 被拒绝
[ ] missing signature 被拒绝
[ ] approve 后动作真正执行
[ ] deny 后动作不会执行
```

---

# 31. 总结

Celest Support 的精髓不是“会聊天”，而是：

```text
把 AI 放在客服调查流程中，
让 AI 快速读 case、查证据、形成方案，
但把最终执行权牢牢放在人类和后端安全闸门手里。
```

如果要复刻一个功能等价的系统，必须同时复刻：

```text
Agent prompt
Policy resources
Skill scripts
Support backend
Optool bridge
Proposal system
Approval UI
State stream
Evidence subagents
Testing framework
```

最终目标是形成这样的闭环：

```text
玩家提交 case
    -> 客服打开 case
    -> AI 调查
    -> AI 创建 proposal
    -> 客服审批
    -> 后端安全执行
    -> 玩家收到回复
    -> 系统留下审计
```

这就是 Celest Support 的真实架构。
