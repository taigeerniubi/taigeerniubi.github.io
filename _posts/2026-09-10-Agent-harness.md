---
layout: post
title: "Agent Harness 是什么？从一次工具调用看懂执行循环、上下文与 MCP"
date: 2026-09-10 00:00:00 +0800
categories: [AI工程]
tags: [Agent, Harness, MCP, 上下文管理]
description: "用修复失败测试的例子，解释 Agent Harness 如何组织工具调用、上下文和权限，以及它与 ReAct、Skills、Memory、MCP 的关系。"
excerpt: "模型提出修改方案之后，谁负责读取文件、执行修改、运行测试，又由谁判断任务是否完成？从一次失败测试的修复过程开始，看懂 Agent Harness 的核心职责与工程边界。"
---

你让一个编程 Agent 做一件事：**“修复这个失败的测试，改完再跑一次。”**

它需要读取报错、找到代码、修改文件、运行测试；如果测试依然失败，还要根据新结果继续排查。模型可以提出这些动作，但实际读写文件、启动测试进程、保存执行记录，需要模型之外的程序来完成。

**Agent Harness，就是围绕模型组织执行、反馈、上下文和权限的运行系统。**

本文先跟着这一个任务看它如何工作，再解释 ReAct、Tools、Skills、Memory 和 MCP 分别处在什么位置。这里的分层是用于理解的工程模型；涉及 Claude API 等具体接口时，会单独说明适用范围。

![Agent Harness 的系统边界：上下文、循环、权限、工具执行、状态以及外部资源](/assets/images/agent-harness/harness-system-map.svg)

