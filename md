# Celest Support 智能客服 复刻操作文档

> 目标：完整重建一个与 `agents/celest-support` 功能等价的多代理客服系统。
>
> 阅读对象：要在另一个项目 / 另一个领域（电商客服、法务工单、IT helpdesk 等）复刻同款架构的工程师。
>
> 本文档基于 2026-05-12 的代码快照。所有路径都基于 `tools/celest/` 工作区。

---

## 0. 一句话概括

Celest Support 不是一个"会调工具的 LLM"，而是一个 **三层规则文档 + 五个领域子智能体 + 一个统一的"提案 + 人审"网关** 的系统。

任何对玩家、账号、世界状态的修改，都必须走 "脚本 → 生成提案卡 → 人类在 UI 上点批准 → 后端执行 optool 命令" 这条路；主代理只负责调查 + 起草，不直接动数据。

复刻它的关键，不是写一个聪明的 system prompt，而是把这套 **"证据-决策-提案-审批"** 的不可逾越的护栏先建好。

---

## 1. 总体架构（先看这张图）

```
┌────────────────────────────────────────────────────────────────┐
│                     Celest Support 主代理                       │
│  - 读 agent.md / SOUL / IDENTITY 作为人格锚点                   │
│  - 受限技能：support-cases / player-information /              │
│    player-actions / player-account / jira-bug-report           │
│  - 唯一职责：调查、判断、起草，绝不直接 mutate                  │
└──────┬─────────────────────────────────┬───────────────────────┘
       │ 委派（delegation packet）         │ 写入（必须经提案）
       ▼                                  ▼
┌──────────────────────┐         ┌──────────────────────────────┐
│  5 个领域子代理        │         │  Support Backend (Bun/Node)  │
│  scoped skill, MCP    │         │  /support/proposals 接口      │
│  受限                 │         │  /support/optool-read 接口   │
│                      │         │  /support/case-attachments    │
│  data-investigator    │         └──────┬───────────────────────┘
│  log-investigator     │                │
│  crash-investigator   │                ▼
│  knowledge-retriever  │         ┌──────────────────────────────┐
│  bug-reporter         │         │  Celest Support UI (前端)     │
└──────────────────────┘         │  人类客服审核提案、批准、编辑   │
       │                          └──────┬───────────────────────┘
       ▼                                  │
┌──────────────────────┐                  ▼
│ ClickHouse / Graylog │         ┌──────────────────────────────┐
│ Splunk / Knowledge   │         │ Optool Gateway (game server)  │
│ Jira                 │         │  真正执行：发邮件、补偿、传送、 │
└──────────────────────┘         │  解锁账号、改密码……          │
                                  └──────────────────────────────┘
```

四个不变量：

1. **主代理无写权限** — 所有可见技能都是 `read-only` 或 `create-proposal`，从不直接发 RPC。
2. **每个 mutation 都对应一张提案卡** — UI 上必须有人点 Approve。
3. **每个证据源都封进一个子代理** — 主代理不能直接 `clickhouse` / `graylog` / `splunk`。
4. **三份不可绕过的规则文档** — `RULES-POLICIES.md` / `DECISION_FRAMEWORK.md` / `CASE_PLAYBOOK.md`，每个 case 开局必读。

---

## 2. 工程文件结构（要原样复制的骨架）

```
agents/celest-support/
├── agent.md                          # 主代理 system prompt（500 行长）
├── resources/                        # 不可绕过的规则文档
│   ├── RULES-POLICIES.md             # 绑定性政策（合规底线）
│   ├── DECISION_FRAMEWORK.md         # 证据-判断框架（六级证据梯）
│   ├── CASE_PLAYBOOK.md              # 41 条剧本的路由索引
│   ├── case_playbook_library/        # 41 个具体剧本
│   │   ├── account-compromise-hack-claim.md
│   │   ├── compensation-amount-or-dispute.md
│   │   └── ...
│   └── FAQ.md                        # 玩家常见问答的标准说法
├── skills/                           # 主代理可调用的工具（按职责切 6 个）
│   ├── support-cases/                # 工单读 + 工单生命周期提案
│   │   ├── SKILL.md
│   │   └── scripts/
│   │       ├── get-support-case-details-and-posts.ts          (R)
│   │       ├── download-support-case-attachments.ts           (R)
│   │       ├── list-player-support-case-history.ts            (R)
│   │       ├── create-claim-support-case-proposal.ts          (W)
│   │       ├── create-reply-to-support-case-proposal.ts       (W)
│   │       ├── create-add-support-case-internal-comment-proposal.ts (W)
│   │       ├── create-close-support-case-proposal.ts          (W)
│   │       ├── create-escalate-support-case-proposal.ts       (W)
│   │       ├── create-modify-support-case-proposal.ts         (W)
│   │       └── create-reopen-support-case-proposal.ts         (W)
│   ├── player-information/           # 玩家档案 / 库存 / 历史（22 个全 R）
│   ├── player-actions/               # 玩家世界状态修改（15 个全 W）
│   ├── player-account/               # 账号 / 支付 / 安全（48 个，17R + 31W）
│   ├── jira-bug-report/              # Bug 直报，唯一非"提案+审批"路径
│   └── self-improvement/             # 工作流改进留档
├── subagents/                        # 5 个领域子代理定义
│   ├── data-investigator.md          # ClickHouse / 经济 / 物品历史
│   ├── log-investigator.md           # Graylog 服务端日志
│   ├── crash-investigator.md         # Splunk 客户端崩溃
│   ├── knowledge-retriever.md        # 知识库 / 机制说明
│   └── bug-reporter.md               # Jira 提单专员
├── lib/                              # 共享 TypeScript 运行时
│   ├── support-proposal.ts           # 提案数据契约 + 构造函数
│   ├── support-read.ts               # 后端 HTTP 客户端 + Optool 代理
│   ├── runtime-paths.ts              # 临时目录解析（hosted vs local）
│   ├── run-support-proposal-script.ts # 提案脚本统一外壳
│   ├── run-support-read-script.ts    # 读脚本统一外壳
│   └── linotp-search-view.ts         # 一个特定的 XML 解析器
├── templates/                        # 给新技能用的脚手架
│   ├── PROPOSAL_SKILL_TEMPLATE.md
│   ├── READ_SKILL_TEMPLATE.md
│   ├── create-proposal-operation-script-template.ts
│   └── create-read-operation-script-template.ts
├── improvements/                     # 留档：未来要改的工作流（异步沉淀）
└── tests / scenarios / templates     # 工程测试，不属于 live guidance
```

