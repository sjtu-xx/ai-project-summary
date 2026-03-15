# 《The Complete Guide to Building Skill for Claude》详细翻译

来源：<https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf>

## 目录

- 引言
- 第 1 章：基础概念
- 第 2 章：规划与设计
- 第 3 章：测试与迭代
- 第 4 章：分发与分享
- 第 5 章：模式与故障排查
- 第 6 章：资源与参考
- 附录 A：快速检查清单
- 附录 B：YAML frontmatter
- 附录 C：完整 Skill 示例

## 引言

这是一份关于如何为 Claude 构建 Skill 的完整指南。

Skill 可以理解为一组经过打包的指令，用来教 Claude 如何稳定处理某类任务或工作流。对于那些会反复出现的任务，Skill 的价值尤其明显。与其在每一次对话里重新解释你的偏好、流程和领域知识，不如把这些内容写成 Skill，一次配置，后续复用。

文档认为，Skill 特别适合以下类型的工作：

- 从规格生成前端设计
- 按固定方法做研究
- 生成符合团队规范的文档
- 编排多步骤流程
- 配合 MCP 一起使用工具能力

这份指南会介绍：

- Skill 的技术结构与最佳实践
- 独立 Skill 与 MCP 增强 Skill 的常见模式
- 一些已经被验证有效的设计方法
- 如何测试、迭代和分发 Skill

文档同时指出，读者可以按两条路径阅读：

- 如果你做的是独立 Skill，重点看 Fundamentals、Planning and Design，以及前两类常见 use case
- 如果你已经有 MCP，重点看 Skills + MCP 相关部分和 MCP 增强类 use case

无论走哪条路径，底层的技术要求是一致的。

文档给出的目标也很明确：掌握这些内容后，你应该能在一次集中工作里完成一个可用 Skill 的首版，通常大约需要 15 到 30 分钟。

## 第 1 章：基础概念

## 什么是 Skill？

Skill 是一个文件夹，通常包含以下内容：

- `SKILL.md`：必需。主说明文件。
- `scripts/`：可选。放可执行脚本，例如 Python、Bash 等。
- `references/`：可选。放按需加载的参考资料。
- `assets/`：可选。放模板、字体、图标等输出资源。

文档给出的基本结构如下：

```txt
your-skill-name/
  SKILL.md
  scripts/
  references/
  assets/
```

## 核心设计原则

### 渐进式披露

Skill 采用三层结构：

- 第一层：YAML frontmatter
  - 始终会被加载到 Claude 的系统提示中
  - 只提供最基本的信息，让 Claude 知道这个 Skill 在什么情况下应该被使用
- 第二层：`SKILL.md` 正文
  - 当 Claude 判断这个 Skill 与当前任务相关时，才会加载正文
  - 这里放完整说明和操作指导
- 第三层：Skill 目录中的链接文件
  - Claude 可以在需要时再去查看
  - 用于承载不适合放进主文档的大量细节

文档强调，这种渐进式披露可以尽量节省上下文，同时保留专业能力。

### 可组合性

Claude 可以同时加载多个 Skill，因此 Skill 的设计要能与其他 Skill 一起工作，而不是默认自己是唯一生效的能力。

### 可移植性

Skill 应尽量保证在 Claude.ai、Claude Code 和 API 中都能一致工作。只要环境支持这个 Skill 所依赖的能力或脚本，一个 Skill 就可以跨不同入口复用。

## 面向 MCP 构建者：Skill 与连接器

文档对 MCP 构建者强调，如果你已经有一个可用的 MCP Server，那么最困难的“连接能力”部分其实已经完成了。Skill 是加在它上面的“知识层”，用来沉淀你已经掌握的流程经验和最佳实践。

### 厨房类比

文档用一个很直观的比喻来解释：

- MCP 提供的是专业厨房：工具、食材、设备、实时访问能力
- Skill 提供的是菜谱：告诉 Claude 该按什么步骤做出结果

两者配合后，用户就不需要自己摸索每一步该怎么做。

文档用表格说明二者差异：

- MCP 负责把 Claude 连接到外部服务，例如 Notion、Asana、Linear
- Skill 负责教 Claude 如何有效使用这些服务
- MCP 决定 Claude 能做什么
- Skill 决定 Claude 应该怎么做

### 为什么这对 MCP 用户很重要

如果没有 Skill，常见问题是：

