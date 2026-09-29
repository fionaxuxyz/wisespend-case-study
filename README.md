# 知省 WiseSpend｜消费复盘与决策 AI Agent

> 面向真实个人账本数据的 AI 消费复盘与决策产品  
> `React` · `Spring Boot` · `Python / FastAPI` · `OpenAI Agents SDK` · `PostgreSQL` · `RAG` · `Long-term Memory`

WiseSpend 基于原有个人记账 Web 应用，从 0→1 升级为面向真实账本数据的消费决策 Agent。

产品支持账本问答、多轮追问、预算建议、消费复盘、长期偏好与财务知识解释。整体设计重点不是把 LLM 做成一个“会聊天的界面”，而是让它能够基于真实业务数据、明确的 evidence 和受约束的 execution flow 生成可验证回答。

> 🔒 **Source code is private.**  
> 本仓库仅用于展示产品定位、System Architecture、Agent Engineering、RAG / Memory 设计及部分产品效果，不包含业务源码。

---

## Product Overview

传统个人财务产品通常可以回答：

- 这个月花了多少钱
- 哪个分类消费最多
- 是否超过预算

WiseSpend 希望进一步回答：

- “这个月外卖是不是比上个月多？”
- “9 月哪些类别超预算了？”
- “最近咖啡是不是花得更多了？”
- “那上个月呢？”
- “我之前说咖啡预算是多少？”
- “什么是沉没成本？”

因此整个 Agent 围绕三类 evidence source 构建：

```text
Finance
真实账本数据

Memory
长期用户偏好与目标

Knowledge
外部财务知识
```

核心原则：

> **LLM 负责理解与规划，确定性程序负责执行与校验。**

LLM 主要负责：

```text
Intent Understanding
Task Planning
Natural Language Generation
```

程序主要负责：

```text
Tool Execution
Data Calculation
Permission Control
Evidence Validation
State Management
```

账本事实来源始终是 Backend / Database，而不是 LLM。

---

## System Architecture

```text
React Frontend
      │
      ▼
Spring Boot
      │
      ├── Transaction / Budget / Account
      ├── Financial Analytics
      └── PostgreSQL
      │
      ▼
Python AI Service
      │
      ├── Semantic Planner
      ├── Canonical Plan
      ├── Evidence Plan
      ├── Tool Execution
      ├── Memory
      ├── Knowledge Retrieval
      ├── Evidence Selector
      └── Validation
      │
      ▼
LLM
```

AI Service 不作为账本事实源。

`Transaction / Budget / Account` 等核心业务数据始终由 Spring Boot 与 PostgreSQL 管理；AI 层只负责理解用户意图、组织任务、调用 Tool、获取 evidence 并生成最终回答。

---

## Agent Harness

WiseSpend 设计了一套受约束的 Agent execution pipeline：

```text
User Query
    ↓
Semantic Planner
    ↓
Canonical Plan
    ↓
Evidence Plan
    ↓
Tool Execution
    ↓
Validation
    ↓
Evidence Selection
    ↓
Final Response
```

### Semantic Planner

负责理解自然语言问题及多轮上下文。

例如：

```text
“那上个月呢？”
```

系统需要继承上一轮的 subject，同时覆盖 period。

Planner 负责语义理解，而不是依赖大量 Regex / keyword rules 硬编码用户表达。

### Canonical Plan

将自然语言意图转换成结构化、可执行的内部表示。

例如：

```text
Intent:
COMPARE_SPENDING

Subject:
Food Delivery

Current Period:
September

Comparison Period:
August
```

这样可以将：

```text
“用户说了什么”
```

与：

```text
“程序应该执行什么”
```

分离。

### Evidence Plan

Agent 在回答之前，先确定回答问题所需要的 evidence。

例如：

```text
“我这个月咖啡是不是花太多了？”
```

可能需要：

```text
Current Coffee Spending
Previous Period Coffee Spending
Transaction Evidence
User Coffee Budget Memory
```

只有拿到足够 evidence 后，系统才允许进入最终回答。

---

## Finance / Memory / Knowledge

WiseSpend 将 Agent 能力拆分为三个独立 evidence domain。

### Finance

负责所有真实账本事实：

```text
Transaction
Category
Merchant
Budget
Account
Period Comparison
Spending Pattern
```

典型 Tool：

```text
getSpendingSummary
getCategoryBreakdown
comparePeriods
getTransactionEvidence
detectSpendingPattern
getBudgetStatus
```

Finance Tool 最终查询真实 Backend / Database 数据。

### Memory

负责跨会话长期用户信息，例如：

```text
“以后咖啡每个月控制在 400 元以内。”

“我最近想少点外卖。”

“我正在为旅行存钱。”
```

