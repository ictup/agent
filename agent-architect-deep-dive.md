# Agent 系统：架构师深度手册

> **这份文档与前几份的关系**：
> - `celest-support-replication-guide.md` — 拆解 Celest Support 的实现
> - `agent-resume-and-playbook.md` — 给简历用的方法论
> - `agent-build-tutorial-from-scratch.md` — 完整可运行的代码教程（"How"）
> - `agent-architect-deep-dive.md`（本文）— **架构选型与设计哲学（"Why"）**
>
> 配合阅读：先读 `build-tutorial` 把代码跑起来，再读本文理解每一步背后的原因；或者先读本文确立设计观，再用 `build-tutorial` 验证。
>
> 本文**不重复代码**，专注：框架选型理由、架构模式命名、Anthropic API 深度、易错点百科、业务/框架边界、垂类适配。

---

# 目录

- [Ch 1 — Agent 系统的设计哲学](#ch1)
- [Ch 2 — 技术栈选型与替代方案](#ch2)
- [Ch 3 — 架构模式（命名 + 应用）](#ch3)
- [Ch 4 — 业务 vs 框架的锐利分层](#ch4)
- [Ch 5 — Anthropic Tool Use API 深度](#ch5)
- [Ch 6 — Prompt Caching：决定单位经济的关键技术](#ch6)
- [Ch 7 — 数据契约（决定系统寿命）](#ch7)
- [Ch 8 — 每日施工：技术 / 业务 / 易错点对照](#ch8)
- [Ch 9 — 易错点百科（按层分类）](#ch9)
- [Ch 10 — 生产工程化](#ch10)
- [Ch 11 — 评估与迭代](#ch11)
- [Ch 12 — 团队与组织影响](#ch12)
- [Ch 13 — 三个垂类的真实适配演练](#ch13)
- [Ch 14 — 写给自己的结尾](#ch14)

---

<a id="ch1"></a>

# Ch 1 — Agent 系统的设计哲学

## 1.1 你在解决的真正问题是什么

LLM Agent 表面上是"让大模型调工具完成任务"，本质上是**两个完全不同的问题被错误地混在一起**：

| 问题 | 性质 | 解法 |
|---|---|---|
| **怎么让 LLM 选对工具、写对参数** | 软问题（提示工程） | Prompt + Few-shot + Schema |
| **怎么让 LLM 即使写错也不能伤害生产** | 硬问题（系统设计） | 物理隔离 + 人审 + 契约 |

绝大多数失败的 Agent 系统都是因为：**把硬问题当软问题来解** — 用更长的 prompt、更多的 example、更聪明的模型，去解决一个本质上是"权限边界"的问题。

正确的认知是：

> **Agent 系统的可靠性不来自模型变好，来自"模型即使变坏也不能造成损失"的架构。**

这是本系统全部设计选择的"第一性原理"。

## 1.2 三种常见死法

### 死法 A："厉害的 Prompt"派

症状：一个 5000 字的 system prompt，里面写了 30 条规则。
死因：长 prompt 在长 session 中会被模型自己说服掉。模型在第 50 轮工具调用时早已忘了第 3 条规则。
治法：把规则写成**可被工具读取的文档**，让模型每个 case 开局重新加载，而不是一次性塞 prompt。

### 死法 B："万能 Agent"派

症状：一个 Agent 持有 50+ 工具，期望它"理解整个业务"。
死因：上下文爆炸、工具选错、推理质量随工具数线性下降。模型在 50 个工具中选对的概率，比在 5 个中选对低得多。
治法：**主代理 + 专科子代理**。主代理少于 15 个工具，每个子代理少于 8 个。

### 死法 C："让 AI 直接执行"派

症状：Agent 调 `delete_user(id=123)`、`refund(amount=999999)` 这种工具直接打到生产。
死因：第一次幻觉就出事故。
治法：**所有 mutation 都通过提案 + 人审网关**。AI 写卡，人按按钮。

## 1.3 系统的四条铁律

1. **铁律 1（写权限）**：主代理在物理层（不是 prompt 层）没有任何下游写权限
2. **铁律 2（单步执行）**：一张提案卡 = 一次下游调用。组合操作开多张卡，由 reviewer 按顺序批
3. **铁律 3（信道分离）**：内部判断词（reason）和用户可见文案（user_message）走完全不同的字段
4. **铁律 4（证据可追溯）**：每一句用户可见陈述都必须能在审计链路中追到具体来源

四条铁律一旦确立，剩下的全部是"业务装配 + 优化"。

## 1.4 什么决定一个 Agent 系统的成败

按重要性排序：

| 决定因素 | 权重 |
|---|---|
| 1. 提案+人审网关是否真的卡住所有 mutation | ★★★★★ |
| 2. 三层规则文档（Policy / Decision Framework / Playbook）的质量 | ★★★★★ |
| 3. 子代理切分是否按"证据源"而不是"业务" | ★★★★ |
| 4. 数据契约（提案 schema、委派包 schema）的稳定性 | ★★★★ |
| 5. Prompt Caching 是否启用 | ★★★★（成本相关） |
| 6. Reviewer UI 是否让审稿人能"只读 reason 就批准" | ★★★ |
| 7. 模型选型与成本/质量平衡 | ★★★ |
| 8. System Prompt 的细节 | ★★ |
| 9. 工具 description 的清晰度 | ★★ |
| 10. 编程语言 / Web 框架选择 | ★ |

**新手把精力放在 8-10，老手把精力放在 1-4**。

---

<a id="ch2"></a>

# Ch 2 — 技术栈选型与替代方案

## 2.1 选型决策矩阵

| 层 | 我的选择 | 替代方案 | 选择理由 |
|---|---|---|---|
| **LLM** | Claude（Anthropic）| GPT / Gemini / 本地模型 | Tool Use 稳定，Prompt Caching 成熟，系统提示遵循度高 |
| **Agent 编排** | 原生 Anthropic SDK | LangChain / LlamaIndex / OpenAI Agents SDK / MCP | 透明度高、可控、没有隐藏抽象（详见 2.3） |
| **运行时** | Bun | Node / Deno / Python | 内置 TS / SQLite / HTTP，零配置 |
| **HTTP 框架** | Hono | Express / Fastify / Koa | 轻、TS 友好、跑在 Bun/Workers/Node 都行 |
| **数据库** | SQLite（bun:sqlite） | Postgres / MySQL / MongoDB | 单文件、零运维、并发足够；上规模再迁 Postgres |
| **审计 / 事件** | SQLite 表 | EventStore / Kafka | 提案吞吐量 ≤ 10K/天时表足够 |
| **UI** | Alpine.js + 原生 HTML | React / Vue / Svelte | 一个 reviewer 工具不需要 SPA 复杂度 |
| **进程编排** | Bun.spawn | Docker / k8s / pm2 | 开发期手起；生产再上 Docker Compose |
| **可观测性** | console + DB | OpenTelemetry / Datadog | MVP 用 console，量化后上 OTEL |
| **评估** | 自写 replay 脚本 | Promptfoo / Braintrust / LangSmith | 自己掌控数据格式，工具按需引入 |

## 2.2 为什么 Claude（而不是 GPT / Gemini）

**这是商业 + 工程的混合考量**，没有"客观最优"。我的实际依据：

| 维度 | Claude | GPT-4o | Gemini 2.5 |
|---|---|---|---|
| Tool Use 稳定性 | 极稳 | 稳 | 稳 |
| 长 system prompt 遵循 | 极强 | 中 | 强 |
| Prompt Caching 成熟度 | 1h TTL，5min 增强缓存 | 隐式缓存，控制弱 | 显式缓存，复杂 |
| 长上下文质量（>100K） | 极好 | 中 | 极好 |
| 价格（Sonnet 级） | $3 / $15 / MTok | $2.5 / $10 | $1.25 / $10 |
| 安全/拒答 | 偏严 | 偏松 | 中 |
| 商业合规友好度 | 极好（Anthropic 是 B2B 取向） | 好 | 好 |

**结论**：客服 / 合规类需要"长指令稳定遵循 + 强缓存"的场景，Claude 优势明显。如果你做的是高吞吐量的纯分类 / RAG / 自动化，GPT 也很合适。

> **重要：架构本身与模型无关**。Tool Use 是 OpenAI 发明的协议格式，三家都兼容。把 client.messages.create 换成 OpenAI 的 chat.completions.create，整套系统照跑。**模型可替换是这套架构的一大优点**。

## 2.3 为什么不用 LangChain / LlamaIndex

这是一个有争议的选择，但我的立场很明确：**Agent 系统是"权限与契约工程"，不是"LLM 调用工程"**。

LangChain 的核心抽象（`Chain`、`Agent`、`Tool`、`Memory`、`Retriever`）解决的是"如何让 LLM 调用更优雅"。但本系统的难点是：

- 提案数据契约的不变性
- 人审 UI 的可读性
- 审计链路的完整性
- 子代理工具集的物理隔离

这些都**不是 LLM 调用层的问题**，LangChain 在这些层面几乎不帮忙，反而引入隐藏抽象，让 debug 时多了一层栈。

**直接用 SDK 的好处**：

1. 工具调用就是 JSON in / JSON out，一目了然
2. 审计直接对应原始 message exchange
3. 模型切换只改一行
4. 不必学一套额外 DSL

**LangChain 适合的场景**：原型期、RAG 重场景、需要快速接入向量库 + 文档加载 + Web search 等组合工具。

**MCP 何时考虑**：当你的工具集需要被多种客户端（Claude Desktop、Codex、第三方）复用时。本系统的工具是私有业务工具，MCP 不是必须。

## 2.4 为什么 Bun（不是 Node 或 Python）

**TypeScript 选型理由**：
- 工具 schema 是 JSON Schema，TS 原生友好
- 业务团队多数会读 TS，不太会 Python
- 后端 / 脚本 / UI 三层全 TS，单语言栈

**Bun 选型理由**：
- 不用 `tsc` / `ts-node` / `swc` 这套链路
- 内置 SQLite（`bun:sqlite`），不用 `better-sqlite3` 编译
- 内置 HTTP，启动 ~50ms
- 内置 spawn 比 Node 快约 3x

**何时该换回 Node**：
- 团队已重度使用 Node 生态
- 需要 Sentry / Datadog 等 Node-only SDK
- Bun 在 Windows 上某个 CI 步骤卡死（实际遇到过 1 次）

**何时该换 Python**：
- 数据分析 / 模型微调路径
- 团队是 ML 背景
- 需要直接调 pandas / numpy

## 2.5 为什么 SQLite（不是 Postgres）

**单进程、单文件**对 MVP 是巨大优势。提案吞吐量计算：

```
假设 1000 工单/天 × 3 张提案/工单 = 3000 写/天 = 0.035 写/秒
SQLite WAL 模式可达 ~10000 写/秒
富余 285,000 倍
```

**何时迁 Postgres**：
- 多个 reviewer 进程并发访问且写竞争出现
- 需要跨进程查询（分析、BI）
- 需要 row-level security
- 法规要求强备份 / PITR

**迁移路径**：表结构 100% 兼容（已避免 SQLite-only 类型），只改 driver。预留接口：

```typescript
// db.ts 把所有 SQL 包在函数里，不在业务代码里写裸 SQL
export function insertProposal(p: ProposalRow) { ... }
export function getProposal(id: string) { ... }
```

迁移时只换实现，业务代码不动。这是**仓储模式（Repository Pattern）**的具体应用，见 Ch 3。

## 2.6 为什么 Alpine.js + 原生 HTML

Reviewer UI 的核心是：列表 / 卡片 / 表单 / 提交。不需要 SPA。

- React/Vue 是 ~200KB JS + 工具链复杂度
- Alpine.js 是 17KB CDN 引入，写起来像 Vue 但零构建
- 一个 reviewer UI 通常迭代频率低（一旦确定操作流程）

**升级触发点**：
- 需要多 tab 实时同步（websocket → 用 Vue/React + Pinia/Redux）
- 需要复杂表单（动态字段、条件渲染、JSON Schema 表单）
- 需要图表 / dashboard（接 ECharts / Recharts）

## 2.7 选型反对：我曾经做错的选择

诚实记录便于后人避雷：

| 早期错误选择 | 翻车原因 | 现在的选择 |
|---|---|---|
| 用 ORM（Prisma）管 SQLite | Prisma 启动慢 + schema 改动需要 migrate | 直接 SQL + 自写 query 函数 |
| 用 LangChain 做主代理 | tool_use 出错时 stack trace 穿 5 层抽象 | 直接 Anthropic SDK |
| 用 Redis 做提案队列 | 提案不是队列任务，是状态机 | SQLite 表 + status 字段 |
| 把 policy 写进 system prompt | 政策变动要 redeploy | policy 写成 markdown，工具加载 |
| 子代理用 OpenAI Functions 风格 | 与主代理混栈，调试痛苦 | 全栈 Claude，统一 message format |
| 用 EventEmitter 模式跨代理通信 | 调试时丢消息 | 同步函数调用 + JSON 返回 |

---

<a id="ch3"></a>

# Ch 3 — 架构模式（命名 + 应用）

> 给工程师 / 面试官 / 架构评审看的内容。系统里用了 9 个经典模式，每个都有明确目的。

## 3.1 命令查询职责分离（CQRS — Command Query Responsibility Segregation）

**用在哪**：技能脚本按读 / 写严格分开，读脚本不可触发任何状态变更。
**为什么**：
- 写路径必须经"提案 + 人审"网关
- 读路径直接代理底层数据源，无任何副作用
- 这种切分让"主代理意外触发写动作"在脚本层就被堵死

**实现层**：脚本目录划分 + 命名约定 + 静态扫描（grep `create-*-proposal.ts` 列举所有写路径）。

## 3.2 仓储模式（Repository Pattern）

**用在哪**：`backend/db.ts` 把所有 SQL 包装为函数（`insertProposal` / `getProposal` / `listProposals`）。
**为什么**：
- 业务代码不知道是 SQLite 还是 Postgres
- 测试时可以 Mock 整个 repository
- 数据库迁移只改 db.ts 一个文件

## 3.3 网关 / 反腐败层（Gateway / Anti-Corruption Layer）

**用在哪**：提案 backend 就是一个 ACL，介于"不可信的 LLM"和"可信的下游业务系统"之间。
**为什么**：
- 下游系统的接口（OMS API、IAM API）是用 30 年前的 SOAP/RPC 风格写的
- 如果让 Agent 直接拼那些请求，每次接口变更都要改 prompt
- Gateway 把"上游用人类语义的提案"翻译成"下游业务命令"
- 改下游接口只改 gateway，不动 Agent

## 3.4 隔板模式（Bulkhead Pattern）

**用在哪**：子代理的工具集物理隔离。
**为什么**：
- 数据子代理崩了，物流子代理还在工作
- 子代理 A 把它的 token 全用完，不影响子代理 B
- 类比：船舱用隔板分隔，一处进水不沉船

**实现**：每个子代理是独立 agent loop，独立 MCP scope / 工具列表，独立模型实例。

## 3.5 断路器模式（Circuit Breaker）

**用在哪**：提案 → 人审 → 下游。人审就是断路器。
**为什么**：
- 模型大量产生不合理提案时，reviewer 自然会拒绝率升高
- 这是"业务回路"的天然断路器
- 比技术断路器更聪明：因为人能识别"看起来正确但实际不对"的提案

**进阶**：可以加一层自动断路器 — 当某 agent 单位时间内提案被拒率 > 50% 时，自动停用并通知工程团队。

## 3.6 策略模式（Strategy Pattern）

**用在哪**：剧本（Playbook）就是不同的"调查与处置策略"。
**为什么**：
- 不同工单类型需要不同的取证顺序、不同的提案路径
- 把策略写成 markdown 文件，主代理动态加载，新增类型无需改代码
- 比 if-else 决策树灵活得多（边角情景模型可以混合多个策略）

## 3.7 责任链模式（Chain of Responsibility）

**用在哪**：主代理 → 委派包 → 子代理 → 返回 → 主代理 → 下一个子代理 → ...
**为什么**：
- 主代理不需要知道每个证据源的实现细节
- 子代理可以"我不会，请换 X 子代理"
- 调试时每环都有结构化日志

## 3.8 模板方法模式（Template Method）

**用在哪**：脚本运行器（`lib/script-runner.ts`）定义脚本骨架，每个具体脚本只需要实现"具体业务"。
**为什么**：
- 强制所有脚本的 flag 解析、错误格式、JSON 输出一致
- 新人写新脚本时不会偏离契约

## 3.9 六边形架构（Hexagonal / Ports & Adapters）

**用在哪**：整个系统的最外层视角。
- **核心域**：政策文档 + 提案数据模型 + 决策框架
- **端口（接口）**：Tool 接口、Proposal 接口、Read 接口
- **适配器**：Anthropic SDK 适配器（可换成 OpenAI）、SQLite 适配器（可换成 Postgres）、HTTP 适配器（可换成 gRPC）、UI 适配器（可换成 Slack bot）

**为什么**：
- 业务核心（政策 + 决策框架）不依赖任何具体技术
- 任何技术层都可替换
- 这是为什么"换模型不改业务代码"成立

## 3.10 这九个模式的关系图

```
                  ┌──────────────────────┐
                  │  Hexagonal（最外层）  │
                  │                      │
                  │  ┌────────────────┐  │
                  │  │ 核心域：政策、    │  │
                  │  │ 提案模型、决策    │  │
                  │  └────────────────┘  │
                  │      ▲    ▲    ▲      │
                  │      │适配│适配│适配    │
                  │  ┌───┴┐ ┌─┴─┐ ┌┴───┐ │
                  │  │LLM ││DB │ │HTTP││
                  │  └────┘ └───┘ └────┘ │
                  └──────────────────────┘

CQRS：读写脚本分离
Repository：DB 操作集中
Gateway：LLM → 业务命令 翻译层
Bulkhead：子代理隔离
Circuit Breaker：人审 + 自动断路
Strategy：剧本作为策略
Chain of Resp.：主→子→子链式委派
Template Method：脚本运行器骨架
```

---

<a id="ch4"></a>

# Ch 4 — 业务 vs 框架的锐利分层

这一节是把系统**移植到新领域时**的核心参考。

## 4.1 总分层表

| 层 | 子层 | 内容 | 移植时怎么办 |
|---|---|---|---|
| **业务层** | 政策文档 | POLICY.md / DECISION_FRAMEWORK.md / Playbooks | **完全重写**（与领域强绑定） |
| | 工具 Schema | `tools.ts` 中每个工具的 name/description/input_schema | **重写**（业务名词不同） |
| | 业务命令白名单 | backend 的 `ALLOWED_COMMANDS` 集合 | **重写**（下游命令不同） |
| | 业务读脚本 | `skills/*/scripts/get-*.ts` | **重写**（数据源不同） |
| | 业务写脚本 | `skills/*/scripts/create-*-proposal.ts` | **重写**（动作不同） |
| | 子代理领域 prompt | `agent/subagents/*.ts` 中的 system prompt | **重写**（证据源不同） |
| | UI 卡片字段 | `index.html` 显示哪些字段 | **微调**（字段映射） |
| **框架层** | Agent Loop | `runPrimaryAgent` 主循环 | **直接复用** |
| | Sub-Agent Loop | 子代理 agent loop | **直接复用** |
| | Proposal Contract | `lib/proposal.ts` 的 schema 与构造函数 | **直接复用** |
| | Read Helper | `lib/read.ts` | **直接复用** |
| | Script Runner | `lib/script-runner.ts` | **直接复用** |
| | Proposal Backend | `backend/server.ts` 与 `db.ts` 的 80% | **直接复用**（只改 ALLOWED_COMMANDS） |
| | DB Schema | `backend/schema.sql` | **直接复用** |
| | UI 骨架 | Alpine.js + 静态 HTML 骨架 | **直接复用** |
| | 委派包结构 | 主代理工具 input_schema 中的字段约定 | **直接复用**（字段名可调整） |
| | 审计模型 | `audit_log` 表与写入点 | **直接复用** |

## 4.2 一条工单走完，业务 vs 框架的贡献占比

| 步骤 | 框架 | 业务 |
|---|---|---|
| 接收工单 | HTTP / queue 框架处理 | 工单字段定义来自业务 |
| 加载政策 | Tool dispatch 机制 | 政策内容 |
| 取证 | 工具调用循环 | 工具 schema + 数据源 |
| 委派 | Sub-agent loop | 委派包字段 + 子代理 prompt |
| 决策 | 模型 + 工具调用机制 | 决策框架 + 剧本 |
| 创建提案 | Proposal contract + HTTP | 提案 schema 中的业务字段 |
| 人审 | UI 骨架 + 审批 API | UI 中显示哪些字段 |
| 下游执行 | Gateway 转发 + 审计 | 业务命令 + 参数 |

**经验值**：移植到新领域时，框架部分代码改动 < 10%，业务部分改动 80%+。

## 4.3 边界测试：移植不动是不是框架，移植要改是不是业务

判断某段代码该放业务还是框架：

> 闭眼问："如果明天换一个完全不同的行业，这段代码要改吗？"
> - 不改 → 框架
> - 改 → 业务

如果答"看情况" — 通常意味着这段代码**职责不纯**，要拆。

例：原本的 `create-refund-proposal.ts` 里有这样一段：

```typescript
const reason = `物流签收异常 GPS 距地址 1.2km，单价 ¥299 在政策范围内`
```

这是业务文案。**不能**写进框架代码。正确做法是 reason 由主代理生成、作为参数传入。

---

<a id="ch5"></a>

# Ch 5 — Anthropic Tool Use API 深度

## 5.1 一次完整调用的 message 拓扑

Anthropic API 是无状态的：每次 `messages.create` 你都得把完整对话历史传上去。这是 Tool Use 模式的核心。

```
请求 1：
{
  "model": "claude-opus-4-7",
  "system": "<system prompt>",
  "tools": [<tool definitions>],
  "messages": [
    { "role": "user", "content": "处理工单 t12345" }
  ]
}

响应 1：
{
  "stop_reason": "tool_use",
  "content": [
    { "type": "text", "text": "好的，开始处理。先加载政策..." },
    { "type": "tool_use", "id": "toolu_01", "name": "read_policies", "input": {} }
  ]
}

请求 2：（你必须把响应 1 完整传回，再加上 tool_result）
{
  ...
  "messages": [
    { "role": "user", "content": "处理工单 t12345" },
    { "role": "assistant", "content": [<响应 1 的 content 数组>] },
    { "role": "user", "content": [
      { "type": "tool_result", "tool_use_id": "toolu_01",
        "content": "POLICY.md 内容..." }
    ]}
  ]
}
```

**关键点**：
1. `assistant` 的 content 必须**原样回传**（包括 text 块和 tool_use 块）
2. `tool_result` 的 `tool_use_id` 必须匹配
3. 一个响应中可以包含多个 tool_use（并行调用）
4. `stop_reason` 是循环退出条件：`end_turn`（结束）/ `tool_use`（继续）/ `max_tokens`（截断）

## 5.2 stop_reason 完整含义

| 值 | 含义 | 你该做什么 |
|---|---|---|
| `end_turn` | 模型说完了，没工具调 | 退出循环 |
| `tool_use` | 模型要调工具 | 执行工具 → 继续循环 |
| `max_tokens` | 输出被截断 | 警告 + 退出 |
| `stop_sequence` | 命中停止序列 | 通常不出现 |
| `pause_turn` | 模型主动暂停（极少） | 通常视为 end_turn |

## 5.3 多工具并发

模型可以在一个响应里输出多个 tool_use（并行）。运行时应该并发执行：

```typescript
if (resp.stop_reason === "tool_use") {
  const toolUses = resp.content.filter(b => b.type === "tool_use")
  // 并发！
  const results = await Promise.all(toolUses.map(async (block) => ({
    type: "tool_result",
    tool_use_id: block.id,
    content: await executeTool(block.name, block.input)
  })))
  messages.push({ role: "user", content: results })
}
```

**Celest 一线经验**：Opus 在中度复杂 case 中，单次响应输出 3-5 个并行工具调用很常见。串行执行会让单 case 处理时间从 30s 涨到 150s。

## 5.4 工具调用错误的处理

工具失败时，把错误塞回 `tool_result.content`，模型会自己处理：

```typescript
toolResults.push({
  type: "tool_result",
  tool_use_id: block.id,
  content: JSON.stringify({ error: "timeout after 10s" }),
  is_error: true,  // Anthropic 推荐显式标记
})
```

模型通常会：
- 重试（如果错误看起来 transient）
- 换工具（如果当前工具用错了）
- 升级（如果错误是 platform 层面）

**反模式**：在工具层 throw exception 让主循环 catch 后无脑重试。模型不知道发生了什么，会重复同样错误。

## 5.5 系统提示的最佳实践

| 实践 | 做 | 不做 |
|---|---|---|
| 长度 | 200-500 行 | > 1000 行（管不过来） |
| 结构 | 编号章节 / 列表 | 大段散文 |
| 工具引用 | 用工具名（必须精确） | 模糊描述 |
| 终止条件 | 明确写"完成后输出 X 然后结束" | 让模型自己猜 |
| 模型角色 | 一段定位（"You are the X agent."） | 玩花活儿（"Pretend you are..."） |
| 例子 | 1-2 个负面例子比 5 个正面例子有效 | 没例子 |
| 政策内容 | 放在工具里动态读 | 全塞 system prompt |

## 5.6 Token / 上下文窗口管理

Opus 4 的上下文是 200K（标准）/ 1M（部分客户）。听起来够，实际很容易塞满：

| 来源 | 典型 token |
|---|---|
| System prompt | 3000-5000 |
| Tool definitions（10 个工具） | 2000-3000 |
| 政策文档（read_policies 一次） | 5000-15000 |
| 5 次工具调用（含工单 / 用户 / 订单 / 子代理输出） | 15000-50000 |
| 中等复杂工单总计 | 30000-80000 |

**控制策略**：

1. **子代理返回精简**：子代理只返回 `finding + evidence_strength + 3-5 条 evidence`，不要把所有 SQL 行数全吐出来
2. **附件单独读**：图片用 `view_image`，不要把 base64 塞进 message
3. **历史压缩**：超过 100K 时让主代理输出"中间总结"，把早期工具调用 prune 掉
4. **超长政策分章节**：read_policies 工具支持 `section` 参数，只读相关章节
5. **监控警报**：在主循环里每轮统计 input_tokens，超过阈值告警

## 5.7 模型选型与成本（2026 年 5 月价格）

| 模型 | 输入价 | 输出价 | 缓存写 | 缓存命中 | 适用 |
|---|---|---|---|---|---|
| Opus 4.7 | $15 / MTok | $75 / MTok | $18.75 | $1.5 | 高难度判断、复杂多步 |
| Sonnet 4.6 | $3 / MTok | $15 / MTok | $3.75 | $0.3 | 主代理日常 |
| Haiku 4.5 | $1 / MTok | $5 / MTok | $1.25 | $0.1 | 轻量子代理、批量分类 |

**推荐配置**：

| 角色 | 模型 | 推理力度 | 月成本估算（10K case） |
|---|---|---|---|
| 主代理 | Sonnet 4.6 | medium | $200-400 |
| Data Investigator | Sonnet 4.6 | high | $80-150 |
| Logistics Investigator | Haiku 4.5 | medium | $20-50 |
| Knowledge Retriever | Haiku 4.5 | low | $10-30 |

**总成本**：~$400-700/月，1 万 case，约 ¥0.3-0.5/案。比一个人工客服小时工资便宜得多。

---

<a id="ch6"></a>

# Ch 6 — Prompt Caching：决定单位经济的关键技术

> 这一节是工程优化里 ROI 最高的单点。**忽视它 = 钱白烧 5-10 倍**。

## 6.1 什么是 Prompt Caching

Anthropic API 允许把 prompt 的某些 prefix 段标记为"可缓存"。下次同样 prefix 请求时：
- **缓存命中**：输入价 = 标准价 × 10% （省 90%）
- **缓存写入**：输入价 = 标准价 × 125% （首次贵 25%）

缓存 TTL：5 分钟（短）/ 1 小时（长，需指定 `cache_control.type: "1h"`）

## 6.2 哪些段最适合缓存

| 段 | 大小 | 复用率 | 是否该缓存 |
|---|---|---|---|
| System prompt | 3-5KB | 100% | ★ 必缓存 |
| Tool definitions | 2-3KB | 100% | ★ 必缓存 |
| 政策文档（POLICY + DECISION + INDEX） | 10-15KB | 100% | ★ 必缓存 |
| 选中的剧本 | 2-5KB | ~30%（按类型） | 可缓存 |
| 工单内容 / 用户档案 | 1-3KB | 0% | 不缓存 |
| 工具调用结果 | 1-20KB | 0% | 不缓存 |

## 6.3 缓存配置代码

```typescript
const resp = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 4096,
  system: [
    {
      type: "text",
      text: SYSTEM_PROMPT,
      cache_control: { type: "ephemeral" },  // 5 分钟
    }
  ],
  tools: TOOL_REGISTRY.map((t, i) => {
    // 在最后一个工具上标 cache_control，整块工具定义都会被缓存
    if (i === TOOL_REGISTRY.length - 1) {
      return { ...t, cache_control: { type: "ephemeral" } }
    }
    return t
  }),
  messages: [...messages],
})
```

要让政策文档也缓存，在第一次 `read_policies` 的 `tool_result` 内容上标 cache_control：

```typescript
{
  type: "tool_result",
  tool_use_id: block.id,
  content: [
    { type: "text", text: policyContents, cache_control: { type: "ephemeral" } }
  ]
}
```

## 6.4 一个具体的成本测算

**未启用缓存的 case**：

```
主代理一次工单：
  System prompt:    5000 tok × $3   / MTok = $0.015
  Tools:            3000 tok × $3   / MTok = $0.009
  Messages 累计:   20000 tok × $3   / MTok = $0.060
  输出:             3000 tok × $15  / MTok = $0.045
  合计：~$0.13/案
```

**启用缓存的 case**（同样工作量，缓存命中）：

```
主代理一次工单：
  System prompt:    5000 tok × $0.3 / MTok = $0.0015 （缓存命中）
  Tools:            3000 tok × $0.3 / MTok = $0.0009 （缓存命中）
  政策内容:        12000 tok × $0.3 / MTok = $0.0036 （缓存命中）
  Messages 增量:    8000 tok × $3   / MTok = $0.024
  输出:             3000 tok × $15  / MTok = $0.045
  合计：~$0.075/案
```

**节省 ~42%**。在每天 1 万 case 的量级，年省 ~$15K-30K。

## 6.5 缓存的陷阱

| 陷阱 | 现象 | 修正 |
|---|---|---|
| 缓存 prefix 中带了 dynamic 字段 | 命中率 = 0 | 把 dynamic 字段移到 prefix 之后 |
| 缓存断点（cache_control）位置错 | 缓存粒度太碎 | 在最大的"会重复"块的末尾打断点 |
| TTL 配错 | 5 分钟反复重写 | 长周期复用块用 `type: "1h"` |
| 工具定义偶尔随机排序 | 命中率不稳定 | 工具列表严格固定顺序 |
| Messages 数组首条变了 | 整个对话缓存失效 | 主代理的首条 user message 用固定模板 |

## 6.6 如何监控缓存有效性

Anthropic API 响应里有：

```json
"usage": {
  "input_tokens": 1500,
  "output_tokens": 800,
  "cache_creation_input_tokens": 0,
  "cache_read_input_tokens": 18000  ← 这是缓存命中的 tokens
}
```

每次响应记录这 4 个数字到审计表。计算 **缓存命中率 = cache_read / (cache_read + input + cache_creation)**。

健康范围：> 70%。如果 < 50%，回到 6.5 排查。

---

<a id="ch7"></a>

# Ch 7 — 数据契约（决定系统寿命）

## 7.1 为什么契约比代码重要

代码每周都改。**数据契约改一次就血流成河**。提案数据契约一旦上线被审计、被 reviewer 看了几个月，再加字段、再改语义，都要走变更管理流程。

**架构师的核心工作**：把契约定义好，让代码可以乱改但契约稳定。

## 7.2 提案 Schema 设计原则

### 7.2.1 字段分类

| 类别 | 字段 | 特点 |
|---|---|---|
| 身份 | id / work_item_id / actor_id | 不可变 |
| 内容 | operation / params / execution | 不可变（创建后） |
| 信道分离 | reason / user_message | reason 仅内部、user_message 仅外发 |
| 状态 | status | 状态机管理 |
| 审计 | created_at / created_by / decided_at / decided_by / executed_at | 只追加 |
| 下游 | downstream_response | 只追加 |

### 7.2.2 状态机

```
pending ──approve──> approved ──execute──> executed
   │                    │
   ├──deny──> denied    └──fail──> failed
   │
   └──supersede──> superseded
```

**不可逆**：denied / executed / failed 都是终态。需要"撤回 + 重做"时新开一张提案，并把老卡 status 改为 `superseded`。

### 7.2.3 不可变性的保证

数据库层面 + 应用层面都要保证：

```sql
-- 数据库：可以加触发器禁止某些字段更新
CREATE TRIGGER prevent_proposal_immutable_update
BEFORE UPDATE OF operation, params, execution, reason ON proposals
BEGIN
  SELECT RAISE(FAIL, "immutable fields cannot be updated");
END;
```

应用层：所有 update 函数只接受白名单字段（status / decided_at / user_message / downstream_response）。

### 7.2.4 版本化

提案 schema 升级时（比如加字段 `risk_level`），不要无痕加。要：

1. 加 `schema_version` 字段（INTEGER，默认 1）
2. 新字段允许 NULL，旧记录不补
3. UI 显示时检查 schema_version 兼容性
4. Reporting / 分析时按 schema_version 分组

## 7.3 委派包 Schema 设计原则

委派包就是主代理 → 子代理的 RPC 数据结构。同样需要稳定契约。

```typescript
interface DelegationPacket {
  // 必填
  claim_to_verify: string         // 要查证的具体声明
  exact_questions: string[]        // 子代理必须回答的问题
  
  // 各专科子代理的领域字段（条件必填）
  user_id?: string
  tracking_no?: string
  order_id?: string
  // ...
  
  // 可选
  case_id?: string
  time_window?: string
  what_would_prove_or_disprove?: { prove: string; disprove: string }
  prior_reads?: Array<{ source: string; result: string }>
  freshness?: string
}
```

**设计要点**：
- `claim_to_verify` + `exact_questions` 是普世必填，定义了"问题"
- 领域字段是"证据锚点"，由各子代理在 input_schema 中声明
- `what_would_prove_or_disprove` 是质量信号 — 主代理能写出来说明它想清楚了
- `prior_reads` 防重复 — 主代理已查过的不要让子代理再查一遍

## 7.4 工具 Schema 设计原则

Tool definition 是模型理解工具的全部信息。

**Description 字段写法**：

| 反例（人类好读但模型理解差） | 正例（结构化） |
|---|---|
| "获取订单" | "Fetch an order's current state: status, items, shipping address, tracking number. Returns null if not found. Use this when you need authoritative current order data; do NOT use for historical order list — use list_user_orders for that." |

关键原则：
1. **第一句**：动词 + 主语 + 简要返回
2. **第二句**：何时用 / 何时不用
3. **第三句（可选）**：常见误用提醒

**input_schema 设计**：

| 维度 | 推荐 | 避免 |
|---|---|---|
| 字段名 | snake_case | camelCase（与 Python 风格混淆） |
| 类型 | 严格类型 + description | 全 string |
| 必填 | `required` 数组 | 全可选 |
| 枚举 | 用 `enum` | 用 description 描述枚举值 |
| 嵌套 | 浅（≤ 2 层） | 深嵌套 |

## 7.5 命令白名单设计

```typescript
const ALLOWED_COMMANDS = new Set([
  "OMS.Refund",
  "OMS.Reship",
  "Ticket.Reply",
  "Ticket.Close",
  "Ticket.Escalate",
])
```

**这一行决定了系统的爆炸半径**。任何不在这个集合的命令，无论模型怎么生成，都被后端拒绝。

新增命令的流程应该是工程任务（PR + Review + 审计），不是配置。

---

<a id="ch8"></a>

# Ch 8 — 每日施工：技术 / 业务 / 易错点对照

对应 `agent-build-tutorial-from-scratch.md` 的 Day 1-10。本节为每天补充：本日技术栈、本日业务部分、本日易错点。

## Day 1 — 政策文档

| 维度 | 内容 |
|---|---|
| 技术 | Markdown，无任何技术栈。文件夹结构 + 命名约定 |
| 业务 | **100% 业务** — 这是整个系统最业务的一天 |
| 框架 | 0 |
| 易错点 | 1. 把决策框架写成 if-else 决策树（应写"证据 → 等级 → 选项"）2. 政策里夹"实施细节"（政策应只写"什么不能做"，怎么做归剧本）3. 文档之间循环引用（应单向引用：剧本 → 决策框架 → 政策） |

## Day 2 — 提案后端

| 维度 | 内容 |
|---|---|
| 技术 | Bun + Hono + bun:sqlite。HTTP API 设计 |
| 业务 | `ALLOWED_COMMANDS` 列表（业务）+ 状态机定义（半业务） |
| 框架 | 80% — schema、insert/select 函数、状态转换都是框架 |
| 易错点 | 1. 允许 multi-step execution（必须强制 = 1）2. ALLOWED_COMMANDS 用数组（应用 Set）3. 写入时不做参数校验（应在边界做）4. 不区分 4xx vs 5xx 错误（4xx 是契约不符，5xx 是后端问题） |

## Day 3 — 人审 UI

| 维度 | 内容 |
|---|---|
| 技术 | 原生 HTML + Alpine.js。无打包，CDN |
| 业务 | 显示哪些字段、字段顺序、视觉强调（业务） |
| 框架 | 列表 / 详情 / 表单 / Approve API 调用模式（框架） |
| 易错点 | 1. 把 user_message 和 reason 显示在一起（应分区强调）2. Approve 没有二次确认（应弹确认）3. 修改后的 user_message 没传到后端（应在 approve API 携带）4. 多人 reviewer 时没显示"谁在看"（应加 lock 或显示其他 reviewer） |

## Day 4 — 共享 SDK

| 维度 | 内容 |
|---|---|
| 技术 | TypeScript 模块、fetch、process.argv 解析 |
| 业务 | 0 |
| 框架 | 100% |
| 易错点 | 1. flag 解析支持 `--key=value` vs `--key value` 不一致 2. emitError 不退出 process（应 exit(1)）3. backend URL 硬编码（应通过环境变量）4. JSON.parse 失败时无错误处理（应 try/catch） |

## Day 5 — 只读技能

| 维度 | 内容 |
|---|---|
| 技术 | 脚本模板。每脚本 ~30 行 |
| 业务 | 数据源 URL + 字段映射（业务） |
| 框架 | 脚本骨架（框架） |
| 易错点 | 1. 没做"not found 视为合法输出"（很多业务场景下，"该用户没有支付记录"是合法事实，不是错误）2. 返回原始接口数据没清洗（应在脚本内做基础清洗，去掉无关字段，省 token）3. 没超时（应统一加 10s 超时） |

## Day 6 — 写技能（提案）

| 维度 | 内容 |
|---|---|
| 技术 | 脚本模板 + 调 proposal API |
| 业务 | `operation` 名称、`params` 字段、`execution.steps[0].command` |
| 框架 | 脚本骨架 + dry-run 模式（框架） |
| 易错点 | 1. dry-run 仍然 POST 到后端（应在 dry-run 时仅打印 payload）2. 把 user_message 拼接进 reason（必须分字段）3. amount 用浮点（必须用 integer cents 防精度丢失）4. 没在脚本层校验"金额上限"（应在脚本层做硬性 guard，比后端再保险）5. 没在 execution 里写 audit_summary（导致审计日志可读性差） |

## Day 7 — 主代理

| 维度 | 内容 |
|---|---|
| 技术 | Anthropic SDK + agent loop + 工具 dispatch + Bun.spawn |
| 业务 | system prompt 的业务部分、工具列表 |
| 框架 | agent loop、并发工具执行、stop_reason 处理 |
| 易错点 | 1. 没并发工具调用（响应时间 5x）2. 没把工具错误回喂模型（模型会假装成功继续）3. assistant content 重组错（应原样回传，不要重新构造）4. 没限 max_iterations（会死循环烧钱）5. 没启用 prompt caching（成本爆 5x，见 Ch 6） |

## Day 8-9 — 子代理

| 维度 | 内容 |
|---|---|
| 技术 | 独立 agent loop + 限定工具集 + 委派包校验 |
| 业务 | 子代理领域 prompt、领域专属工具 |
| 框架 | sub-agent loop、INSUFFICIENT 返回结构、Finding 结构 |
| 易错点 | 1. 子代理也调主代理才有的工具（必须 strict scope）2. INSUFFICIENT 检查放在 system prompt 而不是代码（应代码兜底）3. 子代理输出非 JSON（应在 prompt 里强制"Output ONLY the JSON object"）4. 子代理 token 失控（应在每轮加 input_tokens 警戒）5. 子代理结果不缓存（同样的委派包 5 分钟内可考虑 sub-agent 级缓存） |

## Day 10 — 集成测试

| 维度 | 内容 |
|---|---|
| 技术 | 三个 mock 数据 + 跑主代理 |
| 业务 | 测试 case 的设计（业务专家协助） |
| 框架 | 评估方法（盲审、replay） |
| 易错点 | 1. 测试 case 全是简单 case（应覆盖边角：物流系统挂了、用户身份未知、附件无法读取）2. 不让业务专家盲审 reason（reason 质量决定上线安全）3. 没记录每个 case 的 token / 时间 / 工具调用次数（无法评估单位经济） |

---

<a id="ch9"></a>

# Ch 9 — 易错点百科（按层分类）

> 整理自实际生产经验。每条都附"症状 / 根因 / 治法"三段。

## 9.1 政策层

### P-1：政策与剧本职责混乱
- 症状：剧本里写"绝对不能给超过 ¥2000 退款"
- 根因：硬性红线应该写政策，剧本是"在政策范围内怎么调查与选择"
- 治法：定期 review，把"绝对"语气的句子从剧本搬到政策

### P-2：决策框架写成决策树
- 症状：模型在多重命中或边角情景卡死
- 根因：决策树假设输入可枚举，实际证据组合无穷
- 治法：写"证据强度梯 + 选项"模式，让模型挑而不是匹配

### P-3：政策版本未签名
- 症状：3 个月后审计员问"为什么这张提案是基于当时的 v3 政策？"
- 根因：政策没有版本号 / 时间戳
- 治法：政策文件首行写 `version: 2026.05` 和 git commit hash。提案创建时记录 policy_version 字段

### P-4：合规与运营政策混淆
- 症状：reviewer 不知道某条规则能不能放宽
- 根因：法务硬性要求和运营弹性建议没区分
- 治法：政策分两栏：`[BINDING]` vs `[GUIDANCE]`。前者不可越界，后者允许判断

### P-5：剧本数量增长无序
- 症状：100 多份剧本，主代理读 INDEX 都要 5000 token
- 根因：每来一个新工单类型就开新剧本，没做合并
- 治法：每月做一次"剧本审计"，命中率 < 1% 的合并到大类剧本

## 9.2 模型层

### M-1：模型创建出 multi-step 提案
- 症状：后端返回 400 error
- 根因：模型试图把"退款 + 回复"打包
- 治法：把错误信息回喂模型，prompt 里加"NEVER create multi-step proposals"

### M-2：模型在 user_message 写内部判断词
- 症状：用户收到回复说"经核实您存在欺诈行为"
- 根因：tool description 没强调 reason vs user_message 区分
- 治法：在 create_reply_proposal 的 user_message 参数 description 里反复写"USER-VISIBLE; never include internal disposition like 'verified fraud'"

### M-3：模型幻想数据
- 症状：reason 里写"根据 ClickHouse 查询，用户有 5 次退款" — 但根本没调过相应工具
- 根因：模型从工单内容推测、不去工具验证
- 治法：在 system prompt 里加硬性约束"Every concrete factual statement in reason MUST be backed by a tool call you actually made in this session"

### M-4：模型选错剧本
- 症状：物流类工单走了商品瑕疵剧本
- 根因：INDEX.md 路由优先级不清晰
- 治法：INDEX 里写明优先级顺序，并加"如同时命中多个，按本顺序"

### M-5：模型死循环
- 症状：跑了 50 轮还没结束
- 根因：模型一直查、一直查、不知道何时该决定
- 治法：1) system prompt 加"After gathering core evidence, you MUST make a decision and create proposals" 2) max_iterations 设 30 强制截断

### M-6：模型不调子代理直接回答
- 症状：模型用通识知识回答了一个本该查数仓的问题
- 根因：子代理的工具 description 没说清"我能做什么"
- 治法：在 delegate_to_X 的 description 里写明"USE THIS for ... (do not answer from general knowledge)"

### M-7：模型生成无意义 reason
- 症状：reason = "Per policy, refund approved."
- 根因：缺乏对"什么样的 reason 是好的"的训练样本
- 治法：1) 在 system prompt 里给一个负例 + 一个正例 2) Reviewer 拒绝后把"应该怎么写"加进剧本

### M-8：模型在长 case 后期质量下降
- 症状：前 5 轮挺好，第 20 轮开始乱来
- 根因：context 接近上限，模型注意力分散
- 治法：1) 提前压缩中间过程 2) 监控 input_tokens，超 80K 强制总结 3) 拆 case 为多 session

## 9.3 契约层

### C-1：提案字段 schema 演化无管理
- 症状：UI 显示旧字段时报错
- 根因：加字段没加 schema_version
- 治法：每次 schema 变更必加版本字段，向后兼容

### C-2：工具 description 太短
- 症状：模型频繁选错工具
- 根因："Get user" 这种描述无法区分 get_user / get_user_orders / get_user_payments
- 治法：每个工具至少 3 句话：做什么 / 何时用 / 何时不用

### C-3：必填字段不一致
- 症状：提案脚本里 `--reason` 必填，但 tool schema 没标 required
- 根因：脚本和 schema 各写各的
- 治法：从 schema 生成脚本骨架；或脚本启动时校验 required 与 schema 一致

### C-4：参数类型不严格
- 症状：amount_cents 接收 "29900" 字符串导致下游 OMS 报错
- 根因：从 model input 到下游一路 string 化
- 治法：脚本层 parseInt + Number.isInteger 校验

### C-5：execution 字段太自由
- 症状：模型生成的 `execution.steps[0].parameters` 千奇百怪
- 根因：execution 字段是模型自由写的
- 治法：execution 由**脚本侧根据业务 flag 计算**，模型只传业务 flag。模型不写 execution

### C-6：委派包字段不稳定
- 症状：主代理调子代理时缺关键字段，子代理 INSUFFICIENT
- 根因：主代理 tool input_schema 里某字段没标 required
- 治法：按"领域必填"严格设 required，宁可让主代理多调一次

## 9.4 运行时层

### R-1：Bun 在 Windows + 中文路径报错
- 症状：spawn 子进程时 stdout 是乱码
- 根因：Windows 默认 CP936 编码
- 治法：spawn 时显式 env `PYTHONIOENCODING=utf-8` / `NODE_OPTIONS=...`（视情况）

### R-2：SQLite WAL 文件被误删
- 症状：开发期 git clean -fdx 后数据全没
- 根因：proposals.db / proposals.db-wal / proposals.db-shm 没加 .gitignore
- 治法：.gitignore 加 *.db / *.db-*

### R-3：并发写竞争
- 症状：偶发 "database is locked"
- 根因：多个进程同时写 SQLite
- 治法：1) 启用 WAL 模式 `PRAGMA journal_mode = WAL` 2) 单写者模式，多读者无锁

### R-4：环境变量读取顺序
- 症状：开发环境工作，生产 token 失效
- 根因：`.env` 加载顺序与代码读取时序不一致
- 治法：dotenv 在最 early 加载；启动时 fail-fast 校验必填环境变量

### R-5：长 fetch 没超时
- 症状：偶发请求 hang 死
- 根因：fetch 默认无超时
- 治法：用 AbortController 设 10-30s 超时

## 9.5 运维层

### O-1：审计日志爆磁盘
- 症状：3 个月后磁盘满
- 根因：audit_log 表无清理策略
- 治法：分区或归档老数据；保留 1 年内 hot，旧的转 S3/对象存储

### O-2：成本失控
- 症状：月账单超预期 5x
- 根因：没启用 prompt caching / 子代理无 token 限制 / 死循环烧钱
- 治法：1) Ch 6 配缓存 2) 子代理 max_iter = 10 3) 主代理 max_iter = 30 4) 月度 token 监控告警

### O-3：模型版本切换无回滚
- 症状：升级 Opus 4.7 → 4.8 后质量降，但回不去
- 根因：模型 ID 硬编码
- 治法：模型 ID 从 env 读取，可热切；保留 N-1 版本至少 30 天

### O-4：UI token 泄露
- 症状：浏览器 source 里能看到 `TOKEN = "..."`
- 根因：前端硬编码 token
- 治法：1) 用 session cookie + reviewer 登录 2) 或代理服务把 token 注入后端，前端不见

