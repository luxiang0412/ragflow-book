# 第 24 章 Ingestor 扩容：租约、心跳、重投递、并发与限流

> 本章目标：让多个 Ingestor 进程安全地共享同一个任务队列：一个任务只被一个进程执行，进程死掉后任务能被别人接手，进程正常退出时任务不被误判为失败，一个"毒"文档不会无限重试，对模型服务的并发调用有上限。
> 前置：第 6 章（任务与状态机）。对应代码：`v1.0` 的 `backend/internal/store/task.go`、`internal/ingest/worker.go`。

## 24.1 从一个进程到多个进程

第 6 章的任务队列只在一个进程内部运行：API 和 Ingestor 在同一个进程里，一个 `chan` 就能通知 worker。v1.0 把它们拆成独立进程（`--mode api|ingestor|syncer`），并且 Ingestor 可以起多个。于是有四个新问题：

| 问题 | 场景 | 机制 |
|---|---|---|
| 互斥 | 两个进程同时看到一个 CREATED 任务 | 原子认领 |
| 宕机 | 进程被 `kill -9`、机器断电 | 租约 + 心跳，过期重投递 |
| 正常退出 | 滚动升级时 `docker stop` 发 SIGTERM | 主动归还任务 |
| 毒消息 | 某个文档每次都让进程崩溃 | 投递次数上限 |

外加一个"别把下游打挂"的问题：N 个进程 × M 个 worker 同时调用 LLM，需要**限流**。

## 24.2 RAGFlow 的做法：NATS JetStream

RAGFlow 的任务消息在 NATS JetStream 里，Ingestor 用 pull 模式拉取。消费者的配置（`internal/engine/nats/nats.go:278`）：

```go
	consumer, err := n.stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
		Name:          "RAGFLOW_CONSUMER",
		AckPolicy:     jetstream.AckExplicitPolicy,
		MaxDeliver:    16,
		MaxAckPending: 1024 * 128,
		FilterSubject: "tasks.>",
		AckWait:       60 * time.Second,
		BackOff:       []time.Duration{5 * time.Second, 15 * time.Second, 30 * time.Second, 60 * time.Second},
	})
```

逐项对应上面的四个问题：

- **互斥**：消息被一个消费者拉走后，在确认（Ack）或超时之前不会再发给别人；
- **宕机**：消息没有在 ack 截止时间内被确认，就会被重新投递。设置了 `BackOff` 时，服务端把第一次的截止时间规整为 `BackOff[0] = 5s`。所以 Ingestor 每 **2 秒**发一次 `InProgress()` 续期（`defaultHeartbeatInterval`，`internal/ingestion/service/ingestion_service.go:53`），注释写着它"MUST stay comfortably below the consumer's ack deadline"；
- **毒消息**：`MaxDeliver: 16`，最多投递 16 次；
- **正常退出**：见下文。

### 正常退出：留给重投递

`runTask`（`ingestion_service.go:808`）开头先检查上下文，区分两种取消：

```go
	select {
	case <-ctx.Done():
		if e.ctx.Err() != nil || e.dispatchCtx.Err() != nil {
			common.Info(fmt.Sprintf("Task %s aborted due to ingestor shutdown, leaving for redelivery", task.ID))
			return false
		}
		common.Info(fmt.Sprintf("Task %s cancelled", task.ID))
		e.markTerminalProgress(task)
		stopped := e.markStopped(context.Background(), task.ID)
```

如果是 Ingestor 自己在关闭，任务不算失败，也不算停止，而是"leaving for redelivery"：不确认消息，让它被重新投递。只有用户主动停止，才把任务标成 STOPPED。流水线执行到一半被关闭（`:843`）也是同样的处理。

### 以数据库为准

`settleMessage`（`:964`）决定最终对消息 Ack 还是 Nack，它的注释里有一条原则：

```go
// Settlement queries the DB for the task's actual status: a terminal state
// (COMPLETED/STOPPED/FAILED) means Ack; anything else means Nack. The body's
// return value is advisory only — DB truth is authoritative (BP1).
```

消息队列只负责"叫醒"和"重投"，任务到底做完没有，以数据库里的状态为准。它还用 `recover()` 接住任务里的 panic：把任务标成 FAILED（这样重投递时会直接跳过，不会重跑一个毒消息），再 Nack。连"标成 FAILED"这一步本身也包了一层 recover，因为注释说："a panic raised inside a deferred function is unrecoverable and would take the worker process down"。

### 并发与限流

- 每个 Ingestor 的并发 worker 数由 `--ingestor-max-concurrent-workers` 决定，不传就用 CPU 核数（`NewIngestor`，`:159`）；
- LLM 抽取有一个进程级的 worker 池，并发上限 10（`defaultExtractorConcurrency`，`internal/ingestion/component/extractor.go:88`），对应 Python 版的 `MAX_CONCURRENT_CHATS`；
- embedding 则相反：Go 版不再分批并发，而是把整个文档的 chunk 一次性交给 `Encode`（`component/tokenizer.go` 的注释："ONE `Encode([]string)` call with the entire chunk list"），需要分批的驱动自己在内部处理。

