---
title: "Hermes Agent v0.18.2 Kanban Swarm 深度解析：多智能体协作的工程实现"
date: 2026-07-12
tags: [hermes-agent, multi-agent, kanban, swarm, AI-engineering]
author: Hermes Blog
---

# Hermes Agent v0.18.2 Kanban Swarm 深度解析：多智能体协作的工程实现

凌晨三点，你的 CI 跑完了。四个 AI agent 各自写了一段调研报告，一个 reviewer 验证了数据准确性，最后一个 writer 把所有素材合成了一篇技术文章。整个过程没有人工干预，没有外部编排服务，只有一个 SQLite 文件和几条 shell 命令。

这不是假设。你正在读的这篇文章，就是 Hermes Agent Kanban Swarm 协作出来的产物。

## 为什么多 agent 协作这么难

多 agent 系统的工程难点从来不是"让 LLM 干活"，而是协调。谁先跑？谁等谁？一个 agent 挂了怎么办？中间结果怎么传？

常见方案是加一层编排服务——Temporal、Airflow、或者自己糊一个 Redis queue。但这带来了新问题：你得运维这套基础设施，得处理服务间通信的 failure mode，得为每个 workflow 写 DAG 定义文件。

Hermes 的做法不一样。它把多 agent 协作压缩进了已有的 Kanban 内核，用 SQLite 的事务保证做协调，用 parent-child 依赖做 DAG，用 dispatcher tick 做调度。没有新服务，没有新端口，没有新 failure mode。

这个设计有代价——SQLite 是单写的，不适合跨机器分布式场景。但对于单机或小团队的 agent 协作（2-10 个 worker 并行），它够用，而且简单到不太会出错。

## Swarm 拓扑：四层结构

一个 Swarm 的结构长这样：

```
root (立即完成，充当 blackboard)
├── worker-a (ready, 并行)
├── worker-b (ready, 并行)
├── worker-c (ready, 并行)
└── verifier (todo, 等所有 worker 完成)
     └── synthesizer (todo, 等 verifier 通过)
```

Root 节点创建后立刻标记为 done。它的作用不是执行工作，而是当公告板——所有 worker 通过 `task_comments` 表往上面写结构化 JSON，下游节点读取这些数据。

依赖管理靠 `task_links` 表。Verifier 的 parent 是所有 worker，所以 dispatcher 的 `recompute_ready()` 会在最后一个 worker 完成时把 verifier 从 todo 提升到 ready。Synthesizer 同理，等 verifier。

这套依赖门控不需要额外代码——它复用了 Kanban 内核里已有的 parent-child 机制。Swarm 只是在上面加了一层约定。

## 5 分钟上手：CLI 创建一个 Swarm

```bash
hermes kanban swarm \
  "收集竞品分析数据并生成决策备忘录" \
  --worker "researcher-a:Market scan" \
  --worker "researcher-b:Customer interviews" \
  --verifier reviewer \
  --synthesizer writer
```

这条命令做了什么：

1. 创建 root task，立即完成
2. 创建两个 worker task，assignee 分别是 researcher-a 和 researcher-b，status 为 ready
3. 创建 verifier task，parents 指向两个 worker，status 为 todo
4. 创建 synthesizer task，parent 指向 verifier，status 为 todo
5. 在 root 上写一条 blackboard comment，记录完整拓扑信息

Dispatcher 下一个 tick（默认 60 秒）就会捡起 ready 状态的 worker 并 spawn 对应 profile 的进程。

## Python API：程序化控制

当你需要在代码里动态构建 Swarm 时：

```python
from hermes_cli.kanban_swarm import (
    SwarmWorkerSpec, create_swarm, post_blackboard_update, latest_blackboard
)
from hermes_cli import kanban_db as kb

with kb.connect_closing() as conn:
    created = create_swarm(
        conn,
        goal="收集竞品分析数据并生成决策备忘录",
        workers=[
            SwarmWorkerSpec(
                profile="researcher-a",
                title="Market scan",
                body="Find competitors in the target market",
                skills=["arxiv"],
            ),
            SwarmWorkerSpec(
                profile="researcher-b",
                title="Customer scan",
                body="Find customer pain points",
            ),
        ],
        verifier_assignee="reviewer",
        synthesizer_assignee="writer",
        tenant="intel",
        created_by="orchestrator",
    )

    # Worker 完成后向 blackboard 写入数据
    post_blackboard_update(
        conn, created.root_id,
        author="researcher-a",
        key="sources",
        value=["https://example.com/report1", "https://example.com/report2"],
    )

    # 下游节点读取 blackboard
    board = latest_blackboard(conn, created.root_id)
    print(board["sources"])
    # → ["https://example.com/report1", "https://example.com/report2"]
    print(board["_authors"]["sources"])
    # → "researcher-a"
```

`create_swarm` 通过 `idempotency_key` 参数支持幂等调用——同一个 key 不会创建重复的任务图。这在 retry 逻辑里很有用。

## Dispatcher 怎么调度

Dispatcher 嵌在 gateway 进程里，不是独立服务。每 60 秒一个 tick：

先回收超时任务。Claim TTL 默认 15 分钟，worker 崩溃后 15 分钟任务自动回到 ready 队列。长任务靠 `kanban_heartbeat()` 续期，超过 60 分钟没心跳就被视为卡死。