- 用户接上 MCP 后仍然不知道接下来该做什么
- 经常会收到“怎么用这个集成做某件事”的支持问题
- 每次对话都要重新解释流程
- 用户提示方式不一致，结果也不一致
- 用户容易把工作流指导问题误认为连接器本身的问题

如果配套 Skill，则会变成：

- 常见工作流可以自动触发
- 工具调用方式更稳定
- 最佳实践被默认带入
- 集成的学习门槛更低

## 第 2 章：规划与设计

## 从用例开始

文档建议，在写任何内容之前，先确定 2 到 3 个具体 use case。

一个好的 use case 定义示例如下：

```txt
Use Case: Project Sprint Planning
Trigger: User says "help me plan this sprint" or "create sprint tasks"
Steps:
1. Fetch current project status from Linear (via MCP)
2. Analyze team velocity and capacity
3. Suggest task prioritization
4. Create tasks in Linear with proper labels and estimates
Result: Fully planned sprint with tasks created
```

文档建议你先回答这些问题：

- 用户想完成什么？
- 这个目标需要哪些多步骤流程？
- 需要哪些工具？是内置工具还是 MCP？
- 哪些领域知识或最佳实践应该被写进 Skill？

## 常见的 Skill 用例类别

### 类别 1：文档与资产生成

适用场景：

- 创建高质量、风格一致的输出
- 包括文档、演示、应用、设计、代码、海报等

文档举的例子是 `frontend-design` skill，描述大意为：

“用于创建具有鲜明风格、达到生产质量的前端界面，适用于构建 Web 组件、页面、设计产物、海报或应用。”

常见技巧：

- 内嵌风格指南和品牌标准
- 模板结构
- 输出完成前的质量检查清单
- 不依赖外部工具，仅使用 Claude 内置能力

### 类别 2：工作流自动化

适用场景：

- 有明确步骤的多阶段流程
- 需要统一方法和执行顺序
- 可能跨多个 MCP 服务

文档举的例子是 `skill-creator` skill。

它的定位是：

“以交互方式指导用户创建新 Skill，包含用例定义、frontmatter 生成、主说明编写和验证。”

常见技巧：

- 分步骤流程，并设置验证门
- 提供常见结构模板
- 内置检查和改进建议
- 支持迭代修正

### 类别 3：MCP 增强

适用场景：

- Skill 用来增强某个 MCP Server 的实际可用性
- 让 Claude 不只是“能调工具”，而是“知道怎样调工具更好”

文档举的例子是来自 Sentry 的 `sentry-code-review` skill。

常见技巧：

- 多个 MCP 调用串联执行
- 内嵌领域专业知识
- 帮用户补足本来需要单独说明的上下文
- 对常见 MCP 错误做预处理和兜底

## 定义成功标准

文档建议在设计阶段就明确“这个 Skill 怎样才算成功”。

### 定量指标

可以参考的量化指标：

- 90% 的相关查询能正确触发这个 Skill
- 相比没有 Skill，完成流程所需的工具调用更少
- 每次工作流中的 API 调用失败数为 0

对应的测量方法：

- 准备 10 到 20 个应该触发的测试查询，统计自动加载比例
- 比较有无 Skill 时的工具调用次数和 token 消耗
- 观察 MCP Server 日志里的错误和重试情况

### 定性指标

可以关注的定性指标：

- 用户不需要反复提示 Claude 下一步该做什么
- 工作流在无需用户纠正的情况下完成
- 在不同会话中结果仍然保持一致
- 新用户能不能在几乎没有指导的情况下第一次就用起来

文档说明，这些指标更像经验性目标，而不是严格精确的评估阈值。

## 技术要求

### 文件结构

文档给出的标准结构如下：

```txt
your-skill-name/
SKILL.md
scripts/
  process_data.py
  validate.sh
references/
  api-guide.md
  examples/
assets/
  report-template.md
```

### 关键规则

#### `SKILL.md` 命名规则

- 文件名必须严格是 `SKILL.md`
- 不能写成 `SKILL.MD`、`skill.md` 等变体

#### Skill 文件夹命名规则

- 文件夹名必须使用 kebab-case
- 不能有空格
- 不能有下划线
- 不能有大写字母

#### 不要放 `README.md`

- Skill 文件夹内部不要放 `README.md`
- 人类读者使用的说明应放在仓库根目录，而 Skill 内部文档放到 `SKILL.md` 或 `references/`

## YAML frontmatter：最重要的部分