## 24.3 从零实现：用 SQLite 当队列

mini-ragflow 没有 NATS，任务表本身就是队列。四个机制都落在几条 SQL 上。

### 原子认领

```go
func (s *Store) ClaimTask(owner string, ttl time.Duration) (*model.Task, error) {
	now := util.Now()
	row := s.DB.QueryRow(`UPDATE task SET status = 'RUNNING', lease_owner = ?, lease_until = ?,
	    attempts = attempts + 1, updated_at = ?
	  WHERE id = (SELECT id FROM task
	              WHERE status = 'CREATED' OR (status = 'RUNNING' AND lease_until < ?)
	              ORDER BY created_at LIMIT 1)
	  RETURNING `+taskColumns, owner, now+ttl.Milliseconds(), now, now)
```

一条 `UPDATE … RETURNING`：选中、改状态、写租约、计数，在一条语句里完成。SQLite 同一时刻只有一个写者（WAL 模式下读写可以并发，写和写之间串行），所以两个进程不可能认领到同一行。可认领的有两种：新任务，以及**租约已过期**的运行中任务，后者就是"重投递"。`attempts` 相当于 JetStream 的投递次数。

换成 PostgreSQL 或 MySQL 8，惯用写法是 `SELECT … FOR UPDATE SKIP LOCKED`。

### 心跳

worker 每 `lease/3` 续一次租约（默认租约 15 秒，即每 5 秒一次，比例和 RAGFlow 的 2s/5s 接近）：

```go
	err = s.DB.QueryRow(`UPDATE task SET lease_until = ?, updated_at = ? WHERE id = ? AND lease_owner = ?
	  AND status IN ('RUNNING','STOPPING') RETURNING status`, now+ttl.Milliseconds(), now, id, owner).Scan(&status)
```

续约的 `WHERE` 里带着 `lease_owner = ?`。如果这个 worker 卡住太久（比如 GC 停顿或者网络分区），租约过期后任务被别人认领了，它的续约就会失败（`ok=false`），worker 随即放弃这个任务，不再写任何结果（`errLeaseLost`）。第 6 章的"停止"也搭着心跳实现：续约时顺便读回状态，看到 `STOPPING` 就取消任务。

### 三种结局，三种 cause

`execute` 用 `context.WithCancelCause` 区分任务为什么被取消：

```go
	switch {
	case errors.Is(cause, errLeaseLost):
		// Someone else owns the task now; do not touch it.
	case err == nil:
		_, _ = p.Store.FinishTask(t.ID, owner, model.TaskCompleted, "")
	case errors.Is(cause, errStopped):
		_, _ = p.Store.FinishTask(t.ID, owner, model.TaskStopped, "")
	case errors.Is(cause, errShutdown):
		// Not the task's fault: give it back so another ingestor (or this one
		// after restart) runs it — RAGFlow "leaves it for redelivery".
		if released, _ := p.Store.ReleaseTask(t.ID, owner); released {
	default:
		_, _ = p.Store.FinishTask(t.ID, owner, model.TaskFailed, err.Error())
	}
```

（节选，省略了更新文档状态的行。）`FinishTask` 和 `ReleaseTask` 的 `WHERE` 里也都带着 `lease_owner = ?`，失去租约的 worker 写不进去。

#### 踩坑记录：SIGTERM 让任务失败了

v1.0 的第一版里，任务直接在进程的上下文下运行。作者自查代码时怀疑：收到 SIGTERM 后进程上下文被取消，正在运行的任务就会走进 `default` 分支，被标成 FAILED。在 v0.5 的 worktree 上复现，结果确实如此：

```text
Status:FAILED Error:context canceled
```

正常的滚动升级会把正在解析的文档全部标成失败。修法和 RAGFlow 一样：任务在自己的上下文下运行，进程关闭时用 `errShutdown` 作为原因取消它，`ReleaseTask` 把任务放回 `CREATED`。回归测试 `TestShutdownRequeuesRunningTask`。

### 毒消息

```go
// FailExhausted fails tasks whose lease expired after maxAttempts deliveries:
// a task that keeps killing its worker (a poison document) must stop being
// redelivered. RAGFlow relies on the JetStream consumer's MaxDeliver.
```

每次认领前，worker 先把"租约过期且 `attempts >= --max-attempts`（默认 3）"的任务直接标成 FAILED。`ReleaseTask` 不会把 `attempts` 清零，归还过的任务再被认领时，投递次数照样累加。任务里的 panic 由 `safeRun` 接住，转成普通的失败（`TestPanicFailsTaskNotProcess`）。

### 限流

