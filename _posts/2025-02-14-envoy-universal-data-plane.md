---
layout: post
title: "Envoy as a Universal Data Plane"
date: 2025-02-14 11:10:00 +0530
categories: api-gateway service-mesh envoy
tags: [envoy, xds, api-gateway, service-mesh, kubernetes]
---

# Envoy as a Universal Data Plane

**Envoy** shows up in Istio sidecars, in API gateways (Ambassador, Kong’s Envoy mode, Envoy Gateway, AWS App Mesh historically), in CI/CD edge stacks, and as a standalone process next to a legacy JVM. The reason is the same: a production-grade **L4/L7 proxy** with a remote configuration API (**xDS**), excellent observability, and HTTP/gRPC/TCP first-class.

If you understand Envoy, you understand half of modern “cloud networking” products. They are **control planes** that generate Envoy config.

This post is a practitioner’s map: listeners, clusters, xDS, filters, and how gateway vs sidecar configs differ.

---

## Core Objects

| Object | Meaning |
|--------|---------|
| **Listener** | Bind address + filter chain (TLS inspector, HTTP connection manager, TCP proxy) |
| **Route / Virtual host** | HTTP: domain + path → cluster, timeouts, retries |
| **Cluster** | Upstream: EDS endpoints, load balancing, health, circuit breakers |
| **Endpoint** | IP:port of a replica |
| **Secret** | TLS certs (SDS) |

Traffic: **downstream** (client) → Listener → filters → **cluster** → **upstream** (your service).

```text
Client → Listener :443 (TLS)
           → HTTP connection manager
           → Route match
           → Cluster "orders"
           → Endpoints 10.0.1.4:8080, 10.0.1.5:8080
```

---

## xDS: Why Config Is an API

Static YAML is fine for a single edge proxy. A mesh with 5,000 pods cannot SSH-reload Envoy.

**xDS** (discovery services) is gRPC (or REST) streaming:

- **LDS** — listeners
- **RDS** — routes
- **CDS** — clusters
- **EDS** — endpoints
- **SDS** — secrets

The control plane (Istiod, a gateway operator, Consul) **pushes** incremental updates. Envoy **hot-restarts** or applies in place without dropping all connections (with care).

If xDS is stuck, proxies run **stale** config. That is better than failing closed in many designs—and worse if you needed an emergency deny rule to propagate.

---

## Filter Chains

Envoy is a **filter pipeline**. HTTP filters: decoder/encoder for auth, JWT, RBAC, Lua/WASM, rate limit (often external RLS gRPC), compressor.

Order matters. JWT auth before RBAC before routing is a typical edge chain. A sidecar inbound chain might be: mTLS → HTTP → RBAC → router.

**WASM** extends without forking Envoy. Use it for custom protocol glue, not for your order-checkout saga.

---

## Gateway vs Sidecar Config Shape

**Edge (gateway):**

- Few listeners (`:443`, maybe `:80`)
- Many virtual hosts (customer domains)
- Clusters pointing at Kubernetes Services or mesh internals
- Heavy: WAF, JWT, global rate limits

**Sidecar outbound:**

- Listener often `127.0.0.1:15001` (iptables redirect)
- Routes built from **all** services the pod might call (or from original dst)
- Clusters with **STRICT_DNS** or EDS from control plane
- mTLS origination to other identities

**Sidecar inbound:**

- Listener on redirected inbound port
- Terminate mTLS
- Route to `127.0.0.1:appPort`

Same binary, different **topology**. Debugging `cluster not found` at the edge is a route bug; in a sidecar it is often **discovery** or **mTLS**.

---

## Resilience Features You Should Actually Use

**Timeouts.** `timeout` on routes; `connect_timeout` on clusters. No timeout = goroutine/thread pile-up upstream of Envoy too.

**Retries.** `retry_on: 5xx,reset,connect-failure` with **retry budget** (percentage of req/s). Unbounded retries are load amplifiers.

**Outlier detection.** Eject an endpoint that errors a lot; it is not a full circuit breaker by itself.

**Circuit breakers.** Max connections, max pending requests, max retries **per cluster**. This is how you protect a shrinking replica set.

**Hedging.** Rarely; can double load.

---

## Observability Built In

Envoy emits:

- Downstream/upstream **counters and histograms**
- **Access logs** (JSON to stdout is the Kubernetes-native way)
- **Tracing** (OpenTelemetry / Zipkin / Datadog exporters)

The `x-envoy-upstream-service-time` and similar headers help debug **where** time went (use with caution; do not leak internals to untrusted clients).

Admin interface (`127.0.0.1:9901`) is gold: `clusters`, `config_dump`, `stats`. **Never** expose it on the public internet.

---

## Health and Load Balancing

- **Active health checks** vs Kubernetes readiness: often **EDS + K8s probes** is enough; double health checking can flap.
- **LEAST_REQUEST** vs **ROUND_ROBIN**: least request is often better for uneven RPC costs.
- **Zone-aware** routing reduces cross-AZ cost and latency when configured.

---

## Takeaways

Envoy is the **data plane Linux of cloud-native L7**: listeners, clusters, xDS, filters. Gateways and meshes are **opinions + control planes** on top. Learn `config_dump` and stats; you will debug every vendor eventually.

Configure **timeouts, retry budgets, and circuit breakers** on purpose. Default-open Envoy is a very fast way to DDoS yourself.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
