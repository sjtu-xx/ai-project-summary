# Claude Managed Agents API 学习笔记

> 官方文档：https://platform.claude.com/docs/en/managed-agents/overview

---

## 一、核心概念

### Managed Agents 是什么？

**一句话**：Anthropic 托管的 AI Agent 运行环境，你只需定义"做什么"，Anthropic 负责"怎么运行"。

| | Messages API | Claude Managed Agents |
|---|---|---|
| **本质** | 直接调用模型 | 托管的 Agent 运行框架 |
| **适用场景** | 自定义 Agent 循环、精细控制 | 长时间运行任务、异步工作 |
| **基础设施** | 自己搭建 | Anthropic 托管 |

### 四大核心概念

| 概念 | 英文 | 描述 | 生命周期 |
|------|------|------|----------|
| Agent | Agent | 模型 + 系统提示 + 工具 + MCP 服务器 + Skills | 可复用、可版本化 |
| Environment | Environment | 容器模板（预装包、网络访问规则） | 多会话共享 |
| Session | Session | 运行中的 Agent 实例，执行特定任务 | 每个会话独立容器 |
| Events | Events | 用户和 Agent 之间交换的消息 | 持久化存储 |

### 工作流程

```
1. 创建 Agent → 定义模型、系统提示、工具
2. 创建 Environment → 配置容器（包、网络）
3. 创建 Session → 引用 Agent + Environment
4. 发送 Events → 用户消息、流式响应
5. 引导或中断 → 实时控制 Agent 行为
```

---

## 二、快速开始

### 安装 CLI

```bash
# macOS
brew install anthropics/tap/ant
xattr -d com.apple.quarantine "$(brew --prefix)/bin/ant"

# Linux/WSL
VERSION=1.0.0
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')
curl -fsSL "https://github.com/anthropics/anthropic-cli/releases/download/v${VERSION}/ant_${VERSION}_${OS}_${ARCH}.tar.gz" \
  | sudo tar -xz -C /usr/local/bin ant
```

### 安装 SDK

```bash
pip install anthropic  # Python
npm install @anthropic-ai/sdk  # TypeScript
```

### 创建第一个 Session

```python
from anthropic import Anthropic
client = Anthropic()

# 1. 创建 Agent
agent = client.beta.agents.create(
    name="Coding Assistant",
    model="claude-sonnet-4-6",
    system="You are a helpful coding assistant.",
    tools=[{"type": "agent_toolset_20260401"}],
)

# 2. 创建 Environment
environment = client.beta.environments.create(
    name="quickstart-env",
    config={
        "type": "cloud",
        "networking": {"type": "unrestricted"},
    },
)

# 3. 创建 Session
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    title="Quickstart session",
)

# 4. 发送消息并流式响应
with client.beta.sessions.events.stream(session.id) as stream:
    client.beta.sessions.events.send(
        session.id,
        events=[{
            "type": "user.message",
            "content": [{"type": "text", "text": "Create a Python script..."}],
        }],
    )
    for event in stream:
        if event.type == "agent.message":
            for block in event.content:
                print(block.text, end="")
        elif event.type == "session.status_idle":
            break
```

---

## 三、Agent 配置

### Agent 字段说明

| 字段 | 必填 | 描述 |
|------|------|------|
| `name` | ✅ | 人类可读名称 |
| `model` | ✅ | Claude 模型 ID |
| `system` | ❌ | 系统提示，定义行为和角色 |
| `tools` | ❌ | 可用工具（内置工具集 + 自定义工具） |
| `mcp_servers` | ❌ | MCP 服务器列表 |
| `skills` | ❌ | 领域专业技能 |
| `callable_agents` | ❌ | 可调用的其他 Agent（多 Agent 协作） |
| `description` | ❌ | Agent 描述 |
| `metadata` | ❌ | 自定义元数据 |

### Agent 版本管理

```python
# 创建 Agent（version = 1）
agent = client.beta.agents.create(...)

# 更新 Agent（version 递增）
updated_agent = client.beta.agents.update(
    agent.id,
    version=agent.version,  # 必须传递当前版本
    system="Updated system prompt...",
)

# 查看版本历史
for version in client.beta.agents.versions.list(agent.id):
    print(f"Version {version.version}: {version.updated_at}")

# 归档 Agent（只读，现有会话继续运行）
client.beta.agents.archive(agent.id)
```

### Agent 工具集

```python
agent = client.beta.agents.create(
    name="Coding Assistant",
    model="claude-sonnet-4-6",
    tools=[
        # 内置工具集（bash, read, write, edit, glob, grep, web_fetch, web_search）
        {"type": "agent_toolset_20260401"},
    ],
)
```

