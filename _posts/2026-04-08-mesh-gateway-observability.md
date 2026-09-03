---
layout: post
title: "Mesh, Gateway, and Observability as One Stack"
date: 2026-04-08 14:00:00 +0530
categories: service-mesh api-gateway observability
tags: [service-mesh, api-gateway, observability, istio, envoy, opentelemetry]
---

# Mesh, Gateway, and Observability as One Stack

Teams buy an API gateway, then a service mesh, then an APM vendor, and still cannot answer: **why did checkout fail?** The tools overlap on Envoy, on `traceparent`, and on RED metrics—then **disagree** on status codes, route names, and identity.

This post is an architecture for treating **edge, east-west, and telemetry** as one design, with Go services as the workload in the middle.

---

## The Packet and the Signal

```text
Client
  │  TLS, JWT, rate limit, inject traceparent
  ▼
API Gateway (Envoy / Kong / Gateway API)
  │  access log + RED + SERVER span
  ▼
(optional) mesh ingress
  │
  ▼
orders (Go) + sidecar/ambient
  │  mTLS, retry budget, proxy RED
  │  app spans + /metrics
  ▼
payments (Go) + mesh
  │
  ▼
Postgres / HTTP vendor
```

Three planes:

1. **Data path** — bytes: gateway then mesh then app.
2. **Control path** — Gateway API + mesh policies + certs.
3. **Telemetry path** — same **trace ID**, consistent **resource attributes**, one **schema** for HTTP metrics.

If those planes are owned by three teams with three naming schemes, you will page from Prometheus and debug in a different universe.

---

## One Propagation Standard

Pick **W3C Trace Context** (`traceparent`, `tracestate`) everywhere:

- Gateway injects/respects it
- Mesh proxies propagate it (Istio/Linkerd settings)
- Go `otel.SetTextMapPropagator(TraceContext{})`
- Kafka headers mapped to the same

Baggage is optional and dangerous (PII). Skip it until you have a use case.

**Sampling:** decide at the **gateway** (or collector tail sampling). Mesh and apps use **parent-based** sampling so the tree is complete. If the mesh samples independently at 1% and the app at 1%, you get 0.01% full traces.

---

## One Identity Story

- **North-south:** JWT / API keys at the gateway. Do not make every sidecar a JWT validator for public clients.
- **East-west:** SPIFFE + mTLS in the mesh. Services trust **mesh identity**, not `X-User-Id` from random pods.
- **Gateway → first service:** either the gateway is a mesh identity (`ingressgateway` SA) allowed to call `orders`, or you terminate mesh at ingress and use NetworkPolicy. Document the trust boundary.

AuthorizationPolicy that allows `*` because “the gateway already auth’d” without restricting **which** identity the gateway uses is a hole.

---

## Retries: Pick a Layer

| Layer | Retry? |
|-------|--------|
| Gateway | GET/idempotent, tight budget, toward **first** hop |
| Mesh | East-west, retry budget, outlier ejection |
| App | Business-aware (only if you know idempotency keys) |

**Never** default-on retries on all three. Draw the diagram in the platform README. Checkout POST: **no automatic retry** at gateway or mesh unless the API is idempotent.

Timeouts: **decrease** as you go in, or set **one** budget at the gateway slightly above the SLO and make interior timeouts **stricter** so the gateway is not the first to fire. Interior timeout > gateway timeout wastes work.

---

## Metrics That Line Up

Agree on labels:

- `service` / `job` = the **app**, not `istio-proxy`
- `http_route` = OpenAPI template, not raw URL
- `code` or `status` = same buckets (2xx, 4xx, 5xx)

Proxy metrics (`istio_requests_total`, Envoy clusters) are **hops**. App metrics are **business**. Dashboards should say **which**. An SLO on checkout belongs on **gateway** (or RUM), with drill-down to mesh and app.

Exemplars: Prometheus histograms → Tempo trace IDs. That is how you go from SLO burn to a trace in one click.

---

## Logs

Gateway access logs: request ID = trace ID when possible.

App logs: JSON + `trace_id`.

Mesh access logs: debug **intermittent 503 UC** (upstream connection). Do not index them at the same verbosity as app errors forever.

---

## A Reference Platform Slice

For a new cluster I would pick **one** of each, not a buffet:

- **Gateway API** + Envoy Gateway (or Istio Gateway) for ingress
- **One mesh** (Istio ambient or Linkerd)—not both
- **OTel collector** DaemonSet + tail sampling
- **Prometheus** + **Tempo** + **Loki** (or a single vendor that speaks OTLP)
- Go services: `otelhttp`, parent-based sampler, `GOMEMLIMIT`, bounded `errgroup`

The products can change. The **contracts** (W3C, SPIFFE, route names, retry layer) should not.

---

## Failure Drills

Game days that teach the stack:

1. Kill a payments replica: do **outlier detection** and **gateway** error budgets behave?
2. Expire a mesh cert path: do you see handshake errors, not app panics?
3. Drop sampling to 0% in the collector: do SLOs (metrics) still work? (They must.)
4. Slow Postgres: does the trace show pool wait vs query?

If the drill only works when someone who wrote the YAML is in the room, the stack is not operable.

---

## Takeaways

Gateway, mesh, and observability share **Envoy-shaped data planes** and **trace context**. Treat them as one design: **propagation, identity, retries, and metric names**. Put **user SLOs** at the edge, **security** east-west in the mesh, **business spans** in Go.

Buying more tools without those contracts just adds hops to the same 2.4 second checkout.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
