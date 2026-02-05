---
title: "Orchestrating Containers from Scratch" 
date: 2025-10-03
tags: ["golang", "distributed-systems", "docker", "redis", "backend-engineering"]
description: "Building a Self-Scaling Container Orchestrator from Scratch with Go and Redis"
summary: "A deep dive into systems engineering: how to build an elastic task engine using Go, Redis, and the Docker SDK, featuring custom autoscaling, scale-to-zero logic, and synchronous log streaming."
showToc: false
disableAnchoredHeadings: false
githubLink: "https://github.com/KIRA009/orchestrator"

---

---

When we talk about "scaling," we usually talk about cloud providers doing the work for us. But what happens under the hood? To figure this out, I tried building a **self-scaling distributed task engine** using **Go**, **Redis**, and the **Docker SDK**. 

The goal was to create a system that autonomously manages a fleet of workers based on real-time queue pressure—without using Kubernetes or any heavyweight frameworks.

## 🏗️ The Architecture: Separation of Concerns
The system is built on the "Control Plane vs. Data Plane" philosophy. This decoupling ensures that if the brain (the API) crashes, the muscle (the workers) can finish their current jobs safely.

1. **The Control Plane (Go)**: A service that exposes a REST API for job submission and runs an autonomous Autoscaler loop.
2. **The Backbone (Redis)**: Acts as the source of truth. It handles the task queue (via BRPOP), worker heartbeats, and real-time log Pub/Sub.
3. **The Data Plane (Docker)**: A fleet of worker containers. Each worker is a Go binary that pops a job, executes a multi-step pipeline, and pipes results back home.

## 📈 The Autoscaler: A Reconciliation Loop
The most critical part of the system is the **Autoscaler**. It implements a pattern common in systems like Kubernetes: the **Reconciliation Loop**.

Every few seconds, the loop performs a "Sense-Analyze-Act" cycle:

- **Sense**: It hits Redis to check the queue depth.
- **Analyze**: It calculates how many workers are needed based on a simple calculation (number of unprocessed jobs > current workers), capped at a MAX_WORKERS limit to prevent system exhaustion.
- **Act**: If the current worker count is too low, it triggers the Docker SDK to spawn new instances.

## 🧨 Elasticity: Why Workers Self-Destruct
In many systems, the central scaler is responsible for killing idle workers. I took a different approach: **Graceful Self-Termination**.

Instead of the Control Plane "murdering" workers (which risks killing a worker mid-job), the workers are responsible for their own lifecycle. Using Go's context.WithTimeout and Redis's BRPOP, the logic is simple:

- The worker waits for a job from Redis.
- If no job arrives within 10 seconds, the BRPOP times out.
- The worker exits cleanly.

This allows the system to **Scale-to-Zero** naturally. When the queue is empty, the workers quietly vanish, and system resources return to baseline.

### 2. Synchronous Log Draining
One of the biggest challenges was ensuring logs weren't lost when a container finished. Initially, I ran the log streamer in a background goroutine, but I ran into a race condition: the main thread would delete the container before the log thread finished reading the last "Done!" message.

**The Fix:** I moved to a synchronous draining pattern. The worker blocks on the `ContainerLogs` stream, ensuring every byte is read and persisted to the host disk *before* the `ContainerRemove` command is ever issued.

## 💾 The "T-Junction" Persistence Strategy

In systems engineering, observability is everything. I needed to see logs in a dashboard while also saving them permanently to my local machine. I implemented a **T-Junction** in the log-streaming logic:

```go
// A simplified look at the logic
scanner := bufio.NewScanner(containerLogStream)
for scanner.Scan() {
    line := scanner.Text()
    
    // Path A: The Live Feed (Redis Pub/Sub)
    w.redis.Publish(ctx, "job:logs:"+id, line)
    
    // Path B: The Permanent Audit (Local Disk)
    logFile.WriteString(line + "\n")
}
```

By using a **Docker Bind Mount**, I mapped the worker’s internal `/app/logs` directory directly to my Mac’s filesystem. This allows the system to be truly distributed while keeping the data easily accessible for local debugging.



## 🛠️ Dealing with the Docker SDK "Mojibake"

A fun low-level discovery: Docker doesn't just send plain text logs. It sends a multiplexed stream with 8-byte headers (to distinguish between `stdout` and `stderr`). 

To handle this without adding heavy dependencies, the quickest fix was to enable `Tty: true` in the container config, which tells Docker to send a raw, un-headered stream as if its sending the logs to a terminal.
The ideal way (which I discovered after a bit of research) is to manually parse the 8-byte header to calculate frame length, so we know exactly how much data to read from the logs.

## 📈 Key Takeaways

I started building this with the scope of getting a simple glimpse into how task orchestrators generally work, but really this just reminded me of how much heavy lifting redis really does in all this, from being a task queue, to a hearbeat monitor, and streaming logs in real time without breaking a sweat. 

---