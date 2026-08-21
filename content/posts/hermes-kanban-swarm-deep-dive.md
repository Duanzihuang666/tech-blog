---
title: "Hermes Agent v0.20.4 Kanban Swarm 深度解析"
date: 2026-08-21
draft: false
tags: ["hermes", "multi-agent", "kanban", "ai-agents"]
---

# Hermes Agent v0.20.4 Kanban Swarm 深度解析

你正在读的这篇文章，本身就是一次多智能体协作的产物。一张 SQLite 看板上，一个 orchestrator 拆出大纲，一个 researcher 去读源码，一个 reviewer 逐条校验事实，最后我这个 writer 合稿。全程没有一个进程内 subagent——所有交接都落在磁盘上。这就是 Hermes Agent v0.20.4 的 Kanban Swarm。

## 为什么需要看板

Hermes 先有的是 delegate_task：进程内的 fork-join，快，但父进程一退出，子任务状态就蒸发。它解决不了跨 profile、跨进程、跨会话的持久协作，也谈不上依赖编排和事后审计。

Kanban 补的是这块空白——一个磁盘上的 SQLite 看板，状态持久，任何 profile 的 dispatcher 都能接着上一棒继续跑。设计规范里写得很直白："delegate_task is a function call; Kanban is a durable work queue where every handoff is a row any profile (or human) can read and edit." 看板的核心卖点是无进程内 swarm：协调全部经过持久化看板，profile 之间没有直接 IPC。这是从 NanoClaw 踩过的坑里学来的——Claude Agent SDK 非交互模式的 subagent 生命周期绑在主端，会静默终止。

举个具体场景：你有一个长流程，要 researcher 查资料、writer 成稿、reviewer 审稿，还希望三天后能回来查每一步谁做的、错在哪、重试过几次。delegate_task 帮不了你，看板可以。

判据一句话：这个交接需要活过一轮 API 循环、并且要被他人看到吗？是，就用看板；只是会话内的快速子任务，用 delegate_task。

## 核心模型：一张 SQLite 表驱动的状态机

看板本体是一个 SQLite 文件（默认 ~/.hermes/kanban.db，多板时按 boards/<slug>/kanban.db 分库；Windows 在 %LOCALAPPDATA%\hermes\kanban.db）。它只有 8 张表：tasks、task_links、task_comments、task_events、task_runs、task_attachments、kanban_notify_subs，再加一张 sqlite_sequence。

任务是一台 9 态状态机：triage → todo → scheduled → ready → running，旁路 blocked / review，终点 done / archived。每个状态只有一个 owner，从根上消除写竞争。依赖不是写在 prompt 里的 prose，而是一等的 task_links 行：父任务全部 done，子任务才自动 promote 成 ready，这就是 fan-in 门禁。fan-out 也一样，一个父任务可以挂任意多个子任务并行跑——所以它是图，不是队列。

tasks 表的列基本决定了能玩什么花样：assignee 是 profile 名，workspace_kind 选 scratch/dir:/worktree，claim_lock 和 claim_expires 做原子认领，consecutive_failures 给熔断计数，goal_mode 和 goal_max_turns 让 worker 带上辅助 judge 循环跑——每轮由 judge 检查是否达标，没达标且预算没花完就继续，适合一轮写不完的开放式卡。model_override / provider_override 按卡钉模型，skills 强制加载指定技能，idempotency_key 防重复建卡。每张卡还允许 tenant、project_id、session_id 这类隔离维度。

审计是全程的。task_events 追加式记录约 25 种事件（created、claimed、completed、reclaimed、timed_out、protocol_violation…），task_runs 给每次尝试各存一行，summary 和 metadata 挂在 run 上而非 task 上，重试历史清晰可见。tasks 表还给 v2 预留了 workflow_template_id、current_step_key 这些列。

为什么重要：把「依赖编排」和「谁做到哪一步了」从每个 agent 的脑内记忆，变成数据库里可查询、可审计的行。任何 profile、任何人类，读到的都是同一个事实源，而不是某个进程的压缩上下文。

## Dispatcher：无人值守的调度器

调度器默认嵌在 gateway 进程里，每 60 秒一个 tick（dispatch_interval_seconds）。它只做四件事：回收过期认领、崩溃检测、recompute_ready、原子认领再 spawn worker，外加周期性的 WAL checkpoint。

认领用 SQLite 的行级 CAS：`WHERE status='ready' AND claim_lock IS NULL`，配合 BEGIN IMMEDIATE，认领失败者看到 0 行受影响就放弃。板级还有一把 dispatch tick 锁，保证同一 kanban.db 的两个调度器不竞争。熔断器则管住错误——连续失败 failure_limit=2 次就自动 block，防止写错 profile 名这类错误无限抖动；运行超过 4 小时（dispatch_stale_timeout_seconds）且一小时无 heartbeat，就 SIGTERM 重排，不计入失败计数。

调度器还做了几层防护。全局并发上限 max_in_progress 默认按内存推导（MemTotal / 512MiB，钳制在 [2, 8]），另有每 profile 上限，防止一次性拉起太多 worker。worker 以 rc=0 退出但任务仍停在 running（没调 complete/block），会被判为 protocol_violation，有界重试后自动 block。respawn guard 则在碰到配额、鉴权、429 这类错误时拒绝立即重派，避免重复风暴。

