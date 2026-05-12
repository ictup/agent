# AI Agent 搭建：简历素材 + 通用攻略

> 把 Celest Support 的工程经验抽象成与领域无关的方法论。
> 包含可直接放进简历 / LinkedIn 的项目条目，以及"下次再搭一个 Agent"时的通用蓝图。

---

## Part 1 — 简历条目（三个版本，任选）

### 1.1 短版（主简历用，~150 字）

**LLM 多代理客服系统 · 主导设计与实施 · YYYY-MM ~ 至今**

为 [公司] 客服与运营业务设计了一套基于 LLM 的多代理协作系统，处理高敏感账户、交易、合规类工单。采用"分层政策文档 + 受限子代理 + 提案-人审网关"三段式架构：主代理无下游写权限，所有 mutation 都需通过结构化提案卡经人工审批后由网关执行下游业务命令，从架构层保证 LLM 幻觉不能直接放行到生产。系统上线后将人工处理时间降低 X%，AI 提案首审通过率 Y%，合规事故 0 起。

---

### 1.2 详版（项目专栏 / 作品集用，~350 字 + bullets）

**LLM 多代理客服 / 运营辅助系统**　|　主导设计与实施　|　YYYY-MM ~ 至今

业务背景：日均工单 N 张，涉及账户、支付、退款、安全、合规等高风险场景；目标是用 LLM 把客服从"打字员"升级为"审稿人"，同时把幻觉与误操作的可能性压到 0。

**核心架构**：

- **三层规则文档** —— 绑定政策（合规底线）/ 决策框架（六级证据梯 + 镜像问题）/ 领域剧本（~40 份业务路由），把"客服怎么判断"从隐性经验沉淀为可审计的文本资产
- **读写分离的技能 SDK** —— TypeScript + Bun 实现 6 大技能 / 100+ 脚本，统一 `--dry-run` / 错误 JSON 契约 / 后端发现 / 鉴权头，覆盖工单生命周期、用户档案、账户与支付动作
- **MCP 物理隔离的 5 个专科子代理** —— 数据 / 日志 / 崩溃 / 知识库 / Bug 提单，每个子代理只持有自己领域的工具 token；主代理通过结构化"委派包"调用，证据收集与决策严格分离
- **提案-审批后端 + Web UI** —— 所有 mutation 必须先生成提案卡（payload 在脚本侧算好、不可后端模板插值），经人审后由后端代为调用下游业务网关；审计链路覆盖脚本、模型、审批人、下游响应
- **改进闭环** —— 把代理与人审过程中发现的工作流问题沉淀为版本化 manifest，作为下一轮模型/流程迭代的输入

**结果**（请按你的实际数据填写）：

- 人工处理时间 ↓ X%，AI 提案首审通过率 Y%
- 合规事故 0 起；下游业务系统未出现 LLM 直接触发的误操作
- 沉淀 ~40 份领域剧本、N 份改进 manifest，形成可被新人/新模型复用的知识资产
- 主代理 prompt 不到 500 行，业务复杂度由文档与子代理承载，可在不重训提示词的前提下增量上线新业务路由

**技术栈**：TypeScript · Bun · Claude / GPT 多模型协作 · MCP · OLAP（数仓查询）· 集中式日志 · APM 崩溃监控 · OAuth · SQLite · 工程跟踪系统集成

---

### 1.3 英文版（LinkedIn / 海外简历）

**LLM Multi-Agent Customer-Operations System**　|　Architect & Lead Implementer　|　YYYY-MM – Present

Designed and shipped a multi-agent LLM system that handles high-sensitivity account, payment, and compliance tickets. The architecture — *layered policy docs + scoped specialist sub-agents + a proposal-then-human-approval gateway* — gives the primary agent **zero direct write access**: every mutation is materialized as a structured proposal card that a human reviewer approves before a backend gateway executes the downstream business command. This makes LLM hallucinations architecturally unable to reach production.

Highlights:

- **Three-layer rule corpus**: binding policy / decision framework (six-rung evidence ladder, mirror-test questions) / ~40 domain playbooks — turns tacit support judgment into auditable artifacts
- **Read/write-separated skill SDK** (TypeScript + Bun): 6 skills, 100+ scripts, uniform `--dry-run` contract, JSON error envelope, backend discovery, token auth
- **Five MCP-scoped specialist sub-agents** (data / logs / crashes / knowledge / bug-filing) — physical tool isolation, structured delegation packet, standardized "Finding / Evidence Strength / Evidence / Queries / Caveats" return shape
- **Proposal + Approval backend** (Bun, SQLite, web UI): single-step execution contract, immutable payloads, full audit trail
- **Outcomes**: ↓X% human handling time; Y% AI-proposal first-pass approval rate; zero compliance incidents; ~40 playbooks and N improvement manifests as durable knowledge assets

**Stack**: TypeScript, Bun, Claude / GPT multi-model orchestration, MCP, ClickHouse-class OLAP, centralized log aggregation, APM crash analytics, OAuth, SQLite, issue-tracker integration.

---

## Part 2 — 通用框架（彻底去领域化）

### 2.1 一句话定位

> **"在不可信的 LLM 与不可逆的业务系统之间，插入一层结构化提案 + 人审的护栏；用领域无关的多代理拓扑把证据与决策分开。"**

这套架构不是"AI 客服"独有的。任何一个有以下特征的业务都能套：

- 大量结构化但需要判断的工单 / 工作项
- 涉及不可逆 / 高金额 / 合规敏感的下游动作
- 有结构化证据源（数仓、日志、文档、APM）可被代理读取
- 业务侧愿意"人审"但不愿"人草"

### 2.2 通用架构图

```
┌──────────────────────────────────────────────────────────────┐
│                  Primary Agent (read-only)                   │
│   人格 / 政策锚点 = 三层规则文档（强制开局读）                │
│   工具面 = 4-6 个面向业务流程的技能（读 + 创建提案）           │
│   职责 = 调查、判断、起草；零下游写权限                       │
└──────┬─────────────────────────────┬─────────────────────────┘
       │ 结构化委派包                  │ 创建提案 (HTTP POST)
       ▼                              ▼
┌──────────────────────┐      ┌───────────────────────────────┐
│  N 个专科子代理        │      │  Proposal/Approval Backend     │
│  按"证据源"切，不按业务切│      │  - 单步执行契约               │
│                      │      │  - payload 在脚本侧算好         │
│  Data Investigator    │      │  - 入库 + 审计                │
│  Log Investigator     │      └──────┬────────────────────────┘
│  APM Investigator     │             │
│  Knowledge Retriever  │             ▼
│  Escalation Filer     │      ┌───────────────────────────────┐
└──────┬───────────────┘      │   Human Reviewer (Web UI)       │
       │                       │   - 看 reason / player_message │
       ▼                       │   - 可编辑 player_message      │
┌──────────────────────┐       │   - Approve / Deny             │
│ 数仓 / 日志 / 文档 /    │       └──────┬────────────────────────┘
│ APM / 跟踪系统         │              │
└──────────────────────┘              ▼
                              ┌───────────────────────────────┐
                              │   Downstream Business Gateway  │
                              │   （账号系统 / 支付 / OMS / IAM）│
                              └───────────────────────────────┘
```

### 2.3 名词映射表（套到不同行业）

