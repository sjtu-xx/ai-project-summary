# Hermes Agent 源码介绍

本文以源码介绍文档的方式说明 `hermes-agent` 的核心实现，重点覆盖以下主题：

- `agent loop` 的入口、状态对象和执行顺序
- 上下文的组装、冻结、临时注入与持久化
- 记忆系统的分层结构和数据流
- 上下文压缩的触发条件、算法和会话切分方式
- 动态发现、Hook、MCP 重载与“热加载”的实际边界
- 并发执行模型、后台进程机制、子 Agent 机制与隔离范围

文中提到的文件路径均相对于仓库根目录。

---

## 1. 项目定位与总体结构

Hermes Agent 是一个以 `AIAgent` 为核心的同步式 Agent Runtime。它不是单纯的“模型 + 工具调用”封装，而是将以下能力整合在同一条执行链中：

- 持续会话
- 会话级稳定 system prompt
- 工具发现与权限裁剪
- 工具调用闭环
- 上下文压缩
- SQLite 持久化
- 长期记忆
- 子 Agent 委派
- Gateway / CLI 两种外层运行形态

从职责划分看，主干可以分为六层：

| 层级 | 主要文件 | 作用 |
| --- | --- | --- |
| 对话执行层 | `run_agent.py` | 管理一次完整会话，驱动模型与工具循环 |
| Prompt 组装层 | `agent/prompt_builder.py`、`gateway/session.py` | 构造 system prompt、session context、技能提示 |
| 工具调度层 | `model_tools.py`、`tools/registry.py`、`toolsets.py` | 发现工具、筛选工具、分发工具调用 |
| 记忆与持久化层 | `tools/memory_tool.py`、`agent/memory_manager.py`、`hermes_state.py` | 管理长期记忆、会话存储、全文检索 |
| 压缩与缓存层 | `agent/context_compressor.py`、`agent/prompt_caching.py`、`agent/model_metadata.py` | 控制上下文长度、缓存断点与 token 估算 |
| 外围执行层 | `gateway/run.py`、`tools/delegate_tool.py`、`tools/process_registry.py`、`batch_runner.py` | 负责平台接入、后台进程、并发委派、批量运行 |

项目的一个核心设计原则是：**会话内尽量保持 system prompt 和工具集合稳定**。很多后续实现细节都围绕这条原则展开，例如 memory 的冻结快照、Gateway 继续会话时复用旧 `system_prompt`、技能注入方式、上下文压缩后的 session 切分等。

---

## 2. 核心入口与初始化过程

### 2.1 `AIAgent` 的对外入口

`run_agent.py` 中的 `AIAgent` 提供两个主要入口：

- `chat(message)`：简化接口，只返回最终文本
- `run_conversation(...)`：完整接口，返回最终回复与完整消息历史

实际核心逻辑在 `run_conversation()`。`chat()` 只是薄封装，最终仍然调用 `run_conversation()`。

### 2.2 `AIAgent.__init__()` 的初始化内容

在构造 `AIAgent` 时，会完成以下几类初始化：

1. **模型与运行参数**
   包括 `model`、`provider`、`api_mode`、`max_iterations`、`platform`、`session_id` 等。

2. **工具集合**
   调用 `model_tools.get_tool_definitions()` 获得当前会话可见的工具 schema，并建立：
   - `self.tools`
   - `self.valid_tool_names`

3. **上下文压缩器**
   创建 `ContextCompressor`，读取模型上下文窗口并计算压缩阈值。

4. **记忆组件**
   构造 `MemoryStore` 与 `MemoryManager`，加载内置记忆文件，注册可能存在的外部 memory provider。

5. **持久化组件**
   关联 `SessionDB`，准备 SQLite 会话存储。

6. **运行态状态对象**
   例如 todo store、工具执行回调、重试计数器、缓存字段、背景资源句柄等。

初始化阶段最重要的结果不是“对象建好了”，而是**当前会话的上下文边界被确定了**。尤其是工具集合与 system prompt 的基础层，在会话开始后都不会频繁重建。

---

## 3. 工具发现、工具集解析与工具分发

### 3.1 工具发现发生在 `model_tools.py`

`model_tools._discover_tools()` 会显式导入一组 `tools.*` 模块。每个工具模块在 import 时调用 `registry.register()` 完成注册。这个过程属于**导入时注册**，不是运行时热替换。

