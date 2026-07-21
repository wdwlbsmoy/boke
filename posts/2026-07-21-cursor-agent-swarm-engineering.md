---
title: "从 Cursor Agent Swarm 看多智能体工程化落地"
date: 2026-07-21
tags: [AI, Multi-Agent, Cursor, Agent Swarm, 软件工程, 协调机制, 分布式系统]
---

# 从 Cursor Agent Swarm 看多智能体工程化落地

## 引言：数百个 Agent 同时写一个项目，行得通吗？

2026 年初，Cursor 团队发布了一篇让整个 AI 工程界炸锅的博文：《Scaling long-running autonomous coding》。他们描述了一个疯狂的实验——**数百个 AI Agent 并行运行数周，在同一个代码库中协作完成大型项目**。

成果包括：
- **FastRender**：一个从零开始写的 Web 浏览器引擎，超过 100 万行 Rust 代码，几周内完成
- **Java LSP**：7,400 个 commit，55 万行代码
- **Windows 7 模拟器**：14,600 个 commit，120 万行代码
- **Excel 复刻**：12,000 个 commit，160 万行代码

这不是 Demo，不是论文里的玩具实验——这是 Cursor 团队用自家的 Agent 基础设施跑出的真实工程结果。FastRender 的代码分析显示，它相当于 **约 110 人年的 Rust 开发量**，被压缩到了一周内完成。

但比最终产出更有价值的是他们**一路踩过的坑**。这些教训，恰好是多智能体系统从"看起来能跑"到"真正工程化"的关键路径。

## 一、第一版方案为什么失败：平等协作是幻觉

### 1.1 初始设计：民主式协调

Cursor 团队最初的直觉和大多数人一样——让 Agent 们平等协作。

```
设计思路：
- 所有 Agent 地位平等
- 通过共享文件协调状态
- 每个 Agent 检查别人在做什么，认领任务，更新状态
- 用锁机制防止冲突
```

这看起来很"分布式系统"，很优雅。但在实践中彻底崩溃了。

### 1.2 三个致命问题

**问题一：锁成为瓶颈**

20 个 Agent 同时运行，有效吞吐量降到了 2-3 个的水平。大部分时间都在等锁。Agent 会忘记释放锁、持锁时间过长、甚至在持锁期间崩溃。

**问题二：乐观并发控制也救不了**

切换到乐观并发（读取自由、写入时检查状态是否变化）后，锁的问题解决了。但暴露了更深层的问题——

**问题三：没有层级，Agent 变得保守**

> *"With no hierarchy, agents became risk-averse. They avoided difficult tasks and made small, safe changes instead. No agent took responsibility for hard problems or end-to-end implementation."*

没有人负责全局，每个 Agent 都选择"安全"的小任务。系统长时间"在忙"但没有实质进展。这个现象在人类团队中也存在——没有明确 owner 的项目，没人啃硬骨头。

**核心教训：Multi-Agent 系统中，"去中心化协作"几乎从来不是正确答案。**

## 二、有效方案：Planner-Worker 分层架构

### 2.1 关键转折

Cursor 团队的解决方案出人意料地简单——**引入明确的角色分层**：

```
┌─────────────────────────────────────────┐
│           Planner (规划者)               │
│  - 持续探索代码库                         │
│  - 创建具体任务                          │
│  - 可以生成子 Planner（递归规划）          │
└──────────────────┬──────────────────────┘
                   │ 分发任务
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌────────┐  ┌────────┐    ┌────────┐
│Worker 1│  │Worker 2│ .. │Worker N│
│专注执行│  │专注执行│    │专注执行│
│push代码│  │push代码│    │push代码│
└────────┘  └────────┘    └────────┘
                   │
                   ▼
           ┌─────────────┐
           │ Judge (裁判) │
           │ 决定是否继续  │
           └─────────────┘
```

### 2.2 为什么这个结构有效？

**Worker 不需要协调。** 每个 Worker 只关心自己领到的任务，不和其他 Worker 通信，不关心全局。做完就 push，进入下一个任务。

**Planner 负责全局视野。** Planner 持续扫描代码库状态，根据当前进展动态创建任务。规划本身也是并行的——可以为不同模块生成子 Planner。

**周期性重新评估。** 每个迭代结束后，Judge Agent 判断是否继续。下一个迭代从 fresh state 开始，避免漂移和隧道视野。

### 2.3 对比传统多智能体模式

| 维度 | 平等协作（失败） | Planner-Worker（成功） |
|------|-----------------|---------------------|
| 协调方式 | Agent 间自由通信 | 严格层级，Hub-and-Spoke |
| 任务分配 | 自行认领 | Planner 分发 |
| 状态共享 | 共享文件 + 锁 | 代码库本身（git push） |
| 扩展性 | 20 Agent → 有效 2-3 | 数百 Agent → 线性扩展 |
| 困难任务 | 无人负责 | Planner 明确分配 |

## 三、工程化关键发现：简单胜过复杂

Cursor 团队分享的几个反直觉发现，对所有构建多智能体系统的团队都有参考价值：

### 3.1 删掉"质量控制"角色反而更好

> *"We initially built an integrator role for quality control and conflict resolution, but found it created more bottlenecks than it solved."*

他们最初设计了一个"集成者"角色来做质量把关和冲突解决。结果发现，Worker 自己就能处理冲突，多加一层反而引入了瓶颈。

**启示**：不要过度设计 Agent 角色。如果一个角色的价值不明确，先去掉它。

### 3.2 模型选择因角色而异

不同角色需要不同模型的特质：