Memory 不等于保存完整聊天记录。

系统将长期有效信息转换为结构化 memory claim，并支持：

```text
Create
Update
Supersede
Retrieve
```

例如用户先说：

```text
coffee budget = 300
```

后来修改：

```text
coffee budget = 400
```

系统通过 structured claim 与 `SUPERSEDE` 机制让新偏好覆盖旧偏好，避免长期记忆中同时存在相互冲突的信息。

### Knowledge

用于回答并非来自个人账本的财务知识问题，例如：

```text
“沉没成本是什么？”

“CPI 为什么会上升？”

“预算应该怎么调整？”
```

这些内容通过独立 Finance Knowledge Base 获取，而不是与用户账本数据混在一起检索。

---

## RAG Pipeline

WiseSpend 构建了面向中英文 Finance Knowledge 的 Hybrid RAG Pipeline：

```text
PDF / Knowledge Source
        ↓
Hierarchical Chunking
        ↓
BM25
+
multilingual E5
+
pgvector
        ↓
RRF
        ↓
Evidence Selector
        ↓
Relevant Evidence
        ↓
LLM
```

主要包括：

- `Hierarchical Chunking`
- `BM25`
- `multilingual E5`
- `pgvector`
- `RRF`
- `Evidence Selector`

针对长篇 PDF，系统进行层级化解析与知识切分。

同时针对中文 Query 做跨语言 retrieval，使用户可以使用中文问题检索英文原始 Finance 文献。

当前 Hybrid Retrieval evaluation：

```text
Recall@5: 82.2%
MRR:      0.741
```

---

## Why Not Embed Transactions?

WiseSpend 没有把 Transaction 全部做成 Vector Embedding。

交易数据本身具有明确结构：

```text
amount
merchant
category
date
description
account
```

对于：

```text
“9 月瑞幸花了多少钱？”
```

SQL / Business Tool 可以提供确定结果。

Vector Search 更适合：

```text
“哪些知识和这个问题语义相关？”
```

因此系统明确区分：

```text
Structured Business Data
        ↓
SQL / Tool

Unstructured Knowledge
        ↓
RAG / Vector Retrieval
```

避免为了“用了 RAG”而把所有数据都塞进 Vector Database。

---

## Evidence Grounding

WiseSpend 不允许 Agent 在 evidence 不完整时继续生成一个看起来合理的答案。

```text
Plan
 ↓
Required Evidence
 ↓
Tool / Retrieval
 ↓
Evidence Selector
 ↓
Validation
 ↓
Enough Evidence?
 ├── Yes → Final Answer
 └── No  → Repair / Clarify / Fail Closed
```

核心机制包括：

```text
Evidence Gating
Layered Bounded Repair
Fail-Closed
```

如果关键 evidence 缺失，系统会进入 repair、clarification 或 fail-safe path，而不是让模型自行补全事实。

---

## Agent Runtime

Agent Runtime 基于：

```text
OpenAI Agents SDK
+
Custom Python Orchestration
```

Agents SDK 提供基础能力，例如：

```text
Tool Calling
Tracing
```

自定义 orchestration 负责控制：

```text
Planning
State
Evidence
Validation
Repair
Failure Handling
```

这样既复用 SDK 的通用能力，又保留关键业务链路的 deterministic control。

---

## Multi-turn Conversation

WiseSpend 支持连续业务语境中的多轮追问。

例如：

```text
User:
这个月外卖花了多少？

Agent:
410 元。

User:
那上个月呢？
```

第二轮并没有重新出现：

```text
subject = 外卖
```

因此系统需要继承上一轮 conversation state，同时覆盖：

```text
period
```

新的独立问题则必须与旧问题隔离。

多轮对话不是简单地：

```text
把全部聊天记录重新发给 LLM
```

而是通过 structured state 显式管理：

```text
inherit
override
reset
isolate
```

---

## Reliability

Agent 本质上是存在不确定性的 runtime component。

因此 WiseSpend 对 reliability 单独做了工程设计：

```text
Timeout
Request Cancellation
Bounded Retry
Failure Classification
Health Guard
Fail-Closed
Bounded Repair
```

例如模型 API 暂时 unavailable 时：

- 不无限 Retry
- 不进入无限 Agent loop
- 不在缺失证据时继续生成答案

每种 recovery path 都有明确边界。

---

## Evaluation & Observability

WiseSpend 建立了分层 Evaluation，而不是只通过人工“问几句”判断系统是否可用。

```text
Planner
Tool
Retrieval
Evidence Selection
Citation
Final Answer
```

每一层都可以独立判断 bad case 出现在哪里。

### Agent Tracing

一次完整请求可以追踪：

