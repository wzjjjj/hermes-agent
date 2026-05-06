# Module 01 / Lesson 01：architecture_overview

## 学习目标

- 能用“运行时边界”的方式描述 hermes-agent：哪些属于 Agent runtime，哪些属于 UI/平台适配，哪些属于可插拔能力。
- 能画出核心依赖方向：prompt 构建、工具系统、会话存储、子代理委派如何被主循环串起来。
- 能从代码中定位每个模块的“权力中心”（谁拥有状态、谁只做纯函数、谁负责副作用）。

## 先修知识

- Python 基础（类/函数/模块导入）
- 了解 LLM 工具调用的基本形态：tool schemas + tool_calls + tool results
- 了解“会话持久化”的基本概念（SQLite/日志即可，不要求熟悉 SQL）

## 本节要回答的关键问题

- Hermes 的“Agent”到底是哪个对象？核心循环在哪里？
- 工具为什么不是写死在一个大文件里？如何做到“自注册 + toolset 过滤 + 统一调度”？
- 为什么要把 system prompt 缓存/快照？这跟成本/一致性有什么关系？
- 上下文压缩解决的到底是什么问题？为什么先裁剪 tool results，再总结中间回合？
- 子代理（delegation）如何做到“隔离上下文、隔离权限、只回传总结”？

## 核心概念

- 运行时边界（Runtime Boundary）
  - hermes-agent 把“对话回路（LLM 调用 + 工具迭代 + 状态维护）”作为 runtime 核心；CLI/gateway 只负责输入输出与回调。
- 声明式工具系统（Declarative Tooling）
  - 工具通过模块 import 时自注册到 registry，runtime 再按 toolset 过滤生成 schemas，并在运行中统一 dispatch。
- 缓存友好的提示词策略（Cache-friendly Prompting）
  - system prompt 作为“稳定契约”尽量不变；动态上下文倾向放在 user message（或工具结果）里，避免破坏前缀缓存。
- 分层上下文管理（Context Management）
  - 先做无模型成本的裁剪（旧 tool output），再做有模型成本的总结（中间回合），同时保护对齐最关键的头尾信息。
- 子代理隔离（Subagent Isolation）
  - 子代理是“新会话 + 限权工具集 + 独立 task_id + 深度限制”，父代理只接收总结，不接收过程。

## 代码走读路线