**Profile 文件**（决定主代理 vs 子代理能看到什么）：

```yaml
# profiles/support.yaml
description: Shared support profile for case handling, player investigation, and account actions.
skills:
  - support-cases
  - player-information
  - player-actions
  - player-account
  - jira-bug-report
agents:
  - celest-support
subagents:
  - data-investigator
  - log-investigator
  - crash-investigator
  - knowledge-retriever
  - bug-reporter
```

> **关键设计：MCP 作用域是物理隔离的。** 主代理在 MCP 服务器侧只被授予上面 5 个技能。哪怕它"想"调 ClickHouse 也调不到。子代理被孵化时拿到的是另一份更小的 token，比如 `data-investigator` 只能访问 `clickhouse / economy-gateway / economic-history / item-flow / auction-analytics`。

---

## 3. 三层文档：智能体的 "DNA"

这是整套系统里最容易被低估、又最关键的部分。如果只复制脚本不复制这三份文档，得到的会是一个表面像、实际乱来的山寨版。

### 3.1 `RULES-POLICIES.md` — 绑定法律

**作用**：定义"哪些事一定不能做、哪些事一定要做"。所有提案的 `reason` 字段都必须能在这份文档里找到落脚点。

**必含章节**：

| 章节 | 核心约束 |
|---|---|
| 0. Truth & Evidence Discipline | **"Inventing facts is forbidden."** 玩家可见的每一句具体陈述都必须有来源（工单帖、ClickHouse、Graylog、Splunk、optool、源码、官方文档、已查阅过的附件）。"听起来合理"不是来源。 |
| 1. Progressive Enforcement | 警告 → 临时封禁 → 永久封禁的纪律阶梯。仇恨言论 / 威胁 / 漏洞滥用 走硬线。 |
| 2. Account Security | 系统内置的安全机制是权威；客服只在其上操作。被盗的 PED 按"账号失窃"政策处理，不按同情心处理。 |
| 4. Player Responsibility | 经济结果（爆率、合成、市场）属于正常游戏风险，不予补偿。 |
| 5. Bugs & Technical Issues | Bug 行为必须从源码、构建清单、发布说明、崩溃遥测、日志中验证 — **不能从玩家描述里推断**。 |
| 10. Compensation | 补偿稀有、证据驱动；≥ 1000 PED 必须升级。账号若有"补偿索赔模式"要被标记。 |
| 11. Judgment & Goodwill | "Missing Graylog or Splunk confirmation is not a veto" — 如果库存/经济/支付记录已经证明事实，缺一个数据源不构成否决。账号价值可以影响**调查深度**和**低风险善意补偿**，但**不能**凌驾安全/支付/合规/纪律一致性/不可逆动作。 |

**复刻提示**：不要把"判断流程"写进这份文件，写"边界"。判断流程留给下一份。

### 3.2 `DECISION_FRAMEWORK.md` — 证据-判断协议

**作用**：把"我看到了什么"转换成"我该做什么"。

**核心结构（必须复刻）**：

1. **Actionability Rule**：可执行 ≠ 完美证据。"The agent may make proportionate judgment from incomplete records, but it may not create missing facts to make the story cleaner."

2. **Truth Gate** — 任何 case 先分成 4 类：
   - Claim：玩家说的 / 假设的
   - Verified fact：被认可来源证明的
   - Inference：从已验证事实做的判断
   - Gap：实质性未知

   **玩家可见的回复里不允许把 inference 当 fact 写。**

3. **Source Hierarchy**：
   - 3 小时前的历史 → ClickHouse 优先
   - 当前状态 → optool / support 列表视图
   - 服务端事件 → Graylog
   - 客户端崩溃 → Splunk
   - 产品/客户端行为 → 源码、构建清单、发布说明、knowledge-retriever

