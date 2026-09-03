---
layout: post
title: "API Gateway Patterns for Microservices"
date: 2024-12-08 15:50:00 +0530
categories: api-gateway microservices cloud-native
tags: [api-gateway, kong, envoy, ingress, kubernetes, edge]
---

# API Gateway Patterns for Microservices

An **API gateway** is the **north-south** front door: TLS termination, authn/authz, rate limits, request routing, and often an external developer-facing API. Microservices behind it should not each reimplement JWT validation and CORS.

It is easy to turn the gateway into a **second monolith**—every business rule, every aggregation, every workflow. This post is about patterns that stay at the edge, and the ones that belong in services.

---

## Gateway vs Ingress vs Mesh vs BFF

| Thing | Typical role |
|-------|----------------|
| **Ingress / Gateway API** | Kubernetes object model; implementation may be NGINX, Envoy, Traefik, Istio gateway |
| **API gateway product** | Kong, Apigee, AWS API Gateway, Tyk—policies, developer portal, keys |
| **Service mesh** | East-west: identity, mTLS, retries between services |
| **BFF (Backend for Frontend)** | App-specific aggregation (mobile vs web). Often a **service**, not a generic gateway plugin |

A cluster can have: **Internet → Gateway API / NGINX → (optional mesh ingress) → services**. The gateway handles **clients you do not control**; the mesh handles **services you do**.

---

## Core Edge Responsibilities

**TLS and HTTP.** Certificates (cert-manager), HTTP/2, sometimes HTTP/3. Redirect HTTP→HTTPS.

**Routing.** Host + path + header → backend Service. Strangler pattern: `/v1` old, `/v2` new.

**Authentication.** JWT validation, mTLS for partners, OAuth2 token introspection, API keys. Prefer **validating tokens at the edge** and passing **identity headers** (or leaving the JWT) to services—with a **trust boundary**: only the gateway may set `X-User-Id`, or services verify JWT themselves. Mixing both without a rule causes spoofing.

**Rate limiting and quotas.** Per key, per IP, per JWT `sub`. Protects origin services from abusive clients. Does not replace **server-side** bulkheads for *internal* fan-out.

**WAF / schema validation.** Optional; OpenAPI validation at the edge catches garbage early.

**Observability.** Access logs, RED metrics, trace context injection (`traceparent`) if the client did not send one.

---

## Pattern: Pure Reverse Proxy

Gateway only routes and terminates TLS. Auth is in each service. Simple, duplicates auth logic. Fine for **small** systems or when services **must** not trust the network (zero trust end-to-end).

---

## Pattern: Edge Auth, Internal Trust

Gateway validates JWT, strips it or converts to internal identity, services on a private network trust `X-Authenticated-User`. Faster internals. **Requires** that nothing except the gateway can reach those services (NetworkPolicy, mesh STRICT mTLS with gateway identity).

This is the most common Kubernetes setup.

---

## Pattern: Aggregation / Graph at the Edge

Gateway calls five services and builds one response. Looks convenient. You have just written a **distributed monolith** in Lua/JS plugins:

- Hard to test
- Versioning hell
- Timeouts nested inside the gateway
- Team ownership fights

Prefer a **BFF service in Go/Java** with proper observability, or **GraphQL** as a dedicated service. Use gateway plugins for **cross-cutting** concerns only.

---

## Pattern: Backend for Frontend

```text
Mobile app  →  mobile-bff  →  orders, catalog, recs
Web app     →  web-bff     →  orders, catalog, cms
Partners    →  api-gateway →  public API subset
```

BFFs are **product code**. Gateways are **platform code**. Keep the split.

---

## Rate Limits, Timeouts, Retries

At the **edge**:

- Timeouts should be **slightly larger** than the slowest **client-facing** SLA, not infinite.
- Retries on **POST** are dangerous (non-idempotent). Retry **GET** and idempotent PUT with a budget.
- Rate limits should **fail closed** when Redis (or the counter store) is down, or you have a documented fail-open for availability.

The mesh may **also** retry east-west. **Retry at two layers** without coordination amplifies load. Pick: retries in mesh **or** gateway **or** app—document it.

---

## Kubernetes: Ingress vs Gateway API

**Ingress** is the old, lowest-common-denominator object. Annotations became a vendor DSL.

**Gateway API** (Gateway, HTTPRoute, GRPCRoute, ReferenceGrant) is the successor: role-oriented (infra owns Gateway, app owns HTTPRoute), typed routes, better TLS and splitting.

New platforms should standardize on **Gateway API** plus a data plane (Envoy Gateway, Istio, nginx-gateway-fabric, Traefik). Treat Ingress as legacy.

---

## Multi-Cluster and Hybrid

External clients should not need to know which cluster won the lottery. Patterns:

- **Global LB** (anycast / cloud HTTPS LB) → regional gateways
- **Gateway in each cluster** + DNS weighted records
- **API gateway SaaS** in front of clusters

Keep **sticky** only if you must (WebSockets, sessions). Prefer **stateless** APIs.

---

## Anti-Patterns

1. Business workflows in gateway plugins
2. Gateway as ESB (content-based routing to 40 backends with transformations)
3. No timeout (clients hang; goroutines pile up behind the gateway)
4. Logging **authorization headers** in access logs
5. One giant `HTTPRoute` owned by nobody

---

## Takeaways

The API gateway is the **policy and routing edge** for untrusted clients. Keep it **thin**: TLS, auth, limits, routing, telemetry. Put **product aggregation** in BFFs. Use the **mesh** for service identity inside the cluster. Prefer **Gateway API** on Kubernetes.

If a feature needs a product manager, it probably does not belong in a Kong plugin.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
