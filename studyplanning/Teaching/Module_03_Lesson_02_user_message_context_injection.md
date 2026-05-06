# Module 03 / Lesson 02：user_message_context_injection

## 学习目标

- 能解释“把动态上下文注入 user message，而不是 system prompt”的收益与代价。
- 能从代码中定位并复刻：插件 hook 生成上下文 → API-call-time 注入当前 turn user message → 不污染持久化历史。
- 能为你的业务 Agent 设计一套“动态上下文注入协议”（来源、格式、围栏、作用域、是否持久化）。

## 先修知识

- 已完成 Module 03 / Lesson 01（理解 system prompt 注入与安全）
- 了解“缓存前缀稳定性”对成本/延迟的影响

## 本节要回答的关键问题

- pre_llm_call hook 为什么只在 turn 开头调用一次？返回的 context 如何合并？
- 插件/外部检索上下文如何保证“不进入 session DB、不污染历史 replay”？
- `ephemeral_system_prompt` 与 “user message 注入” 有何区别？什么时候用哪个？
- Anthropic 的 cache_control 断点为什么放在这里应用？

## 核心概念

- API-call-time 注入（Transient Injection）
  - 注入只影响“这一次 API 请求”，不修改内存里的 `messages`，也不写入 DB。
- system prompt 的领土（System Prompt Territory）
  - system prompt 保持稳定，以便缓存命中；动态信息走 user message/tool results。

## 代码走读路线

1. 插件 hook：pre_llm_call 收集上下文（字符串或 dict{context}）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7292-L7326)
2. 动态注入：只在准备 API messages 时改写当前 turn 的 user message（不改原 messages）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7408-L7434)
3. 构建最终 system message：cached system prompt + ephemeral_system_prompt（仍是 API-call-time）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7462-L7475)
4. 应用 Anthropic prefix cache 控制点（cache_control breakpoints）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7483-L7489)

## 关键代码讲解

### 1) pre_llm_call：把“动态上下文生产”变成可插拔 hook

Hermes 在进入 while 工具迭代前调用一次 `pre_llm_call` hook，并把所有返回的 context 合并成一段字符串：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7292-L7326)

这个设计适合你做业务扩展：

- 你可以写多个插件，分别提供不同来源的上下文（权限、偏好、业务知识库、告警状态）。
- hook 返回结构允许 dict 或 string，便于逐步演进到结构化数据。

### 2) API-call-time 注入：只改 api_messages，不改 messages

Hermes 在构建 `api_messages` 时，对当前 turn 的 user message 做“临时追加”，并且明确写了：原 `messages` 不会被 mutate，因此不会泄漏到 session persistence。

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7417-L7434)

注入源包括：

- 外部 memory provider 的一次性 prefetch（`_ext_prefetch_cache`）
- 插件 hook 的 `_plugin_user_context`

并且外部 recall 会被包装成 fenced block（避免混淆用户原始输入）：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7424-L7427)

可迁移的业务约束建议：

- 给注入内容加围栏（fence）与来源标识
- 给每段注入加“作用域声明”（仅本 turn / 仅本工具迭代 / 可持久化）
- 对注入总长度设置上限（避免反向把上下文挤爆）

### 3) ephemeral_system_prompt：也是 API-call-time，但属于“system 层一次性补丁”

Hermes 构建最终 system message 时，会把 cached system prompt 与 `ephemeral_system_prompt` 拼接后作为 system role message prepend 到 `api_messages`：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7462-L7475)

选择 user message 注入 vs ephemeral system prompt 的经验法则：

- 影响“如何回答/如何推理/格式约束”的一次性规则 → ephemeral system prompt
- 影响“本回合要用到的事实/上下文信息”的动态内容 → user message 注入

### 4) 为什么强调“不改 system prompt”：缓存命中与职责分离

Hermes 反复强调插件上下文不改 system prompt，因为 system prompt 变化会破坏缓存前缀：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7469-L7473)

同时在启用 Anthropic prompt caching 时，会在 API messages 上加 cache_control 断点：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7483-L7489)

你可以把它理解成：“为了让缓存机制工作，必须把稳定/不稳定的内容隔离开”。

## 动手练习

1) 为你的业务 Agent 设计一个“动态注入协议”

- 输出一段结构化约定（可以用 YAML/伪 JSON 描述）：
  - sources：哪些模块能注入
  - format：每段注入的围栏/标题/来源字段
  - limits：单段与总注入长度上限
  - persistence：是否写入历史（默认不写）

2) 选择一个你业务里的动态上下文来源，并写出注入样例

- 例如：用户权限、当前订单状态、feature flag、运行时告警。
- 写出“用户原始输入 + 注入块”拼接后的最终 user message 样例（用 fenced block）。

## 验收清单

- [ ] 能解释 pre_llm_call hook 的输入输出与调用时机
- [ ] 能解释为什么注入只改 api_messages 而不改 messages（避免持久化污染）
- [ ] 能区分 user message 注入 与 ephemeral system prompt 的适用场景
- [ ] 能解释 cache_control 的断点策略与 system prompt 稳定性的关系

## 下一课预告

下一模块进入工具系统：工具如何自注册、如何按 toolset 过滤生成 schemas、以及为何要做动态 schema 修补来避免“描述引用不存在的工具”。

- 入口：[model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L132-L353)
- 注册表：[registry.py](file:///d:/编程学习记录/hermes-agent/tools/registry.py#L59-L167)

