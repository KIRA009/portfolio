---
title: "Goroutines, channels, and graceful death"
date: 2026-03-25
tags: ["go", "concurrency", "kafka", "distributed-systems"]
description: "Goroutine pools, priority dispatch, and graceful shutdown in Go"
summary: "Building a bounded worker pool with priority Kafka dispatch and clean SIGTERM shutdown — no work abandoned, no offsets lost."
githubLink: "https://github.com/KIRA009/foreman"
showToc: false
disableAnchoredHeadings: false
---

When I set out to build Foreman's worker, I had a rough mental model: consume from Kafka, run the job, write the result. Simple enough. What I didn't anticipate was how much of the implementation would be about the *structure* of concurrent execution — not the work itself, but the machinery that contains it.

This post is about that machinery: goroutine pools, priority dispatch, and how to shut a concurrent system down without losing work.

---

## What the worker actually does

The `foreman-worker` binary consumes job messages from three Kafka topics (`foreman.jobs.high`, `foreman.jobs.normal`, `foreman.jobs.low`) and for each message:

1. Looks up the job in Postgres to verify it's still pending (idempotency)
2. Transitions the job to `running`
3. POSTs the job payload to its `target_url`
4. Transitions to `completed`, `failed`, or `dead` based on the HTTP response
5. Commits the Kafka offset (so the message won't be redelivered)

Steps 3-4 are the "execution" — the rest is bookkeeping. But the bookkeeping is where correctness lives.

---

## The pool

Without bounds, a burst of Kafka messages could spin up thousands of goroutines, each blocking on an outbound HTTP connection or a Postgres write. Connection pools would be exhausted, memory would spike, and the worker would become a bottleneck rather than a relief valve.

The fix is a pool — a bounded execution environment. Here's the core of it:

```go
type Pool struct {
    sem    chan struct{}
    wg     sync.WaitGroup
    active int32
}

func NewPool(size int) *Pool {
    return &Pool{sem: make(chan struct{}, size)}
}

func (p *Pool) Submit(ctx context.Context, fn func()) error {
    select {
    case p.sem <- struct{}{}: // acquire a slot
    case <-ctx.Done():
        return ctx.Err()
    }
    p.wg.Add(1)
    go func() {
        defer func() {
            <-p.sem // release the slot
            p.wg.Done()
            atomic.AddInt32(&p.active, -1)
        }()
        atomic.AddInt32(&p.active, 1)
        fn()
    }()
    return nil
}
```

A buffered channel of `struct{}` values acts as a semaphore. Sending to it acquires a slot; receiving releases one. If all slots are taken, `Submit` blocks on the `select` — it either waits for a slot to open or returns if the context is cancelled.

I could have used pre-spawned goroutines reading from a shared job channel instead. The tradeoff: pre-spawned workers have slightly lower per-job overhead (no goroutine creation) but require managing a separate channel, and context propagation per job is messier. For a job executor where the dominant cost is network I/O, goroutine creation overhead is irrelevant.

The `wg sync.WaitGroup` is how `Drain()` works:

```go
func (p *Pool) Drain() {
    p.wg.Wait()
}
```

When the worker receives SIGTERM, it calls `pool.Drain()`. This blocks until every in-flight job finishes. No work is abandoned; the process doesn't exit until the last job commits its Kafka offset.

---

## Priority dispatch

Foreman uses three separate Kafka topics for priority. The worker must prefer high-priority messages over normal ones, and normal over low, but it can't afford to starve low-priority work indefinitely.

My first implementation used short-lived contexts to poll each topic in sequence:

```go
func (d *PriorityDispatcher) tryFetch(ctx context.Context) *Message {
    shortCtx, cancel := context.WithTimeout(ctx, 10*time.Millisecond)
    defer cancel()
    if msg, err := d.high.FetchMessage(shortCtx); err == nil {
        return msg
    }
    // ... try normal with 50ms, low with 100ms
}
```

This seemed reasonable: check high first, give it 10ms, fall through if nothing. In testing, it completely failed.

The problem is Kafka consumer group setup. When a consumer joins a group for the first time, it needs to: find the group coordinator → JoinGroup → SyncGroup → receive partition assignments. This handshake takes 2-5 seconds for a fresh group. A 10ms context cancels mid-handshake. The consumer retries, and the context cancels again. The consumer never gets partitions and never fetches any messages.

The fix was to separate the fetching from the prioritization:

```go
// Three background goroutines, one per topic.
// Each blocks indefinitely on FetchMessage.
go d.fetchLoop(ctx, d.high, d.highCh)
go d.fetchLoop(ctx, d.normal, d.normalCh)
go d.fetchLoop(ctx, d.low, d.lowCh)

func (d *PriorityDispatcher) fetchLoop(ctx context.Context, c *Consumer, ch chan<- *Message) {
    for {
        msg, err := c.FetchMessage(ctx)
        if err != nil {
            if ctx.Err() != nil {
                return
            }
            continue
        }
        select {
        case ch <- msg:
        case <-ctx.Done():
            return
        }
    }
}
```

Now each consumer can complete its group setup without interruption. Messages flow into buffered channels (`highCh`, `normalCh`, `lowCh`) as they arrive.

Priority is then implemented in the dispatch loop using a three-step non-blocking select:

```go
func (d *PriorityDispatcher) pickMessage(ctx context.Context) *Message {
    // 1. Drain any queued high messages immediately.
    select {
    case msg := <-d.highCh:
        return msg
    default:
    }

    // 2. Prefer high over normal (non-blocking on both).
    select {
    case msg := <-d.highCh:
        return msg
    case msg := <-d.normalCh:
        return msg
    default:
    }

    // 3. Block until any topic has a message.
    select {
    case msg := <-d.highCh:
        return msg
    case msg := <-d.normalCh:
        return msg
    case msg := <-d.lowCh:
        return msg
    case <-ctx.Done():
        return nil
    }
}
```

Step 1 drains any buffered high-priority messages before looking at normal. Step 2 prevents a race where a high-priority message arrives at the same time as a normal one. Step 3 blocks efficiently without spinning when all queues are empty.

This approach also meant I discovered the second bug: the integration test created only the `normal` topic. The `high` and `low` consumers were failing with `Unknown Topic Or Partition`, looping through errors, and never contributing to the consumer group. Once I created all three topics in the test setup, everything worked.

---

## SSE vs WebSockets

Before I move on, I want to address a question that comes up whenever I describe the real-time update path: why SSE and not WebSockets?

SSE (Server-Sent Events) is a one-directional HTTP stream. The server pushes events; the client only reads. WebSockets are bidirectional.

For Foreman's use case — the dashboard watching job status updates — the direction of communication is entirely server-to-client. There's no need for the browser to send data to the API over the same connection. SSE is the right fit.

The practical advantages:
- Proxies and load balancers handle SSE correctly without special configuration (it's just HTTP). WebSocket upgrades can cause proxy surprises.
- SSE reconnects automatically on disconnect (built into the browser EventSource API). WebSocket reconnect requires client-side code.
- No framing complexity. Each SSE event is a few lines of text.

The one limitation: SSE is HTTP/1.1 and HTTP/2. HTTP/2 multiplexes SSE streams over a single connection, but HTTP/1.1 limits browsers to 6 connections per domain — if a user opens multiple SSE tabs, they'll hit that limit. For a monitoring dashboard with one SSE stream per tenant, this isn't a real concern.

---

## Graceful death

SIGTERM handling in Go is straightforward:

```go
func GracefulShutdown(cancel context.CancelFunc, pool *Pool, hb *Heartbeat, timeout time.Duration, logger *slog.Logger) {
    ch := make(chan os.Signal, 1)
    signal.Notify(ch, syscall.SIGTERM, syscall.SIGINT)
    <-ch

    logger.Info("shutdown signal received")
    cancel() // signals all goroutines to stop

    done := make(chan struct{})
    go func() {
        pool.Drain()
        close(done)
    }()

    select {
    case <-done:
        logger.Info("all jobs drained")
    case <-time.After(timeout):
        logger.Warn("drain timeout exceeded, exiting")
    }

    hb.Deregister(context.Background())
}
```

The sequence matters:
1. `cancel()` stops the fetch loops (they see `ctx.Err() != nil` and return)
2. `pool.Drain()` waits for in-flight jobs to finish
3. `hb.Deregister()` removes the worker from Redis

Step 1 happens before step 2. Once the fetch loops stop, no new jobs enter the pool. The drain only has to wait for jobs that were already submitted. If we stopped heartbeats before draining (wrong order), the monitoring layer would think the worker is dead while it still has active jobs.

The drain timeout (default 30s) is a safety valve. A job that takes longer than 30s to complete after the shutdown signal is an edge case — if it's a legitimate long-running job, the timeout needs to be configured accordingly. If it's a hung job, the timeout prevents the worker from blocking indefinitely.

Jobs that don't finish before the timeout are abandoned: their Postgres state is `running` (not `completed`) and their Kafka offsets are uncommitted. On the next worker startup, Kafka will redeliver those messages. The idempotency check in `processMessage` will find the job in `running` state and transition it to `pending` again (via `CanTransitionTo` logic) before executing it.

Actually — that's not quite right yet. The current state machine allows `running → running` (which `CanTransitionTo` would block), meaning an orphaned `running` job would get skipped rather than re-executed. A future improvement will add a Postgres sweep that resets stale `running` jobs back to `pending` after a configurable staleness threshold (e.g., 10 minutes). For now, the happy path works correctly; the edge case is documented.

---

## What's next

The worker can execute jobs, but there's no visibility into what's happening unless you query Postgres directly. The next post covers:

- A Redis cache so job state reads don't always hit Postgres
- Pub/sub so the API can push state changes to connected SSE clients
- A live dashboard feed: watch jobs go `pending → running → completed` in real time

The worker publishes a `job.updates` event to Redis after every state transition. The API subscribes and fans out to SSE clients filtered by `tenant_id`. It's a small amount of code for a lot of perceived interactivity.

*Next post: [building the real-time layer with Redis pub/sub and SSE.](/blogs/foreman/05-observability)*
