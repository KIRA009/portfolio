---
title: "Drawing lines: how I carved Foreman into services"
date: 2026-03-15
tags: ["architecture", "go", "kafka", "postgres", "distributed-systems"]
description: "Service boundary decisions in a distributed job queue"
summary: "Why three binaries instead of one, Postgres as source of truth, and the transactional outbox seam between Postgres and Kafka."
githubLink: "https://github.com/KIRA009/foreman"
showToc: false
disableAnchoredHeadings: false
---

One of the first decisions you make when building a distributed system is where to draw the service boundaries. It's easy to under-split (you end up with a monolith that's hard to scale) or over-split (you end up with 40 microservices coordinating over the network for what used to be a function call). Foreman lands somewhere in between: three Go binaries, one React dashboard, one Postgres database, and a message bus.

This post walks through how I arrived at that structure, why the data-flow looks the way it does, and what tradeoffs I accepted along the way.

## Why three binaries instead of one?

The simplest version of Foreman would be a single binary: accept jobs over HTTP, process them in goroutines, done. That works fine up to maybe a few hundred concurrent jobs. Beyond that you start hitting limits.

The core problem is that job *submission* and job *execution* have very different scaling profiles. Submission is latency-sensitive: a client POSTs a job and expects a fast 202. Execution is throughput-sensitive: you want to run as many jobs as possible, as fast as possible, without blocking the HTTP server. If you run them in the same process, a surge in job submissions can starve in-flight executions (both compete for goroutines, memory, and CPU), and a surge in job executions can make submissions feel sluggish.