### O-5：日志包含 PII
- 症状：审计日志里有用户邮箱 / 手机号
- 根因：reason / user_message 含 PII，整条记录
- 治法：1) 日志层做 PII redaction（regex 替换） 2) 长期归档前 hash 处理

### O-6：跨时区时间戳混乱
- 症状：reviewer 看到的时间是 UTC，业务里聊的是本地
- 根因：DB 存 UTC，UI 不转
- 治法：DB 永远 UTC（毫秒 epoch），UI 显示时按 reviewer 时区转

## 9.6 业务 / 法务层

### B-1：政策更新不通知 Agent
- 症状：政策 v4 发布，但 Agent 还在用 v3
- 根因：政策文件存路径与 Agent 读路径脱钩
- 治法：政策 = Agent 工程的一部分，发版即重启 Agent

### B-2：reviewer 权限平等
- 症状：实习生能批 ¥5000 退款
- 根因：approval 接口不区分 reviewer 角色
- 治法：UI + 后端均检查 reviewer.role；高金额必须高级 reviewer 双签

### B-3：reviewer 改写后无审计
- 症状：approve 时 reviewer 改了 user_message，但 audit_log 只记了"approved"
- 根因：approve 时没记录 before/after diff
- 治法：approve API 记录 user_message_original 与 user_message_final