| 通用术语 | 客服场景 | 法务/合规 | IT 运维 | 金融风控 |
|---|---|---|---|---|
| Primary Agent | 智能客服 | 合规初审 | L1 处理 | 风控初判 |
| Work Item | 工单 | matter / 案件 | incident | 报警 / 待决 |
| Policy 文档 | 合规底线 | 法规手册 | 运行约束 | 风控政策 |
| Decision Framework | 客服判断协议 | 法律意见框架 | 故障决策树 | 风险评级框架 |
| Playbook | 工单剧本 | 案件类型 | runbook | 风险路径 |
| Data Investigator | 数仓查证 | 文档检索 | 监控聚合 | 数据特征查证 |
| Log Investigator | 服务端日志 | 系统审计 | 集中式日志 | 交易明细 |
| APM Investigator | 客户端崩溃 | n/a | APM | 异常交易追踪 |
| Knowledge Retriever | FAQ / 手册 | 判例搜索 | runbook 库 | 政策库 |
| Escalation Filer | Bug 提单 | 律师介入 | L3 升级 | 升级人审 |
| Downstream Gateway | 账号/支付系统 | DMS | AD / 工单平台 | 核心交易系统 |
| Proposal Card | 客服动作卡 | 拟办意见 | 工单建议 | 处置建议 |
| Human Reviewer | 客服主管 / staff | 律师 / 法务 | SRE on-call | 风控经理 |

### 2.4 三层文档的通用形态

| 层 | 通用职责 | 必含部分 | 复刻提示 |
|---|---|---|---|
| **Policy（绑定政策）** | "什么绝对不能做" | 真相纪律 / 责任划分 / 阶梯式惩处 / 高金额阈值 / 不可逆动作清单 / 善意补偿条件 | 写**边界**，不写流程 |
| **Decision Framework（决策框架）** | "证据不全时怎么判" | 证据等级梯（建议 5-6 级）/ 真相门（claim/fact/inference/gap 分类）/ 数据源优先级 / 镜像问题（"换一个用户会得到同样结果吗？"） | 把判断**显式化**为可审计的步骤 |
| **Playbook（领域剧本）** | "这类问题该去哪找证据" | Use When / Core Evidence / Delegate When / Resolution Paths / Escalate When | 30-50 份起步；不要写成 if-else |

> **三份文档是 Agent 的 "DNA"**，比 system prompt 更重要。它们也是新人 onboarding、新模型迁移、合规审计的共同语言。

### 2.5 子代理的通用切分原则

不是按"业务"切，而是按"**证据源 / 工具集**"切：

| 子代理 | 持有工具 | 单一职责 |
|---|---|---|
| Data Investigator | 数仓 SQL / 分析脚本 | 历史 / 结构化数据问题 |
| Log Investigator | 集中式日志 API | 服务端事件 / 时序问题 |
| APM Investigator | 崩溃 / APM 平台 | 客户端 / 性能异常 |
| Knowledge Retriever | 文档库 / 知识图谱 / 源码 | 静态知识、定义、机制 |
| Escalation Filer | 工程 / 法务 / 升级单系统 | 把已有证据打包提交给下一环 |

**通用契约**：

- 主代理发**委派包**（claim / time window / known IDs / exact question / what proves-or-disproves / allowed sources）
- 子代理校验委派包；不全则返回 `INSUFFICIENT_DELEGATION_PACKET` + 缺失字段
- 返回结构统一：`Finding / Evidence Strength / Evidence / Queries / Caveats`
- **不做业务判断**，结尾固定一句 *"You provide facts. The primary agent decides the outcome."*

### 2.6 提案-审批网关的通用契约

- 一张提案卡 = 一次下游调用（强制单步，组合操作开多张卡）
- 脚本侧把所有参数计算完毕，**后端不做模板插值**
- payload 不可变；如要改文案，开新卡 supersede 老卡
- 必含字段：`work_item_id` / `actor_id` / `operation` / `params` / `reason`（内部）/ `user_message`（外部）/ `execution`
- 审计链路：脚本 → 提案 → 审批人 → 下游响应 → 状态变更，全留存

---

## Part 3 — Agent 搭建攻略（下一次复用）

### 3.1 第一性问题：到底要不要这套架构？

**该用**的信号：

- 业务存在**不可逆 / 高金额 / 合规** 动作
- 有**结构化证据源**（数仓、日志、文档、APM）
- 团队接受"AI 起草、人类批准"的协作模式
- 工单 / 工作项有**重复模式**，但每张都需要判断
- 合规要求**可审计链路**

**不该用**的信号：

- 所有动作都低风险、可回滚 → 直接全自动化
- 没有结构化证据，只能靠对话 → 普通聊天 bot 就够
- 团队期望"零人审" → 这套架构毫无意义
- 单价低 / 量极大 → 人审会成本爆炸