4. **Evidence Sufficiency Ladder（六级证据梯）**：
   - **Direct proof** → 直接创建提案
   - **Converging evidence**（多个部分证据，无重大冲突）→ 低/中风险可执行；高风险升级
   - **Mixed but plausible** → 仅允许低风险善意补偿；其他情况挂起 / 升级
   - **Scoped negative evidence**（搜索范围/时间/标识都对，没结果）→ 走政策性拒绝
   - **Load-bearing gap**（关键附件/数据缺失）→ 升级或挂起
   - **Player-only assertion** → 不够动手

5. **Low-Risk Goodwill Rules**：≤ 1000 PED；具体且合理；证据混合（不是已被证伪）；无重复索赔模式；账号关系/服务摩擦支持给。**善意补偿的内部原因留 reason 字段，玩家不可见**。

6. **Final Decision Check**（提案前的 6 个镜像问题）：
   - 公平所需的事实是什么？
   - 哪些记录证明 / 削弱 / 没能证明？
   - 缺的数据源是否真的载荷关键？
   - 动作与风险/不可逆性是否成正比？
   - **同样的事实换一个玩家，会得到一样的结果吗？**
   - 审核者能否只读 reason 就批准，不必再跑一遍调查？

### 3.3 `CASE_PLAYBOOK.md` + `case_playbook_library/*.md` — 领域剧本

41 份剧本，每份都长这样：

```markdown
## Use When
<这个剧本接什么样的工单：信号词、症状>

## Core Evidence
<开头要读哪些数据：工单帖、玩家全报告、特定 ClickHouse 查询、optool 当前状态>

## Delegate When
<什么时候交给子代理：data-investigator / log-investigator / crash-investigator>

## Resolution Paths
<3-4 种常见走向：每种附带具体的 proposal 脚本路径>

## Proposal Paths
<可调用的提案脚本完整命令>

## Escalate When
<硬性升级触发：高金额、支付/合规、不可逆操作、未授权动作>
```

**Routing Precedence Rules**（防止重叠路由不一致）：

```
Compensation 优先于 missing-item
Trapped Space Storage 优先于 missing-item
Auction/marketplace 优先于 missing-item/PED
Multi-account/ToS lock 优先于 item restoration
Account compromise 优先于 compensation/missing-item
Duplicate/bump cases → 直接升级，除非加了实质事实
```

> **复刻经验**：剧本不是 "if-then" 决策树，更像"领域提示卡"。主代理永远保留最终判断权，剧本只负责"指你去哪儿找证据"。一个新领域至少要 30 份剧本起，否则代理会在边角情景下乱编。

---

## 4. 主代理 `agent.md` 的写法范本

约 500 行，分成下列固定章节（顺序很重要）：

```
1. Operating Contract          # 每个 case 的 15 步默认序列
2. Canonical Sources           # 哪些文件是真相来源
3. Support Knowledge Boundary  # rg 只允许扫这些目录
4. Actionable Bug Reports      # 什么时候交 bug-reporter
5. Human Operator Authority    # 人类客服是最终决定人；推回一次就执行
6. Truth & Source Discipline   # 永远不要编事实
7. Platform Failure Protocol   # 平台故障的统一报错格式
8. Proposal Model              # 写路径只有提案
9. Player Communication        # 回复签名 "Kind regards, Entropia Universe Support"
10. Evidence Priority          # 3 小时以上 → ClickHouse 优先
11. Capability Surface         # 5 个技能 + 5 个子代理的完整清单
12. Environment Scope          # 环境变量、Graylog 环境绑定
13. Subagent Delegation Packet # 委派包的字段格式
14. Subagent Skill Scopes      # 每个子代理用什么模型 / 推理力度
15. Investigation Depth        # 风险匹配深度
16. Evidence Habits            # webshop / 附件 / 误判处理细则
17. Misconduct Cases           # 双方都要查
18. Outcomes                   # 三种合法结局
19. Case Workflow              # 默认新工单的 12 步流程
20. Investigation Summary      # 留档格式
```

**关键句式（强烈建议照抄）**：

- "Read \<file\> at the start of every case."
- "When evidence and a playbook disagree, follow the evidence inside policy."
- "Use the approved support scripts as the evidence surface. Platform repair belongs to Celest engineering, not the case workflow."
- "You provide facts. The main support agent decides the support outcome."（子代理通用收尾句）

---

## 5. Skill / Script 契约（最容易写错的部分）

### 5.1 Skill 目录形态

```
skills/<skill-name>/
├── SKILL.md                          # 用 YAML frontmatter 声明 name + description
└── scripts/
    ├── get-*.ts                      # 读脚本
    ├── list-*.ts                     # 读脚本
    ├── view-*.ts                     # 读脚本
    ├── create-*-proposal.ts          # 写脚本（必须以 -proposal.ts 结尾）
    └── ...
```

`SKILL.md` 的 frontmatter：

```yaml
---
name: support-cases
description: "Read support-case data and create proposal cards for case lifecycle actions: claim, reply, internal comment, reopen, escalate, modify, close, and bulk close."
---
```

文档主体强制有 5 块：

```markdown
## Read Scripts           # 表格：脚本路径 + 用途
## Proposal Scripts       # 表格：脚本路径 + 用途
## Common Proposal Flags  # --case-id / --player / --planet-id / --reason / --dry-run / --verbose / --backend-url
## Operating Notes        # 玩家可见 vs reason 的边界、签名规则、附件规则
## Platform Failure       # 出现 5xx / allowlist 错时怎么报
```

### 5.2 读脚本骨架