### B-4：误执行的撤销
- 症状：reviewer 误点 Approve，退款已发出
- 根因：approval 不可撤销 / 下游不支持回滚
- 治法：1) UI 增 5 秒 "撤销" 缓冲（类似 Gmail）2) 下游接 idempotency key + 业务侧补偿（reverse-refund 工单）

---

<a id="ch10"></a>

# Ch 10 — 生产工程化

## 10.1 可观测性最小集

| 信号 | 工具 | 频率 |
|---|---|---|
| 提案创建数 / 状态 | DB query → Grafana | 实时 |
| Agent token 消耗 | Anthropic usage API → 自存 DB | 每 case |
| 工具调用频次 | audit_log 聚合 | 每天 |
| 平均处理时间（AHT） | per-case start/end 时间戳 | 实时 |
| 错误率 | 5xx 计数 → Prometheus | 实时 |
| 缓存命中率 | usage.cache_read / usage.input | 每 case |
| Reviewer 拒绝率 | DB 聚合 | 每天 |
| user_message 改写率 | approve API 记录原/新对比 | 每天 |

简易实现：所有数据写一张 `metrics` 表，定期跑 SQL 输出图表 / 推 Slack。

## 10.2 安全与权限

### 鉴权链路

```
Reviewer 浏览器
    │ (session cookie 或 Bearer token)
    ▼
Reviewer Web Server （含 OAuth / OIDC 接入）
    │ (内部 user.role)
    ▼
Proposal Backend （检查 role + scope）
    │ (downstream-token，按命令 scope）
    ▼
Downstream Gateway
```

