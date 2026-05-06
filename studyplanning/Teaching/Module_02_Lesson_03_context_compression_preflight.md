# Module 02 / Lesson 03：context_compression_preflight

## 学习目标

- 能解释“预检压缩（preflight compression）”为什么在进入工具迭代前发生，以及它解决的真实故障模式。
- 能理解 Hermes 的压缩闭环：裁剪旧工具输出 → 总结中间回合 → 必要时切分 session → 重建 system prompt。
- 能把这一套策略迁移到你的业务 Agent：定义阈值、选择保护头尾、决定何时总结、如何处理失败与退化。

## 先修知识

- 已完成 Module 02 / Lesson 01-02
- 了解“上下文窗口”与 token 粗估的意义（不需要精确计数）

## 本节要回答的关键问题

- 为什么要在 API 调用前做 token 粗估？为什么把 tools schemas 也算进去？
- 压缩为什么不是一次完成？为什么允许多次 pass？
- 压缩后为什么要重建 system prompt？这和“快照一致性”是什么关系？
- 为什么压缩时要先 flush memories？为什么还要清理 file read dedup？
- SessionDB 的“压缩切分”在这里扮演什么角色？

## 核心概念

- 防爆炸策略（Fail-safe）：把“必然会 413/超上下文”的情况提前处理，避免 API 直接失败导致 turn 中断。
- 分层降本：低成本裁剪优先，高成本总结其次；总结需要结构化输入与失败冷却。
- 压缩 = 语义折叠 + 状态迁移：不仅压缩 messages，还要处理相关运行时状态（system prompt 快照、todo 注入、dedup cache 重置、DB session lineage）。

## 代码走读路线

1. 预检压缩触发点（进入主循环之前）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7234-L7291)
2. 压缩实现：`_compress_context()` 做了哪些“压缩以外”的工作
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6063-L6170)
3. 压缩策略对象：ContextCompressor 的算法说明与关键参数
   - [context_compressor.py](file:///d:/编程学习记录/hermes-agent/agent/context_compressor.py#L53-L112)
4. 旧工具输出裁剪：为什么先做这一层（无 LLM 成本）
   - [context_compressor.py](file:///d:/编程学习记录/hermes-agent/agent/context_compressor.py#L151-L210)

## 关键代码讲解

### 1) 预检压缩：为什么要在 tool loop 之前做

Hermes 在进入主循环前，会粗估本次请求 token，并把 tools schemas 也计入估算，然后在超过阈值时主动压缩，最多尝试多次 pass：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7234-L7291)

这主要防两类问题：

- 用户切换到更小上下文模型：历史消息在原模型可用，但在新模型直接超限。
- 工具数量很大：schemas 本身吃掉大量 token，而旧估算只看 messages 会误判。

### 2) ContextCompressor：压缩不是“总结一下”这么简单

ContextCompressor 在注释中写了完整算法：

- 先裁剪旧 tool results（便宜）
- 再保护 head
- 再按 token 预算保护 tail
- 再用结构化 LLM prompt 总结 middle turns（贵）
- 后续压缩会迭代更新 summary

见：

- [context_compressor.py](file:///d:/编程学习记录/hermes-agent/agent/context_compressor.py#L53-L62)

这里的可迁移点是：把压缩分成“确定性压缩（裁剪）”与“语义压缩（总结）”，并明确保护头尾。

### 3) `_compress_context()`：压缩动作会触发一系列“状态迁移”

`_compress_context()` 不是只调用 `context_compressor.compress()`，它还做了几件关键的运行时维护：

1) 压缩前 flush memories

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6075-L6076)

动机：压缩会丢弃中间内容，如果模型还没把长期价值信息写入 memory，就可能永久丢失。

2) 压缩后注入 todo 快照（把“运行时状态”迁移回 messages）

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6087-L6090)

动机：todo 是 Agent-level 状态，但压缩后 messages 变短，模型需要重新看到计划上下文。

3) 失效并重建 system prompt（并更新缓存）

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6091-L6094)

这就是“只在压缩事件后重建 system prompt”的实现落点：既保持大多数 turn 的缓存稳定，又在语义结构发生大变化（压缩）时做一次一致性重建。

4) SessionDB 切分：为“压缩后的新会话”创建新的 session_id，并把 system_prompt 快照写入 DB

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6095-L6119)

把压缩当作一次“会话分叉”而不是“原地修改”，让检索与审计变得更清晰（lineage 可追踪）。

5) 清理 file read dedup：压缩后必须允许重新读同一文件

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6155-L6162)

原因：旧 read 内容可能已被总结/裁剪掉，如果还用 dedup stub，会导致模型拿不到真实内容。

### 4) 旧工具输出裁剪：为何是压缩第一层

裁剪旧 tool results 的核心逻辑是把很老且很长的 tool message content 替换成占位符：

- [context_compressor.py](file:///d:/编程学习记录/hermes-agent/agent/context_compressor.py#L155-L210)

这是非常划算的优化：

- tool results 往往是长文本（网页、日志、文件内容），最占 token。
- 很多时候模型不需要“原始全文”，只需要“曾经查过/曾经成功过”的事实。

你做业务 Agent 时，如果工具返回很大（搜索结果、数据库 dump、文档全文），强烈建议做类似的“老结果清空 + 占位符”策略。

## 动手练习

1) 给你的业务 Agent 设定一套“压缩策略配置”

- 写出 6 个参数（可以仿照 Hermes 思路）：
  - context_length（或从模型元数据拿）
  - threshold_percent
  - protect_first_n
  - protect_last_n（或 tail token budget）
  - summary_target_ratio
  - max_summary_tokens（或上限策略）

2) 设计一次“模型切换导致超上下文”的故障演练

- 场景：用户在一个很长的会话中把模型从 200k 切到 32k。
- 任务：描述你的 Agent 在 turn 开头应该如何：
  - 粗估 token
  - 决定压缩 pass 次数
  - 失败时降级策略（例如只裁剪工具输出、不总结）

## 验收清单

- [ ] 能解释预检压缩的动机与典型故障模式
- [ ] 能解释压缩的两层策略：裁剪 vs 总结
- [ ] 能指出压缩会触发 system prompt 重建，并解释“为什么只在此时重建”
- [ ] 能解释压缩过程中涉及的状态迁移（memory flush、todo 注入、DB 切分、dedup 重置）

## 下一课预告

下一模块开始把“system prompt 构建”独立拆开讲：上下文文件注入（SOUL/AGENTS/.hermes.md）的发现优先级、截断策略、以及注入前扫描如何降低 prompt injection 风险。

- 入口：[prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L822-L990)