导入完成后，工具注册表里保存了：

- 工具名
- 所属 toolset
- schema
- handler
- 可用性检查函数 `check_fn`

### 3.2 `toolsets.py` 的作用

Hermes 不直接通过“全部工具”暴露能力，而是先定义 `toolset`。`toolsets.py` 负责：

- 定义工具集合
- 解析复合 toolset
- 验证工具组名是否合法
- 返回最终要启用的工具名集合

这一步的作用是将“能力选择”与“具体工具注册”分离。调用方只需要声明启用哪些 toolset，不需要显式维护单个工具列表。

### 3.3 `get_tool_definitions()` 的输出不是注册表原样转发

`model_tools.get_tool_definitions()` 在生成最终 schema 时，还会做几件重要的事情：

1. 根据 `enabled_toolsets` / `disabled_toolsets` 求出候选工具集合
2. 调用 `registry.get_definitions()`，只保留 `check_fn` 通过的工具
3. 基于最终可用工具名，动态修正某些工具的 schema 描述

例如：

- `execute_code` 的 schema 会根据当前会话实际可用工具重建，避免 schema 中引用不可用工具
- `browser_navigate` 的描述在缺失 web 工具时会移除交叉引用，防止模型幻觉调用不存在的工具

因此，`get_tool_definitions()` 的职责不是“读注册表”，而是**生成一个会话级一致、无交叉失真的工具视图**。

### 3.4 普通工具与 Agent Loop 工具

Hermes 将工具分成两类：

#### 普通工具

通过 `model_tools.handle_function_call()` 进入 `registry.dispatch()` 执行。调用前后会触发插件 hook：

- `pre_tool_call`
- `post_tool_call`

#### Agent Loop 工具

`model_tools._AGENT_LOOP_TOOLS` 中定义了四个特殊工具：

- `todo`
- `memory`
- `session_search`
- `delegate_task`

这些工具虽然仍然有 schema，但执行时不走普通 registry 分发，而是在 `run_agent.py` 中直接处理。原因是它们依赖 `AIAgent` 内部状态，例如：

- todo store
- memory store
- session db
- child agent 构造逻辑

这种拆分避免了“所有工具都必须是无状态函数”的限制。

---

## 4. `run_conversation()`：一次对话回合的完整执行顺序

`run_conversation()` 是项目中最关键的函数。一次用户消息从输入到结束，主要经历以下阶段。

### 4.1 输入清理与本轮状态重置

函数开始时首先做运行安全准备：

- 安装安全 stdio，避免守护进程或管道断开导致写输出崩溃
- 为当前线程打上 session 上下文，便于日志过滤
- 恢复主 runtime，防止上一轮 fallback 状态污染当前回合
- 清理用户输入中的 surrogate 字符，避免 OpenAI SDK JSON 序列化失败

随后重置本轮状态：

- `effective_task_id = task_id or uuid`
- 清空若干重试计数器
- 创建新的 `IterationBudget(self.max_iterations)`

`effective_task_id` 非常重要。它会参与 terminal / sandbox / 文件去重缓存等资源隔离，因此并发任务与子 Agent 之间不会天然共享同一个任务上下文。

### 4.2 构造 `messages`

`messages` 是本轮会话的真实消息账本。构造规则如下：

1. 若传入 `conversation_history`，先复制历史列表，避免直接修改调用方对象
2. 若当前 agent 的 todo store 为空，但历史存在，会从历史中恢复 todo 状态
3. 将当前 `user_message` 追加为 `{"role": "user", "content": ...}`

到这一步为止，`messages` 代表“真实会话上下文”，还没有注入任何 API 专用临时内容。

### 4.3 处理 system prompt

system prompt 通过 `self._cached_system_prompt` 缓存，处理规则如下：

1. 若缓存为空，优先检查当前 `session_id` 在 `SessionDB` 中是否已有 `system_prompt`
2. 若存在，直接复用，避免继续会话时因 memory、时间等差异导致前缀变化
3. 若不存在，调用 `_build_system_prompt()` 从头构建
4. 新建 system prompt 后写回 SQLite

这里体现出 Hermes 对 prompt cache 的重视。继续会话时，不重新拼接 system prompt，而是优先复用旧快照。

### 4.4 预检压缩

