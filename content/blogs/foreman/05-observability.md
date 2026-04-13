---
title: "Making a distributed system legible: OTel, Prometheus, and the dashboards I actually look at"
date: 2026-04-01
tags: ["observability", "opentelemetry", "prometheus", "grafana", "jaeger", "go"]
description: "End-to-end observability for a distributed job queue"
summary: "Structured logs, OpenTelemetry traces across Kafka, Prometheus metrics, and the four Grafana panels that actually matter."
githubLink: "https://github.com/KIRA009/foreman"
showToc: false
disableAnchoredHeadings: false
---

There's a moment in every distributed system project where you realize that `fmt.Println` doesn't cut it anymore. You have three processes, a message bus, a cache, and a database, and a job just failed somewhere in the middle. Where? When? Which component? The logs say "error processing job" but don't tell you whether the error happened in the worker, the executor, the HTTP call to the target, or the Postgres write afterwards.

Observability is the practice of making your system answer those questions without requiring you to add more print statements. Foreman's observability stack has three layers: structured logs with correlation IDs, distributed traces via OpenTelemetry, and metrics via Prometheus. Each layer answers a different question.

## The three layers, in plain English

**Logs** answer "what happened." Every log line in Foreman is structured JSON, emitted via `log/slog`. Every line in a request path carries a `correlation_id` (a client-supplied or auto-generated ID that links all log lines for a single job's lifecycle) and a `trace_id` (the OpenTelemetry trace ID). If you know the correlation ID, you can grep the logs of all three services and reconstruct exactly what happened to that job.

**Traces** answer "how long did each part take." A trace is a tree of spans. The root span is the HTTP request that submitted the job. Child spans are created for every significant operation: the Postgres write, the Kafka produce, the Kafka consume on the worker side, the HTTP call to the target URL. Each span records its start time, end time, and any attributes you attach (tenant ID, job ID, queue name, etc.). In Jaeger, you can visualize this tree and immediately see which operation was slow.

**Metrics** answer "is the system healthy right now." Prometheus scrapes numeric counters, gauges, and histograms from each binary every 15 seconds. Grafana visualizes them over time. If job completion rate drops and DLQ rate spikes, that's visible on the dashboard immediately — you don't have to search through logs to notice something is wrong.

## Why correlation IDs aren't enough

Before I added OpenTelemetry, I had correlation IDs propagated through logs. They work well for following a single request through one service. They fall apart when you need to connect:

- The HTTP request in `foreman-api` that created the job
- The outbox relay goroutine that produced to Kafka
- The worker process (different binary, different host, different PID) that consumed from Kafka and executed the job

You can propagate the correlation ID across all of these — and Foreman does — but it doesn't tell you *how long* the Kafka message sat in the topic before the worker picked it up, or which part of the executor was slow. Traces give you that.

## How W3C traceparent flows through Kafka

The hard part of distributed tracing in a message-queue system is the handoff between the producer and the consumer. HTTP is easy: you inject trace context into request headers and extract it on the other side. Kafka requires the same pattern but using Kafka message headers.

In Foreman, the outbox relay creates a span when it produces a message to Kafka:

```go
ctx, span := tracer.Start(ctx, "outbox.produce",
    trace.WithAttributes(
        attribute.String("kafka.topic", topic),
        attribute.String("job.id", jobID),
    ),
)
defer span.End()

// Inject W3C traceparent into Kafka message headers
headers := make([]kafka.Header, 0)
propagator := otel.GetTextMapPropagator()
propagator.Inject(ctx, kafkaHeaderCarrier(headers))

// Produce the message with trace headers
writer.WriteMessages(ctx, kafka.Message{
    Headers: headers,
    Value:   payload,
})
```

The `kafkaHeaderCarrier` is a small adapter that implements `propagation.TextMapCarrier` over a slice of Kafka headers — basically a string map that the OTel propagator can read from and write to.

On the consumer side, the worker extracts the trace context before doing any work:

```go
propagator := otel.GetTextMapPropagator()
ctx = propagator.Extract(ctx, kafkaHeaderCarrier(msg.Headers))

ctx, span := tracer.Start(ctx, "job.execute",
    trace.WithAttributes(
        attribute.String("job.id", jobID),
        attribute.String("job.queue", queue),
    ),
)
defer span.End()
```

The `Extract` call recovers the trace context from the headers and makes the consumer's spans children of the producer's spans. In Jaeger, the resulting trace looks like:

```
POST /api/v1/jobs (45ms)
  └─ postgres.tx (8ms)
       ├─ INSERT jobs (3ms)
       └─ INSERT outbox (2ms)
  └─ outbox.produce (12ms)
       └─ kafka.write (10ms)

[~80ms later, in the worker process]

job.execute (230ms)
  └─ postgres.read (5ms)
  └─ http.post target_url (218ms)
  └─ postgres.update_state (4ms)
  └─ redis.publish (1ms)
```

The gap between `kafka.write` and `job.execute` is the Kafka consumer group lag — how long the message sat in the topic before a worker picked it up. That number matters: if it's consistently above a few seconds, you need more workers.

## The metrics I actually look at

I added about 12 Prometheus metrics. In practice, there are four that I look at first when something seems off:

**`foreman_jobs_completed_total{state="dead"}`** — the DLQ rate. If this goes up, something is failing. The `state` label distinguishes `completed`, `failed`, and `dead`. Failed jobs are expected (bad payloads, 4xx from the target); dead jobs mean you've exhausted all retries.

**`foreman_job_duration_seconds{queue="normal"}` at p95** — end-to-end job processing time. This is a histogram so you get the full distribution. If p95 climbs while p50 stays flat, you have a tail latency problem — probably network calls to the target URL timing out.

**`foreman_outbox_backlog`** — how many outbox rows are waiting to be published to Kafka. This is a gauge, scraped from Postgres. If it climbs steadily, the outbox relay is falling behind. Usually caused by Kafka being slow to accept writes (broker pressure) or the relay process being restarted.

**`foreman_kafka_consumer_lag{topic="foreman.jobs.normal"}`** — how far behind the worker consumer group is. This tells you whether your worker pool is keeping up with the submission rate. If it climbs under load, you need to increase worker concurrency or add worker instances.

The others (`api_requests_total`, `api_request_duration_seconds`, `worker_active_jobs`) are useful for capacity planning and debugging specific issues, but they're not what I reach for first.

## The Grafana dashboard

The provisioned Grafana dashboard has six panels arranged in two rows:

Row 1 (health at a glance):
- Job submission rate (counter, rate over 1m)
- Job completion rate broken down by state (completed/failed/dead)
- p95 job duration over time (histogram panel)

Row 2 (infrastructure):
- Worker active jobs gauge
- Kafka consumer lag by topic
- Outbox backlog (gauge, from Postgres)

The alerting rules fire on:
- DLQ rate > 0.1/s for 5 minutes (`ForemanHighDLQRate`)
- Consumer lag > 1000 messages for 2 minutes (`ForemanConsumerLag`)
- Outbox backlog > 500 rows for 5 minutes (`ForemanOutboxBacklog`)
- API error rate > 5% for 5 minutes (`ForemanHighErrorRate`)
- Worker process down (`ForemanWorkerDown`)

These aren't arbitrary thresholds — they're set to fire before the user-visible impact becomes significant, giving you time to investigate before jobs start failing.

## The correlation ID alongside the trace ID

One thing I wanted to preserve even after adding OTel was the correlation ID. Trace IDs are 128-bit hex strings like `4bf92f3577b34da6a3ce929d0e0e4736` — they're machine-readable but not human-friendly. Correlation IDs are short strings like `seed-job-1` that clients supply and that show up in dashboards, error messages, and support tickets.

Every log line in Foreman carries both:

```json
{
  "time": "2026-04-11T10:23:45Z",
  "level": "INFO",
  "msg": "job completed",
  "correlation_id": "seed-job-1",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "job_id": "01j2x3y4z5...",
  "duration_ms": 234
}
```

In practice: a customer reports "job seed-job-1 didn't complete." You grep logs for `correlation_id=seed-job-1`, find the trace ID, search Jaeger by trace ID, and see the full span tree. Two steps to full context.

## What I'd instrument differently

Adding OTel instrumentation after the fact is annoying. The cleanest approach is to propagate context from the very start — pass `context.Context` through every function that does IO, and add spans at the boundaries. Foreman does this, but there were a few places where I had to retrofit context parameters into functions that didn't originally accept them.

I'd also add exemplars to the Prometheus histograms — exemplars are trace ID annotations on histogram observations that let Grafana link a high-latency bucket directly to the Jaeger trace that caused it. The Go Prometheus client supports exemplars but they require careful instrumentation to thread the trace ID through.

Finally, I'd add synthetic monitoring: a scheduled job that fires every minute, calls a known-good target URL, and asserts that the full round-trip (submission → execution → completion) happens within a deadline. That's the canary that catches integration failures that no individual component metric would surface.

---

*Next: [What I'd do differently: a retrospective on Foreman](/blogs/foreman/06-retrospective) — an honest look at what's over-engineered, what I cut corners on, and what I'd change if I were starting over.*