所有远程模型共享一个信号量（`internal/llm/llm.go`，`--max-model-calls`，默认 4）。它是**每个进程**的上限：起 2 个 Ingestor，对模型服务的并发就是最多 8。RAGFlow 也是进程级的。要做全局限流，需要一个共享的令牌桶（比如放在 Redis 里），这留作练习。

### 谁在干活

每个 Ingestor 和 Syncer 每 5 秒往 `worker` 表里写一次心跳（名字、角色、主机、运行中的任务数），API 的 `/system/status` 读这张表，系统页显示"在线/离线"（见图 22-2 的"进程"表）。

## 24.4 运行与验证

单元测试：

```text
TestClaimIsExclusiveAndRejectsDuplicates
TestExpiredLeaseIsRedelivered
TestStopBeforeAndAfterClaim
TestStopRunningTask / TestFailureIsRecorded / TestShutdownRequeuesRunningTask
TestPanicFailsTaskNotProcess / TestPoisonTaskStopsAfterMaxAttempts / TestWorkerRegistry
```

真正的多进程实验：1 个 API 进程、2 个 Ingestor 进程（各 1 个 worker，租约 6 秒），共用一个数据目录。上传一份手册，开启自动关键词和自动问题（真实的 qwen2.5:3b，跑在 CPU 上，一份文档要二十多秒），等任务运行 3 秒后，杀掉正在执行它的那个 Ingestor。

**SIGKILL**（模拟宕机，真实输出）：

```text
[KILL] task runs on ingestor-9f9463c2/0 (pid 2696302, ingB.log); sending SIGKILL
   t+  0.0s  task RUNNING   owner=ingestor-9f9463c2/0    attempts=1
   t+  4.5s  task RUNNING   owner=ingestor-76c5c2e5/0    attempts=2
   t+ 26.6s  task COMPLETED owner=ingestor-76c5c2e5/0    attempts=2
   t+ 26.6s  document DONE
```

被杀的进程来不及做任何事，任务只能等租约过期：4.5 秒后另一个 Ingestor 认领了它（第 2 次投递），从头重新执行。

**SIGTERM**（模拟 `docker stop`，真实输出）：

```text
[TERM] task runs on ingestor-dc1d4b00/0 (pid 2698652, ingB.log); sending SIGTERM
   t+  0.0s  task RUNNING   owner=ingestor-dc1d4b00/0    attempts=1
   t+  0.5s  task CREATED   owner=-                      attempts=1
   t+  1.0s  task RUNNING   owner=ingestor-76c5c2e5/0    attempts=2
   t+ 23.1s  task COMPLETED owner=ingestor-76c5c2e5/0    attempts=2
   t+ 23.1s  document DONE
   progress log:
     Task claimed by ingestor-76c5c2e5/0 (attempt 2).
     Start book pipeline for s-term.md (attempt 2).
     ...
```

任务被主动归还（`CREATED`），1 秒内就被接手，不用等租约过期。两种情况都是第 2 次投递，最终完成，没有被标成失败。（`t+` 的时间来自脚本每 0.5 秒一次的轮询，只能精确到这个粒度。作者在开发 v1.0 时也跑过一次，那次的数字是 4.0s/22.1s 和 1.0s/23.6s。）

注意进度日志从 "attempt 2" 开始：重新认领时进度被重置，第一次执行的记录和 "Requeued: ingestor … shut down." 这一行都被覆盖了，只能在 Ingestor 自己的日志里看到。这是一个可以改进的地方（练习 2）。

#### 踩坑记录：把自己的 shell 杀了

第一次做这个实验时，作者用 `for p in $(pgrep -f "mr --mode ingestor"); do kill $p; done` 去杀 Ingestor，结果执行这个循环的 bash 自己的命令行里也包含这串字符，被一起杀掉了（退出码 144）。之后改为通过 `/proc/<pid>/cmdline` 精确匹配可执行文件的路径，再看它的标准输出指向哪个日志文件，确定要杀的是哪一个进程（`failover.py` 的 `find_pid`）。

## 24.5 练习

1. **巩固**：把 `--lease` 改成 2 秒，同时让心跳间隔维持在 `lease/3`，在 CPU 很忙的机器上会发生什么？用实验验证 `errLeaseLost` 分支。
2. **扩展**：重新认领时不要清空进度日志，而是追加一行"第 N 次执行"，保留之前每次执行的记录。
3. **读源码**：RAGFlow 的 `settleMessage` 以数据库状态决定 Ack 还是 Nack。举一个"任务函数返回 true，但数据库里不是终态"的场景，说明为什么不能相信内存里的返回值。

## 24.6 延伸阅读

- `internal/ingestion/service/ingestion_service.go`、`heartbeat.go`，`internal/engine/nats/nats.go`
- NATS 文档：JetStream Consumers（AckWait、BackOff、MaxDeliver）
- Martin Kleppmann, *How to do distributed locking*（2016），关于租约与 fencing token