在真正进入模型循环前，Hermes 会做一次上下文长度预估：

- `estimate_request_tokens_rough(messages, system_prompt, tools)`

这里不仅估算消息和 system prompt，还把工具 schema 一起算入。原因是复杂工具集合本身可能带来大量 token 开销。

若估算值已经超过 `context_compressor.threshold_tokens`，则会在进入主循环前先执行 `_compress_context()`。必要时可以连续压缩多次，直到低于阈值或无法继续缩减。

### 4.5 构造 `api_messages`

进入 while loop 后，每轮会基于 `messages` 构造一份 `api_messages`，这份列表是实际发送给模型的视图。它和 `messages` 的区别在于包含了若干临时注入层：

1. `active_system_prompt`
2. `ephemeral_system_prompt`
3. `prefill_messages`
4. 当前轮 user message 上附加的插件上下文
5. 当前轮 user message 上附加的 memory prefetch 内容
6. `cache_control`
7. API 允许的角色与字段清洗

这种分层使 Hermes 可以同时满足两类需求：

- 会话账本保持稳定、可持久化
- 模型每轮仍然获得额外的即时上下文

### 4.6 主循环

主循环可以概括为：

1. 消耗一次 `iteration_budget`
2. 构造并清洗 `api_messages`
3. 调用模型 API
4. 读取 assistant 结果
5. 若包含 `tool_calls`，执行工具并追加 `tool` 消息
6. 若不包含 `tool_calls`，将 assistant 消息视为最终结果并退出

循环终止条件包括：

- 模型给出最终文本
- 达到最大迭代次数
- 发生无法恢复的错误

### 4.7 assistant 返回后如何分支

#### 情况 A：返回 `tool_calls`

这时会：

- 构造 assistant 消息并追加到 `messages`
- 校验工具名是否合法
- 处理 JSON 参数
- 对 `delegate_task` 数量做裁剪
- 对批量工具调用去重
- 执行工具
- 将每个工具结果以 `{"role": "tool", ...}` 追加到 `messages`

#### 情况 B：没有 `tool_calls`

这时将 assistant 回复视为最终结果。框架还会处理一些边缘情况，例如：

- 空回复重试
- 某些模型的特殊返回格式
- think / scratchpad 清理

### 4.8 工具执行的顺序与并发策略

`run_agent.py` 中通过 `_should_parallelize_tool_batch()` 决定工具批次是否可以并发执行。

判断依据包括：

- 是否出现 `_NEVER_PARALLEL_TOOLS`
- 是否为纯只读工具
- 文件工具是否访问重叠路径
- 工具参数是否可以成功解析

满足条件时，走 `_execute_tool_calls_concurrent()`；否则走 `_execute_tool_calls_sequential()`。

这意味着 Hermes 的并发不是“看起来像无副作用就并发”，而是有一套显式保守规则。

### 4.9 回合结束后的收尾

当本轮结束时，框架会做收尾工作，包括：

- flush 会话消息到 SQLite
- 记录 token / cost 统计
- 同步 memory provider
- 关闭运行中资源
- 返回包含 `messages`、`final_response`、`api_calls` 等字段的结果对象

---

## 5. 上下文系统：稳定前缀、临时注入与 API 视图分离

Hermes 的上下文处理方式可以概括为：**稳定内容进入会话级 system prompt，易变内容在 API 调用时临时注入**。

### 5.1 `_build_system_prompt()` 的组成

`AIAgent._build_system_prompt()` 会按固定顺序拼接 system prompt，各层如下：

1. **身份层**
   - 优先读取 `SOUL.md`
   - 若不存在，则使用 `DEFAULT_AGENT_IDENTITY`

2. **工具感知引导**
   - 若当前会话启用了 `memory`，注入 `MEMORY_GUIDANCE`
   - 若启用了 `session_search`，注入 `SESSION_SEARCH_GUIDANCE`
   - 若启用了 skills 工具，注入 `SKILLS_GUIDANCE`

3. **模型行为约束**
   - 根据配置和模型名称决定是否注入 tool-use enforcement
   - 对部分模型注入额外操作指导，例如 Google / OpenAI 家族的执行规则

4. **外部 system_message**
   - 来自 CLI / Gateway 或上层调用

5. **内置长期记忆**
   - `MEMORY.md`
   - `USER.md`

