---
layout: post
title: "Agent Memory 怎么搭：别把聊天记录全塞进向量数据库"
date: 2026-09-11 00:00:00 +0800
categories: [AI工程]
tags: [Agent, Memory, RAG, 上下文管理]
description: "从短期记忆、任务状态和长期记忆出发，给出一套可落地的 Agent Memory 分层、写入、召回与上下文装配方案。"
excerpt: "向量数据库只是 Agent Memory 的一个检索索引。真正的难点，是决定什么值得记、放在哪里、何时召回，以及新旧信息冲突时相信谁。"
---

用户昨天告诉 Agent：“这个项目用 Python，响应延迟必须低。”今天换了一个新会话，又说：“继续把 memory 做完。”

要让 Agent 接得住这句话，系统至少要知道三件事：用户长期偏好什么，项目当前做到哪里，最近几轮具体说过什么。把所有聊天记录做 embedding，再从向量数据库里捞几段，看起来能跑，却很快会遇到重复、冲突、召回错误和上下文膨胀。

**Agent Memory 不是一个数据库功能，而是一套围绕信息写入、更新、召回、压缩和遗忘的上下文系统。**

本文给出一套适合从 MVP 走向生产的分层模型，并重点回答两个问题：向量数据库应该放在哪里，以及每轮 message 应该怎样管理。

![Agent Memory 的四层结构与外部知识边界](/assets/images/agent-memory/memory-system-map.svg)

*图：模型真正看到的是本轮 Working Context；短期记忆、任务状态、长期记忆和知识 RAG 都只是它的输入来源。*

## 一、先分清四种“记忆”

### 1. Working Context：模型此刻正在看的内容

Working Context 是一次模型调用的输入，包括系统指令、当前任务、相关记忆、最近消息和工具结果。它类似人的工作台，而不是仓库。

模型不会因为数据库里存在某条记录就自动知道它。只有被选中并装进本轮 Context 的信息，才会参与这次推理。

这带来第一个设计原则：

> 数据库负责保存，Context 负责思考；保存得多，不等于模型知道得更多。

### 2. Short-term Memory：当前会话刚刚发生了什么

短期记忆通常以 `thread_id` 为边界，保存：

- 最近几轮 user、assistant message；
- 尚未结束的工具调用及结果；
- 较早对话的压缩摘要；
- 本会话上传文件或临时产物的引用。

它主要解决指代和连续性。例如用户先说“用 PostgreSQL”，下一句只说“连接池设成 20”，Agent 需要知道后一句在修改什么。

短期记忆适合 checkpoint、关系数据库或 Redis，不需要为了“记住最近十条消息”引入向量数据库。

### 3. Task Memory：这件事现在做到哪里

任务记忆保存的是当前事实状态，而不是聊天流水：

```json
{
  "goal": "为客服 Agent 增加跨会话记忆",
  "status": "implementing",
  "constraints": ["数据按用户隔离", "P95 延迟低于 800ms"],
  "decisions": ["使用 PostgreSQL + pgvector"],
  "completed": ["消息表", "用户 Profile"],
  "next_steps": ["实现 episodic memory 召回"],
  "open_questions": ["记忆默认保留多久"]
}
```

这是长任务能否稳定推进的关键。只依赖 message history，Agent 很容易忘记验收标准、重复已完成步骤，或把讨论过但已经否决的方案当成最终决定。

Task Memory 应该结构化、可覆盖更新，并在每轮稳定加载。它通常也不需要向量检索。

### 4. Long-term Memory：跨会话仍然值得知道什么

长期记忆跨线程存在，可以进一步分成三类：

| 类型 | 保存什么 | 常见实现 |
| --- | --- | --- |
| Semantic | 用户偏好、项目事实、业务约束 | JSON Profile 或可检索事实集合 |
| Episodic | 过去发生的事情、采取的动作和结果 | 事件记录 + 向量索引 |
| Procedural | Agent 应怎样工作 | System Prompt、配置、Skills |

例如“用户希望代码示例带类型标注”是 semantic memory；“上次通过减少在线检索把 P95 从 1.8 秒降到 400 毫秒”是 episodic memory；“数据库迁移前必须准备回滚方案”则更适合成为 procedural memory。

LangChain/LangMem 也使用 semantic、episodic、procedural 的划分，并建议把固定结构的当前画像与需要按需检索的 memory collection 区分开。[1]

## 二、Memory 和 RAG 很像，但不是一回事