**内置工具列表**：

| 工具 | 名称 | 描述 |
|------|------|------|
| Bash | `bash` | 执行 shell 命令 |
| Read | `read` | 读取文件 |
| Write | `write` | 写入文件 |
| Edit | `edit` | 编辑文件（字符串替换） |
| Glob | `glob` | 文件模式匹配 |
| Grep | `grep` | 正则搜索 |
| Web fetch | `web_fetch` | 获取 URL 内容 |
| Web search | `web_search` | 搜索网页 |

### 禁用特定工具

```python
agent = client.beta.agents.create(
    ...,
    tools=[{
        "type": "agent_toolset_20260401",
        "configs": [
            {"name": "web_fetch", "enabled": False},
            {"name": "web_search", "enabled": False},
        ],
    }],
)
```

### 自定义工具

```python
agent = client.beta.agents.create(
    name="Weather Agent",
    model="claude-sonnet-4-6",
    tools=[
        {"type": "agent_toolset_20260401"},
        {
            "type": "custom",
            "name": "get_weather",
            "description": "Get current weather for a location",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {"type": "string", "description": "City name"},
                },
                "required": ["location"],
            },
        },
    ],
)
```

**自定义工具最佳实践**：
- **描述要极其详细**：至少 3-4 句话，说明工具用途、何时使用、参数含义
- **合并相关操作**：用 `action` 参数区分操作，而不是创建多个工具
- **使用有意义的命名空间**：如 `db_query`, `storage_read`
- **返回高信号信息**：只返回 Claude 需要的字段

---

## 四、Environment 配置

### 创建 Environment

```python
environment = client.beta.environments.create(
    name="data-analysis",
    config={
        "type": "cloud",
        "packages": {
            "pip": ["pandas", "numpy", "scikit-learn"],
            "npm": ["express"],
        },
        "networking": {"type": "unrestricted"},
    },
)
```

### 包管理器支持

| 字段 | 包管理器 | 示例 |
|------|----------|------|
| `apt` | apt-get | `"ffmpeg"` |
| `cargo` | Rust cargo | `"ripgrep@14.0.0"` |
| `gem` | Ruby gem | `"rails:7.1.0"` |
| `go` | Go modules | `"golang.org/x/tools/cmd/goimports@latest"` |
| `npm` | Node.js npm | `"express@4.18.0"` |
| `pip` | Python pip | `"pandas==2.2.0"` |

### 网络配置

| 模式 | 描述 |
|------|------|
| `unrestricted` | 完整出站网络访问（默认） |
| `limited` | 仅允许 `allowed_hosts` 列表中的域名 |

```python
config = {
    "type": "cloud",
    "networking": {
        "type": "limited",
        "allowed_hosts": ["api.example.com"],
        "allow_mcp_servers": True,
        "allow_package_managers": True,
    },
}
```

**生产环境建议**：使用 `limited` 网络 + 显式 `allowed_hosts` 列表。

### Environment 生命周期

- 持久化直到显式归档或删除
- 多个 Session 可共享同一 Environment
- 每个 Session 获得独立容器实例，不共享文件系统状态

---

## 五、Session 管理

### 创建 Session

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    title="My session",
    metadata={"user_id": "user_123"},
    resources=[
        {
            "type": "github_repository",
            "url": "https://github.com/user/repo",
            "authorization_token": "ghp_xxx",
            "mount_path": "/workspace/repo",
        }
    ],
    vault_ids=[vault.id],
)
```

### Session 状态

| 状态 | 描述 |
|------|------|
| `rescheduling` | 临时错误，自动重试 |
| `running` | Agent 正在处理 |
| `idle` | Agent 等待输入 |
| `terminated` | 会话结束（不可恢复错误） |

### Session 统计信息

```python
session = client.beta.sessions.retrieve(session.id)

# Token 使用量
print(f"Input tokens: {session.usage.input_tokens}")
print(f"Output tokens: {session.usage.output_tokens}")
print(f"Cache read: {session.usage.cache_read_input_tokens}")