```typescript
import { fetchPlayerView, parseInteger } from "../../../lib/support-read.ts"
import {
  parseSupportReadScriptFlags,
  runSupportReadScript,
  type SupportReadScriptFlags,
} from "../../../lib/run-support-read-script.ts"

type Flags = SupportReadScriptFlags & { accountId?: number }

function parseArgs(args: string[]): Flags {
  const flags: Flags = { verbose: false, backendUrl: undefined, accountId: undefined }
  return parseSupportReadScriptFlags(args, flags, (arg, value, flags) => {
    if (arg === "--account-id") { flags.accountId = parseInteger(value, arg); return true }
    return false
  })
}

export async function main(args = Bun.argv.slice(2)) {
  await runSupportReadScript({
    args,
    parseArgs,
    execute: async (flags) => {
      if (!flags.accountId) throw new Error("Missing required argument: --account-id")
      return fetchPlayerView("account_profile", flags.accountId, flags)
    },
  })
}

main().catch((error) => {
  const message = error instanceof Error ? error.message : String(error)
  console.error(JSON.stringify({ status: "error", error: message }))
  process.exit(1)
})
```

**契约**：

- stdout 必须是合法 JSON
- stderr 上的错误也必须是 JSON：`{ "status": "error", "error": "..." }`
- 一定支持 `--verbose` 和 `--backend-url`
- **永远不写文件**

### 5.3 写脚本骨架（提案）

```typescript
import {
  requireNumber,
  requireText,
  parseInteger,
  optoolExecution,
  type SupportProposalInput,
} from "../../../lib/support-proposal.ts"
import {
  parseSupportProposalScriptFlags,
  runSupportProposalScript,
  type SupportProposalScriptFlags,
} from "../../../lib/run-support-proposal-script.ts"

type Flags = SupportProposalScriptFlags & {
  caseId?: number
  player?: number
  planetId?: number
  reason?: string
  playerMessage?: string
}

function parseArgs(args: string[]): Flags {
  const flags: Flags = {
    dryRun: false, list: false, verbose: false,
    caseId: undefined, player: undefined, planetId: undefined,
    reason: undefined, playerMessage: undefined,
  }
  return parseSupportProposalScriptFlags(args, flags, (arg, value, flags) => {
    if (arg === "--case-id") { flags.caseId = parseInteger(value, arg); return true }
    if (arg === "--player") { flags.player = parseInteger(value, arg); return true }
    if (arg === "--planet-id") { flags.planetId = parseInteger(value, arg); return true }
    if (arg === "--reason") { flags.reason = value; return true }
    if (arg === "--player-message") { flags.playerMessage = value; return true }
    return false
  })
}

function buildProposal(flags: Flags): SupportProposalInput {
  const caseId   = requireNumber(flags.caseId,   "--case-id")
  const player   = requireNumber(flags.player,   "--player")
  const planetId = requireNumber(flags.planetId, "--planet-id")
  const reason   = requireText(flags.reason,     "--reason")

  return {
    case_id: caseId,
    account_id: player,
    operation: "Operator.AddComment",
    planet_id: planetId,
    params: {
      comment_type: 3,
      comment_text: "Internal note",
      created_by_skill: "player-account",
      created_by_script: "create-add-operator-comment-on-account-proposal.ts",
    },
    reason,
    player_message: flags.playerMessage,
    execution: optoolExecution(
      "Operator.AddComment",
      { AccountID: player, CommentType: 3, Text: "Internal note", Comment: reason },
      `Added operator comment on account ${player}`,
    ),
  }
}

export async function main(args = Bun.argv.slice(2)) {
  await runSupportProposalScript({ args, parseArgs, buildProposal, renderList: () => [] })
}

main().catch((error) => {
  const message = error instanceof Error ? error.message : String(error)
  console.error(JSON.stringify({ status: "error", error: message }))
  process.exit(1)
})
```

**关键不变量**：

- `SupportProposalInput.execution` 字段**必须存在**，里面的 `steps` **必须正好 1 个**（后端拒绝 multi-step）
- 提案 = 一次 optool 调用 = 一张人审卡。需要"补偿 + 回复 + 关单"就开 3 张提案，按顺序排
- `parameters` 在脚本里就要算好；后端只做"按字面值传给 optool"，不做模板插值
- 玩家可见消息（`player_message`）和内部原因（`reason`）**严格不能混**

---

## 6. 共享 lib 的实现要点

### `support-proposal.ts` 必须导出

```typescript
export interface ProposalOptoolStep {
  command: string                              // "Economy.AcceptPendingTransaction" 或 "__support_noop__"
  parameters: Record<string, unknown>
  failure_message?: string
  if_player_message?: boolean                  // 关单时是否要附带最终回复
  substitute_player_message_as?: string        // 让 staff 改过的回复写入 optool 参数的哪个键
}

export interface ProposalExecution {
  steps: ProposalOptoolStep[]                  // 长度只能是 1
  audit_summary?: string
  evicts_case?: boolean                        // 关单/最终结案后从工作区清掉
}

export interface SupportProposalInput {
  case_id: number
  account_id: number
  operation: string                            // UI 显示用，后端不再 dispatch
  planet_id: number
  params: Record<string, unknown>
  reason: string                               // 仅内部可见
  player_message?: string                      // 玩家可见
  presentation?: Partial<ProposalPresentation> // 卡片渲染契约
  execution: ProposalExecution                 // 必填
}

// 构造器
export function optoolExecution(command, params, audit_summary, failure_message?): ProposalExecution
export function noopExecution(audit_summary, result?): ProposalExecution

// 保留命令
export const NOOP_PROPOSAL_COMMAND = "__support_noop__"
export const SUPPORT_IMPROVEMENT_MANIFEST_COMMAND = "__support_create_improvement_manifest__"

// 网关
export async function createSupportProposal(input, opts): Promise<{ proposal: {...} }>
```

