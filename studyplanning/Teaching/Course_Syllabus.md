# 课程大纲：用 hermes-agent 学会 Agent 设计

## 学习目标（总）

- 画清“对话回路”：system prompt、messages、tool schemas、tool results、终止条件如何协作。
- 设计一套可扩展的工具系统：工具注册、toolset 过滤、统一调度、异步桥接、失败返回格式。
- 让 Agent 可长期运行：会话持久化、检索、上下文压缩、prompt 缓存、子代理隔离与汇总契约。
- 最终产出：你能独立写出一个“可配置 toolset + 可压缩上下文 + 可委派子任务”的业务 Agent 原型，并能解释每个设计选择的权衡。

## Module 01：全局架构与关键数据流

- Lesson 01：项目结构与“Agent 运行时”边界
  - 目标：能说清 run_agent / tool registry / prompt builder / session DB / CLI / gateway 的分工与依赖方向。
- Lesson 02：一次用户输入的全链路地图
  - 目标：能画出从用户输入到模型输出、再到工具调用与回填的时序图（包含缓存与压缩分支）。

## Module 02：对话回路（AIAgent.run_conversation）

- Lesson 01：消息组装与系统提示词缓存策略
  - 目标：理解“为什么不在每回合重建 system prompt”，以及续聊时如何复用快照。
- Lesson 02：工具调用迭代与停止条件
  - 目标：理解迭代预算、失败重试、工具结果注入、以及何时进入最终回答。
- Lesson 03：上下文压缩的触发点与预检压缩
  - 目标：理解 context window 逼近时的多层策略（裁剪/总结/保护头尾）与模型切换风险。

## Module 03：Prompt 构建与上下文文件注入

- Lesson 01：上下文文件发现、截断与注入防护
  - 目标：能复刻一套“上下文文件优先级 + 注入前扫描 + 长度上限”的机制。
- Lesson 02：把可变上下文放进 user message 的收益
  - 目标：理解缓存友好性与职责边界：system prompt 是运行时固定契约，动态上下文走 user turn。

## Module 04：工具系统（可扩展性的核心）

- Lesson 01：工具自注册与 toolset 过滤
  - 目标：能写出一套 registry.register + get_tool_definitions 的最小可用版本，并知道如何做动态 schema 修补。
- Lesson 02：工具调度、async bridging 与参数纠正
  - 目标：理解同步外壳如何稳定调用异步工具、以及如何让 LLM 传参更健壮（coercion）。

## Module 05：上下文与状态（让 Agent 可长期工作）

- Lesson 01：SessionDB（SQLite+FTS5）在续聊与检索中的作用
  - 目标：理解为何要把系统提示词快照写入 DB、以及 FTS 查询安全化的工程细节。
- Lesson 02：Agent-level 状态为何要被“主循环拦截”
  - 目标：理解 todo/memory/session_search/delegate 这类工具为什么不能走普通 registry.dispatch。

## Module 06：子代理（Delegation）与并行化

- Lesson 01：delegate_task 的隔离边界
  - 目标：能设计“子代理权限最小化”的边界：禁止递归、禁止交互、禁止共享状态写入。
- Lesson 02：把子代理输出做成业务可组合产物
  - 目标：能定义子代理 summary 的结构化契约，让父代理能安全合并、复用、审计。

