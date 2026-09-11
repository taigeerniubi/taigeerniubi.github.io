---
layout: post
title: "Agent Harness 到底管什么？从一句“测试通过”说起"
date: 2026-09-10 00:00:00 +0800
categories: [AI工程]
tags: [Agent, Harness, MCP, 上下文管理]
description: "从修复失败测试的过程出发，解释 Agent Harness 如何接住模型的判断，并负责工具、权限、状态和验证。"
excerpt: "模型可以说测试通过了，但谁真正运行测试、记录退出码、处理失败和保存进度？这些工作属于 Agent Harness。"
---

让一个编程 Agent 修复失败测试，看起来只是一句话：

> 修好它，然后再跑一次测试。

模型读完代码，很快给出修改方案。接着它说：“测试已经通过。”

问题是，测试真的跑了吗？跑的是哪个文件？进程退出码是多少？如果修改已经写进磁盘，但保存执行记录之前程序崩了，下次恢复会不会再改一遍？

这些事都发生在模型外面。负责接住它们的那层系统，就是 Agent Harness。

![Agent Harness 的系统边界：上下文、循环、权限、工具执行、状态以及外部资源](/assets/images/agent-harness/harness-system-map.svg)

*图：模型负责判断下一步，Harness 负责让这一步安全地落到真实环境里。*

## 模型只负责“想”，谁负责“做”

还是看修测试这件事。

模型可能依次提出：读取报错、打开相关函数、修改一行代码、运行测试。真正执行这些动作的，是模型外围的程序。它要检查路径是否允许访问，调用文件或终端工具，把结果放回对话，再决定是否继续。

我更愿意把 Harness 理解成 Agent 的运行时。它至少要处理五件事：

- 给模型准备这一轮需要的上下文；
- 把模型输出解析成工具调用；
- 在执行前检查参数和权限；
- 保存消息、工具结果、任务进度；
- 判断什么时候真的可以结束。

这也解释了为什么“模型更聪明”不能替代 Harness。模型可以更准确地选择动作，却无法凭一句自然语言让文件发生变化。真正产生副作用的永远是执行层。

## 最小循环其实很短

![修复失败测试时，Harness 组装上下文、调用模型、执行工具并回填结果的完整循环](/assets/images/agent-harness/agent-task-loop.svg)

*图：每次工具结果都会进入下一轮判断，直到满足结束条件或耗尽预算。*

把框架和 SDK 都拿掉，一个 Agent 循环大概只有这些东西：

```python
while budget.available():
    context = build_context(task, recent_messages, relevant_memory)
    response = model.generate(context, tools)

    if response.tool_calls:
        for call in response.tool_calls:
            check_arguments(call)
            check_permission(call)
            result = execute(call)
            save_tool_result(call.id, result)
        continue

    if completion_checks_pass(response):
        return response
```

难点不在 `while`，而在每个看起来普通的函数。

`build_context` 需要决定哪些历史消息还值得保留；`check_permission` 要区分读取代码和修改生产数据；`execute` 要处理超时、截断和部分成功；`completion_checks_pass` 则要找到模型声明之外的证据。

以测试为例，下面三句话完全不同：

```text
“我认为修改可以修复问题”
“我调用了测试工具”
“测试进程退出码为 0，相关回归测试也通过”
```

第一句是推断，第二句是动作，第三句才接近可验证结果。

## Tool call 结构正确，不代表事情做对了

假设模型返回：

```json
{"tool": "run_tests", "arguments": {"target": "tests/test_discount.py"}}
```

JSON Schema 可以约束 `target` 是字符串，却不能保证这个文件存在，更不能保证它覆盖了本次修改。调用 ID 可以把请求与结果配对，也不能证明结果是真实的。

因此 Harness 需要分别处理三层检查：

1. **格式**：名称和参数是否符合工具协议；
2. **授权**：这个用户是否允许执行这项操作；
3. **结果**：环境状态是否满足任务验收条件。