```text
User Query
    ↓
Semantic Plan
    ↓
Canonical Plan
    ↓
Evidence Plan
    ↓
Tool Calls
    ↓
Tool Results
    ↓
Retrieval
    ↓
Evidence Selection
    ↓
Validation
    ↓
Final Response
```

通过 Trace 可以定位：

```text
Planner Error
Tool Error
Retrieval Error
Evidence Error
Generation Error
```

而不是简单归因成：

```text
“LLM 又答错了。”
```

### Regression Workflow

典型 bad case 会被转化为 Regression Case：

```text
Production / Testing Failure
        ↓
Trace Analysis
        ↓
Failure Classification
        ↓
Regression Case
        ↓
Fix
        ↓
Replay
```

这样 Agent 行为可以通过 evaluation 持续迭代，而不是每次修改 Prompt 后重新人工试一遍。

---

## Full-stack Integration

### Frontend

```text
React
JavaScript / TypeScript
REST API
SSE
```

负责：

- Financial Dashboard
- Transactions
- Budget
- AI Chat
- Streaming Response

### Application Backend

```text
Spring Boot
PostgreSQL
```

负责：

- Account
- Transaction
- Category
- Budget
- Financial Calculation

是个人账本事实的 Source of Truth。

### AI Service

```text
Python
FastAPI
OpenAI Agents SDK
```

负责：

- Planning
- Tool Calling
- RAG
- Memory
- Evidence Validation
- Agent Runtime

### Data & Infrastructure

```text
PostgreSQL
pgvector
Redis
Docker
Git
```

---

## Selected Product Workflows

### Finance Query

```text
“这个月外卖花了多少钱？”
        ↓
Semantic Planner
        ↓
Finance Evidence Plan
        ↓
Tool
        ↓
Backend / SQL
        ↓
Structured Financial Result
        ↓
Grounded Answer
```

### Finance + Memory

```text
“我这个月咖啡是不是花太多了？”
        ↓
Finance
当前咖啡消费

+

Memory
用户咖啡预算 / preference

        ↓
Combined Evidence
        ↓
Final Answer
```

### Knowledge Query

```text
“沉没成本是什么？”
        ↓
Knowledge Domain
        ↓
Hybrid Retrieval
        ↓
Evidence Selector
        ↓
Finance Knowledge Evidence
        ↓
Answer + Citation
```

---

## My Work

作为个人项目，我负责产品设计、Agent Architecture 与 Full-stack Development。

主要工作包括：

- 将原个人记账 Web 应用升级为消费决策 AI Agent
- 完成 React AI Chat 与既有账本产品的端到端集成
- 设计 `Semantic Planner → Canonical Plan → Evidence Plan → Tool Execution → Validation` Agent harness
- 划分 `Finance / Memory / Knowledge` 三类 evidence domain
- 设计 Finance Tool 与真实 Backend 数据之间的调用链路
- 构建 Long-term Memory 与 structured claim 更新机制
- 构建 `Hierarchical Chunking + BM25 + multilingual E5 + pgvector + RRF + Evidence Selector` Hybrid RAG
- 基于 `OpenAI Agents SDK + Custom Python Orchestration` 实现 Agent Runtime
- 建立 `Evidence Gating / Bounded Repair / Fail-Closed` 机制
- 实现 `Timeout / Request Cancellation / Bounded Retry`
- 建立 `Planner / Tool / Retrieval / Evidence / Citation / Final Answer` 分层 Evaluation
- 使用 Agent tracing 分析 bad case，并沉淀 Regression Workflow

---

## Tech Stack

**Frontend**

`React` `JavaScript` `TypeScript`

**AI / Agent**

`Python` `FastAPI` `OpenAI Agents SDK` `Tool Calling` `RAG` `Long-term Memory`

**Retrieval**

`BM25` `multilingual E5` `pgvector` `RRF`

**Backend & Data**

`Spring Boot` `PostgreSQL` `Redis`

**Engineering**

`Docker` `Git` `Tracing` `Evaluation`

---

## Engineering Takeaway

WiseSpend 最核心的工程问题不是：

> **How do I make the LLM smarter?**

而是：

> **How do I make an uncertain model reliable inside a deterministic software system?**

最终设计围绕：

```text
LLM Understanding
        +
Structured Planning
        +
Business Tools
        +
Evidence
        +
RAG
        +
Memory
        +
Validation
        +
Evaluation
        +
Observability
```

让 Agent 从一个能够 Demo 的 Chatbot，变成一个 **可执行、可验证、可测试、可追踪的 application component**。

---

## Screenshots

1. `WiseSpend Dashboard`
2. `AI Chat / Multi-turn Query`
3. `Architecture / Agent Trace`