文档反复强调，YAML frontmatter 是决定 Claude 是否会加载这个 Skill 的关键。

### 最小必需格式

```yaml
---
name: your-skill-name
description: What it does. Use when user asks to [specific phrases].
---
```

### 字段要求

#### `name`（必填）

- 只能使用 kebab-case
- 不能有空格或大写字母
- 最好与文件夹名一致

#### `description`（必填）

- 必须同时包含两部分：
  - 这个 Skill 做什么
  - 什么时候应该使用它
- 长度不超过 1024 字符
- 不能包含 XML 标签尖括号
- 应写入用户可能会说的具体任务短语
- 如有必要，写明相关文件类型

#### `license`（可选）

- 公开 Skill 时可使用
- 常见值如 `MIT`、`Apache-2.0`

#### `compatibility`（可选）

- 长度 1 到 500 字符
- 用于说明环境要求，例如目标产品、所需系统包、网络要求等

#### `metadata`（可选）

- 可放自定义键值对
- 文档建议的字段包括 `author`、`version`、`mcp-server`

示例：

```yaml
metadata:
  author: ProjectHub
  version: 1.0.0
  mcp-server: projecthub
```

## 安全限制

frontmatter 中禁止出现：

- XML 形式的尖括号
- 名称中包含 `claude` 或 `anthropic`

原因是 frontmatter 会被放进 Claude 的系统提示，安全要求更高。

## 如何编写有效的 Skill

### `description` 字段

文档引用了 Anthropic 工程博客中的观点：

frontmatter 提供的是“刚好足够 Claude 判断何时使用这个 Skill 的最小信息”，它是渐进式披露的第一层。

推荐结构为：

```txt
[What it does] + [When to use it] + [Key capabilities]
```

### 好的 `description` 示例

```txt
description: Analyzes Figma design files and generates developer handoff documentation. Use when user uploads .fig files, asks for "design specs", "component documentation", or "design-to-code handoff".
```

```txt
description: Manages Linear project workflows including sprint planning, task creation, and status tracking. Use when user mentions "sprint", "Linear tasks", "project planning", or asks to "create tickets".
```

```txt
description: End-to-end customer onboarding workflow for PayFlow. Handles account creation, payment setup, and subscription management. Use when user says "onboard new customer", "set up subscription", or "create PayFlow account".
```

### 不好的 `description` 示例

```txt
description: Helps with projects.
```

```txt
description: Creates sophisticated multi-page documentation systems.
```

```txt
description: Implements the Project entity model with hierarchical relationships.
```

这些不好的例子存在的问题分别是：

- 过于模糊
- 没有触发条件
- 太偏技术实现，缺少用户场景

## 编写主说明正文

在 frontmatter 之后，正文使用 Markdown 编写。

文档建议的结构模板如下：

```txt
---
name: your-skill
description: [...]
---

# Your Skill Name

## Instructions

### Step 1: [First Major Step]
Clear explanation of what happens.

Example:
bash
python scripts/fetch_data.py --project-id PROJECT_ID
Expected output: [describe what success looks like]

(Add more steps as needed)

Examples

Example 1: [common scenario]

User says: "Set up a new marketing campaign"

Actions:

1. Fetch existing campaigns via MCP
2. Create new campaign with provided parameters

Result: Campaign created with confirmation link

(Add more examples as needed)

Troubleshooting

Error: [Common error message]

Cause: [Why it happens]

Solution: [How to fix]

(Add more error cases as needed)
```

## 指令编写最佳实践

### 具体且可执行

好的写法：

```txt
Run `python scripts/validate.py --input {filename}` to check data format.

If validation fails, common issues include:
- Missing required fields (add them to the CSV)
- Invalid date formats (use YYYY-MM-DD)
```

差的写法：

```txt
Validate the data before proceeding.
```

### 包含错误处理

例如：

```txt
## Common Issues
### MCP Connection Failed
If you see "Connection refused":
1. Verify MCP server is running: Check Settings > Extensions
2. Confirm API key is valid
3. Try reconnecting: Settings > Extensions > [Your Service] > Reconnect
```

### 清晰引用附带资源

例如在正文中明确说明：

在编写查询前，先查看 `references/api-patterns.md`，其中包含：

- 限流建议
- 分页模式
- 错误码与处理方法

### 使用渐进式披露

`SKILL.md` 只保留核心指令，详细文档尽量移动到 `references/`，再在正文中显式引用。