### 3.2 搭建顺序（强烈推荐）

| 阶段 | 时长 | 产物 | 检查点 |
|---|---|---|---|
| **0. 业务规则梳理** | 1 周 | Policy + Decision Framework + 5-10 份 Playbook（纯 markdown） | 找 2 位资深业务人盲读：他们的判断能否在文档里找到落脚点？ |
| **1. 提案后端 + 审批 UI** | 1-2 周 | `POST /proposals` 入库 + 简单 web UI | 能创建 / 列出 / 批准 / 否决一张测试卡，链路落地 |
| **2. 共享 SDK** | 1 周 | `proposal.ts` / `read.ts` / 脚本运行器 / 模板 | 用模板 30 分钟生成一个新脚本，跑 dry-run 通过 |
| **3. 主代理面向业务的技能** | 2-3 周 | 4-6 个技能，每个 5-15 个脚本 | 业务 5 个高频场景能跑通 |
| **4. 子代理** | 1-2 周 | 3-5 个子代理（按证据源切） | 主代理发委派包，子代理返回标准结构 |
| **5. 主代理 prompt + 回放** | 1 周 | 完整 `agent.md` + 30 个历史 case 回放 | 2 位人审盲读 reason 通过率 ≥ 80% |
| **6. 灰度上线** | 1-2 周 | 单一低风险路由先开 | 提案首审通过率、改写率、停留时间监控 |

**核心原则：先建护栏，再加能力。** 反过来做必定埋雷。

### 3.3 关键决策点（设计时易踩坑）

| 决策 | 推荐 | 理由 |
|---|---|---|
| 主代理写权限 | **零** | 物理隔离 > Prompt 约束 |
| 子代理切分维度 | 按**证据源**切 | 按业务切会和主代理职责重叠 |
| 提案执行步数 | **强制单步** | 多步 = 部分失败的灾难 |
| 内部 reason 与外部 message | **强制分离** | 同一字段共用一定会出事故 |
| Playbook 数量 | **30-50 份起** | 少了边角情景会编 |
| 主代理 prompt 长度 | < 500 行 | 业务复杂度交给文档与子代理承载 |
| 多模型 | **主代理 / 子代理可异构** | 子代理用更强推理（xhigh），主代理更稳 |
| 工具暴露方式 | **MCP scoped token** | 每个 agent 拿到的工具列表是物理限定的 |
| Dry-run | **强制实现** | 没 dry-run = 审稿人没法预览 |
| 错误格式 | **JSON 错误信封** | 主代理能稳定 parse，比 stack trace 友好 |

### 3.4 衡量成熟度的客观指标

把这些指标做进 dashboard，简历里就能引用真数据：

| 指标 | 含义 | 健康范围 |
|---|---|---|
| **AI 提案首审通过率** | reviewer 不修改直接 Approve 的比例 | > 70% 优秀 |
| **player/user_message 改写率** | reviewer 修改外发文案的比例 | < 25% 优秀 |
| **平均处理时间 (AHT) 下降** | 同业务在引入前后的对比 | -30% 起跑线 |
| **合规事故** | 误操作 / 数据泄露 / 越权 | **必须 = 0** |
| **平均提案数 / 工单** | 一张工单创建的提案卡数 | 1.5 - 3 健康 |
| **委派包完整率** | 子代理收到的包未触发 INSUFFICIENT 的比例 | > 90% 优秀 |
| **证据强度分布** | direct proof / converging / gap 的比例 | direct + converging > 70% |
| **回放盲审通过率** | 历史 case 离线回放 + 人审盲读 | > 80% 上线门槛 |
| **Playbook 命中率** | 工单能匹配到剧本的比例 | > 85% 表示覆盖完整 |

---

## Part 4 — 面试 / 答辩 FAQ

> 简历里这种"AI agent 系统"项目，面试官最常问的几类问题。先想清楚答案。