**关键**：proposal-backend 不应该直接接受 reviewer 的浏览器请求，应通过 reviewer web server 二次校验。

### 权限矩阵建议

| 角色 | 可见提案 | 可批准金额 | 可批准的命令 |
|---|---|---|---|
| junior_reviewer | 自己 + 共享队列 | ≤ ¥500 | 退款 / 回复 / 关单 |
| senior_reviewer | 全部 | ≤ ¥5000 | 全 + 账号变更 |
| manager | 全部 | 不限 | 全 |
| auditor | 全部（只读） | 0 | 0 |

### PII 处理

- 数据库存原始数据，但提供 redacted view
- 日志 / Slack 通知用 redacted 版本
- 长期归档（> 90 天）做 hash + 关键字段单独加密

## 10.3 部署拓扑

| 阶段 | 拓扑 |
|---|---|
| 开发 | 单机 Bun，4 个进程（backend / mock-downstream / mock-data / agent） |
| 灰度 | Docker Compose，加 Nginx 反代，DB 挂载持久卷 |
| 生产 | K8s 1 个 Deployment（agent worker）+ 1 个 Deployment（backend）+ 外接 Postgres + Redis（短缓存） |
| 高可用 | Backend 多副本（无状态）+ Postgres 主从 + Reviewer UI 多副本 + Agent worker 队列消费 |

