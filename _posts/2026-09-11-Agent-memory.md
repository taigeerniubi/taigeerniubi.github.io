---
layout: post
title: "Agent Memory 的结构：短期、任务与长期记忆怎么配合"
date: 2026-09-11 00:00:00 +0800
categories: [AI工程]
tags: [Agent, Memory, RAG, 上下文管理]
description: "从 Working Context、短期对话、当前任务和长期记忆出发，解释 Agent Memory 的分层、流动与存储。"
excerpt: "Agent 要同时记住刚才说了什么、任务做到哪里，以及以后仍然值得知道什么。这几类记忆不能混在一张聊天记录里。"
---

用户昨天告诉 Agent：“这个项目用 Python，延迟要低。”今天换了一个会话，只说：“继续把 memory 做完。”

为了接住这句话，Agent 要同时找回几种信息：用户和项目的稳定事实，任务昨天进行到哪里，以及最近讨论中“memory”具体指什么。它们的寿命、读取方式和更新规则都不一样。

很多 Memory 实现把这些内容混在聊天记录里，或者全部切块后塞进向量数据库。演示时通常没问题，数据多起来就会出现旧决定和新决定一起被召回、临时要求变成永久偏好、相似项目互相串线。

Memory 首先是一个结构问题。检索只是其中一环。

![Agent Memory 的四层结构与外部知识边界](/assets/images/agent-memory/memory-system-map.svg)

*图：不同层负责不同时间尺度的信息，Context Assembler 决定这一轮哪些内容值得占用模型注意力。*

## 模型真正使用的是 Working Context

模型每次调用真正看到的，是 Working Context：系统规则、当前任务、相关记忆、最近消息和工具结果组成的一份临时输入。

它更像桌面，不像仓库。数据库里保存了一百万条记录，如果这一轮没有选进 Context，模型仍然不知道。反过来，Context 里的内容调用结束后也不会自动成为长期记忆。

所以完整的 Memory 系统有两类动作：一类把信息保存到合适的层，另一类在每次调用前把需要的信息取回来。Context Assembler 连接了这两边。

## 短期记忆：刚才说了什么

短期记忆属于当前 `thread_id`，主要保存最近几轮消息、工具调用和对话摘要。

用户先说“数据库用 PostgreSQL”，下一句只说“连接池设成 20”，Agent 需要靠最近消息理解这个指代。这里用普通 message 表或 checkpoint 就够了，没有必要做语义搜索。

历史不能一直原样累加。较早的对话可以压缩成摘要，最近几轮保留原文，大型工具结果只保留结论和文件位置。否则聊得越久，费用和延迟越高，旧信息还会干扰当前判断。长上下文研究也发现，模型对位于上下文中部的信息利用并不稳定。[1]

## 任务记忆：这件事做到哪里

聊天记录告诉我们“说过什么”，任务状态回答“现在是什么情况”。

```json
{
  "goal": "为客服 Agent 增加跨会话记忆",
  "constraints": ["按用户隔离", "P95 小于 800ms"],
  "decisions": ["使用 PostgreSQL 保存事实源"],
  "completed": ["消息表", "用户 Profile"],
  "next": ["增加历史经验召回"],
  "open_questions": ["默认保留多久"]
}
```

任务状态应该结构化、可覆盖更新，并在每轮稳定加载。它不适合埋在向量库里等相似度碰运气。

对于需要跑几十轮的 Agent，这一层往往比所谓“永久记忆”更重要。没有它，模型很容易重复已经完成的步骤，或者把讨论过但否决的方案重新捡回来。

## 长期记忆：下次还值得知道什么

长期记忆跨会话存在。实际实现时，我会区分三类：

- **Semantic memory**：用户和项目事实，例如语言偏好、技术栈和业务约束；
- **Episodic memory**：过去事件及结果，例如某次降延迟方案是否有效；
- **Procedural memory**：工作规则，例如迁移数据库前必须准备回滚。

固定字段直接放 Profile：

```json
{
  "language": "zh-CN",
  "preferred_code": "Python",
  "timezone": "Asia/Shanghai"
}
```

固定且高频使用的事实适合结构化 Profile；不断增长的事实和历史事件适合 Collection，需要时再检索。Procedural memory 通常落在 System Prompt、配置或 Skills 中，而不是和聊天片段混在一起。LangChain/LangMem 也采用 semantic、episodic 和 procedural 的分类，并区分 Profile 与 Collection。[2]

## 外部知识不等于长期记忆

Agent 回答问题时还会读取产品文档、公司制度、代码仓库或实时 API。这些资料属于知识 RAG，不属于某个用户的长期记忆。

它们在读取时确实很像：

```text
当前问题 → 检索候选 → 重排 → 选几条放进 Context
```

差别主要出现在内容的归属和写入端。知识 RAG 通常来自文档同步；Agent Memory 来自对话、动作和反馈，每轮都可能变化。

一份产品文档很少因为用户说了一句话就失效。用户偏好却会变，任务决定也会被推翻。因此 Memory 还要处理：该不该写、覆盖还是新增、旧记录何时过期、来源是否可信。

可以共用检索组件，但最好拆开作用域：