这三层最好落在确定性代码里。把“修改数据库前先确认”“完成后必须跑测试”只写在 prompt 中，意味着每一轮都在赌模型有没有记住。

工具返回值也应该尽量结构化。与其只返回一段 `success`，不如明确提供：

```json
{
  "status": "failed",
  "exit_code": 1,
  "duration_ms": 842,
  "output_ref": "logs/test-20260910.txt",
  "truncated": true
}
```

这样模型能区分“测试失败”“进程没启动”和“输出被截断”。日志仍然保存在外部，需要时再读取，不必每轮把几万行输出重新塞回上下文。

## Tools、Skills、Memory、MCP 别揉成一个词

这几个概念经常一起出现，但它们解决的问题不同。

![Tools、Skills、Memory 与 MCP 分别解决动作、方法、历史和标准接入问题](/assets/images/agent-harness/capability-map.svg)

*图：它们最终都会进入 Harness，但各自的职责不同。*

**Tools** 是动作接口，比如读文件、运行测试、发送请求。模型选择工具，Harness 执行工具。

**Skills** 更像操作手册。一个调试 Skill 可以要求先复现、再定位、最后回归；其中附带的脚本仍要通过工具执行。

**Memory** 保存以后还值得使用的信息，例如项目测试命令、已经确认的约束和上次失败的原因。存下来并不等于每轮都加载，Harness 仍要判断什么时候取用。

**MCP** 是接入外部能力与上下文的一套协议，可以暴露 Tools，也可以暴露 Resources 和 Prompts。[1] 它解决连接方式，不负责整个 Agent 循环。

真正把这些东西串起来的，还是 Harness。

## 上下文很容易越跑越脏

一个修复任务经过十几轮后，Context 里可能已经堆着源码、长日志、失败尝试、旧计划和多次工具返回。全部保留看似保险，实际会让当前目标越来越不显眼。

我的取舍是：目标、约束和当前计划稳定保留；大文件和长日志只留位置；失败探索压缩成结论；最近几轮保留原文。

```text
当前目标与验收条件
当前任务状态
相关代码或文档片段
已压缩的早期过程
最近完整消息与工具结果
当前请求
```

Prompt caching 能减少相同前缀的重复处理成本，但不会替你管理状态，也不会扩大模型的有效注意力。[2] Context 还是需要主动裁剪。

## 中断恢复，最怕重复副作用

长任务迟早会遇到进程崩溃、超时、网络中断或用户取消。恢复时至少要知道：目标是什么、哪些步骤完成了、产物在哪里、哪些验证还没做。

有一种危险情况很常见：工具已经执行成功，成功记录却还没落盘。恢复后 Harness 再执行一次，就可能重复发消息、重复扣款或重复修改数据。

所以高风险工具需要幂等设计。例如使用稳定的任务 ID 去重，或者把“追加一条配置”改成“确保配置项等于某值”。只告诉模型“不要重复操作”不够可靠。

## 第一版不必做得很大

如果从零开始，我会先做这几个部分：

1. 三到五个边界清楚的工具；
2. 一个带最大轮数、超时和取消能力的循环；
3. 完整保存 message、tool call 与 tool result；
4. 对写操作做明确的权限检查；
5. 为任务完成准备一项环境侧验证。

先把一条链路跑得可检查，再增加 Memory、Skills、子 Agent 或复杂恢复机制。

判断 Harness 是否够用，也不必先看架构图。看它能不能回答几个朴素问题就行：改了什么，运行了什么，结果在哪里，失败后发生了什么，还有哪些事没做。

模型说“完成”很容易。Harness 的价值，是让这句话后面有证据。

## 参考资料

1. [Model Context Protocol：Architecture](https://modelcontextprotocol.io/specification/2025-06-18/architecture)
2. [Claude：Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
3. [Claude：Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)
4. [Claude Code：Hooks reference](https://code.claude.com/docs/en/hooks)
5. [ReAct：Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
6. [Anthropic：Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