1. 总览入口：AIAgent 定义与主循环所在文件
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L439-L740)
2. 工具系统的三层结构：发现/过滤/调度
   - 工具发现与 schema 生成：[model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L132-L353)
   - 中央注册表与 dispatch：[registry.py](file:///d:/编程学习记录/hermes-agent/tools/registry.py#L24-L167)
3. system prompt 里的“上下文文件注入”机制
   - [prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L822-L990)
4. 上下文压缩策略（算法说明与关键参数）
   - [context_compressor.py](file:///d:/编程学习记录/hermes-agent/agent/context_compressor.py#L53-L112)
5. 子代理委派的隔离边界（blocked tools、depth、summary）
   - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L1-L80)
6. 会话持久化与 FTS 搜索（支撑“续聊/检索/快照”）
   - [hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py#L115-L214)

## 关键代码讲解

### 1) “核心 Agent”是谁：AIAgent 作为运行时中枢

- AIAgent 不是“UI”，而是运行时编排器：它同时持有工具集、上下文压缩器、system prompt 缓存、迭代预算、回调接口等运行时状态。
- 在 gateway 场景下，每条消息可能创建一个新的 AIAgent；因此有些状态要么落 DB，要么设计成可从 history 恢复（例如 todo store 的 hydrate）。
  - 可以从 [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L447-L451) 看到有“跨实例去重”的设计动机（网关模式频繁创建实例）。

### 2) 工具系统为何要“registry + toolsets”

- 工具不是写死在 run_agent.py 里，而是由每个工具模块在 import 时调用 `registry.register(...)` 声明自己：
  - [registry.py](file:///d:/编程学习记录/hermes-agent/tools/registry.py#L59-L94)
- `model_tools._discover_tools()` 负责把工具模块 import 进来触发注册：
  - [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L132-L170)
- runtime 再根据 enabled/disabled toolsets 过滤并生成最终 schemas：
  - [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L234-L353)
- 设计收益（你未来做业务 Agent 时可直接复用）：
  - 新增工具不需要修改“核心循环”，降低耦合。
  - toolset 让“能力集”可配置（不同平台/不同用户/不同安全级别启用不同工具组合）。
  - schema 可在运行时做动态修补，避免“描述里提到不存在的工具”导致模型幻觉调用。

### 3) Prompt 构建：上下文文件为何要“优先级 + 截断 + 注入前扫描”

- hermes-agent 支持把项目上下文文件注入系统提示词，但明确设置了优先级（只选一种项目上下文来源）并做截断：
  - [prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L951-L990)
- 这背后对应两条工程原则：
  - 防 prompt injection：上下文文件来自用户/项目，不可信，必须做扫描/保护。
  - 控制输入成本：上下文文件永远有上限，避免把上下文窗口耗尽在静态文本上。

### 4) 上下文压缩：先裁剪 tool output，再总结 middle turns

- ContextCompressor 的算法说明非常直白：先做“无 LLM 成本”的裁剪，再做“有 LLM 成本”的总结，并保护头尾：
  - [context_compressor.py](file:///d:/编程学习记录/hermes-agent/agent/context_compressor.py#L53-L62)
- 你在设计业务 Agent 时，可以把这当作一套通用策略：
  - 低成本手段（裁剪/丢弃/占位符）优先。
  - 高成本手段（总结）要结构化、可迭代更新，并有失败冷却。

### 5) 子代理委派：隔离 + 限权 + 只回传总结

- delegate_tool 明确规定子代理禁止的工具集合（不能递归委派、不能 ask user、不能写共享 memory、不能跨平台副作用）：
  - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L28-L35)
- 子代理的系统提示词也强调“只输出总结”，父代理不接收过程：
  - [delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py#L48-L80)
- 这是一种“把并行能力变成可控产物”的方式：你在业务里可以把子代理当成“任务执行器”，而父代理当成“总编排器 + 审计者”。

## 动手练习

1) 画一张你的版本的“Hermes Runtime 架构图”

- 要求：至少包含 6 个盒子：AIAgent、PromptBuilder、ToolRegistry、Toolsets/Tool filtering、ContextCompressor、SessionDB、Delegate(Subagent)。
- 在每条箭头上标注“数据/控制流”：例如 schemas 从哪来、tool results 回到哪里、system prompt 什么时候写入 DB。

2) 用代码定位“每个模块的权力中心”

- 任务：分别找出以下对象/函数“谁持有状态，谁只做纯处理”：
  - AIAgent（状态中心）
  - ToolRegistry（全局注册表）
  - ContextCompressor（策略对象，持有阈值/预算）
  - SessionDB（持久化）
- 输出：写 10 行以内的总结，回答“如果我想换掉其中一个模块，最小替换接口是什么？”

## 验收清单

- [ ] 能指出主循环入口在 `run_agent.py`，并能口述一次 turn 的关键阶段
- [ ] 能解释工具系统的三层：注册、过滤生成 schema、统一 dispatch
- [ ] 能解释 system prompt 为什么要尽量稳定（与缓存/一致性相关）
- [ ] 能解释上下文压缩为何“先裁剪 tool output，再总结 middle turns”
- [ ] 能解释 delegate_task 的隔离边界（为何要禁用某些工具）

## 下一课预告

下一课把“抽象概念”落到一条具体链路：从 `run_conversation()` 开始，走完一次用户输入如何触发 system prompt 缓存、上下文预检压缩、插件 hook、以及进入工具调用迭代。

- 入口文件：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7041)