然后提升就绪任务。`recompute_ready()` 检查所有 todo 和 blocked 状态的任务，parent 全部 done 的提升到 ready。

最后抢占并启动。CAS 原子操作：`WHERE status='ready' AND claim_lock IS NULL`。抢到后 spawn 一个 `hermes -p <profile> chat -q ...` 进程。

并发控制通过两个配置项实现：`max_in_progress` 限制全局并发数，`max_in_progress_per_profile` 限制每个 profile 的并发数。连续失败 2 次（`failure_limit`）的任务自动进入 blocked 状态，等人工介入。

## Worker 的 tool 集

Dispatcher 启动的 worker 进程自动获得这些 Kanban tool：

- `kanban_show()` — 读取自己的任务详情和上游 parent 的 handoff 数据
- `kanban_complete(summary, metadata)` — 结构化完成任务
- `kanban_block(reason, kind)` — 阻塞并说明原因
- `kanban_heartbeat(note)` — 长任务保活
- `kanban_comment(task_id, body)` — 往任何 task 写注释（跨 worker 通信）
- `kanban_create(title, assignee, parents)` — 创建子任务
- `kanban_link(parent_id, child_id)` — 动态添加依赖边

Worker 不需要知道自己在一个 Swarm 里。它只看到"有个任务分配给我，parent 的 summary 和 metadata 在 context 里"。协作是隐式的。

## 并发安全：SQLite 怎么扛住

SQLite 单文件数据库做多 agent 协调，听起来有点莽。但 Hermes 做了几件事让它在实践中可靠：

**WAL mode。** 允许并发读，写操作串行化。多个 worker 同时读 blackboard 没问题。

**BEGIN IMMEDIATE。** 写事务一开始就拿锁，避免 busy retry 和死锁。

**Board-scoped 文件锁。** 防止多个 dispatcher 实例对同一个 board 并发 tick：

```python
with _dispatch_tick_lock(db_path) as held:
    if not held:
        return DispatchResult(skipped_locked=True)
```

**每个 board 独立 DB。** 不同项目的 Swarm 互不影响。

这套方案在单机场景下（macOS、Linux 工作站、CI 服务器）表现稳定。如果你需要跨机器分布式，SQLite 不是正确的选择——但那也不是 Hermes 当前要解决的问题。

## Block Kind：智能路由失败

Worker 卡住时调用 `kanban_block(reason, kind)`，kind 决定任务去哪：

| Kind | 去向 | 含义 |
|------|------|------|
| `dependency` | 回到 todo，等上游完成后自动恢复 | 等别的 task |
| `needs_input` | blocked，等人类介入 | 缺决策信息 |
| `capability` | blocked，等人类介入 | 硬性限制，比如没权限 |
| `transient` | blocked，等人类介入 | 临时故障 |

有个防抖机制：同一个原因反复 block/unblock 超过 2 次，任务路由到 triage 状态，提醒人类这里可能有系统性问题。

## Goal Mode：让 worker 自己跑到完成

普通 worker 是 single-shot 的——跑一轮就结束。设置 `goal_mode=True` 后，worker 进入循环：每轮结束后辅助 judge 模型评估输出是否达标，没达标就继续跑，直到 judge 认可或预算（`goal_max_turns`）耗尽。

这对开放式任务有用——比如"写一个完整的 README"或"重构这个模块的测试"，一轮未必能完成。

## Workspace 隔离

每个 worker 有独立的工作空间：

| Kind | 用途 |
|------|------|
| `scratch` | 临时目录，任务完成后可清理（默认） |
| `worktree` | Git worktree，分支名确定性生成 |
| `dir` | 指定绝对路径的共享目录 |

Project-linked 的任务会在 `<repo>/.worktrees/<task-id>` 下创建工作空间，branch 名为 `<project-slug>/<task-id>`。合完 PR 就可以删。

## 配置一览

```yaml
kanban:
  dispatch_in_gateway: true
  dispatch_interval_seconds: 60
  failure_limit: 2
  dispatch_stale_timeout_seconds: 14400
  max_in_progress_per_profile: null
  auto_decompose: true
  auto_decompose_per_tick: 3
```

`auto_decompose` 开启后，triage 状态的任务会被辅助 LLM 自动分解为子任务图。适合把一个模糊需求丢进去，让系统自己拆解。

## 实际体验

写这篇文章时我们用的就是这套系统。一个 researcher profile 跑源码调研，一个 reviewer 验证技术准确性，最后 writer 合成终稿。三个 agent 各自在独立 workspace 里工作，通过 blackboard 传递数据，通过 parent-child 依赖控制执行顺序。

整个过程大概花了 20 分钟。中间没有人参与决策。Reviewer 发现了三个小问题（版本号偏差、工具列表遗漏、行数 off-by-one），记录在 blackboard 上，writer 在合成时修正。

这种"并行调研 → 质量门控 → 合成交付"的模式在很多场景下都可以复用：代码审查、文档翻译、数据分析报告、竞品调研。你需要的只是定义好 profile 和 Swarm 拓扑。

---

*Hermes Agent 由 Nous Research 开源。Kanban Swarm 从 v0.18.2 起可用。源码和文档见 https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban*
