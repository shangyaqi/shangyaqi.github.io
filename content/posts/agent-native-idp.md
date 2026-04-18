---
title: "你的 IDP 只是 AI 加强版 —— 我的是 Agent-Native"
date: 2026-04-18
draft: false
tags: ["IDP", "AI Agent", "Platform Engineering", "MCP", "SRE", "架构"]
categories: ["架构思考"]
summary: "AI-enhanced IDP 把 AI 当插件,Agent-Native IDP 把 Agent 当一等用户。两者的架构差异比想象中大得多 —— 这篇讲我为什么要做后者,以及过程中最核心的一个发现:Blueprint 混合编排引擎。"
ShowToc: true
TocOpen: true
---

过去 18 个月,每个稍微像样的 Internal Developer Platform 都发布了"AI 功能"。Backstage 上了 MCP plugin。Port 有 AI Actions。Cortex 在 workflow 里加了 LLM 节点。Humanitec 提供 AI 辅助部署。

这是 **2026 年的门票费**,不是护城河。

真正的问题是:**几乎所有这些平台的底层假设,仍然是"人类是一等用户,AI 是帮手"**。Catalog 分页是为人类设计的,审批队列是发给人类的,Golden Path 是给人填表用的。AI 被当成边上的一个聊天窗口、一个动作建议器、或者一个代码补全。

这是 **AI-enhanced**。不是 **Agent-Native**。

过去一年我在做一个我后来称之为 **Agent-Native IDP Control Plane** 的东西。不是"AI 加强版 Backstage"。不是"MCP gateway"。是一个真正以 AI Agent 为一等用户、人类通过 Agent 操作平台的控制面。这篇文章讲我在这个过程中停止把 AI 当 sidecar 之后,到底有什么东西变了。

## 三句话测试

如果你想知道自己的平台到底是 AI-enhanced 还是 Agent-Native,跑这三个测试:

1. **把人类 UI 完全移除,Agent 还能不能拿到 100% 的平台能力?**
2. **对平台能做的每一个动作,是否有机器可读的风险声明、可逆性声明、rollback 契约?**
3. **当 Agent 输出质量回归时,CI 会 fail 吗?就像单测跑挂那样?**

只要任何一个是 "no",你就是在做 AI-enhanced。这没错 —— 是个完全合理的产品。但它在 12 个月后会被普及化,当每个 IDP 厂商都发布同样的 bolt-on 时。值得问的问题是:**你在建什么东西不会被那一波拍扁?**

## "Agent-Native" 到底改变了什么

从我的实际经验,改变了三件事:

### 1. 协议优先,UI 第二

在 AI-enhanced 的 IDP 里,你先给前端写 REST API,之后可能在上面暴露一些 MCP tools。在 Agent-Native 的 IDP 里,**顺序反过来**。

每个能力都以 MCP tool 的形式声明,带明确的输入 schema、风险分级、副作用说明。Dashboard 是一个只读视图,消费同一套工具。**如果没法通过 MCP 调用平台,那平台就是坏的** —— 不管 UI 能不能用。

二阶效应:你会停止把行为藏在 UI 里。不再有"按钮点击前会做个确认检查,然后调 API" —— 因为 Agent 本来就在直接调 API。每一个门控、每一个 rollback、每一个策略检查都被推到协议层,本来就该在的地方。

### 2. Workflow 既不是纯代码,也不是纯 LLM 调用,而是一张混合图

这是我花时间最多、也是我认为最有原创性的部分。我用我自己建的东西来解释。

2026 年解决"编排长流程工作"的传统答案是 Temporal(或者 Airflow、Cadence)。这些引擎给你 durable execution、retry、可见性。它们假设每一步都是**确定性的** —— 同样输入产生同样输出,否则 retry。

AI Agent 系统的新答案是 LangGraph、CrewAI、AutoGen。它们给你 stateful 的 Agent 图,节点可以推理、调工具、决定下一步做什么。这些框架很好。它们也缺少两样让 Temporal 风格引擎能跑生产的东西:**跨重启的持久状态** + **显式的 human-in-the-loop checkpoint**。

所以我做了一个 **Blueprint 引擎**,里面四种节点类型可以同图共存:

- **Deterministic nodes** — 普通代码。给定输入 X 返回 Y。失败 retry。同 Temporal activity。
- **Agentic nodes** — LLM + tool calls。输出是非确定性的,但会被 schema 校验。失败 = schema 违反,不是 exception。
- **Approval nodes** — 工作流暂停等人类(或有权限的 Agent)签字。24h TTL。跨重启持久。
- **Gate nodes** — 策略检查。读上下文(风险级、服务关键度、时间段、最近失败率)决定下一步能不能跑。纯逻辑,无 LLM。

生产环境的事件响应流在这个引擎里长这样:

```
  Alertmanager webhook
    → [Deterministic] 规范化告警载荷
    → [Agentic]      诊断:"出了什么问题?影响面多大?"
    → [Gate]         策略:这个服务是 SLO 关键 & 在非工作时段吗?
    → [Approval]     如果 Gate 说"是"则必要,否则跳过
    → [Deterministic] 通过 MCP tool 执行补救
    → [Deterministic] 写入 Context Lake
```

这个设计解锁的东西:**LLM 可以在 Agentic 节点里自由推理,但工作流的"形状"是可审计、可暂停、可策略限定的**。你拿到了 Agent 系统的灵活性 + workflow 引擎的持久性。单独用 Temporal 或者单独用 LangGraph 都给不了你这个。

### 3. Agent 带 eval 契约上线,不靠 "感觉还行"

我平台里每个 Agent 都有 fixture。不是测"跑 Agent 的代码"的单元测试 —— 是测 **"给这类输入,Agent 的输出必须包含这些字段、满足这些断言、confidence 分数高于阈值"** 的 fixture。

改 prompt 时、upstream 模型升级时、Scorecard 逻辑调整时,这些 fixture 都在 CI 里跑。pass rate 跌破 80%,MR 直接被 block。

这听起来很显然。**它在市场上几乎不存在**。绝大多数 Agent 系统上线时没有 regression harness —— 这就是为什么"以前能用,现在在幻觉"是 2026 年所有生产 LLM 功能的默认状态。

如果 Agent 是平台的一等用户,**Agent 的质量就必须是一等交付物**。Eval 不是可选项。

## 我也有不确定的地方

说实话:我**不知道** "Agent-Native IDP" 是一个持久的品类,还是一个 3 年窗口之后会坍缩回通用 "IDP with AI" 的概念。

不确定性在这里:

- 如果接下来 24 个月自主 Agent 成本下降 10 倍(SLM + 端侧推理都在这么暗示),那么每个平台都必须假设 Agent-first —— 品类成立
- 如果 Agent 自主性平台化,绝大多数"AI 功能"仍然停留在辅助阶段,那么 Port / Cortex 的实用路径获胜,我在协议和 eval 上的过度投入看起来会像行为艺术

我赌第一种。但我做了对冲 —— **这套架构在第二种情形下也是一个完全好的 IDP 后端**。就算没人用 Agent 层,我也不亏。我指望的护城河(Blueprint engine + Eval Harness + 结构化决策记忆)在两个世界都加分。

## 你可以偷走的东西

即使你不买我整体的 framing,三个具体建议我会强推:

**1. 写一个 ADR 命名你的 positioning**

"AI-enhanced" 和 "Agent-Native" 不应该是一个意外。把它写出来会强迫你回答"我们会**拒绝**建什么?" —— 这个问题才真正定义了路线图。

**2. 挑三个长期差异化,把其他的从对外叙事里删掉**

我的是 Blueprint engine、Eval Harness、决策记忆。不是五个,不是七个。**三个,一口气能说完**。超过三个的"差异化"通常意味着你其实没有差异化。

**3. Agent 质量检查放在 CI 里,不是放在一个没人跑的测试脚本里**

如果 eval 是 nice-to-have,它会腐烂。如果 MR 会被 block,它会一直是绿的。二选一,没有第三条路。

## 为什么写这篇

我是一个 SRE,在做我自己希望存在的平台。我不卖东西。我写这个是因为 2026 年 Q1 我看到了差不多 20 篇 "AI + IDP" 的文章,它们描述的是**同一个 bolt-on 架构,换了不同的标签**。如果有人正在走类似的路、也注意到 bolt-on 的天花板在哪里,我希望他们能找到一个第二数据点。

如果你也在思考这件事,欢迎联系我。我想知道我的地图哪里画错了。

---

*如果这篇对你有用,[Morpheus 仓库](https://github.com/shangyaqi/Morpheus) 里的 ADR 0012 和 0013 把本文的 positioning 正式化了。它们设计成脱离语境也能独立阅读。*
