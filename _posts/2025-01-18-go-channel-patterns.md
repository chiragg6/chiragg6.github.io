---
layout: post
title: "Channel Patterns for Production Services"
date: 2025-01-18 09:45:00 +0530
categories: golang concurrency
tags: [golang, channels, pipelines, select, concurrency]
---

# Channel Patterns for Production Services

Channels are Go’s typed pipes. The language makes them look simple: `ch <- v` and `v := <-ch`. Production code uses a handful of **patterns**—generator, fan-in, fan-out, `select` with timeout, and explicit close protocols—not a new channel type for every problem.

Misuse is equally patterned: sending on a closed channel (panic), leaking goroutines blocked on send, and using a channel as a mutex because someone heard “don’t communicate by sharing memory” and stopped thinking.

This post is a field guide: patterns that scale, close/ownership rules, and when a mutex is the better tool.

---

## Ownership: Who Closes?

**The sender closes.** Receivers never close. Multiple senders need a coordinator (WaitGroup, then one closer) or a different design.

```go
func produce(n int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out) // sender owns close
        for i := 0; i < n; i++ {
            out <- i
        }
    }()
    return out
}

for v := range produce(10) {
    fmt.Println(v)
}
```

Returning a **receive-only** channel (`<-chan int`) encodes ownership in the type system. Callers cannot close or send.

Sending on a closed channel **panics**. Receiving from a closed channel returns the zero value and `ok == false`. `range` exits when the channel is closed and drained.

---

## Unbuffered vs Buffered

| Kind | Sync? | Use |
|------|--------|-----|
| Unbuffered | Send happens-before corresponding receive | Signaling, strict backpressure, handoff |
| Buffered (N) | Send proceeds until N items queued | Decouple bursts, avoid deadlock in known cyclic setups (carefully) |

A buffer of 1 is a common “signal with overwrite” building block. A buffer of 10,000 is usually a bug wearing a capacity.

Happens-before: a send on an unbuffered channel **synchronizes with** the receive. Buffered channels synchronize when the value is actually received, not merely queued (see the memory model).

---

## Generator / Pipeline

Each stage is a goroutine that reads an in channel and writes an out channel.

```go
func sq(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case <-ctx.Done():
                return
            case out <- n * n:
            }
        }
    }()
    return out
}
```

**Always take `ctx`.** Without it, a downstream consumer that stops ranging leaves upstream stages blocked on send forever.

---

## Fan-Out and Fan-In

**Fan-out:** multiple workers read the same `jobs` channel (competing consumers).

**Fan-in:** merge several channels into one.

```go
func merge(ctx context.Context, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

Fan-in **must** wait for all producers before `close(out)`, or a late send panics.

---

## `select`: Timeouts, Cancellation, Default

```go
select {
case v := <-results:
    handle(v)
case <-time.After(time.Second):
    return errTimeout
case <-ctx.Done():
    return ctx.Err()
}
```

`time.After` in a hot loop **leaks timers** until they fire. Prefer `time.NewTimer` and `Reset`, or `context.WithTimeout`.

`select` with `default` is non-blocking poll:

```go
select {
case ch <- v:
default:
    dropped++
}
```

Use that only when dropping is an explicit policy (metrics, logs). Silent drops are outages with extra steps.

---

## Done Channel vs Context

Older code uses `done <-chan struct{}`. New code should prefer **`context.Context`**: it already composes timeouts, parent cancel, and values. A done channel is still fine **inside** a package as an implementation detail; at API boundaries, take `ctx`.

---

## Channels as Mutexes

```go
sem := make(chan struct{}, 1)
sem <- struct{}{}
// critical section
<-sem
```

This works. `sync.Mutex` is clearer, faster for uncontended locks, and supports `defer Unlock()`. Use a mutex for **protecting memory**. Use a channel for **transferring ownership of a value** or **signaling an event**.

`sync.Cond` is rarely needed; a channel or mutex + broadcast pattern is easier to get right.

---

## Nil Channels in `select`

A nil channel is never ready. You can disable a case dynamically:

```go
var pause <-chan time.Time
if waiting {
    pause = time.After(delay)
}
select {
case <-pause:
    // ...
case v := <-work:
    // ...
}
```

This is a real technique in state machines. Accidental nil channels (forgot to initialize) look like mysterious hangs.

---

## Common Failures

1. **Goroutine leak:** send without receiver and without `ctx`.
2. **Close twice / close from receiver.**
3. **Unbuffered cyclic wait:** A waits on B’s channel, B waits on A’s—deadlock. A buffer of 1 is a bandage; restructuring is the fix.
4. **Using `<-ch` in a library that should take a callback or mutex-protected struct**—over-synchronizing simple state.

---

## Takeaways

- Encode **close ownership** in the type (`<-chan T` out of producers).
- Pipelines need **cancellation** or they leak.
- Fan-in needs a **WaitGroup** before close.
- Prefer **mutexes for shared state**, channels for **handoff and events**.
- Treat large buffers as **unbounded queues** until proven otherwise.

Channels are not a programming model by themselves. They are a primitive. The model is **ownership, lifetime, and backpressure**—the same as worker pools, just with a different shape.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