## 第 3 章：测试与迭代

文档指出，Skill 的测试可以按不同强度来做：

- 在 Claude.ai 中手工测试
- 在 Claude Code 中做脚本化测试
- 通过 Skills API 构建系统化评估

选择哪种方式取决于：

- 你的质量要求
- Skill 的使用规模
- 这个 Skill 对外影响有多大

文档特别给了一个建议：

先围绕一个高难度、代表性的任务不断迭代，直到 Claude 能稳定完成，再把这套做法抽象成 Skill。这样通常比一开始就大范围铺开测试更高效。

## 推荐的测试方法

文档推荐从三个方面测试。

### 1. 触发测试

目标：确保 Skill 在该触发的时候触发，不该触发的时候不触发。

测试内容：

- 明显相关的请求应触发
- 改写后的请求也应触发
- 无关请求不应触发

示例测试集：

```txt
Should trigger:
- "Help me set up a new ProjectHub workspace"
- "I need to create a project in ProjectHub"
- "Initialize a ProjectHub project for Q4 planning"

Should NOT trigger:
- "What's the weather in San Francisco?"
- "Help me write Python code"
- "Create a spreadsheet"
```

### 2. 功能测试

目标：验证 Skill 执行出的结果是否正确。

检查内容：

- 输出是否有效
- API 调用是否成功
- 错误处理是否可用
- 边界情况是否覆盖

示例：

```txt
Test: Create project with 5 tasks
Given: Project name "Q4 Planning", 5 task descriptions
When: Skill executes workflow
Then:
- Project created in ProjectHub
- 5 tasks created with correct properties
- All tasks linked to project
- No API errors
```

### 3. 性能对比

目标：验证 Skill 是否真的比不使用 Skill 更高效。

对比方式示例：

```txt
Without skill:
- User provides instructions each time
- 15 back-and-forth messages
- 3 failed API calls requiring retry
- 12,000 tokens consumed
```

```txt
With skill:
- Automatic workflow execution
- 2 clarifying questions only
- 0 failed API calls
- 6,000 tokens consumed
```

## 使用 `skill-creator` Skill

文档介绍说，`skill-creator` 可以帮助你：

- 从自然语言描述生成 Skill
- 生成格式正确的 `SKILL.md`
- 给出结构和触发词建议

它还能用于审查 Skill，例如：

- 指出常见问题
- 判断可能存在的过度触发或触发不足
- 基于 Skill 的用途建议测试用例

此外，它也适合做后续迭代：

- 当你在真实使用中遇到失败案例时，可以把问题和解决办法重新喂给 `skill-creator`，让它帮助改进 Skill

文档中的使用方式示例是：

“Use the skill-creator skill to help me build a skill for [your use case]”

并特别说明：

`skill-creator` 适合做设计和优化，但不替代自动测试或严格量化评估。

## 基于反馈进行迭代

文档把迭代信号分成三类。

### 触发不足的信号

现象：

- Skill 本该触发时没有触发
- 用户需要手动启用
- 用户经常问这个 Skill 何时使用

解决方法：

- 让 `description` 更具体
- 补充更多关键词，尤其是技术术语

### 过度触发的信号

现象：

- Skill 在不相关请求中被触发
- 用户会主动关闭它
- 对用途产生困惑

解决方法：

- 加入负向触发条件
- 把描述写得更具体，缩小范围

### 执行问题

现象：

- 结果不一致
- API 调用失败
- 用户经常需要纠正 Claude

解决方法：

- 改进说明文字
- 增加错误处理
- 把关键验证逻辑写得更明确

## 第 4 章：分发与分享

文档指出，Skill 能让一个 MCP 集成更完整。因为当用户比较不同连接器时，带有 Skill 的连接器更容易让用户快速获得实际价值。

## 当前的分发模式（2026 年 1 月）

### 个人用户如何获取 Skill

1. 下载 Skill 文件夹
2. 如有需要，将其压缩为 zip
3. 在 Claude.ai 的 Settings > Capabilities > Skills 中上传
4. 或将其放到 Claude Code 的 skills 目录

### 组织级 Skill

- 管理员可以在工作区范围内部署 Skill
- 支持自动更新
- 支持集中管理

## 一个开放标准

文档强调，Agent Skills 已被作为开放标准发布。

设计目标是：

- 与 MCP 一样，Skill 应尽量跨工具和平台复用
- 同一个 Skill 不应只在单一入口中有效

