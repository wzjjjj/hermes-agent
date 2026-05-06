# Module 06 / Lesson 01：delegation_isolation

## 学习目标

- 能解释 Hermes 的子代理（delegate_task）如何实现“隔离上下文、隔离权限、隔离执行环境（task_id）”。
- 能指出子代理为什么必须禁用一部分工具，并理解这些限制如何提升安全性与可控性。
- 能把同样的隔离思想迁移到你的业务 Agent：哪些任务应该委派给子代理，如何设置最小权限集合。

## 先修知识

- 已完成 Module 02（tool loop 与 runtime 工具拦截）
- 已完成 Module 04（toolset 与工具调度）

## 本节要回答的关键问题

- 子代理拿到的上下文是什么？为什么是“新会话（no parent history）”？
- 子代理的工具集如何确定？如何保证“子代理不获得父代理没有的能力”？
- 哪些工具被强制禁用？为什么？
- 子代理如何继承/覆盖模型与凭据？为什么要支持 override provider/model？

## 核心概念

- 最小权限（Least Privilege）：默认不给，按需给；尤其是会产生副作用的工具。
- 上下文隔离：子代理不共享父代理 history，避免把父代理上下文噪声带入子任务，也避免子任务污染父代理思考。
- 运行时隔离：子代理有独立的迭代预算与执行资源（例如终端会话/工具缓存）。

## 代码走读路线

1. blocked tools：子代理永远不可用的工具集合
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L28-L35)
2. 子代理系统提示词模板：强调只输出总结、不猜路径
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L48-L80)
3. 子代理构建：toolset 继承/交集、skip_context_files/skip_memory、enabled_toolsets 限权
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L196-L336)
4. 委派深度限制（防递归爆炸）
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L532-L540)

## 关键代码讲解

### 1) 禁用工具列表：为什么要禁用这些

子代理强制禁用的工具包括：

- delegate_task：防递归委派导致指数级爆炸
- clarify：子代理不能直接问用户（交互必须由父代理统一管理）
- memory：防止子代理写共享 MEMORY.md 导致不可审计的污染
- send_message：防跨平台副作用（乱发消息）
- execute_code：鼓励子代理逐步推理，不让它用脚本绕过约束

见：

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L28-L35)

你做业务 Agent 时，强烈建议也写一份“子代理禁用清单”，尤其是：

- 写数据库/发通知/支付/删除等不可逆操作
- 任何会改变全局配置/权限的操作

### 2) toolset 的交集：保证子代理不越权

`_build_child_agent()` 的关键约束是：当用户指定 toolsets 时，仍然要与父代理的有效 toolsets 做交集，避免子代理“获得父代理禁用的能力”。

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L224-L249)

同时，它还会剔除被 blocked 的 toolset（delegation/clarify/memory/code_execution）：

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L108-L113)

### 3) 子代理的“会话隔离”与“状态隔离”

构建子代理时，Hermes 做了几条关键隔离选择：

- `quiet_mode=True`：子代理不走交互 UI
- `skip_context_files=True`：不注入上下文文件，避免把父代理工作区规则带进子任务
- `skip_memory=True`：不加载/写入共享记忆
- `clarify_callback=None`：彻底禁止子代理交互
- `enabled_toolsets=child_toolsets`：强制限权

见：

- [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L287-L316)

### 4) 为什么支持 override provider/model

委派允许子代理使用不同 provider/model（例如子任务用更快更便宜的模型），同时父代理保持高质量模型做总控与合并。

这一点对业务非常实用：你可以把“便宜模型做检索/抽取/整理，高质量模型做决策/合并”作为默认架构。

## 动手练习

1) 为你的业务 Agent 设计一份“子代理禁用清单”

- 列出至少 8 条禁用项（工具/操作类别），并为每条写一句理由。

2) 设计 3 类适合委派的子任务

- 每类写清楚：
  - 目标（goal）
  - 允许 toolsets（最小集合）
  - 产出形式（父代理如何使用它）

## 验收清单

- [ ] 能解释为什么子代理要禁用 clarify/memory/send_message 等工具
- [ ] 能解释子代理 toolsets 必须与父代理做交集（防越权）
- [ ] 能指出子代理构建时的关键隔离参数（skip_context_files/skip_memory/quiet/clarify_callback）
- [ ] 能说明 override provider/model 的业务价值

## 下一课预告

下一课讲“子代理输出契约”：delegate_task 如何把子代理结果包装成结构化 JSON（summary、exit_reason、tool_trace、tokens），以及你如何为业务定义可组合的子代理产物。

- 入口：[delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L384-L467)

