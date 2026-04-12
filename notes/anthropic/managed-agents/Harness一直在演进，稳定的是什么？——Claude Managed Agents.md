

> 原文：[Scaling Managed Agents: Decoupling the brain from the hands](https://www.anthropic.com/engineering/managed-agents)
> 作者：Lance Martin, Gabe Cemaj, Michael Cohen (Anthropic)
> 对应的api文档： https://platform.claude.com/docs/en/managed-agents/overview

---

> [!note]
> 感觉anthropic在cc源码泄漏后，想把cc的核心能力全部放在云端了。

## 一句话总结

**构建稳定的接口，而非假设特定的实现。** —— 这是 Managed Agents 的核心设计哲学。

---

## 问题：Harness 中的假设会"过期"

### 一个真实的故事

Claude Sonnet 4.5 有个奇怪的行为——"上下文焦虑"。当它感知到上下文快用完时，会过早结束任务。

解决方案很简单：在 Harness（控制框架）里加个上下文重置机制。

但问题来了：当同样的 Harness 用在 Claude Opus 4.5 上时，这个行为消失了。之前加的重置机制变成了"死代码"。

### 问题的本质

Harness 编码的是"Claude 做不到什么"的假设：
- "Claude 会提前结束任务" → 加重置机制
- "Claude 会遗忘上下文" → 加记忆工具
- "Claude 会做错误决策" → 加人工确认

但随着模型越来越强，这些假设一个个失效。**昨天补的坑，明天就成了绊脚石。**

> 这呼应了 Rich Sutton 的《The Bitter Lesson》：利用人类知识编码的假设，往往随计算/模型能力提升而变得过时。

---

## 解决方案：向操作系统学习

操作系统面对过同样的问题：如何为"尚未想到的程序"设计系统？

答案是：**虚拟化硬件，提供稳定抽象**。

- `read()` 不关心你是 1970 年代的磁盘还是现代 SSD
- `process` 不关心你跑在什么 CPU 上

Managed Agents 做了同样的事——虚拟化 Agent 的三大组件：

| 组件 | 类比 | 本质 | 核心接口 |
|------|------|------|----------|
| Session | 内存 | 只追加的事件日志 | `getEvents()` |
| Harness | CPU | 调用 Claude 并路由工具 | `wake(sessionId)` |
| Sandbox | 外设 | 代码执行环境 | `execute(name, input) → string` |

**接口稳定，实现可变。**

---

## 三大核心抽象详解

### 1. Session：记忆的容器

**它是什么**：一个只追加的事件日志，记录会话中发生的一切。

**为什么需要它**：
- Claude 的上下文窗口有限，长任务会超出
- 传统方案（压缩、修剪）都是**不可逆**的——删掉的内容无法恢复
- Session 把完整历史存在外面，Claude 可以随时回去查看

**怎么用**：
```python
# 查询历史事件
events = getEvents(start=100, end=150)  # 查看第100-150条事件
events = getEvents(before="event_500", count=10)  # 查看某事件前的10条
```

**关键洞察**：
- Session ≠ Claude 的 Context Window
- Session 是完整的、可回溯的历史记录
- Context Window 是当前窗口的快照，由 Harness 决定如何填充

### 2. Harness：无状态的大脑

**它是什么**：一个循环——调用 Claude，拿到工具调用，执行工具，再调用 Claude。

**关键特性**：**无状态**。所有状态都在 Session 里。

**为什么无状态很重要**：
```
传统方式：
Harness 崩溃 → 状态丢失 → 会话报废

解耦方式：
Harness 崩溃 → 新 Harness 调用 wake(sessionId) → 从 Session 恢复 → 继续工作
```

**核心接口**：
```python
wake(sessionId)           # 唤醒一个会话
getSession(id)            # 获取事件日志
emitEvent(id, event)      # 写入新事件
```

### 3. Sandbox：可互换的双手

**它是什么**：代码执行环境——容器、手机、模拟器，都行。

**关键接口**：
```python
execute(name, input) → string
provision({resources})    # 创建新沙箱
```

就这么简单：给名字和输入，拿回一个字符串。

**Harness 不需要知道**：
- 沙箱是什么类型
- 在哪里运行
- 怎么创建的

这带来了巨大的灵活性——沙箱可以随时创建、销毁、替换。

---

## 架构演进：从"宠物"到"牛群"

### 初始设计：宠物模式

把所有东西塞进一个容器：

```
┌─────────────────────────────────────┐
│            一个容器                   │
│  ┌───────────────────────────────┐  │
│  │  Harness + Session + Sandbox  │  │
│  │  + 用户数据 + 凭证             │  │
│  └───────────────────────────────┘  │
│                                     │
│  这个容器有个名字，必须精心呵护       │
│  一旦挂掉，会话就没了               │
└─────────────────────────────────────┘
```

**问题清单**：

| 问题 | 具体表现 |
|------|----------|
| 调试困境 | 只能通过 WebSocket 观察，无法定位故障根源 |
| 安全风险 | 容器有用户数据，工程师不能随便进去调试 |
| 状态耦合 | 容器死亡 = 会话丢失 |
| 扩展困难 | 每个 Harness 都要一个容器，启动慢 |
| 网络限制 | 客户想连自己的 VPC？要么网络对等，要么自己跑 Harness |

### 解耦设计：牛群模式

把"大脑"和"双手"分开：

```
         ┌──────────────┐
         │   Harness    │  ← 无状态，可随时重启
         │   (大脑)      │
         └──────┬───────┘
                │
                │ execute(name, input) → string
                │
         ┌──────┴───────┐
         │              │
    ┌────┴────┐    ┌────┴────┐
    │Sandbox 1│    │Sandbox 2│  ← 可互换，按需创建/销毁
    │  (手1)   │    │  (手2)   │
    └─────────┘    └─────────┘

         ┌──────────────┐
         │   Session    │  ← 独立存储，持久化
         │  (记忆)       │
         └──────────────┘
```

**故障恢复流程**：

```
沙箱崩溃：
  Harness 捕获错误 → 告诉 Claude → Claude 决定重试 → provision() 创建新容器 → 继续

Harness 崩溃：
  新 Harness → wake(sessionId) → 从 Session 读取最后事件 → 继续
```

**结果**：容器从"精心呵护的宠物"变成了"可随时替换的牛群"。

---

## 性能提升：TTFT 降低 60%-90%

TTFT（Time To First Token）= 用户等待第一个 token 的时间。这是用户最直接感知的延迟。

| 架构 | 流程 | 问题 |
|------|------|------|
| 耦合设计 | 接收任务 → 启动容器 → 克隆仓库 → 初始化 → 开始推理 | 每个会话都要等这整套流程 |
| 解耦设计 | 接收任务 → 开始推理（需要时再创建容器） | 不需要沙箱的会话立即开始 |

**效果**：
- p50 TTFT 下降约 **60%**
- p95 TTFT 下降超过 **90%**

---

## 安全设计：凭证永远不在沙箱里

### 问题场景

在耦合设计中：
```
沙箱里跑着 Claude 生成的代码
        +
沙箱里有访问凭证
        =
提示词注入可能诱导 Claude 读取凭证 → 攻击者获得凭证 → 可以启动新会话绕过限制
```

### 解决方案

**核心原则**：凭证永远不在沙箱可达范围内。

| 场景 | 实现方式 | 安全机制 |
|------|----------|----------|
| Git 操作 | 初始化时克隆仓库，token 绑定到 git remote | 沙箱里 push/pull 不接触 token |
| 自定义工具 | OAuth token 存在 Vault，通过代理调用 | 代理取凭证，沙箱只看到调用结果 |

**架构**：
```
┌─────────────────┐
│     Harness     │  ← 不存凭证
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
┌───┴───┐ ┌───┴────┐
│Sandbox│ │MCP代理  │  ← 沙箱无凭证，代理有
│(无凭证)│ │        │
└───────┘ └───┬────┘
              │
        ┌─────┴─────┐
        │   Vault   │  ← 凭证存在这里
        │ (凭证存储) │
        └───────────┘
```

---

## Session vs Context Window：不是一回事

| 维度 | Session | Context Window |
|------|---------|----------------|
| 存储位置 | 独立存储，Harness 和沙箱之外 | Claude 模型内部 |
| 容量 | 无限制（只追加日志） | 受限于模型上下文窗口（200K token） |
| 可访问性 | 任意位置查询、回溯、重读 | 只能访问当前窗口内容 |
| 持久性 | 永久保存 | 任务结束即消失 |
| 决策性质 | 可逆（完整历史可恢复） | 不可逆（压缩/修剪后无法恢复） |

**传统方法的困境**：

| 方法 | 描述 | 问题 |
|------|------|------|
| 压缩 | Claude 保存上下文摘要 | 信息丢失，无法恢复 |
| 记忆工具 | 写入文件，跨会话学习 | 需要主动管理，增加复杂度 |
| 修剪 | 删除旧工具结果/思考块 | 不可逆，可能删掉重要内容 |

**Session 的优势**：不做取舍决策，保留完整历史，让 Claude 按需查询。

---

## 扩展性：多大脑，多双手

### Many Brains（多大脑）

**解耦前**：想扩展？每个 Harness 都要一个完整容器。

**解耦后**：
- Harness 是无状态的，启动一个就是加个进程
- 沙箱按需创建，不占用 Harness 资源
- 连接客户 VPC？Harness 不在容器里，网络假设不存在了

### Many Hands（多双手）

每个"手"都是统一接口 `execute(name, input) → string`，可以是：
- 容器
- 手机
- 模拟器
- MCP 服务器
- 任何能执行代码的东西

**高级玩法**：大脑之间可以传递"手"

```
Brain 1 ──把 Hand A 传递给──→ Brain 2
   │                            │
   └── Hand A ──────────────────┘
```

---

## 为什么叫 "Meta-Harness"？

Managed Agents 不做具体 Harness，而是提供一个框架，让各种 Harness 都能跑在上面。

| 对什么有态度（接口层面） | 对什么无态度（实现层面） |
|--------------------------|--------------------------|
| Claude 需要操作状态 | 具体 Session 怎么存 |
| Claude 需要执行计算 | 沙箱是什么类型、有多少个 |
| 需要支持多大脑多双手 | 大脑和手在哪里、怎么分布 |
| 长期运行要安全可靠 | 具体容错策略怎么实现 |
| 需要持久化上下文 | 具体上下文工程怎么做 |

**类比**：
- 操作系统提供 `read()` 接口，不关心底层是磁盘还是 SSD
- Managed Agents 提供 Session/Harness/Sandbox 接口，不关心具体实现

---

## 设计原则总结

| 原则 | 解释 |
|------|------|
| 接口稳定，实现可变 | 为"尚未想到的程序"设计系统 |
| 解耦状态 | Session 独立于 Harness 和沙箱 |
| 无状态 Harness | 可随时重启，从 Session 恢复 |
| 统一工具接口 | `execute(name, input) → string`，足够简单足够通用 |
| 安全边界 | 凭证永远不在沙箱可达范围内 |
| 按需分配 | 容器仅在需要时创建，降低 TTFT |

---

## 核心图示：整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        Managed Agents                        │
│                                                              │
│  ┌──────────────┐        ┌──────────────┐                   │
│  │   Harness    │←──────→│   Session    │                   │
│  │   (大脑)      │        │   (记忆)      │                   │
│  │   无状态      │        │   持久化      │                   │
│  └──────┬───────┘        └──────────────┘                   │
│         │                                                    │
│         │ execute(name, input) → string                      │
│         │                                                    │
│    ┌────┴────────────────────────────────┐                  │
│    │                                     │                  │
│  ┌─┴────┐  ┌────────┐  ┌────────┐  ┌────┴───┐              │
│  │沙箱1  │  │ 沙箱2  │  │ MCP    │  │ 自定义 │              │
│  │(容器) │  │ (手机) │  │ 服务器 │  │  工具  │              │
│  └──────┘  └────────┘  └────────┘  └────────┘              │
│                                                              │
│  ┌─────────────────────────────────────┐                    │
│  │            Vault (凭证存储)           │                    │
│  │            Sandbox 不可达             │                    │
│  └─────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 参考链接

- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Harness design for long-running apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [The Bitter Lesson](http://www.incompleteideas.net/IncIdeas/BitterLesson.html)
- [TAOUP - Chapter 3](http://www.catb.org/esr/writings/taoup/html/ch03s01.html)
- [Managed Agents Docs](https://platform.claude.com/docs/en/managed-agents/overview)
- [Pets vs Cattle](https://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/)