6. **外部 memory provider 的 system block**

7. **skills 相关提示**

8. **上下文文件**
   - `AGENTS.md`
   - `.cursorrules`
   - 其它可识别上下文文件

9. **时间与环境信息**
   - 当前会话开始时间
   - session id
   - model / provider
   - WSL / Termux 等环境提示
   - 平台提示

构建完成后，system prompt 会被缓存为 `_cached_system_prompt`。

### 5.2 为什么 `ephemeral_system_prompt` 不直接拼进 `_build_system_prompt()`

源码里明确注明：`ephemeral_system_prompt` 只在 API 调用时注入，不进入缓存后的 system prompt。原因有两个：

1. 它可能每轮变化，会破坏前缀稳定性
2. 它不适合被持久化到 session DB

因此，Hermes 将其视为**本轮请求的临时层**，而不是会话的固定层。

### 5.3 `messages` 与 `api_messages` 的边界

这两个对象的区别是理解 Hermes 的关键：

| 对象 | 作用 | 是否持久化 | 是否必须稳定 |
| --- | --- | --- | --- |
| `messages` | 真实会话账本 | 是 | 相对稳定 |
| `api_messages` | 本轮模型调用视图 | 否 | 可临时变化 |

一些只应该给模型看的信息，例如：

- memory recall
- 插件在 `pre_llm_call` 返回的上下文
- prefill 示例消息

都只会进入 `api_messages`，不会直接写回 `messages`。

### 5.4 `gateway/session.py` 提供的 session context

Gateway 模式下，模型还会收到一个动态 session context，内容可能包括：

- 来源平台
- chat 类型
- 用户信息
- 群聊或 thread 信息
- 可发送结果的平台与 home channel

这里也有明显的 cache 友好设计。例如在共享 thread 场景中，不把单个 `user_name` 固定写进 system prompt，而是仅说明“这是多用户线程”，具体发送者通过每条 user message 的前缀体现。这样可以避免线程参与者变化时频繁重建 system prompt。

### 5.5 `apply_anthropic_cache_control()`

`agent/prompt_caching.py` 中的 `apply_anthropic_cache_control()` 会在发往 Anthropic 的消息上打 cache 标记。策略为：

- system prompt 作为第一个断点
- 最近 3 条非 system 消息作为其余断点

总共最多 4 个 cache breakpoints，与 Anthropic 的限制保持一致。

这一策略与前文提到的“system prompt 稳定化”是配套关系：如果 system prompt 每轮变化，这里的缓存策略就无法有效命中。

---

## 6. 记忆系统：短期上下文、会话持久化与长期记忆

Hermes 的“记忆”不是单一组件，而是三类不同目的的数据结构。

### 6.1 短期记忆：`messages`

`messages` 保存当前会话中的原始消息序列。特点是：

- 信息最完整
- 保留 tool call / tool result
- 能直接参与下一轮推理
- 随会话增长会变得过长，因此必须配合压缩器使用

### 6.2 会话持久化：`SessionDB`

`hermes_state.py` 中的 `SessionDB` 使用 SQLite 存储会话。核心表包括：

- `sessions`
- `messages`
- `messages_fts`

其中：

- `sessions` 保存 session 级元信息，如 `system_prompt`、`parent_session_id`、token 统计、标题、结束原因等
- `messages` 保存逐条消息
- `messages_fts` 是 FTS5 虚拟表，用于消息全文检索

几个值得注意的实现点：

1. **WAL 模式**
   用于支持多 reader 与单 writer 的常见并发模式。

2. **应用层重试**
   写事务采用 `BEGIN IMMEDIATE`，遇到锁竞争时由应用层进行带随机抖动的重试，而不是完全依赖 SQLite 内置退避。

3. **会话链**
   上下文压缩后不会覆盖旧 session，而是新建一个 session，并将其 `parent_session_id` 指向旧 session。

### 6.3 Gateway 层的 `SessionStore`

`gateway/session.py` 中的 `SessionStore` 不负责底层数据库写入，它更像 session 编排层，负责：

- `session_key -> SessionEntry` 映射
- transcript 读写
- legacy JSONL 与 SQLite 的兼容
- 存储 Gateway 侧元信息，例如 `last_prompt_tokens`

因此：

