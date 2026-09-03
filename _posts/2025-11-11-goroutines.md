---
layout: post
title: "Deep Dive into Goroutines"
date: 2025-11-11 12:00:00 +0530
categories: golang concurrency
tags: [golang, goroutines, scheduler, concurrency, scaling]
---

# Deep Dive into Goroutines

Go’s marketing line is “cheap threads.” That undersells what a **goroutine** actually is. It is a user-space, cooperatively scheduled unit of work with a tiny initial stack that grows and shrinks as needed. You can spawn tens of thousands of them on a laptop. You can also leak them, starve them, or pin them to OS threads without noticing—until production latency explodes.

This post is about how goroutines really work: stacks, the G/M/P model, when they block, how they scale, and how to keep them from becoming the bug you debug at 2 a.m.

---

## What a Goroutine Is (and Is Not)

A goroutine is **not** an OS thread. Creating an OS thread costs kilobytes of kernel structures and a large fixed stack (often 1–8 MiB). A goroutine starts with a stack on the order of **a few kilobytes** (historically 2 KiB; current versions are in that neighborhood) and is multiplexed onto a smaller pool of OS threads by the Go runtime.

```go
go func() {
    doWork()
}()
```

That `go` statement:

1. Allocates a **G** (goroutine object) and an initial stack.
2. Puts it on a run queue.
3. Returns immediately to the caller.

There is no `join` in the language. If you need to wait, you use channels, `sync.WaitGroup`, `errgroup`, or similar. Forgetting to wait is how you ship “it works on my machine” and then drop work on shutdown.

---

## The G, M, P Model

The scheduler is built from three letters:

| Letter | Name | Role |
|--------|------|------|
| **G** | Goroutine | The user-level task: stack, program counter, status |
| **M** | Machine | An OS thread that executes Go code |
| **P** | Processor | A logical processor; holds a local run queue and scheduler state |

`GOMAXPROCS` (default: number of CPU cores) is the count of **P**s. At most that many goroutines run Go code in parallel. Extra Ms exist for blocking syscalls, cgo, and the network poller.

```text
          ┌──────── P0 ────────┐     ┌──────── P1 ────────┐
          │ local run queue    │     │ local run queue    │
          │ G G G G            │     │ G G                │
          └────────┬───────────┘     └────────┬───────────┘
                   │                          │
                   ▼                          ▼
                  M0                         M1
              (OS thread)                (OS thread)
                   │                          │
                   ▼                          ▼
              running G                  running G
```

**Work stealing:** if a P’s local queue is empty, it steals half the goroutines from another P, or takes from the **global** run queue. That is how load spreads without a giant lock on every `go` statement.

---

## Stacks: Grow, Shrink, Don’t Guess

Each goroutine has its own stack, separate from the OS thread stack.

- **Growth:** when a function call would overflow the stack, the runtime copies the stack to a larger contiguous segment (or uses stack splitting historically; modern Go uses **contiguous copy**).
- **Shrink:** stacks can shrink after GC if usage dropped.
- **Pointers:** the runtime rewrites pointers into the stack during copy. That is why you cannot take the address of a Go variable and pass it to C without care, and why stack-allocated values move.

Deep recursion or huge stack frames (`var buf [1 << 20]byte` in a hot function) blow stacks and can OOM a process that looked “fine” with a few goroutines.

Prefer heap allocation for large buffers (`make([]byte, n)`) so they live on the heap, not on every goroutine’s stack.

---

## When a Goroutine Stops Running

Go is **cooperatively** scheduled at well-defined points, plus **asynchronous preemption** (Go 1.14+):

- Function calls (stack checks)
- Channel operations, mutexes, `select`
- Garbage collection safe points
- Syscalls and cgo
- Preemption signals for tight loops that never call anything (the runtime can stop a G that has been running too long)

A tight loop that only does integer math **used to** starve the scheduler and GC. Preemption largely fixed that, but you should still not spin forever without checking a context or a stop channel.

### Blocking vs parking

