# Module 04 / Lesson 01：tool_registry_and_toolsets

## 学习目标

- 能复刻 Hermes 的“声明式工具系统”：工具自注册、toolset 分组、运行时按 toolset 过滤生成 schemas。
- 能解释为什么 Hermes 需要 toolset（能力集配置）而不是直接“全部工具都给模型”。
- 能把这一套迁移到你的业务 Agent：给不同用户/场景提供不同能力集，并保证模型不会幻觉调用不存在的工具。

## 先修知识

- 已完成 Module 02（理解 tool loop 的闭环）
- 了解 Python import 的副作用（模块 import 会执行顶层代码）

## 本节要回答的关键问题

- 工具是如何“自注册”的？注册表里到底存了什么？
- 为什么工具发现是 import 一个列表，而不是扫描目录？
- toolset 是如何被解析（includes 递归）并与插件工具集融合的？
- 为什么 `get_tool_definitions()` 还要做动态 schema 修补？

## 核心概念

- 工具声明（Declaration）与工具执行（Execution）分离：声明发生在 import 时，执行发生在 runtime dispatch。
- toolset = 能力集配置单元：决定“模型看得到哪些工具 schema”，从源头降低越权与幻觉调用。
- 可用性过滤（check_fn）：工具 schema 只有在依赖满足时才会暴露给模型。

## 代码走读路线

1. 工具注册表：注册与 schema 获取
   - [registry.py](file:///d:/编程学习记录/hermes-agent/tools/registry.py#L24-L143)
2. 工具发现：import 列表触发各工具模块的 registry.register
   - [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L132-L170)
3. toolset 定义与解析（includes、all/*、插件 toolset 融合）
   - [toolsets.py](file:///d:/编程学习记录/hermes-agent/toolsets.py#L29-L378)
   - [toolsets.py](file:///d:/编程学习记录/hermes-agent/toolsets.py#L398-L452)
   - [toolsets.py](file:///d:/编程学习记录/hermes-agent/toolsets.py#L477-L552)
4. 运行时选择工具 schema：enabled/disabled toolsets + check_fn 过滤 + 动态 schema 修补
   - [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L234-L353)

## 关键代码讲解

### 1) registry.register：工具声明的“唯一入口”

每个工具模块在 import 时调用 `registry.register(...)` 声明自己：

- name：工具名（模型调用用）
- toolset：所属 toolset（能力分组）
- schema：OpenAI function schema
- handler：执行函数
- check_fn：可用性检查（依赖/环境变量/订阅能力等）

见：

- [registry.py](file:///d:/编程学习记录/hermes-agent/tools/registry.py#L59-L94)

你做业务 Agent 时，这个注册表建议也保留：

- 统一的错误格式（JSON string）
- 每工具的最大输出预算（避免单次 tool result 爆炸）
- 每工具的安全等级/副作用等级（用于审批与并行策略）

### 2) 工具发现：为什么用“显式 import 列表”

Hermes 通过 `_discover_tools()` import 一组工具模块，触发它们在 import 时自注册：

- [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L132-L170)

显式列表的好处：

- 可控：哪些工具属于核心发行版一目了然。
- 可容错：import 某个可选工具失败，不会阻断整个工具系统（catch 异常继续）。
- 可审计：工具能力的入口点不会被“目录里多了个文件”悄悄改变。

如果你偏向自动扫描目录，也建议保留 allowlist 或签名校验，否则容易引入安全/可预测性问题。

### 3) toolset：把“能力集”变成配置项

toolsets.py 定义了大量 toolset，其中一些面向“类别”（web/file/terminal），一些面向“场景”（debugging/safe），还有面向平台的 hermes-cli、hermes-telegram 等。

示例：

- hermes-cli toolset 直接引用 `_HERMES_CORE_TOOLS`：
  - [toolsets.py](file:///d:/编程学习记录/hermes-agent/toolsets.py#L278-L282)
- `resolve_toolset()` 支持 includes 递归与 all/* 别名：
  - [toolsets.py](file:///d:/编程学习记录/hermes-agent/toolsets.py#L398-L432)

插件 toolset 融合也在 toolsets.py 内处理：如果 registry 里出现了一个 toolset 名称不在静态 TOOLSETS 中，就把它当作“插件 toolset”加入可选列表：

- [toolsets.py](file:///d:/编程学习记录/hermes-agent/toolsets.py#L477-L516)

### 4) get_tool_definitions：过滤 + 可用性检查 + 动态 schema 修补

`get_tool_definitions()` 做三件很关键的事：

1) 基于 enabled/disabled toolsets 计算 `tools_to_include`
2) 让 registry 用 check_fn 过滤掉不可用工具（例如缺 API key）
3) 基于“实际可用工具集合”做动态 schema 修补，避免 schema 描述中提到不存在的工具引发幻觉调用

见：

- [model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L234-L353)

你做业务 Agent 时，强烈建议也做第 3 点：任何跨工具引用都应当由“运行时真实可用工具集”驱动，而不是静态写死在描述里。

## 动手练习

1) 为你的业务 Agent 设计 3 个 toolset

- 例如：
  - `read_only`：只读查询（搜索/读取/检索）
  - `ops_safe`：可执行但需要审批（有限 terminal、写文件）
  - `full_power`：全量能力（仅管理员）
- 写清楚每个 toolset 包含哪些工具，并说明“为什么要分开”。

2) 给每个工具写一个 check_fn 规则（纸上即可）

- 选 5 个工具，分别写出它们的可用性条件：
  - env var 是否存在
  - 外部服务连通性
  - 用户权限
  - 是否在 gateway 模式

## 验收清单

- [ ] 能解释工具自注册如何工作（import-time side effect）
- [ ] 能解释 toolset 的价值（能力集配置、降低越权/幻觉）
- [ ] 能解释 check_fn 过滤的意义（schema 层面就隐藏不可用工具）
- [ ] 能解释为什么要做动态 schema 修补（避免描述引用不存在工具）

## 下一课预告

下一课深入“工具调度”的工程细节：同步主循环如何安全调用异步工具（async bridging），以及如何把模型不稳定的参数类型纠正为 schema 声明的类型。

- 入口：[model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L35-L126)
- 参数纠正：[model_tools.py](file:///d:/编程学习记录/hermes-agent/model_tools.py#L372-L457)