不过，如果某个 Skill 明显依赖某个平台能力，也可以在 `compatibility` 字段中注明。

## 通过 API 使用 Skill

对于程序化使用场景，API 提供了对 Skill 的管理和调用方式。

文档提到的能力包括：

- `/v1/skills` 接口可用于列出和管理 Skill
- 在 Messages API 请求中，可通过 `container.skills` 参数附带 Skill
- 可通过 Claude Console 做版本管理
- 可与 Agent SDK 结合使用

文档还给出一个“适用场景对照”：

- Claude.ai / Claude Code：适合最终用户直接交互，也适合人工测试与迭代
- API：适合应用、生产部署、自动化流水线和 Agent 系统

并说明：

在 API 中使用 Skill 目前需要 Code Execution Tool beta 支持。

## 当前推荐做法

文档建议当前最推荐的做法是：

1. 在 GitHub 上公开托管 Skill
2. 在仓库根目录写清楚 README
3. 提供示例用法和截图
4. 在 MCP 仓库文档中链接到 Skill
5. 解释为什么 MCP + Skill 的组合更有价值

文档还给了一个安装说明模板：

```txt
## Installing the [Your Service] skill

1. Download the skill:
- Clone repo: `git clone https://github.com/yourcompany/skills`
- Or download ZIP from Releases

2. Install in Claude:
- Open Claude.ai > Settings > skills
- Click "Upload skill"
- Select the skill folder (zipped)

3. Enable the skill:
- Toggle on the [Your Service] skill
- Ensure your MCP server is connected

4. Test:
- Ask Claude: "Set up a new project in [Your Service]"
```

## 如何定位你的 Skill

文档提醒，在 README、文档或介绍文案里描述 Skill 时，应强调“结果和收益”，而不是描述其底层格式。

好的表达：

“The ProjectHub skill enables teams to set up complete project workspaces in seconds - including pages, databases, and templates - instead of spending 30 minutes on manual setup.”

不好的表达：

“The ProjectHub skill is a folder containing YAML frontmatter and Markdown instructions that calls our MCP server tools.”

同时建议明确讲清：

“MCP 提供服务访问，Skill 提供工作流知识，两者结合才能形成真正可用的体验。”

## 第 5 章：模式与故障排查

## 选择你的方式：问题导向还是工具导向

文档用 Home Depot 做比喻：

- 有些人是带着问题去的，例如“我想修厨房橱柜”
- 有些人是拿着工具去问的，例如“我买了电钻，现在该怎么用它完成这件事”

Skill 也类似：

- Problem-first：用户描述目标，Skill 负责选择步骤和工具
- Tool-first：用户已经接上某个 MCP，Skill 负责提供最佳实践

文档指出，大多数 Skill 会偏向其中一边，而知道自己的定位有助于选择正确模式。

## 模式 1：顺序工作流编排

适用场景：用户需要按顺序执行的多步骤流程。

示例结构：

```txt
## Workflow: Onboard New Customer

## Step 1: Create Account
Call MCP tool: `create_customer`
Parameters: name, email, company

## Step 2: Setup Payment
Call MCP tool: `setup_payment_method`
Wait for: payment method verification

## Step 3: Create Subscription
Call MCP tool: `create_subscription`
Parameters: plan_id, customer_id (from Step 1)

## Step 4: Send Welcome Email
Call MCP tool: `send_email`
Template: welcome_email_template
```

关键技巧：

- 明确步骤顺序
- 写清步骤之间的依赖
- 每一步都做验证
- 对失败情况给出回滚或补救方法

## 模式 2：多 MCP 协同

适用场景：一个流程跨多个服务完成。

文档给出的示例是“设计到开发交接”：

### Phase 1: Design Export (Figma MCP)

1. 从 Figma 导出设计资源
2. 生成设计说明
3. 创建资源清单

### Phase 2: Asset Storage (Drive MCP)

1. 在 Drive 中创建项目文件夹
2. 上传资源
3. 生成分享链接

### Phase 3: Task Creation (Linear MCP)

1. 创建开发任务
2. 把资源链接附加到任务
3. 指派给工程团队

### Phase 4: Notification (Slack MCP)

1. 在 `#engineering` 频道发布交接摘要
2. 附上资源链接和任务引用

关键技巧：

- 用阶段划分流程
- 在服务之间传递数据
- 进入下一阶段前先做验证
- 做统一的错误处理

## 模式 3：迭代优化

