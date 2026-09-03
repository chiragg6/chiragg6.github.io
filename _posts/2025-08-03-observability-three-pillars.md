---
layout: post
title: "The Three Pillars of Observability"
date: 2025-08-03 12:25:00 +0530
categories: observability sre cloud-native
tags: [observability, metrics, logs, tracing, sre, prometheus]
---

# The Three Pillars of Observability

**Observability** is the ability to explain a system’s behavior from its **external outputs**—without SSHing into a box and hoping. The industry shorthand is three pillars: **metrics**, **logs**, and **traces**. They are not competitors. They answer different questions, and a mature platform **correlates** them (same `trace_id`, same `pod`, same `deployment`).

This post is the mental model I use when someone asks for “more logging” during an incident that actually needed a histogram.

---

## What Each Pillar Is For

| Pillar | Good at | Weak at |
|--------|---------|---------|
| **Metrics** | SLOs, alerts, capacity, cheap cardinality-controlled numbers | Explaining *one* bad request |
| **Logs** | Errors, audit, payload-level “what happened” | High-QPS cost; unstructured search as a database |
| **Traces** | Latency breakdown across services | Completeness unless you sample; storage cost |

**Events / continuous profiling** are sometimes called extra pillars. Profiling answers “why is CPU 90%.” Keep the first three solid before buying a fourth vendor.

---

## Metrics: The Nervous System

Use **RED** for request-driven services:

- **Rate** — requests per second
- **Errors** — failed requests (define failure: 5xx, not 4xx unless you mean it)
- **Duration** — latency histogram, not a single average

Use **USE** for resources: Utilization, Saturation, Errors (disk, CPU, queues).

**Prometheus** + **histograms** (or native histograms) beat “time this in the app and log it” for SLOs. Cardinality is the enemy: do not put `user_id` on a metric label. Put it on a **trace** or a **sampled log**.

Alert on **symptoms** (SLO burn, error rate, saturation), not on “pod restarted” unless that is a reliable signal. Alerting on every 5xx in a 1-replica hobby app is fine; in a 200-service platform you need **multi-window burn rates**.

---

## Logs: Narrative and Evidence

Logs are for:

- Exceptions with stack traces
- Audit (“who deleted the order”)
- Debugging **after** metrics told you *when* and traces told you *where*

Rules that save money and sanity:

1. **Structured JSON** (`level`, `msg`, `trace_id`, `span_id`, `error`).
2. **No secrets.** Tokens, PAN, PII—redact at the logger.
3. **Levels:** `error` is pageable; `debug` is not on in production by default.
4. **Index sparingly.** Object storage + a query engine (Loki, Elastic with ILM, ClickHouse) beats infinite hot retention.

A log line without `trace_id` in a microservices world is a dead end.

---

## Traces: The Request’s Biography

A **trace** is a tree of **spans**: gateway → service A → DB → service B. Each span has timing, status, and attributes (`http.status_code`, `db.statement` truncated).

You need:

- **Context propagation** (W3C `traceparent` / `tracestate`) through HTTP, gRPC, and message queues
- **Instrumentation** (OpenTelemetry) at frameworks and clients
- A **backend** (Jaeger, Tempo, Honeycomb, Datadog, etc.)

**Head sampling** (decide at the start) is simple and biased toward whatever started. **Tail sampling** (keep errors and slow traces) is better for incidents and more operationally complex.

Traces without **exemplars** linking a Prometheus histogram bucket to a trace ID leave you doing time-range archaeology.

---

## Correlation: The Actual Superpower

Incident flow that works:

1. **Alert** fires on SLO burn (metrics).
2. Dashboard shows **which** operation and **which** dependency (metrics + RED per service).
3. Jump to **traces** for that service/operation in the window (exemplars or trace search).
4. Open **logs** for the failing span’s `trace_id`.

If those IDs do not line up, you have three tools, not an observability system.

In Kubernetes, always attach: `service`, `version`/`commit`, `pod`, `namespace`, `region`. Mesh proxies add hop-level spans if configured—useful, not a substitute for app spans inside handlers.

---

## Sampling and Cost

| Signal | Typical production stance |
|--------|---------------------------|
| Metrics | Always on; control labels |
| Traces | Sample (1%–10%+) + always keep errors/slow |
| Logs | Sample debug; keep errors; rate-limit noisy paths |

“Log every request body” is GDPR and SSD in one decision.

---

## Observability vs Monitoring

**Monitoring** asks: is it broken? (known questions, dashboards, alerts.)

**Observability** asks: why is this novel failure happening? (high-cardinality events, traces, ad-hoc queries.)

You need both. A beautiful Grafana with no traces still leaves you guessing across 30 services.

---

## Takeaways

Metrics for **health and SLOs**, logs for **evidence**, traces for **causality across hops**. Correlate with **trace IDs** and resource attributes. Spend money on **cardinality discipline** and **tail sampling**, not on logging `fmt.Sprintf("%+v", req)` in the hot path.

If you can only instrument one thing this quarter: **RED metrics + `traceparent` through the gateway and your Go HTTP clients.** That pair finds most production fires.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