## 10.4 数据保留策略

| 数据 | 热（< 90 天）| 温（90 天 - 1 年）| 冷（> 1 年）|
|---|---|---|---|
| 提案记录 | DB | DB（归档表） | S3 + 索引保留 |
| 审计日志 | DB | DB（归档表） | S3 不可改 |
| 工具调用日志 | DB | 删除或归档 | 删除 |
| 模型 message dump | 仅 sampled 5% | 删除 | 删除 |

## 10.5 灰度与回滚

```
功能开关（FeatureFlag）层 + 工单分流层

工单进入
    │
    ▼
┌──────────────┐
│  路由策略     │
│ Type=refund? │
│ Risk=low?     │
│ Random 10%   │
└──────┬───────┘
   是  │  否
       ▼
   Agent 处理     ──→ 人工处理
       │
       ▼
   人审 UI
       │
       ▼
   下游执行
```

灰度策略：
1. 单一低风险路由先开 10%
2. 跑 1 周，对比 Agent 路由与人工路由的指标
3. 没问题扩到 50% → 100%
4. 再开下一个路由

回滚：路由策略一行配置，秒级关闭。

---

<a id="ch11"></a>

# Ch 11 — 评估与迭代

## 11.1 评估方法层级

### 11.1.1 离线 Replay
- 拉取历史已结案工单 + 实际 staff 决策
- Agent 在隔离环境跑一遍（不真创建提案，只输出 dry-run payload）
- 业务专家盲读 Agent 的 reason，判断"我会不会批准"
- 计算：盲审通过率 / Agent 与 staff 一致率