| Event | What happens |
|-------|----------------|
| Channel send/recv (unbuffered, empty/full) | G is **parked**; M can run another G |
| `time.Sleep`, timers | G parked; woken by the timer heap |
| Network I/O (`net` package) | Registers with the **netpoller**; G parks; M is free |
| Blocking syscall (`read` on a file, cgo) | M is blocked; runtime may **spin up another M** so other Ps keep running |
| `sync.Mutex` contention | G parks on the mutex wait list |

The netpoller is why 50,000 idle HTTP connections can sit in goroutines without 50,000 OS threads. File I/O and cgo do **not** get that gift for free.

---

## Scaling: How Many Is Too Many?

Goroutines are cheap, not free.

Each G has:

- A `g` struct in the runtime
- A stack (grows)
- Presence on queues and GC stacks

A million idle goroutines is a memory experiment, not a design. Production services usually scale with:

- **One goroutine per connection** for network servers (idiomatic `net/http`)
- **Bounded worker pools** for CPU work or bounded downstream concurrency
- **`errgroup` with a limit** (`SetLimit`) so fan-out cannot explode

```go
g, ctx := errgroup.WithContext(parent)
g.SetLimit(32) // at most 32 goroutines in this group

for _, item := range items {
    item := item
    g.Go(func() error {
        return process(ctx, item)
    })
}
return g.Wait()
```

If every request fans out to 200 backends with no limit, you have a thundering herd with extra steps.

---

## Goroutine Leaks

A leak is a goroutine that never returns. Typical causes:

1. **Blocked on a channel** nobody will send on or close.
2. **Waiting on `ctx.Done()`** that is never canceled (forgot `cancel()`).
3. **Blocked on mutex** held by a deadlocked owner.
4. **HTTP client** whose body is never closed (`resp.Body.Close()`), keeping the connection and goroutine alive.

```go
// leak: nobody receives
ch := make(chan int)
go func() {
    ch <- 1 // blocks forever if unbuffered and no receiver
}()
```

Debug with:

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=1
```

Or `runtime/pprof` and `go tool pprof`. Look for stacks stuck in `chan receive`, `select`, or your worker loop.

---

## GOMAXPROCS, Parallelism, and Latency

- **Concurrency** = many goroutines in flight (I/O).
- **Parallelism** = goroutines actually running on different cores at once.

Raising `GOMAXPROCS` past the number of cores rarely helps CPU-bound work and can increase scheduler overhead. Lowering it on a noisy neighbor node (Kubernetes CPU limits) can **reduce** latency variance because the Go runtime and the Linux CFS scheduler fight less.

`GOMAXPROCS` is now also aware of **cgroup CPU limits** in recent Go versions: the runtime can set P count from the container quota. Know which Go version you ship.

---

## Practical Rules

1. **Start a goroutine only when you know who waits for it** (or it is a process-lifetime background loop with a shutdown path).
2. **Always propagate `context.Context`** into work you spawn from a request.
3. **Bound concurrency** at the edge of fan-out.
4. **Close channels from the sender**; never close from multiple goroutines.
5. **Do not use goroutines to “make it faster”** around a mutex-protected map—measure first.
6. **Profile goroutine count** (`runtime.NumGoroutine()`) as a metric. A slow climb is a leak.

---

## Mental Model

```text
go f()  →  new G on a P’s run queue
                │
                ▼
         M executes G until park / preempt / done
                │
                ├─ I/O  → netpoller wakes G later
                ├─ syscall → extra M may be created
                └─ return → G recycled, stack freed
```

Treat goroutines as **work with a lifetime**, not as fire-and-forget threads. The runtime will multiplex them brilliantly if you give them a way to finish.

---

## Conclusion

Goroutines are the unit of concurrency in Go because they are **cheap to create, cheap to switch, and expensive only when they leak or unbounded-fan-out**. The G/M/P scheduler, growable stacks, and the netpoller are why a single binary can hold tens of thousands of connections.

Write every `go` statement with an exit condition, a bound, and a context. That is the difference between “Go scales” and “we have 400,000 goroutines and no idea why.”

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