### `support-read.ts` 必须提供

```typescript
// 后端地址解析（按这个优先级）
export function supportBackendBaseUrl(): string {
  return env.CELEST_SUPPORT_BACKEND_URL
      || env.TERMINAL_WEBUI_SUPPORT_BACKEND_URL
      || env.CELEST_TERMINAL_WEBUI_URL
      || "http://localhost:31367"
}

// 鉴权
export function supportAgentHeaders(): HeadersInit {
  return env.CELEST_SUPPORT_AGENT_TOKEN
    ? { "x-celest-support-agent-token": env.CELEST_SUPPORT_AGENT_TOKEN }
    : {}
}

// 读 API
export async function fetchSupportJson(path, opts)
export async function fetchPlayerView(viewName, accountId, opts)       // 白名单 optool views
export async function fetchOptoolSearchView(viewName, params, opts)
export async function execOptoolReadCommand(command, params, opts)
export async function fetchPlayerReport(accountId, opts)
export async function fetchSupportCaseAttachments(caseId, opts)
export async function downloadSupportCaseAttachment(...)
export async function registerCaseRelatedPlayerAccount(caseId, accusedAccountId, reason)
```

### `runtime-paths.ts`

```typescript
export function supportScratchRoot(): string {
  // 1. CELEST_SUPPORT_SCRATCH_DIR
  // 2. /data/celest-support （当 CELEST_RUNTIME_CONTEXT=hosted）
  // 3. .celest-support （本地）
}

export function supportScratchPath(kind: "case-attachments" | ..., ...segments: string[]): string
```

---

## 7. Subagent 复刻范本

每个 subagent 是一个独立的 markdown 文件，YAML frontmatter + 内容。

```yaml
---
name: data-investigator
description: "Historical data specialist for Celest Support. Use for ClickHouse, economy, item-flow, auction, and bounded historical evidence questions."
mode: subagent
model: gpt-5.5            # Codex 用；Claude Code 侧会自动注入 claude-opus-4-7
reasoningEffort: xhigh
skills:
  - clickhouse
  - economy-gateway
  - economic-history
  - item-flow
  - auction-analytics
tools: [Read, Grep, Glob, Bash]
---
```

**正文必须含**：

1. **Use Cases**：什么样的问题该来找我
2. **Input Contract**：委派包要带哪些字段
3. **Available Skills**：我能用哪些工具
4. **Source Review**：什么时候可以读源码（narrowly）
5. **Evidence Sources**：具体的查询入口 / 环境变量
6. **Method**：调查步骤
7. **Output**：返回结构必须是

```markdown
## Finding
<对委派问题的直接回答>

## Evidence strength
<direct proof | converging evidence | scoped negative evidence | load-bearing gap | inconclusive> - <一句话理由>

## Evidence
- <时间戳 / id / 金额 / 持有人 / 来源表>

## Queries run
- <跑过的 query 及理由>

## Caveats
<新鲜度、搜索范围、不确定性>
```

8. **末句**："You provide facts. The main support agent decides the support outcome."

**主代理委派包格式**（必须照抄）：

```text
[DELEGATION_PACKET]
case_id: <id>
case_created_at: <timestamp or unknown>
claim_to_verify: <具体事实主张>
incident_window: <带时区的时间窗>
player_or_account_ids: <ids or unknown>
avatar_or_login: <name/login>
environment: <pro01/rel01/...>
location_or_server_context: <如相关>
known_identifiers: <item ids / tx ids / 崩溃签名 / dump url>
prior_reads:
- <script/source>: <result>
exact_questions:
- <子代理必须回答的问题>
what_would_prove_or_disprove:
- prove: <预期会出现的证据>
- disprove: <预期不会出现的证据>
allowed_sources: <ClickHouse/Graylog/Splunk/...>
freshness_or_scope: <可接受的搜索窗口>
```

子代理收到不全包要返回 `INSUFFICIENT_DELEGATION_PACKET` + 缺失字段名，**不要硬猜**。

---

## 8. 后端 API 契约（Bun/Node 服务侧）

你的 Support Backend 必须实现这几个端点（默认监听 `:31367` 或 `:4097`）：

| 方法 | 路径 | 用途 |
|---|---|---|
| GET  | `/support/cases/:case_id` | 工单详情 + post |
| GET  | `/support/cases/:case_id/attachments` | 附件列表 |
| GET  | `/support/cases/:case_id/attachments/:id/download` | 附件下载 |
| GET  | `/support/player-view/:view/:account_id` | 白名单 optool view 代理读 |
| POST | `/support/optool-read` | 白名单 optool 命令代理读 |
| POST | `/support/optool-search-view` | optool 搜索视图 |
| GET  | `/support/players/:account_id/report` | 玩家全报告（聚合多源） |
| POST | `/support/case-related-accounts` | 给指控对象的账号发"案件相关"临时 token |
| POST | `/support/proposals` | **创建提案卡（唯一写入入口）** |

