---
layout: post
title: "Distributed Tracing from Request to Database"
date: 2026-02-27 17:35:00 +0530
categories: observability microservices
tags: [tracing, distributed-tracing, jaeger, tempo, opentelemetry, golang]
---

# Distributed Tracing from Request to Database

A user waited 2.4 seconds. Grafana shows the API gateway was “fine.” The orders service p99 is 80 ms. The database dashboard is a forest of averages. This is the gap **distributed tracing** fills: one **trace ID** from the browser (or mobile) through the gateway, service mesh, Go handler, SQL driver, and the hop to payments.

This post walks a single request hop by hop, then covers messaging, sampling, and how to read a trace without drowning.

---

## Anatomy of a Trace

```text
trace_id = 4bf92f3577b34da6a3ce929d0e0e4736

span: GET /checkout                 2400ms   gateway
 └─ span: HTTP POST /v1/orders      2100ms   orders-go
     ├─ span: SELECT orders         40ms     pgx
     ├─ span: redis GET cart        8ms
     ├─ span: HTTP POST /charge     1900ms   payments
     │   └─ span: stripe.capture    1800ms
     └─ span: kafka produce         15ms
```

Parent/child is **causality**. Clock skew between nodes can make children **appear** to start before parents; backends try to correct; do not treat timestamps as legal evidence.

**Span kinds:** `SERVER`, `CLIENT`, `INTERNAL`, `PRODUCER`, `CONSUMER`. Getting kind right makes service maps usable.

---

## Hop 1: The Edge

The gateway should:

1. Accept inbound `traceparent` if present (real user monitoring, mobile).
2. **Start a span** for the inbound request if missing.
3. Inject `traceparent` on the upstream request.

If the gateway generates a new trace **always**, you **break** RUM-to-backend correlation. If it **never** generates one, untraced clients stay invisible.

Mesh sidecars can add **proxy spans** (inbound/outbound). They show queueing and TLS time. They will not show `discount.compute` inside your process.

---

## Hop 2: The Go Server

`otelhttp` creates a **SERVER** span. Your handler must pass `r.Context()` into:

- `http.NewRequestWithContext` for other HTTP APIs
- `QueryContext` / `pgx` tracing
- Kafka producers that accept headers

A worker `go func()` that uses `context.Background()` **detaches**. If the work is still part of the request, derive from `r.Context()`. If it is fire-and-forget after response, start a **new** trace or link spans explicitly (`span.AddLink`) so you do not extend client timeouts forever.

---

## Hop 3: The Database

Instrument the driver:

- **database/sql**: `otelsql` or similar wrappers
- **pgx**: hooks / tracer interfaces
- **Redis**: `redisotel`

Attributes: `db.system`, `db.name`, sanitized `db.statement` (or operation name only). **Never** put raw bound parameters with PII in spans.

N+1 queries show up as a **comb** of tiny SQL spans under one handler span. That is tracing earning its keep.

Connection pool wait should be visible (custom span or metrics). A 1.9 s “DB” that is actually **pool exhaustion** looks like Postgres was slow if you only have query time.

---

## Hop 4: Async Messaging

HTTP is easy: headers. Kafka/NATS/SQS need **explicit** propagation:

- Inject context into **message headers** on produce
- Extract on consume; start a **CONSUMER** span as child (or a linked trace if you do not want 10-minute traces for async)

**Decision:** should a checkout trace include the email worker 30 seconds later? Usually **no**—use **links** or a new trace with `causation` IDs in logs. Long traces are hard to sample and store.

---

## Reading Traces in an Incident

1. Sort by **duration**; open the slowest trace, not a random one.
2. Find the **critical path** (the longest child chain), not the busiest service.
3. Look at **span status** and events (`exception`).
4. Compare a **fast** trace of the same `http.route` — what extra spans exist?

Service maps lie when traffic is sparse or names are wrong (`HTTP GET` without route templates explodes cardinality: `/orders/1`, `/orders/2`). Use **low-cardinality route names** (`/orders/{id}`).

---

## Sampling Strategy

| Goal | Strategy |
|------|----------|
| Cheap overview | Head sample 1–5% |
| Debug errors | Tail sample: keep 100% of errors |
| Debug slowness | Tail sample: latency > SLO |
| Launch / incident | Temporarily raise sample rate with a known cost |

Head sampling at 1% **misses rare bugs** unless tail sampling or error-biased rules exist.

---

## Backends

**Jaeger**, **Grafana Tempo**, **Honeycomb**, **Datadog**, **Elastic APM** all speak OTLP nowadays. Tempo + Grafana is a common OSS pair: traces next to metrics. The backend matters less than **propagation being correct**. An empty Jaeger with perfect Grafana metrics is still an uninstrumented app.

---

## Takeaways

A useful trace is a **connected tree** from gateway to DB and outbound HTTP, with **sanitized** attributes and **templated** route names. Async work should be **linked**, not always parented. Sampling should **prefer errors and slowness**.

If you only instrument the Go handler and not the SQL client, you will keep blaming the database from a span that is actually **everything after `ListenAndServe`**.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
