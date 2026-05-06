# Module 02 / Lesson 02：tool_call_loop

## 学习目标

- 能把“工具调用迭代”讲清楚：解析 tool_calls → 执行 → 回填 tool messages → 下一次 LLM 调用。
- 能解释串行/并行两条路径的差异与选择理由，并能为你的业务设计一套并行策略。
- 能解释 Hermes 的工具失败处理与预算压力注入（budget pressure）为何设计成现在这样。

## 先修知识

- 已完成 Module 02 / Lesson 01（理解 system prompt 与 turn 结构）
- 了解 JSON：工具参数来自 `tool_call.function.arguments` 的 JSON 字符串

## 本节要回答的关键问题

- Hermes 在哪里决定“串行还是并行”执行工具？
- 工具参数 JSON 解码失败会怎样？如何做防御式处理？
- 为什么要在工具执行前做 checkpoint（文件/危险命令）？
- 工具结果如何被追加到 messages？为什么要附带 `tool_call_id`？
- budget warning 为什么注入到 tool result 里？为什么 turn 开头还要清理它？

## 核心概念

- Tool loop 是一个闭环：工具结果本质上是“模型的外部感知”，必须结构稳定、顺序稳定、可被压缩。
- 并行化是策略，不是默认：正确性（依赖/交互）优先于速度。
- 失败要“可恢复 + 可观测”：统一返回 JSON 字符串，错误可记录到日志，必要时给模型可用的错误上下文。

## 代码走读路线

1. 选择串行/并行执行路径的入口
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6180-L6190)
2. agent-level 工具拦截与统一调用口 `_invoke_tool()`
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6194-L6267)
3. 并行执行：线程池并发 + 按原顺序回填 tool messages + 预算控制
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6268-L6491)
4. 串行执行：更细粒度分支（clarify/delegate/quiet spinner）+ 逐个回填
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6492-L6799)
5. 通用 dispatch：参数类型纠正 + registry.dispatch
   - [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L372-L548)

## 关键代码讲解

### 1) 串行/并行的分流点

Hermes 在执行工具前会判断是否适合并行；不适合就走串行路径：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6180-L6190)

你做业务 Agent 时，这个点就是你插入“依赖分析/风险控制”的最佳位置。

### 2) `_invoke_tool()`：为什么要把一部分工具从通用 dispatch 中“抬出来”

`_invoke_tool()` 的结构本质上是一个路由表：先处理 runtime 工具，再落到通用 `handle_function_call()`。

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6194-L6267)

你应该把它当作“状态与副作用的边界层”。当工具需要：

- 访问 Agent 内部状态（todo store、memory store、session db）
- 依赖平台回调（clarify）
- 需要父代理上下文（delegate_task）

就不要让它成为“纯 registry 工具”，否则你会把大量运行时状态泄漏到全局工具层，最后难以测试/难以限权。

### 3) 工具参数解析：永远不要信任模型输出是合法 JSON

无论串行还是并行，Hermes 都会对 `tool_call.function.arguments` 做 `json.loads`，失败就降级为空 dict，并确保最终是 dict：

- 并行路径解析：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6289-L6305)
- 串行路径解析：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6520-L6527)

这对应一个非常重要的工程习惯：把“LLM 输出”视作不可信输入（类似外部 API），必须做类型/结构校验与降级策略。

### 4) 执行前 checkpoint：把“不可逆副作用”变成“可回滚操作”

Hermes 在某些高风险工具执行前做 checkpoint：

- 文件写/patch 前 checkpoint：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6306-L6314)
- destructive terminal command 前 checkpoint：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6316-L6326)

你做业务 Agent 时也应该定义“不可逆副作用点”（例如：支付、删除、发布、通知），在这些点前插入：

- 审批（approval）
- 快照（checkpoint）
- 幂等键（idempotency key）

### 5) 并行执行的关键要求：按原顺序回填 messages

并行执行里真正关键的不是线程池，而是这条规则：

- 结果可以并行算，但 messages 必须按原 tool_call 顺序追加回去。

代码体现为：results 数组按 index 收集结果，回填时按 parsed_calls 顺序 append tool messages：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6358-L6467)

这是为了保证模型下一轮看到的“工具结果序列”与它发出的调用序列一致，避免语义错位。

### 6) budget pressure 注入：为什么放在 tool result 里

并行路径在工具回填后，会把预算压力（caution/warning）注入到最后一个 tool message（优先作为 JSON 字段 `_budget_warning`）：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6474-L6487)

动机可以理解为：

- 不额外插入消息，尽量维持 messages 结构的稳定性。
- 给模型一个“收敛信号”，避免无限工具循环。

与之配套的，是上一课提到的“turn 开头清理 budget warnings”（否则一次性信号变长期指令）。

### 7) 通用 dispatch 的两件防御性工作：coercion + 统一错误返回

`handle_function_call()` 在真正 dispatch 前做参数类型纠正（LLM 常把数字/布尔当成字符串），并在异常时保证返回 JSON 错误：

- [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L372-L548)

这让“工具层失败”不会把主循环直接打崩，而是转化为模型可消费的失败信息（从而触发重试/改参/换路）。

## 动手练习

1) 手工模拟一次工具调用回填

- 假设模型输出一个 tool_call：`{"name":"search_files","arguments":"{\"query\":\"SessionDB\"}"}`（随便举例）
- 你在纸上写出 messages 应该如何变化：
  - assistant 带 tool_calls
  - tool role 带 content + tool_call_id
  - 下一轮 LLM 调用拿到的新 messages

2) 给你的业务 Agent 设计“并行策略表”

- 输出一个 2 列表：
  - 工具名/类别
  - 是否可并行 + 依据（依赖、顺序、共享资源、外部副作用）

## 验收清单

- [ ] 能指出串行/并行分流点，并能解释为什么不能一刀切全并行
- [ ] 能解释参数解析失败的降级策略（以及为什么要强制 dict）
- [ ] 能解释 checkpoint 的位置选择与副作用控制思路
- [ ] 能解释“并行计算 + 顺序回填”的必要性
- [ ] 能解释 budget pressure 的注入方式与其配套清理逻辑

## 下一课预告

下一课聚焦“上下文压缩”进入 turn loop 的方式：为什么要做预检压缩、压缩后为什么会重建 system prompt、以及压缩如何与 tool outputs 的裁剪配合。

- 入口：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7234-L7291)
- 算法对象：[context_compressor.py](file:///d:/编程学习记录/hermes-agent/agent/context_compressor.py#L53-L62)