适用场景：输出质量需要通过多轮修正逐步提升。

示例流程：

### Initial Draft

1. 通过 MCP 获取数据
2. 生成第一版报告
3. 保存到临时文件

### Quality Check

1. 运行验证脚本：`scripts/check_report.py`
2. 识别问题，例如：
   - 缺少章节
   - 格式不一致
   - 数据校验错误

### Refinement Loop

1. 逐一修复问题
2. 重新生成受影响部分
3. 再次验证
4. 重复，直到达到质量阈值

### Finalization

1. 应用最终格式
2. 生成摘要
3. 保存最终版本

关键技巧：

- 提前定义质量标准
- 采用显式迭代
- 使用验证脚本
- 明确何时停止迭代

## 模式 4：上下文感知的工具选择

适用场景：同一目标下，工具选择取决于上下文。

示例是文件存储：

### Decision Tree

1. 先检查文件类型和大小
2. 再决定最合适的存储位置：
   - 大文件（大于 10MB）：云存储 MCP
   - 协作文档：Notion 或 Docs MCP
   - 代码文件：GitHub MCP
   - 临时文件：本地存储

### Execute Storage

- 根据决策调用对应工具
- 附带该服务所需的元数据
- 生成访问链接

### Provide Context to User

- 告诉用户为什么选用了这种存储方式

关键技巧：

- 决策标准必须明确
- 要有回退方案
- 让用户知道选择原因

## 模式 5：领域特定智能

适用场景：Skill 不仅调用工具，还内嵌某个专业领域的判断规则。

文档给出的例子是支付合规：

### Before Processing (Compliance Check)

1. 获取交易详情
2. 应用合规规则：
   - 检查制裁名单
   - 检查司法辖区限制
   - 评估风险等级
3. 记录合规判断结果

### Processing

如果合规通过：

- 调用支付处理 MCP 工具
- 应用适当的欺诈检查
- 继续交易

如果合规不通过：

- 标记人工复核
- 创建合规案件

### Audit Trail

- 记录所有合规检查
- 记录处理决策
- 生成审计报告

关键技巧：

- 将领域知识写入流程逻辑
- 在动作发生前先做合规判断
- 保留完整记录
- 明确治理和决策边界

## 故障排查

## Skill 无法上传

### Error: "Could not find SKILL.md in uploaded folder"

原因：

- 文件名不是严格的 `SKILL.md`

解决方法：

- 重命名为 `SKILL.md`
- 用 `ls -la` 检查是否存在该文件

### Error: "Invalid frontmatter"

原因：

- YAML 格式错误

常见错误：

```txt
# Wrong - missing delimiters
name: my-skill
description: Does things
```

```txt
# Wrong - unclosed quotes
name: my-skill
description: "Does things
```

正确示例：

```txt
---
name: my-skill
description: Does things
---
```

### Error: "Invalid skill name"

原因：

- 名称包含空格或大写字母

```txt
# Wrong
name: My Cool Skill

# Correct
name: my-cool-skill
```

## Skill 不会触发

现象：

- Skill 完全不会自动加载

解决思路：

重新检查 `description`。

快速检查项：

- 是否过于泛泛，例如 “Helps with projects”
- 是否包含用户真实会说的触发词
- 是否在需要时写明文件类型

文档给出的调试方法：

直接问 Claude：

“When would you use the [skill name] skill?”

Claude 往往会复述这段 description，你可以据此判断描述哪里不够清楚。

## Skill 触发过于频繁

现象：

- 不相关请求也会触发

解决方法一：加入负向触发条件

```txt
description: Advanced data analysis for CSV files. Use for statistical modeling, regression, clustering. Do NOT use for simple data exploration (use data-viz skill instead).
```

解决方法二：把描述写得更具体

```txt
Too broad
description: Processes documents

More specific
description: Processes PDF legal documents for contract review
```

解决方法三：澄清作用范围

```txt
description: PayFlow payment processing for e-commerce. Use specifically for online payment workflows, not for general financial queries.
```

## MCP 连接问题

现象：

- Skill 已加载，但 MCP 调用失败

检查清单：

1. 确认 MCP Server 已连接
2. 确认 API key 有效
3. 脱离 Skill 单独测试 MCP
4. 确认 Skill 中引用的工具名正确，注意大小写

文档建议先验证 MCP 本身可用，再判断是不是 Skill 写法问题。

## 指令未被遵循

现象：

