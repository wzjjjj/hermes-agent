# Module 02 / Lesson 01：system_prompt_and_messages

## 学习目标

- 能明确区分并解释：system prompt、ephemeral_system_prompt、user message、tool results 各自的职责边界。
- 能从代码中复刻一套“稳定 system prompt + 动态回合上下文”的设计（缓存友好、不破坏一致性）。
- 能解释为什么续聊要复用 DB 里存的 system prompt 快照，而不是每回合重建。

## 先修知识

- 了解 messages 数组的基本结构（role/user/assistant/tool）
- 了解 prefix cache 的基本思想（相同前缀 → 成本下降/速度提升）

## 本节要回答的关键问题

- system prompt 由哪些层组成？为什么要分层？
- `ephemeral_system_prompt` 为什么刻意不写入缓存/持久化？
- 续聊时复用旧 system prompt 快照解决了什么问题？会牺牲什么？
- 为什么要“清理历史里的 budget warnings”？这和模型行为有什么关系？

## 核心概念

- 稳定契约 vs 动态上下文
  - 稳定契约：system prompt（尽量跨 turn 不变）
  - 动态上下文：user message 本回合追加、tool results、插件注入（每 turn 变化）
- 快照一致性（Snapshot Consistency）
  - 续聊时复用旧快照，保证模型看到的系统提示词与上一回合一致，从而不破坏缓存，也不引入“模型自己写的记忆”再被重复加载导致指令漂移。

## 代码走读路线

1. system prompt 分层构建（含关键注释：为何只在 session 内构建一次）
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L2677-L2836)
2. run_conversation 中的 system prompt 缓存/DB 快照复用逻辑
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7182-L7233)
3. 进入 turn 前的历史清理：移除上一 turn 注入的 budget warnings
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7136-L7143)

## 关键代码讲解

### 1) system prompt 的“层”是什么，为什么要分层

`_build_system_prompt()` 的注释把层次写得很清楚：身份、用户/平台系统提示词、记忆、技能、上下文文件、时间戳、平台提示等。

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L2685-L2693)

你应该把它理解成一种“运行时系统约束的组合器”：每一层都对应一种不同来源、不同信任级别、不同更新频率的信息。

一个可迁移的业务设计原则：

- 高信任、低频变化 → system prompt
- 低信任、或高频变化 → user message / tool results

### 2) 为什么 `ephemeral_system_prompt` 不进入 system prompt

代码里明确写了：ephemeral_system_prompt 不在 `_build_system_prompt()` 里拼接，而是在 API 调用时注入，这样它不会进入缓存/持久化快照。

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L2755-L2759)

这适合放什么？

- 只对当前一次调用有意义、但不应该污染会话语义的提示
- 例如：一次性“请你用某种风格回答”、或临时的安全/格式约束

你做业务 Agent 时也会遇到这种需求：一次调用要加约束，但不想让它“永久生效”。这个模式可以直接复用。

### 3) 续聊为什么要复用 DB 里的 system prompt 快照

`run_conversation()` 在构建 system prompt 时优先走“复用 stored_prompt”的路径：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7182-L7233)

它解决两个核心问题：

- 缓存命中：system prompt 变化会破坏前缀缓存，导致成本上升。
- 一致性：如果你每 turn 重新加载磁盘上的 memory/context 文件，模型会看到“它自己刚写入的内容”又被重新塞进 system prompt，指令语义可能重复叠加或漂移。

代价与权衡：

- 复用快照意味着：在不触发“上下文压缩事件”的情况下，磁盘上的记忆变化不会自动体现在 system prompt 里。
- Hermes 通过“只在压缩后重建 system prompt”把这个代价限制在少数关键节点上（避免频繁破坏缓存）。

### 4) 为什么要从历史里剥离 budget warnings

Hermes 会把迭代预算压力注入 tool result（而不是单独插入消息）以减少结构破坏。但如果把这些注入内容原样带入下一 turn，某些模型会误以为它仍然是“当前指令”，从而降低/停止工具调用。

因此在 turn 开头会清理历史：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7136-L7143)

这条经验非常通用：任何“仅对当前 turn 有效”的信号，都需要在 replay history 时做清理，否则就会变成隐形的长期指令。

## 动手练习

1) 写一个你的业务版 `_build_system_prompt()` 分层清单

- 输出一个 8–12 行的列表，每一行是一个层：
  - 层名
  - 来源（配置/数据库/外部系统/用户输入）
  - 更新频率（每会话/每 turn/按事件）
  - 信任级别（高/中/低）

2) 设计你的“ephemeral 提示”场景

- 给出 2 个你业务里会出现的“一次性约束”，并说明：
  - 为什么不能进长期 system prompt
  - 放在 user message 还是放在 API-call-time 注入更合适

## 验收清单

- [ ] 能列出 system prompt 的主要层，并解释每层存在的理由
- [ ] 能解释 ephemeral_system_prompt 的定位与典型用途
- [ ] 能解释续聊复用 system prompt 快照与缓存命中的关系
- [ ] 能解释为何要清理历史中的一次性信号（budget warnings）

## 下一课预告

下一课进入工具迭代的“主战场”：工具调用是如何从模型输出被解析、执行、再回填进 messages 的，以及为什么 Hermes 需要串行/并行两条执行路径。

- 入口：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6180-L6473)

