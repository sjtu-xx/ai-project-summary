# OpenViking 项目关键分析

## 一句话定位
OpenViking 是一个面向 AI Agent 的“上下文数据库平台”，核心不是单纯向量检索，而是把资源、记忆、技能统一到文件系统范式下，做完整的上下文生命周期管理。

## 它做了什么
1. 统一上下文类型
- Resource（资源）: 用户导入知识（文档/网页/代码仓库）
- Memory（记忆）: 会话中自动提取的长期记忆（8类）
- Skill（技能）: Agent 可调用能力及其定义

2. 统一上下文表示
- 使用 `viking://` 虚拟 URI 组织上下文树
- 使用 L0/L1/L2 三层内容模型:
  - L0: `.abstract.md`（短摘要）
  - L1: `.overview.md`（结构化概览）
  - L2: 原始内容/文件（按需读取）

3. 统一访问方式
- Python SDK（本地嵌入）
- HTTP Server（独立服务）
- Rust CLI（`ov`）
- Console UI 与 Vikingbot

## 怎么做的（架构）

### 1) 分层架构
Client -> Service -> Storage

- Service 层解耦业务与传输，核心服务包括:
  - FSService（文件系统操作）
  - SearchService（检索）
  - ResourceService（资源导入）
  - SessionService（会话与记忆提交）
  - Relation/Pack/Debug 等

### 2) 双层存储
- AGFS/RAGFS: 保存内容（文件、目录、L0/L1/L2、关系）
- Vector Index: 保存向量与元数据（引用 URI，不存正文）

关键思想: 文件系统是事实源，向量索引是衍生索引。

### 3) 多语言职责分工
- Python: 业务编排、检索策略、会话记忆流程、HTTP API
- Rust: AGFS/RAGFS 文件系统实现与插件系统（含 Python 绑定）
- C++: 本地向量索引引擎（SIMD 变体 + 持久化）

## 三条核心链路

### A. 资源导入链路（add_resource）
1. `ResourceService.add_resource`
2. `ResourceProcessor.process_resource`
3. `UnifiedResourceProcessor` 做 source 访问（本地/URL/Git）+ parser 自动路由
4. Parser 输出到临时目录
5. `TreeBuilder.finalize_from_temp` 确定最终 URI 与目录树
6. 入 SemanticQueue / EmbeddingQueue 异步生成 L0/L1 + 向量化

特点:
- 写入与语义处理解耦（异步队列）
- 大目录和多模态导入更稳

### B. 检索链路（find/search）
- `find`: 无 session 上下文的语义检索
- `search`: 带 session 上下文，先做意图分析

核心过程:
1. IntentAnalyzer（可选）将请求改写成多个 TypedQuery
2. HierarchicalRetriever 全局召回 + 目录递归搜索
3. rerank（如配置）
4. 热度因子（active_count/更新时间）做最终分数融合

特点:
- 不只是“向量 topk”，而是“目录递归 + 重排 + 热度”
- 与 `viking://` 树结构深度耦合

### C. 会话提交链路（session.commit）
1. Phase 1：归档当前消息到 `history/archive_N`
2. Phase 2（后台任务）：
   - 生成归档摘要（L0/L1）
   - 提取长期记忆（8类）
   - 更新关系与活跃度
   - 语义/向量任务入队

特点:
- 前台快速返回 task_id
- 后台完成记忆提取与索引刷新
- 会话可持续“自迭代”

## 队列与可靠性设计

### 队列系统
- QueueManager 统一管理 Embedding/Semantic 队列
- 支持并发 worker、重试、失败重入、熔断保护

### 事务与一致性
- Path Lock（点锁/子树锁/mv 锁）保护关键写路径
- RedoLog 用于关键流程崩溃恢复（尤其 session memory）

设计原则:
- 宁可暂时少召回，也不要返回脏结果

## 多租户与隔离模型
- 三层身份边界: account / user / agent
- URI 规范化 + 访问控制由 `namespace + request context` 统一治理
- 资源可在 account 内共享，记忆/会话按 user/agent 隔离

## 项目工程特征
- 体量较大（约 1897 文件）
- 测试覆盖目录广（server/session/storage/retrieve/parse 等）
- 不是 demo，而是平台级工程（SDK + Server + CLI + Bot + Console）

## 适合场景
- 需要长期记忆与多轮会话演进的 Agent
- 需要把文档/仓库/技能统一成可管理上下文系统
- 需要服务化部署与多租户隔离的团队

## 风险与注意点
1. 系统复杂度高，接入前需明确数据流与运维边界
2. 异步队列路径较多，需做好观测与告警
3. 模型依赖（embedding/vlm/rerank）对稳定性和成本有直接影响

## 结论
OpenViking 的核心价值在于把“RAG 检索工具”升级为“上下文操作系统”：
- 用文件系统范式统一上下文
- 用 L0/L1/L2 控制成本与效果
- 用会话提取驱动记忆自增长
- 用锁与重做机制保障一致性

这使它更适合中长期运行、需要可维护上下文资产的 Agent 系统。
