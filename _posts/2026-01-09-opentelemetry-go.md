---
layout: post
title: "OpenTelemetry in Go Services"
date: 2026-01-09 10:00:00 +0530
categories: observability golang
tags: [opentelemetry, golang, tracing, metrics, instrumentation]
---

# OpenTelemetry in Go Services

**OpenTelemetry (OTel)** is the vendor-neutral way to produce **traces, metrics, and logs** from applications. In Go, that means the `go.opentelemetry.io/otel` family: a **TracerProvider**, a **MeterProvider**, context propagation, and **instrumentation libraries** for `net/http`, gRPC, and databases.

The point is not “use OTel because CNCF.” The point is **one** set of spans and resource attributes that can go to Jaeger this year and Grafana Cloud next year without rewriting handlers.

This post is a practical Go setup: providers, HTTP, exporters, and mistakes that silently drop traces.

---

## Core Ideas

- **API** (`otel.Tracer`, `span.SetStatus`) is stable-ish; you depend on this in app code.
- **SDK** (BatchSpanProcessor, resource, sampler) lives in `main` / a `telemetry` package.
- **Instrumentation** wraps frameworks so you do not start a span in every handler by hand (you still add **business** spans).
- **Exporter** sends OTLP to a **collector** (recommended) or directly to a backend.

```text
Go app  --OTLP gRPC/HTTP-->  OTel Collector  -->  Tempo / Prometheus / Loki
```

The collector is where you **tail-sample**, scrape Prometheus, and redact attributes. Putting vendor SDKs in every service is how you get seven ways to name `http.status_code`.

---

## Bootstrap in `main`

```go
func setupOTel(ctx context.Context) (shutdown func(context.Context) error, err error) {
    res, err := resource.Merge(
        resource.Default(),
        resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("orders"),
            semconv.ServiceVersion(version),
        ),
    )
    if err != nil {
        return nil, err
    }

    exp, err := otlptracegrpc.New(ctx)
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exp),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return tp.Shutdown, nil
}
```

Always **`Shutdown`** on process exit so the batch processor flushes. Kubernetes `SIGTERM` + ignored shutdown = missing tail of traces during deploys.

`ParentBased` + ratio: if an upstream (gateway) already sampled, you respect it. That is how a 1% mesh still traces a full tree.

---

## HTTP: Incoming and Outgoing

Incoming:

```go
handler := otelhttp.NewHandler(mux, "orders-http")
```

Outgoing: wrap the transport:

```go
client := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
```

Use **`http.NewRequestWithContext`** and pass **`ctx`** so the child span links to the incoming request. `http.Get` without context **breaks the trace**.

gRPC: `otelgrpc` stats handlers / interceptors. Same rule: **context is the chain**.

---

## Spans You Should Add Yourself

Auto-instrumentation gives you `HTTP GET /v1/orders/{id}`. You add:

```go
ctx, span := tracer.Start(ctx, "discount.compute")
defer span.End()
span.SetAttributes(attribute.String("discount.campaign", id))
```

Do not turn every function into a span. Span **boundaries** that match **SLO operations** and **dependencies** (Redis, SQL, Kafka produce). Too many spans are expensive and unreadable.

On errors:

```go
span.RecordError(err)
span.SetStatus(codes.Error, err.Error())
```

---

## Metrics with OTel vs Prometheus

Go services often still **export Prometheus** `/metrics` because Kubernetes and Grafana are built around scrape. OTel **metrics** can export OTLP or a Prometheus exporter.

A pragmatic split:

- **RED and runtime** via `prometheus/client_golang` or OTel metric SDK—pick one per process.
- **Traces** via OTel (this is where OTel is undisputed).

Duplicating the same counter in Prometheus and OTel metrics is how dashboards disagree. One source of truth per signal.

---

## Context, Baggage, and PII

**Baggage** is key-value propagated to children. It is not a replacement for `context.Value`. Do not put PII in baggage; it may be serialized on every hop and stored in backends.

Trace attributes: follow **semantic conventions** (`http.route`, `server.address`). Custom attributes for business IDs are fine if cardinality is bounded or they live only on **spans**, not metric labels.

---

## Collector Tips

- Run the collector as a **DaemonSet** (node) or **gateway** deployment (cluster). Apps talk to the local collector.
- Enable **memory limiter** and **retry**. An unbounded collector OOMs and you lose everything.
- Tail sampling in the collector: keep `status == ERROR` and latency > threshold.

---

## Common Go Footguns

1. **Global TracerProvider never set** — you get no-op tracers; “OTel is integrated” but the backend is empty.
2. **Not wrapping HTTP client** — missing client spans; traces look like the service did nothing.
3. **Using `context.Background()` in workers** started from a request — broken parent.
4. **Sampling 100% in prod** — bill shock; sampling 0% — unusable.
5. **Ignoring `Shutdown`** — last spans dropped.

---

## Takeaways

In Go, OpenTelemetry is **provider in `main`, wrap HTTP/gRPC, pass `ctx`, flush on shutdown**. Prefer **OTLP to a collector**. Add **manual spans** at business and dependency boundaries. Align sampling with the **gateway and mesh** so a sampled request stays sampled.

Instrumentation is production code. Review it like you review mutexes.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
