---
layout: post
title: "Service Mesh Fundamentals: What the Sidecar Actually Does"
date: 2025-04-21 10:15:00 +0530
categories: service-mesh kubernetes cloud-native
tags: [service-mesh, istio, linkerd, envoy, sidecar, kubernetes]
---

# Service Mesh Fundamentals: What the Sidecar Actually Does

A **service mesh** is infrastructure for **service-to-service** traffic: identity, encryption, retries, timeouts, traffic splitting, and telemetry—without rewriting every application. In Kubernetes, the classic shape is a **sidecar proxy** next to each workload. The app talks to `localhost`; the proxy talks to the rest of the cluster.

That sounds like an API gateway. It is not. Gateways sit at the **edge** (ingress/egress). A mesh sits **between** services. You often want both.

This post covers the data plane vs control plane, what the sidecar intercepts, and what a mesh is bad at.

---

## Why Meshes Appeared

Microservices turned every language team into a networking team:

- Retries and timeouts implemented five different ways
- mTLS “we’ll do it next quarter”
- Golden signals missing for service B calling service C
- Canaries requiring app-level feature flags only

A mesh **standardizes L7 (and L4) policy** at a proxy that is upgraded independently of the app. The app keeps using HTTP/gRPC as usual.

The cost is **operational**: another moving part, extra hops, and a control plane you must secure.

---

## Control Plane vs Data Plane

```text
┌─────────────────────────────────────────┐
│              Control plane              │
│  (Istiod, Linkerd destination, etc.)    │
│  - Service discovery                    │
│  - Certificate authority (mTLS)         │
│  - Push config: routes, retries, RBAC   │
└──────────────────┬──────────────────────┘
                   │ xDS / equivalent
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
  sidecar       sidecar       sidecar
  (Envoy)       (Envoy)       (Linkerd
                              proxy)
     ▲             ▲             ▲
     │ iptables /  │             │
     │ eBPF redirect             │
   app pod       app pod       app pod
```

| Plane | Job |
|-------|-----|
| **Data plane** | Proxies that carry bytes. Envoy (Istio, Consul, AWS App Mesh historically) or purpose-built proxies (Linkerd). |
| **Control plane** | Watches Kubernetes Services/Endpoints, issues certs, computes routes, pushes config to proxies. |

If the control plane is down, **existing** proxies often keep last-known config. New pods may not get identity. Treat control plane HA as seriously as etcd.

---

## Traffic Interception

Typical Istio-style path:

1. Init container (or CNI plugin) installs **iptables** / **nftables** rules.
2. Outbound TCP from the app is redirected to the sidecar’s outbound listener.
3. Inbound TCP to the pod is redirected to the sidecar’s inbound listener, then to the app on `localhost`.

The app believes it connected to `other-svc.ns.svc.cluster.local:80`. The sidecar **does** the real connection, HTTP parse (if protocol is known), mTLS, and metrics.

**Headless services, raw TCP, and some CNI combos** break or bypass interception. Always test: Kafka, databases, and non-HTTP protocols need explicit **protocol selection** or exclusion.

**Ambient mesh** (Istio) and **Linkerd’s** model reduce or eliminate per-pod sidecars for some traffic; the *job* of the mesh stays the same, the *hop* changes. More on that in a later post.

---

## What You Gain

**Identity.** Workload identity (SPIFFE IDs like `spiffe://cluster.local/ns/foo/sa/bar`) instead of trusting pod IPs. Policies say “frontend may call payments,” not “10.0.0.0/8 may call payments.”

**mTLS.** Proxies terminate TLS; apps can stay HTTP on localhost. Rotation is the control plane’s problem.

**Resilience policy.** Timeouts, retries with retry budgets, circuit breaking, outlier detection—**uniform** and **central**. Retries without budgets amplify outages; a good mesh makes the budget visible.

**Observability.** Proxies emit golden metrics (request rate, error rate, duration) and can propagate **trace context** (B3, W3C `traceparent`). You still instrument the app for *internal* spans.

**Traffic shaping.** Weighted routes, header-based routing, fault injection for game days.

---

## What a Mesh Is Not

- **Not an ingress.** North-south still needs a gateway (mesh ingress is a specialized proxy at the edge).
- **Not a replacement for app timeouts.** If the app blocks forever on CPU, the proxy cannot save you.
- **Not free.** Each hop adds latency (often sub-millisecond to a few ms) and CPU. Sidecar memory × replica count is a real bill.
- **Not a service catalog.** It consumes Kubernetes (or Consul) discovery; it does not replace product domain modeling.

---

## Istio vs Linkerd (Operator View)

| | **Istio** | **Linkerd** |
|--|-----------|-------------|
| Proxy | Envoy | Rust micro-proxy |
| Feature surface | Very large (Gateway API, WASM, multi-cluster, ambient) | Intentionally smaller |
| Complexity | Higher | Lower for the common path |
| mTLS | On by default in modern setups (PERMISSIVE vs STRICT) | On by default |

Pick based on **team size and required features**, not Twitter. Many teams want Linkerd’s simplicity; many platforms need Istio’s knobs and vendor ecosystem.

---

## Adoption Path That Works

1. **Mesh one namespace** in PERMISSIVE mTLS; compare metrics with and without.
2. Turn **STRICT** mTLS when you can prove nothing talks plaintext around the proxy.
3. Add **timeouts and retry budgets** for idempotent reads first—not for POSTs.
4. Export **RED metrics** from the proxy into the same Grafana you already use.
5. Only then: canaries, multi-cluster, WASM filters.

Skipping to “mesh everything + 15 VirtualServices” is how meshes get a bad reputation.

---

## Takeaways

A service mesh is a **programmable L4/L7 data plane** with a **control plane that understands service identity**. The sidecar (or ambient waypoint) is where policy and telemetry actually run. Use it for **uniform security and visibility** between services; keep **edge routing** on an API gateway; keep **business retries** honest and idempotent.

If you cannot explain what iptables (or eBPF) is doing to a packet in a pod, you are not ready to debug the mesh in production—yet. That packet path is the whole product.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