### 11.1.2 在线影子（Shadow）
- 真实工单同时给人工处理 + Agent 处理
- 人工的决定真上线，Agent 的提案只入库不执行
- 一周后对比

### 11.1.3 A/B
- 50% 工单走 Agent + 人审
- 50% 工单走纯人工
- 对比 AHT / 用户满意度 / 升级率 / 重复申诉率

### 11.1.4 冠军-挑战者（Champion-Challenger）
- 当前 Agent v3 是冠军
- v4 候选挑战者，在 5% 流量跑
- 双方指标对比，差异显著且占优才提升为冠军

## 11.2 关键指标

| 指标 | 目标 | 报警阈值 |
|---|---|---|
| AI 提案首审通过率 | > 70% | < 50% |
| user_message 改写率 | < 25% | > 40% |
| AHT vs 纯人工对比 | -30% | 0% 或更慢 |
| 合规事故 | 0 | ≥ 1 |
| 平均提案数 / 工单 | 1.5-3 | > 5 或 < 1 |
| 委派包完整率 | > 90% | < 80% |
| 缓存命中率 | > 70% | < 50% |
| 月成本 / case | < 业务设定上限 | 超 20% |
| Reviewer 满意度（调研） | > 7/10 | < 6 |

## 11.3 迭代闭环

