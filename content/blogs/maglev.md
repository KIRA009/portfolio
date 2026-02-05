---
title: "Building Google’s Maglev Hashing in Go: A Weekend Deep Dive" 
date: 2026-02-05
tags: ["golang", "distributed-systems", "load-balancing", "maglev", "consistent-hashing", "high-availability"]
description: "Build Google’s Maglev hashing in Go. Learn to implement a weighted, $O(1)$ lookup table with minimal disruption and lock-free atomic updates."
summary: "An implementation of Google's Maglev consistent hashing in Go, using fixed-size lookup to solve the \"thundering herd\" problem."
showToc: false
disableAnchoredHeadings: false
githubLink: "https://github.com/KIRA009/go-implementations/tree/main/maglev"

---

I’ve been obsessed with how giant systems handle traffic lately. If you have ten million users and a hundred backend servers, how do you decide which user goes to which server? 

The "obvious" answer is `hash(user_id) % number_of_servers`. It works great until one server dies. Suddenly, your "number of servers" changes from 100 to 99, the math shifts for every single user, and your entire cache layer is nuked because everyone is being sent to a new home. 

That led me to **Maglev**, Google’s custom-built distributed load balancer. I decided to spend the weekend implementing it in Go from scratch. Here’s the breakdown of how it works and how I built it.

---

## The Core Problem: The "Thundering Herd"
Traditional consistent hashing usually involves a "Ring." You map nodes and keys onto a circle ($0$ to $2^{64}-1$) and find the nearest neighbor. It’s elegant, but it has two annoying properties: it’s $O(\log N)$ to find a node because you have to binary search the ring, and it’s notoriously hard to get a perfectly "flat" distribution without adding thousands of "virtual nodes."

Maglev takes a different path. It uses a **fixed-size lookup table**. It’s faster ($O(1)$ lookup) and spreads the load with almost surgical precision.

---

## The Game Plan
My implementation plan had three distinct phases:
1.  **Preference Generation:** Every backend generates a unique "permutation" (a list of every slot in the table, ordered by how much it wants to sit there).
2.  **The Interleaving Race:** The backends "compete" for slots in the table until it’s full.
3.  **The Atomic Swap:** Ensuring we can update the table live without dropping packets.

---

## Phase 1: Generating the Permutations
Each backend needs a unique way to traverse the table. We generate two values for each server using its name: an **Offset** (where it starts) and a **Skip** (how far it jumps if a seat is taken).



In `pkg/hash.go`, I used FNV-1a hashing. The math ensures that every backend eventually visits every single slot in the table, but in a different order than its peers.

```go
func (m *Maglev) getOffsetAndSkip(name string) (uint64, uint64) {
    h := getHash(name)
    offset := h % m.m
    skip := (h % (m.m - 1)) + 1 // Skip can't be 0
    return offset, skip
}
```

---

## Phase 2: The "Interleaving" Algorithm
This is the clever part. We don't just let Backend A fill its favorite spots and then let Backend B pick the leftovers. That would be unfair. Instead, they take turns in a round-robin fashion.

Imagine a row of 65,537 empty chairs. Backend 0 picks its #1 choice. Then Backend 1 picks its #1 choice. If someone's favorite spot is already taken, they skip to their next preference using their `skip` value.



```go
for filled < m.m {
    for i := 0; i < m.n; i++ {
        // Each backend tries to find its next preferred empty slot
        c := m.permutation[i][next[i]]
        for m.lookupTable[c] != -1 { 
            next[i]++
            c = m.permutation[i][next[i]]
        }
        m.lookupTable[c] = i // Claim the seat!
        next[i]++
        filled++
        if filled == m.m { return }
    }
}
```

---

## Phase 3: Handling Weights and "Hot Swaps"
Real-world servers aren't equal. To handle this, I modified the loop so that a "heavy" backend can take two or three turns in a single round. This allows a beefy server to occupy more slots in the table proportionally.

But the biggest challenge in Go is **concurrency**. If I’m updating the list of servers (maybe a health check failed) while 10,000 requests per second are hitting the balancer, I can't just mutate the table in place. That's a one-way ticket to a race condition panic.

I used `atomic.Pointer` from the `sync/atomic` package. We build the new table entirely in the background, and once it's ready, we perform a single-instruction pointer swap.

```go
func (lb *LoadBalancer) Get(key string) string {
    // Atomically load the pointer so we don't race during an update
    return lb.current.Load().Get(key)
}
```

---

## Does it actually work?
I ran a simulation with 5 backends and 100,000 dummy requests. I gave `backend-4` double the weight of the others.

**The results were spot on:**
* **Standard Backends (Weight 1):** ~16.6% traffic each.
* **Weighted Backend (Weight 2):** ~33.3% traffic.

The "Disruption" test was the real clincher. When I removed the weighted backend, only **32.8%** of the keys were remapped to new servers. In a naive modulo system, almost **100%** of traffic would have shifted. This "minimal disruption" is exactly what keeps databases and caches from melting down during a rollout.

---

## Takeaways
Maglev is surprisingly elegant. By trading a tiny bit of memory (to store the lookup table) and some startup time (to build the permutations), you get a load balancer that is $O(1)$, perfectly fair, and weighted.

If you’re building high-performance Go services, don't just reach for a simple modulo. Consistent hashing—specifically the Maglev flavor—is a tool you definitely want in your belt.

**What’s next?** I’m looking into calculating the permutations on-the-fly to save RAM, and maybe adding active health-check probes. But for now, the table is full and the traffic is flowing perfectly.