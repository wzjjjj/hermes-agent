# Hermes-Agent 教学材料（Agent 设计主线）

这套材料以 hermes-agent 的“核心 Agent 回路”为主线，目标是让你能把其设计思想迁移到自己的业务里：能画清数据流、选对扩展点、设计工具系统与状态管理，并能落地一个可运行的业务 Agent。

## 使用方式

- 建议按模块顺序学习：每节课都给出“代码走读路线”，你跟着跳转即可。
- 每节课末尾都有“动手练习 + 验收清单”：做完再进入下一课。
- 课程里引用代码时使用本机绝对路径 `file:///d:/编程学习记录/hermes-agent/...`；如果你把仓库挪了位置，需要全局替换这些路径。

## 课程大纲

- [Course_Syllabus.md](./Course_Syllabus.md)

## 逐课索引

### Module 01：全局架构与关键数据流

- Lesson 01：[architecture_overview](./Module_01_Lesson_01_architecture_overview.md)
- Lesson 02：[end_to_end_turn](./Module_01_Lesson_02_end_to_end_turn.md)

### Module 02：对话回路（AIAgent.run_conversation）

- Lesson 01：[system_prompt_and_messages](./Module_02_Lesson_01_system_prompt_and_messages.md)
- Lesson 02：[tool_call_loop](./Module_02_Lesson_02_tool_call_loop.md)
- Lesson 03：[context_compression_preflight](./Module_02_Lesson_03_context_compression_preflight.md)

### Module 03：Prompt 构建与上下文文件注入

- Lesson 01：[context_files_and_safety](./Module_03_Lesson_01_context_files_and_safety.md)
- Lesson 02：[user_message_context_injection](./Module_03_Lesson_02_user_message_context_injection.md)

### Module 04：工具系统（可扩展性的核心）

- Lesson 01：[tool_registry_and_toolsets](./Module_04_Lesson_01_tool_registry_and_toolsets.md)
- Lesson 02：[dispatch_async_coercion](./Module_04_Lesson_02_dispatch_async_coercion.md)

### Module 05：上下文与状态（让 Agent 可长期工作）

- Lesson 01：[sessiondb_and_search](./Module_05_Lesson_01_sessiondb_and_search.md)
- Lesson 02：[agent_level_state_todo_memory](./Module_05_Lesson_02_agent_level_state_todo_memory.md)

### Module 06：子代理（Delegation）与并行化

- Lesson 01：[delegation_isolation](./Module_06_Lesson_01_delegation_isolation.md)
- Lesson 02：[subagent_outputs_and_contracts](./Module_06_Lesson_02_subagent_outputs_and_contracts.md)

