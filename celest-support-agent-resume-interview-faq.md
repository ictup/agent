# Celest Support 垂直领域 Agent 项目简历包装与面试 FAQ 文档

版本：2026-05-12  
适用对象：希望将 Celest Support Agent 搭建经验写入简历，并在面试中系统展示垂直领域 Agent 架构能力的候选人  
目标：把“搭建 Celest Support 智能客服 Agent”的经验包装成有工程深度、有业务理解、有安全意识、有可落地性的项目经历

---

## 目录

1. [项目在简历中的定位](#1-项目在简历中的定位)
2. [推荐项目名称](#2-推荐项目名称)
3. [一句话项目描述](#3-一句话项目描述)
4. [中文简历 Bullet Points](#4-中文简历-bullet-points)
5. [英文简历 Bullet Points](#5-英文简历-bullet-points)
6. [如何体现对垂直领域 Agent 的深刻理解](#6-如何体现对垂直领域-agent-的深刻理解)
7. [面试开场介绍模板](#7-面试开场介绍模板)
8. [面试 FAQ](#8-面试-faq)
9. [简历关键词](#9-简历关键词)
10. [面试中应避免的说法](#10-面试中应避免的说法)
11. [最终推荐简历成稿](#11-最终推荐简历成稿)
12. [核心表达主线](#12-核心表达主线)

---

# 1. 项目在简历中的定位

如果要把 Celest Support Agent 搭建经验写入简历，核心目标是：**不要把它写成“我做了一个客服机器人”**。

更好的定位是：

```text
我设计并落地了一套面向高风险业务场景的垂直领域 Agent 工作流系统。
```

这个项目最有价值的地方不是“能聊天”，而是把大模型安全地嵌入真实客服业务流程，形成：

```text
工单理解
领域知识
工具调用
证据调查
处理建议
人审提案
后端执行
审计追踪
```

的完整闭环。

面试时要突出：

- 垂类 Agent 不是简单 Prompt。
- 真实业务里安全边界比模型能力更重要。
- Agent 应该基于证据和流程，而不是自由发挥。
- 高风险动作必须 human-in-the-loop。
- 生产级 Agent 要有权限、审计、评估和测试体系。

---

# 2. 推荐项目名称

## 中文项目名

可以使用：

```text
垂直领域智能客服 Agent 平台
内部客服 AI 调查工作台
高风险客服场景 AI Agent 工作流系统
面向复杂客服工单的 AI 调查与审批平台
```

如果涉及公司内部或敏感业务，不建议直接写真实产品名，可以写成：

```text
面向大型线上虚拟经济系统的内部客服智能体平台
```

## 英文项目名

可以使用：

```text
Vertical AI Support Agent Platform
Human-in-the-loop AI Support Investigation System
Domain-specific Customer Support Agent Platform
AI-powered Internal Support Workbench
Risk-controlled Support Agent Workflow Platform
```

---

# 3. 一句话项目描述

## 中文详细版

```text
设计并实现了一套面向复杂客服场景的垂直领域 AI Agent 系统，将 LLM 推理、领域知识库、工具调用、权限控制、人审提案流和客服工作台集成为完整闭环，支持 AI 自动调查工单、调用内部工具收集证据、生成可审计处理方案，并通过人工审批安全执行高风险操作。
```

## 中文简洁版

```text
搭建了一个面向内部客服的垂直领域 AI Agent 平台，支持工单理解、证据调查、工具调用、处理方案生成、人审提案流与安全执行闭环。
```

## 英文版

```text
Designed and implemented a domain-specific AI support agent platform that integrates LLM reasoning, tool calling, policy playbooks, human-in-the-loop proposal approval, and auditable backend execution for complex customer support workflows.
```

---

# 4. 中文简历 Bullet Points

## 4.1 通用项目经历版本

```text
垂直领域智能客服 Agent 平台
- 设计并落地面向复杂客服场景的 AI Agent 工作流，将工单理解、领域知识库、工具调用、证据调查、处理建议、人审提案与后端安全执行串联为完整闭环。
- 构建基于 Proposal 的人机协作安全机制：Agent 仅负责调查与生成处理方案，所有涉及账号、资产、经济、工单状态变更的操作必须经人工审批后由后端受控执行，避免 LLM 直接执行高风险动作。
- 设计领域 Skill / Tool 体系，将客服工单读取、玩家资料查询、账号状态调查、物品流转追踪、历史经济数据查询、附件分析、回复生成等能力封装为可审计工具，提升 Agent 在垂直场景中的可控性和可复用性。
- 搭建 Policy / Playbook 驱动的决策框架，使 Agent 能够根据不同 case 类型执行标准化调查路径，降低幻觉与随意判断风险，确保输出符合业务规则与客服口径。
- 实现前后端联动的 Agent 工作台，包括工单列表、工单详情、Agent 调查会话、Proposal 卡片、审批/拒绝、玩家回复编辑、状态实时同步与审计记录。
- 设计最小权限与安全隔离机制：Agent 不持有高权限后台凭证，只能通过 scoped token 访问当前工单范围内的白名单工具；后端对工具来源、操作类型、目标账号、执行命令进行多层校验。
- 建立测试与验收体系，覆盖 skill 脚本、proposal 创建、非法操作拒绝、审批执行、玩家回复格式、场景闭环等关键路径，保障 Agent 在高风险客服场景中的稳定性。
```

## 4.2 偏工程落地版本

```text
- 基于 TypeScript / React / Bun / Docker 构建内部客服 Agent 平台，后端提供 `/support/*` API、SSE 状态同步、Proposal Store、审批执行器和内部工具代理，前端提供客服工作台与提案审批界面。
- 将 Agent 能力拆分为多个 domain-specific skills，包括 support case、player information、player actions、account operations、bug report 等模块，并通过脚本契约和 allowlist 约束工具调用边界。
- 设计 data-driven execution model，Proposal 不直接依赖 operation 分支执行，而是通过受控 `execution.steps` 描述后端动作，提升扩展性、审计能力和安全校验能力。
```

## 4.3 如果有指标，可以这样写

如果你有真实指标，可以加入：

```text
- 将客服复杂工单的初步调查时间从 [X 分钟] 降低至 [Y 分钟]，覆盖 [N] 类高频 support 场景，沉淀 [N] 个可复用调查工具与 [N] 条标准化处理路径。
```

如果没有正式指标，不要编造，可以写：

```text
- 显著减少客服在多系统间手动查询、复制证据和组织回复的成本，并为后续自动化客服流程建立了可扩展、安全可审计的 Agent 架构基础。
```

---

# 5. 英文简历 Bullet Points

```text
Vertical AI Support Agent Platform
- Designed and implemented a domain-specific AI support agent platform that combines LLM reasoning, policy playbooks, tool calling, human-in-the-loop approval, and auditable backend execution.
- Built a proposal-based safety architecture where the agent investigates cases and drafts actions, while all high-risk mutations are reviewed and approved by human operators before backend execution.
- Designed a modular skill/tool system for case retrieval, player/account investigation, item history tracing, economic history lookup, attachment inspection, support replies, and account operations.
- Implemented least-privilege access controls: the agent uses scoped case tokens and whitelisted tools only, while privileged backend operations require operator authentication and proposal approval.
- Integrated the agent into an internal support workbench with case lists, case detail views, embedded agent sessions, proposal cards, editable player replies, approval/denial actions, real-time state updates, and audit trails.
- Created a policy/playbook-driven decision framework to reduce hallucination and ensure consistent support decisions across different case categories.
- Established validation and testing coverage for tool scripts, proposal creation, approval execution, invalid action rejection, response formatting, and end-to-end support scenarios.
```

---

# 6. 如何体现对垂直领域 Agent 的深刻理解

面试官真正想听的不是“我会调 OpenAI API”，而是你是否理解：

```text
垂类 Agent = 业务流程建模 + 权限边界 + 工具系统 + 领域知识 + 人机协作 + 评估体系 + 工程集成
```

下面 8 点是展示深度的核心。

## 6.1 不是聊天，而是业务流程自动化

可以这样说：

```text
我没有把它设计成单纯问答机器人，而是把客服流程拆成了多个可控阶段：读取工单、识别诉求、拆解 claim、查证据、匹配 policy/playbook、生成 proposal、人审、执行、审计。LLM 只是推理层，真正的系统价值来自流程编排和工具闭环。
```

## 6.2 垂类 Agent 必须有领域知识

可以这样说：

```text
通用模型不知道内部客服政策、账号系统、经济系统和操作边界，所以我把规则沉淀为 policy、decision framework 和 case playbooks，让 Agent 不依赖自由发挥，而是按领域流程做判断。
```

## 6.3 工具调用必须受控

可以这样说：

```text
我没有给 Agent 任意 API 或数据库访问权限，而是设计了 skill 层，把内部能力封装成明确的脚本，每个工具有固定输入输出、适用场景和安全边界。
```

## 6.4 高风险场景不能全自动

可以这样说：

```text
客服场景涉及账号、资产、虚拟经济和安全操作，所以我采用 human-in-the-loop proposal 模型。Agent 只生成建议和结构化执行计划，最终必须由人工审批。
```

## 6.5 权限设计比 prompt 更重要

可以这样说：

```text
Prompt 只能约束模型意图，不能作为安全边界。真正的安全边界应该在后端，通过 token scope、allowlist、script contract、account/case 校验和审批机制实现。
```

## 6.6 可审计性是生产级 Agent 的关键

可以这样说：

```text
每个 proposal 都包含原因、目标、参数、执行步骤和玩家回复，审批和执行都有记录。这样出了问题可以追溯，而不是只看一段聊天记录。
```

## 6.7 垂类 Agent 要能处理证据

可以这样说：

```text
客服不是纯文本问答。Agent 需要查历史数据、账号状态、物品流转、附件、日志和 crash 信息。因此我把证据收集能力设计成专门工具和子 Agent，而不是让模型凭空判断。
```

## 6.8 测试不能只测 prompt

可以这样说：

```text
我认为垂类 Agent 的测试应该覆盖工具调用、权限拒绝、proposal 创建、审批执行、回复格式和 end-to-end case scenario，而不仅仅是看模型回答是否自然。
```

---

# 7. 面试开场介绍模板

## 7.1 30 秒版本

```text
我做过一个面向内部客服场景的垂直领域 AI Agent 系统。它不是普通聊天机器人，而是一个完整的客服调查工作台。Agent 可以读取工单、调用内部工具查询玩家和账号信息、根据 policy/playbook 做判断，并生成结构化 proposal。所有涉及账号、资产或工单状态变更的操作，都必须由人工客服审批后由后端安全执行。这个项目让我比较深入地理解了垂类 Agent 的核心：不是单纯 prompt，而是业务流程建模、工具封装、权限边界、人机协作和可审计执行闭环。
```

## 7.2 2 分钟版本

```text
这个项目是一个内部客服 AI Agent 平台，目标是帮助客服处理复杂 support cases。传统客服需要在多个后台系统中查询工单、账号、玩家状态、物品记录、历史经济数据和附件，然后再根据内部政策判断是否补偿、关闭、升级或回复玩家。

我的设计是把这个流程拆成几个层次：第一层是领域知识，包括客服政策、决策框架和 case playbook；第二层是 skill/tool 层，把工单读取、玩家调查、账号查询、物品追踪、回复生成、操作提案等能力封装成受控脚本；第三层是 Agent 工作流，负责理解工单、拆解玩家诉求、调用工具收集证据、生成处理建议；第四层是 Proposal 审批系统，所有高风险操作都不允许 Agent 直接执行，而是生成 proposal，由人工客服在前端审批；第五层是后端执行器，用 allowlist、script contract、case/account scope 校验和人工登录态来安全执行。

我觉得这个项目最有价值的地方在于，它没有把 LLM 当成万能执行器，而是把 LLM 放在一个受控的业务系统里。模型负责推理和建议，权限控制、审计、执行和最终责任都由系统和人类承担。这也是我对垂类 Agent 的理解：真正难点不是接模型，而是把模型安全地嵌入真实业务流程。
```

---

# 8. 面试 FAQ

下面这些 FAQ 可以直接用于面试准备。建议不要机械背诵，而是理解背后的逻辑后，用自己的语言表达。

---

## Q1：你这个项目到底是 Chatbot 还是 Agent？

推荐回答：

```text
它不是传统 chatbot，而是一个垂直领域 Agent 工作流系统。Chatbot 主要处理自然语言对话，而这个系统需要理解工单、查询内部系统、调用工具收集证据、基于政策判断、生成结构化 proposal，并把 proposal 接入人工审批和后端执行流程。

所以它更像是一个 internal support investigator。LLM 是推理层，真正的产品形态是客服工作台 + 工具系统 + 审批执行流。
```

加分点：

```text
我会区分 conversational interface 和 agentic workflow。前者关注对话体验，后者关注目标完成、工具调用、状态管理和安全边界。
```

---

## Q2：为什么说它是垂类 Agent？

推荐回答：

```text
因为它不是通用问答，而是深度绑定客服领域的业务规则、操作流程和内部系统。它需要理解 support case 分类、客服政策、玩家账号结构、虚拟经济、物品历史、账号安全和工单处理流程。

我做的不是简单让模型回答问题，而是把垂直领域知识转化成 policy、playbook、skill scripts 和 proposal contracts，让 Agent 按照业务规则完成任务。
```

---

## Q3：这个 Agent 的核心架构是什么？

推荐回答：

```text
核心架构可以分为五层：

第一层是前端客服工作台，包括工单列表、详情、AI 会话和 proposal 卡片。
第二层是 support backend，提供 /support/* API、状态同步、agent token、proposal store 和执行器。
第三层是 Agent runtime，启动时注入 case JSON、mode、token 和相关上下文。
第四层是 skill/tool 层，封装工单读取、玩家调查、账号操作、物品追踪等能力。
第五层是安全执行层，所有 mutation 都通过 proposal、人审、allowlist 和后端执行器完成。

整个链路是：客服打开 case -> 启动 Agent -> Agent 调查 -> 创建 proposal -> 人工审批 -> 后端执行 -> 审计记录。
```

---

## Q4：为什么不让 Agent 直接执行操作？

推荐回答：

```text
因为客服场景涉及高风险操作，比如账号修改、资产补偿、玩家传送、物品恢复、工单关闭等。LLM 可能幻觉、误解上下文，也可能被 prompt injection 影响。

所以我采用 proposal-based human-in-the-loop 模型。Agent 只能提出结构化建议，不能直接执行。真正执行前，后端会检查 proposal 的 case scope、account scope、command allowlist、script contract 和人工审批状态。这样即使模型出错，也不会直接影响生产系统。
```

加分句：

```text
我认为 prompt 不是安全边界，后端权限系统才是安全边界。
```

---

## Q5：Proposal 机制具体是什么？

推荐回答：

```text
Proposal 是 Agent 生成的结构化操作建议。它包含 case_id、account_id、operation、reason、params、玩家回复草案、展示信息，以及真正执行用的 execution.steps。

Agent 创建 proposal 后，前端会展示成 proposal card，客服可以查看原因、目标、风险和玩家回复，也可以编辑回复。审批后，后端再次校验 proposal 是否仍然合法，然后执行其中的受控命令。
```

可以补充：

```text
这里的关键设计是，operation 不是执行依据，真正执行的是 execution.steps。operation 用于展示和审计，execution.steps 用于受控执行。这种设计更可扩展，也更容易做 allowlist 和 contract 校验。
```

---

## Q6：为什么 Proposal 不直接根据 operation 执行？

推荐回答：

```text
如果后端写成 switch(operation)，每加一个动作都要改执行器逻辑，扩展性差，而且容易出现 operation 和实际执行不一致的问题。

我更倾向 data-driven execution model：proposal 携带受控的 execution.steps，每一步有 command 和 parameters。后端只负责统一校验和执行这些步骤。这样新增动作时，只需要新增 skill script、注册 contract、加入 allowlist 和测试，不需要把执行器写成大量业务分支。
```

---

## Q7：如何防止低权限工具伪造高风险 proposal？

推荐回答：

```text
后端做了 proposal source validation。每个 proposal 都会带 created_by_skill 和 created_by_script，后端会检查这个脚本是否存在、路径是否合法，以及它是否被允许创建对应 operation 和 command。

也就是说，不是任何脚本都能创建任何 proposal。比如一个普通回复脚本不能伪造成账号修改 proposal。这个由 script contract map 和 command allowlist 双重约束。
```

---

## Q8：Skill / Tool 是怎么设计的？

推荐回答：

```text
我把工具按照业务领域拆成几个 skill：support case、player information、player actions、player account、jira bug report 等。每个 skill 下有一组脚本，每个脚本完成一个明确动作，比如读取 case、下载附件、查询玩家状态、追踪物品历史、创建回复 proposal。

这样做的好处是：
1. Agent 不需要直接拼复杂 API。
2. 每个工具输入输出明确。
3. 权限边界清楚。
4. 后端可以按脚本来源做校验。
5. 未来新增能力可以按 skill 模块扩展。
```

---

## Q9：如何让 Agent 不幻觉？

推荐回答：

```text
我主要从三层控制幻觉。

第一是流程层：Agent 不能直接根据玩家描述下结论，而要把玩家诉求拆成可验证 claim，再调用工具查证据。
第二是知识层：客服政策、decision framework 和 case playbooks 被显式提供给 Agent，让它按领域流程判断。
第三是执行层：即使 Agent 产生错误建议，也只能生成 proposal，最终由人工和后端校验拦截。

所以不是单靠 prompt 要求“不要幻觉”，而是通过证据驱动和权限边界降低幻觉影响。
```

---

## Q10：Policy / Playbook 的作用是什么？

推荐回答：

```text
Policy 决定什么能做、什么不能做；Decision Framework 决定如何判断；Playbook 决定不同 case 类型应该怎么调查。

比如物品丢失、账号安全、玩家卡住、付款问题、崩溃问题，它们需要查的数据源和处理方式都不一样。通过 playbook，Agent 不会盲目自由发挥，而是按照标准路径调查。
```

---

## Q11：为什么垂类 Agent 需要 Playbook？

推荐回答：

```text
因为垂直场景不是开放问答，而是业务流程。优秀客服的经验本质上是一套 playbook：遇到什么类型 case、先查什么、再查什么、什么情况下补偿、什么情况下拒绝、什么情况下升级。

把这些经验沉淀成 playbook，Agent 才能稳定复用专家流程，而不是每次重新推理。
```

---

## Q12：Agent 怎么处理附件？

推荐回答：

```text
工单里如果有附件，Agent 必须下载并检查。因为很多 support case 的关键信息在截图里，比如错误提示、交易截图、位置截图、崩溃窗口等。

这是我特别关注的一点：真实客服场景不是纯文本，Agent 必须具备多模态证据意识。跳过附件会导致判断不完整。
```

---

## Q13：如何处理历史数据问题？

推荐回答：

```text
如果玩家反馈的是几天前或几周前的问题，不能只看当前 live state。当前库存或位置不能证明历史发生了什么。

所以系统需要接入历史数据源，比如经济历史、物品流转、拍卖记录、日志、crash reports 等。Agent 遇到历史 claim 时，应优先查历史证据，而不是只看当前状态。
```

---

## Q14：为什么要用子 Agent？

推荐回答：

```text
不是所有调查都适合主 Agent 自己做。比如经济历史、日志、崩溃分析、知识库检索都需要专门上下文和工具。

所以可以把这些能力拆给 data-investigator、log-investigator、crash-investigator、knowledge-retriever 等子 Agent。主 Agent 负责 case 总控和最终建议，子 Agent 负责垂直证据调查。
```

加分点：

```text
这种模式可以降低主 Agent 上下文污染，也让工具权限更清晰。
```

---

## Q15：如何做权限控制？

推荐回答：

```text
权限控制分几层：

1. Agent 只拿 scoped token，不拿后台原始凭证。
2. Token 绑定当前 case 和操作范围。
3. 只读工具走 command whitelist 和 view templates。
4. mutation 必须生成 proposal。
5. proposal 需要校验 case_id、account_id、planet_id。
6. command 必须在 allowlist。
7. source script 必须符合 contract。
8. 最终执行必须有人类审批和操作员登录态。

所以安全不是靠单点，而是多层防御。
```

---

## Q16：如何防 prompt injection？

推荐回答：

```text
我不会把 prompt 当作唯一防线。Prompt injection 可能让模型尝试越权、忽略规则或生成恶意 proposal。

我的防御思路是：
1. 玩家输入只作为 untrusted content。
2. Agent 不能直接执行 mutation。
3. 后端校验 proposal 的来源、scope 和 command。
4. 高风险操作必须人工审批。
5. 不允许 agent 获取原始高权限凭证。
6. 玩家回复不能泄露内部信息。
```

加分句：

```text
即使模型被注入影响，后端 policy enforcement 也应该能拦截危险动作。
```

---

## Q17：如何防止数据泄露？

推荐回答：

```text
主要从输出和权限两方面做。

输出上，玩家回复只能包含对玩家可见的信息，不能暴露内部日志、数据库字段、风控规则、检测逻辑和安全细节。

权限上，Agent 只能访问当前 case 需要的数据，且只通过白名单工具访问。对于账号安全、支付、安全设备等敏感信息，必须严格限制展示和回复内容。
```

---

## Q18：Agent 如何生成玩家回复？

推荐回答：

```text
Agent 会基于调查结论生成玩家可读的回复，但回复必须符合客服语气和政策要求，不能暴露内部证据细节。回复通常通过 reply proposal 提交，客服可以在前端编辑，审批后才发送。

另外系统强制回复签名格式，确保客服口径一致。
```

---

## Q19：如果 Agent 给出错误 proposal 怎么办？

推荐回答：

```text
有三层处理：

第一，客服可以在 UI 上拒绝 proposal。
第二，后端 approve 前会重新校验 proposal 的合法性。
第三，测试体系会覆盖常见非法 proposal，比如错误 account、非法 command、缺少签名、越权 related account 等。

所以错误 proposal 不应该直接造成生产影响。
```

---

## Q20：这个系统如何做实时状态同步？

推荐回答：

```text
前端启动时先请求 support state，然后通过 SSE 建立状态流。后端 proposal store、case state 和 agent session 状态变化后，通过 state store 推送到前端。这样客服可以实时看到 proposal 创建、审批、拒绝和执行结果。
```

---

## Q21：为什么用 SSE，而不是轮询？

推荐回答：

```text
客服工作台需要及时看到 proposal 和 case 状态变化。轮询简单但延迟和资源浪费更高。SSE 对这种服务端单向推送场景足够轻量，前端实现也简单。
```

---

## Q22：如何做测试？

推荐回答：

```text
我会分层测试：

1. Skill script 测试：确保每个工具能正确调用和返回。
2. Proposal creation 测试：确保脚本生成结构合法。
3. Validation 测试：非法 account、非法 command、缺少签名、越权操作应被拒绝。
4. Approval execution 测试：approve 后执行正确 command，deny 后不执行。
5. Scenario 测试：从 case 到 agent 调查、proposal、审批、执行、审计的完整闭环。
```

---

## Q23：如何评估 Agent 效果？

推荐回答：

```text
我不会只看模型回答是否流畅，而会看业务指标和安全指标。

业务指标包括：
- 平均工单调查时间
- 首次有效建议时间
- proposal 被采纳率
- 客服编辑率
- 一次解决率
- 升级率变化

安全指标包括：
- 非法 proposal 拒绝率
- 越权调用拦截
- 玩家回复泄露风险
- 审批前后不一致问题
- 工具调用失败率

垂类 Agent 的评价应该是任务完成质量，而不是聊天质量。
```

---

## Q24：如果没有量化指标，面试时怎么说？

推荐回答：

```text
如果没有正式线上指标，我会诚实说明项目当前阶段，并强调我建立了可量化的评估框架。比如 proposal 采纳率、客服编辑率、case 处理时长、非法操作拦截率、场景测试通过率等。

这能体现我不是只做 demo，而是在按生产系统标准设计。
```

---

## Q25：这个系统最难的部分是什么？

推荐回答：

```text
最难的不是接入 LLM，而是把 LLM 安全地嵌入真实业务流程。客服场景里有很多高风险操作，模型可能理解错、证据不完整或被注入影响。

所以难点在于：如何把自由文本推理转化成结构化、可校验、可审批、可审计的操作流。这也是我设计 proposal gate、skill contract、allowlist 和 human-in-the-loop 的原因。
```

---

## Q26：你如何看待“全自动客服”？

推荐回答：

```text
我觉得要分场景。低风险 FAQ、状态查询、流程引导可以更自动化；但涉及账号、安全、资产、经济和处罚类问题，应该保留 human-in-the-loop。

我的设计不是排斥自动化，而是按照风险分层。低风险自动，高风险提案，人类审批。
```

---

## Q27：如果让你重新设计一次，你会优化什么？

推荐回答：

```text
我会进一步加强三点：

第一，评估体系，把历史 case 做成 golden dataset，自动评估 Agent 的调查路径、证据引用和 proposal 质量。
第二，权限 DSL，让不同工具和 proposal 的权限策略更声明式，而不是分散在代码里。
第三，观测性，记录每次 Agent 的工具调用链路、决策依据和失败原因，便于持续优化。
```

---

## Q28：Agent 怎么知道什么时候该调用哪个工具？

推荐回答：

```text
通过三部分决定：

1. case playbook：不同 case 类型有推荐调查路径。
2. skill descriptions：每个工具说明适用场景和输入输出。
3. Agent 的 claim decomposition：先拆解玩家诉求，再根据需要验证的 claim 选择工具。

例如玩家说物品丢了，就需要查库存、物品历史、经济历史和交易记录；玩家说卡住了，就查当前位置和最近位置。
```

---

## Q29：怎么避免工具太多导致 Agent 混乱？

推荐回答：

```text
我会按领域拆 skill，而不是把几十个工具平铺给模型。每个 skill 有清晰边界，比如 support-cases 只处理工单，player-information 只做只读调查，player-actions 只创建玩家动作 proposal。

同时工具命名要具体，输入参数要简单，输出要结构化。必要时用 playbook 引导工具选择。
```

---

## Q30：你怎么理解 Agent 的 memory 和 knowledge？

推荐回答：

```text
在这个项目里，我会区分三类信息：

1. Case context：当前工单内容，只在当前任务内使用。
2. Domain knowledge：政策、playbook、机制说明，属于长期知识。
3. Audit/history：proposal、审批、执行结果，用于追踪和复盘。

不能把所有东西都塞进 prompt。不同信息要放在不同层，用不同生命周期管理。
```

---

## Q31：为什么需要审计？

推荐回答：

```text
客服操作可能影响用户账号和资产。如果没有审计，出问题后无法知道是谁批准、Agent 为什么建议、执行了什么参数、玩家收到了什么回复。

Proposal 机制天然适合审计，因为它把 reason、params、execution steps、审批人和执行结果结构化保存下来。
```

---

## Q32：如何处理工具调用失败？

推荐回答：

```text
工具失败不能让 Agent 猜。应该明确返回错误，并让 Agent 标记证据缺失。如果关键数据源失败，Agent 应该建议稍后重试或升级，而不是基于不完整信息创建高风险 proposal。

后端执行 proposal 时也要有 failure_message，并保证失败不会造成半执行状态。
```

---

## Q33：如果一个 proposal 需要多个动作怎么办？

推荐回答：

```text
我倾向于避免复杂多步 proposal，尤其是高风险操作。最好拆成多个独立 proposal，让客服按顺序审批。

比如移动玩家和回复玩家应该拆开。这样每个 proposal 的风险、目标和审计都更清晰。
```

---

## Q34：如何处理玩家多个账号相关的问题？

推荐回答：

```text
默认 proposal 只能作用于当前 case 的 account。如果涉及 related account，必须先注册 related account，并且只有少数低风险或明确允许的操作可以作用于 related account。

这样可以防止 Agent 因为玩家描述或名字相似而误操作其他账号。
```

---

## Q35：如何保证玩家回复语气一致？

推荐回答：

```text
通过 policy 和回复格式约束。一方面 Agent 需要基于客服政策生成回复，避免承诺过度或泄露内部信息；另一方面系统强制固定签名，并允许客服在审批前编辑回复。

最终回复不是模型直接发出，而是 proposal 的一部分，由人审后发送。
```

---

## Q36：这个系统和 RAG 有什么关系？

推荐回答：

```text
它可以包含 RAG，但不等于 RAG。RAG 主要解决知识检索问题，而这个系统还包括工具调用、权限控制、审批执行、状态同步和审计。

在垂类 Agent 里，RAG 只是 domain knowledge 层的一部分。真正落地还需要 workflow 和 action safety。
```

---

## Q37：你怎么选择模型？

推荐回答：

```text
客服调查需要较强推理、长上下文和工具调用能力，所以主 Agent 应该用推理能力强的模型。对于简单分类、摘要或格式化任务，可以用较低成本模型。

我更关注模型在业务闭环中的可靠性，而不是单次回答效果。模型选择要结合工具调用质量、长上下文稳定性、成本和延迟。
```

---

## Q38：如何控制成本？

推荐回答：

```text
可以从几个方面控制：

1. 不把所有历史数据直接塞进上下文，而是按需查工具。
2. 用 playbook 降低无效探索。
3. 简单任务用低成本模型，复杂调查用强模型。
4. 缓存 case detail、历史查询和工具结果。
5. 通过 structured output 减少反复修正。
```

---

## Q39：如何做扩展性设计？

推荐回答：

```text
扩展点主要有三个：

1. 新增 skill script，扩展 Agent 可用工具。
2. 新增 proposal operation 和 execution command，扩展可审批动作。
3. 新增 playbook，扩展 case 类型处理能力。

每新增一个动作，都要同步注册 contract、allowlist、前端展示和测试。
```

---

## Q40：如果面试官问“你具体负责了什么？”

推荐回答模板：

```text
我主要负责整体 Agent 工作流和安全执行架构，包括：
1. 梳理客服业务流程并拆成 Agent 可执行阶段。
2. 设计 skill/tool 分层，把内部系统能力封装成受控工具。
3. 设计 proposal-based human-in-the-loop 机制，避免 Agent 直接执行高风险操作。
4. 设计后端校验，包括 token scope、allowlist、script contract、case/account 校验。
5. 参与前端客服工作台和 proposal card 的交互设计。
6. 建立测试和验收路径，覆盖工具、proposal 和端到端 case。
```

如果你实际只参与一部分，不要夸大，可以说：

```text
我重点参与了其中的 Agent 架构、工具抽象、Proposal 安全模型和复刻文档梳理。
```

---

# 9. 简历关键词

## 9.1 英文关键词

可以在简历技能或项目描述里自然加入：

```text
Vertical AI Agent
Domain-specific Agent
LLM Tool Calling
Human-in-the-loop
Proposal-based Execution
Agent Workflow Orchestration
Policy-driven Agent
Playbook-driven Reasoning
Least-privilege Access Control
Scoped Token
Command Allowlist
Tool Contract Validation
Audit Trail
SSE Real-time State Sync
Agent Evaluation
End-to-end Scenario Testing
Customer Support Automation
Internal Tooling
Risk-controlled Automation
```

## 9.2 中文关键词

```text
垂直领域 Agent
人机协作审批流
工具调用
领域知识建模
客服工作流自动化
最小权限原则
操作白名单
结构化提案
可审计执行
高风险动作隔离
业务 Playbook
端到端场景测试
```

---

# 10. 面试中应避免的说法

## 10.1 不要把项目说成普通聊天机器人

不要说：

```text
我做了一个客服聊天机器人。
```

应该说：

```text
我做的是一个面向客服场景的垂直领域 Agent 工作流系统。
```

## 10.2 不要暗示 Agent 可以直接改生产数据

不要说：

```text
Agent 可以自动改玩家账号。
```

应该说：

```text
Agent 只能创建 proposal，必须人工审批后由后端执行。
```

## 10.3 不要说安全靠 Prompt

不要说：

```text
我们靠 prompt 保证安全。
```

应该说：

```text
Prompt 只是行为指导，真正的安全边界在后端权限、allowlist、contract 和人审机制。
```

## 10.4 不要说 LLM 自己判断

不要说：

```text
LLM 会自己判断。
```

应该说：

```text
Agent 根据 policy、playbook 和工具证据做判断。
```

---

# 11. 最终推荐简历成稿

## 11.1 中文版

可以直接使用这一版：

```text
垂直领域智能客服 Agent 平台
- 设计并落地面向复杂客服场景的 AI Agent 工作流系统，将工单理解、领域知识、工具调用、证据调查、处理建议、人审提案和后端安全执行串联为完整闭环。
- 构建 Proposal-based human-in-the-loop 安全机制：Agent 仅负责调查与生成结构化处理方案，所有涉及账号、资产、经济或工单状态变更的操作均需人工审批后由后端受控执行。
- 设计领域 Skill / Tool 体系，将工单读取、玩家资料查询、账号状态调查、物品流转追踪、历史数据查询、附件检查、回复生成等能力封装为可审计工具，并通过 token scope、command allowlist 和 script contract 限制调用边界。
- 搭建 Policy / Playbook 驱动的决策框架，使 Agent 能够根据不同 case 类型执行标准化调查路径，降低幻觉和随意判断风险，确保输出符合业务规则与客服口径。
- 实现客服工作台与 Agent 执行链路集成，包括工单列表、工单详情、Agent 调查会话、Proposal 卡片、审批/拒绝、玩家回复编辑、实时状态同步与审计记录。
- 建立测试与验收体系，覆盖 skill 调用、proposal 创建、非法操作拒绝、人工审批执行、玩家回复格式校验和端到端客服场景闭环。
```

## 11.2 英文版

```text
Vertical AI Support Agent Platform
- Designed and implemented a domain-specific AI support agent workflow that connects case understanding, policy playbooks, tool calling, evidence investigation, proposal generation, human approval, and auditable backend execution.
- Built a proposal-based human-in-the-loop safety mechanism where the agent investigates and drafts structured actions, while all account, asset, economy, or case-state mutations require operator approval before backend execution.
- Designed a modular skill/tool system for case retrieval, player/account investigation, item history tracing, historical data lookup, attachment inspection, and support reply generation, secured by scoped tokens, command allowlists, and script contract validation.
- Developed a policy/playbook-driven decision framework to guide standardized investigation paths across case categories and reduce hallucination risk.
- Integrated the agent into an internal support workbench with case details, embedded agent sessions, proposal cards, editable player replies, approval/denial actions, real-time state updates, and audit trails.
- Established validation and testing coverage for tool calls, proposal creation, invalid action rejection, approval execution, response formatting, and end-to-end support scenarios.
```

---

# 12. 核心表达主线

这句话是面试中展示“垂类 Agent 深刻理解”的主线：

```text
我做的不是“让大模型回答客服问题”，而是“把大模型安全地嵌入真实客服业务流程，让它在权限受控、证据驱动、人类审批、可审计执行的框架下完成垂直领域任务”。
```

展开后就是：

```text
垂类 Agent 的难点不是会不会调用模型，
而是如何把真实业务流程拆成 Agent 可执行的步骤，
如何把领域知识转化成 policy/playbook，
如何把内部系统能力封装成受控工具，
如何限制权限和数据范围，
如何在高风险动作前引入人审，
如何让每一步都可追踪、可测试、可回滚、可审计。
```

这也是 Celest Support Agent 项目最适合在简历和面试中强调的核心价值。