**Q1：为什么主代理不直接写下游？**
A：LLM 幻觉无法事前消除，只能事后拦截。在不可逆动作上做事后拦截 = 已经出事了。物理隔离主代理的写权限 + 强制人审 + 提案单步原子化，是把"幻觉到事故"的距离最大化。Prompt 约束在长 session 里会被代理自己说服掉，MCP 工具隔离不会。

**Q2：为什么要切多个子代理而不是一个大代理？**
A：三个原因。（1）**工具集物理隔离** — 每个数据源的 token / scope 不一样，混在一起的 agent 在权限管理上失控。（2）**推理力度可异构** — 历史查证可以给 xhigh，主代理保持稳态。（3）**故障域隔离** — 数仓挂了不影响日志查证，主代理可以选择继续或挂起。

**Q3：为什么三份文档要分开写，而不是写一个大 prompt？**
A：三份文档承担不同生命周期：Policy 由合规 / 法务定义，半年改一次；Decision Framework 由业务负责人定义，季度迭代；Playbook 由前线运营沉淀，每周长。混进 prompt 后无法分别 review、无法版本控制、无法被新人单独阅读。把它们留在 markdown 里，反而是更好的工程实践。

**Q4：怎么衡量这套系统是否在"帮倒忙"？**
A：监控**改写率**和**首审通过率**这两个指标。首审通过率高、改写率低 = 代理真的在帮忙；首审通过率高、改写率也高 = staff 在被 AI 牵着走，是危险信号；首审通过率低 = 代理在浪费 reviewer 时间。再加上**事故率必须为 0** 的硬约束。

**Q5：怎么避免代理因为 case 复杂就乱发证据？**
A：（1）委派包契约强制结构化；不全直接 `INSUFFICIENT_DELEGATION_PACKET`，不让子代理猜。（2）子代理返回结构里强制带 **Evidence Strength** 标签和 **Caveats**。（3）Decision Framework 里要求主代理在提案前做 **claim audit**：把每句玩家可见的具体陈述列出来、标来源；没来源的删掉。

**Q6：你最大的设计后悔是什么？**（典型行为面试题）
A（建议回答框架）：早期我把"工作流改进"的 capture 机制做晚了 — 前几周代理犯的同类错没有沉淀渠道，到后来发现要回头补 40 份 manifest。下次会把 `self-improvement` 通道放在 MVP 里，与主代理同步上线。

**Q7：怎么和现有客服 / 运营团队磨合？**
A：（1）先把代理定位为"起草员"而不是"决策者" — 这一点要写进系统第一份文档里。（2）灰度上线只开 1-2 条低风险路由，让 staff 用一两周建立信任。（3）每周读 reject 案例（reviewer 否决 / 大幅改写的），把模式回填到 Playbook。

**Q8：如果让你重新做一遍，会变什么？**
A 候选答案：
- 三份规则文档**优先于代码**这个原则不变
- 子代理孵化机制提前到 MVP（之前是 Phase 4，会调到 Phase 2）
- 提案后端的审计字段从一开始就齐全（事后补审计字段很痛）
- 给 reviewer 一个"批量批准低风险类型"的视图（如果业务允许）

---

## Part 5 — 最容易忘的几条软原则

1. **写文档 > 调 prompt**。规则用 markdown 承载，模型可替换、新人可读、合规可审。
2. **物理隔离 > 文字约束**。工具暴露面用 MCP scope 控，不要靠 prompt 让代理"自觉不做"。
3. **多张小卡 > 一张大卡**。原子化让人审能精确否决某一步。
4. **dry-run 是核心 API**，不是开发便利。
5. **数据源切代理，不要业务切代理**。
6. **Reviewer 的可读性 = 系统的可维护性**。reason 字段如果三个月后看不懂，代理就在制造债。
7. **失败要分类**：业务证据不足 vs 平台故障 vs 代理幻觉，三类的处置完全不同，要在协议里显式分开。

---

简历条目用 Part 1，自己看的方法论用 Part 2-5。
项目数据（X%、Y%、N、AHT）记得换成你自己的真实数据再交出去。
