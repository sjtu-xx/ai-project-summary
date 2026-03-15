# Skill Creator 技能详解

来源：Anthropic 官方仓库 [anthropics/skills](https://github.com/anthropics/skills) 中的 `skills/skill-creator/`。  
技能路径（本地参考）：`project/github-project/skills/skills/skill-creator`。

## 一、定位与触发

**skill-creator** 是用于**创建、修改、优化技能并测量其表现**的元技能。

**适用场景：**
- 从零创建新技能
- 编辑或优化已有技能
- 跑评测（evals）验证技能
- 做基准测试与方差分析
- 优化技能的 description，提高「何时该触发」的准确率

**触发描述（frontmatter）：**  
Create new skills, modify and improve existing skills, and measure skill performance. Use when users want to create a skill from scratch, edit, or optimize an existing skill, run evals to test a skill, benchmark skill performance with variance analysis, or optimize a skill's description for better triggering accuracy.

---

## 二、核心流程概览

整体是一个「起草 → 测试 → 评审 → 改进」的循环：

1. **明确意图**：技能要做什么、在什么场景触发、输出长什么样、要不要写测试用例。
2. **访谈与调研**：问清边界、输入输出、示例、成功标准；必要时查 MCP/文档、参考同类技能。
3. **写 SKILL.md**：填 name、description（含触发场景）、compatibility（可选），以及正文说明。
4. **设计测试用例**：2–3 条真实用户风格的 prompt，存到 `evals/evals.json`。
5. **跑测试**：对每条用例同时跑「带技能」和「基线」（无技能或旧版技能）的子任务。
6. **打分与汇总**：用 grader 或脚本对每条 run 打分 → 用 `aggregate_benchmark` 生成 benchmark → 用 `generate_review.py` 生成评测查看器。
7. **用户评审**：用户在浏览器里看输出和 Benchmark，填反馈，提交得到 `feedback.json`。
8. **根据反馈改技能**：读 `feedback.json`，改 SKILL.md（或脚本/参考文档），再进入下一轮迭代。
9. **（可选）Description 优化**：用 `run_loop` 等脚本，用「应触发 / 不应触发」的查询优化 frontmatter 里的 description。
10. **（可选）打包**：用 `package_skill` 打成 `.skill` 文件，方便安装。

技能强调：先搞清楚用户卡在流程的哪一步，再针对性推进；若用户不想跑大批评测，也可以只做轻量迭代。

---

## 三、技能写作规范（SKILL.md）

- **结构**：YAML frontmatter（必含 `name`、`description`）+ Markdown 正文。
- **description**：既要写「做什么」，也要写「什么时候用」；可以略偏「主动」，以减少 undertrigger。
- **渐进披露**：
  - 元数据（name + description）始终在上下文；
  - SKILL.md 正文在触发时加载（建议 <500 行）；
  - `scripts/`、`references/`、`assets/` 等按需引用。
- **原则**：无恶意、不误导；多用「为什么」解释要求，少用生硬的 MUST；输出格式用模板+示例写清楚。

### 目录结构示意

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description required)
│   └── Markdown instructions
└── Bundled Resources (optional)
    ├── scripts/    - 确定性/重复性任务的可执行代码
    ├── references/ - 按需加载的文档
    └── assets/     - 输出用到的文件（模板、图标、字体等）
```

---

## 四、测试与评测体系

### 4.1 目录约定

- 结果放在 `<skill-name>-workspace/`，与技能目录同级。
- 按迭代组织：`iteration-1/`、`iteration-2/`…
- 每个测试用例一个目录：`eval-0/`、`eval-1/`…，内含 `with_skill/`、`without_skill/`（或 `old_skill/`）及各自的 outputs。

### 4.2 运行策略

- **同一轮里**：所有用例的「带技能」和「基线」run 在同一轮一起发起，便于并行和对比。
- **基线定义**：新建技能时基线 = 无技能；改进已有技能时基线 = 旧版（可先 `cp` 到 workspace 的 snapshot）。

### 4.3 断言与打分

- 先在 `evals/evals.json` 里写 prompt 和 `expected_output`；run 进行中再补**可客观验证的断言**。
- 每条 run 用 `agents/grader.md` 或脚本根据 transcript + outputs 打分，结果写到各 run 目录的 `grading.json`（字段需含 `text`、`passed`、`evidence`，供 viewer 使用）。

### 4.4 汇总与查看

- 使用：
  ```bash
  python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
  ```
  生成 `benchmark.json` / `benchmark.md`（通过率、时间、token、均值±标准差、与基线的 delta）。
- 用 `eval-viewer/generate_review.py` 生成评测页面：**Outputs** 标签看每条用例的输出与反馈，**Benchmark** 标签看量化对比。
- 无浏览器环境时用 `--static <output_path>` 生成静态 HTML；用户通过「Submit All Reviews」下载 `feedback.json`，再放回 workspace 供下一轮使用。

### evals.json 结构示例

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User's task prompt",
      "expected_output": "Description of expected result",
      "files": []
    }
  ]
}
```