- **Planner**：需要全局理解和长程规划能力 → GPT-5.2 胜过 Codex
- **Worker**：需要精确执行和持续专注 → GPT-5.2 的延续性更好
- **短任务**：Opus 4.5 倾向于快速交付（"take shortcuts when convenient"）

**启示**：不要用一个模型打天下。强推理模型做规划，快速模型做执行。

### 3.3 Prompt 比架构更重要

> *"A surprising amount of the system's behavior comes down to how we prompt the agents."*

让 Agent 协调好、避免病态行为、在长时间运行中保持专注——这些更多取决于 prompt 工程，而非系统架构。

**启示**：在优化架构之前，先花时间优化 prompt。Agent 的"性格"由 prompt 塑造。

### 3.4 "刚好够"的结构

> *"Too little structure and agents conflict, duplicate work, and drift. Too much structure creates fragility."*

```
结构不足 ──────────── 最佳点 ──────────── 结构过多
冲突、重复、漂移      高效协作         脆弱、瓶颈、僵化
```

## 四、从 Cursor Swarm 到你的项目：实操落地指南

### 4.1 Git Worktree：Agent 隔离的工程基础

Cursor 的并行 Agent（用户可用版本）采用 **Git Worktree** 实现隔离——每个 Agent 在独立的工作目录中操作同一个仓库的不同副本：

```bash
# 为每个 Agent 创建独立工作树
git worktree add ../agent-1-workspace feature/auth
git worktree add ../agent-2-workspace feature/tests
git worktree add ../agent-3-workspace feature/docs

# Agent 各自在自己的目录中工作，互不干扰
# 完成后各自 push 自己的分支
```

核心约束：**任务必须独立。** 如果 Agent B 需要 Agent A 的输出，就不能并行。

### 4.2 适用场景 vs 不适用场景

**✅ 适合 Agent Swarm 的场景：**
- 大规模重构（不同模块独立修改）
- 并行写测试（每个模块的测试互不依赖）
- 多维度分析（安全审查 + 性能审查 + 风格审查）
- 批量迁移（100 个文件的 API 升级）

**❌ 不适合 Agent Swarm 的场景：**
- 紧密耦合的功能开发（前后端联调）
- 需要频繁同步状态的工作流
- 创意性/探索性工作（需要人在循环中）
- 小任务（协调开销 > 并行收益）

### 4.3 成本控制的现实考量

Cursor 的博文坦率提到"trillions of tokens"的消耗。在实际项目中：

```python
# Agent 数量 vs 成本的关系
cost_model = {
    "1_agent":  {"tokens/hour": "50K",   "cost/hour": "$0.5"},
    "3_agents": {"tokens/hour": "150K",  "cost/hour": "$1.5"},
    "8_agents": {"tokens/hour": "400K",  "cost/hour": "$4.0"},
    "100_agents": {"tokens/hour": "5M",  "cost/hour": "$50"},
}

# 判断是否值得并行
def should_parallelize(task_hours_sequential, agent_count, hourly_cost):
    parallel_hours = task_hours_sequential / agent_count * 1.3  # 30% 协调开销
    parallel_cost = parallel_hours * hourly_cost * agent_count
    sequential_cost = task_hours_sequential * hourly_cost
    return parallel_cost < sequential_cost * 2  # 2x 成本以内算值得
```

关键原则：**并行不是免费的。只有当时间压缩的价值大于额外成本时才值得。**

## 五、更大的图景：Agent Swarm 的行业趋势

Cursor 的实验不是孤立事件。2026 年，"Agent Swarm" 已经从研究概念走向工程实践：

| 项目 | 模式 | 规模 | 产出 |
|------|------|------|------|
| Cursor FastRender | Planner-Worker | 数百并行 | 100万行浏览器代码 |
| Cursor GPU Kernels | Multi-Agent | 多 Agent 协作 | GPU kernel 性能提升 38% |
| Cursor Solid→React | Planner-Worker | 3周持续运行 | +266K/-193K 行迁移 |
| SwarmStation（社区） | 分布式 Agent | 10+ 并行 | 每晚生产 10 个 PR |

但也要看到局限：

- FastRender 是 100 万行代码，但**不是生产级浏览器**
- 生成速度快不等于质量高——仍需人类审查
- Token 成本在企业规模下可能不可接受
- 模型能力仍是上限——Agent 做不了模型做不了的事

## 结语：正确的启示

Cursor Agent Swarm 实验给我们的最大启示不是"AI 能写 100 万行代码"——而是**多智能体协作的工程化，本质上是分布式系统设计问题**，遵循相同的规律：

1. **去中心化几乎总是错误的**——需要明确的层级和 ownership
2. **简单结构优于复杂结构**——Planner + Worker 就够了，不需要更多角色
3. **隔离优于共享**——Git Worktree > 共享文件 + 锁
4. **周期性重置**——防止漂移比防止冲突更重要
5. **Prompt > Architecture**——Agent 的行为更多由 prompt 决定

当你下次设计多智能体系统时，不要从"我需要多少种角色"开始，而是从"最简单的有效结构是什么"开始。答案大概率是：**一个好的 Planner + N 个专注的 Worker + 明确的完成标准。**

仅此而已。

---

*参考资料：*
- *Cursor Blog: "Scaling long-running autonomous coding"*
- *Simon Willison: "FastRender: a browser built by thousands of parallel agents"*
- *LinkedIn Analysis: "We analyzed the code of Cursor's AI-built browser FastRender"*
- *Medium: "Parallel AI Agents in Cursor 2.0: A Practical Guide"*
- *Cursor Blog: "Speeding up GPU kernels by 38% with a multi-agent system"*
