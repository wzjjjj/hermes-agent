# Hermes-Agent Agent 设计学习计划（Plan）

## Summary

目标：基于 hermes-agent 代码库，按“核心 Agent 回路”主线分模块教学，最终让你具备独立设计并适配业务的 Agent 能力。交付物为一套可持续迭代的 Markdown 课程材料（大纲 + 逐课 + 索引），每课包含动手练习与验收清单。

## Current State Analysis（基于仓库实况）

### 关键入口与模块

- Agent 主循环与运行时编排： [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py)
  - `AIAgent` 负责对话回路、系统提示词缓存、上下文压缩预检、工具调用迭代、插件 hook 等。
- 工具发现/筛选/分发（tool registry + toolsets）： [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py) + [tools/registry.py](file:///d:/编程学习记录/hermes-agent/tools/registry.py) + [toolsets.py](file:///d:/编程学习记录/hermes-agent/toolsets.py)
  - 工具模块在 import 时自注册；`get_tool_definitions()` 根据 toolset 过滤并做动态 schema 修补；`handle_function_call()` 统一调度。
- 系统 Prompt 构建与上下文文件注入： [agent/prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py)
  - 支持 SOUL.md / AGENTS.md / .hermes.md 等上下文文件加载、注入前扫描与截断。
- 上下文压缩： [agent/context_compressor.py](file:///d:/编程学习记录/hermes-agent/agent/context_compressor.py)
  - 以“先裁剪旧工具输出、再总结中间回合、保护头尾”为核心策略，避免超上下文窗口。
- 子代理/委派： [tools/delegate_tool.py](file:///d:/编程学习记录/hermes-agent/tools/delegate_tool.py)
  - 子代理隔离上下文、隔离 task_id、限制 toolset，父代理仅接收总结。
- Session 持久化 + FTS 搜索： [hermes_state.py](file:///d:/编程学习记录/hermes-agent/hermes_state.py)
  - SQLite + FTS5，支撑跨会话检索、系统提示词快照复用等能力。

### 已确认的学习偏好（来自你的选择）

- 主线：核心 Agent 回路优先
- 文档结构：大纲 + 逐课（每课单独 md）+ 索引 README
- 每课包含：动手练习 + 验收清单
- 总目标：形成可迁移的 Agent 设计能力（不仅能读懂，还能复刻/改造）

## Proposed Changes（将要产出的学习材料与组织方式）

### 输出目录（新建）

在仓库内新增课程材料目录（若不存在则创建）：

- `studyplanning/Teaching/README.md`：课程索引入口（模块/课表/学习建议）
- `studyplanning/Teaching/Course_Syllabus.md`：课程大纲（只列结构与每课目标）
- `studyplanning/Teaching/Module_XX_Lesson_YY_<slug>.md`：逐课文档（正文）

### 课程模块划分（以核心回路为主线，必要时引入支撑模块）

#### Module 01：全局架构与关键数据流

- Lesson 01：项目结构与“Agent 运行时”边界（CLI/网关/工具/插件的分工）
- Lesson 02：一次用户输入从入口到输出的全链路地图（包含工具调用回路）

#### Module 02：对话回路（AIAgent.run_conversation）

- Lesson 01：消息组装与系统提示词缓存策略（为什么不能每回合重建 system prompt）
- Lesson 02：工具调用迭代与停止条件（max_iterations / budget pressure / 失败恢复）
- Lesson 03：上下文压缩的触发点与“预检压缩”（避免小上下文模型切换导致崩溃）

#### Module 03：Prompt 构建与上下文文件注入

- Lesson 01：SOUL.md / AGENTS.md / .hermes.md 的发现优先级、截断与注入防护
- Lesson 02：把“可变上下文”放进 user message 而不是 system prompt 的设计收益

#### Module 04：工具系统（可扩展性的核心）

- Lesson 01：工具自注册与 toolset 过滤（registry.register → model_tools.get_tool_definitions）
- Lesson 02：工具调度、async bridging 与参数类型纠正（coerce_tool_args / _run_async）

#### Module 05：上下文与状态（让 Agent 可长期工作）

- Lesson 01：SessionDB（SQLite+FTS5）在“续聊、检索、提示词快照”中的作用
- Lesson 02：Todo/Memory 等“需要 Agent-level 状态”的工具为何被 agent loop 拦截

#### Module 06：子代理（Delegation）与并行化

- Lesson 01：delegate_task 的隔离边界（上下文、工具权限、task_id、递归深度）
- Lesson 02：如何把“子代理总结”设计成你业务可用的可组合产物（模板与约束）

### 每节课文档模板（强制结构）

每个 `Module_XX_Lesson_YY_*.md` 统一包含：

- 学习目标（可验证）
- 先修知识
- 本节要回答的关键问题（3–6 个）
- 核心概念（强调设计动机与权衡）
- 代码走读路线（文件链接 + 关键函数/类名，尽量带行号范围）
- 关键代码讲解（聚焦数据结构与控制流，而非堆代码）
- 动手练习（至少 1 个，可包含“破坏性实验”）
- 验收清单（Markdown checklist）
- 下一课预告（承接阅读入口）

## Assumptions & Decisions

- 课程材料写在仓库内（便于和代码一起版本化），默认路径为 `studyplanning/Teaching/`。
- 教学以“设计能力迁移”为目标：每节课输出一组“可复用的设计原则/接口契约/扩展点清单”，并在练习中要求做一次小改造或推演。
- 代码引用使用本机绝对路径 `file:///d:/编程学习记录/hermes-agent/...`，并尽量补齐行号范围；若行号后续变化，允许在更新课程时修正。

## Verification（计划执行后的验收方式）

- 目录与文件存在：`studyplanning/Teaching/README.md`、`Course_Syllabus.md`、以及所有 Lesson 文件创建完成。
- README 索引完整：每课链接可点击，模块/课编号一致。
- 每课包含模板中的全部小节（尤其是练习与验收清单）。
- 每课至少包含 3 处代码引用链接（含关键函数/类），并且路径指向当前仓库文件。

## Execution Steps（用户确认后立即执行）

1. 创建 `studyplanning/Teaching/` 目录与 `README.md`、`Course_Syllabus.md`。
2. 逐模块产出 Lesson 文档：
   - 先写 Module 01 / Module 02（核心回路）
   - 再写 Prompt 构建、工具系统、状态、子代理
3. 为每节课补齐代码引用链接（尽量带行号），并在 README 更新索引。
4. 快速自检：打开索引，抽查 2–3 个链接与结构一致性。

