# Module 03 / Lesson 01：context_files_and_safety

## 学习目标

- 能复刻一套“项目上下文文件注入 system prompt”的机制：发现优先级、长度上限、注入前扫描。
- 能解释为什么 Hermes 把上下文文件当作“不可信输入”，并用规则阻断潜在 prompt injection。
- 能在你的业务 Agent 中定义“可注入上下文”的来源清单与安全策略。

## 先修知识

- 已完成 Module 02（理解 system prompt 的定位与稳定性）
- 了解基本的 prompt injection 风险（例如：要求忽略规则、要求泄露 secrets）

## 本节要回答的关键问题

- Hermes 会从哪些文件加载上下文？优先级是什么？为什么只选一种项目上下文来源？
- 为什么要截断上下文文件？截断后的用户体验如何补偿？
- Hermes 如何扫描上下文文件中的潜在注入？扫描规则是怎样写进代码的？
- gateway 模式为什么要用 `TERMINAL_CWD` 来决定上下文文件发现路径？

## 核心概念

- 上下文文件是“人类写给 Agent 的契约”，但来源可能不可信（仓库内容、第三方文档、复制粘贴）。
- 注入策略必须同时满足：
  - 可用性：能提供足够的项目背景
  - 安全性：不让“文件中的恶意指令”覆盖系统规则
  - 成本控制：有严格长度上限

## 代码走读路线

1. 上下文文件注入前扫描（威胁模式 + 不可见字符）
   - [prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L31-L73)
2. 项目上下文文件发现优先级（只加载一种 project context）
   - [prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L951-L990)
3. 截断策略（head/tail + marker）
   - [prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L826-L835)
4. system prompt 构建时如何注入 context_files_prompt，并为何使用 `TERMINAL_CWD`
   - [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L2798-L2807)

## 关键代码讲解

### 1) 注入前扫描：把上下文文件当作“不可信输入”

Hermes 在把上下文文件塞进 system prompt 之前，会先扫描：

- 不可见 unicode（常用于隐藏指令）
- 一组注入威胁正则（例如“ignore previous instructions”“do not tell the user”等）

见：

- [prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L31-L73)

如果命中威胁，Hermes 会阻断该文件内容，返回一个 `[BLOCKED: ...]` 占位文本。

你做业务 Agent 时，至少应该覆盖三类规则：

- 让模型忽略规则/绕过限制的指令
- 让模型隐藏信息/欺骗用户的指令
- 引导读取/外传 secrets 的指令（例如 .env、token、credentials）

### 2) 发现优先级：为什么只加载“一种”项目上下文来源

Hermes 的 project context 发现是“first match wins”：

1) `.hermes.md / HERMES.md`（向上走到 git root）
2) `AGENTS.md`（cwd）
3) `CLAUDE.md`（cwd）
4) `.cursorrules / .cursor/rules/*.mdc`（cwd）

见：

- [prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L951-L979)

这个设计的收益是：

- 避免把多套规则叠加造成冲突（system prompt 里规则越多越难控）。
- 给用户一个明确的“生效文件优先级”，便于调试。

### 3) 截断策略：限制成本，同时保留可解释性

Hermes 用 head/tail 截断，并在中间插入 marker，告诉模型“内容被截断，需要用文件工具读取全文”：

- [prompt_builder.py](file:///d:/编程学习记录/hermes-agent/agent/prompt_builder.py#L826-L835)

这是一个很好的“成本与可用性折中”：

- system prompt 里只放足够引导的信息
- 真正需要全文时，让模型用 read/search 工具去读

### 4) gateway 模式的 `TERMINAL_CWD`：避免误注入 dev 仓库的规则

在 `_build_system_prompt()` 注入 context_files_prompt 时，会优先用 `TERMINAL_CWD` 来做上下文文件发现路径：

- [run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L2798-L2807)

原因是 gateway 进程的工作目录可能是安装目录/开发仓库目录，如果直接 `os.getcwd()` 会把不该注入的开发文件（例如本仓库的 AGENTS.md）塞进提示词，白白浪费 token，甚至污染对话行为。

这条经验可迁移到任何“多工作区/多平台”的 Agent：

- “上下文文件发现路径”必须来自用户的实际工作目录，而不是服务进程的 cwd。

## 动手练习

1) 定义你的业务 Agent 的“上下文文件协议”

- 写出你希望用户能提供哪些文件（2–4 种），例如：
  - `BUSINESS.md`（业务规则）
  - `RUNBOOK.md`（操作手册）
  - `POLICY.md`（安全边界）
- 为它们定义优先级与长度上限，并说明冲突时怎么处理（first wins / merge / explicit include list）。

2) 设计你的上下文扫描规则

- 输出 6 条规则（可以是正则或语义描述），覆盖：
  - 忽略规则/越权
  - 隐藏信息/欺骗
  - secrets 外传

## 验收清单

- [ ] 能解释上下文文件注入前扫描做了什么
- [ ] 能说清 project context 的发现优先级与“只选一种来源”的理由
- [ ] 能解释截断策略为何是 head/tail + marker
- [ ] 能解释 gateway 为什么用 `TERMINAL_CWD` 作为上下文发现路径

## 下一课预告

下一课讲一个更“缓存友好”的关键点：Hermes 把动态上下文（插件注入、外部检索上下文）放进 user message，而不是 system prompt，从而保持 system prompt 的稳定前缀。

- 入口：[run_agent.py](file:///d:/编程学习记录/hermes-agent/run_agent.py#L7292-L7474)

