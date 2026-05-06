# Module 04 / Lesson 02：dispatch_async_coercion

## 学习目标

- 能解释 Hermes 的统一工具调度链路：LLM 输出 → handle_function_call → registry.dispatch → handler。
- 能理解并复刻“sync 主循环调用 async 工具”的稳定桥接策略（避免 event loop 关闭问题）。
- 能解释并应用“参数类型纠正”（string → int/bool/float）来提高工具调用成功率。

## 先修知识

- 已完成 Module 04 / Lesson 01（理解 registry 与 toolsets）
- 了解 Python 的 asyncio 基本概念（事件循环、协程）

## 本节要回答的关键问题

- 为什么不能在每次工具调用都用 `asyncio.run()`？
- Hermes 如何在 CLI、gateway、以及线程池子线程中分别安全执行 async 工具？
- 为什么需要参数类型纠正？纠正的边界是什么（什么能纠正，什么不能）？
- 为什么 delegate_task 需要处理 `_last_resolved_tool_names` 这种全局状态？

## 核心概念

- 调度器（Dispatcher）必须是“稳定的同步入口”：主循环是同步的，但工具可能是 async。
- 工具调用输入是“不可信的”：LLM 常把数字/布尔输出成字符串，需要 schema 驱动的纠正。
- 并发/子代理会引入“全局共享状态污染”，必须显式保存/恢复。

## 代码走读路线

1. sync→async 桥接：持久 event loop（主线程/子线程分别维护）
   - [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L35-L126)
2. registry.dispatch：async 工具通过 `_run_async` 执行
   - [registry.py](file:///d:/编程学习记录/hermes-agent/tools/registry.py#L149-L167)
3. 参数类型纠正：coerce_tool_args / _coerce_value
   - [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L372-L457)
4. 统一调度：handle_function_call（含 pre/post tool hook 与 execute_code 特判）
   - [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L459-L548)
5. 子代理对全局 `_last_resolved_tool_names` 的保存/恢复
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L580-L608)

## 关键代码讲解

### 1) 为什么不用 `asyncio.run()`：Event loop is closed 的真实坑

`_run_async()` 的注释点名了问题：如果每次都 `asyncio.run()`，会创建并关闭一个新 loop，但缓存的 async client（例如 httpx/AsyncOpenAI）仍绑定到旧 loop，后续 GC 或复用时就会触发 “Event loop is closed”。

Hermes 的策略是：

- 主线程：维护一个长期存在的 loop
- 子线程：为每个 worker thread 维护 thread-local 的长期 loop
- 已处于 async 上下文（gateway/RL env）：用一个一次性线程包住 `asyncio.run()` 避免冲突

见：

- [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L44-L125)

这是你做业务 Agent 时非常值得直接复用的“运行时稳定性补丁”。

### 2) registry.dispatch：统一入口 + 保证错误格式

registry.dispatch 做两件重要的事：

- async handler 走 `_run_async(...)`
- 捕获所有异常，并返回 `{"error": ...}` 的 JSON 字符串

见：

- [registry.py](file:///d:/编程学习记录/hermes-agent/tools/registry.py#L149-L167)

这让主循环可以把“工具失败”当作普通 tool result 回填给模型，而不是让异常直接打断整条链路。

### 3) 参数类型纠正：schema 驱动的“弱类型修复”

LLM 常见错误：`"42"`（字符串）本应是 integer；`"true"` 本应是 boolean。

Hermes 在 `handle_function_call` 开头调用 `coerce_tool_args`，根据 registry 里登记的 JSON Schema 尝试纠正：

- [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L372-L457)

边界很重要：

- 只对“字符串但 schema 期望 number/integer/boolean”的情况做纠正
- 纠正失败就保持原值（宁可让工具报可读错误，也不要隐式改错）

你做业务工具时，建议 schema 写得更严格一些（required、enum、pattern），让模型更容易学会正确传参。

### 4) handle_function_call：调度层的“粘合剂”

`handle_function_call` 除了 dispatch 之外，还做了几件粘合工作：

- file read/search 的“连续读取计数”重置（避免 read 工具被过度使用）
- 插件 pre_tool_call/post_tool_call hook（可用于审计/埋点/策略控制）
- execute_code 特判：需要知道 sandbox 允许哪些工具（来自 `enabled_tools` 或 `_last_resolved_tool_names`）

见：

- [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L459-L548)

你做业务 Agent 时，也可以把“审计/权限/限流/幂等”放在这一层，而不要散落在每个工具 handler 里。

### 5) 子代理与全局状态：`_last_resolved_tool_names` 的污染与恢复

`model_tools._last_resolved_tool_names` 是进程级全局变量，用于让某些工具（例如 execute_code）知道当前会话可用工具集合。

但子代理构造时会调用 `get_tool_definitions()`，从而覆盖这个全局，导致父代理的工具集信息被污染。

delegate_task 因此在构造子代理前保存、构造后恢复：

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L580-L608)

这条经验非常实用：只要你引入了“并发/子上下文”，就要系统性排查进程级全局变量，必要时做保存/恢复或改成显式参数传递。

## 动手练习

1) 设计你的工具调度层（dispatcher）职责清单

- 输出 8 条以内的清单，包含：
  - 参数校验/纠正
  - 权限检查
  - 审计日志
  - async 桥接策略
  - 统一错误格式

2) 设计一个“最小 schema 驱动 coercion”规则

- 给你的 3 个业务工具写 schema（只写参数 properties + type），并说明：
  - 哪些字段允许被 coercion（string→int/bool/float）
  - 哪些字段禁止 coercion（例如 ID 字符串、正则格式）

## 验收清单

- [ ] 能解释为什么 `asyncio.run()` 会导致 loop 关闭问题
- [ ] 能描述 Hermes 的三路 async 执行策略（主线程/子线程/已在 async 上下文）
- [ ] 能解释 schema 驱动 coercion 的动机与边界
- [ ] 能解释子代理为何要保存/恢复 `_last_resolved_tool_names`

## 下一课预告

下一模块进入“状态与持久化”：SessionDB（SQLite+FTS5）如何支撑续聊、检索与 system prompt 快照，以及为什么一些状态型工具必须由 agent loop 拦截。

- SessionDB：[hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py#L115-L214)
- agent-level 工具拦截：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6194-L6267)