- `SessionDB` 是底层数据存储
- `SessionStore` 是 Gateway 模式下的会话索引与策略层

### 6.4 长期记忆：`MEMORY.md` 与 `USER.md`

`tools/memory_tool.py` 定义了 `MemoryStore`。它维护两类文件：

- `MEMORY.md`
- `USER.md`

两者的区别在语义上而不是技术上：

- `MEMORY.md` 用于保存环境事实、项目约定、工具注意事项
- `USER.md` 用于保存用户偏好、交流习惯、长期工作上下文

### 6.5 `MemoryStore` 的双视图设计

`MemoryStore` 在实现上区分了两种状态：

1. **实时状态**
   - `memory_entries`
   - `user_entries`
   这些列表会随着 memory 工具调用实时变化，并立即写盘。

2. **system prompt 快照**
   - `_system_prompt_snapshot`
   该快照在 `load_from_disk()` 时生成，作为当前会话 system prompt 注入内容的冻结版本。

这样做的结果是：

- memory 工具调用后，文件与工具返回结果立即反映新内容
- 当前会话的 system prompt 不会立即变化
- 下一次新会话开始时，新的 memory 才会进入 system prompt

这是为了保护 prefix cache 稳定性。

### 6.6 memory 安全检查

`tools/memory_tool.py` 还对 memory 内容做了显式安全扫描：

- Prompt Injection 模式
- 试图读取或外传 secret 的命令模式
- 不可见 Unicode 字符

原因是这些记忆条目最终会注入 system prompt，因此 memory 并不是普通文本存储，而是**高权限上下文输入**。

### 6.7 外部 memory provider：`MemoryManager`

`agent/memory_manager.py` 用于统一编排：

- 内置 provider
- 至多一个外部 provider

它暴露的关键接口包括：

- `build_system_prompt()`
- `prefetch_all()`
- `queue_prefetch_all()`
- `sync_all()`
- `on_pre_compress()`
- `on_memory_write()`
- `on_delegation()`

这里最重要的限制是：**外部 provider 最多只能有一个**。这能避免：

- tool schema 膨胀
- 多个记忆后端产生冲突
- 同一问题被多个 provider 重复注入

### 6.8 轨迹文件不属于会话记忆

`agent/trajectory.py` 中的 `save_trajectory()` 用于将对话转换为训练或采样格式并写入 JSONL。它与 `SessionDB`、`MemoryStore` 属于不同系统，不参与常规会话恢复。

---

## 7. 上下文压缩：触发条件、算法与 session 切分

### 7.1 压缩阈值的来源

`ContextCompressor` 在初始化时会调用 `get_model_context_length()` 获取模型上下文窗口长度，并计算：

- `threshold_tokens`
- `tail_token_budget`
- `max_summary_tokens`

阈值不是固定常量，而是：

- `context_length * threshold_percent`
- 与 `MINIMUM_CONTEXT_LENGTH` 取较大值

这样可以避免在大窗口模型上过早压缩，也避免在小窗口模型上压缩阈值过低。

### 7.2 触发压缩的两个主要时机

#### 预检压缩

在进入主 loop 前进行，依据是 `estimate_request_tokens_rough()`。这种压缩用于处理：

- 长会话继续对话
- 切换到更小上下文窗口的模型
- 工具 schema 本身导致的额外 token 开销

#### 回合中压缩

在工具调用完成后，根据：

- 模型返回的真实 token usage
- 或 fallback 的 rough token 估算

来判断是否应当压缩。

### 7.3 `ContextCompressor.compress()` 的算法步骤

压缩过程不是简单截断，而是分阶段进行：

1. **旧工具输出剪枝**
   - 对较早、较长的 `role=tool` 内容替换为占位符
   - 这是廉价预处理，不调用 LLM

2. **计算头尾边界**
   - 头部保留 `protect_first_n`
   - 尾部通过 token budget 计算，而不是固定条数

3. **抽取中段**
   - 头尾之间的消息作为待压缩片段

4. **生成结构化摘要**
   - 调用辅助模型总结中段
   - 若存在旧摘要，则进行迭代更新，而非完全重写

5. **组装压缩后消息**
   - 头部原样保留
   - 中段替换为摘要
   - 尾部原样保留

6. **修复 tool pair**
   - 清理压缩后可能出现的孤儿 `tool_call` / `tool_result`

