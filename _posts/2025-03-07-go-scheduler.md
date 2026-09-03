---
layout: post
title: "Inside the Go Scheduler: GOMAXPROCS, Syscalls, and Latency"
date: 2025-03-07 16:20:00 +0530
categories: golang concurrency scaling
tags: [golang, scheduler, gomaxprocs, performance, scaling]
---

# Inside the Go Scheduler: GOMAXPROCS, Syscalls, and Latency

When a Go service is “slow,” people reach for more replicas or more `go` statements. Often the bottleneck is the **runtime scheduler**: too few Ps for the work shape, too many Ms stuck in syscalls, or a fight between `GOMAXPROCS` and a Kubernetes CPU limit.

You do not need to read the runtime source to operate a service, but you do need a mental model of **what runs in parallel**, **what parks**, and **what creates extra OS threads**. This post is that model, plus practical tuning.

---

## Recap: G, M, P

- **G** — goroutine
- **M** — OS thread
- **P** — logical processor; `GOMAXPROCS` ≈ number of Ps

A G runs only when an M is bound to a P. If all Ms that have Ps are in **blocking syscalls**, the runtime creates or wakes other Ms so idle Ps can keep running Go code. That is why a burst of file I/O can balloon thread count.

```text
GOMAXPROCS = 4

P0 P1 P2 P3     ← at most 4 goroutines in Go code at once
│  │  │  │
M  M  M  M      ← typically 4 threads executing Go

+ extra M's     ← blocked in syscall / cgo
```

---

## What Counts as a Blocking Syscall

The net poller handles **non-blocking network sockets**. A `Read` on a TCP connection usually **parks the G** and frees the M.

These typically **block the M**:

- File I/O (`os.ReadFile` on a large disk file)
- DNS in some configurations (less true with the pure Go resolver)
- cgo calls
- Some syscalls (`flock`, synchronous disk)

If 200 goroutines call `os.ReadFile` at once, you can see hundreds of OS threads. That can hit `ulimit -u` or make the node thrash.

Mitigations: bound concurrency, use `io_uring`/specialized libraries where it matters, or move heavy disk work to a dedicated process.

---

## `GOMAXPROCS` and Containers

Historically, Go saw **all host CPUs** even inside a container with `cpu: 500m`. The runtime would create many Ps, Linux CFS would give you half a core, and you would get **high tail latency** from oversubscription.

Recent Go versions (1.18+ improvements, then **cgroup-aware `GOMAXPROCS`** in later releases) can set P count from **CPU quota**. Still verify:

```bash
go env GOMAXPROCS
# or in process:
runtime.GOMAXPROCS(0)
```

In Kubernetes, prefer:

- Setting requests/limits honestly
- Using a Go version that respects cgroups
- Optionally `runtime.GOMAXPROCS` from an operator like `automaxprocs` (Uber) if you are on an older Go

**Requests vs limits:** if limit is 1 CPU and `GOMAXPROCS=8`, you scheduled like an 8-core machine on 1 core of CFS. Latency suffers.

---

## Fairness, Preemption, and GC

Go 1.14 added **asynchronous preemption**. Tight loops can be interrupted so other goroutines and GC can run. You should still:

- Check `ctx.Err()` in long CPU loops
- Avoid holding locks across huge computations
- Remember GC stop-the-world is short but not zero; huge heaps still cost

`GOGC` and GOMEMLIMIT (Go 1.19+) interact with latency. A memory-constrained pod without `GOMEMLIMIT` can GC thrash. That is scheduler-adjacent: mutator (your goroutines) and GC compete for the same Ps.

---

## Scheduler Traces

When you need evidence:

```bash
GODEBUG=schedtrace=1000 ./app
```

Prints runqueue lengths, syscall counts, etc. For deeper work:

```bash
go tool trace trace.out
```

Look for:

- Long **syscall** time
- Ps idle while work exists (rare; usually a lock)
- GC STW
- Incoming network stuck (netpoller)

`pprof` profiles (`cpu`, `block`, `mutex`, `goroutine`) answer different questions. CPU pprof will not show a G parked on a channel as “hot”—block profile will.

---

## Practical Tuning Checklist

1. **Match `GOMAXPROCS` to effective CPUs** (quota, not node size).
2. **Bound** file I/O, cgo, and “one goroutine per file” patterns.
3. **Do not** raise `GOMAXPROCS` to “use more cores” if the app is I/O bound—add concurrency (goroutines) instead, with limits.
4. **Watch** `go_sched_latencies` / runtime metrics (Go 1.16+ `/debug/metrics` or OpenTelemetry runtime instrumentation).
5. **Pinning:** `runtime.LockOSThread` is for thread-local OS APIs (some GUI, some GPU). It reduces scheduler freedom; do not use it for speed.

---

## Latency vs Throughput

- **Throughput:** keep Ps busy with useful work; avoid excessive syscalls and lock contention.
- **Tail latency:** avoid oversubscription vs CFS; avoid GC storms; avoid convoy effects on a single mutex; keep queues short (backpressure).

Spinning up more goroutines increases **concurrency**, not automatically **parallelism**. Parallelism is capped by Ps (and by the kernel).

---

## Takeaways

The Go scheduler is a work-stealing multiplexor of Gs onto a small number of Ms, with extra threads for blocking syscalls. **`GOMAXPROCS` is the parallelism knob**; goroutine count is the concurrency knob. In Kubernetes, those knobs must match **CPU quota**, or you pay in p99.

If you remember one operational fact: **network is cheap in goroutine count; blocking syscalls and cgo are not.** Bound the latter.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
