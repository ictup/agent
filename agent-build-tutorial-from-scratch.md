# 从零搭建一个 "受控多代理客服系统" — 完全零基础详细教程

> **谁该读这份文档**：
> 1. 没接触过 LLM Agent / 多代理 / MCP 的开发者
> 2. 想把这套架构搬到自己的垂直领域（客服、合规、IT、风控、医疗工单…）
> 3. 时间预算 10-15 个工作日 / 一个工程师独立完成
>
> **你将造出什么**：
> 一个完整能跑的 Agent 系统，包含：1 个主代理、2 个专科子代理、4 个业务技能、6 个脚本、1 个提案后端、1 个人审 UI、3 份政策文档。整个工程约 30 个源文件，全部代码本文给齐，照抄即可运行。
>
> **阅读方式**：
> 强烈建议**边读边动手**。每一节末尾都有"验证步骤"，过了再进下一节。卡住时先回到上一节的验证步骤检查。

---

# 目录

- [Part A 概念扫盲（零基础必读）](#part-a)
- [Part B 系统总览与设计哲学](#part-b)
- [Part C 选你的领域 / 本教程的示例选型](#part-c)
- [Part D 环境准备](#part-d)
- [Day 1 — 写三份规则文档（不写代码）](#day-1)
- [Day 2 — 搭提案后端 + SQLite](#day-2)
- [Day 3 — 搭人审 UI](#day-3)
- [Day 4 — 写共享 SDK（脚本运行器）](#day-4)
- [Day 5 — 第一个只读技能](#day-5)
- [Day 6 — 第一个写技能（提案）](#day-6)
- [Day 7 — 主代理：跑通端到端](#day-7)
- [Day 8 — 第一个子代理 + 委派协议](#day-8)
- [Day 9 — 第二个子代理 + 多源证据汇总](#day-9)
- [Day 10 — 集成测试 / 三个真实 case 回放](#day-10)
- [Part E 适配到你的领域 / 迁移 checklist](#part-e)
- [Part F 故障排查](#part-f)
- [附录 完整文件树 / 环境变量 / 命令速查](#appendix)

---

<a id="part-a"></a>

# Part A — 概念扫盲（零基础必读）

如果下面这些词你已经熟，直接跳到 Part B。

## A1. 什么是 LLM Agent

最朴素的定义：**"一个能调用外部工具的 LLM 对话循环"**。

普通聊天 LLM：

```
你 → 模型 → 回复
```

LLM Agent：

```
你 → 模型 → [模型决定调一个工具] → 工具返回结果 → 模型 → [可能再调工具] → ... → 最终回复
```

关键差别：**模型不只是说话，它可以"行动"**。比如读数据库、发邮件、修改用户密码。

Anthropic 官方 SDK 里这个循环用 `tool_use` 与 `tool_result` 实现，本教程 Day 7 会给完整代码。

## A2. 什么是工具（Tool）

一个工具就是一段你定义的代码（函数或脚本），LLM 通过 JSON Schema 知道它的名字、用途、参数。模型决定调用时，运行环境真正执行那段代码，把结果回填给模型。

工具的 JSON 定义示例：

```json
{
  "name": "get_order",
  "description": "根据订单号查询订单详情",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string", "description": "订单号" }
    },
    "required": ["order_id"]
  }
}
```

模型看到这份描述后，会在合适的时刻输出：

```json
{ "type": "tool_use", "name": "get_order", "input": { "order_id": "o456" } }
```

你的程序拿到后真的去查数据库，把结果再喂回模型。

## A3. 什么是多代理 / 子代理

一个 Agent 拥有的工具越多，它越容易选错工具、上下文越长、推理越慢。**多代理**就是把一个大 Agent 拆成"主代理 + 多个专科子代理"。主代理不持有专科工具，而是通过一个"委派"工具把任务交给子代理处理。

类比：
- 主代理 ≈ 全科医生
- 子代理 ≈ 影像科、化验科、外科

主代理决定"该挂哪个科"，专科代理负责"具体怎么查、怎么做"。

**为什么有效**：
- 工具集物理隔离，安全可控
- 每个子代理上下文窄，质量更高
- 子代理可以用不同模型 / 不同推理力度

## A4. 什么是 MCP（看一眼即可）

**Model Context Protocol**：Anthropic 推的一套标准协议，让"工具供给方"和"Agent 运行方"解耦。你写一个 MCP server 暴露工具，任何兼容 MCP 的 Agent 客户端（Claude Code、Codex 等）都能拿到这些工具。

本教程**不依赖 MCP**，我们直接用 Anthropic SDK 的原生 tool_use 接口，因为对新手更直观。等你做大了再考虑 MCP。

## A5. 什么是"提案 + 人审"模式

**核心矛盾**：LLM 会幻觉，但业务里有些动作（退款、改密码、封号）不可逆。

**解决思路**：LLM 不能直接执行写操作。它只能"提案" — 生成一张待审卡，写明：要做什么、为什么、对谁、用什么参数、外发给用户什么文案。然后**真人**在一个 Web UI 上点 Approve，后端才真正执行下游业务命令。

这一层叫**提案-审批网关（Proposal Gateway）**，是整个系统的"防呆护栏"。哪怕模型疯了乱开提案，最坏结果也只是 Reviewer 否决 N 次而已，永远不会进生产数据库。

## A6. 我们要造的东西到底是什么

一个**"AI 起草 + 人类批准"**的客服 / 运营辅助系统。

适用场景：
- 业务一线每天有几十到几千张工单 / 工作项
- 每张都需要"调查 → 判断 → 执行动作 → 回复用户"
- 调查的证据来自多个结构化数据源（数仓、日志、文档）
- 动作敏感（涉及金钱 / 账号 / 合规），不能让 AI 直接执行

---

<a id="part-b"></a>

# Part B — 系统总览与设计哲学

## B1. 一张图

```
                       ┌──────────────────────────────────┐
                       │     用户提交工单（来自任何渠道）    │
                       └─────────────────┬────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                       Primary Agent（主代理）                       │
│                                                                    │
│   开局必读：POLICY.md  +  DECISION_FRAMEWORK.md  +  PLAYBOOK_INDEX  │
│                                                                    │
│   持有工具：                                                        │
│     - 读：get_ticket / get_user / get_order / get_payment          │
│     - 写：create_refund_proposal / create_reply_proposal / ...     │
│     - 委派：delegate_to_data_investigator / ...                    │
│                                                                    │
│   职责：调查 → 判断 → 起草提案；零下游写权限                          │
└─────┬───────────────────────────────────────────┬─────────────────┘
      │ 委派包                                     │ POST /proposals
      ▼                                           ▼
┌──────────────────────┐                ┌──────────────────────────┐
│ 专科子代理            │                │ Proposal Backend         │
│  (各自独立 agent loop)│                │  Bun + SQLite            │
│                      │                │                          │
│ Data Investigator     │                │  /proposals (POST/GET)    │
│   - SQL 数仓           │                │  /proposals/:id/approve   │
│ Logistics Investigator│                │  /proposals/:id/deny      │
│   - 物流日志           │                │  审计日志                  │
│                      │                └─────────┬────────────────┘
└──────────────────────┘                          │
                                                  ▼
                                       ┌──────────────────────────┐
                                       │  Human Reviewer Web UI   │
                                       │  - 看 reason             │
                                       │  - 编辑 user_message     │
                                       │  - Approve / Deny        │
                                       └─────────┬────────────────┘
                                                 │ Approve
                                                 ▼
                                       ┌──────────────────────────┐
                                       │  Downstream Business     │
                                       │  Gateway                 │
                                       │  - OMS（发退款）           │
                                       │  - 账号系统（改状态）       │
                                       │  - 邮件（发回复）           │
                                       └──────────────────────────┘
```

## B2. 数据流（七个时刻）

1. **接单**：工单系统推送一条新工单到主代理
2. **加载政策**：主代理开局读 3 份 markdown 文档
3. **取证**：主代理调读工具（get_ticket / get_order / ...）拿基础事实
4. **委派**：若需要历史/日志/文档，主代理向相应子代理发"委派包"
5. **起草**：主代理判断结论后，调写工具（create_*_proposal）生成提案卡
6. **人审**：真人在 Web UI 上看 → 编辑 → Approve 或 Deny
7. **执行**：Approve 后，后端调下游网关真正执行；记审计；通知主代理结束本工单

## B3. 三层文档：系统的"DNA"

这一点最容易被忽略也最关键。**先写文档，再写代码**。

| 文档 | 答什么问题 | 谁写 | 改的频率 |
|---|---|---|---|
| `POLICY.md` | "什么绝对不能做" | 合规 / 法务 / 业务负责人 | 半年 |
| `DECISION_FRAMEWORK.md` | "证据不够时怎么判" | 业务负责人 | 季度 |
| `PLAYBOOK_LIBRARY/*.md` | "这类问题该去哪找证据" | 一线运营 | 每周 |

为什么不写在 Prompt 里？三份文档生命周期不同、责任人不同、合规审计要分别 review。混进 Prompt 等于失去工程化能力。

## B4. 四个不变量（设计时必须坚守）

1. **主代理零写权限** — 不可能直接调下游接口，所有写都走"提案 + 人审"
2. **一张提案 = 一次下游调用** — 不允许多步原子化，组合操作开多张卡
3. **payload 在脚本侧算好** — 后端不做模板插值，审稿人看到的就是最终参数
4. **内部 reason 与外部 user_message 严格分离** — 用不同字段名、不同存储

---

<a id="part-c"></a>

# Part C — 选你的领域 / 本教程的示例选型

## C1. 检查你的领域是否适合

逐条回答：

- [ ] 业务里有**不可逆 / 高金额 / 合规敏感**的下游动作？
- [ ] 有**结构化证据源**可被 Agent 读取（数据库、日志、文档）？
- [ ] 团队接受**"AI 起草、人类批准"**这种协作？
- [ ] 工单 / 工作项有**重复模式**，但每张需要判断？
- [ ] 你能为合规留**审计链路**？

≥ 4 个"是" — 适合套用本架构。≤ 2 个 — 别折腾，普通脚本就够。

## C2. 本教程的示例：电商订单退款客服

为了让概念落地，整本教程围绕一个示例展开：

- **业务**：跨境电商，日单量约 3000，工单约 200/天
- **典型工单**：用户 #u789 提交工单 #t12345，说订单 #o456（¥299）"已签收但没收到"
- **下游系统**：OMS（订单/退款）、IAM（账号）、物流系统、邮件平台

**主代理工具**：

| 工具 | 类型 | 用途 |
|---|---|---|
| `get_ticket` | 读 | 取工单详情 |
| `get_user_profile` | 读 | 取用户档案与历史工单 |
| `get_order` | 读 | 取订单详情 |
| `get_payment_records` | 读 | 取该用户近 90 天支付记录 |
| `create_refund_proposal` | 写 | 创建退款提案卡 |
| `create_reply_proposal` | 写 | 创建用户可见回复提案卡 |
| `create_close_ticket_proposal` | 写 | 创建关单提案卡 |
| `create_escalate_proposal` | 写 | 升级到人工经理 |
| `delegate_to_data_investigator` | 委派 | 历史 / 数仓查证 |
| `delegate_to_logistics_investigator` | 委派 | 物流签收查证 |

**子代理**：

| 子代理 | 工具范围 | 解决什么问题 |
|---|---|---|
| `data_investigator` | 数仓 SQL（订单历史、用户历史、退款历史） | 这个用户是否有"频繁退款"特征？这单是否首次申诉？ |
| `logistics_investigator` | 物流 API + 物流日志 | 订单是否真签收？签收人是谁？签收 GPS？ |

## C3. 把示例翻译到你的领域

迁移时只改三件事：

| 系统抽象 | 电商客服 | 套到你的领域 |
|---|---|---|
| 工单 | 客服工单 | 你的"工作项" |
| 主代理工具（读） | get_order / get_user | 你领域的查询 API |
| 主代理工具（写） | create_refund_proposal | 你领域的提案动作 |
| 子代理 | 数仓 / 物流 | 你的证据源 |
| 下游网关 | OMS / 邮件 | 你的核心业务系统 |

**架构本身不变。只换内容。**

---

<a id="part-d"></a>

# Part D — 环境准备

## D1. 安装 Bun

Bun 是 Node.js 的替代品，自带 TypeScript、SQLite、HTTP server，零配置。

Windows：
```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

macOS / Linux：
```bash
curl -fsSL https://bun.sh/install | bash
```

验证：
```bash
bun --version
# 应该输出 1.x.x
```

## D2. 创建项目

```bash
mkdir my-agent && cd my-agent
bun init -y
```

你会得到：
```
my-agent/
├── package.json
├── tsconfig.json
└── index.ts
```

## D3. 安装依赖

```bash
bun add @anthropic-ai/sdk hono
bun add -d @types/bun
```

- `@anthropic-ai/sdk` — Claude API 客户端
- `hono` — 轻量 HTTP 框架（用来写后端）

## D4. Anthropic API Key

1. 去 https://console.anthropic.com 注册 / 登录
2. 进入 "API Keys"，创建一个新 key
3. 充值（最低 $5 即可开发）

把 key 存到 `.env`：

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxx
PROPOSAL_BACKEND_URL=http://localhost:31367
PROPOSAL_BACKEND_TOKEN=dev-token-change-me
```

并在 `.gitignore` 里加上 `.env`。

## D5. 目标文件树

教程结束时，你的项目长这样：

```
my-agent/
├── .env
├── package.json
├── tsconfig.json
├── agent/                        # 主代理 + 子代理
│   ├── policies/
│   │   ├── POLICY.md
│   │   ├── DECISION_FRAMEWORK.md
│   │   └── playbooks/
│   │       ├── INDEX.md
│   │       ├── refund-not-received.md
│   │       ├── refund-defective.md
│   │       └── account-question.md
│   ├── primary-agent.ts          # 主代理循环
│   ├── subagents/
│   │   ├── data-investigator.ts
│   │   └── logistics-investigator.ts
│   └── tools.ts                  # 工具注册中心
├── skills/                       # 业务技能脚本
│   ├── tickets/
│   │   └── scripts/
│   │       └── get-ticket.ts
│   ├── users/
│   │   └── scripts/
│   │       └── get-user-profile.ts
│   ├── orders/
│   │   └── scripts/
│   │       ├── get-order.ts
│   │       └── get-payment-records.ts
│   └── actions/
│       └── scripts/
│           ├── create-refund-proposal.ts
│           ├── create-reply-proposal.ts
│           ├── create-close-ticket-proposal.ts
│           └── create-escalate-proposal.ts
├── lib/                          # 共享 SDK
│   ├── proposal.ts
│   ├── read.ts
│   └── script-runner.ts
├── backend/                      # 提案后端 + UI
│   ├── server.ts
│   ├── db.ts
│   ├── schema.sql
│   └── ui/
│       └── index.html
└── mock-downstream/              # 模拟下游业务系统（开发期用）
    └── server.ts
```

---

<a id="day-1"></a>

# Day 1 — 写三份规则文档（不写代码）

**目标**：把"客服怎么判断"沉淀为可审计的 markdown。
**前置**：和 1-2 位业务专家聊 1 小时，搞清楚红线、判断逻辑、最常见 5 种工单。
**今天不要写代码。**

## 1.1 `agent/policies/POLICY.md`

```markdown
# 政策（绑定底线）

## 0. 真相纪律

向用户发出的每一句具体陈述，必须来自下列认可来源：
- 工单原文 / 用户提交的截图（已实际查阅）
- 数仓查询结果（带时间戳 / 查询 SQL）
- 下游系统当前状态读取
- 物流系统响应
- 公开 FAQ / 退款政策官方页面

**禁止**：把"听起来合理的解释""模型常识"作为事实写进用户回复。

## 1. 退款上限

| 类型 | 金额上限 | 处理 |
|---|---|---|
| 直接退款 | ≤ ¥500 | 单提案即可 |
| 直接退款 | ¥500 ~ ¥2000 | 单提案 + reason 中必须说明证据等级 |
| 直接退款 | > ¥2000 | 强制升级，禁止直接退款提案 |
| 全单补发 | ≤ 1 件 | 单提案即可 |
| 全单补发 | > 1 件 | 强制升级 |

## 2. 欺诈识别红线

下列任一情况出现，主代理必须**升级**，不允许直接退款：

- 同一用户 90 天内退款次数 ≥ 3 次
- 同一用户 90 天内"未收到"申诉 ≥ 2 次
- 物流签收 GPS 与下单地址距离 ≤ 100 米但用户声称未收到
- 单价 > ¥1000 的高价值商品 + 首次申诉

## 3. 账号 / 安全

- 用户身份信息（手机、邮箱、地址）变更必须由用户自己操作
- 客服**不允许**代为修改任何账号字段
- 涉及账号冻结 / 解冻 — 必须升级

## 4. 用户回复纪律

- 回复必须以「您好，感谢联系 [品牌] 客服。」开头
- 回复必须以「祝您购物愉快，[品牌] 客服团队」结尾
- 禁止在回复中提及：内部判断词（"已核实欺诈"）、内部系统名（"OMS 查无此单"）、其他用户信息

## 5. 善意补偿

允许在以下条件**全部满足**时给低风险补偿（≤ ¥100 优惠券或 ≤ ¥50 退款）：
- 用户诉求具体且合理
- 证据等级为"converging"或"mixed but plausible"
- 用户 90 天内无任何补偿记录
- 客服关系建议给（账号活跃、长期客户）

善意补偿的真正原因写在内部 `reason`，**不进** `user_message`。
```

## 1.2 `agent/policies/DECISION_FRAMEWORK.md`

```markdown
# 决策框架

## 真相门

每张工单开局，先把信息分成四桶：

| 桶 | 定义 | 处理 |
|---|---|---|
| Claim | 用户陈述 | 标记为待验证，不当事实 |
| Verified fact | 认可来源已确认 | 可用于回复 |
| Inference | 从已验证事实做出的判断 | 仅用于内部 reason，不能写进用户回复当事实 |
| Gap | 关键未知 | 决定是否升级 |

## 证据强度梯

| 等级 | 定义 | 允许的动作 |
|---|---|---|
| **direct proof** | 关键事实有直接证据 | 直接创建提案 |
| **converging** | 多个间接证据指向同一结论，无重大冲突 | 中低风险动作；高风险升级 |
| **mixed but plausible** | 证据混合，但合理 | 仅允许低风险善意补偿 |
| **scoped negative** | 在正确的范围/时间/标识下未发现证据 | 政策性拒绝 |
| **load-bearing gap** | 决定性证据缺失 | 升级或挂起 |
| **user-only assertion** | 仅有用户陈述 | 不允许动作 |

## 数据源优先级

| 问题类型 | 第一证据源 |
|---|---|
| 历史模式 / 重复申诉 / 退款记录 | 数仓（→ data_investigator） |
| 物流签收 / 异常 | 物流系统（→ logistics_investigator） |
| 用户当前账号状态 | IAM 直读 |
| 订单当前状态 | OMS 直读 |
| 退款政策 / FAQ | 公开文档 |

## 镜像问题（提案前的自检）

提案创建前，主代理必须能在内部对自己回答：

1. 公平所需的事实是什么？
2. 哪些记录证明 / 削弱 / 无法证明？
3. 缺失的证据是否真的"载荷关键"？
4. 动作和风险 / 不可逆性是否成正比？
5. **换一个同样情况的用户，会得到同样的结果吗？**
6. Reviewer 只读 reason 就能批准吗？

## 善意补偿规则

低风险善意补偿 = **服务恢复**，不是事实证明。仅在：
- 金额 ≤ 政策上限
- 证据为 mixed but plausible
- 用户 90 天内无补偿记录
- 关系上合理

时允许。**理由写内部 reason，不写 user_message**。
```

## 1.3 `agent/policies/playbooks/INDEX.md`

```markdown
# 剧本索引

主代理根据工单关键词路由到具体剧本。优先级从高到低：

1. **欺诈嫌疑**（user_history.refunds_last_90d >= 3）→ `escalation-fraud.md`
2. **物流异常**（关键词：未收到 / 已签收 / 丢件）→ `refund-not-received.md`
3. **商品瑕疵**（关键词：损坏 / 破损 / 与描述不符）→ `refund-defective.md`
4. **账号疑问**（关键词：登录 / 密码 / 改信息）→ `account-question.md`
5. **其他** → `general-inquiry.md`

## 路由优先级规则

工单同时命中多个关键词时按上述顺序，先匹配的优先。
```

## 1.4 `agent/policies/playbooks/refund-not-received.md`

```markdown
# 剧本：物流类退款（"未收到"）

## Use When

用户工单关键词包含："未收到 / 没收到 / 已签收但没拿到 / 丢件 / 包裹丢失"

## Core Evidence（必须按顺序取）

1. `get_ticket` — 工单原文 / 用户附件（拍照？签收回执？）
2. `get_order` — 订单当前状态 / 物流单号 / 收货地址
3. `get_user_profile` — 用户档案 / 历史工单
4. `delegate_to_logistics_investigator` — 物流签收记录 / 签收 GPS / 签收人姓名
5. **若** `user.refunds_last_90d >= 2` **则** `delegate_to_data_investigator` — 该用户历史"未收到"申诉模式

## Decision Tree

| 情况 | 证据强度 | 处理 |
|---|---|---|
| 物流显示未签收 | direct proof | 直接退款提案 + 回复提案 |
| 物流签收但 GPS 异常（距离地址 > 1km） | converging | 退款提案 + 回复提案 |
| 物流签收 + GPS 在地址附近 + 首次申诉 + 单价 ≤ ¥500 | mixed but plausible | 善意补偿（半额优惠券）+ 回复提案 |
| 物流签收 + GPS 正常 + 用户 90 天内 ≥ 2 次同类申诉 | scoped negative | 政策性拒绝 + 升级标记 |
| 单价 > ¥2000 | 任何 | **强制升级**，不允许直接退款 |

## Resolution Paths

- **退款 + 通知**：开两张卡 — `create_refund_proposal` + `create_reply_proposal`
- **善意补偿**：`create_refund_proposal(amount=半额)` + `create_reply_proposal`
- **政策拒绝**：`create_reply_proposal` + `create_close_ticket_proposal`
- **升级**：`create_escalate_proposal(reason="...")`

## Escalate When

- 金额 > ¥2000
- 用户 90 天内"未收到"申诉 ≥ 2 次
- 物流系统返回错误 / 不可用
- 用户措辞威胁性 / 法律性
```

## 1.5 完成判定

- [ ] 三份文件都写好，并且**业务专家盲读后认可**判断逻辑落得到这里
- [ ] 至少 3 份具体剧本（建议最终 ≥ 20 份）
- [ ] 文件没有"如果情况X 则做Y"硬编码 — 都是"信号 + 证据指引 + 选项"

> **常见错误**：把决策框架写成大 if-else 决策树。Agent 会卡死在没覆盖的边角情况。要写成"证据 → 等级 → 选项"，让 Agent 自己判断。

---

<a id="day-2"></a>

# Day 2 — 搭提案后端 + SQLite

**目标**：跑通 `POST /proposals` → 入库 → `GET /proposals` → `POST /approve` → mock 下游被调用。
**前置**：D 节准备好；项目骨架已建。

## 2.1 数据库 schema

`backend/schema.sql`：

```sql
CREATE TABLE IF NOT EXISTS proposals (
    id TEXT PRIMARY KEY,
    work_item_id TEXT NOT NULL,
    actor_id TEXT NOT NULL,
    operation TEXT NOT NULL,
    params TEXT NOT NULL,           -- JSON
    reason TEXT NOT NULL,
    user_message TEXT,
    execution TEXT NOT NULL,        -- JSON: { command, parameters, audit_summary }
    status TEXT NOT NULL,           -- pending / approved / denied / executed / failed
    created_at INTEGER NOT NULL,
    created_by TEXT NOT NULL,       -- 哪个 agent / 哪个脚本创建
    decided_at INTEGER,
    decided_by TEXT,                -- 哪个 reviewer 处理
    executed_at INTEGER,
    downstream_response TEXT        -- JSON
);

CREATE INDEX IF NOT EXISTS idx_proposals_status ON proposals(status);
CREATE INDEX IF NOT EXISTS idx_proposals_work_item ON proposals(work_item_id);

CREATE TABLE IF NOT EXISTS audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    proposal_id TEXT NOT NULL,
    event TEXT NOT NULL,            -- created / approved / denied / executed / failed
    actor TEXT NOT NULL,            -- agent name 或 reviewer name
    details TEXT,                   -- JSON
    timestamp INTEGER NOT NULL
);
```

## 2.2 数据库初始化

`backend/db.ts`：

```typescript
import { Database } from "bun:sqlite"
import { readFileSync } from "fs"
import { join } from "path"

const DB_PATH = process.env.DB_PATH ?? "./proposals.db"

export const db = new Database(DB_PATH)
db.exec(readFileSync(join(__dirname, "schema.sql"), "utf-8"))

export interface ProposalRow {
  id: string
  work_item_id: string
  actor_id: string
  operation: string
  params: string
  reason: string
  user_message: string | null
  execution: string
  status: string
  created_at: number
  created_by: string
  decided_at: number | null
  decided_by: string | null
  executed_at: number | null
  downstream_response: string | null
}

export function insertProposal(p: ProposalRow) {
  db.prepare(`
    INSERT INTO proposals (id, work_item_id, actor_id, operation, params,
      reason, user_message, execution, status, created_at, created_by)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
  `).run(p.id, p.work_item_id, p.actor_id, p.operation, p.params,
    p.reason, p.user_message, p.execution, p.status, p.created_at, p.created_by)
  logAudit(p.id, "created", p.created_by, { operation: p.operation })
}

export function getProposal(id: string): ProposalRow | null {
  return db.prepare("SELECT * FROM proposals WHERE id = ?").get(id) as ProposalRow | null
}

export function listProposals(status?: string): ProposalRow[] {
  if (status) return db.prepare("SELECT * FROM proposals WHERE status = ? ORDER BY created_at DESC").all(status) as ProposalRow[]
  return db.prepare("SELECT * FROM proposals ORDER BY created_at DESC LIMIT 100").all() as ProposalRow[]
}

export function updateProposalStatus(id: string, status: string, decidedBy: string, extras: Partial<ProposalRow> = {}) {
  const now = Date.now()
  db.prepare(`
    UPDATE proposals
    SET status = ?, decided_at = ?, decided_by = ?, user_message = COALESCE(?, user_message)
    WHERE id = ?
  `).run(status, now, decidedBy, extras.user_message ?? null, id)
  logAudit(id, status, decidedBy, extras)
}

export function markExecuted(id: string, response: unknown) {
  const now = Date.now()
  db.prepare(`
    UPDATE proposals
    SET status = 'executed', executed_at = ?, downstream_response = ?
    WHERE id = ?
  `).run(now, JSON.stringify(response), id)
  logAudit(id, "executed", "system", response)
}

export function markFailed(id: string, error: string) {
  const now = Date.now()
  db.prepare(`
    UPDATE proposals
    SET status = 'failed', executed_at = ?, downstream_response = ?
    WHERE id = ?
  `).run(now, JSON.stringify({ error }), id)
  logAudit(id, "failed", "system", { error })
}

export function logAudit(proposalId: string, event: string, actor: string, details: unknown) {
  db.prepare(`
    INSERT INTO audit_log (proposal_id, event, actor, details, timestamp)
    VALUES (?, ?, ?, ?, ?)
  `).run(proposalId, event, actor, JSON.stringify(details), Date.now())
}
```

## 2.3 HTTP 服务

`backend/server.ts`：

```typescript
import { Hono } from "hono"
import { cors } from "hono/cors"
import { serveStatic } from "hono/bun"
import { randomUUID } from "crypto"
import {
  insertProposal, getProposal, listProposals,
  updateProposalStatus, markExecuted, markFailed,
} from "./db"

const TOKEN = process.env.PROPOSAL_BACKEND_TOKEN ?? "dev-token-change-me"
const DOWNSTREAM = process.env.DOWNSTREAM_URL ?? "http://localhost:31368"
const ALLOWED_COMMANDS = new Set([
  "OMS.Refund",
  "OMS.Reship",
  "Ticket.Reply",
  "Ticket.Close",
  "Ticket.Escalate",
])

const app = new Hono()
app.use("*", cors())

// 中间件：鉴权
app.use("/proposals/*", async (c, next) => {
  const token = c.req.header("x-agent-token")
  if (token !== TOKEN) return c.json({ error: "unauthorized" }, 401)
  await next()
})

// 创建提案
app.post("/proposals", async (c) => {
  const body = await c.req.json()

  // 校验
  const required = ["work_item_id", "actor_id", "operation", "params", "reason", "execution"]
  for (const k of required) {
    if (body[k] === undefined) return c.json({ error: `missing field: ${k}` }, 400)
  }
  if (!Array.isArray(body.execution.steps) || body.execution.steps.length !== 1) {
    return c.json({ error: "execution.steps must contain exactly 1 step" }, 400)
  }
  const command = body.execution.steps[0].command
  if (!ALLOWED_COMMANDS.has(command)) {
    return c.json({ error: `command not allowed: ${command}` }, 400)
  }

  const id = randomUUID()
  insertProposal({
    id,
    work_item_id: String(body.work_item_id),
    actor_id: String(body.actor_id),
    operation: body.operation,
    params: JSON.stringify(body.params),
    reason: body.reason,
    user_message: body.user_message ?? null,
    execution: JSON.stringify(body.execution),
    status: "pending",
    created_at: Date.now(),
    created_by: body.created_by ?? "unknown-agent",
    decided_at: null,
    decided_by: null,
    executed_at: null,
    downstream_response: null,
  })

  return c.json({ proposal: { id, status: "pending" } })
})

// 列出提案
app.get("/proposals", (c) => {
  const status = c.req.query("status")
  const rows = listProposals(status)
  return c.json({ proposals: rows })
})

// 拿单张
app.get("/proposals/:id", (c) => {
  const p = getProposal(c.req.param("id"))
  if (!p) return c.json({ error: "not found" }, 404)
  return c.json({ proposal: p })
})

// 批准
app.post("/proposals/:id/approve", async (c) => {
  const body = await c.req.json().catch(() => ({}))
  const id = c.req.param("id")
  const p = getProposal(id)
  if (!p) return c.json({ error: "not found" }, 404)
  if (p.status !== "pending") return c.json({ error: `cannot approve from status ${p.status}` }, 400)

  const finalUserMessage = body.user_message ?? p.user_message

  updateProposalStatus(id, "approved", body.reviewer ?? "anonymous", { user_message: finalUserMessage })

  // 调下游
  const execution = JSON.parse(p.execution)
  const step = execution.steps[0]
  // 如果是回复类提案，把 user_message 注入下游参数
  if (step.substitute_user_message_as) {
    step.parameters[step.substitute_user_message_as] = finalUserMessage
  }

  try {
    const downstreamResp = await fetch(`${DOWNSTREAM}/execute`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ command: step.command, parameters: step.parameters }),
    })
    const respJson = await downstreamResp.json()
    if (!downstreamResp.ok) {
      markFailed(id, JSON.stringify(respJson))
      return c.json({ status: "approved", execution: "failed", response: respJson }, 502)
    }
    markExecuted(id, respJson)
    return c.json({ status: "executed", response: respJson })
  } catch (e: any) {
    markFailed(id, e.message)
    return c.json({ status: "approved", execution: "error", error: e.message }, 502)
  }
})

// 否决
app.post("/proposals/:id/deny", async (c) => {
  const body = await c.req.json().catch(() => ({}))
  const id = c.req.param("id")
  const p = getProposal(id)
  if (!p) return c.json({ error: "not found" }, 404)
  if (p.status !== "pending") return c.json({ error: `cannot deny from status ${p.status}` }, 400)
  updateProposalStatus(id, "denied", body.reviewer ?? "anonymous", { user_message: body.reason })
  return c.json({ status: "denied" })
})

// 静态 UI
app.use("/ui/*", serveStatic({ root: "./backend" }))
app.get("/", (c) => c.redirect("/ui/index.html"))

const PORT = Number(process.env.PORT ?? 31367)
export default { port: PORT, fetch: app.fetch }
console.log(`✓ Proposal backend listening on http://localhost:${PORT}`)
```

## 2.4 Mock 下游

`mock-downstream/server.ts`：

```typescript
import { Hono } from "hono"

const app = new Hono()

app.post("/execute", async (c) => {
  const body = await c.req.json()
  console.log(`[MOCK DOWNSTREAM] command=${body.command}`, body.parameters)
  // 模拟一个简单的"全部成功"
  return c.json({ ok: true, command: body.command, processed_at: new Date().toISOString() })
})

export default { port: 31368, fetch: app.fetch }
console.log("✓ Mock downstream listening on http://localhost:31368")
```

## 2.5 启动 + 验证

两个终端：

```bash
# Terminal 1
bun run backend/server.ts

# Terminal 2
bun run mock-downstream/server.ts
```

测试创建一个提案：

```bash
curl -X POST http://localhost:31367/proposals \
  -H "x-agent-token: dev-token-change-me" \
  -H "content-type: application/json" \
  -d '{
    "work_item_id": "t12345",
    "actor_id": "u789",
    "operation": "Refund",
    "params": { "order_id": "o456", "amount_cents": 29900 },
    "reason": "测试创建一张退款卡",
    "user_message": "您好，您的退款已处理。",
    "execution": {
      "steps": [{
        "command": "OMS.Refund",
        "parameters": { "order_id": "o456", "amount_cents": 29900 }
      }],
      "audit_summary": "Refund order o456 for ¥299"
    },
    "created_by": "manual-curl-test"
  }'
```

期待响应：

```json
{ "proposal": { "id": "uuid-xxx", "status": "pending" } }
```

记下 id，调用 approve：

```bash
curl -X POST http://localhost:31367/proposals/<id>/approve \
  -H "x-agent-token: dev-token-change-me" \
  -H "content-type: application/json" \
  -d '{ "reviewer": "test-reviewer" }'
```

期待响应：

```json
{ "status": "executed", "response": { "ok": true, ... } }
```

Mock 下游终端会打印一行：

```
[MOCK DOWNSTREAM] command=OMS.Refund { order_id: 'o456', amount_cents: 29900 }
```

## 2.6 完成判定

- [ ] `POST /proposals` 成功返回 id
- [ ] `GET /proposals/:id` 返回完整记录
- [ ] `POST /proposals/:id/approve` 让 mock 下游收到调用
- [ ] 多步提案（`steps.length !== 1`）会被拒绝
- [ ] 不在 allowlist 的 command 会被拒绝
- [ ] 不带 `x-agent-token` 的请求会被 401

---

<a id="day-3"></a>

# Day 3 — 搭人审 UI

**目标**：一个单页应用，列出 pending 提案、能编辑 user_message、能点 Approve / Deny。
**前置**：Day 2 后端能跑。

## 3.1 完整 HTML（无框架，原生 + Alpine.js）

`backend/ui/index.html`：

```html
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="utf-8">
<title>提案审批控制台</title>
<style>
  body { font-family: -apple-system, sans-serif; max-width: 1200px; margin: 0 auto; padding: 20px; }
  h1 { font-size: 20px; }
  .card { border: 1px solid #ddd; border-radius: 8px; padding: 16px; margin: 12px 0; background: #fff; }
  .card.pending { border-left: 4px solid #f5a623; }
  .card.executed { border-left: 4px solid #2c8c2c; opacity: 0.6; }
  .card.denied { border-left: 4px solid #999; opacity: 0.5; }
  .row { display: flex; gap: 12px; margin: 6px 0; }
  .label { color: #666; min-width: 110px; font-size: 13px; }
  .value { flex: 1; font-family: SFMono-Regular, monospace; font-size: 13px; }
  pre { background: #f6f6f6; padding: 10px; border-radius: 4px; overflow-x: auto; }
  textarea { width: 100%; min-height: 80px; font-family: inherit; padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
  button { padding: 8px 16px; border-radius: 4px; border: none; cursor: pointer; font-size: 14px; }
  .approve { background: #2c8c2c; color: white; margin-right: 8px; }
  .deny { background: #c33; color: white; }
  .filter { margin: 12px 0; }
  .badge { display: inline-block; padding: 2px 8px; border-radius: 10px; font-size: 12px; background: #eef; }
</style>
</head>
<body x-data="app()" x-init="load()">

<h1>提案审批控制台</h1>

<div class="filter">
  状态：
  <button @click="filter='pending'; load()">Pending</button>
  <button @click="filter='approved'; load()">Approved</button>
  <button @click="filter='executed'; load()">Executed</button>
  <button @click="filter='denied'; load()">Denied</button>
  <button @click="filter=''; load()">All</button>
  <button @click="load()">刷新</button>
</div>

<template x-for="p in proposals" :key="p.id">
  <div class="card" :class="p.status">
    <div class="row">
      <span class="label">ID</span>
      <span class="value" x-text="p.id"></span>
      <span class="badge" x-text="p.status"></span>
    </div>
    <div class="row"><span class="label">Work Item</span><span class="value" x-text="p.work_item_id"></span></div>
    <div class="row"><span class="label">Actor</span><span class="value" x-text="p.actor_id"></span></div>
    <div class="row"><span class="label">Operation</span><span class="value" x-text="p.operation"></span></div>
    <div class="row"><span class="label">Created By</span><span class="value" x-text="p.created_by"></span></div>
    <div class="row"><span class="label">Reason (内部)</span><span class="value" style="white-space:pre-wrap" x-text="p.reason"></span></div>
    <div class="row"><span class="label">Params</span><pre x-text="prettyJson(p.params)"></pre></div>
    <div class="row"><span class="label">Execution</span><pre x-text="prettyJson(p.execution)"></pre></div>

    <template x-if="p.status === 'pending'">
      <div>
        <div class="row" style="flex-direction: column">
          <span class="label">User Message (用户可见，可编辑)</span>
          <textarea x-model="p._editMessage"></textarea>
        </div>
        <div class="row">
          <button class="approve" @click="approve(p)">Approve</button>
          <button class="deny" @click="deny(p)">Deny</button>
        </div>
      </div>
    </template>
    <template x-if="p.status !== 'pending'">
      <div class="row"><span class="label">User Message</span><span class="value" style="white-space:pre-wrap" x-text="p.user_message"></span></div>
    </template>
    <template x-if="p.downstream_response">
      <div class="row"><span class="label">Downstream</span><pre x-text="prettyJson(p.downstream_response)"></pre></div>
    </template>
  </div>
</template>

<script src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js" defer></script>
<script>
const TOKEN = "dev-token-change-me"  // 真上线请改方案
const BASE = ""

function app() {
  return {
    proposals: [],
    filter: "pending",

    async load() {
      const q = this.filter ? `?status=${this.filter}` : ""
      const resp = await fetch(`${BASE}/proposals${q}`, { headers: { "x-agent-token": TOKEN } })
      const data = await resp.json()
      this.proposals = data.proposals.map(p => ({ ...p, _editMessage: p.user_message ?? "" }))
    },

    prettyJson(s) {
      try { return JSON.stringify(JSON.parse(s), null, 2) } catch { return s }
    },

    async approve(p) {
      if (!confirm("确认 Approve 这张提案？批准后会立即调用下游业务系统。")) return
      const resp = await fetch(`${BASE}/proposals/${p.id}/approve`, {
        method: "POST",
        headers: { "x-agent-token": TOKEN, "content-type": "application/json" },
        body: JSON.stringify({ reviewer: "manual-reviewer", user_message: p._editMessage }),
      })
      const data = await resp.json()
      alert("结果：" + JSON.stringify(data, null, 2))
      this.load()
    },

    async deny(p) {
      const reason = prompt("否决理由（会记入审计）：")
      if (!reason) return
      await fetch(`${BASE}/proposals/${p.id}/deny`, {
        method: "POST",
        headers: { "x-agent-token": TOKEN, "content-type": "application/json" },
        body: JSON.stringify({ reviewer: "manual-reviewer", reason }),
      })
      this.load()
    },
  }
}
</script>
</body>
</html>
```

## 3.2 启动 + 验证

后端已经在 Day 2 跑了。浏览器打开 http://localhost:31367/ui/index.html

按 Day 2 用 curl 创建一张提案。刷新页面，看到 pending 卡片。点 Approve，期待：
1. 弹出"结果"对话框
2. 卡片变为 executed 状态
3. Mock 下游终端打印调用日志

## 3.3 完成判定

- [ ] 浏览器能看到 pending 卡片
- [ ] 编辑 user_message → 点 Approve → 下游收到的参数中 user_message 是编辑后的版本
- [ ] Deny 后卡片消失（pending 列表）
- [ ] 多个状态过滤切换正常

---

<a id="day-4"></a>

# Day 4 — 写共享 SDK（脚本运行器）

**目标**：让以后写读 / 写脚本时，重复样板代码全部消失。
**前置**：Day 2 后端能跑。

## 4.1 `lib/proposal.ts`

```typescript
const BACKEND = process.env.PROPOSAL_BACKEND_URL ?? "http://localhost:31367"
const TOKEN = process.env.PROPOSAL_BACKEND_TOKEN ?? "dev-token-change-me"

export interface ProposalStep {
  command: string
  parameters: Record<string, unknown>
  failure_message?: string
  substitute_user_message_as?: string  // 让审稿人编辑后的 user_message 写入 parameters 的哪个键
}

export interface ProposalExecution {
  steps: ProposalStep[]               // 必须长度 = 1
  audit_summary?: string
}

export interface ProposalInput {
  work_item_id: string
  actor_id: string
  operation: string
  params: Record<string, unknown>
  reason: string
  user_message?: string
  execution: ProposalExecution
  created_by: string
}

export function singleStepExecution(
  command: string,
  parameters: Record<string, unknown>,
  audit_summary: string,
): ProposalExecution {
  return { steps: [{ command, parameters }], audit_summary }
}

export function replyExecution(
  command: string,
  baseParams: Record<string, unknown>,
  audit_summary: string,
): ProposalExecution {
  return {
    steps: [{
      command,
      parameters: baseParams,
      substitute_user_message_as: "message",
    }],
    audit_summary,
  }
}

export async function createProposal(input: ProposalInput): Promise<{ id: string; status: string }> {
  const resp = await fetch(`${BACKEND}/proposals`, {
    method: "POST",
    headers: { "x-agent-token": TOKEN, "content-type": "application/json" },
    body: JSON.stringify(input),
  })
  if (!resp.ok) {
    const text = await resp.text()
    throw new Error(`proposal backend ${resp.status}: ${text}`)
  }
  const data = await resp.json() as { proposal: { id: string; status: string } }
  return data.proposal
}

export function requireString(value: string | undefined, name: string): string {
  if (!value) throw new Error(`missing required argument: ${name}`)
  return value
}

export function requireNumber(value: number | undefined, name: string): number {
  if (value === undefined || Number.isNaN(value)) throw new Error(`missing required argument: ${name}`)
  return value
}
```

## 4.2 `lib/read.ts`

```typescript
const MOCK_DB_URL = process.env.MOCK_DB_URL ?? "http://localhost:31369"

export async function readGet(path: string): Promise<unknown> {
  const resp = await fetch(`${MOCK_DB_URL}${path}`)
  if (!resp.ok) {
    const text = await resp.text()
    throw new Error(`read ${path} ${resp.status}: ${text}`)
  }
  return resp.json()
}
```

> 注：真实项目里你会改成调你的 OMS / IAM / 数据库。本教程为了可跑，下面会写一个 mock 数据库 server。

## 4.3 Mock 业务数据库

`mock-downstream/data-server.ts`（与 mock-downstream 复用进程，独立端口）：

```typescript
import { Hono } from "hono"

const app = new Hono()

const tickets: Record<string, any> = {
  "t12345": {
    id: "t12345", user_id: "u789", order_id: "o456",
    title: "已签收但没收到",
    body: "您好，我的订单显示已签收，但我并没有收到包裹。物流单号是 SF1234567890。",
    created_at: "2026-05-10T03:21:00Z",
    attachments: [],
  },
}

const users: Record<string, any> = {
  "u789": {
    id: "u789", email: "user@example.com", joined_at: "2024-01-15",
    lifetime_orders: 23, lifetime_spend_cents: 458900,
    refunds_last_90d: 0, not_received_claims_last_90d: 0,
  },
}

const orders: Record<string, any> = {
  "o456": {
    id: "o456", user_id: "u789", status: "delivered",
    amount_cents: 29900, items: [{ sku: "A1", qty: 1, name: "无线耳机" }],
    shipping_address: "上海市浦东新区XX路1号",
    tracking_no: "SF1234567890",
    delivered_at: "2026-05-08T11:42:00Z",
  },
}

const payments = {
  "u789": [
    { order_id: "o456", amount_cents: 29900, status: "captured", at: "2026-05-05" },
  ],
}

const logistics = {
  "SF1234567890": {
    tracking_no: "SF1234567890",
    events: [
      { at: "2026-05-05T10:00:00Z", event: "pickup" },
      { at: "2026-05-06T09:30:00Z", event: "in_transit" },
      { at: "2026-05-08T11:42:00Z", event: "delivered", signed_by: "前台代收", gps: { lat: 31.2304, lng: 121.4737, distance_to_address_m: 35 } },
    ],
  },
}

app.get("/tickets/:id", (c) => c.json(tickets[c.req.param("id")] ?? null))
app.get("/users/:id", (c) => c.json(users[c.req.param("id")] ?? null))
app.get("/orders/:id", (c) => c.json(orders[c.req.param("id")] ?? null))
app.get("/users/:id/payments", (c) => c.json(payments[c.req.param("id") as keyof typeof payments] ?? []))
app.get("/logistics/:tracking_no", (c) => c.json(logistics[c.req.param("tracking_no") as keyof typeof logistics] ?? null))

export default { port: 31369, fetch: app.fetch }
console.log("✓ Mock data server on http://localhost:31369")
```

启动它：

```bash
bun run mock-downstream/data-server.ts
```

## 4.4 `lib/script-runner.ts`

```typescript
export interface ScriptResult {
  status: "success" | "error"
  data?: unknown
  error?: string
}

export function emitSuccess(data: unknown): never {
  console.log(JSON.stringify({ status: "success", data }))
  process.exit(0)
}

export function emitError(message: string): never {
  console.error(JSON.stringify({ status: "error", error: message }))
  process.exit(1)
}

export function parseFlags(args: string[]): Record<string, string | boolean> {
  const out: Record<string, string | boolean> = {}
  for (let i = 0; i < args.length; i++) {
    const a = args[i]
    if (!a.startsWith("--")) continue
    const key = a.slice(2)
    const next = args[i + 1]
    if (!next || next.startsWith("--")) {
      out[key] = true
    } else {
      out[key] = next
      i++
    }
  }
  return out
}
```

## 4.5 完成判定

- [ ] mock data server 在 31369 端口跑通
- [ ] curl `http://localhost:31369/tickets/t12345` 返回工单 JSON
- [ ] 三个 lib 文件 typecheck 通过：`bun build lib/proposal.ts lib/read.ts lib/script-runner.ts`

---

<a id="day-5"></a>

# Day 5 — 第一个只读技能

**目标**：用 SDK 写 4 个读脚本，每个都是独立可调用的 CLI。
**前置**：Day 4 完成。

## 5.1 `skills/tickets/scripts/get-ticket.ts`

```typescript
import { readGet } from "../../../lib/read"
import { parseFlags, emitSuccess, emitError } from "../../../lib/script-runner"

async function main() {
  const flags = parseFlags(process.argv.slice(2))
  const ticketId = flags["ticket-id"] as string
  if (!ticketId) emitError("missing --ticket-id")

  try {
    const ticket = await readGet(`/tickets/${ticketId}`)
    if (!ticket) emitError(`ticket ${ticketId} not found`)
    emitSuccess(ticket)
  } catch (e: any) {
    emitError(e.message)
  }
}

main()
```

测试：

```bash
bun run skills/tickets/scripts/get-ticket.ts --ticket-id t12345
```

期待：

```json
{ "status": "success", "data": { "id": "t12345", ... } }
```

## 5.2 `skills/users/scripts/get-user-profile.ts`

```typescript
import { readGet } from "../../../lib/read"
import { parseFlags, emitSuccess, emitError } from "../../../lib/script-runner"

async function main() {
  const flags = parseFlags(process.argv.slice(2))
  const userId = flags["user-id"] as string
  if (!userId) emitError("missing --user-id")

  try {
    const profile = await readGet(`/users/${userId}`)
    if (!profile) emitError(`user ${userId} not found`)
    emitSuccess(profile)
  } catch (e: any) {
    emitError(e.message)
  }
}

main()
```

## 5.3 `skills/orders/scripts/get-order.ts`

```typescript
import { readGet } from "../../../lib/read"
import { parseFlags, emitSuccess, emitError } from "../../../lib/script-runner"

async function main() {
  const flags = parseFlags(process.argv.slice(2))
  const orderId = flags["order-id"] as string
  if (!orderId) emitError("missing --order-id")
  try {
    const order = await readGet(`/orders/${orderId}`)
    if (!order) emitError(`order ${orderId} not found`)
    emitSuccess(order)
  } catch (e: any) { emitError(e.message) }
}
main()
```

## 5.4 `skills/orders/scripts/get-payment-records.ts`

```typescript
import { readGet } from "../../../lib/read"
import { parseFlags, emitSuccess, emitError } from "../../../lib/script-runner"

async function main() {
  const flags = parseFlags(process.argv.slice(2))
  const userId = flags["user-id"] as string
  if (!userId) emitError("missing --user-id")
  try {
    const records = await readGet(`/users/${userId}/payments`)
    emitSuccess(records)
  } catch (e: any) { emitError(e.message) }
}
main()
```

## 5.5 完成判定

- [ ] 4 个脚本独立运行都返回 `{"status":"success",...}`
- [ ] 漏 flag 时返回 `{"status":"error",...}` 且 exit code = 1
- [ ] 不存在的 ID 返回 `{"status":"error",...}`

---

<a id="day-6"></a>

# Day 6 — 第一个写技能（提案）

**目标**：写 4 个提案脚本，每个都支持 `--dry-run`。
**前置**：Day 4 完成 + Day 2 后端在跑。

## 6.1 `skills/actions/scripts/create-refund-proposal.ts`

```typescript
import {
  createProposal, singleStepExecution, requireString, requireNumber,
} from "../../../lib/proposal"
import { parseFlags, emitSuccess, emitError } from "../../../lib/script-runner"

async function main() {
  const flags = parseFlags(process.argv.slice(2))
  const ticketId = flags["ticket-id"] as string
  const userId = flags["user-id"] as string
  const orderId = flags["order-id"] as string
  const amountCents = Number(flags["amount-cents"])
  const reason = flags["reason"] as string
  const dryRun = flags["dry-run"] === true

  try {
    requireString(ticketId, "--ticket-id")
    requireString(userId, "--user-id")
    requireString(orderId, "--order-id")
    requireNumber(amountCents, "--amount-cents")
    requireString(reason, "--reason")
  } catch (e: any) { emitError(e.message) }

  const payload = {
    work_item_id: ticketId,
    actor_id: userId,
    operation: "Refund",
    params: { order_id: orderId, amount_cents: amountCents },
    reason,
    execution: singleStepExecution(
      "OMS.Refund",
      { order_id: orderId, amount_cents: amountCents },
      `Refund order ${orderId} for ${amountCents/100} CNY`,
    ),
    created_by: "skill:create-refund-proposal",
  }

  if (dryRun) emitSuccess({ dry_run: true, payload })

  try {
    const proposal = await createProposal(payload)
    emitSuccess({ created: true, proposal })
  } catch (e: any) { emitError(e.message) }
}

main()
```

测试 dry-run：

```bash
bun run skills/actions/scripts/create-refund-proposal.ts \
  --ticket-id t12345 --user-id u789 --order-id o456 \
  --amount-cents 29900 --reason "物流签收异常 GPS 距地址 1.2km" \
  --dry-run
```

期待：完整的 payload 预览，但没有真的创建。

去掉 `--dry-run` 再跑一次，去 UI 看到 pending 卡片。

## 6.2 `skills/actions/scripts/create-reply-proposal.ts`

```typescript
import { createProposal, replyExecution, requireString } from "../../../lib/proposal"
import { parseFlags, emitSuccess, emitError } from "../../../lib/script-runner"

async function main() {
  const flags = parseFlags(process.argv.slice(2))
  const ticketId = flags["ticket-id"] as string
  const userId = flags["user-id"] as string
  const userMessage = flags["user-message"] as string
  const reason = flags["reason"] as string
  const dryRun = flags["dry-run"] === true

  try {
    requireString(ticketId, "--ticket-id")
    requireString(userId, "--user-id")
    requireString(userMessage, "--user-message")
    requireString(reason, "--reason")
  } catch (e: any) { emitError(e.message) }

  const payload = {
    work_item_id: ticketId,
    actor_id: userId,
    operation: "Reply",
    params: { ticket_id: ticketId },
    reason,
    user_message: userMessage,
    execution: replyExecution(
      "Ticket.Reply",
      { ticket_id: ticketId },
      `Reply to ticket ${ticketId}`,
    ),
    created_by: "skill:create-reply-proposal",
  }

  if (dryRun) emitSuccess({ dry_run: true, payload })
  try {
    const proposal = await createProposal(payload)
    emitSuccess({ created: true, proposal })
  } catch (e: any) { emitError(e.message) }
}
main()
```

## 6.3 `skills/actions/scripts/create-close-ticket-proposal.ts`

```typescript
import { createProposal, singleStepExecution, requireString } from "../../../lib/proposal"
import { parseFlags, emitSuccess, emitError } from "../../../lib/script-runner"

async function main() {
  const flags = parseFlags(process.argv.slice(2))
  const ticketId = flags["ticket-id"] as string
  const reason = flags["reason"] as string
  const dryRun = flags["dry-run"] === true

  try {
    requireString(ticketId, "--ticket-id")
    requireString(reason, "--reason")
  } catch (e: any) { emitError(e.message) }

  const payload = {
    work_item_id: ticketId, actor_id: "system", operation: "CloseTicket",
    params: { ticket_id: ticketId }, reason,
    execution: singleStepExecution("Ticket.Close", { ticket_id: ticketId }, `Close ticket ${ticketId}`),
    created_by: "skill:create-close-ticket-proposal",
  }
  if (dryRun) emitSuccess({ dry_run: true, payload })
  try { emitSuccess({ created: true, proposal: await createProposal(payload) }) }
  catch (e: any) { emitError(e.message) }
}
main()
```

## 6.4 `skills/actions/scripts/create-escalate-proposal.ts`

```typescript
import { createProposal, singleStepExecution, requireString } from "../../../lib/proposal"
import { parseFlags, emitSuccess, emitError } from "../../../lib/script-runner"

async function main() {
  const flags = parseFlags(process.argv.slice(2))
  const ticketId = flags["ticket-id"] as string
  const escalateTo = flags["to"] as string ?? "manager-queue"
  const reason = flags["reason"] as string
  const dryRun = flags["dry-run"] === true

  try {
    requireString(ticketId, "--ticket-id")
    requireString(reason, "--reason")
  } catch (e: any) { emitError(e.message) }

  const payload = {
    work_item_id: ticketId, actor_id: "system", operation: "Escalate",
    params: { ticket_id: ticketId, to: escalateTo }, reason,
    execution: singleStepExecution("Ticket.Escalate", { ticket_id: ticketId, to: escalateTo }, `Escalate to ${escalateTo}`),
    created_by: "skill:create-escalate-proposal",
  }
  if (dryRun) emitSuccess({ dry_run: true, payload })
  try { emitSuccess({ created: true, proposal: await createProposal(payload) }) }
  catch (e: any) { emitError(e.message) }
}
main()
```

## 6.5 完成判定

- [ ] 4 个写脚本 dry-run 都能输出完整 payload
- [ ] 真创建后能在 UI 看到卡片
- [ ] Approve 后 mock 下游收到调用
- [ ] 缺 flag 时报错；非法 amount 报错

---

<a id="day-7"></a>

# Day 7 — 主代理：跑通端到端

**目标**：让 Claude 看到工单后自己调读工具、判断、创建提案。
**前置**：所有前面的天都完成。

## 7.1 工具注册中心

`agent/tools.ts`：

```typescript
import { spawn } from "child_process"
import { readFileSync } from "fs"
import { join } from "path"
import type Anthropic from "@anthropic-ai/sdk"

export const TOOL_REGISTRY: Anthropic.Tool[] = [
  {
    name: "read_policies",
    description: "Load the three policy/decision/playbook-index documents at the start of every work item. Must be called first.",
    input_schema: { type: "object", properties: {}, required: [] },
  },
  {
    name: "read_playbook",
    description: "Read a specific playbook by file name (without .md).",
    input_schema: {
      type: "object",
      properties: { name: { type: "string", description: "playbook file name without extension" } },
      required: ["name"],
    },
  },
  {
    name: "get_ticket",
    description: "Fetch a support ticket by its ID, including title, body, attachments, timestamps.",
    input_schema: {
      type: "object",
      properties: { ticket_id: { type: "string" } },
      required: ["ticket_id"],
    },
  },
  {
    name: "get_user_profile",
    description: "Fetch a user's profile: lifetime orders, spend, refund history, recent claims pattern.",
    input_schema: {
      type: "object",
      properties: { user_id: { type: "string" } },
      required: ["user_id"],
    },
  },
  {
    name: "get_order",
    description: "Fetch an order's current state: status, items, shipping address, tracking number.",
    input_schema: {
      type: "object",
      properties: { order_id: { type: "string" } },
      required: ["order_id"],
    },
  },
  {
    name: "get_payment_records",
    description: "Fetch a user's recent payment records.",
    input_schema: {
      type: "object",
      properties: { user_id: { type: "string" } },
      required: ["user_id"],
    },
  },
  {
    name: "delegate_to_logistics_investigator",
    description: "Delegate logistics evidence gathering to the logistics investigator sub-agent. Use for: signed-by-who, GPS, delivery time, scan history.",
    input_schema: {
      type: "object",
      properties: {
        claim_to_verify: { type: "string" },
        tracking_no: { type: "string" },
        order_id: { type: "string" },
        exact_questions: { type: "array", items: { type: "string" } },
        what_would_prove_or_disprove: { type: "string" },
      },
      required: ["claim_to_verify", "tracking_no", "exact_questions"],
    },
  },
  {
    name: "delegate_to_data_investigator",
    description: "Delegate historical/pattern evidence to the data investigator sub-agent. Use for: refund-history patterns, repeat-claim patterns, account anomalies.",
    input_schema: {
      type: "object",
      properties: {
        claim_to_verify: { type: "string" },
        user_id: { type: "string" },
        time_window: { type: "string" },
        exact_questions: { type: "array", items: { type: "string" } },
        what_would_prove_or_disprove: { type: "string" },
      },
      required: ["claim_to_verify", "user_id", "exact_questions"],
    },
  },
  {
    name: "create_refund_proposal",
    description: "Create a refund proposal. Goes to human review. NEVER auto-executes.",
    input_schema: {
      type: "object",
      properties: {
        ticket_id: { type: "string" },
        user_id: { type: "string" },
        order_id: { type: "string" },
        amount_cents: { type: "integer" },
        reason: { type: "string", description: "INTERNAL reason for reviewer; NOT user-visible" },
      },
      required: ["ticket_id", "user_id", "order_id", "amount_cents", "reason"],
    },
  },
  {
    name: "create_reply_proposal",
    description: "Create a user-visible reply proposal. The reply must start with brand greeting and end with brand signature per POLICY.md.",
    input_schema: {
      type: "object",
      properties: {
        ticket_id: { type: "string" },
        user_id: { type: "string" },
        user_message: { type: "string", description: "User-VISIBLE message" },
        reason: { type: "string", description: "INTERNAL reason for reviewer" },
      },
      required: ["ticket_id", "user_id", "user_message", "reason"],
    },
  },
  {
    name: "create_close_ticket_proposal",
    description: "Close the ticket. Use after a reply proposal when no further action is needed.",
    input_schema: {
      type: "object",
      properties: {
        ticket_id: { type: "string" },
        reason: { type: "string" },
      },
      required: ["ticket_id", "reason"],
    },
  },
  {
    name: "create_escalate_proposal",
    description: "Escalate to human manager queue. Use when policy mandates escalation (amount > 2000 CNY, fraud signals, etc.)",
    input_schema: {
      type: "object",
      properties: {
        ticket_id: { type: "string" },
        to: { type: "string" },
        reason: { type: "string" },
      },
      required: ["ticket_id", "reason"],
    },
  },
]

const POLICY_DIR = join(__dirname, "policies")

function runScript(scriptPath: string, args: string[]): Promise<{ status: string; data?: any; error?: string }> {
  return new Promise((resolve) => {
    const proc = spawn("bun", ["run", scriptPath, ...args], { stdio: ["ignore", "pipe", "pipe"] })
    let stdout = "", stderr = ""
    proc.stdout.on("data", (d) => stdout += d.toString())
    proc.stderr.on("data", (d) => stderr += d.toString())
    proc.on("close", () => {
      try {
        const out = stdout.trim() || stderr.trim()
        resolve(JSON.parse(out))
      } catch {
        resolve({ status: "error", error: `failed to parse script output: ${stdout} | ${stderr}` })
      }
    })
  })
}

function flagsFromInput(input: Record<string, unknown>): string[] {
  const out: string[] = []
  for (const [k, v] of Object.entries(input)) {
    const key = "--" + k.replace(/_/g, "-")
    if (typeof v === "boolean") {
      if (v) out.push(key)
    } else {
      out.push(key, String(v))
    }
  }
  return out
}

export async function executeTool(
  name: string,
  input: Record<string, unknown>,
  subagentRunners: Record<string, (packet: unknown) => Promise<unknown>>,
): Promise<string> {
  switch (name) {
    case "read_policies": {
      const policy = readFileSync(join(POLICY_DIR, "POLICY.md"), "utf-8")
      const decision = readFileSync(join(POLICY_DIR, "DECISION_FRAMEWORK.md"), "utf-8")
      const index = readFileSync(join(POLICY_DIR, "playbooks", "INDEX.md"), "utf-8")
      return JSON.stringify({ POLICY: policy, DECISION_FRAMEWORK: decision, PLAYBOOK_INDEX: index })
    }
    case "read_playbook": {
      const file = join(POLICY_DIR, "playbooks", `${input.name}.md`)
      try { return readFileSync(file, "utf-8") }
      catch { return JSON.stringify({ error: `playbook not found: ${input.name}` }) }
    }
    case "get_ticket":
      return JSON.stringify(await runScript("skills/tickets/scripts/get-ticket.ts", flagsFromInput(input)))
    case "get_user_profile":
      return JSON.stringify(await runScript("skills/users/scripts/get-user-profile.ts", flagsFromInput(input)))
    case "get_order":
      return JSON.stringify(await runScript("skills/orders/scripts/get-order.ts", flagsFromInput(input)))
    case "get_payment_records":
      return JSON.stringify(await runScript("skills/orders/scripts/get-payment-records.ts", flagsFromInput(input)))
    case "create_refund_proposal":
      return JSON.stringify(await runScript("skills/actions/scripts/create-refund-proposal.ts", flagsFromInput(input)))
    case "create_reply_proposal":
      return JSON.stringify(await runScript("skills/actions/scripts/create-reply-proposal.ts", flagsFromInput(input)))
    case "create_close_ticket_proposal":
      return JSON.stringify(await runScript("skills/actions/scripts/create-close-ticket-proposal.ts", flagsFromInput(input)))
    case "create_escalate_proposal":
      return JSON.stringify(await runScript("skills/actions/scripts/create-escalate-proposal.ts", flagsFromInput(input)))
    case "delegate_to_logistics_investigator":
      return JSON.stringify(await subagentRunners.logistics(input))
    case "delegate_to_data_investigator":
      return JSON.stringify(await subagentRunners.data(input))
    default:
      return JSON.stringify({ error: `unknown tool: ${name}` })
  }
}
```

## 7.2 主代理循环

`agent/primary-agent.ts`：

```typescript
import Anthropic from "@anthropic-ai/sdk"
import { TOOL_REGISTRY, executeTool } from "./tools"
import { runLogisticsInvestigator } from "./subagents/logistics-investigator"
import { runDataInvestigator } from "./subagents/data-investigator"

const client = new Anthropic()
const MODEL = "claude-opus-4-7"

const SYSTEM_PROMPT = `You are the primary customer-support agent for a cross-border e-commerce business.

Your authority:
- You investigate, judge, and DRAFT proposal cards.
- You have ZERO direct write access. All mutations go through proposal cards that a human reviewer approves.

Operating contract (in order for every work item):

1. Call read_policies once at the very start. Treat the three documents as binding.
2. Read the ticket (get_ticket) and the user (get_user_profile).
3. Identify which playbook to load via PLAYBOOK_INDEX, then call read_playbook.
4. Gather core evidence per the playbook (get_order, get_payment_records, delegate_to_*).
5. Map evidence to one of: direct proof / converging / mixed but plausible / scoped negative / load-bearing gap / user-only.
6. Decide outcome per DECISION_FRAMEWORK and POLICY.
7. Create separate proposal cards for separate downstream actions (one card = one downstream call).
8. user_message must follow brand greeting/signature rules in POLICY.md.

Hard rules:
- Never put internal disposition wording ("verified fraud", "no refund needed") into user_message.
- Never claim a fact in user_message that has no verified source.
- Escalate when POLICY mandates (amount > 2000 CNY, repeated claims, etc.).
- If evidence is a load-bearing gap, create an escalate proposal, not a guess.

End every work item with a short text summary of what you did and which proposals you created.`

interface RunOptions {
  workItem: { ticket_id: string; user_id: string }
  maxIterations?: number
  verbose?: boolean
}

export async function runPrimaryAgent(opts: RunOptions) {
  const max = opts.maxIterations ?? 30

  const subagentRunners = {
    logistics: runLogisticsInvestigator,
    data: runDataInvestigator,
  }

  const userMessage = `New work item.

Ticket ID: ${opts.workItem.ticket_id}
User ID: ${opts.workItem.user_id}

Start your investigation per the operating contract.`

  const messages: Anthropic.MessageParam[] = [{ role: "user", content: userMessage }]
  let iterations = 0

  while (iterations++ < max) {
    const resp = await client.messages.create({
      model: MODEL,
      max_tokens: 4096,
      system: SYSTEM_PROMPT,
      tools: TOOL_REGISTRY,
      messages,
    })

    if (opts.verbose) console.log("=== assistant turn ===\n", JSON.stringify(resp.content, null, 2))

    messages.push({ role: "assistant", content: resp.content })

    if (resp.stop_reason === "end_turn") {
      const finalText = resp.content.find(b => b.type === "text") as any
      console.log("\n=== AGENT FINAL ===\n", finalText?.text ?? "(no text)")
      return
    }

    if (resp.stop_reason === "tool_use") {
      const toolResults: any[] = []
      for (const block of resp.content) {
        if (block.type !== "tool_use") continue
        if (opts.verbose) console.log(`>> tool: ${block.name}`, block.input)
        const result = await executeTool(block.name, block.input as any, subagentRunners)
        toolResults.push({ type: "tool_result", tool_use_id: block.id, content: result })
      }
      messages.push({ role: "user", content: toolResults })
      continue
    }

    console.warn("unexpected stop_reason:", resp.stop_reason)
    break
  }

  console.log("\n[max iterations reached]")
}

// CLI
if (import.meta.main) {
  const ticketId = process.argv[2]
  const userId = process.argv[3]
  if (!ticketId || !userId) {
    console.error("usage: bun run agent/primary-agent.ts <ticket_id> <user_id>")
    process.exit(1)
  }
  runPrimaryAgent({
    workItem: { ticket_id: ticketId, user_id: userId },
    verbose: process.env.VERBOSE === "1",
  })
}
```

## 7.3 子代理临时桩

Day 8 才正式实现子代理，今天先让主代理可以跑：

`agent/subagents/logistics-investigator.ts`：

```typescript
export async function runLogisticsInvestigator(_packet: unknown) {
  return {
    finding: "(stub) logistics investigator not yet implemented",
    evidence_strength: "inconclusive",
    evidence: [],
    queries: [],
    caveats: "implement in Day 8",
  }
}
```

`agent/subagents/data-investigator.ts`：

```typescript
export async function runDataInvestigator(_packet: unknown) {
  return {
    finding: "(stub) data investigator not yet implemented",
    evidence_strength: "inconclusive",
    evidence: [],
    queries: [],
    caveats: "implement in Day 9",
  }
}
```

## 7.4 跑起来

确保都在跑：
- `bun run backend/server.ts` （:31367 提案后端）
- `bun run mock-downstream/server.ts` （:31368 mock 下游业务）
- `bun run mock-downstream/data-server.ts` （:31369 mock 数据）

然后：

```bash
VERBOSE=1 bun run agent/primary-agent.ts t12345 u789
```

期待发生的事：

1. 主代理首先调用 `read_policies`
2. 主代理调用 `get_ticket`、`get_user_profile`
3. 主代理识别这是物流类工单，调用 `read_playbook(name="refund-not-received")`
4. 主代理调用 `get_order` 拿到订单状态 "delivered"
5. 主代理调用 `delegate_to_logistics_investigator`（虽然现在是桩，返回 inconclusive）
6. 主代理因为子代理还没实现，可能选择创建 escalate 提案，或试图直接判断
7. 主代理输出"我创建了 X 张提案"

去 http://localhost:31367/ui/index.html 看新出现的卡片。

## 7.5 完成判定

- [ ] 主代理 自己 完成完整循环：读政策 → 读工单 → 读用户 → 读剧本 → 取证 → 创建至少 1 张提案
- [ ] 至少 1 张提案 出现在 UI
- [ ] Reason 字段写得清晰（reviewer 能读懂）
- [ ] user_message 符合品牌格式（有"您好"开头、签名结尾）

> **常见问题**：模型创建了"多步合并"提案 → 因为后端拒绝 `steps.length !== 1`，会报错。这是正确的——把它的反馈喂回模型，下一轮它会改成多张卡。

---

<a id="day-8"></a>

# Day 8 — 第一个子代理 + 委派协议

**目标**：用一个独立的 agent loop 跑物流证据查证。
**前置**：Day 7 完成。

## 8.1 委派协议（最重要的）

委派包 = 主代理调用子代理时传的 JSON。**协议在主代理工具定义里就已经声明了**（见 7.1 的 `delegate_to_logistics_investigator` 的 input_schema）。

子代理收到后必须：
1. 校验关键字段（这里至少要有 `tracking_no` 或 `order_id` + `exact_questions`）
2. 如果不全，返回 `INSUFFICIENT_DELEGATION_PACKET` 不要硬猜
3. 输出标准结构：`{ finding, evidence_strength, evidence, queries, caveats }`
4. **不做业务判断**

## 8.2 子代理实现

替换 `agent/subagents/logistics-investigator.ts`：

```typescript
import Anthropic from "@anthropic-ai/sdk"
import { readGet } from "../../lib/read"

const client = new Anthropic()

const SUBAGENT_SYSTEM = `You are the logistics-evidence sub-agent.

You return precise, sourced facts about shipping / delivery / signing. You do NOT make business decisions; you provide facts the primary agent can use.

You must validate the delegation packet FIRST:
- tracking_no OR order_id is required
- exact_questions is required (non-empty array)

If any required field is missing, return:
{ "result": "INSUFFICIENT_DELEGATION_PACKET", "missing": [...] }

Otherwise gather evidence using the logistics_lookup tool and return:
{
  "finding": "<direct answer to the delegated question>",
  "evidence_strength": "<direct proof | converging | scoped negative | load-bearing gap | inconclusive>",
  "evidence": ["<timestamped fact / source>", ...],
  "queries": ["<what you looked up and why>", ...],
  "caveats": "<freshness / scope / uncertainty>"
}

Output ONLY the JSON object, no prose.`

const SUBAGENT_TOOLS: Anthropic.Tool[] = [
  {
    name: "logistics_lookup",
    description: "Look up a shipment by tracking number. Returns scan history, sign-by, GPS, delivered_at.",
    input_schema: {
      type: "object",
      properties: { tracking_no: { type: "string" } },
      required: ["tracking_no"],
    },
  },
]

async function executeSubagentTool(name: string, input: any): Promise<string> {
  if (name === "logistics_lookup") {
    const data = await readGet(`/logistics/${input.tracking_no}`)
    return JSON.stringify(data)
  }
  return JSON.stringify({ error: `unknown tool: ${name}` })
}

export async function runLogisticsInvestigator(packet: unknown) {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: `[DELEGATION_PACKET]\n${JSON.stringify(packet, null, 2)}` },
  ]

  let iter = 0
  while (iter++ < 10) {
    const resp = await client.messages.create({
      model: "claude-opus-4-7",
      max_tokens: 2048,
      system: SUBAGENT_SYSTEM,
      tools: SUBAGENT_TOOLS,
      messages,
    })
    messages.push({ role: "assistant", content: resp.content })

    if (resp.stop_reason === "end_turn") {
      const text = resp.content.find(b => b.type === "text") as any
      try { return JSON.parse(text?.text ?? "{}") }
      catch { return { result: "PARSE_ERROR", raw: text?.text } }
    }

    if (resp.stop_reason === "tool_use") {
      const results: any[] = []
      for (const block of resp.content) {
        if (block.type !== "tool_use") continue
        const r = await executeSubagentTool(block.name, block.input)
        results.push({ type: "tool_result", tool_use_id: block.id, content: r })
      }
      messages.push({ role: "user", content: results })
    }
  }
  return { result: "MAX_ITERATIONS" }
}
```

## 8.3 测试主代理 → 子代理 →

重新跑 Day 7 的工单：

```bash
VERBOSE=1 bun run agent/primary-agent.ts t12345 u789
```

期待发生的事：

1. 主代理走到 `delegate_to_logistics_investigator`，传入 tracking_no = "SF1234567890"
2. 子代理实例独立运行，查 logistics API
3. 子代理返回 `{ finding: "已签收，签收人=前台代收，GPS 距地址 35 米", evidence_strength: "direct proof", ... }`
4. 主代理基于此判断"GPS 离地址很近、首次申诉、单价 ¥299" → 选"善意补偿"路径
5. 主代理创建 1 张退款提案（半额）+ 1 张回复提案
6. UI 上出现两张 pending 卡

> 如果想看子代理拒绝不全包：手动发一个缺 tracking_no 的工单试试，子代理应该返回 `INSUFFICIENT_DELEGATION_PACKET`。

## 8.4 完成判定

- [ ] 子代理独立 agent loop 跑通
- [ ] 主代理委派后能拿到结构化 finding
- [ ] 缺字段时子代理返回 INSUFFICIENT_DELEGATION_PACKET
- [ ] 全链路：主代理 → 子代理 → mock API → 子代理总结 → 主代理决策 → 提案卡 → 人审 → 下游执行

---

<a id="day-9"></a>

# Day 9 — 第二个子代理 + 多源证据汇总

**目标**：第二个子代理 `data_investigator`，让主代理同时调两个子代理，做多源融合判断。
**前置**：Day 8 完成。

## 9.1 实现 data_investigator

替换 `agent/subagents/data-investigator.ts`：

```typescript
import Anthropic from "@anthropic-ai/sdk"
import { readGet } from "../../lib/read"

const client = new Anthropic()

const SUBAGENT_SYSTEM = `You are the historical-data sub-agent.

You return precise, sourced facts about user history, refund patterns, repeat-claim patterns. You do NOT make business decisions.

Required packet fields: user_id, exact_questions (non-empty array).

If missing, return:
{ "result": "INSUFFICIENT_DELEGATION_PACKET", "missing": [...] }

Otherwise return:
{
  "finding": "...",
  "evidence_strength": "<direct proof | converging | scoped negative | load-bearing gap | inconclusive>",
  "evidence": [...],
  "queries": [...],
  "caveats": "..."
}

Output ONLY the JSON object.`

const SUBAGENT_TOOLS: Anthropic.Tool[] = [
  {
    name: "user_history_lookup",
    description: "Fetch user history: refunds_last_90d, not_received_claims_last_90d, lifetime spend.",
    input_schema: {
      type: "object",
      properties: { user_id: { type: "string" } },
      required: ["user_id"],
    },
  },
]

async function execTool(name: string, input: any): Promise<string> {
  if (name === "user_history_lookup") return JSON.stringify(await readGet(`/users/${input.user_id}`))
  return JSON.stringify({ error: `unknown: ${name}` })
}

export async function runDataInvestigator(packet: unknown) {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: `[DELEGATION_PACKET]\n${JSON.stringify(packet, null, 2)}` },
  ]
  let iter = 0
  while (iter++ < 10) {
    const resp = await client.messages.create({
      model: "claude-opus-4-7",
      max_tokens: 2048,
      system: SUBAGENT_SYSTEM,
      tools: SUBAGENT_TOOLS,
      messages,
    })
    messages.push({ role: "assistant", content: resp.content })
    if (resp.stop_reason === "end_turn") {
      const text = resp.content.find(b => b.type === "text") as any
      try { return JSON.parse(text?.text ?? "{}") }
      catch { return { result: "PARSE_ERROR", raw: text?.text } }
    }
    if (resp.stop_reason === "tool_use") {
      const results: any[] = []
      for (const block of resp.content) {
        if (block.type !== "tool_use") continue
        results.push({ type: "tool_result", tool_use_id: block.id, content: await execTool(block.name, block.input) })
      }
      messages.push({ role: "user", content: results })
    }
  }
  return { result: "MAX_ITERATIONS" }
}
```

## 9.2 制造一个让两个子代理都被调用的 case

在 mock 数据中加一个"问题用户"：

编辑 `mock-downstream/data-server.ts`：

```typescript
// 在 users 里加：
"u999": {
  id: "u999", email: "shady@example.com", joined_at: "2026-03-01",
  lifetime_orders: 4, lifetime_spend_cents: 89900,
  refunds_last_90d: 3, not_received_claims_last_90d: 2,
},
// 在 tickets 里加：
"t99999": {
  id: "t99999", user_id: "u999", order_id: "o777",
  title: "包裹丢失",
  body: "您好，我又没收到包裹了，请退款。",
  created_at: "2026-05-11T08:00:00Z",
  attachments: [],
},
// 在 orders 里加：
"o777": {
  id: "o777", user_id: "u999", status: "delivered",
  amount_cents: 89900, items: [{ sku: "B2", qty: 1, name: "高级护肤套装" }],
  shipping_address: "北京市朝阳区XX路", tracking_no: "SF9999999999",
  delivered_at: "2026-05-09T14:00:00Z",
},
// 在 logistics 里加：
"SF9999999999": {
  tracking_no: "SF9999999999",
  events: [
    { at: "2026-05-09T14:00:00Z", event: "delivered", signed_by: "本人签收", gps: { lat: 39.9, lng: 116.4, distance_to_address_m: 8 } },
  ],
},
```

重启 mock data server，跑：

```bash
VERBOSE=1 bun run agent/primary-agent.ts t99999 u999
```

期待发生的事：

1. 主代理读 user → 看到 `refunds_last_90d=3` 命中政策红线
2. 主代理调 `delegate_to_data_investigator` 确认重复模式
3. 主代理调 `delegate_to_logistics_investigator` 确认本次签收正常
4. 两个子代理都返回 `direct proof`
5. 主代理选"政策性拒绝 + 升级"路径
6. 创建：1 张回复提案（含礼貌拒绝模板）+ 1 张升级提案

## 9.3 完成判定

- [ ] 主代理在一个工单中调用 2 个子代理
- [ ] 子代理返回独立结论，主代理融合后判断
- [ ] 高欺诈嫌疑的工单**不会**直接创建退款提案

---

<a id="day-10"></a>

# Day 10 — 集成测试 / 三个真实 case 回放

**目标**：用 3 个不同形态的工单验证系统稳定性。
**前置**：Day 9 完成。

## 10.1 三个 case

**Case A** — 正常物流异常（应直接退款）：
- t10001 / u100 / o100，amount=15000
- 物流：未签收（status=in_transit 但 7 天没动）

**Case B** — 边角善意补偿（应给优惠券）：
- t10002 / u101 / o101，amount=29900
- 物流：签收，GPS 距地址 1.5km
- 用户 90 天无任何不良记录

**Case C** — 欺诈嫌疑（应拒绝 + 升级）：
- 已经在 Day 9 做过

加 mock 数据，跑三遍。每遍记录：
1. 主代理总共调了哪些工具
2. 创建了哪些提案
3. Reason 字段是否清晰
4. user_message 是否合规
5. 平均 tokens 消耗

## 10.2 验收 checklist

- [ ] **Case A**：直接退款提案 + 回复提案，不创建升级
- [ ] **Case B**：半额退款（或优惠券）+ 回复提案，reason 说明"善意补偿"
- [ ] **Case C**：拒绝 reply 提案 + escalate 提案，**没有**任何退款提案
- [ ] 三个 case 加起来：≤ 50 次工具调用、≤ 30 万 token

## 10.3 离线盲审

把三张 case 的所有提案 reason 给 2 位业务专家盲读（不让他们看 user_message）。问：

- 你能凭这份 reason 决定是否批准吗？
- 你的批准结果会和 Agent 自己的状态一致吗？

如果通过率 ≥ 80% → 系统可以灰度上线。

---

<a id="part-e"></a>

# Part E — 适配到你的领域

## E1. 改 3 个地方

| 地方 | 原 | 你的 |
|---|---|---|
| `agent/policies/*` | 客服政策 | 你的领域政策 |
| `skills/*/scripts/` | 客服读写脚本 | 你的领域读写脚本 |
| `agent/subagents/*` | 物流 / 数仓 | 你的证据源 |
| `agent/tools.ts` | 客服工具列表 | 你的工具注册 |
| `backend/server.ts` 的 `ALLOWED_COMMANDS` | OMS.Refund 等 | 你的下游命令 |
| `mock-downstream/*` | 客服 mock | 你的真实下游接入 |

## E2. 架构本身不动的部分

- `lib/proposal.ts` / `lib/read.ts` / `lib/script-runner.ts`
- `backend/db.ts` / `backend/schema.sql` / 入库流程
- 主代理循环模板 / 子代理循环模板
- 委派包契约
- 输出结构

## E3. 决定性的选择

| 选择 | 推荐 |
|---|---|
| 主代理是否直接持有所有工具？ | 否，按数据源切子代理 |
| Reviewer 角色是否可分级？ | ≥ 一定金额/敏感度要"高级 reviewer"双签 |
| 是否支持 N 步组合提案？ | 否，永远 1 步 = 1 卡 |
| 是否允许"已批准"提案修改？ | 否，要改就 supersede 新卡 |
| 模型选择 | 主代理 stable / sub-agent 可用高推理 |
| 审计日志多久不可删？ | 至少 1 年（合规） |
| Rate limit | 主代理每 case 最多 30 工具调用、≤ $X token 成本 |

---

<a id="part-f"></a>

# Part F — 故障排查

| 症状 | 可能原因 | 排查 |
|---|---|---|
| 主代理不停循环 | system prompt 没说明何时 end_turn | 加一句"After creating all needed proposals, end with a one-paragraph summary" |
| 主代理乱编事实 | 政策文档没强制"truth discipline" | 检查 POLICY.md 第 0 节 |
| 提案 backend 报 multi-step | 模型生成了组合执行 | 让它创建多张卡，错误信息回喂模型即可 |
| user_message 暴露内部判断 | 模型混淆 reason 与 user_message | 在 tool description 里反复强调 INTERNAL vs USER-VISIBLE |
| 子代理硬猜 | system prompt 没强制 INSUFFICIENT 返回 | 加显式校验逻辑 |
| 工具调用失败 | 子进程编码 / 路径问题 | Bun 在 Windows 下注意 `process.argv` 切片 |
| Approve 后下游 502 | mock-downstream 没起 / ALLOWED_COMMANDS 漏配 | 检查端口 / allowlist |
| 模型选错剧本 | INDEX.md 路由优先级不清晰 | 把"优先级规则"写明 |
| Reviewer 频繁改 user_message | 模型起草质量不够 | 把成功 case 中的好 user_message 加进 POLICY 作为示例 |
| 多代理上下文爆炸 | 主代理把所有工具结果都看一遍 | 让子代理返回更精简的 finding |

---

<a id="appendix"></a>

# 附录

## A1. 完整文件树

```
my-agent/
├── .env
├── package.json
├── tsconfig.json
├── proposals.db                  (运行后自动生成)
├── agent/
│   ├── policies/
│   │   ├── POLICY.md
│   │   ├── DECISION_FRAMEWORK.md
│   │   └── playbooks/
│   │       ├── INDEX.md
│   │       ├── refund-not-received.md
│   │       ├── refund-defective.md
│   │       └── account-question.md
│   ├── primary-agent.ts
│   ├── subagents/
│   │   ├── data-investigator.ts
│   │   └── logistics-investigator.ts
│   └── tools.ts
├── skills/
│   ├── tickets/scripts/get-ticket.ts
│   ├── users/scripts/get-user-profile.ts
│   ├── orders/scripts/get-order.ts
│   ├── orders/scripts/get-payment-records.ts
│   └── actions/scripts/
│       ├── create-refund-proposal.ts
│       ├── create-reply-proposal.ts
│       ├── create-close-ticket-proposal.ts
│       └── create-escalate-proposal.ts
├── lib/
│   ├── proposal.ts
│   ├── read.ts
│   └── script-runner.ts
├── backend/
│   ├── server.ts
│   ├── db.ts
│   ├── schema.sql
│   └── ui/index.html
└── mock-downstream/
    ├── server.ts
    └── data-server.ts
```

## A2. 环境变量

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxx

# 后端
PORT=31367
DB_PATH=./proposals.db
PROPOSAL_BACKEND_TOKEN=dev-token-change-me
DOWNSTREAM_URL=http://localhost:31368

# 客户端
PROPOSAL_BACKEND_URL=http://localhost:31367
MOCK_DB_URL=http://localhost:31369

# 主代理
VERBOSE=1   # 看到每一步工具调用
```

## A3. 常用命令速查

```bash
# 起所有服务（4 个终端 或用 just / pm2 编排）
bun run backend/server.ts
bun run mock-downstream/server.ts
bun run mock-downstream/data-server.ts

# 单独测试某个脚本
bun run skills/tickets/scripts/get-ticket.ts --ticket-id t12345
bun run skills/actions/scripts/create-refund-proposal.ts --dry-run \
  --ticket-id t12345 --user-id u789 --order-id o456 \
  --amount-cents 29900 --reason "..."

# 跑主代理
bun run agent/primary-agent.ts t12345 u789

# 查 DB
sqlite3 proposals.db "SELECT id, status, operation, created_at FROM proposals ORDER BY created_at DESC LIMIT 10"
sqlite3 proposals.db "SELECT * FROM audit_log ORDER BY timestamp DESC LIMIT 20"

# UI
open http://localhost:31367/ui/index.html
```

## A4. 上线前最后一遍 checklist

- [ ] 主代理 prompt 里有"开局必读三份文档"
- [ ] 政策文档里有"truth discipline"硬性约束
- [ ] 后端拒绝 multi-step / 不在 allowlist 的命令
- [ ] 所有 mutation 都生成 proposal，**grep 主代理代码确认没有直接 fetch 下游**
- [ ] dry-run 在每个写脚本上都工作
- [ ] 审计表 `audit_log` 在每次状态变化都写
- [ ] reviewer 角色至少有一个（鉴权可以是简单口令，但要存）
- [ ] 错误格式都是 JSON，不是 stack trace
- [ ] 30 个历史 case 离线回放后业务专家盲审通过率 ≥ 80%
- [ ] 灰度路由只开 1-2 个低风险类型

---

# 写在最后

这套架构的**真正护城河**不是哪段代码，而是：

> **"主代理在物理上无法直接修改生产数据。即使它疯了，最多也只是创建一堆被否决的提案。"**

如果你忠实执行了：
1. 主代理零写权限（grep 验证）
2. 提案后端拒绝任何不在 allowlist 的命令
3. 三层文档 + 子代理切分
4. 人审 UI 作为唯一执行入口

你就拥有了一个**架构层防呆**的 LLM 系统。比再聪明的 prompt 都靠谱。

---

下次再搭一个类似系统：
- 第一次读这份文档，照抄
- 第二次读这份文档，调整顺序
- 第三次，你已经不需要这份文档了