# 时间统计
print(f"Active seconds: {session.stats.active_seconds}")
print(f"Duration: {session.stats.duration_seconds}s")
```

---

## 六、Events 和 Streaming

### 事件类型

**用户事件**（发送给 Agent）：

| 类型 | 描述 |
|------|------|
| `user.message` | 用户消息 |
| `user.interrupt` | 中断 Agent |
| `user.custom_tool_result` | 自定义工具结果 |
| `user.tool_confirmation` | 工具确认（允许/拒绝） |
| `user.define_outcome` | 定义目标（Research Preview） |

**Agent 事件**（接收自 Agent）：

| 类型 | 描述 |
|------|------|
| `agent.message` | Agent 文本响应 |
| `agent.thinking` | Agent 思考内容 |
| `agent.tool_use` | 使用内置工具 |
| `agent.tool_result` | 工具执行结果 |
| `agent.mcp_tool_use` | 使用 MCP 工具 |
| `agent.mcp_tool_result` | MCP 工具结果 |
| `agent.custom_tool_use` | 使用自定义工具 |
| `agent.thread_context_compacted` | 上下文压缩 |

**Session 事件**：

| 类型 | 描述 |
|------|------|
| `session.status_running` | Agent 正在处理 |
| `session.status_idle` | Agent 等待输入 |
| `session.status_rescheduled` | 自动重试中 |
| `session.status_terminated` | 会话结束 |
| `session.error` | 错误发生 |

### 流式响应

```python
with client.beta.sessions.events.stream(session.id) as stream:
    # 先发送消息
    client.beta.sessions.events.send(
        session.id,
        events=[{
            "type": "user.message",
            "content": [{"type": "text", "text": "Summarize the README"}],
        }],
    )
    
    # 处理流式事件
    for event in stream:
        if event.type == "agent.message":
            for block in event.content:
                if block.type == "text":
                    print(block.text, end="")
        elif event.type == "session.status_idle":
            break
        elif event.type == "session.error":
            print(f"Error: {event.error.message}")
            break
```

### 中断 Agent

```python
client.beta.sessions.events.send(
    session.id,
    events=[
        {"type": "user.interrupt"},
        {
            "type": "user.message",
            "content": [{"type": "text", "text": "Stop! Focus on something else."}],
        },
    ],
)
```

### 处理自定义工具调用

```python
with client.beta.sessions.events.stream(session.id) as stream:
    for event in stream:
        if event.type == "session.status_idle":
            if event.stop_reason.type == "requires_action":
                for event_id in event.stop_reason.event_ids:
                    tool_event = events_by_id[event_id]
                    result = execute_tool(tool_event.name, tool_event.input)
                    
                    client.beta.sessions.events.send(
                        session.id,
                        events=[{
                            "type": "user.custom_tool_result",
                            "custom_tool_use_id": event_id,
                            "content": [{"type": "text", "text": result}],
                        }],
                    )
```

### 工具确认

当权限策略要求确认时：

```python
client.beta.sessions.events.send(
    session.id,
    events=[{
        "type": "user.tool_confirmation",
        "tool_use_id": event_id,
        "result": "allow",  # 或 "deny"
        "deny_message": "Reason for denial",  # 可选
    }],
)
```

---

## 七、MCP 连接器

### 配置 MCP 服务器

```python
agent = client.beta.agents.create(
    name="GitHub Assistant",
    model="claude-sonnet-4-6",
    mcp_servers=[{
        "type": "url",
        "name": "github",
        "url": "https://api.githubcopilot.com/mcp/",
    }],
    tools=[
        {"type": "agent_toolset_20260401"},
        {"type": "mcp_toolset", "mcp_server_name": "github"},
    ],
)
```

**注意**：MCP 工具集默认权限策略为 `always_ask`（每次调用需确认）。

### Vault 凭证管理

Vault 用于存储用户凭证，与 MCP 服务器配合使用：

```python
# 1. 创建 Vault
vault = client.beta.vaults.create(
    display_name="Alice",
    metadata={"external_user_id": "usr_abc123"},
)

# 2. 添加凭证
credential = client.beta.vaults.credentials.create(
    vault_id=vault.id,
    display_name="Alice's Slack",
    auth={
        "type": "mcp_oauth",
        "mcp_server_url": "https://mcp.slack.com/mcp",
        "access_token": "xoxp-...",
        "expires_at": "2026-04-15T00:00:00Z",
        "refresh": {
            "token_endpoint": "https://slack.com/api/oauth.v2.access",
            "client_id": "1234567890.0987654321",
            "scope": "channels:read chat:write",
            "refresh_token": "xoxe-1-...",
            "token_endpoint_auth": {
                "type": "client_secret_post",
                "client_secret": "abc123...",
            },
        },
    },
)

