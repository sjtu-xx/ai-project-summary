# Pi Monorepo 项目总结

来源：[badlogic/pi-mono](https://github.com/badlogic/pi-mono)

## 这个项目是什么

`pi-mono` 不是单一应用，而是一个围绕 `pi` 构建的 **AI Agent 单仓**。它把不同职责拆成多个 package：

- `pi-ai`：统一多 provider 的 LLM API
- `pi-agent-core`：Agent runtime，负责消息、工具调用和事件流
- `pi-coding-agent`：真正面向用户的交互式 coding agent CLI
- 另外还有终端 UI、Web UI、Slack bot 和部署工具

如果只看一句话：

**这个仓库的核心目标，是把“模型访问层 + Agent 循环层 + 可交互产品层”拆开，让它既能直接作为 coding agent 使用，也能被别人复用成自己的 Agent 系统。**

## 核心定位

这个仓库主要承担四种角色：

- **LLM 统一接入层**：把 OpenAI、Anthropic、Google、Bedrock、Mistral 等 provider 收敛到统一接口
- **Agent 框架层**：提供可复用的消息状态、工具执行、流式事件和会话推进机制
- **Coding Agent 产品层**：提供 `pi` CLI、会话管理、工具、技能、扩展和交互模式
- **周边能力集合**：包括 TUI、Web UI、Slack bot、GPU/vLLM 部署管理

根 `README.md` 其实已经明确给出阅读重点：如果你是来找 “pi coding agent” 的，先看 `packages/coding-agent`。

## 仓库结构

这是一个 `npm workspaces` 单仓，根 `package.json` 说明了几个关键事实：

- 使用 `npm` 作为主包管理器
- 运行时要求是 `Node.js >= 20`
- 代码使用 ESM
- 构建、检查、测试都由根脚本统一调度

顶层结构里最重要的是：

- `packages/ai/`
- `packages/agent/`
- `packages/coding-agent/`
- `packages/tui/`
- `packages/web-ui/`
- `packages/mom/`
- `packages/pods/`
- `scripts/`

其中真正的主产品链路集中在前三个包里。

## 最重要的分层

如果按职责理解，这个仓库最值得关注的是下面三层。

### 1. 模型接入层：`packages/ai`

这一层负责统一不同 provider 的差异，包括：

- 模型注册与发现
- 流式输出
- tool calling
- thinking/reasoning
- API key 与 OAuth
- 跨 provider 上下文延续

它本质上是一个 **跨 provider 的 LLM 抽象层**。上层并不直接关心每家 API 长什么样，而是通过 `pi-ai` 的统一类型和流接口来访问模型。

### 2. Agent runtime 层：`packages/agent`

这一层负责真正的 Agent 主循环：

- 维护 `AgentState`
- 管理消息流
- 执行工具
- 处理 steering 和 follow-up 队列
- 把一次用户请求拆成若干个 turn

最关键的文件是：

- `packages/agent/src/agent.ts`
- `packages/agent/src/agent-loop.ts`

这里最重要的设计点是：内部一直使用更灵活的 `AgentMessage[]`，只有在真正调用 LLM 的边界，才把它转换成 provider 能接受的 `Message[]`。这让系统可以容纳更多运行时消息类型，而不被 provider 协议绑死。

### 3. 产品编排层：`packages/coding-agent`

这一层才是用户真正感知到的 `pi`。它负责：

- CLI 参数解析
- interactive / print / rpc 模式切换
- 会话恢复与存储
- skills、prompt templates、themes、extensions 的发现和装载
- 内建工具与扩展工具注册
- system prompt 组装
- 把完整 runtime 拼装成一个可以工作的 agent session

如果说 `pi-ai` 解决“怎么和模型说话”，`pi-agent-core` 解决“怎么跑 agent loop”，那 `pi-coding-agent` 解决的是：

**怎样把模型、工具、资源、扩展、会话、UI 真正拼成一个能用的产品。**

## 一条最值得追踪的主流程

读这个仓库时，最有效的黄金路径是：

`CLI -> main -> createAgentSession -> AgentSession -> Agent -> agent-loop -> pi-ai`

具体可以按下面顺序跟：

1. `packages/coding-agent/src/cli.ts`
2. `packages/coding-agent/src/main.ts`
3. `packages/coding-agent/src/core/sdk.ts`
4. `packages/coding-agent/src/core/agent-session.ts`
5. `packages/agent/src/agent.ts`
6. `packages/agent/src/agent-loop.ts`
7. `packages/ai/src/index.ts`

这条链路能把整个仓库最核心的三层一次串起来。

## 主流程是怎么跑起来的

### 第一步：CLI 入口

`packages/coding-agent/src/cli.ts` 非常薄，只做少量进程级初始化，然后把参数交给 `main()`。

这说明 CLI 文件本身不是业务核心，真正的编排都在 `main.ts`。

### 第二步：主调度 `main.ts`

`packages/coding-agent/src/main.ts` 负责：

- 解析 CLI 参数
- 先加载 extensions，以便发现它们自定义的 flags
- 初始化 `AuthStorage`、`ModelRegistry`、`ResourceLoader`
- 解析 session、model、thinking level、tools
- 调用 `createAgentSession()`
- 按模式进入 interactive、print 或 rpc

这一步可以看出：`main.ts` 的职责不是“跑 agent loop”，而是把一切运行前条件准备好。

### 第三步：创建 session

`packages/coding-agent/src/core/sdk.ts` 里最重要的事，是创建 `Agent` 和 `AgentSession`。

它会处理：

- 模型恢复
- thinking level 恢复
- session 恢复
- `getApiKey` 动态解析
- `transformContext` 和 `onPayload`
- 内建工具集合

也就是说，`sdk.ts` 是从“CLI 参数世界”走向“Agent Runtime 世界”的桥。

### 第四步：`AgentSession` 做产品级装配

`packages/coding-agent/src/core/agent-session.ts` 是整个 `pi-coding-agent` 最关键的一层。

它负责：

- 订阅 agent 事件并落盘到 session
- 建立工具注册表
- 绑定 extension hooks
- 从 `ResourceLoader` 取出 skills、context files、system prompt、append system prompt
- 根据当前启用工具重建 system prompt
- 在发 prompt 前处理 `/skill:*` 和 prompt templates

这说明 `AgentSession` 不是简单包装器，而是 **产品编排器**。

### 第五步：`Agent` 进入运行态

`packages/agent/src/agent.ts` 负责维护运行时状态，比如：

- 当前 model
- thinking level
- 当前消息
- 正在进行的流式响应
- 正在执行的工具
- steering / follow-up 队列

这里的 `prompt()` 最终会进入 `_runLoop()`，把一次用户请求送进真正的 agent loop。

### 第六步：`agent-loop` 驱动 turn

`packages/agent/src/agent-loop.ts` 是最值得精读的文件之一。

它的核心节奏是：

1. 把当前上下文转换成 LLM 可接受的上下文
2. 调 `streamSimple` 获取 assistant response
3. 解析出 tool calls
4. 校验参数并执行工具
5. 把 tool results 回写到上下文
6. 如果还有工具调用或 follow-up/steering 消息，就继续下一轮

这其实就是一个典型的 Agent 执行循环，只是这里把：

- 流式输出
- 工具执行
- steering / follow-up
- retry / abort

都纳进了统一事件流里。

### 第七步：`pi-ai` 落到具体 provider

到了最底层，`pi-ai` 才真正接触具体 provider API。

因此可以把整个系统粗略理解成：

- `coding-agent`：面向用户的产品层
- `agent`：运行时控制层
- `ai`：模型通讯层

## 这个仓库最有特点的地方

### 1. 它把“产品”和“框架”拆得很开

很多 AI Agent 项目把模型访问、Agent 循环、CLI/UI 都揉在一起。`pi-mono` 则明显是按层拆开的：

- `pi-ai` 可独立用
- `pi-agent-core` 可独立用
- `pi-coding-agent` 是建立在前两层上的一个具体产品

这让复用性更强，但阅读时必须跨包理解，门槛也更高。

### 2. 它故意把核心做小，把高级能力外置

`packages/coding-agent/README.md` 里讲得很直接：很多其他产品会内建的能力，这里选择不硬塞进 core，比如：

- MCP
- sub-agents
- plan mode
- permission popups
- built-in todos

取而代之的是：

- extensions
- skills
- prompt templates
- themes
- pi packages

这是一种很明确的设计哲学：**核心最小化，扩展能力外置化。**

### 3. `AgentSession` 是真正的“胶水层”

这个仓库最容易低估的不是 `main.ts`，而是 `AgentSession`。

因为真正把这些东西粘起来的是它：

- session persistence
- extension runtime
- resource loading
- tool registry
- system prompt rebuild
- slash command 扩展
- compaction / retry / branching 等会话能力

所以如果你只读 `main.ts`，会知道程序怎么启动；但如果你想知道它“为什么能变成一个可扩展 coding agent”，就必须读 `AgentSession`。

### 4. 它是事件驱动的，不是单纯请求-响应式

`agent` 层非常强调 event streaming。事件不只是给模型看，也是给 UI、extension、session persistence 和中间态控制看的。

这说明它不是“发一个 prompt，拿一个字符串回来”的简单聊天壳，而是一个具有运行中可观察性的 agent runtime。

## 推荐阅读顺序

如果只给一个 30 分钟版本，我会推荐：

1. `README.md`
2. `packages/coding-agent/README.md`
3. `packages/coding-agent/src/main.ts`
4. `packages/coding-agent/src/core/sdk.ts`
5. `packages/coding-agent/src/core/agent-session.ts`
6. `packages/agent/README.md`
7. `packages/agent/src/agent.ts`
8. `packages/agent/src/agent-loop.ts`
9. `packages/ai/README.md`

如果之后还想继续扩展：

10. `packages/coding-agent/src/core/resource-loader.ts`
11. `packages/coding-agent/src/core/tools/index.ts`
12. 再看 `packages/tui`、`packages/mom`、`packages/pods`、`packages/web-ui`

## 一句话总结

`pi-mono` 的本质不是“一个 coding agent 应用”，而是一套分层很清楚的 Agent 平台：`pi-ai` 负责统一模型访问，`pi-agent-core` 负责 agent loop 与工具执行，`pi-coding-agent` 负责把会话、工具、技能、扩展和交互模式装配成最终产品。它最大的特点不是功能堆得多，而是核心做得小、边界划得清、扩展点留得非常多。