```
Reviewer 否决 / 大幅改写
        │
        ▼
   原因分类
        │
        ▼
  ┌─────┴────┐
  │          │
模型不会     政策不清
  │          │
  ▼          ▼
更好的     改剧本或
工具 desc / 决策框架
few-shot     │
  │          ▼
  │       发版 → 重启 Agent
  ▼          │
A/B 测试  ←──┘
  │
  ▼
冠军-挑战者
```

**实操**：每周一固定半小时，复盘上周的 reject / heavy edit 案例。每条都问：
- 这是模型问题（→ 工程改）还是政策问题（→ 业务改）？
- 怎么让 N+1 次遇到时不犯同样错？
- 这个改动应该加在政策、决策框架、剧本、工具描述、还是 system prompt 哪一层？

## 11.4 政策版本化与回滚

```
policy/
├── archive/
│   ├── 2026-Q1/
│   │   ├── POLICY.md
│   │   └── ...
│   └── 2026-Q2/
└── current/
    ├── POLICY.md
    └── ...
```

每次发版：
1. 当前 `current/` 复制到 `archive/yyyy-Qn/`
2. 修改 current
3. policy_version 字段 +1
4. 部署 + 重启 Agent

提案表里记 `policy_version_used`，6 个月后审计能精确还原"那张提案是基于哪版政策"。

---

<a id="ch12"></a>

# Ch 12 — 团队与组织影响

## 12.1 引入这套系统会改变什么

| 维度 | Before | After |
|---|---|---|
| 客服角色 | 起草 + 决定 | 审稿 + 兜底 |
| 客服人数 | N | ~ 0.3-0.5N |
| 平均薪资带 | 初级 | 中-高级（因为决策权更重） |
| 培训成本 | 高 | 低（政策即培训） |
| 政策迭代速度 | 季度 | 月度甚至周度 |
| 合规审计成本 | 高（追个案） | 低（结构化审计链路） |
| 新业务上线时间 | 2-4 周 | 3-7 天（写剧本即上） |

## 12.2 新的组织角色

引入后通常需要：

- **AI Ops 工程师**（1-2 人）：维护 Agent、监控、调 prompt、迭代政策
- **政策架构师**（1 人，通常是资深业务）：维护三层文档，做版本管理
- **Reviewer Lead**：管理 reviewer 团队，做培训与质量校准
- **合规对接人**：每季度审计一次政策与审计日志

## 12.3 与现有团队的磨合

**风险点 1：客服团队抗拒**
- 现象：感觉被"AI 取代"
- 治法：定位为"AI 起草员，您是审稿专家"。AI 做枯燥的取证 + 起草，人做判断 + 兜底。客服薪资同步提升。

**风险点 2：合规 / 法务怀疑**
- 现象："AI 不可控不能上线"
- 治法：把"提案 + 人审 + 审计"链路逐步展示。强调铁律 1：**主代理在物理层无写权限**。让法务深度参与 POLICY.md 的起草。

**风险点 3：产品 / 业务期望过高**
- 现象："AI 都能搞了，为什么还需要这么多人？"
- 治法：展示 AHT 数据 + reviewer 处理能力的上限。强调"人审是断路器，不可省"。

## 12.4 ROI 模型

简化模型：