二者都可能使用 embedding、向量搜索和 reranker，所以从读取链路看非常相似：

```text
当前问题 → 生成查询 → 检索候选 → 重排 → 放进 Context
```

区别在于管理对象和生命周期。

| 维度 | Knowledge RAG | Agent Memory |
| --- | --- | --- |
| 主要对象 | 文档、知识库、数据库 | 用户事实、任务历史、Agent 经验 |
| 写入来源 | 文档同步或人工导入 | 对话、行为、工具结果和反馈 |
| 更新方式 | 文档版本更新 | 每轮都可能新增、覆盖或失效 |
| 核心难题 | 找到正确资料 | 决定什么值得记，以及何时遗忘 |
| 作用域 | 组织、项目、知识域 | 用户、线程、任务、Agent |

RAG 更像去图书馆查资料；Memory 更像管理自己的笔记。搜索是两者都会用到的能力，但笔记还需要整理、修订和删除。

因此，可以共用检索基础设施，但应分开 namespace：

```text
knowledge/company/policies
knowledge/project/api-docs

memory/user/42/semantic
memory/project/17/decisions
memory/agent/coding/episodes
```

## 三、向量数据库应该放在哪里

答案不是“所有 memory 都进去”，而是：**只给需要语义召回的长尾信息建立向量索引。**

适合向量检索的内容：

- 大量零散的长期事实；
- 过去任务及其结果；
- 用户可能换一种说法查询的历史事件；
- 无法通过明确 ID 或字段直接定位的内容。

不适合只靠向量检索的内容：

- 当前任务状态；
- 用户姓名、语言、权限等固定字段；
- 必须始终生效的系统规则；
- 订单号、错误码等精确标识符；
- 原始 message 的唯一事实记录。

工程上可以把 PostgreSQL 作为事实源，把 `pgvector` 当作同表中的一个索引：

```sql
CREATE TABLE memories (
  id            uuid PRIMARY KEY,
  namespace     text[] NOT NULL,
  memory_type   text NOT NULL,
  content       text NOT NULL,
  source_ids    text[] NOT NULL,
  importance    real NOT NULL DEFAULT 0.5,
  confidence    real NOT NULL DEFAULT 1.0,
  status        text NOT NULL DEFAULT 'active',
  valid_from    timestamptz NOT NULL,
  valid_to      timestamptz,
  embedding     vector(1536)
);
```

这里的 `source_ids` 很重要。Memory 是从对话中提炼出的派生数据；如果模型记错了，系统需要回到原始 message 或工具结果核验，而不是让错误摘要反复传播。

当数据量不大时，先用 JSONB、全文搜索和 metadata filter 就够了。等到确实出现“同义表达找不到历史信息”的问题，再增加向量检索，比一开始部署复杂的独立向量数据库更稳。

## 四、一轮消息如何流过 Memory 系统

![一轮消息中的记忆读取、上下文组装、执行与异步写回流程](/assets/images/agent-memory/memory-turn-loop.svg)

*图：在线路径负责低延迟回答；耗时的记忆提取、去重和合并放到异步写回路径。*

### 读取阶段：先过滤，再搜索

收到新 message 后，推荐按下面的顺序读取：

1. 用 `user_id`、`project_id`、权限和 memory 类型做硬过滤；
2. 对当前问题做关键词搜索与向量搜索；
3. 合并候选并用 reranker 重排；
4. 去掉重复、过期和互相矛盾的记录；
5. 在 token budget 内只取少量高价值 memory。

只使用向量相似度并不够。它可能漏掉精确错误码，也可能把语义相似但属于另一个项目的信息找回来。Anthropic 的 Contextual Retrieval 实验同样表明，embedding 与 BM25、上下文化 chunk 和 reranking 组合，比单独使用 embedding 更可靠。[2]

一个可用于起步的综合分数是：

```text
score = relevance × 0.45
      + keyword   × 0.20
      + recency   × 0.15
      + importance× 0.10
      + confidence× 0.10
```

这些权重不是行业标准，必须用自己的任务集调整。真正重要的是：相关性不应该是唯一信号。

### 装配阶段：给注意力设置预算

每轮可以按下面的顺序装配 Context：

```text
System / Developer Rules
Runtime Context：用户、权限、时间和环境
Current Task State
Retrieved Long-term Memory
Retrieved Knowledge
Conversation Summary
Recent Complete Turns
Current User Message
```