`POST /support/proposals` 的请求体就是 `SupportProposalInput` 序列化。后端必须：

1. 校验 `execution.steps.length === 1`
2. 校验 `command` 在白名单内（optool 命令列表 + 两个保留命令）
3. 入库 SQLite（或别的轻库），状态 `pending`
4. UI 显示后等人审；批准时由后端再走 optool gateway 真正执行

**鉴权**：所有 `/support/*` 请求头要带 `x-celest-support-agent-token`。

---

## 9. 复刻 step-by-step（按这个顺序做）

### Phase 1：地基（先把规则文档建好，1-2 天）

1. 把 `RULES-POLICIES.md` 翻译/改写到你的领域（合规底线、补偿规则、纪律阶梯、责任划分）
2. 写 `DECISION_FRAMEWORK.md`，保留六级证据梯和 final-check
3. 写 `CASE_PLAYBOOK.md` 索引 + 至少 20 份具体剧本

> 这一阶段产物是 markdown，**先不写代码**。先让一个 LLM 用这三份文档跑沙箱演练，找漏洞。

### Phase 2：lib + 后端（3-5 天）

1. 用 Bun 起一个最小 server，实现 `/support/proposals` 入库（先不接真实下游）
2. 写 `support-proposal.ts` / `support-read.ts` / `run-support-*-script.ts`
3. 用 templates 写出 1 个读脚本 + 1 个写脚本，能跑 `--dry-run` 并打出合法 JSON
4. 跑通"写脚本 → 后端创建 pending 提案 → SQLite 入库"这条链

### Phase 3：人审 UI（3-5 天）

1. 列出待审提案
2. 渲染卡片：操作摘要 + reason + player_message（可编辑）+ execution 详情
3. Approve 按钮：调下游执行；Deny 按钮：直接关掉
4. 审计日志：谁批的、什么时候、最后下游返回什么

> **没有 UI 不准上线。** 整个系统的安全性就建立在这一步上。

### Phase 4：技能套装（按领域按需，1-3 周）

复刻 Celest 的 6 个技能模板：

- `<case>-cases` — 工单生命周期（claim / reply / close / escalate / 内部备注）
- `<player>-information` — 只读档案查询
- `<player>-actions` — 用户世界状态修改
- `<player>-account` — 账号 / 安全 / 支付
- `<bug>-report` — 直发开发 Jira（**唯一不走"提案+审批"的路径**，必须严格限定触发条件：人类客服显式要求）
- `self-improvement` — 工作流改进留档

每个技能严格按读 / 写分离。读不要 dry-run；写一定有 dry-run。

### Phase 5：子代理（1-2 周）

1. 先建 MCP 子代理孵化机制：能用更小的 token 启动子进程
2. 一个一个加：先 `knowledge-retriever`（最简单），再 `data-investigator`（最重要），最后 `crash-investigator` / `log-investigator` / `bug-reporter`
3. 每个子代理写完后，**用主代理给它发 5 个不同的委派包，验证它会正确返回 `INSUFFICIENT_DELEGATION_PACKET`**

### Phase 6：主代理 prompt + 验收（1 周）

1. 写完整的 `agent.md`，按 20 章节顺序
2. 跑 30 个真实历史 case 的回放（不接真实下游，只看提案 + reason 的质量）
3. 让 2 位人类客服盲审：他们能否仅凭 reason 就批准？

### Phase 7：上线（与产品节奏对齐）

1. 先开 1-2 个低风险路由（FAQ / 工单认领 / 玩家姓名修改）
2. 监控：提案批准率、改写率（staff 改 player_message 的频率）、平均处理时间
3. 逐路由扩展

---

## 10. 踩坑清单（前人血泪）

### A. 架构级

- **不要让主代理直接调下游数据源**。一旦主代理能直查 ClickHouse，过一阵子它就会跳过子代理直接乱查，证据质量崩盘。MCP 强物理隔离比 prompt 约束有效得多。
- **不要把 case_playbook 写成决策树**。要写成"信号 + 证据指引"。决策树会让代理在边角情景下卡住或乱编。
- **不要省 `DECISION_FRAMEWORK.md`**。只有 `RULES-POLICIES` 没有判断协议，模型会在证据混杂时随机摇摆。
- **不要把同情心写进政策**。补偿必须证据驱动，否则会被玩家系统性钓利。
- **不要让 `--player-message` 和 `--reason` 共用文本**。Celest 出过事故：staff 把内部判断词（"no refund needed"）直接发给玩家。在 SKILL.md 里硬性约束。

### B. 数据契约级

- **`ProposalExecution.steps` 长度必须是 1**。后端拒绝 multi-step。需要原子组合 → 不可能；要么改 optool 加新命令，要么拆成多张卡。
- **`parameters` 字段在脚本里算好**。不要在后端做模板替换 — 会让审批人看不到最终值。
- **必须支持 `--dry-run`**。审核流程要求 staff 能预览。
- **后端 URL 解析顺序错了会静默走错环境**。env var 优先级链一定要测。

### C. 子代理级

