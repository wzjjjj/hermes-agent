# Module 06 / Lesson 02：subagent_outputs_and_contracts

## 学习目标

- 能理解 Hermes 如何把子代理的“过程”压缩成父代理可用的结构化产物（JSON），并确保可审计。
- 能设计你自己的业务版“子代理输出契约”（contract）：字段、约束、失败语义、可组合性。
- 能理解 tool_trace 如何基于 tool_call_id 正确匹配并行工具调用与结果。

## 先修知识

- 已完成 Module 06 / Lesson 01（理解隔离与限权）
- 了解 tool_call_id 的意义（并行工具调用配对）

## 本节要回答的关键问题

- 父代理为什么“只接收总结，不接收子代理过程”？这对上下文成本/安全有什么好处？
- 子代理系统提示词如何约束输出结构？如何避免子代理输出一堆无用叙述？
- delegate_task 返回的 JSON 结构是什么？哪些字段是你业务里最该保留/扩展的？
- tool_trace 如何做到并行情况下仍能配对正确？

## 核心概念

- 输出契约（Output Contract）：把“子代理的自由文本”变成父代理可组合的结构化数据。
- 可审计最小轨迹（Minimal Trace）：不把完整过程塞回父上下文，但保留足够的元信息（哪些工具、结果大小、是否错误）。
- 失败语义清晰：区分 failed/error/interrupted/max_iterations，便于父代理决定重试/降级/改写任务。

## 代码走读路线

1. 设计原则：父代理不看到子代理过程（只看到 delegation 调用与 summary）
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L9-L16)
2. 子代理输出约束：系统提示词要求“完成后给总结清单”
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L68-L79)
3. 子代理结果结构化：status/summary/exit_reason/tokens/tool_trace
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L384-L467)
4. tool_trace 基于 tool_call_id 做并行配对
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L399-L436)

## 关键代码讲解

### 1) 为什么父代理不接收子代理过程

delegate_tool 文件头部明确写了设计目标：父上下文只看到 delegation 调用与 summary，不看到中间工具调用/推理过程。

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L9-L16)

收益：

- 成本：子代理过程往往很长，如果回传会迅速挤爆上下文窗口。
- 安全：子代理可能接触到敏感信息（文件/日志/凭据遮罩），只回传经过约束的 summary 更可控。
- 可靠：父代理只处理“结构化产物”，减少被子代理叙述带偏的风险。

### 2) 用系统提示词“先约束输出形态”

子代理系统提示词明确要求输出包含：

- What you did
- What you found or accomplished
- Any files you created or modified
- Any issues encountered

见：

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L68-L74)

这是一种非常实用的模式：与其指望子代理自己“总结得好”，不如提前定义输出骨架，让子代理以填空方式完成。

### 3) 结果结构化：让父代理能做决策与合并

`_run_single_child()` 会把子代理 `run_conversation()` 的结果包装成一个 dict，核心字段包括：

- status：completed/failed/interrupted/error
- summary：子代理最终输出文本
- api_calls、duration_seconds：运行成本线索
- exit_reason：completed / max_iterations / interrupted
- tokens：输入/输出 token（如可用）
- tool_trace：最小可审计轨迹（工具名、参数大小、结果大小、是否 error）

见：

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L450-L466)

业务迁移建议（你可以直接照搬并扩展）：

- 加一个 `artifacts` 字段：子代理产出的“可直接消费对象”（文件路径、结构化 JSON、链接、提取的实体列表）。
- 加一个 `confidence` 或 `needs_review`：当子代理检测到不确定性时显式标记。
- 把 `summary` 分成 `summary_markdown` 与 `summary_plain`（便于不同渠道渲染）。

### 4) tool_trace：并行工具调用配对的关键是 tool_call_id

子代理消息里 assistant 的 tool_calls 与 tool role 的结果，通过 `tool_call_id` 配对：

- 先遍历 assistant messages，记录每个 tool_call 的 id → trace entry
- 再遍历 tool messages，用 tool_call_id 回填 result_bytes 与 status

见：

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L399-L436)

这让父代理在不回收完整 tool outputs 的情况下，仍能知道“子代理到底做了哪些外部动作，以及是否有失败”。

## 动手练习

1) 为你的业务设计子代理输出契约（JSON schema 草案）

- 设计至少这些字段：
  - status（enum）
  - summary（string）
  - artifacts（array/object）
  - tool_trace（array）
  - errors（array，可选）
- 写出每个字段的意义，以及父代理如何使用它。

2) 设计 2 条父代理合并策略

- 场景 A：两个子代理都返回 completed，但 summary 冲突
- 场景 B：一个子代理 max_iterations，另一个 completed
- 说明父代理应如何：
  - 决定重试/降级/拆分任务
  - 把多份 summary 合并成最终对用户的交付物

## 验收清单

- [ ] 能解释“只回传总结”对成本/安全/可靠性的收益
- [ ] 能解释子代理系统提示词如何约束输出结构
- [ ] 能读懂 delegate_task 的结果 dict，并能指出你业务要扩展的字段
- [ ] 能解释 tool_trace 如何用 tool_call_id 解决并行配对问题

## 下一步建议（从学习到落地）

- 选一个你的业务场景，把它拆成 2–3 个可委派子任务
- 给每个子任务定义 toolsets 最小集合与输出契约
- 用父代理做“总控合并 + 审计 + 失败重试策略”