# 3. 在 Session 中使用
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    vault_ids=[vault.id],
)
```

### 凭证类型

| 类型 | 描述 | 适用场景 |
|------|------|----------|
| `mcp_oauth` | OAuth 2.0 凭证 | 需要 Token 刷新的服务 |
| `static_bearer` | 静态 Bearer Token | API Key、PAT 等 |

---

## 八、Skills（技能）

### 什么是 Skills？

Skills 是可复用的、基于文件系统的领域专业知识，让通用 Agent 变成专家。

**与普通 Prompt 的区别**：
- Prompt：对话级指令，一次性
- Skills：按需加载，只影响需要时的上下文

### Skills 类型

| 类型 | 描述 |
|------|------|
| Anthropic Skills | 预构建技能（PowerPoint, Excel, Word, PDF 等） |
| Custom Skills | 自定义技能 |

### 启用 Skills

```python
agent = client.beta.agents.create(
    name="Financial Analyst",
    model="claude-sonnet-4-6",
    system="You are a financial analysis agent.",
    skills=[
        {"type": "anthropic", "skill_id": "xlsx"},
        {"type": "custom", "skill_id": "skill_abc123", "version": "latest"},
    ],
)
```

**限制**：每个 Session 最多 20 个 Skills。

---

## 九、Memory（记忆）[Research Preview]

### 概述

Memory Store 让 Agent 跨 Session 保持记忆——用户偏好、项目约定、错误教训等。

**特点**：
- 自动检查和更新，无需额外配置
- 可通过 API 直接管理
- 每次修改创建不可变版本（审计、回滚）

### 创建 Memory Store

```python
store = client.beta.memory_stores.create(
    name="User Preferences",
    description="Per-user preferences and project context.",
)
```

### 预填充 Memory

```python
client.beta.memory_stores.memories.write(
    memory_store_id=store.id,
    path="/formatting_standards.md",
    content="All reports use GAAP formatting. Dates are ISO-8601...",
)
```

### 附加到 Session

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    resources=[{
        "type": "memory_store",
        "memory_store_id": store.id,
        "access": "read_write",  # 或 "read_only"
        "prompt": "User preferences. Check before starting any task.",
    }],
)
```

**限制**：每个 Session 最多 8 个 Memory Store。

### Memory 工具

Agent 自动获得以下工具：

| 工具 | 描述 |
|------|------|
| `memory_list` | 列出 Memory |
| `memory_search` | 全文搜索 |
| `memory_read` | 读取内容 |
| `memory_write` | 创建/覆盖 |
| `memory_edit` | 修改 |
| `memory_delete` | 删除 |

---

## 十、多 Agent 协作 [Research Preview]

### 概念

**Thread**：每个 Agent 运行在独立的 Thread 中，拥有隔离的上下文和对话历史。

**协调者模式**：
- 主 Agent（协调者）调用其他 Agent
- 每个 Agent 使用自己的配置（模型、提示、工具）
- 共享容器和文件系统

### 配置可调用 Agent

```python
orchestrator = client.beta.agents.create(
    name="Engineering Lead",
    model="claude-sonnet-4-6",
    system="You coordinate engineering work. Delegate to reviewers and testers.",
    tools=[{"type": "agent_toolset_20260401"}],
    callable_agents=[
        {"type": "agent", "id": reviewer_agent.id, "version": reviewer_agent.version},
        {"type": "agent", "id": tester_agent.id, "version": tester_agent.version},
    ],
)
```

**限制**：只支持一层委托（协调者可调用其他 Agent，但被调用的 Agent 不能再调用其他 Agent）。

### Thread 管理

```python
# 列出所有 Thread
for thread in client.beta.sessions.threads.list(session.id):
    print(f"[{thread.agent_name}] {thread.status}")

# 流式读取特定 Thread
with client.beta.sessions.threads.stream(
    thread.id, session_id=session.id
) as stream:
    for event in stream:
        if event.type == "agent.message":
            print(event.content[0].text)
```

### 多 Agent 事件

| 类型 | 描述 |
|------|------|
| `session.thread_created` | 创建新 Thread |
| `session.thread_idle` | Thread 完成工作 |
| `agent.thread_message_sent` | Agent 发送消息到其他 Thread |
| `agent.thread_message_received` | Agent 接收来自其他 Thread 的消息 |

---

## 十一、Outcomes（目标）[Research Preview]

### 概念

Outcomes 把"对话"变成"工作"——定义目标，Agent 迭代直到达成。

**机制**：
1. 定义目标 + Rubric（评分标准）
2. Agent 执行任务
3. Grader 独立评估（单独上下文窗口）
4. 返回反馈，Agent 继续迭代

### 创建 Rubric

