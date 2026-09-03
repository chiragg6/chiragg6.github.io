---
layout: post
title: "Worker Pools and Backpressure in Go"
date: 2024-11-22 11:30:00 +0530
categories: golang concurrency scaling
tags: [golang, worker-pool, backpressure, channels, scaling]
---

# Worker Pools and Backpressure in Go

Spawning a goroutine per task is the default instinct in Go. It is also how you turn a traffic spike into a memory spike. A **worker pool** is a deliberate bottleneck: a fixed (or slowly adaptive) set of goroutines that pull work from a queue so that CPU, database connections, and downstream APIs stay within budget.

The missing half of the pattern is **backpressure**. If producers can enqueue faster than workers can drain, you have not bounded anything—you have moved the explosion into a channel buffer or an unbounded `slice` of jobs.

This post covers pool designs that hold up in production, how to apply backpressure, and when a pool is the wrong tool.

---

## Why Bound Concurrency

Unbounded `go process(job)` is fine when:

- Jobs are rare
- Each job is I/O and the OS/netpoller absorbs it
- Failure of one job must not wait on a semaphore

It is a bad default when:

- Each job opens a DB connection or an HTTP client call
- Jobs are CPU-heavy (JSON, crypto, image)
- A Kafka/SQS burst can deliver 100× normal rate
- You run in Kubernetes with a memory limit

A pool answers: **at most N units of this work run at once.**

---

## The Minimal Pool

```go
func runPool(ctx context.Context, n int, jobs <-chan Job) error {
    var wg sync.WaitGroup
    wg.Add(n)
    for i := 0; i < n; i++ {
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    handle(ctx, job)
                }
            }
        }()
    }
    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()
    select {
    case <-done:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

Producers send on `jobs` and **close it** when finished. Workers exit on close or cancel. Capacity of `jobs` is a second knob: `make(chan Job, 64)` absorbs jitter; `make(chan Job)` (unbuffered) applies the strongest backpressure—producers block until a worker is free.

---

## Backpressure Is a Feature

| Queue | Behavior |
|-------|----------|
| Unbuffered channel | Producer blocks until a worker receives. Strongest backpressure. |
| Small buffer | Smooths bursts; still blocks when full. |
| Huge / unbounded queue | Looks fast until RAM or GC dies. Hidden unbounded pool. |
| Drop newest / drop oldest | Protects latency; must be explicit and metered. |
| HTTP 429 / retry-after | Backpressure across process boundaries. |

If you use a message bus, **the bus is your queue**. A Go channel in front of Kafka that buffers 1M messages duplicates the broker and loses the point of the broker.

### HTTP handler example

```go
sem := make(chan struct{}, 32) // max in-flight expensive work

func handler(w http.ResponseWriter, r *http.Request) {
    select {
    case sem <- struct{}{}:
        defer func() { <-sem }()
    case <-r.Context().Done():
        return
    default:
        http.Error(w, "busy", http.StatusServiceUnavailable)
        return
    }
    doExpensive(r.Context())
}
```

`default` **fails fast** instead of queueing inside the process. Pair with load-balancer health checks and horizontal scale. Blocking on `sem <-` without `default` queues requests in the Go process—sometimes what you want, often not behind a reverse proxy that already queued them.

---

## `errgroup.SetLimit`

For request-scoped fan-out, a long-lived pool is overkill:

```go
g, ctx := errgroup.WithContext(r.Context())
g.SetLimit(8)

for _, id := range ids {
    id := id
    g.Go(func() error {
        return fetch(ctx, id)
    })
}
err := g.Wait()
```

`SetLimit` caps goroutines **inside that group**. Combined with a parent timeout, this is the usual pattern for “call N services, at most 8 at a time.”

---

## Dynamic Pools and Tuning

Fixed `N` is easier to reason about than auto-scaling workers inside one process. If load is diurnal, **scale pods**, not an in-process elastic pool that fights the orchestrator.

When you do tune `N`:

- **CPU-bound:** start near `GOMAXPROCS` or slightly above if there is a bit of I/O.
- **I/O-bound:** `N` can be much larger; the limit is usually **file descriptors**, **DB `max_connections`**, or **downstream RPS quotas**.
- Measure **queue wait time** and **worker busy ratio**. If wait time grows and workers are 100% busy, add capacity or shed load. If workers are idle and wait time is high, you have a lock or a slow dependency, not too few workers.

Export metrics:

- `jobs_enqueued_total` / `jobs_processed_total`
- `queue_depth`
- `worker_busy` (gauge)
- `job_duration_seconds` (histogram)

---

## Shutdown

Never kill workers mid-job if the job is not idempotent. Pattern:

1. Stop **accepting** new work (close listener, stop Kafka consume).
2. Close the jobs channel **or** cancel with a grace period.
3. `Wait` for in-flight jobs with a **hard deadline**; after that, log and exit.

```go
stopAccept()
close(jobs)
select {
case <-workersDone:
case <-time.After(30 * time.Second):
    log.Print("shutdown timeout; some jobs aborted")
}
```

---

## When Not to Use a Pool

- **`net/http` servers** already run a goroutine per request. Adding a pool *inside* every handler for tiny CPU work adds hops. Pool **shared scarce resources** (DB, rate-limited APIs), not “all code.”
- **Pipelines of stages** (see channel patterns) are often clearer than one mega-pool if stages have different concurrency needs.
- **Single-threaded invariants** (talking to a process that is not thread-safe) need **one** worker, not a pool.

---

## Takeaways

- A worker pool without backpressure is an unbounded queue with extra goroutines.
- Prefer **unbuffered or small buffers**, fail-fast at HTTP edges, and **let brokers be the buffer**.
- Use **`errgroup.SetLimit`** for scoped fan-out; use a long-lived pool for process-wide resource limits.
- Tune `N` from **dependency limits and metrics**, not from a blog’s favorite number.
- Shutdown is part of the pool contract: drain, then die.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