- **委派包不全要 `INSUFFICIENT_DELEGATION_PACKET`，不要硬猜**。猜出来的"证据"会被主代理当真，污染整个工单。
- **子代理不能跨域**。`data-investigator` 不准查日志；`log-investigator` 不准查 ClickHouse。报"我搞不定，请换 specialist"，不要自己越界。
- **evidence strength 标签必须返回**。主代理用这个标签喂 DECISION_FRAMEWORK。
- **不要让子代理输出 "我建议补偿 50 PED"**。子代理只返回 fact，主代理才做判断。强约束："You provide facts. The main support agent decides the support outcome."

### D. 玩家可见层

- **回复必须有统一签名**。Celest 是 `Kind regards, Entropia Universe Support`。在 `support-proposal.ts` 里校验。
- **不要在玩家回复里暴露内部机制**。客户端版本号、Optool 命令、ClickHouse 表名一律不行。
- **"You will receive compensation"** 这种承诺只能在提案已经创建 PED 补偿步骤后写。否则会让 staff 陷入"必须批"的尴尬。
- **附件必须真的看过才能引为证据**。`view_image` 调用要落到每张图。"基于附件" 但没真打开 → 幻觉重灾区。

### E. 环境 / 运维级

- **Graylog 是 per-env 的，ClickHouse 在 Celest 当前只用 pro01**。委派包必须带 environment 字段，否则代理会因为示例用 `pro01` 就盲查 `pro01`。
- **凭据通过 env var 注入**：`CLICKHOUSE_USER/PASS`、`GRAYLOG_USER/PASS`、`CELEST_SPLUNK_CRASH_HOST`、`CELEST_SUPPORT_AGENT_TOKEN`。在容器/启动器里检查注入成功，否则代理会把"找不到日志"上报成业务结论。
- **scratch dir 路径区分 hosted vs local**。容器里写 `/data/celest-support/case-attachments/<case_id>`，本地写 `.celest-support/...`。

### F. 流程级

- **平台故障要走专门协议**：`[PLATFORM_MISCONFIGURATION]` 模板。不要让模型把后端 500 解释成"调查无果"。
- **不要让主代理"自我修复"工作流**。改进想法走 `self-improvement` 提案，sleep 到工程评审 — 不要让代理边干边改 prompt。
- **重复 / bump 工单先升级再说**。除非加了新事实。否则代理会把同一个 case 调查 3 次。

---

## 11. 验收清单（上线前自检）

- [ ] 主代理无任何直接 mutate 下游的代码路径（grep 所有 fetch / POST / write）
- [ ] 每个 write 脚本都生成 `SupportProposalInput`，`execution.steps.length === 1`
- [ ] `--dry-run` 能在不创建提案的前提下输出完整 payload
- [ ] 所有读脚本 stdout 是 JSON，stderr 错误也是 JSON
- [ ] 5 个子代理各被 5 个不全包测试过，都返回 `INSUFFICIENT_DELEGATION_PACKET`
- [ ] 主代理 system prompt 启动时强制读 `RULES-POLICIES.md` + `DECISION_FRAMEWORK.md`
- [ ] 玩家可见回复必带统一签名，由 `support-proposal.ts` 强制校验
- [ ] UI 有人审；Approve 才能真正调下游
- [ ] 审计日志：提案 id / 创建脚本 / 审批人 / 下游执行结果，全留存
- [ ] 30 个回放 case，2 位人类盲审通过率 ≥ 80%

---

## 12. 适配到不同领域的速查表

| Celest 客服 | 电商客服 | IT helpdesk | 法务工单 |
|---|---|---|---|
| support-cases | tickets | incidents | matters |
| player-information | customer-profile | asset / user | client-profile |
| player-actions | order-actions（退款/补发）| user-actions（重置密码）| filing-actions |
| player-account | account / payment | identity / SSO | client-account |
| jira-bug-report | jira / linear | jira / servicenow | docket entry |
| data-investigator | warehouse / orders DB | CMDB / monitoring | doc retrieval |
| log-investigator | server logs | syslog / SIEM | system audit |
| crash-investigator | app crash | endpoint EDR | n/a |
| knowledge-retriever | FAQ / manual | runbook | precedent search |
| bug-reporter | engineer escalation | L3 escalation | partner referral |
| Optool gateway | OMS / 支付通道 | AD / 工单平台 | DMS |

**通用迁移原则**：

1. 一切修改先走"创建提案 → 人审"，永远不直接 mutate
2. 一份绑定政策 + 一份判断框架 + 一组领域剧本，三者不能合并
3. 数据源按"领域"切子代理，每个子代理只懂自己的工具
4. 主代理只有 5-6 个面向业务的技能，下游证据全靠委派
5. 子代理统一的输出契约（Finding / Evidence strength / Evidence / Queries / Caveats）
6. 玩家/客户可见输出和内部 reason 严格分离
7. `INSUFFICIENT_*_PACKET` 是合法返回，不是失败

---

## 13. 最小化 PoC（如果只想先证可行性）

如果你想用 2 个工程日先验证整套思想：

- **Day 1**：
  - 写 `RULES-POLICIES.md`（500 字）+ `DECISION_FRAMEWORK.md`（六级梯）+ 3 份剧本
  - Bun 起 server，只实现 `POST /support/proposals` 入 SQLite + 一个最简 HTML 审核页
  - 一个读脚本（`get-ticket.ts` 调你的 API）+ 一个写脚本（`create-refund-proposal.ts`）

