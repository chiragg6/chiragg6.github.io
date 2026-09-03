---
layout: post
title: "gRPC, HTTP/2, and Go: Concurrency at the RPC Layer"
date: 2024-10-19 18:10:00 +0530
categories: golang concurrency api-gateway
tags: [golang, grpc, http2, concurrency, protobuf]
---

# gRPC, HTTP/2, and Go: Concurrency at the RPC Layer

Go’s `net/http` and `google.golang.org/grpc` both sit on **HTTP/2** (gRPC requires it). That is not a trivia fact: HTTP/2 **multiplexes** many RPCs on one TCP connection, which changes **connection pooling**, **head-of-line blocking** (at TCP, not H2 streams), **timeouts**, and how a mesh or API gateway should treat the traffic.

This post connects gRPC’s concurrency model in Go to gateways, meshes, and load balancing—where people accidentally build a **single fat connection** to one pod and call it “scaled.”

---

## Why gRPC Feels Different

- **Protobuf** contracts, not ad-hoc JSON
- **Deadlines** as first-class (`context` + `grpc.WithTimeout` / incoming deadline)
- **Streaming**: client, server, bidi—long-lived goroutines
- **Status codes** (`codes.Unavailable` vs HTTP 503 mapping at a gateway)

A unary RPC in Go is typically: one handler goroutine, interceptors, then your code. A **bidi stream** is a long-lived handler; leaking it leaks a goroutine **and** an HTTP/2 stream.

---

## HTTP/2 Multiplexing and Load Balancing

Many gRPC clients (including Go) **reuse connections**. If your Kubernetes Service is `ClusterIP` and kube-proxy/IPVS hashes **connections**, all multiplexed RPCs on that connection hit **one backend pod**.

That is the classic **gRPC load-balancing problem**.

Mitigations:

- **gRPC-aware L7** (Envoy, mesh sidecar) that load-balances **per RPC**
- Client-side **lookaside** load balancing / DNS with many addresses + `grpc.WithDefaultServiceConfig` round_robin
- **xDS** (same as Envoy) in gRPC clients
- Avoid assuming L4 LB = fair RPC distribution

At the **API gateway**, HTTP/2 to clients plus HTTP/1.1 to backends (or vice versa) is a normal translation. gRPC-Web is another translation. Know which hop is H2.

---

## Deadlines Beat Ad-Hoc Timeouts

```go
ctx, cancel := context.WithTimeout(ctx, 800*time.Millisecond)
defer cancel()
resp, err := client.Charge(ctx, req)
```

gRPC propagates deadlines to the server (`grpc-timeout` header). The server should **honor `ctx`**. If the server ignores it and the client gave up, you still do the work—classic wasted capacity.

Gateways should set **max** timeouts; services should set **tighter** ones for dependencies. Same story as the mesh/gateway stack post.

---

## Interceptors: Where Platform Meets App

Unary interceptors are the right place for:

- Auth (JWT already validated at gateway? still verify **internal** tokens if needed)
- Logging with `trace_id`
- Metrics (or use `otelgrpc`)
- Recovery from panics

Do not do heavy business logic in interceptors. Do not start unbounded goroutines there without waiting.

Keepalive: `keepalive.ServerParameters` / `ClientParameters`. Too aggressive keepalive + a mesh idle policy = flapping connections. Align with Envoy idle timeouts.

---

## Streaming and Backpressure

Server streaming:

```go
func (s *Srv) Tail(req *pb.TailReq, stream pb.Foo_TailServer) error {
    for {
        select {
        case <-stream.Context().Done():
            return stream.Context().Err()
        case ev := <-events:
            if err := stream.Send(ev); err != nil {
                return err
            }
        }
    }
}
```

If `Send` blocks because the client is slow, you have **backpressure**. If you buffer events unbounded in `events`, you OOM. Same as channel buffers in the rest of Go.

Gateways in front of streams need **idle timeouts** that match product (notifications vs a 5-second unary).

---

## Mesh and gRPC

Protocol must be **detected as HTTP/2 / gRPC** or retries and HTTP route matching will be wrong. Istio `port.protocol: GRPC` (or explicit AppProtocol) matters.

Retries: gRPC **retry policy** in service config vs mesh retries—**pick one**. Retry `UNAVAILABLE` is common; retry `INVALID_ARGUMENT` is a bug.

mTLS: mesh terminates TLS; gRPC can stay h2c on localhost. App-level TLS + mesh TLS is usually redundant.

---

## Observability

- Map gRPC **status** to RED error definition (`OK` vs not)
- Span names: `/package.Service/Method` (low cardinality)
- Tracing interceptors on both client and server
- Access logs at gateway for north-south gRPC (grpc-gateway JSON transcoding is a separate route name)

---

## Takeaways

gRPC in Go is **context + streams + multiplexed H2**. Scaling requires **L7 per-RPC load balancing**, not hoping ClusterIP is enough. Honor **deadlines**, bound **stream buffers**, and align **retries/timeouts** with the gateway and mesh.

If one pod is at 100% CPU and four are idle, you probably multiplexed all RPCs onto one connection. That is not a Go scheduler bug.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