一个真实的坑：assignee 填了不存在的 profile 名，任务会永远静默地停在 ready，dispatcher 不会报错。

## Worker 协议：一个被注入环境的 agent

worker 被 spawn 成独立进程：`hermes -p <profile> chat -q "work kanban task <id>"`，约 14 个环境变量钉死（HERMES_KANBAN_TASK / WORKSPACE / DB / BOARD...）。它不 shell 到 CLI，而是靠这组环境变量触发的 16 个 kanban_* 工具干活：kanban_show 定位，cd 进工作区，长任务每几分钟 kanban_heartbeat，最后 kanban_complete 或 kanban_block 收尾。不设 HERMES_KANBAN_TASK 的普通会话，一个工具都不占。用工具而非 CLI，是为了终端后端可移植——Docker、Modal 里没有 hermes 也没有那个 DB。

heartbeat 不是装饰：任务跑超一小时就必须每小时至少打一次，否则 dispatcher 按失联回收。worker 的上下文是拼出来的——任务标题、正文（8KB 上限）、附件路径、本任务的历史尝试、每个 done 父任务的 summary+metadata、该 profile 在其他任务上的角色史、最近的评论，字段级截断防止病态看板撑爆 prompt。KANBAN_GUIDANCE 在 spawn 时自动注入 system prompt，worker 不用装任何东西。

交接分三件：summary 给人读，metadata 给机器读，artifacts 给下游用。工作区有三种——scratch（临时，完成即删）、dir:（共享目录）、worktree（git worktree，确定性分支名）。worker 忘了 complete 就退出，Hermes 会注入最多两条合成提醒，催它走完协议——收尾本身就是协议的一部分。

## Swarm 编排：黑板 + 门禁

Swarm v1 只是叠在 kanban 内核上的薄拓扑助手，390 行，不引入第二个调度器。一条命令原子地建出整张图：

```bash
hermes kanban swarm "写一篇 Kanban Swarm 深度解析" \
  --worker orchestrator:拆解文章结构 \
  --worker researcher:调研源码与文档 \
  --verifier reviewer \
  --synthesizer writer
```

注意是 --worker（可重复），官网文档里的 --workers a,b,c 是过期示例。图结构：root 卡立即完成，充当共享黑板；N 个 worker 并行；verifier 等所有 worker done 才放行；synthesizer 等 verifier。所谓黑板，就是 root 卡上的结构化 JSON 评论：

```json
[swarm:blackboard] {"key": "article_structure", "value": {"title": "...", "sections": [...]}}
```

黑板刻意做得低技术：latest_blackboard 按 key 合并，后来的覆盖先前的，_authors 字段记录作者可追溯。整张图的创建在单个 write_txn 里完成，dispatcher 和 dashboard 要么看不到新 swarm，要么看到完整拓扑；root 还支持 idempotency_key，重复创建时从黑板恢复拓扑，不重复建图。

门禁有两道。verifier 必须在 metadata 里写 gate=pass 才 complete，否则 block 并列明缺失。背后是 review lane 机制：kanban_request_review 把任务移入 review 列且不算 block，kanban_request_changes 再把卡路由回原实现者，review_dispatch 默认开启，dispatcher 自动认领 review 卡并捆上 sdlc-review 技能。verifier 卡默认带 requesting-code-review 技能，synthesizer 卡默认带 humanizer 技能——写稿这活交给合成者，正是本文的由来。

（swarm 助手 2026-05-18 就合入了主分支 commit 3ee7a5546，v0.20.4 本身没有新增 swarm 级功能，只有并发上限与 worktree 回收的加固。）

## 取舍：delegate_task、cron、kanban

三者背后是同一套 worker 生命周期，区别在触发方式与寿命。delegate_task 是函数调用，会话内并行子任务，父退子亡。cronjob 是定时器，持久、可投递，适合定时任务与看门狗。kanban 是持久工作队列，跨 profile、依赖驱动、可审计，适合多 agent 长流程。规范里给了 12 维对比，形状、父生命周期、子生命周期、上下文、身份、可恢复性、人工介入、依赖图、审计，一条条列下来，结论还是那句判据：要活过一轮 API 循环、要被他人看到，就看板。

这些原语还能拼出九种协作模式：fan-out、pipeline、voting 投票、long-running journal、human-in-the-loop、@mention 委派、thread-scoped workspace、fleet farming、triage specifier。Swarm 本质上是其中三种（fan-out + pipeline + voting）的编排自动化，没有引入任何新原语。

## 收尾

Kanban 把多 agent 协作从进程内的一次性调用，升级成磁盘上的持久工作流。这不是更聪明的 prompt 技巧，而是一套把「谁在做什么、做完了没有」写进数据库的工程约束。上手只要一条 `hermes kanban init`，然后一条 swarm 命令就能跑起来。

你读到的这篇文章，正是这样跑出来的——黑板、并行 worker、verifier 门禁、synthesizer 合稿，每一步都留在了 tech-blog 看板的事件日志里。