```
Before:
  10 个客服 × ¥10K/月 = ¥100K/月

After:
  4 个 reviewer × ¥15K/月 + AI 成本 ¥3-5K/月 + 工程维护折算 ¥10K/月 = ¥73-75K/月

每月省 ¥25K，年省 ¥300K
首年实施成本 ~¥500K（工程 + 培训 + 顾问）
回本时间 ~20 个月
```

但**真正价值**通常不在省人，而在：
- 处理能力上限提升 3-5x（业务峰值）
- 政策一致性（不同 reviewer 之间偏差减小）
- 审计链路完整（应对合规检查）
- 知识资产化（政策 + 剧本 + 改进 manifest 都是公司资产）

---

<a id="ch13"></a>

# Ch 13 — 三个垂类的真实适配演练

下面三个例子全部基于本架构，只做"业务层重写"演练。框架层一行不动。

## 13.1 医疗保险理赔自动初审

### 业务画像
- 保险公司，每天 5000 张索赔单
- 每张单要：核保单 / 看医院记录 / 算药品费用合规性 / 出初审结论
- 涉及 PHI（受保护健康信息），合规要求极高

### 三层文档

**POLICY.md（强合规版）**：
- 真相纪律（每条结论必须可追溯到保单条款 / 医院记录）
- HIPAA 红线（PHI 处理、最小披露）
- 自动批准上限（如 < $500 + 标准诊断 + 网络内医院）
- 拒绝必含理由（保单条款引用）

**DECISION_FRAMEWORK.md**：
- 证据等级：直接（病历 + 保单 + 处方匹配）/ 收敛（多源一致）/ 缺失（关键单据未上传）/ 异常（药品超过医师惯用量）
- 镜像问题（同保单同诊断会得到同结论吗？）

**Playbooks**：
- 门诊普通理赔
- 急诊理赔
- 慢病长期用药
- 牙科 / 视光特殊
- 海外就医
- 涉嫌欺诈（连锁医院 / 重复申报）

### 主代理工具

| 工具 | 用途 |
|---|---|
| get_claim | 拿理赔单详情 |
| get_member_policy | 拿保单条款（覆盖 / 免赔 / 限额） |
| get_medical_record | 拿医院上传的病历 / 处方 |
| get_provider_info | 拿医院 / 医师网络资质 |
| check_diagnosis_code | ICD 编码查询 |
| check_drug_formulary | 处方集合规性 |
| create_approve_claim_proposal | 创建批准提案 |
| create_deny_claim_proposal | 创建拒绝提案 |
| create_request_more_info_proposal | 缺单据，要求补交 |
| create_flag_for_audit_proposal | 涉嫌欺诈，转专员 |
| delegate_to_history_investigator | 历史索赔模式 |
| delegate_to_fraud_investigator | 欺诈信号 |

### 子代理

| 子代理 | 工具范围 | 解决什么 |
|---|---|---|
| history_investigator | claims_dw, member_history | 同会员历史索赔、网络医院、累计金额 |
| fraud_investigator | claims_dw, provider_dw, ml_score | 提供商 / 处方 / 时间集群异常 |

### 易错点（行业特异）
- HIPAA：所有日志必须 redact PHI 字段，audit log 加密
- 解释义务：拒绝必含保单条款引用，不能仅说"政策不允许"
- 时效：紧急理赔需 4 小时内出结论，剧本里要标 SLA
- 多方利益：避免 Agent 偏袒任一方

## 13.2 SaaS IT 工单 L1 助手

### 业务画像
- 中型 SaaS 公司，员工 5000，IT 工单 200/天
- 重置密码、激活账号、配置 VPN、安装软件、解锁文件
- 大多是低风险但高频

### 三层文档

**POLICY.md**：
- 数据访问最小化（只读员工本人配置）
- 自动操作上限：密码 reset 必须 MFA 验证、提权请求必须升级
- 离职员工冻结无人审快速通道

**DECISION_FRAMEWORK.md**：
- 证据等级：员工身份验证强弱（OAuth session + MFA / 仅 SSO / 邮件链接）

**Playbooks**：
- 重置密码
- 解锁账号
- VPN 配置
- 软件安装请求
- 文件 / 系统访问申请
- 提权请求

### 工具

| 工具 | 用途 |
|---|---|
| get_employee | 员工资料 / OU |
| get_employee_devices | 设备 / 软件清单 |
| verify_mfa | 验证身份 |
| create_password_reset_proposal | 触发重置流程 |
| create_unlock_account_proposal | AD 解锁 |
| create_grant_access_proposal | 授权（高风险） |
| create_install_software_proposal | 软件分发 |
| delegate_to_compliance_investigator | 高敏感访问审计 |

### 易错点
- 内部威胁：管理员账号 / 高级访问必须双人审批
- 钓鱼防护：身份验证不能仅靠"工单里说我是 X"
- 员工自助：低风险动作可考虑 1-2 步极简 UI 自助流程

## 13.3 金融场景反洗钱（AML）告警分流

### 业务画像
- 银行 / 支付公司，AML 告警系统每天产生 10000+ 信号
- 大部分是误报，小部分需要 SAR（可疑活动报告）
- 涉及监管报送（FinCEN / 央行）

### 三层文档

**POLICY.md**：
- 真相纪律（每个 SAR 必须可追溯到具体交易行为）
- 监管时限（30 / 60 天报送窗口）
- 客户秘密性（不可向客户透露 SAR）
- 自动结案上限（< 极轻微 + 历史无标记）

**DECISION_FRAMEWORK.md**：
- 证据等级：直接（明显结构化转移 / 已知黑名单）/ 模式收敛（多次小额跨境）/ 缺失（无客户履职档案）
- KYC 等级影响：高 KYC 完整度可更激进结案

**Playbooks**：
- 结构化拆分（structuring）
- 跨境异常
- 高风险地区
- PEP（政治敏感人士）
- 黑名单匹配
- 突发大额

### 工具

| 工具 | 用途 |
|---|---|
| get_alert | 告警详情 / 触发规则 |
| get_customer_kyc | KYC 档案 |
| get_transaction_history | 近 N 月交易历史 |
| check_sanctions_list | 黑名单匹配 |
| get_related_accounts | 关联账号网络 |
| create_close_alert_proposal | 误报结案 |
| create_escalate_to_sar_proposal | 升级 SAR 调查 |
| create_freeze_account_proposal | 冻结账户（极高门槛） |
| delegate_to_network_investigator | 关联账号图分析 |
| delegate_to_pattern_investigator | 时序模式分析 |

### 易错点
- 报送时限：超时未结案 = 监管处罚，AHT 监控极严
- 客户保密：客户不能知道 SAR 存在
- 模型偏见：避免地理 / 族群偏见，需做公平性测试
- 监管审计：所有决策链路必须可还原，包括"为什么不升级"

---

<a id="ch14"></a>

# Ch 14 — 写给自己的结尾

如果你只能从本文记住一句话：

> **Agent 系统的本质是"权限工程 + 数据契约工程"，不是"提示工程"。**

如果你能记住三句话：

1. **架构层防呆**：主代理在物理层无写权限。Prompt 约束在长 session 中会被模型自己说服掉，MCP / 工具集物理隔离不会。
2. **三份文档大于一份 prompt**：政策、决策框架、剧本生命周期不同、责任人不同。混进 prompt 等于失去工程化。
3. **提案-审批网关 + 单步原子化**：reviewer 是断路器，单步是粒度。这两个一起决定了系统的爆炸半径。

如果你能记住五件事：

1. 先写三层文档，再写代码
2. 启用 Prompt Caching（不启用 = 钱白烧 5 倍）
3. 子代理按"证据源"切，不按"业务"切
4. 业务 vs 框架的边界要锐利（"换行业要不要改"判 5 秒）
5. 每条 reviewer 否决都是政策迭代的输入

---

## 配套文档导航

| 想要 | 读 |
|---|---|
| 跑通一份能演示的代码 | `agent-build-tutorial-from-scratch.md` |
| 理解 Celest Support 真实做法 | `celest-support-replication-guide.md` |
| 简历用 / 面试用 / 团队讲方法论 | `agent-resume-and-playbook.md` |
| 架构选型 / 设计决策 / 易错百科 | 本文 |

四份配合起来，覆盖"实操 → 复刻 → 包装 → 架构"四个层次。

---

最后：

> **本架构不依赖 Anthropic，不依赖 Bun，不依赖任何具体技术栈**。
>
> 它依赖的是一个判断：**"在 LLM 与不可逆动作之间，必须有结构化的人审网关"**。
>
> 任何认同这个判断的工程师，都能用任何技术栈搭出来。

