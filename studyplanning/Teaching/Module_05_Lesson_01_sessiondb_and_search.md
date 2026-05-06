# Module 05 / Lesson 01：sessiondb_and_search

## 学习目标

- 能理解 Hermes 为什么要有 SessionDB：它解决“续聊一致性、跨会话检索、提示词快照复用”的哪些问题。
- 能读懂 SessionDB 的并发写入策略（WAL + jitter retry）与 FTS5 查询安全化。
- 能把同样的“会话存储 + 检索”能力迁移到你的业务 Agent（哪怕不用 SQLite）。

## 先修知识

- 已完成 Module 02/03（理解续聊时 system prompt 快照复用的意义）
- 了解 SQLite 是单文件数据库、FTS5 是全文检索扩展（不要求写 SQL）

## 本节要回答的关键问题

- 为什么 gateway/CLI 并发写入会导致卡顿？Hermes 如何缓解？
- 为什么要存 system_prompt 快照？它和 prompt cache 有什么关系？
- FTS5 的 MATCH 查询为什么需要 sanitize？Hermes 的 sanitize 策略是什么？

## 核心概念

- 续聊一致性：同一个 session 的 system prompt 必须稳定；否则历史 replay 会变形。
- 可检索性：把消息内容做 FTS5 索引，能在后续 turn 以“工具”形式取回上下文。
- 并发友好：多进程/多线程写入同一个 state.db 时，必须做锁竞争治理。

## 代码走读路线

1. SessionDB：WAL + 写入重试抖动（避免 convoy）
   - [hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py#L115-L214)
2. FTS5 查询 sanitize（避免 MATCH 语法错误、保留短语语义）
   - [hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py#L1003-L1055)
3. search_messages：FTS5 snippet + 上下文窗口（前后各一条）
   - [hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py#L1056-L1157)
4. AIAgent 使用 SessionDB：续聊复用 system_prompt 快照
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7193-L7230)

## 关键代码讲解

### 1) 写入竞争：为什么不用 SQLite 默认 busy handler

SessionDB 的注释非常明确：多进程共享一个 state.db 时，WAL 写锁竞争会导致 UI 卡顿；SQLite 默认的 deterministic backoff 会产生 convoy 效应。

Hermes 的策略是：

- timeout 设短（1s）
- 在应用层做随机抖动重试（20–150ms）
- 周期性 PASSIVE checkpoint（不阻塞）

见：

- [hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py#L123-L214)

这条经验对“任何单点写入资源”（数据库、队列、文件锁）都适用：当默认退避策略造成同步拥塞时，引入 jitter 往往是简单有效的工程解。

### 2) FTS5 sanitize：为什么要把 dotted/hyphenated 词包进引号

FTS5 的 tokenizer 会把 `chat-send` 拆成 `chat AND send`，把 `P2.2` 拆成 `p2 AND 2`，这对“命令名/文件名/版本号”这种查询很不友好。

Hermes 的 sanitize 做了：

- 保护成对双引号短语
- 移除不安全的特殊字符
- 处理 `*` 前缀匹配的边界
- 清理首尾 dangling boolean operators
- 把未加引号的 dotted/hyphenated terms 包上引号

见：

- [hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py#L1003-L1055)

这对你做业务检索非常有用：只要你允许用户输入自由查询，就必须处理“检索语法注入/语法错误”，否则检索工具会变成不稳定的失败源。

### 3) search_messages：为何返回 snippet 而不是全文

search_messages 使用 FTS5 `snippet(...)` 返回带标记的片段，并额外查询前后各一条消息作为上下文，但最后会移除全文 `content` 字段：

- [hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py#L1106-L1157)

动机是“省 token”：检索工具的返回如果过长，会反过来挤爆上下文窗口，并让模型在噪声中迷失。

### 4) system_prompt 快照：让续聊的 system prompt 不漂移

在 run_conversation 中，如果是续聊（已有 history 且有 session_db），Hermes 会尝试从 DB 取出上次写入的 system prompt 快照并复用：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7193-L7230)

这背后同时服务两件事：

- prompt cache 前缀稳定
- 避免“系统提示词层”因为外部文件变化而在续聊中发生漂移

## 动手练习

1) 为你的业务 Agent 设计一个“检索返回预算”

- 写出你的检索工具应该返回什么字段（推荐：snippet、来源、时间、相邻上下文），并定义：
  - 单条结果最大字符数
  - 最大返回条数
  - 是否需要相邻上下文

2) 设计你的“查询 sanitize”规则

- 如果你使用 FTS/DSL/搜索引擎，写出你如何处理：
  - 引号不成对
  - 布尔操作符
  - dotted/hyphenated 标识符（命令名/文件名）

## 验收清单

- [ ] 能解释写入抖动重试解决的并发卡顿问题
- [ ] 能解释 FTS5 sanitize 的主要步骤与动机
- [ ] 能解释检索返回 snippet 的 token 成本收益
- [ ] 能解释 system_prompt 快照复用为何对续聊至关重要

## 下一课预告

下一课讲“运行时状态工具”：todo/memory/session_search 等为什么必须由 agent loop 拦截，如何在 gateway 模式下从 history 恢复状态，以及压缩前为何要 flush memories。

- todo hydrate：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7144-L7149)
- runtime 工具拦截：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L6194-L6267)