完整 schema 见技能内 `references/schemas.md`。

---

## 五、改进技能时的思路

1. **从反馈泛化**：避免只针对少数用例的过拟合，用不同说法、工作模式尝试更通用的改进。
2. **精简 prompt**：删掉无效、冗余的说明；结合 transcript 看是否在无用步骤上浪费时间。
3. **解释原因**：多写「为什么」要这样做，让模型理解意图，而不是堆砌 ALWAYS/NEVER。
4. **复用重复劳动**：若多个用例里都出现同类脚本或流程，考虑把脚本放进 `scripts/`，在技能里统一引用。

---

## 六、Description 优化（触发准确率）

- **目的**：让 frontmatter 的 `description` 更准确控制「该触发 / 不该触发」。
- **步骤概要**：
  1. 生成约 20 条 **trigger eval** 查询（8–10 条应触发、8–10 条不应触发），要真实、具体、含边界情况。
  2. 用 `assets/eval_review.html` 让用户审阅、编辑、导出 `eval_set.json`。
  3. 后台运行：
     ```bash
     python -m scripts.run_loop --eval-set <path> --skill-path <path> --model <model-id> --max-iterations 5 --verbose
     ```
     脚本会做 train/test 划分、多次触发率评估、调用 Claude 改 description、迭代多轮，最后给出 `best_description`。
  4. 将 `best_description` 写回 SKILL.md frontmatter。

**注意**：触发机制是「只有模型自己搞不定或需要专门能力时才查技能」，所以 eval 查询要足够「有难度、有场景」，太简单的单步请求不适合用来测触发。

---

## 七、高级：盲测对比

- 当需要严格对比两个技能版本（例如「新版本是否真的更好」）时，可用 **comparator**：把两个输出匿名交给另一个 agent 打分。
- 细节见技能内 `agents/comparator.md` 和 `agents/analyzer.md`，通常需要 subagent，多数用户以人工评审为主即可。

---

## 八、配套脚本与文件

| 路径 | 作用 |
|------|------|
| `scripts/aggregate_benchmark.py` | 汇总各 run 的 grading，生成 benchmark 统计 |
| `scripts/run_loop.py` | Description 优化主循环（eval → 改描述 → 再测） |
| `scripts/run_eval.py` | 单次 trigger eval |
| `scripts/improve_description.py` | 描述改进逻辑 |
| `scripts/package_skill.py` | 打包技能为 `.skill` |
| `scripts/quick_validate.py`、`generate_report.py` | 快速校验与报告 |
| `eval-viewer/generate_review.py` | 生成评测查看器（含 Outputs + Benchmark） |
| `references/schemas.md` | evals.json、grading.json、benchmark 等 JSON 结构说明 |
| `agents/grader.md` | 打分子任务的指令 |
| `agents/comparator.md` | 盲测对比指令 |
| `agents/analyzer.md` | 分析 benchmark、方差、胜出原因等 |
| `assets/eval_review.html` | 用户审阅 trigger eval 集的 HTML 模板 |

---

## 九、环境差异

- **Claude Code（有 subagent）**：可以并行跑 with-skill / baseline、用 grader/comparator、起 viewer 服务、跑 description 优化。
- **Claude.ai（无 subagent）**：单线程按用例执行、无基线对比、可不用浏览器而直接在对话里展示结果并收反馈；description 优化依赖 `claude` CLI，没有则跳过。
- **Cowork**：有 subagent，但无显示器时用 `--static` 生成静态 HTML；反馈通过下载 `feedback.json` 再放回 workspace；强调**先跑完测试就生成 eval viewer**，再根据人工反馈改技能。

---

## 十、与仓库其他内容的关系

- 在 anthropic-skills-repo 的「开发与技术类」里，skill-creator 和 `webapp-testing`、`mcp-builder` 一样，核心价值是**把一套稳定流程固化下来**：先判断任务分支、再决定读哪些文件/跑哪些脚本、最后按既定流程验证结果。
- 适合在学完《Building Skills for Claude》和仓库其他示例后，用 skill-creator 系统化地创建和迭代自己的技能；也可直接参考其 SKILL.md 和脚本学习「如何评测与优化技能」。

---

## 一句话总结

**skill-creator** 把「技能创作」拆成可重复的流程：意图捕获 → 写 SKILL.md → 设计 evals → 并行跑测试与基线 → 打分与 benchmark → 人工评审 → 根据 feedback 改进 → 可选 description 优化与打包。它既规定目录结构、JSON schema 和脚本用法，也强调沟通（根据用户是否熟悉术语调整表述）、可验证的断言和从反馈中泛化，适合系统化地创建和迭代 Claude/Cursor 技能。