### 7.4 为什么摘要前缀写成 handoff reference

`SUMMARY_PREFIX` 明确告诉模型：

- 这是被压缩后的旧上下文摘要
- 仅用于参考
- 不是新的活动指令
- 不要回应摘要中的旧问题
- 只应响应摘要之后出现的最新用户消息

这段前缀的目的不是增加信息量，而是**降低摘要被误执行的风险**。

### 7.5 `_compress_context()` 做的不仅是文本压缩

`AIAgent._compress_context()` 负责将压缩器接入主会话流程。它除了调用 `compress()` 外，还会做这些事情：

- `flush_memories(messages, min_turns=0)`
- 通知 `memory_manager.on_pre_compress(messages)`
- 注入 todo 快照
- `_invalidate_system_prompt()`
- 重建 `_cached_system_prompt`
- 结束旧 SQLite session
- 创建新的 SQLite session
- 设置 `parent_session_id`
- 重置 flush cursor
- 清理文件读取去重缓存

因此，Hermes 的“上下文压缩”在工程上等价于：

**文本压缩 + system prompt 重建 + 新 session 建立 + 状态游标重置**

### 7.6 压缩的限制

源码中对压缩效果的限制处理比较明确：

- 多次压缩会导致信息损耗累积
- 超过一定次数会给出警告
- 若摘要生成失败，会插入静态 fallback 标记，避免上下文被静默删除

这说明压缩被视为必要机制，但并不被视为无损机制。

---

## 8. “热加载”、动态发现、Hook 与技能系统

### 8.1 项目中不存在通用意义上的核心代码热加载

Hermes 没有针对主循环或内置工具提供通用的 Python 代码热替换机制。修改核心代码后通常仍需重启进程。

项目中存在的是以下三种能力：

- 启动时动态发现
- 某些子系统的显式重载
- 事件 Hook 与技能注入

### 8.2 启动时动态发现

动态发现主要体现在三处：

1. `model_tools._discover_tools()`
   - import `tools.*`
   - 触发工具注册

2. `discover_mcp_tools()`
   - 发现 MCP server 提供的工具

3. `hermes_cli.plugins.discover_plugins()`
   - 发现用户或项目插件

这些都属于**启动时扫描 / 导入**，而不是运行中自动重载代码。

### 8.3 MCP 重载是“局部重载”

项目中最接近“热加载”的能力是 MCP 重载：

- Gateway 提供 `/reload-mcp`
- CLI 会监视 `mcp_servers` 配置变化并触发 `_reload_mcp()`

这类重载只针对 MCP 子系统，不等于核心 Agent 代码热替换。

### 8.4 Gateway Hook：事件总线

`gateway/hooks.py` 中的 `HookRegistry` 负责发现并分发事件 hook。hook 目录结构为：

- `HOOK.yaml`
- `handler.py`

支持的事件包括：

- `gateway:startup`
- `session:start`
- `session:end`
- `session:reset`
- `agent:start`
- `agent:step`
- `agent:end`
- `command:*`

特点如下：

- 支持精确事件与通配事件
- 允许同步或异步 handler
- handler 出错时只记录日志，不阻塞主流程

### 8.5 插件 Hook：嵌入主循环生命周期

除了 Gateway Hook，项目还有插件级 hook，分布在 `run_agent.py` 与 `model_tools.py`。已知切入点包括：

- `on_session_start`
- `pre_llm_call`
- `pre_tool_call`
- `post_tool_call`
- session finalize / reset 相关钩子

这一层 Hook 更靠近主执行链，适合做：

- 工具调用审计
- 上下文增强
- 会话起止事件处理

### 8.6 技能系统：不是 Hook，而是消息注入层

`agent/skill_commands.py` 负责扫描 `SKILL.md` 并生成 `/skill-name` 命令。技能内容最终通过消息注入给模型，而不是直接改写 system prompt。

这种实现方式的效果是：

- 技能可以按需加载
- 技能可以附带支持文件
- 技能可被 CLI 与 Gateway 共用
- 技能不会破坏会话级 system prompt 的稳定性

因此，技能系统更接近“可路由的行为模板”，而不是 Hook 或热加载机制。

---

## 9. 并发模型：工具并发、后台进程、批量运行与子 Agent 并发

Hermes 中至少存在四类并发。

