---
title: "Foreman: why I built a distributed job queue from scratch"
date: 2026-03-11
tags: ["go", "kafka", "postgres", "distributed-systems", "portfolio"]
description: "Building a distributed job queue from scratch with Go, Kafka, Postgres, and Redis"
summary: "The motivation and stack choices behind building a self-scaling distributed job queue from scratch."
showToc: false
disableAnchoredHeadings: false
githubLink: "https://github.com/KIRA009/foreman"
---

Every backend engineer eventually meets a job queue. If you're lucky it's a boring tool that quietly ships work to the right places. If you're less lucky — and most of us are less lucky — it's the thing you're paged about at 3am because a worker crashed mid-task, a retry loop is stuck, or somebody's "send email" job is on attempt 47 with exponential backoff that stopped being exponential fourteen attempts ago.

I've used Celery, Sidekiq, BullMQ, and a homegrown SQS-based dispatcher. Each of them does 80% of the job, and the other 20% is where all the interesting questions live. Questions like: what exactly happens when the producer commits the DB row but dies before calling the broker? What happens when two workers pick up the same task because the visibility timeout fired? How do you know a job actually ran, versus silently disappeared into a Kafka partition whose consumer group got rebalanced in the middle of a poll?

So I'm building one. From scratch. In Go. With Kafka, Postgres, Redis, and a React dashboard. I'm calling it **Foreman**.

This first post is the why. The rest of the series is the how.

## What this thing actually does

At the surface level: clients submit jobs over a REST API, the system queues them, workers pick them up, run them, report the result, and optionally fire a webhook callback. Jobs can be scheduled on a cron. Failed jobs get retried with exponential backoff. Permanently dead jobs land in a dead letter queue where a human can triage them. There's a dashboard that shows everything in real time.

If that sounds a lot like Celery, that's because it is. I'm not pretending to build something nobody has built before. I'm building it so that when I eventually have to answer interview questions about how I'd design a distributed job queue, I can gesture at the code and say "like this, and here's why."

## Why not just use Celery

**Well, Celery hides the interesting parts.** Every decision about exactly-once semantics, partition assignment, retry delay encoding, and idempotency key handling is somewhere inside the framework, made for you. Using it teaches you how to configure it. Building one teaches you why those configurations matter.

**Also, the failure modes in a job queue are exactly the failure modes you need to understand to be senior.** Dual writes. Backpressure. Poison messages. Leader election. Graceful shutdown. Consumer rebalancing. Exactly-once is a lie, outbox pattern only gets you close, and knowing *how* close is a distinguishing skill. You can read about this stuff. You can also build it and watch it break.

## The stack, and what got ruled out

Here's what I landed on and why.

**Go for everything backend.** because Go's concurrency primitives (goroutines, channels, `context.Context`, `sync.WaitGroup`) are the exact vocabulary a job queue wants to speak. And the Kafka and Postgres ecosystems are mature enough that I don't have to fight my tools.

**Postgres as the source of truth.** Not Kafka. Kafka is a transport, not a database. The canonical state of every job lives in a Postgres row. This means I can answer "where is job X right now?" with a SQL query, not a topic scan. It also makes the transactional outbox pattern possible, which I'll spend an entire post on later.

**Kafka as the queue.** Not RabbitMQ, not NATS, not SQS. Kafka because partitions give me ordering-per-tenant for free, because consumer groups handle worker rebalancing, and because the operational story (retention, replay, lag monitoring) is battle-tested. Priority levels become separate topics (`foreman.jobs.high`, `...normal`, `...low`) — Kafka has no native priority within a topic, so workers poll high first, then normal, then low, and that gets me what I want.

**Redis for hot state.** Job state cache with read-through, pub/sub for real-time dashboard updates, sliding-window rate limiting, and leader election for the scheduler. Every one of those is a textbook Redis use case. Redis is not a database for me; it's a coordination primitive.

**Protobuf for Kafka messages.** Not JSON. I want to tell an honest schema-evolution story — what happens when you add a field, rename a field, remove a field — and protobuf with `buf` gives me the tooling (`buf lint`, `buf breaking`) to enforce that story in CI. It's a little more ceremony upfront and a lot more discipline later.

**React + Mantine for the dashboard.** I want live updates via Server-Sent Events, not WebSockets. SSE is unidirectional, proxy-friendly, reconnects trivially, and carries everything the dashboard needs. Mantine gets me a polished table, tabs, timeline, and notifications without having to rebuild those from scratch, so I can spend my time on the parts that matter.

**Observability from day one, not bolted on.** OpenTelemetry traces flowing through HTTP headers *and* Kafka headers (the W3C traceparent spec covers both), Prometheus metrics from every binary, Grafana dashboards, Jaeger for trace visualization. If I can't see what the system is doing, I haven't built a distributed system, I've built a distributed problem.

## What's in the box

Right now, after one focused building session, the repo has:

- A full monorepo skeleton — `cmd/`, `internal/`, `proto/`, `pkg/`, `web/dashboard/`, `deployments/`, the works.
- A `go.mod` pinned to `github.com/KIRA009/foreman`.
- A Docker Compose stack that boots Postgres 16, Redis 7, and Kafka (in KRaft mode — no ZooKeeper, because it's 2026).
- Six PostgreSQL migrations covering every table I'll need: `jobs`, `schedules`, `dead_letter_queue`, `outbox`, `webhook_log`, `api_keys`. Every index, every constraint, every column is there.
- A Makefile that knows how to build, test, lint, migrate, and boot the stack.
- Stub `main.go` files for `foreman-api`, `foreman-worker`, `foreman-scheduler`, and `foremanctl`, each parsing their config via `envconfig` and logging a startup line via `slog`.
- A `buf` toolchain ready for the protobuf schemas that arrive next.

Not much, but it all compiles, it all lints, and `docker compose config -q` validates clean. The foundation is a foundation.

Here's the tree as of right now:

```
foreman/
├── cmd/{api,worker,scheduler,foremanctl}/main.go
├── internal/
│   ├── api/ worker/ scheduler/ domain/ kafka/ webhook/
│   ├── repository/{postgres,redis}/
│   ├── config/config.go
│   └── observability/
├── proto/foreman/v1/
├── pkg/{idgen, httputil, pb/foreman/v1/}
├── deployments/
│   ├── docker-compose.yml
│   ├── kafka/ prometheus/ grafana/ otel/
├── web/dashboard/
├── Makefile  buf.yaml  buf.gen.yaml  go.mod  .golangci.yml
```

Docker compose status after boot

## What's next

Next is where this stops being scaffolding and starts being code. Domain entities for jobs, schedules, and DLQ entries. A state machine. Postgres repositories for every table, with integration tests spun up via `testcontainers-go` against a real Postgres — no mocks at the repo layer, ever, because a mock will happily accept an SQL statement that the real database would reject at parse time.

After that comes Kafka, the outbox relay, and the first real proof that this thing can actually exchange messages between two processes without losing anything. That's also where I'll write the post I've been wanting to write for years — "exactly-once is a lie, but the outbox pattern gets close." Stay tuned for that one.

*Next in the series: [how I carved Foreman into services — and why I chose three binaries instead of one.](/blogs/foreman/02-service-boundaries)*