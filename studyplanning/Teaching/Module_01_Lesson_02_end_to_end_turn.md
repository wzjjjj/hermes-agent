# Module 01 / Lesson 02：end_to_end_turn

## 学习目标

- 能把一次用户输入（一个 turn）拆成稳定的阶段，并指出每个阶段的输入/输出数据结构。
- 能在代码里定位：system prompt 缓存、上下文预检压缩、工具调用执行（串行/并行）、以及“哪些工具必须由主循环拦截”。
- 能写出你自己的业务 Agent 的最小“turn loop”伪代码（包含工具回填与终止条件）。

## 先修知识

- 已完成 Lesson 01（知道主要模块在哪里）
- 知道 OpenAI 风格 messages：`role=user/assistant/tool`，以及 `tool_call_id`

## 本节要回答的关键问题

- “一次 turn”的边界在哪里？哪些状态跨 turn 保留，哪些必须重置？
- system prompt 为何要缓存？续聊为什么要从 DB 复用快照？
- 工具调用是如何从模型输出被解析出来，并如何回填成 tool messages 的？
- 并行工具调用在什么条件下发生？为什么不是所有工具都并行？
- todo/memory/session_search/delegate_task 为什么不走统一 registry.dispatch？

## 核心概念

- Turn = “用户消息 + 可能多轮工具迭代 + 最终自然语言回答”
- 稳定系统提示词（system prompt）与可变回合输入（user message/tool result）分离
- Tool execution boundary：工具是副作用边界，必须能审计、能限权、能压缩

## 代码走读路线

1. 工具集在 Agent 初始化阶段生成（tool schemas 是 turn loop 的一部分输入）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L893-L999)
2. turn 入口：`run_conversation()` 的开头阶段（清理、恢复、组装消息、system prompt 缓存）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7041-L7233)
3. 预检压缩：在进入主循环前做 token 粗估并可能多次压缩
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7234-L7291)
4. 工具调用执行：串行/并行两条路径，以及 agent-level 工具的拦截执行
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6194-L6267)
   - 并行执行与回填：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6268-L6473)
   - 串行执行与分支：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6492-L6799)

## 关键代码讲解

### 1) 初始化阶段：工具 schemas 只生成一次（每个 Agent 实例）

在 `AIAgent.__init__()` 中，runtime 会调用 `get_tool_definitions()` 生成 `self.tools` 与 `self.valid_tool_names`：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L893-L914)

这里有两个你设计业务 Agent 时非常值得照搬的点：

- toolset 过滤是“实例级配置”：不同会话/平台可以拿到不同能力集。
- `valid_tool_names` 是“运行时真相”：后续任何“提到其他工具”的描述，都应该以它为准来动态修补，避免模型幻觉调用不存在的工具。

### 2) turn 入口：重置什么，保留什么

`run_conversation()` 一开始做的事情可以分成三类：

1) “本 turn 必须重置”的计数器（否则前一 turn 的失败会污染下一 turn）

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7093-L7104)

2) “跨 turn 必须保持一致”的东西：system prompt 快照

- 它在首 turn 构建并缓存；续聊时从 DB 读取旧快照，避免前缀缓存失效：
  - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7182-L7233)

3) “从历史恢复运行时状态”的桥接：todo store hydrate

- gateway 模式每条消息可能新建 AIAgent，因此要从 history 恢复 todo：
  - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7144-L7149)

### 3) 预检压缩：在 API 调用前先排雷

关键点不是“压缩”本身，而是“压缩发生在主循环之前”：在进入工具迭代前就判断是否会超上下文窗口，尤其是在切换到更小上下文模型时。

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7234-L7291)

这里有一个很实用的工程选择：粗估 token 时会把工具 schemas 一并计入（工具多时 schemas 占用会非常夸张），因此预检更靠谱。

### 4) 工具执行：为什么要有“agent-level 工具拦截”

在并行执行路径里，有一个 `_invoke_tool()` 用来统一调用工具。它先检查是否属于“必须由主循环拦截”的工具（todo/memory/session_search/clarify/delegate_task），否则才走通用 `handle_function_call()`：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6194-L6267)

为什么这些工具不能当作普通工具？

- todo：需要读写本实例的 TodoStore（运行时状态）。
- memory：需要写入 MemoryStore，并可能桥接外部 memory provider。
- session_search：依赖 SessionDB（运行时注入的 db 连接）。
- clarify：需要平台回调（CLI/网关）才能“问用户”。
- delegate_task：需要 parent_agent 上下文，且要注册子代理用于 interrupt 传播。

如果你在业务里要引入“需要 runtime 状态”的工具（比如订单上下文缓存、用户权限对象、事务上下文），非常建议也采用同样的分层：

- 普通工具：纯函数式 + registry dispatch
- runtime 工具：由主循环显式拦截，拥有状态与副作用权限

### 5) 并行 vs 串行：不是越并行越好

并行执行路径会把工具调用丢到线程池，并保持“按原 tool_call 顺序回填 messages”（否则模型看到的 tool results 顺序会乱）：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6268-L6473)

串行执行路径则更适合：

- 交互工具（clarify）或依赖顺序的工具链
- 需要更细粒度展示/控制的执行

你在业务里可以把“是否可并行”变成一个策略函数：基于工具类型、依赖关系、以及风险级别决定。

## 动手练习

1) 写出你的“turn loop”伪代码

- 只写 25 行以内，必须包含：
  - messages 组装（含 user）
  - system prompt 构建/缓存策略
  - while 工具迭代（检测 tool_calls → 执行 → append tool messages）
  - 终止条件（无 tool_calls → 返回最终回答）

2) 找出“并行工具调用”决策点，并解释你会如何扩展它

- 从 [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6180-L6190) 出发，追踪：
  - 哪些工具可能阻止并行？
  - 你会不会为你的业务增加一个“可并行性声明”（例如 `tool_meta.parallelizable=true`）？放在哪里最合适？

## 验收清单

- [ ] 能口述 turn 的阶段划分（初始化/组装/预检/循环/收尾）
- [ ] 能解释 system prompt 快照复用与“缓存不被破坏”的动机
- [ ] 能解释 runtime 工具拦截的必要性与边界
- [ ] 能解释并行执行的收益与风险（顺序回填、交互/依赖不并行）
- [ ] 你的伪代码包含完整的工具迭代闭环

## 下一课预告

下一模块开始深入 `AIAgent.run_conversation()`：先把 system prompt 缓存与 messages 组装讲透，尤其是“续聊为什么要复用 DB 里的 system_prompt 快照”这一点。

- 入口：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7182-L7233)