### 9.1 工具批次并发

发生在单次 assistant 消息内部。若模型一次返回多个工具调用，且这些调用被判定为安全独立，则可以进入并发执行。

限制条件包括：

- 不能包含 `clarify`
- 只读工具更容易并发
- 文件类工具必须路径不冲突
- 工作者线程数有上限

### 9.2 后台进程并发

`tools/terminal_tool.py` 与 `tools/process_registry.py` 提供 `background=true` 模式。进程被放到 `ProcessRegistry` 管理后，Agent 不需要阻塞等待其完成。

`ProcessRegistry` 维护的信息包括：

- 进程 ID
- task_id
- session_key
- 输出缓冲
- watcher 配置
- 通知队列

这类并发的重点不是多模型协作，而是**让命令生命周期脱离单轮会话**。

### 9.3 Gateway watcher 驱动的异步推进

Gateway 会轮询 `process_registry.completion_queue`。当后台进程：

- 结束
- 命中 watch pattern
- 触发 watcher 事件

系统会把这些事件转成新输入，再触发新一轮 agent 执行。这使 Hermes 具备“事件触发下一轮会话”的能力。

### 9.4 `batch_runner.py` 的多进程批处理

`batch_runner.py` 使用 `multiprocessing.Pool` 并行处理多个 batch。粒度是：

- batch 之间并行
- batch 内 prompt 串行

这类并发主要用于：

- 数据集评测
- 轨迹采样
- 工具使用统计

它与交互式 CLI / Gateway 路径共享 `AIAgent`，但运行目标不同。

### 9.5 子 Agent 并发

`delegate_task` 在批量模式下通过 `ThreadPoolExecutor` 并发运行多个 child agent。这里的并发是：

- 同一进程内多线程
- 每个子 Agent 各自运行自己的 `run_conversation()`

它不是多进程隔离，而是逻辑隔离。

---

## 10. 子 Agent 机制：构造、隔离范围与结果回传

### 10.1 `delegate_task` 的定位

`tools/delegate_tool.py` 的目标是构造一个受限 child agent 来完成子任务，并将结果摘要返回给父 agent。父 agent 不接收 child 的完整中间上下文。

### 10.2 child agent 的构造规则

创建 child agent 时，框架会显式应用以下限制：

- 使用新的 conversation，不继承父级 `messages`
- 用委派任务构造 child system prompt
- `skip_memory=True`
- `skip_context_files=True`
- 子工具集取父工具集与请求工具集的交集
- 屏蔽高风险工具

`DELEGATE_BLOCKED_TOOLS` 中默认屏蔽：

- `delegate_task`
- `clarify`
- `memory`
- `send_message`
- `execute_code`

### 10.3 隔离是如何实现的

#### 任务隔离

child agent 使用独立 `task_id`，这会影响：

- terminal session
- 文件读取去重缓存
- sandbox 资源

#### 上下文隔离

child 不继承父级 `messages`。父级只看到：

- 委派调用本身
- child 最终摘要

child 中间产生的 tool calls、tool results、reasoning 都不会直接写回父级消息链。

#### 预算隔离

child agent 具有独立 `IterationBudget`。父级预算不会被 child 直接消费。

#### 工具集隔离

child 只能看到裁剪后的工具集合，且部分工具无论如何都会被移除。

### 10.4 仍然共享的资源

Hermes 的 child agent 不是 OS 级沙箱，因此仍可能共享：

- 同一 Python 进程
- 进程级模块全局状态
- 同一个 SQLite 存储
- 某些 credential pool

其中一个典型问题是 `model_tools._last_resolved_tool_names`。这是进程级全局变量，child agent 构造时可能修改它。为此，委派路径里显式保存与恢复旧值，减少对子 Agent 以外逻辑的副作用。

### 10.5 为什么父级只接收摘要

这种设计的作用是：

- 避免父级上下文被子任务细节撑爆
- 保持父级仍然拥有高层规划能力
- 控制跨任务信息回流量
- 降低委派结果对父级会话的污染程度

代价是摘要质量必须足够高，否则父级会失去部分执行细节。

---

## 11. 设计特征与实现取舍

### 11.1 设计中心是稳定前缀，而不是即时一致

很多实现都体现了这个取舍：