始终保留当前请求、当前任务目标和最近完整 turn。工具调用与对应的 tool result 也不能被裁剪成不完整的消息序列。

较早的原始消息可以压缩为摘要；已经写入文件或数据库的大型工具输出，只保留结论、位置和稳定 ID。上下文的目标不是完整，而是让每个 token 都支持当前决策。Anthropic 将这种实践概括为 compaction、结构化笔记和 just-in-time retrieval。[3]

### 写入阶段：不是每句话都值得记

回复完成后，再异步提取长期 memory，避免增加用户可感知的延迟：

```python
def process_completed_turn(turn):
    candidates = extract_memory_candidates(turn)

    for candidate in candidates:
        if not worth_remembering(candidate):
            continue

        existing = find_related_memories(candidate)
        reconcile(candidate, existing)  # insert / update / merge / invalidate
```

建议记住：

- 用户明确表达的稳定偏好；
- 已确认的事实和重要约束；
- 关键决定及原因；
- 未完成承诺；
- 可复用的成功或失败经验。

通常不要长期保存：

- 寒暄和一次性要求；
- Agent 未经确认的推测；
- 重复事实；
- 已经失效的任务状态；
- 大段原始工具输出；
- 不必要的敏感信息。

“这次回答简单一点”是本轮指令；“以后都简洁回答”才可能成为长期偏好。Memory 写入的难点，恰恰是识别这种边界。

## 五、新旧记忆冲突时相信谁

假设 Profile 中写着“用户喜欢详细回答”，但当前 message 是“这次只回答三句话”。本轮显然应该服从当前请求。

可以把信息优先级拆成两套规则。

指令权限：

```text
System / Developer Policy
> Current User Instruction
> Memory
> 外部文档中的普通文本
```

事实新鲜度：

```text
当前确认状态
> 最近且有来源的事实
> 较旧的长期记忆
> Agent 推测
```

长期记录至少要包含 `valid_from`、`valid_to`、`last_confirmed_at`、`confidence` 和来源。新事实出现时，应更新、合并或失效旧记录，而不是让互相矛盾的 memory 永久留在向量库中竞争。

## 六、从零开始的最小实现

第一版不需要复杂的“记忆大脑”。下面这组组件已经可以支撑大多数单 Agent 产品：

```text
PostgreSQL
├── messages              原始消息，审计事实源
├── thread_checkpoints    短期会话状态
├── task_states           目标、约束、进度和待办
├── profiles              用户与项目当前画像
└── memories              长期事实与历史经验
      └── pgvector        可选的语义索引

Object Storage
└── 附件、大型工具结果和历史归档
```

实现顺序建议是：

1. 先持久化原始 messages；
2. 增加结构化 task state；
3. 对早期消息做摘要，保留最近完整 turn；
4. 增加用户 Profile；
5. 最后再为 episodic 和长尾 semantic memory 增加检索。

这个顺序很重要。很多 Agent 不是缺少向量数据库，而是没有把“刚才说了什么”“现在做到哪里”和“以后值得记住什么”分开。

## 七、上线前至少测这六件事

- **Write precision**：写入的内容是否真的值得长期记忆；
- **Recall**：需要某条 memory 时能否召回；
- **False recall**：是否注入了无关或错误记忆；
- **Conflict accuracy**：新旧信息冲突时是否选择正确版本；
- **Isolation**：是否可能跨用户、跨项目泄露；
- **Deletion**：用户删除后，正文、向量索引和缓存是否同步清除。

不要只测试“问名字能不能答出来”。更有价值的测试是：修改偏好后旧记录是否失效、任务中断后能否继续、语义相似但属于另一个项目的信息会不会误召回，以及删除请求能否真正清干净。

## 最后

Agent Memory 可以用一句话概括：

> 短期记忆维持对话连续，任务记忆维护当前进度，长期记忆保存跨会话仍有价值的信息，RAG 提供当前问题需要查询的外部知识；Context Engineering 决定这一轮究竟把哪些内容交给模型。

向量数据库只解决其中的“语义召回”。真正决定系统质量的，是边界、来源、状态、冲突规则和上下文预算。

## 参考资料

1. [LangChain：Memory overview](https://docs.langchain.com/oss/python/concepts/memory)；[LangMem：Core Concepts](https://langchain-ai.github.io/langmem/concepts/conceptual_guide/)
2. [Anthropic：Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)
3. [Anthropic：Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
4. [MemGPT：Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560)
5. [Lost in the Middle：How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
