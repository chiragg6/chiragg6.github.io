---
layout: post
title: "Scaling Go Services: From One Binary to Many Pods"
date: 2025-05-12 08:30:00 +0530
categories: golang scaling kubernetes
tags: [golang, scaling, kubernetes, performance, concurrency]
---

# Scaling Go Services: From One Binary to Many Pods

Go’s reputation is “we rewrote it and used 10% of the RAM.” That is often true for a **single** process. Scaling a Go service in Kubernetes is a different problem: **horizontal replicas**, **CPU limits vs GOMAXPROCS**, **connection pools**, **in-memory caches that do not share**, and **graceful shutdown** so you do not drop in-flight work.

This post is the checklist I use when a Go API needs to go from one laptop binary to a fleet.

---

## Vertical vs Horizontal

**Vertical:** bigger machine, higher `GOMAXPROCS`, bigger caches. Hits a wall (NUMA, GC, blast radius).

**Horizontal:** more pods. Go makes this easy because processes are **share-nothing** by default. That is also the trap: **in-memory rate limits, caches, and leader locks** do not add up across pods unless you designed them to.

Scale **stateless** handlers horizontally. Scale **state** with Redis, DB, or sticky sessions only if you must (you must not, usually).

---

## The Request Path Budget

A replica’s capacity is approximately:

```text
RPS ≈ min(
  GOMAXPROCS-bound CPU work,
  memory / per-request alloc,
  DB pool size,
  downstream RPS quota,
  file descriptors / 2
)
```

Adding pods increases **aggregate** RPS only if the **bottleneck is not shared**. 20 pods × 10 DB connections = 200 connections. Postgres `max_connections` and pgbouncer matter more than goroutines.

**Bound:**

- `http.Server` timeouts (`ReadTimeout`, `WriteTimeout`, `IdleTimeout`)
- Downstream clients (`Timeout`, `MaxConnsPerHost`)
- `database/sql` `SetMaxOpenConns` **per pod**, sized so `pods × max_open` fits the DB
- Fan-out with `errgroup.SetLimit`

---

## Kubernetes CPU and Go

- **Requests** are scheduler guarantees; **limits** throttle with CFS.
- Go + CFS limits without matching `GOMAXPROCS` → **latency spikes**.
- Memory: set `GOMEMLIMIT` near the pod limit (minus a safety margin) so GC works **with** the cgroup instead of OOMing.

Readiness vs liveness:

- **Readiness:** fail when the process should not get traffic (not warmed, draining).
- **Liveness:** fail only when a restart will help (deadlock). Do not liveness-probe a downstream DB; you will kill healthy pods in a DB outage.

---

## Graceful Shutdown

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go func() { _ = srv.ListenAndServe() }()

<-ctx.Done() // SIGTERM
shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx)
```

Order:

1. **Fail readiness** (or remove from Service) so kube-proxy/EndpointSlice stops new traffic.
2. **Wait** a moment for propagation.
3. **`Shutdown`**: stop accepts, wait in-flight.
4. Exit before the pod **grace period** (often 30s). Mesh sidecars need **extra** time (`terminationDrainDuration`, `preStop` sleep) or you cut connections while Envoy is still up.

`Kill` without `Shutdown` is how you get 502s on every deploy.

---

## Caching and Multi-Pod

| Cache | Multi-pod behavior |
|-------|-------------------|
| Process map | N independent caches; stampede on deploy |
| LRU in-process | Same |
| Redis | Shared; add timeouts and single-flight |
| CDN | Shared at edge |

**Singleflight** (`golang.org/x/sync/singleflight`) prevents one pod from thundering a DB. It does **not** coordinate 50 pods. For that: lock in Redis, or request coalescing at the gateway, or cache TTL jitter.

---

## Load Shedding

When overloaded, **fail fast**:

- Semaphore on expensive handlers (see worker pool post)
- Gateway 429 / 503
- Mesh circuit breakers

A Go server that queues unbounded `http` goroutines will **OOM** or GC-death before the load balancer understands. `MaxConnsPerHost` and server-side limits are production features.

---

## What to Measure

- RPS, p50/p99, error rate (RED)
- `go_goroutines`, `go_memstats_heap_alloc_bytes`
- GC pause (runtime metrics)
- DB pool wait
- Scheduler latency if you have Go 1.16+ metrics
- Pod CPU throttle (`container_cpu_cfs_throttled_seconds`)

If throttle is high, your Go “CPU problem” is **the limit**, not the algorithm.

---

## Takeaways

Scale Go by **bounding resources per process** and **adding replicas** for stateless work. Align **GOMAXPROCS** and **GOMEMLIMIT** with cgroups. Size **connection pools** for the fleet, not the pod. **Shutdown** with the mesh in mind. Do not pretend in-memory state is a cluster.

Goroutines scale concurrency inside a pod. Kubernetes scales pods. You need both knobs, and you need them not to fight.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