- session 内 system prompt 尽量不重建
- memory 写盘后不立即改 system prompt
- recall 上下文进入 API 视图而不是账本
- 多用户 thread 不把发送者写死进 system prompt
- 技能通过消息注入，不直接改固定前缀
- 继续会话时复用 SQLite 中存储的 `system_prompt`

这类设计在语义上不是“最实时”，但在工程上换来了更好的 prompt cache 命中率与上下文稳定性。

### 11.2 上下文被明确拆成三层

Hermes 对上下文的拆分非常清楚：

1. **会话固定层**
   `_cached_system_prompt`

2. **会话账本层**
   `messages`

3. **本轮 API 视图层**
   `api_messages`

这种结构使它能够在保持主账本稳定的同时，支持临时注入、技能加载、memory recall 和缓存断点。

### 11.3 压缩被视为会话演化的一部分

在 Hermes 中，上下文压缩不是“失败后的补丁”，而是 session 生命周期的一部分。压缩之后会：

- 重建 system prompt
- 切出新的 session
- 维护 parent-child session lineage

因此压缩在这里更接近“上下文窗口换代”。

### 11.4 工具分层比表面上更复杂

从模型视角看，所有工具都是 function schema；但从运行时视角看，工具至少分为：

- registry 工具
- agent-loop 直管工具
- memory provider 工具
- MCP 工具
- 插件注册工具

如果不区分这些层，很容易误判某个工具是“普通函数调用”还是“会修改 Agent 内部状态的特殊入口”。

---

## 12. 关键文件说明

### `run_agent.py`

项目核心。负责：

- `AIAgent` 生命周期
- system prompt 构造与缓存
- `run_conversation()` 主循环
- 工具调用执行
- 上下文压缩接入
- 最终收尾与持久化

### `model_tools.py`

负责：

- 工具发现
- 工具 schema 生成
- toolset 过滤后的最终工具视图
- 普通工具分发
- 插件级 `pre_tool_call` / `post_tool_call`

### `toolsets.py`

定义 toolset 以及 toolset 组合关系，是“按能力选择工具”的入口。

### `tools/registry.py`

工具注册中心，保存工具元数据与 handler，是 registry 工具的统一分发点。

### `tools/memory_tool.py`

实现 `MemoryStore`、`MEMORY.md` / `USER.md` 写盘、字符上限控制、安全扫描与冻结快照。

### `agent/memory_manager.py`

负责将内置记忆与外部 provider 统一到同一组生命周期接口上。

### `agent/context_compressor.py`

负责上下文阈值计算、旧工具输出剪枝、摘要生成、摘要迭代更新和压缩后消息重组。

### `agent/prompt_caching.py`

负责为 Anthropic 模型打 `cache_control` 标记，使用 system + recent 3 的缓存策略。

### `hermes_state.py`

提供 SQLite `SessionDB`，支持：

- 会话元信息存储
- 消息存储
- FTS5 全文检索
- 压缩后 session lineage

### `gateway/session.py`

处理 Gateway 模式下的：

- session context prompt
- session key 管理
- transcript 与 legacy 存储兼容

### `gateway/hooks.py`

实现 HookRegistry，负责事件 hook 的发现、注册和分发。

### `agent/skill_commands.py`

实现 skill 扫描、技能消息构造、技能附带文件说明、`/plan` 等 prompt 型命令的共用逻辑。

### `tools/delegate_tool.py`

实现子 Agent 委派、toolset 裁剪、child system prompt 构造、批量 child 并发执行与结果回传。

### `tools/process_registry.py`

实现后台进程注册、状态轮询、输出缓冲、watch pattern、通知队列与 crash recovery。

### `batch_runner.py`

实现批量运行、trajectory 采样、工具统计与多进程数据集处理。

---

## 13. 建议阅读顺序

若需要继续深入阅读源码，推荐按以下顺序进行：

1. `run_agent.py`
2. `model_tools.py`
3. `tools/registry.py`
4. `tools/memory_tool.py`
5. `agent/memory_manager.py`
6. `agent/context_compressor.py`
7. `hermes_state.py`
8. `gateway/session.py`
9. `tools/delegate_tool.py`
10. `tools/process_registry.py`
11. `gateway/hooks.py`
12. `agent/skill_commands.py`

这个顺序基本对应从主执行链到外围扩展层的阅读路径。