```text
knowledge/company/policies
knowledge/project/api-docs

memory/user/42/profile
memory/project/17/decisions
memory/agent/coding/episodes
```

不先做 namespace 和权限过滤，检索质量再高也可能把别人的记忆找回来。

## 向量数据库放在长期记忆的检索层

向量数据库没有消失，只是回到了更准确的位置。下面这些内容适合向量检索：

- 数量很多的长期事实；
- 过去任务和结果；
- 用户可能换一种说法提到的历史事件。

当前任务状态、姓名、权限、订单号、系统规则，则应该通过结构化字段或精确搜索读取。

原始 message 也要保留在普通数据库中。Memory 是从消息里提炼出的派生记录，模型提炼错了，还得回到原文核验。

一个简化的数据结构可以是：

```sql
CREATE TABLE memories (
  id          uuid PRIMARY KEY,
  namespace   text[] NOT NULL,
  type        text NOT NULL,
  content     text NOT NULL,
  source_ids  text[] NOT NULL,
  confidence  real NOT NULL,
  valid_from  timestamptz NOT NULL,
  valid_to    timestamptz,
  embedding   vector(1536)
);
```

检索时也别只看向量相似度。错误码、专有名词和 ID 更适合关键词搜索；旧记录即使语义很像，也不一定应该排在前面。

我会先按用户、项目、权限和时间过滤，再组合 BM25、向量相似度、时效性与置信度，最后做一次 rerank。Anthropic 的 Contextual Retrieval 实验也显示，embedding 与 BM25、上下文化 chunk 和 reranking 组合优于单独 embedding。[3]

## 一轮消息里，这些记忆怎么配合

![一轮消息中的记忆读取、上下文组装、执行与异步写回流程](/assets/images/agent-memory/memory-turn-loop.svg)

*图：读取走在线路径；记忆提取、合并和失效可以在回复后异步完成。*

用户消息进来以后，在线路径尽量短：

```text
识别用户与任务
→ 读取当前 Task State
→ 检索少量相关记忆
→ 拼上摘要和最近消息
→ 调用模型
```

本轮 Context 可以按这个顺序装配：

```text
System / Developer Rules
Runtime Context：用户、权限、时间
Current Task State
Retrieved Memory
Retrieved Knowledge
Conversation Summary
Recent Complete Turns
Current User Message
```

最近完整 turn、当前目标和当前请求不要丢。工具调用与对应的 tool result 也要成对保留。旧日志和大文件只留引用，需要时再读。

回复完成后再走写入路径：

```python
for candidate in extract_memories(completed_turn):
    if worth_remembering(candidate):
        reconcile(candidate)  # 新增、合并、更新或失效
```

异步写入减少了主链路延迟，也方便一次性处理多个重复事实。明确的“请记住这件事”可以同步写，其他内容没必要边聊边存。

## 哪些话值得记

我会保存稳定偏好、已确认事实、关键决定及原因、未完成承诺，以及以后可能复用的成功或失败经验。

寒暄、临时要求、Agent 自己的猜测、大段工具输出和已经失效的状态，通常不进入长期记忆。

这两句话看起来很像，作用域却不同：

```text
“这次回答简单一点。”      → 当前 message
“以后回答都简洁一点。”    → 候选长期偏好
```

即使是第二句，也要保存来源和更新时间。以后用户改口说“技术方案请详细写”，系统才能知道应该更新哪个字段，而不是留下两条互相打架的 embedding。

当前请求与旧记忆冲突时，优先使用当前请求；当前确认状态与旧事实冲突时，让旧记录失效。Memory 只能辅助理解，不能覆盖系统权限和用户这一次明确表达的意图。

## 把结构落到存储上

```text
PostgreSQL
├── messages             原始消息
├── thread_checkpoints   短期会话
├── task_states          目标、进度和待办
├── profiles             当前用户/项目画像
└── memories             长期事实和历史事件
     └── pgvector        需要时再加

Object Storage
└── 附件、大型工具结果和归档
```

这几张表分别承接不同层：`messages` 和 checkpoint 是短期记忆，`task_states` 保存当前任务，`profiles` 与 `memories` 保存长期信息；对象存储则接住不适合长期占用 Context 的大文件。

落地顺序也很朴素：先保存 messages，再做 task state 和对话摘要，然后增加 Profile。等这些层次稳定以后，再给长尾 memory 加向量检索。

上线前至少测四件事：该记的能否写进去，需要时能否找到，不相关的信息会不会误召回，删除后正文、索引和缓存能否一起消失。多用户系统还要单独测试跨用户和跨项目泄漏。

一套 Memory 结构最终要回答三句话：刚才发生了什么，现在做到哪里，以后还要记住什么。

向量数据库只回答“哪些历史内容语义上可能相关”。它很有用，但排在分层、作用域、来源和更新规则之后。

## 参考资料

1. [Lost in the Middle：How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
2. [LangChain：Memory overview](https://docs.langchain.com/oss/python/concepts/memory)；[LangMem：Core Concepts](https://langchain-ai.github.io/langmem/concepts/conceptual_guide/)
3. [Anthropic：Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)
4. [Anthropic：Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
5. [MemGPT：Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560)
