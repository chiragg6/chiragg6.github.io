---
layout: post
title: "mTLS and Zero Trust Inside the Cluster"
date: 2025-07-19 13:05:00 +0530
categories: service-mesh security kubernetes
tags: [mtls, spiffe, zero-trust, service-mesh, istio, security]
---

# mTLS and Zero Trust Inside the Cluster

Kubernetes NetworkPolicy can say which pods may connect. It cannot say **which workload identity** presented a certificate, and it does not encrypt the payload. Cluster networks are often flat; a compromised pod that can reach `payments:8080` is enough if the server trusts the network.

**Mutual TLS (mTLS)** in a service mesh binds **SPIFFE identity** to every connection: both sides present certificates, the proxy verifies SAN/URI, and bytes on the wire are encrypted. That is the practical core of **zero trust** for east-west traffic: never trust the pod IP; trust the **service account / SPIFFE ID**.

This post covers identity, PERMISSIVE vs STRICT, authorization, and the failure modes that look like “the mesh broke DNS.”

---

## SPIFFE, Not “Just TLS”

A typical identity:

```text
spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>
```

The control plane is a **certificate authority** (or talks to one). Short-lived certs are pushed to proxies. Apps on localhost can remain HTTP; the **proxy** is the TLS principal.

This is different from ingress TLS, where only the **server** has a cert and clients are browsers or API consumers. mTLS means **both** peers authenticate.

---

## PERMISSIVE vs STRICT

| Mode | Behavior |
|------|----------|
| **PERMISSIVE** | Accept plaintext and mTLS. Migration mode. A leftover plaintext client still works. |
| **STRICT** | Reject plaintext. Real zero-trust posture for that port. |

PERMISSIVE forever is **theater**. Attackers use the plaintext path. The rollout is: mesh inject → prove traffic is mTLS in metrics/telemetry → STRICT → watch for `401`/`503` from non-meshed clients (Jobs, out-of-cluster, mis-injected pods).

**Port exclusions** exist for a reason: a database protocol that is not proxied must not be “STRICT HTTP” on 5432. Exclude or use TCP mode deliberately.

---

## Authorization: Who May Call Whom

mTLS answers **“who are you?”** Authorization policy answers **“are you allowed?”**

Example intent:

- `frontend` may `GET` `orders`
- `checkout` may `POST` `payments`
- nobody else may hit `payments`

In Istio this is `AuthorizationPolicy` (and older `PeerAuthentication` for mTLS mode). In Linkerd, `Server` + `ServerAuthorization` / `MeshTLS` policies. The details differ; the model is **identity-aware allow lists**.

Default-deny **in-mesh** plus NetworkPolicy **at L3** is defense in depth. Mesh policy without NetworkPolicy still helps against confused apps; NetworkPolicy without mTLS still allows spoofing if an attacker gets a pod in the right CIDR (depending on CNI).

---

## Trust Domain and Multi-Cluster

The **trust domain** in the SPIFFE ID must match how you federate CAs. Multi-cluster meshes either:

- Share a root CA, or
- Bundle multiple roots and map identities across clusters

Misaligned trust domains show up as **handshake failures**, not as HTTP 404. Debug with proxy logs (`TLS error`, `certificate verify failed`), not with application logs.

---

## Rotation and Outage Modes

Certs rotate **before** expiry. If the control plane cannot issue:

- New pods fail to get identity
- Existing connections may live until cert expiry (implementation-dependent)

Watch:

- Certificate expiry metrics
- Control plane availability
- `istiod` / identity service logs
- Sudden rise in **connection failures** (not just HTTP 5xx)

Clock skew on nodes breaks JWT-like validity windows and cert `NotBefore`. NTP is a security dependency.

---

## What mTLS Does Not Do

- **Does not authorize at the method level inside the app** unless you also enforce HTTP policy. A stolen identity with broad allow rules is still powerful.
- **Does not replace** secrets management for *data* (S3 keys, DB passwords). Those stay in Vault/KMS.
- **Does not encrypt at rest.**
- **Does not stop** a compromised app from using its **own** identity to call whatever that identity is allowed to call. Zero trust is **least privilege**, not magic.

---

## App-Level TLS vs Mesh mTLS

You can do TLS in Go with `tls.Config` and SPIFFE libraries (`go-spiffe`). That is valid for services **outside** a mesh or for extra encryption.

Doing **both** mesh mTLS and app TLS on the same hop is usually waste (double crypto) unless compliance demands it. Pick a layer. Mesh wins when you have **many languages** and want **one** identity system.

---

## Debugging Checklist

1. Is the port **intercepted**? (`iptables`/`istioctl`/`linkerd tap`)
2. Is PeerAuthentication **STRICT** while a client is not meshed?
3. Do SAN/SPIFFE IDs match the **AuthorizationPolicy** principals?
4. Protocol: HTTP vs TCP vs gRPC? Wrong protocol detection + mTLS is a common footgun.
5. Headless / `ClusterIP: None` / Direct pod IP: discovery and mTLS identity may not match your mental model of “I called a Service.”

---

## Takeaways

Zero trust east-west is **short-lived workload identity + mTLS + explicit authorization**, not a flat CNI. STRICT mode is the goal; PERMISSIVE is a migration tool. The mesh makes this operationally tractable across languages.

If policy still allows every service account to call every service, you have encrypted a **flat network**. Encryption without least privilege is only half the job.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