The scheduler has a different profile entirely: it needs exactly-once firing semantics (you don't want the same schedule to fire twice just because you deployed a new instance), which means you need leader election. Running leader election inside the API binary would couple the scheduler's liveness to the API's liveness. A scheduler crash shouldn't take the API down.

So three binaries:

- `foreman-api` — HTTP server, job submission, real-time streaming, owns all writes via transactional outbox
- `foreman-worker` — Kafka consumer, job executor, retry logic, DLQ, webhooks
- `foreman-scheduler` — single-leader cron evaluator, fires scheduled jobs via the outbox

Each binary has a clear ownership boundary and can be scaled, restarted, or debugged independently.

## Postgres as the source of truth

Early in the design I considered using Kafka as the primary store — the "event sourcing" approach where the job's state is derived by replaying the event log. I ruled it out for two reasons.

First, Kafka's retention is finite. You configure it in time or bytes, and old messages are eventually compacted or deleted. A job that was submitted six months ago and is now in the DLQ needs to be queryable forever; that's what databases are for.

Second, CRUD queries against Kafka are painful. "Give me all jobs that are in the `failed` state, sorted by creation time, for tenant X, paginated" is a three-line SQL query. Against Kafka it's a full topic scan.

Postgres is the source of truth. Kafka is the transport layer — it moves work from the API to the workers, nothing more. This split is philosophically important: if Kafka were to disappear and be replaced with RabbitMQ or SQS, the domain model (jobs, schedules, state machine) would not change at all.

## The transactional outbox: the seam between Postgres and Kafka

The obvious way to submit a job is:

1. INSERT the job into Postgres
2. Produce a message to Kafka

The problem is that these two operations are not atomic. If the process crashes between step 1 and step 2, you have a job in Postgres but no corresponding Kafka message — the worker will never pick it up. If you do them in the reverse order (produce first, then insert), you get the opposite problem: a Kafka message for a job that doesn't exist in the database.

The transactional outbox pattern solves this. The API writes to two tables in a single Postgres transaction:

```sql
BEGIN;
INSERT INTO jobs (...) VALUES (...);
INSERT INTO outbox (payload, topic) VALUES (..., 'foreman.jobs.normal');
COMMIT;
```

A separate goroutine — the outbox relay — polls `outbox WHERE published = FALSE` every 100ms, produces to Kafka, and marks the row as published. If the relay crashes mid-flight, the row stays unpublished and will be picked up on the next poll. The only "duplicate" scenario is if the relay crashes *after* producing to Kafka but *before* marking the row as published — in that case the worker gets the message twice. The worker handles this with an idempotency check: it reads the job's current state before doing any work, and skips it if it's already in a terminal state.

This is the key insight: exactly-once delivery is impossible at the network level. The best you can do is *at-least-once* delivery with idempotent consumers. The outbox gives you at-least-once; the idempotency key gives you the consumer-side safety net.

## Priority via separate Kafka topics

Kafka doesn't have per-message priority within a topic. If you want high-priority jobs to be processed before low-priority ones, you can't just set a priority flag on the message — the consumer reads messages in order of offset.

The standard workaround is separate topics. Foreman uses three: `foreman.jobs.high`, `foreman.jobs.normal`, and `foreman.jobs.low`. The worker's priority dispatcher polls them in order: it checks the high topic first, then normal, then low. If there's work available on the high topic, it never touches the low topic. This gives you strict priority ordering at the cost of three separate Kafka consumers.

The tradeoff is explicit: low-priority jobs can starve if high-priority jobs never stop arriving. That's the intended behavior — if you submit a job with priority `high`, you're saying "this matters more than everything else." Users who don't want strict prioritization can submit everything to `normal`.

## The data-flow diagram, walked step by step

Here's what happens when a client submits a job:

```
POST /api/v1/jobs
  │
  └─► API handler validates request, generates job ID
        │
        └─► Single Postgres transaction:
              INSERT INTO jobs (id, state='pending', ...)
              INSERT INTO outbox (topic, payload=protobuf bytes)
              COMMIT
                │
                └─► Outbox relay goroutine (polls every 100ms):
                      Produce to Kafka foreman.jobs.{priority}
                      UPDATE outbox SET published=TRUE WHERE id=...
                        │
                        └─► Worker (priority dispatcher):
                              Read message from Kafka
                              Idempotency check (skip if terminal)
                              UPDATE jobs SET state='running'
                              Publish to Redis job.updates channel
                              HTTP POST to payload.target_url
                                │
                                ├── 2xx → UPDATE state='completed'
                                ├── 4xx → UPDATE state='failed' (no retry)
                                └── 5xx → Produce to foreman.jobs.retry
                                           (exponential backoff)
                                           ... after max_retries:
                                           INSERT dead_letter_queue
                                           UPDATE state='dead'
                              │
                              └─► Publish state change to Redis job.updates
                                    │
                                    └─► API SSE endpoint fans out to clients
```

Every state change — running, completed, failed, dead — is published to a Redis pub/sub channel. The API maintains a pool of SSE connections and fans out those events to subscribers, filtered by tenant. The React dashboard subscribes to this stream and updates in real time without polling.

## Why the scheduler is a singleton

The scheduler's job is to evaluate cron expressions and fire jobs at the right time. If two instances of the scheduler run simultaneously and both evaluate the same schedule, you get double-fired jobs. That's usually bad.

The solution is leader election: only one instance runs at a time, and if it crashes, another takes over within seconds. Foreman uses Redis for this: the scheduler tries to acquire a lock (`SET scheduler:lock {id} NX EX 10`) every 5 seconds. The instance that holds the lock is the leader and runs the tick loop. If the leader crashes, the key expires after 10 seconds and another instance picks it up.

The `NX` flag means "only set if the key doesn't already exist" — this is Redis's built-in compare-and-swap primitive, and it's what makes the election safe even under concurrent attempts.

I chose Redis over etcd or Zookeeper because Redis was already in the stack for rate limiting and pub/sub. Adding a separate consensus system for a single-concern singleton would be overkill. The downside is that Redis itself is a single point of failure — if Redis goes down, no new jobs fire from schedules. That's an acceptable tradeoff for a portfolio project; a production system might want Redis Sentinel or Cluster.

## What I'd scope differently

If I were building this for a real production workload, I'd think carefully about a few things:

**The outbox relay as a separate process.** Right now it runs inside `foreman-api`. That means a noisy API instance (lots of HTTP traffic) competes with the relay for CPU. Pulling the relay into its own binary or goroutine pool would give better isolation.

**Kafka consumer group rebalancing.** The `CooperativeStickyAssignor` minimizes rebalance disruption, but rebalances still happen when workers are added or removed. Under heavy load, a rebalance can cause a short processing pause. A more sophisticated setup would use a static membership model.

**The scheduler's 1-second tick.** Polling every second works for minute-level cron granularity, but it's wasteful for a system with many schedules. A smarter implementation would compute the next fire time across all schedules and sleep until the earliest one, waking up only when needed.

These aren't defects — they're deliberate simplifications that let the core architecture tell a clear story without getting lost in operational edge cases.

---

*Next: [Exactly-once is a lie, but the outbox pattern gets close](/blogs/foreman/03-outbox-pattern) — a deep dive into dual-write problems, Kafka transaction limits, and how the outbox relay works at the code level.*