```markdown
# DCF Model Rubric

## Revenue Projections
- Uses historical revenue data from the last 5 fiscal years
- Projects revenue for at least 5 years forward
- Growth rate assumptions are explicitly stated and reasonable

## Cost Structure
- COGS and operating expenses are modeled separately
- Margins are consistent with historical trends

## Discount Rate
- WACC is calculated with stated assumptions

## Output Quality
- All figures in a single .xlsx file
- Key assumptions on a separate "Assumptions" sheet
```

### 定义 Outcome

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    title="Financial analysis",
)

client.beta.sessions.events.send(
    session.id,
    events=[{
        "type": "user.define_outcome",
        "description": "Build a DCF model for Costco in .xlsx",
        "rubric": {"type": "text", "content": RUBRIC},
        "max_iterations": 5,  # 默认 3，最大 20
    }],
)
```

### Outcome 评估事件

| 类型 | 描述 |
|------|------|
| `span.outcome_evaluation_start` | 开始评估 |
| `span.outcome_evaluation_ongoing` | 评估进行中（心跳） |
| `span.outcome_evaluation_end` | 评估结束 |

**评估结果**：

| 结果 | 下一步 |
|------|--------|
| `satisfied` | Session 进入 idle |
| `needs_revision` | Agent 开始新一轮迭代 |
| `max_iterations_reached` | 达到最大迭代次数 |
| `failed` | Rubric 与任务不匹配 |
| `interrupted` | 用户中断 |

### 获取输出文件

```python
# 列出 Session 产生的文件
files = client.beta.files.list(scope_id=session.id)

# 下载文件
content = client.beta.files.download(files.data[0].id)
content.write_to_file("output.xlsx")
```

---

## 十二、权限策略

### 策略类型

| 类型 | 描述 |
|------|------|
| `always_allow` | 自动执行，无需确认 |
| `always_ask` | 每次执行前需用户确认 |

### 配置

```python
agent = client.beta.agents.create(
    ...,
    tools=[{
        "type": "agent_toolset_20260401",
        "configs": [
            {
                "name": "bash",
                "permission_policy": {"type": "always_ask"},
            },
        ],
    }],
)
```

---

## 十三、Rate Limits

| 操作 | 限制 |
|------|------|
| Create 端点（agents, sessions, environments 等） | 60 请求/分钟 |
| Read 端点（retrieve, list, stream 等） | 600 请求/分钟 |

组织级别的消费限制和基于 Tier 的 Rate Limits 也适用。

---

## 十四、Beta Headers

所有 Managed Agents API 请求需要 beta header：

```
anthropic-beta: managed-agents-2026-04-01
```

SDK 自动设置。Research Preview 功能需要额外 header：

```
anthropic-beta: managed-agents-2026-04-01,managed-agents-2026-04-01-research-preview
```

---

## 十五、整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Managed Agents API                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │    Agents    │    │ Environments │    │   Sessions   │       │
│  │  (可复用)     │    │  (容器模板)   │    │  (运行实例)   │       │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
│         │                   │                    │               │
│         └───────────────────┼────────────────────┘               │
│                             │                                    │
│                    ┌────────┴────────┐                          │
│                    │    Container    │                          │
│                    │   (Sandbox)     │                          │
│                    └─────────────────┘                          │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │    Tools     │    │ MCP Servers  │    │    Skills    │       │
│  │  (内置+自定义) │    │   (外部服务)  │    │  (领域专家)   │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │    Vaults    │    │ Memory Store │    │   Outcomes   │       │
│  │  (凭证管理)   │    │  (跨会话记忆) │    │  (目标导向)   │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 参考链接

- [Managed Agents Overview](https://platform.claude.com/docs/en/managed-agents/overview)
- [Quickstart](https://platform.claude.com/docs/en/managed-agents/quickstart)
- [Agent Setup](https://platform.claude.com/docs/en/managed-agents/agent-setup)
- [Environments](https://platform.claude.com/docs/en/managed-agents/environments)
- [Tools](https://platform.claude.com/docs/en/managed-agents/tools)
- [Events and Streaming](https://platform.claude.com/docs/en/managed-agents/events-and-streaming)
- [MCP Connector](https://platform.claude.com/docs/en/managed-agents/mcp-connector)
- [Skills](https://platform.claude.com/docs/en/managed-agents/skills)
- [Memory](https://platform.claude.com/docs/en/managed-agents/memory)
- [Multi-agent](https://platform.claude.com/docs/en/managed-agents/multi-agent)
- [Define Outcomes](https://platform.claude.com/docs/en/managed-agents/define-outcomes)
- [Vaults](https://platform.claude.com/docs/en/managed-agents/vaults)
- [API Reference](https://platform.claude.com/docs/en/api/beta/sessions)
