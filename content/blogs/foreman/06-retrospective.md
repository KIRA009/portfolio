---
title: "What I'd do differently: a retrospective on Foreman"
date: 2026-04-07
tags: ["retrospective", "architecture", "distributed-systems", "go", "lessons-learned"]
description: "An honest retrospective on building Foreman"
summary: "What's over-engineered, what I cut corners on, and the one thing I'd change if I started over."
showToc: false
disableAnchoredHeadings: false
githubLink: "https://github.com/KIRA009/foreman"
---

Foreman is done. From empty repo to a running distributed job queue with a React dashboard, OpenTelemetry tracing, Prometheus metrics, and a Grafana dashboard. It works. I'm satisfied with it. And there's a pile of things I'd do differently if I started over.

This post is the honest retrospective. Not the "here's what I built" tour — I've written five posts about that already. This is the post where I try to articulate what I actually learned, what feels over-engineered in hindsight, and what I cut corners on that I probably shouldn't have.

## What I'm genuinely proud of

**The transactional outbox.** This is the seam I spent the most time thinking about, and it held up. Every job that was submitted made it to Kafka. Every job that Kafka delivered was processed at least once. The idempotency check meant double-deliveries were handled cleanly. In load tests at 500 VUs for 5 minutes, I saw zero job loss. The pattern is not complicated, but it requires you to truly understand why it's necessary — and I do now in a way I didn't when I started.

**The three-binary structure.** Separating API, worker, and scheduler into distinct binaries felt like over-engineering at first. When I got to adding leader election to the scheduler, I was glad they were separate. The scheduler's liveness model is completely different from the API's — the API should never go down, the scheduler is fine to have brief gaps as long as the Redis lock expires quickly. Conflating them would have been messy.

**The priority dispatcher.** Three Kafka topics, polling high → normal → low in order. Simple. Deterministic. Testable. The alternative (per-message priority within a single topic, which Kafka doesn't support) would have required either a different message broker or a priority queue in the worker's memory — neither of which is as clean.

**testcontainers-go.** Every integration test spins up real Postgres, real Kafka, real Redis. No mocks for the infrastructure layer. This caught several bugs that a mocked test would have missed — Kafka producer errors under partition leadership changes, Postgres connection pool exhaustion under load, Redis pub/sub delivery semantics. The tests are slower but they're honest.

## What's over-engineered

**The observability stack.** OTel tracing, Prometheus metrics, Grafana dashboards, alerting rules, Jaeger — for a portfolio project, this is more than necessary. The value was in learning to instrument a distributed system end-to-end, and I don't regret it for that reason. But if I were building this as a real production system on a deadline, I'd start with structured logs and a single metrics dashboard, and add distributed tracing only after I'd shipped and identified a specific debugging need.

**The CLI (`foremanctl`).** A Cobra CLI with `--output table|json`, Viper config, and a `--wait` flag that polls until terminal state is a solid piece of work. But in practice, the seed script and `curl` were always faster for ad-hoc operations during development. The CLI makes sense for users of Foreman; it's of limited value to me as the sole developer. I'd build it as a polish step at the end, not as a primary deliverable.

**The webhook system.** Three retries, exponential backoff, a `webhook_log` table, a channel-fed async dispatcher. Fine. But the jobs themselves are already dispatched via HTTP to a target URL — webhooks are callbacks on the callback. In most use cases, the job submitter would poll the API or subscribe to SSE instead of registering a webhook. I'd make webhooks optional from the start and skip the retry/log infrastructure until there was demonstrated demand.

## What I cut corners on

**Error messages.** The API returns JSON error envelopes with a `code` and `message` field. The `code` values are strings like `"invalid_request"`, `"not_found"`, `"rate_limit_exceeded"`. These are fine for machine consumption. They're not fine for humans — there's no documentation of what each code means, no link to docs, no actionable suggestion in the message. A production API would have a proper error catalog.

**Schema migrations.** All six migration files exist and run correctly. But there's no migration history tracking in staging vs. production, no dry-run mode, no rollback testing. `make migrate-up` runs all pending migrations against whatever `POSTGRES_DSN` points to. That's fine for a local compose stack; it would be genuinely dangerous in a multi-environment deployment.

**The dashboard.** The React dashboard works and looks good. But the SSE reconnect logic on the `useSSE` hook is naive — it reconnects with exponential backoff but doesn't distinguish between "server is down" and "no events have arrived for a while." In production you'd want heartbeat events from the server to distinguish between a quiet system and a broken connection.

**Multi-tenancy.** Every endpoint is tenant-scoped. Tenant A cannot see tenant B's jobs. But "tenants" are just a field on the API key — there's no admin interface for managing tenants, no tenant provisioning flow, no tenant-level quotas or plan limits. The isolation is correct; the management surface is absent.

**Security hardening.** The security audit checklist is in `docs/operations/security-audit.md`. All items are checked. But "all SQL parameterized" is a property of the code today, not a property that's verified continuously — there's no automated check in CI (no CI at all, in fact) that would catch a future developer adding a string-concatenated query. A real production security posture would include linting rules, automated scanning, and mandatory code review for any query changes.

## What I learned that I'll carry forward

**Context cancellation is load-bearing, not defensive.** I added `context.Context` to every function that does IO from the start. This felt ceremonial at first — just passing context everywhere because that's what you do in Go. By the time I implemented graceful shutdown, I understood why: the SIGTERM handler cancels the root context, and that cancellation flows automatically through every goroutine, every Kafka consume call, every Postgres query. The shutdown is clean because the context wiring was done correctly from the start.

**The outbox relay is the most important goroutine.** It's a 50-line polling loop that most users will never think about. But it's the thing that makes "exactly-once-ish" delivery actually work. I spent more time testing the outbox relay than any other single component — crash scenarios, mid-publish failures, database reconnects. That investment paid off.

**Protobuf forces you to think about schema evolution.** Using JSON for Kafka messages feels simpler. It is simpler — until you need to add a field and realize that your consumers are deployed independently of your producers and some of them are still running the old version. Protobuf's field numbering and reserved fields make the compatibility story explicit. `buf breaking` in CI would catch a breaking schema change before it reaches production. I added the tooling; I'd enforce it with CI in a real project.

**Distributed system failures are boring.** I expected the failure scenarios to be dramatic and subtle. Mostly they weren't. The most common failure mode was "a process crashed mid-operation and left some state partially written." The outbox pattern handles this. The idempotency keys handle this. The retry logic handles this. The interesting part was designing the recovery paths, not the failures themselves.

## The one thing I'd change from the beginning

I'd add a simple end-to-end test from the very start. Something like:

```
start docker-compose
POST a job
assert state=completed within 10 seconds
```

Running this test after every commit would have caught integration regressions immediately. Instead I was often running the full manual verification checklist in the README by hand. That's slow and error-prone. An automated end-to-end test harness, however simple, would have been worth 2 hours to set up from the start and would have saved many hours of debugging later on.

---

Foreman is a portfolio project. Its job is to demonstrate that I can design and build a real distributed system from scratch, make defensible architectural decisions, and ship something that actually works. I think it does that. Whether it also demonstrates good judgment about what *not* to build is for the reader to decide.

The code is at [github.com/KIRA009/foreman](https://github.com/KIRA009/foreman).
