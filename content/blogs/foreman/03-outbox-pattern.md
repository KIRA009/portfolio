---
title: "Exactly-once is a lie, but the outbox pattern gets close"
date: 2026-03-21
tags: ["foreman", "kafka", "postgres", "distributed-systems", "outbox"]
description: "The dual-write problem and the transactional outbox pattern"
summary: "Why you can't atomically write to Postgres and Kafka, and how the outbox pattern with idempotent consumers gets you exactly-once semantics."
githubLink: "https://github.com/KIRA009/foreman"
showToc: false
disableAnchoredHeadings: false
---

When I started building Foreman's job submission flow, I hit a question I'd seen glossed over in tutorials: how do you atomically write to a database *and* publish a message to Kafka? The answer seems obvious — do both. But "do both" isn't a transaction. One can succeed and the other can fail, and now your system is lying to itself.

This post is about the dual-write problem, why Kafka transactions don't fix it, and how the transactional outbox pattern does.

## The broken version

Here's the naive job submission handler:

```go
// DON'T DO THIS
func (h *Handler) CreateJob(w http.ResponseWriter, r *http.Request) {
    job := parseJobFromRequest(r)
    
    if err := h.jobsRepo.Create(ctx, job); err != nil {
        http.Error(w, "db write failed", 500)
        return
    }
    
    // What if the process crashes here?
    
    if err := h.producer.Publish(ctx, job.KafkaTopic(), job); err != nil {
        // Job is in Postgres but will never be picked up.
        // Do we delete the job? Retry the publish? Log and hope?
        http.Error(w, "kafka write failed", 500)
        return
    }
    
    w.WriteHeader(202)
}
```

The comment says it all. If the process crashes between the Postgres write and the Kafka produce, the job is in the database in `pending` state forever. No worker will ever see it. The client got a 500, so they don't know whether to retry. If they do retry, they create a duplicate.

This isn't a theoretical edge case — process crashes happen. OOM kills, deployments, kernel panics. On a system processing thousands of jobs per hour, this will happen.

## Why Kafka transactions don't help

At this point you might reach for Kafka's transaction API. Kafka does support transactions — a producer can open a transaction, produce to multiple topics/partitions, and commit or abort atomically. If you've used two-phase commit in databases, it's the same idea applied to Kafka.

But Kafka transactions cover the *Kafka side only*. They guarantee that either all your Kafka messages commit or none do. They say nothing about what happens in Postgres.

To atomically commit a Postgres write *and* a Kafka produce would require two-phase commit across Postgres and Kafka — and neither supports acting as the transaction coordinator for the other. This is a distributed transaction, and distributed transactions are famously hard to get right.

The outbox pattern sidesteps this entirely by not crossing the transaction boundary.

## The outbox pattern

The insight: both writes happen inside a single Postgres transaction.

```sql
BEGIN;
  INSERT INTO jobs (...) VALUES (...);
  INSERT INTO outbox (topic, partition_key, payload, ...) VALUES (...);
COMMIT;
```

The `outbox` table is a staging area. Instead of writing directly to Kafka, the handler writes to a Postgres table in the same transaction as the job. If the transaction commits, both rows exist. If it fails (for any reason), neither does. Postgres's ACID guarantees handle the atomicity.

Then a background goroutine — the outbox relay — polls for unpublished rows and produces them to Kafka:

```go
func (r *OutboxRelay) flush(ctx context.Context) error {
    entries, err := r.source.ListUnpublished(ctx, 100)
    if err != nil {
        return err
    }

    var published []int64
    for _, e := range entries {
        if err := r.producer.PublishRaw(ctx, e.Topic, e.PartitionKey, e.CorrelationID, e.Payload); err != nil {
            r.logger.Error("produce failed", "outbox_id", e.ID, "err", err)
            continue // retry on next tick
        }
        published = append(published, e.ID)
    }

    if len(published) > 0 {
        return r.source.MarkPublished(ctx, published)
    }
    return nil
}
```

The relay runs every 100ms. What happens on crash? If the relay crashes after producing but before marking published, those rows remain unpublished. On the next startup, the relay picks them up and produces them again. The worker receives the message twice.

This is *at-least-once* delivery, not exactly-once. But the job is not lost.

## Consumer-side idempotency

The duplicate delivery problem is solved on the consumer side: the worker checks the idempotency key before executing.

When a worker picks up a `JobMessage` from Kafka, the first thing it does is attempt to transition the job from `pending` to `running` in Postgres. If the job is already `running` or in a terminal state, it means a duplicate message arrived. The worker drops it and commits the Kafka offset.

The idempotency unique index (`UNIQUE (tenant_id, idempotency_key)`) enforces this at the database level — no race condition is possible between two workers racing to claim the same job.

Together, the outbox pattern (at-least-once) and the idempotency check (dedup) combine to give you the application-level guarantee that each job executes exactly once. It's not a single atomic operation — it's a carefully designed protocol.

## What's stored in the outbox

The outbox `payload` column is `BYTEA`, not `JSONB`. The payload is serialized protobuf bytes, stored exactly as they'll be sent to Kafka. When the relay runs, it reads the bytes and calls `WriteMessages` directly — no deserialization, no re-serialization.

Why protobuf instead of JSON? Schema safety — covered in the next post. But the storage choice follows from the format: since Kafka messages are binary protobuf, storing the serialized bytes avoids a round-trip.

## The invariant

Here's the invariant the outbox pattern provides, stated plainly:

> If a job row exists in Postgres with state `pending`, there is an unpublished row in the outbox that will eventually be produced to Kafka and result in the job being picked up.

This is guaranteed by the transaction: the two rows are written atomically. And the relay is guaranteed to eventually produce the row (it retries on failure). The only gap is time — up to 100ms between commit and produce.

## Crash scenarios

Let me trace through the scenarios that used to be failures:

**Scenario: API crashes between Postgres commit and relay tick**
- `jobs` row exists, `outbox` row exists with `published = FALSE`
- On restart, the relay polls and finds the unpublished row
- Produces to Kafka, marks published
- Worker picks up job and executes
- No job lost, no duplicate

**Scenario: Relay crashes after Kafka produce but before marking published**
- `outbox` row still has `published = FALSE`
- On restart, relay produces again
- Worker receives duplicate message
- Worker checks: job already `completed` → drops message, commits offset
- No job lost, no double execution

**Scenario: Postgres transaction fails (e.g. idempotency conflict)**
- Neither `jobs` nor `outbox` row is written
- Client receives 409 with the existing job
- No Kafka message produced, because there's no outbox row to relay
- Clean failure

## Operational consideration: outbox cleanup

The outbox table grows as jobs are submitted. Published rows are no longer needed but aren't deleted automatically. A periodic goroutine (added later) runs `DELETE FROM outbox WHERE published = TRUE AND published_at < NOW() - INTERVAL '24 hours'`. Without this, the table grows unboundedly and the `idx_outbox_unpublished` partial index becomes less efficient as the table size diverges from the number of unpublished rows.

---

The outbox pattern is one of those things that feels like extra work until the first time you see a job queue silently drop work. The implementation in Foreman is about 100 lines of Go and a two-column SQL query. The protocol it implements has been battle-tested for years across distributed systems. Worth understanding deeply before dismissing it as complexity overhead.

*Next up: [the worker pool — goroutines, channels, and graceful shutdown under SIGTERM.](/blogs/foreman/04-worker-pool)*
