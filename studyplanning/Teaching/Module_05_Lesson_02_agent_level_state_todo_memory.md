# Module 05 / Lesson 02：agent_level_state_todo_memory

## 学习目标

- 能解释为什么 todo/memory/session_search/clarify/delegate_task 这类工具必须由 agent loop 拦截执行，而不是走通用 registry.dispatch。
- 能理解 gateway 模式“每条消息新建 AIAgent”带来的状态问题，以及 Hermes 如何从 history 恢复 todo 状态。
- 能把这些经验迁移到你的业务 Agent：哪些状态必须放在 runtime、如何持久化、如何在压缩/切分会话时迁移。

## 先修知识

- 已完成 Module 02（理解 tool loop）
- 已完成 Module 05 / Lesson 01（理解 SessionDB 的角色）

## 本节要回答的关键问题

- 什么叫“Agent-level state”？为什么不能放在全局工具层？
- todo 工具的状态在哪里？为什么需要从 history hydrate？
- memory 写入为什么要桥接 external memory provider？
- 压缩时为什么要 flush memories？为什么还要把 todo snapshot 注入回 messages？

## 核心概念

- 状态归属（State Ownership）：谁拥有状态，谁负责持久化，谁负责迁移。
- 纯工具 vs runtime 工具：
  - 纯工具：stateless、可并行、可重试、可在 registry 层统一调度
  - runtime 工具：依赖 Agent 内部状态/平台回调/父子关系，必须在 agent loop 内处理

## 代码走读路线

1. TodoStore 的初始化（每个 agent/session 一份）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L996-L999)
2. gateway 模式下从 history 恢复 todo 状态（hydrate）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7144-L7149)
3. runtime 工具拦截执行：`_invoke_tool()` 的分支
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6194-L6260)
4. 压缩前 flush memories + 压缩后注入 todo snapshot
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6075-L6090)

## 关键代码讲解

### 1) 为什么 runtime 工具必须由 agent loop 拦截

在并行执行路径里，Hermes 用 `_invoke_tool()` 统一调用工具，并在这里显式拦截 runtime 工具：

- todo：写入 `self._todo_store`
- session_search：依赖 `self._session_db`
- memory：写入 `self._memory_store`，并可桥接 `self._memory_manager`
- clarify：需要 `self.clarify_callback`（平台层提供）
- delegate_task：需要 `parent_agent=self`（父子关系、interrupt 传播）

见：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6194-L6260)

如果把这些工具做成“普通 registry 工具”，你会遇到三个问题：

- 工具 handler 必须拿到大量 runtime 对象（db、callback、store），导致接口污染与测试困难。
- 安全边界变模糊（例如子代理能否写共享 memory？是否允许发消息？）
- 运行时迁移更难（压缩/切分会话时，状态如何跟着走？）

### 2) gateway 模式下 todo 为什么要 hydrate

Hermes 在 run_conversation 里有一段注释解释得很直接：gateway 每条消息会创建一个新的 AIAgent，内存里的 todo store 是空的，所以要从 conversation history 里找最近一次 todo tool 的响应，把 items replay 回 store。

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7144-L7149)

这是一种“把状态编入对话轨迹”的策略：

- 只要 todo 工具的 tool result 进了 history，新的 agent 实例就能恢复状态。
- 同时避免把 todo 状态强依赖某个进程内存（否则 gateway 重启就丢）。

你做业务 Agent 时也可以用同样思路处理轻量状态（计划、进度、临时上下文），尤其在“无粘性 worker/多实例”的部署方式下非常好用。

### 3) 压缩时为什么要 flush memories + 注入 todo snapshot

压缩会丢弃中间对话内容，如果长期价值信息没落盘到 memory，就会永远丢失。因此 `_compress_context()` 先做 memory flush：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6075-L6076)

压缩后还会把 todo 快照以 user message 形式注入回 messages：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6087-L6090)

这体现了一个通用规律：

- 压缩不仅是“减少 token”，更是一次“状态迁移事件”。
- 你必须决定哪些状态要在压缩后仍对模型可见（todo/关键约束/工作上下文）。

### 4) memory 写入为何要桥接 external memory provider

Hermes 支持外部 memory provider（插件），但仍保留内置 MEMORY.md/USER.md 的写入路径。为了让外部 provider 感知内置 memory 变更，memory 工具写入后会触发桥接回调：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6230-L6239)

这是一种“多后端一致性”的设计：同一个用户意图（写记忆）可以同时落到不同存储介质中。

## 动手练习

1) 给你的业务 Agent 列出 6 个“runtime 状态对象”

- 例如：权限上下文、租户配置、请求追踪 ID、任务计划、短期缓存、会话存储连接。
- 对每个状态对象写出：
  - 生命周期（每 turn / 每 session / 全局）
  - 持久化位置（history/DB/外部存储/不持久化）
  - 在压缩/会话切分时如何迁移

2) 设计一个“状态型工具”并决定它应不应该被拦截

- 任选一个业务功能（例如“设置当前订单”“切换工作项目”“写入用户偏好”）。
- 说明它属于：
  - 普通工具（registry 工具）还是 runtime 工具（agent loop 拦截）
- 给出理由（状态依赖/安全边界/迁移成本）。

## 验收清单

- [ ] 能解释 runtime 工具拦截的必要性与判断标准
- [ ] 能解释 gateway 模式下 todo hydrate 的动机与机制
- [ ] 能解释压缩为何是“状态迁移事件”，并举出至少 2 个需要迁移的状态
- [ ] 能解释 external memory provider 桥接的意义

## 下一课预告

下一模块进入“子代理委派”：如何限制子代理工具权限、隔离上下文、隔离 task_id，并把子代理结果设计成父代理可组合的结构化产物。

- blocked tools：[delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L28-L35)
- 子代理系统提示词模板：[delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L48-L80)

