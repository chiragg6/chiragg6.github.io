---
layout: post
title: "Prometheus Histograms, Cardinality, and Go Instrumentation"
date: 2026-06-20 12:45:00 +0530
categories: observability golang
tags: [prometheus, metrics, golang, cardinality, histograms]
---

# Prometheus Histograms, Cardinality, and Go Instrumentation

Prometheus is still the default metrics system next to Kubernetes. It is also how teams **accidentally DDoS their monitoring**: a label for `user_id`, a histogram bucket list copied from a blog, and a Go handler that uses `path` instead of `route`.

This post is about instrumenting Go services so **SLOs work**, **cardinality stays finite**, and histograms mean something at the gateway and in the app.

---

## Counter, Gauge, Histogram, Summary

| Type | Use |
|------|-----|
| **Counter** | Requests, errors, bytes—only goes up (per process) |
| **Gauge** | In-flight requests, goroutines, queue depth |
| **Histogram** | Latency, request size—**buckets**, aggregatable across pods |
| **Summary** | Quantiles **in the process**—cannot aggregate p99 across replicas correctly |

For fleet SLOs, use **histograms**, not summaries. Summaries are for a single process when you cannot afford histogram cost and do not need global p99.

---

## The Golden HTTP Metrics in Go

`prometheus/client_golang` + `promhttp` or `oklog`/`metrics` middleware. A minimal RED set:

```go
var (
    reqs = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "route", "code"},
    )
    dur = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Request duration",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "route"},
    )
)
```

**`route` must be a template:** `/orders/{id}` not `/orders/8492`. Chi/Gin/Echo all have a way to get the matched route. If you use the raw URL, cardinality = number of IDs.

**`code`** as `"200"` vs `"2xx"`: fine-grained is useful until you have 50 status codes × 40 routes × 4 methods. Collapse 2xx/4xx/5xx if needed.

---

## Histogram Buckets and SLOs

Your SLO is “99% of checkouts < 300 ms.” You need a bucket **at 0.3** (or native histograms in Prometheus 3 / remote write with native hist).

If buckets are `0.1, 0.5, 1` you cannot compute 300 ms SLI accurately—you only know “faster than 500 ms.”

**Native histograms** reduce the bucket tax. Until you run them everywhere, **custom buckets per SLO** beat default `prometheus.DefBuckets` (which are often too coarse or too wide for APIs).

---

## Cardinality Math

Time series ≈ `product(label values) × (1 + buckets for histograms)`.

Example disaster:

- `route` = raw path (10k IDs)
- `user_agent` = 2k strings
- histogram with 10 buckets

You just created tens of millions of series. Prometheus RAM explodes; Grafana dies; you page on **the monitoring system**.

Rules:

1. Bounded enums only: method, code class, route template, `db_op`.
2. No customer IDs, emails, full URLs, exception messages.
3. High-cardinality debug goes to **traces and logs**.
4. Relabel at scrape to **drop** mistakes you already shipped.

---

## Runtime and Process Metrics

Enable Go collectors: goroutines, GC, memstats. Add:

- `process_resident_memory_bytes`
- `go_sched_latencies_seconds` (newer Go)
- Custom: DB pool in-use / idle

These are **gauges/counters** with almost no labels. Cheap and essential for the scaling post’s “is it throttle or leak?”

---

## Exemplars

Prometheus histograms can attach **exemplars** (trace ID) to a bucket observation. Grafana then jumps to Tempo. In Go, client_golang supports exemplars when you have a sampled trace in context.

This is the bridge from **SLI burn** to **why**. Worth the setup time.

---

## Mesh and Gateway Double Counting

Envoy and Istio emit request metrics **per hop**. Summing `istio_requests_total` with `http_requests_total` from the app **double counts** if you blindly add them.

Dashboards:

- **Edge SLO:** gateway metrics only
- **Service:** app or inbound sidecar, pick one
- **Dependency:** outbound sidecar or client metrics

Name the panel after the **hop**.

---

## Recording Rules

Do not make Grafana compute `histogram_quantile(0.99, sum by (le, route) (rate(...[5m])))` on huge dashboards at 200 viewers. Use **recording rules** for SLO burn and per-route p99. That is operational hygiene, not premature optimization.

---

## Takeaways

Instrument Go with **templated routes**, **histograms that match SLOs**, and **labels you could list on a whiteboard**. Leave user-level detail to traces. Keep mesh/gateway/app metrics on **separate** charts. Cardinality is a production incident with extra steps.

A 99.9% SLO is only as good as the **denominator and the buckets**. Garbage labels make both lies.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