- Skill 已加载，但 Claude 没有按照说明执行

常见原因：

1. 指令太啰嗦
2. 关键指令埋得太深
3. 语言模糊
4. 过度依赖自然语言，而不是程序化校验

文档给的例子：

不好的写法：

```txt
Make sure to validate things properly
```

更好的写法：

```txt
CRITICAL: Before calling create_project, verify:
- Project name is non-empty
- At least one team member assigned
- Start date is not in the past
```

文档还提到一个进阶做法：

对于关键验证，如果可能，最好提供脚本来做验证。代码的确定性比自然语言更高。

### 关于“模型偷懒”

文档还给出一个经验做法：

```txt
## Performance Notes
- Take your time to do this thoroughly
- Quality is more important than speed
- Do not skip validation steps
```

并补充说明：

这类鼓励语放在用户提示里，通常比写在 `SKILL.md` 中更有效。

## 大上下文问题

现象：

- Skill 响应变慢
- 输出质量下降

可能原因：

- Skill 内容太长
- 同时启用了太多 Skill
- 没有利用渐进式披露，导致一次性加载过多内容

解决方法：

1. 缩短 `SKILL.md`
2. 详细说明移入 `references/`
3. 控制同时启用的 Skill 数量

文档建议：如果同时启用了 20 到 50 个 Skill，应考虑减少启用数量或按主题拆分管理。

## 第 6 章：资源与参考

文档建议新手优先参考：

- Best Practices Guide
- Skills Documentation
- API Reference
- MCP Documentation

还提到以下资源：

- 博客文章：Introducing Agent Skills、Equipping Agents for the Real World、Skills Explained、How to Create Skills for Claude、Building Skills for Claude Code、Improving Frontend Design through Skills
- 公共技能仓库：`anthropics/skills`

## `skill-creator` Skill

文档再次强调：

- `skill-creator` 能帮你从描述生成 Skill
- 也能帮你审查并指出常见问题
- 适合作为迭代辅助工具

例如可以这样使用：

- “Help me build a skill using skill-creator”
- “Review this skill and suggest improvements”

## 获取支持

技术问题可以去 Claude Developers Discord 社区论坛提问。

Bug 报告可以在 `anthropics/skills` 仓库的 GitHub Issues 中提交，并附上：

- Skill 名称
- 错误信息
- 复现步骤

## 附录 A：快速检查清单

这是一份上传前后的快速检查清单。

### 开始之前

- 已识别 2 到 3 个具体 use case
- 已识别所需工具
- 已查看本指南和示例 Skill
- 已规划文件夹结构

### 开发过程中

- 文件夹命名符合 kebab-case
- `SKILL.md` 文件存在且拼写正确
- YAML frontmatter 有 `---` 分隔符
- `name` 字段符合 kebab-case
- `description` 同时包含 what 和 when
- 没有 XML 尖括号
- 指令清晰且可执行
- 包含错误处理
- 提供了示例
- 明确引用了参考资料

### 上传之前

- 测试了明显应触发的请求
- 测试了改写表达
- 验证了无关请求不会触发
- 功能测试通过
- 工具集成可用
- 已压缩为 zip

### 上传之后

- 在真实对话中测试
- 观察是否存在触发不足或过度触发
- 收集用户反馈
- 迭代 description 和 instructions
- 更新 metadata 中的版本号

## 附录 B：YAML frontmatter

### 必填字段

```yaml
---
name: skill-name-in-kebab-case
description: What it does and when to use it. Include specific trigger phrases.
---
```

### 全部可选字段示例

```yaml
name: skill-name
description: [required description]
license: MIT
allowed-tools: "Bash(python:*) Bash(npm:*) WebFetch"
metadata:
  author: Company Name
  version: 1.0.0
  mcp-server: server-name
  category: productivity
  tags: [project-management, automation]
  documentation: https://example.com/docs
  support: support@example.com
```

### 安全说明

允许：

- 标准 YAML 类型
- 自定义 metadata
- 最长 1024 字符的 description

禁止：

- XML 尖括号
- 在 YAML 中嵌入恶意内容
- 使用保留前缀命名

## 附录 C：完整 Skill 示例

文档建议，如果想看更完整的生产级示例，可以参考：

- Document Skills
- Example Skills
- Partner Skills Directory

这些示例可用于：

- 学习结构
- 参考触发词设计
- 借鉴错误处理和 references 拆分方式
- 作为模板做二次修改

