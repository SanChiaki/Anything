# The Anatomy of an Agent Harness

> A deep dive into what Anthropic, OpenAI, Perplexity and LangChain are actually building.
> Covering the orchestration loop, tools, memory, context management, and everything else that transforms a stateless LLM into a capable agent.
>
> Author: Akshay Pachaar (@akshay_pachaar)
> Date: April 6, 2026

## 核心观点

你搭建了一个聊天机器人。也许你接了一个 ReAct 循环和几个工具。演示没问题。但当你尝试构建生产级应用时，问题就来了：模型忘记了三步前做了什么、工具调用静默失败、上下文窗口被垃圾信息填满。

**问题不在模型本身，而在模型周围的一切。**

LangChain 证明了这一点：他们只更换了基础设施（同样的模型、同样的权重），TerminalBench 2.0 排名就从 30 名开外飙升到第 5 名。

这种基础设施现在有一个名字：**Agent Harness（智能体框架）**。

---

## 什么是 Agent Harness？

Harness 是围绕 LLM 的完整软件基础设施，包括：
- 编排循环（Orchestration Loop）
- 工具系统（Tools）
- 内存管理（Memory）
- 上下文管理（Context Management）
- 状态持久化（State Persistence）
- 错误处理（Error Handling）
- 安全护栏（Guardrails）

Anthropic 的 Claude Code 文档说得很直白：SDK 就是"为 Claude Code 供电的 agent harness"。OpenAI 的 Codex 团队也把"agent"和"harness"等同起来，指代让 LLM 有用的非模型基础设施。

> "If you\'re not the model, you\'re the harness." — LangChain 的 Vivek Trivedy

Beren Millidge 在 2023 年的文章中做了精准类比：
- 原始 LLM = 没有 RAM、磁盘和 I/O 的 CPU
- 上下文窗口 = RAM（快但有限）
- 外部数据库 = 磁盘存储（大但慢）
- 工具集成 = 设备驱动
- **Harness = 操作系统**

---

## 三个工程层级

围绕模型有三个同心层级的工程：

1. **Prompt Engineering（提示工程）** — 构建模型接收到的指令
2. **Context Engineering（上下文工程）** — 管理模型看到什么、何时看到
3. **Harness Engineering（框架工程）** — 包含以上两者，加上整个应用基础设施

Harness 不是提示的包装器，而是使自主智能体行为成为可能的完整系统。

---

## 生产级 Harness 的 12 个组件

### 1. 编排循环（The Orchestration Loop）
这是心跳。实现 Thought-Action-Observation (TAO) 循环，也叫 ReAct 循环。

Anthropic 将其描述为"dumb loop"——所有智能都在模型中，循环只管理回合。

### 2. 工具（Tools）
工具是智能体的手。Claude Code 提供六大类工具：文件操作、搜索、执行、网页访问、代码智能和子智能体生成。OpenAI 的 Agents SDK 支持函数工具、托管工具和 MCP 服务器工具。

### 3. 内存（Memory）
多时间尺度的内存系统：
- **短期** — 单会话对话历史
- **长期** — 跨会话持久化

Claude Code 的三层内存架构：
- 轻量级索引（每条约 150 字符，始终加载）
- 详细主题文件（按需拉取）
- 原始转录文件（仅通过搜索访问）

### 4. 上下文管理（Context Management）
这是许多智能体静默失败的地方。

**上下文腐烂（Context Rot）**：当关键内容落在窗口中间位置时，模型性能下降 30%+。

生产级策略：
- **压缩（Compaction）** — 接近限制时总结对话历史
- **观察遮蔽（Observation Masking）** — JetBrains 的 Junie 隐藏旧工具输出
- **即时检索（Just-in-time Retrieval）** — 维护轻量级标识符，动态加载数据
- **子智能体委托（Sub-agent Delegation）** — 每个子智能体广泛探索，只返回 1000-2000 token 的精简摘要

### 5. 提示构建（Prompt Construction）
层级组装：系统提示 > 工具定义 > 内存文件 > 对话历史 > 当前用户消息。

### 6. 输出解析（Output Parsing）
现代 Harness 依赖原生工具调用，模型返回结构化的 `tool_calls` 对象。

### 7. 状态管理（State Management）
- LangGraph — 类型化字典流经图节点，支持检查和回滚
- OpenAI — 四种互斥策略
- Claude Code — git commits 作为检查点

### 8. 错误处理（Error Handling）
10 步流程、每步 99% 成功率，端到端成功率只有 ~90.4%。

LangGraph 区分四种错误类型：
- 瞬态错误（重试）
- LLM 可恢复错误（返回给模型调整）
- 用户可修复错误（中断等待人工输入）
- 意外错误（上报调试）

### 9. 护栏与安全（Guardrails and Safety）
OpenAI 的 SDK 实现三个级别：输入护栏、输出护栏、工具护栏。

Anthropic 架构上分离权限执行和模型推理。Claude Code 对约 40 个离散工具能力进行独立门控。

### 10. 验证循环（Verification Loops）
Anthropic 推荐三种方法：
- 基于规则的反馈（测试、linter、类型检查器）
- 视觉反馈（Playwright 截图）
- LLM-as-judge（独立子智能体评估输出）

Boris Cherny（Claude Code 创始人）指出：给模型验证工作的能力可使质量提升 2-3 倍。

### 11. 子智能体编排（Subagent Orchestration）
Claude Code 支持三种执行模型：
- Fork（父上下文的字节级复制）
- Teammate（独立终端窗格 + 文件邮箱通信）
- Worktree（独立 git worktree）

### 12. Ralph Loop（长期任务模式）
Anthropic 开发的两阶段模式：
- **初始化智能体** — 设置环境、进度文件、初始 git commit
- **编码智能体** — 每次会话读取 git 日志和进度文件，处理最高优先级任务

---

## 真实框架的实现对比

| 框架 | 核心理念 |
|------|---------|
| Anthropic Claude SDK | "dumb loop"，智能全在模型中 |
| OpenAI Agents SDK | "code-first"，用 Python 表达工作流 |
| LangGraph | 显式状态图，节点+条件边 |
| CrewAI | 角色基础的多智能体架构 |
| AutoGen | 对话驱动编排 |

---

## 定义每个 Harness 的 7 个决策

1. **单智能体 vs 多智能体** — 先最大化单个智能体，拆分仅在工具超过 10 个或任务领域明显分离时进行
2. **ReAct vs Plan-and-Execute** — ReAct 灵活但成本高，LLMCompiler 报告 3.6x 加速
3. **上下文窗口管理策略** — 五种生产方法
4. **验证循环设计** — 计算验证 vs 推理验证
5. **权限和安全架构** — 宽松 vs 严格
6. **工具范围策略** — 工具越少越好，Vercel 移除 80% 工具后效果更好
7. **Harness 厚度** — Anthropic 倾向于薄 Harness，模型能力增强后 Harness 应变薄

---

## 关键结论

> The harness is not a solved problem or a commodity layer. It\'s where the hard engineering lives.
> Harness 不是已解决的问题或商品化层，而是硬工程所在之处。

> The next time your agent fails, don\'t blame the model. Look at the harness.
> 下次你的智能体失败时，不要怪模型。看看 Harness。

---

原文链接: https://x.com/akshay_pachaar/status/2041146899319971922