- **Day 2**：
  - 主代理 `agent.md` 缩写到 100 行
  - 一个子代理 `data-investigator`：只查 Postgres / sqlite，返回标准格式
  - 跑 5 个 case，看主代理是否会：
    - 开局就读两份文档
    - 给 data-investigator 发完整委派包
    - 创建符合 schema 的提案
    - 在 UI 上点 Approve 后真的调到下游 mock 函数

把这个 PoC 跑通，再按 Phase 2 的清单扩展。**不要先写很多技能再想着加护栏，护栏永远先建**。

---

## 附录 A：完整脚本文件清单（按技能）

### support-cases（15 个，7R + 8W）

| 脚本 | 类型 |
|---|---|
| get-support-case-details-and-posts.ts | R |
| download-support-case-attachments.ts | R |
| list-player-support-case-history.ts | R |
| search-support-cases.ts | R |
| view-support-queue-status.ts | R |
| list-standard-reply-templates.ts | R |
| get-standard-reply-template.ts | R |
| create-claim-support-case-proposal.ts | W |
| create-reply-to-support-case-proposal.ts | W |
| create-add-support-case-internal-comment-proposal.ts | W |
| create-reopen-support-case-proposal.ts | W |
| create-escalate-support-case-proposal.ts | W |
| create-modify-support-case-proposal.ts | W |
| create-close-support-case-proposal.ts | W |
| create-bulk-close-inactive-cases-proposal.ts | W |

### player-information（22 个，全 R）

入口：
- get-player-full-investigation-report.ts （**主代理拿到玩家 id 后第一个调的**）
- find-player-account-by-name-or-email.ts
- register-case-related-player-account.ts （查指控对象前必跑）

聚焦：
- get-player-account-profile-and-status.ts
- view-player-inventory-and-storage.ts
- list-player-auction-and-marketplace-history.ts
- view-player-economic-balances.ts
- trace-item-history-by-id.ts
- 等等

### player-actions（15 个，全 W）

主要场景：传送到安全区、强制登出、补偿 PED、重建物品、重置任务、清除负面效果。
**唯一不需要 `--player-message` 的技能** — 因为这些是世界状态修改，玩家回复要单独开 reply 提案。

### player-account（48 个，17R + 31W）

读：操作员注释、支付状态、卡列表、安全设备、邮件历史……
写：改邮箱、改密码、改额度、加 / 减账号 flag、增加操作员注释、发标准邮件、终止账号……

### jira-bug-report（1 个）

`submit-jira-bug-report.ts` — **不走"提案+审批"，直接提单**。但只有在：
- 人类客服显式要求
- 已经收集到 confirmed_bug / highly_likely_bug 等级证据
- 13 个必填章节全都填好
- 验证过没有未解决的占位符

时才允许调用。这是整个系统**唯一**的例外。

### self-improvement（1 个）

`create-support-workflow-improvement-proposal.ts` — 工作流改进，要走特殊保留命令 `__support_create_improvement_manifest__`，由后端在 Perforce 里加一个文件。

---

## 附录 B：保留命令速查

| 命令 | 含义 |
|---|---|
| `__support_noop__` | 不调 optool，但走完审批 + 审计。用于 read-modify-write 时目标状态已满足的情况 |
| `__support_create_improvement_manifest__` | 在 improvements/ 下创建一份评审过的改进 manifest 并 p4 add |
| Optool 真命令 | "SupportCase.CheckOut" / "Economy.AcceptPendingTransaction" / "Operator.AddComment" / "Player.Teleport" 等等 |

---

## 附录 C：所有环境变量

```bash
# 后端发现 (按优先级)
CELEST_SUPPORT_BACKEND_URL=http://localhost:4097
TERMINAL_WEBUI_SUPPORT_BACKEND_URL=...
CELEST_TERMINAL_WEBUI_URL=...
# 默认 http://localhost:31367

# 鉴权
CELEST_SUPPORT_AGENT_TOKEN=<token>

# 临时目录
CELEST_RUNTIME_CONTEXT=hosted          # 控制 /data/... 还是 .celest-support/...
CELEST_DATA_DIR=/data
CELEST_SUPPORT_SCRATCH_DIR=/custom

# 证据源
CLICKHOUSE_USER / CLICKHOUSE_PASS
CLICKHOUSE_USERNAME / CLICKHOUSE_PASSWORD     # 别名
CELEST_CLICKHOUSE_ENVIRONMENT=pro01

GRAYLOG_USER / GRAYLOG_PASS
CELEST_GRAYLOG_ENVIRONMENT
CELEST_SUPPORT_EVIDENCE_ENVIRONMENT

CELEST_SPLUNK_CRASH_HOST

# Optool（用于 get-support-case-data 直连）
OPTOOL_SERVER_URL
OPTOOL_USERNAME
OPTOOL_PASSWORD
```

---

完了。把这份文档配合 `tools/celest/agents/celest-support/` 目录里的真文件一起读，照着复刻一遍，最多两周能拿到一个能跑的同款架构。

最大的提醒只有一个：**先把三层规则文档 + 提案+审批流写好，再写脚本，再加子代理**。反过来会把自己埋进幻觉里。