*本文使用的 Harness 分层：模型提出下一步，Harness 负责组装上下文、控制循环、检查权限、执行工具并保存状态。参考 [Arize AI 的 Harness 架构](https://arize.com/blog/what-is-an-agent-harness/)重新绘制。*

## 一、先看一个任务：修复失败的测试

假设项目里有一个折扣函数，测试要求输入 `100`、折扣率 `0.2` 时返回 `80`，实际却返回了 `20`。下面是用于解释分工的示例轨迹，并非某次真实运行记录。

| 步骤 | 模型提出什么 | Harness 做什么 | 得到什么反馈 |
| --- | --- | --- | --- |
| 1 | 读取测试和相关代码 | 检查读取范围，调用文件工具 | 测试断言与函数实现 |
| 2 | 将“折扣金额”改成“折后金额” | 检查写入权限，执行修改 | 修改成功，或补丁应用失败 |
| 3 | 运行对应测试 | 启动测试进程，记录退出码和输出 | 测试通过，或新的错误信息 |
| 4 | 根据结果继续或提交结论 | 控制剩余预算，保存记录并返回结果 | 修改说明与验证证据 |

这里有一个容易忽略的区别：模型说“已经修复”，只是生成了一句话；测试进程实际退出、返回了成功状态，才是可以检查的执行证据。

当然，一个测试通过也只证明覆盖到的行为通过了检查。这个任务是否完成，还取决于事先约定的验收范围，例如是否需要跑相关回归测试。

这就是 Harness 的作用：**让模型的下一步判断，接上实际环境返回的结果。**

## 二、最小骨架：模型与工具的循环

把上面的过程压缩一下，就得到最常见的工具调用循环：

![修复失败测试时，Harness 组装上下文、调用模型、执行工具并回填结果的完整循环](/assets/images/agent-harness/agent-task-loop.svg)

*循环是否结束，需要同时查看模型输出、接口停止原因、剩余预算和任务验收证据。*

对常见的无状态推理接口来说，模型不会在两次请求之间自动保存你的会话。历史消息、工具结果和任务进度，需要由应用或服务端会话机制保存，并在后续推理时提供。

一个最小 Harness 的控制逻辑可以写成下面这样。**这是与具体 SDK 无关的伪代码，重点是职责，不是可直接运行的 API 示例。**

```python
while budget.remaining():
    context = assemble_context(task, history, relevant_memory)
    response = model.generate(context, available_tools)
    history.append(response)

    if response.requires_tools:
        for call in response.tool_calls:
            try:
                validate_arguments(call)
                require_permission(call)
                result = execute(call, timeout=tool_timeout)
            except PermissionDenied as error:
                result = denied_result(error)
            except ToolError as error:
                result = failure_result(error)

            history.append(tool_result(call.id, result))
        continue

    if response.finished:
        return report_with_evidence(response, history)

    handle_interruption_or_truncation(response)

return report_incomplete(reason="budget_exhausted", history=history)
```

循环之外，还要处理模型 API 请求失败、用户取消、状态落盘等问题。即使是小型实现，也建议从一开始设置最大轮数、工具超时和总预算，避免同一个错误被反复重试。

“没有工具调用”也不总等于任务结束。例如 Claude API 区分 `end_turn`、`max_tokens`、`pause_turn` 等停止原因，处理方式不同。[1]

### ReAct 和这个循环是什么关系？

ReAct 的核心思想是让推理与行动交替发生：根据当前信息决定动作，再利用环境反馈更新判断。[2]

它与结构化工具调用并不冲突。你可以用文本格式实现 ReAct，也可以使用原生工具调用接口实现相同的思路。更合适的比较对象，是两种工具调用方式：

| 比较点 | 基于文本约定的调用 | 原生结构化工具调用 |
| --- | --- | --- |
| 动作表达 | 模型输出约定格式的动作文本 | 接口返回工具名称、参数等字段 |
| 解析 | 应用解析文本并校验 | 应用读取结构化调用对象 |
| 调用与结果对应 | 应用自行约定 | 常见接口提供调用 ID |
| 仍需处理的问题 | 格式偏离、选错工具、执行失败 | 参数约束、选错工具、执行失败 |

结构化接口能减少协议处理上的歧义，但不会自动判断“这个工具是否选对了”。ReAct 也不是一种只能靠正则解析的旧格式。

![ReAct 将思考、行动与环境观察交替组织起来的论文示意图](/assets/images/agent-harness/react.png)

*ReAct 论文把 reasoning trace、action 和 observation 交替组织起来。图源：[ReAct 论文](https://arxiv.org/abs/2210.03629)。*

## 三、格式正确，为什么仍然需要权限和验证？

假设模型请求执行：

```json
{"tool": "run_tests", "arguments": {"target": "tests/test_discount.py"}}
```

这里至少有三件不同的事需要检查：

1. **格式是否合规**：`target` 是否存在，类型是否正确？
2. **动作是否被授权**：是否允许在这个工作区启动测试进程？
3. **结果是否支持结论**：运行的是相关测试吗？退出码和输出是否表明验证通过？

### JSON Schema 的保证有条件

声明工具参数的 JSON Schema，不代表所有接口都会自动开启严格约束。

以 Claude API 为例，官方区分普通工具调用与 `strict: true` 的严格工具调用。后者可以保证参数满足受支持的 Schema 约束，但仍不能保证业务含义正确。[3]

例如 `target` 是字符串，只能证明类型正确；字符串可能指向不存在的文件，也可能指向一个与这次修改无关的测试。

### 调用 ID 解决配对，不解决真实性

Claude API 的客户端工具结果通过 `user` 消息里的 `tool_result` 内容块回填，并用 `tool_use_id` 对应请求。这是该 API 的设计，不是所有模型接口通用的消息格式。[3]

这样的结构有助于避免把 A 调用的结果误配给 B，但模型仍可能在正文中声称“测试通过”。应用需要保存实际退出码、输出与产物，才能核对结论。

工具返回的网页、文件和日志也可能包含不可信指令。它们应作为待处理的数据，不能因为进入了上下文就被当成更高优先级的系统规则。[3]

### 权限与验证各做一件事

权限规则约束哪些动作可以执行；沙箱限制执行环境的文件、网络等访问范围；测试和结果校验检查任务是否达到目标。需要人工确认的操作，则根据已有授权和风险决定何时暂停。

Hooks 可以在工具执行前后触发程序逻辑，也能用于阻断操作或补充上下文。它并不只属于“输入侧”。对于必须执行的固定检查，应放进程序控制流、Hooks 或 CI 等机制，而不是仅在提示词里要求模型记住。[4]

## 四、Tools、Skills、Memory、MCP 分别负责什么？

回到修复测试的例子，这几个概念可以这样区分：

| 概念 | 主要回答的问题 | 在例子中的作用 |
| --- | --- | --- |
| Tools | 能执行什么动作？ | 读文件、应用补丁、运行测试 |
| Skills | 某类任务应该怎么做？ | 提供调试流程、项目约定和辅助脚本 |
| Memory | 哪些历史信息值得继续使用？ | 记住项目测试命令或已确认的偏好 |
| MCP | 如何标准化接入外部能力与上下文？ | 连接问题跟踪系统、文档资源或其他工具 |

Harness 负责把这些能力组织进执行过程。一个简单 Agent 可以只有两三个工具，不必一开始就配置全部资源。

![Tools、Skills、Memory 与 MCP 分别解决动作、方法、历史和标准接入问题](/assets/images/agent-harness/capability-map.svg)

*四个概念最终由 Harness 组织进执行循环。MCP 可以提供 Tools，也可以提供 Resources 和 Prompts，因此不能简单等同于“工具商店”。*

### Tools：真正执行动作的接口

模型根据工具描述选择调用，实际代码负责执行。工具设计除了参数格式，还应该说明作用范围、返回值、常见错误，以及是否会产生副作用。

例如，“运行测试”工具除了返回一段文字，还可以返回退出码、耗时和输出是否截断。这样模型更容易区分“测试失败”和“测试进程根本没有启动”。

### Skills：可复用的任务知识与工作流程

把 Skill 理解成“操作手册”很有帮助，但它不一定只有一段 prompt。Agent Skills 的目录可以包含 `SKILL.md`、脚本、参考资料和模板。[5]

例如一个调试 Skill 可以约定：先复现错误，再缩小修改范围，最后运行相关测试。它指导模型如何使用工具；附带的脚本仍需要通过执行机制运行。

### Memory：保存以后还值得使用的信息

项目测试命令、已经确认的设计约束，可以写入持久存储；临时猜测和已经过时的判断，不应该未经筛选就反复注入。

存下来的信息也不会自动成为模型当前可用的知识。系统需要在合适的时候读取、检索或注入它。记忆最好保留来源和更新时间，影响当前操作时再检查是否仍然适用。

### MCP：不只提供工具

MCP 定义了客户端、宿主与服务器之间的交互，并包含 Tools、Resources、Prompts 等能力。[6]

当宿主把 MCP 工具转换为模型能调用的工具接口后，模型可以像选择其他工具一样选择它。宿主再把请求转发给 MCP Server，并处理结果。

区别在于接入协议，而不在于“原生工具一定进程内执行、MCP 工具一定远程执行”。原生工具也可以调用外部服务，MCP Server 也可以部署在本地。

![MCP Host、Client 与多个 Server 之间的连接关系](/assets/images/agent-harness/mcp-architecture.png)

*MCP 采用 Host—Client—Server 架构，并可暴露 Tools、Resources 和 Prompts。图源：[MCP 官方架构](https://modelcontextprotocol.io/specification/2025-06-18/architecture)。*

## 五、上下文有限：该常驻什么，该按需读取什么？

修复任务进行几轮后，上下文会积累源码、测试日志、失败尝试和中间结论。把所有内容一直保留，既占窗口，也可能掩盖关键约束。

比较实用的策略是：**核心信息稳定保留，长尾材料按需读取。**

| 信息 | 一种可选策略 | 需要防范什么 |
| --- | --- | --- |
| 用户目标、关键约束 | 保留在当前任务状态中 | 压缩后丢失验收标准 |
| 少量核心工具 | 保留完整描述与参数定义 | 工具描述含糊，容易误用 |
| 大量长尾工具、Skills | 保留可发现的索引，按需加载 | 搜不到，或描述不足以判断用途 |
| 大文件、长日志 | 提取相关片段，保留原始位置 | 截断隐藏关键错误 |
| 已完成的探索过程 | 总结结论、证据与未解决问题 | 摘要遗漏重要细节 |

这是一组设计选择，并不意味着所有资源都应该懒加载，也没有通用的“核心工具必须约 15 个”的规定。

### 压缩会丢信息，但不一定永久丢失

上下文压缩通常是有损的：模型当前看到的摘要无法保留所有细节。如果原始会话、文件或日志仍在，就可以再次读取。摘要最好记录证据位置，而不是只留一句“已经检查过”。

### 缓存降低开销，不扩展窗口

Prompt caching 可以复用相同的提示前缀，降低重复处理的成本和延迟。它不负责替你保存任务记忆，也不消除上下文长度限制。[7]

稳定的系统指令和工具定义通常有助于复用缓存，但“改一个字，所有缓存都失效”过于绝对。具体失效范围取决于接口的前缀和缓存规则，价格也要以对应模型的文档为准。[7]

如果每轮增加近似固定长度的信息，又始终重发全部历史，累计输入 token 量会随轮数呈平方级增长。缓存能降低重复部分的处理成本，仍然需要配合上下文管理。

## 六、任务变大以后，再考虑子 Agent 与恢复机制

### 子 Agent：把独立工作交出去

假设修复前需要搜索整个项目的类似用法，可以让子 Agent 完成检索，主 Agent 只接收相关位置和结论。这能减少主会话被大量搜索结果占满的情况。

但要区分两种隔离：

- **上下文隔离**：子 Agent 使用自己的对话记录；是否继承主历史由实现决定。
- **工作区隔离**：子 Agent 是否共享文件，还是使用独立 worktree，需要另外配置。

独立上下文并不意味着它看到的文件系统停留在启动时刻。共享目录中的文件可能继续变化；独立 worktree 也不会自动隔离数据库或远程服务。[8]

值得交出去的任务通常边界清楚、依赖较少、结果容易汇总。委派也有启动、通信和验证成本，不一定比主 Agent 直接完成更划算。

![协调者动态拆分任务、交给多个 Worker，并汇总结果](/assets/images/agent-harness/orchestrator-workers.png)

*Orchestrator-workers 模式：协调者动态拆分任务，Worker 分别执行，最终由协调者综合结果。图源：[Anthropic](https://www.anthropic.com/engineering/building-effective-agents)。*

### 汇总要保留失败与证据

子任务失败如何返回，取决于编排接口，不能假设它一定是 `null`。一个更有用的结果结构是：

```json
{
  "task_id": "review-discount",
  "status": "failed",
  "summary": "测试环境未启动，尚未完成验证",
  "evidence": ["logs/test-startup.txt"],
  "unresolved": ["需要恢复测试依赖后重新验证"]
}
```

结构化字段让遗漏更容易被发现，却不能保证结论正确。功能审查、安全审查和复现检查也不能简单按多数票决定：两个维度通过，无法抵消另一个维度发现的真实缺陷。

### 恢复执行：特别注意副作用

长任务中断后，至少要知道目标是什么、哪些步骤已完成、产物在哪里、哪些检查还没做。会话日志、检查点和工作流重放，是可采用的不同机制。

这里最容易踩坑的是：**工具已经修改成功，但成功记录还没落盘，进程就崩溃了。** 恢复后再执行一次，可能产生重复副作用。

所以，安全重试依赖操作本身的幂等设计。例如“确保配置项为某值”通常比“再追加一行配置”更容易重复执行；必要时还要在执行端使用任务 ID 去重。只在提示词里写“不要重复”，不能形成可靠保证。

采用确定性工作流重放时，时间、随机数、模型输出和外部调用需要遵守框架的记录与重放规则。例如 Temporal 区分工作流逻辑与外部副作用，并提供相应机制；不能把它概括成所有日志系统都禁止使用时钟。[9]

## 七、判断一个 Harness 是否够用，看执行证据

假设开头的折扣测试已经通过，接下来应该检查的是这些问题：

| 检查项 | 应该能回答的问题 |
| --- | --- |
| 任务完成 | 验收条件是什么？有哪些实际验证结果？ |
| 执行记录 | 修改了哪些文件？调用了哪些工具？ |
| 错误处理 | 哪一步失败？是否重试？还有什么没完成？ |
| 权限控制 | 操作是否在已授权范围内？ |
| 运行边界 | 是否限制轮数、时间、费用和并发？ |
| 状态恢复 | 中断后能找到进度与产物吗？重试会重复修改吗？ |

进一步比较两种 Harness 设计时，可以固定模型、任务集和预算，记录任务成功率、耗时、成本以及人工介入次数。只看一次演示成功，很难知道改进来自系统设计，还是任务本身碰巧容易。

对于自己的第一个实现，可以从少量工具、一个有预算上限的循环、一份执行日志和明确的验收条件开始。遇到具体瓶颈后，再加入检索、Skills、子 Agent 或更复杂的恢复机制。

回到最初的修复任务：一个可检查的最终交付，应该同时包含**代码修改、验证结果和未完成事项**。这三样，比模型一句“已完成”更能说明系统是否把事情做到了位。

## 参考资料

以下资料用于区分通用设计与具体接口行为。产品功能会更新，实现相关结论应结合所用版本确认。

1. [Claude：Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)
2. [ReAct：Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
3. [Claude：Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)
4. [Claude Code：Hooks reference](https://code.claude.com/docs/en/hooks)
5. [Agent Skills：格式与能力介绍](https://agentskills.io/home)
6. [MCP：Architecture，2025-06-18 协议版本](https://modelcontextprotocol.io/specification/2025-06-18/architecture)
7. [Claude：Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
8. [Claude Code：Subagents](https://code.claude.com/docs/en/sub-agents)；[Worktrees](https://code.claude.com/docs/en/worktrees)
9. [Temporal：Workflow Determinism](https://community.temporal.io/t/workflow-determinism/4027)

延伸阅读：[Anthropic：Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)。文中的 augmented LLM、orchestrator-workers 等模式可以帮助理解相关设计，但不应被视为唯一的 Harness 结构标准。
