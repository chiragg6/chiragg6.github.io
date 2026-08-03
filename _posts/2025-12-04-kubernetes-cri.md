---
layout: post
title: "Deep Dive into Kubernetes CRI"
date: 2025-12-04 12:00:00 +0530
categories: kubernetes containers devops
tags: [kubernetes, cri, containerd, cri-o, kubelet, containers]
---

# Deep Dive into Kubernetes CRI

When a Pod lands on a node, something has to pull images, create containers, wire up networking, and tear everything down when the Pod disappears. For years that "something" was tightly coupled to Docker. Kubernetes grew faster than any single runtime could stay the default forever.

The **Container Runtime Interface (CRI)** is the abstraction that fixed that. It defines a stable gRPC contract between the **kubelet** and whatever runtime actually runs your containers. Today, **containerd** and **CRI-O** are the common choices; Docker sits behind them as an optional layer, not as the thing the kubelet talks to directly.

In this post we walk through why CRI exists, how the kubelet uses it, what the API looks like, and what happens on a node from Pod admission to container exit.

---

## Why Kubernetes Needed CRI

Before CRI (Kubernetes 1.5 era), the kubelet embedded Docker-specific logic. Every new runtime feature—rootless containers, different snapshotters, alternative image stores—required changes deep inside Kubernetes core.

That created three problems:

1. **Tight coupling** — Kubernetes releases were blocked on Docker integration details.
2. **Slow innovation** — Alternative runtimes (rkt, hypervisor-based runtimes) could not plug in cleanly.
3. **Maintenance burden** — The in-tree Docker code path grew large and fragile.

CRI moved runtime integration **out of the Kubernetes tree** and into small shim processes that implement a protobuf/gRPC API. The kubelet only speaks CRI. Runtimes compete on performance, security, and operability without forking Kubernetes.

The removal of **dockershim** in Kubernetes 1.24 made this official: the kubelet no longer talks to Docker Engine directly. If you still use `docker` CLI on a node, it is almost certainly backed by **containerd** underneath.

---

## Architecture at a Glance

On a typical Linux node today, the stack looks like this:

```text
┌─────────────────────────────────────────────────────────┐
│                     Control Plane                       │
│              (API Server, Scheduler, etc.)              │
└──────────────────────────┬──────────────────────────────┘
                           │ Pod spec
                           ▼
┌─────────────────────────────────────────────────────────┐
│                        kubelet                          │
│   - Watches Pod specs assigned to this node             │
│   - Calls CRI to create sandboxes and containers        │
│   - Reports status back to API server                   │
└──────────────────────────┬──────────────────────────────┘
                           │ gRPC (CRI)
                           ▼
┌─────────────────────────────────────────────────────────┐
│              CRI implementation (shim)                  │
│         containerd (built-in CRI plugin) or CRI-O       │
└──────────────────────────┬──────────────────────────────┘
                           │ OCI runtime spec
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   low-level runtime                     │
│              runc, crun, kata-runtime, etc.             │
└─────────────────────────────────────────────────────────┘
```

A few roles worth naming explicitly:

| Component | Role |
|-----------|------|
| **kubelet** | Node agent; orchestrates Pod lifecycle via CRI |
| **CRI shim** | gRPC server implementing `RuntimeService` and `ImageService` |
| **OCI runtime** | Creates the actual OS process namespaces, cgroups, rootfs |
| **CNI plugin** | Configures Pod network (not part of CRI, but called around sandbox creation) |

CRI handles **containers and images**. Networking is delegated to **CNI**. Storage attach/mount is coordinated by the kubelet with **CSI** drivers. CRI is one leg of the stool, not the whole node stack.

---

## The CRI API

CRI is defined in protobuf under the Kubernetes project (`k8s.io/cri-api`). The kubelet acts as a **client**; the runtime is the **server**. Two services matter most:

### RuntimeService

Lifecycle of **sandboxes** (Pod infra containers) and **containers** inside them.

Common RPCs:

- `RunPodSandbox` — create the Pod network namespace and sandbox container
- `StopPodSandbox` / `RemovePodSandbox` — tear down the sandbox
- `CreateContainer` / `StartContainer` — create and start workload containers
- `StopContainer` / `RemoveContainer` — stop and delete containers
- `ListContainers`, `ListPodSandbox` — introspection for status sync
- `Exec`, `Attach`, `PortForward` — streaming operations for `kubectl`

### ImageService

Image pull and inspection.

Common RPCs:

- `PullImage` — pull (or ensure presence of) an image
- `ListImages` — images available locally
- `ImageStatus` — metadata for a specific image
- `RemoveImage` — garbage-collect unused images

The API is **versioned** (`runtime.v1` is current). Runtimes must implement the version the kubelet expects for a given Kubernetes release.

---

## Pod Lifecycle Through CRI

When the scheduler binds a Pod to your node, the kubelet reconciles desired state. At a high level:

1. **Pull images** — `ImageService.PullImage` for each container image (with backoff on failure).
2. **Create sandbox** — `RunPodSandbox` sets up the Pod's network namespace. CNI runs as part of this path (kubelet invokes CNI; the sandbox holds the netns).
3. **Create containers** — For each container in the Pod, `CreateContainer` with the OCI spec (command, env, mounts, security context).
4. **Start containers** — `StartContainer` for each; the low-level runtime forks the process.
5. **Probe and report** — kubelet watches container exit codes and probe results; syncs status to the API server.
6. **Teardown** — On Pod deletion, stop containers, stop/remove sandbox, optionally remove images per GC policy.

Sandbox vs container is an important distinction. In Docker terms, the sandbox is like the "pause" container (`registry.k8s.io/pause` or similar). Workload containers join that sandbox's namespaces.

```text
Pod (logical)
├── Sandbox (infra / pause)     ← RunPodSandbox
├── Container: app              ← CreateContainer + StartContainer
└── Container: sidecar          ← CreateContainer + StartContainer
```

---

## containerd vs CRI-O

Both are production-grade CRI implementations. The kubelet does not care which one you use as long as the CRI contract is satisfied.

### containerd

- Graduated CNCF project; Docker Engine itself uses containerd internally.
- CRI is a **plugin** inside containerd (`cri` plugin).
- Widely used on managed clouds and kubeadm clusters.
- Strong ecosystem (nerdctl, buildkit integration).

### CRI-O

- Purpose-built for Kubernetes by the Open Container Initiative community.
- Smaller surface area: CRI + OCI, without broader "general container platform" scope.
- Default on OpenShift; common on RHEL/Fedora-based distros.

| Feature | containerd | CRI-O |
|---------|------------|-------|
| Origin | General container runtime | Kubernetes-focused |
| Image store | containerd snapshotter model | containers/storage (shared with Podman) |
| Typical CLI | `ctr`, `nerdctl` | `crictl`, `podman` (compatible store) |
| CRI server | Built into containerd | `crio` daemon |

For most greenfield clusters, **either works**. Operational familiarity and distro support usually drive the choice.

---

## Debugging with crictl

`crictl` is the standard CLI for talking CRI directly—useful when `kubectl` shows a Pod stuck in `ContainerCreating` and you need to see what the runtime thinks.

```bash
# List sandboxes (Pod infra)
crictl pods

# List containers
crictl ps -a

# Inspect a container
crictl inspect <container-id>

# Pull an image through the runtime
crictl pull nginx:latest

# Stream logs
crictl logs <container-id>
```

Point `crictl` at your socket in `/etc/crictl.yaml`:

```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
```

For CRI-O, the socket is typically `unix:///var/run/crio/crio.sock`.

---

## CRI and the OCI Spec

CRI sits **above** the [Open Container Initiative (OCI)](https://opencontainers.org/) specs:

- **runtime-spec** — how to configure `config.json` for `runc` (cgroups, namespaces, capabilities, mounts).
- **image-spec** — how images are structured (layers, manifest).

The CRI server translates Kubernetes Pod/Container fields into an OCI bundle, then invokes `runc` (or equivalent). This layering is why you can swap `runc` for `crun` or a VM-isolated runtime without changing the kubelet.

Security contexts from the Pod spec—`runAsUser`, `readOnlyRootFilesystem`, dropped capabilities—flow into the OCI `config.json` the runtime executes.

---

## Streaming: Exec, Attach, Port Forward

Interactive `kubectl exec` and `kubectl logs` do not go through the API server's generic REST alone. The kubelet opens a **streaming** connection to the CRI server, which attaches to the container's stdio or TTY.

Rough flow for `kubectl exec`:

```text
kubectl → API server → kubelet → CRI Exec RPC → runtime → container process
```

The CRI streaming server often listens on a separate port or socket from the main gRPC service. Misconfigured firewall rules on that path are a common reason exec works on some nodes but not others.

---

## Image Pull Policy and CRI

The kubelet decides **when** to pull based on `imagePullPolicy`:

| Policy | Behavior |
|--------|----------|
| `IfNotPresent` | `ImageStatus`; pull only if missing locally |
| `Always` | Pull on every Pod start |
| `Never` | Fail if image not already present |

Credentials come from **imagePullSecrets**, resolved by the kubelet and passed into `PullImage` as auth config. CRI does not implement registry auth policy—that remains kubelet and API machinery.

---

## Garbage Collection

The kubelet runs GC loops that call CRI to reclaim resources:

- **Dead containers** — exited containers past a threshold
- **Unused images** — images not referenced by any container, subject to high/low watermark settings
- **Sandboxes** — orphaned sandboxes after Pod removal

Tuning `--eviction-hard`, image GC thresholds, and runtime-specific settings together matters on disk-constrained nodes.

---

## Common Failure Modes

**`ContainerCreating` forever**

- Image pull errors (auth, registry down, wrong tag)
- CNI plugin failure during `RunPodSandbox`
- SELinux/AppArmor denials when starting the sandbox

**`RunContainerError` / `CreateContainerError`**

- Invalid command or missing binary in image
- Mount path conflicts or missing volumes
- Seccomp or capabilities blocked by policy

**Runtime socket missing**

- containerd or CRI-O not running
- kubelet configured with wrong `--container-runtime-endpoint`

Check kubelet logs (`journalctl -u kubelet`) and `crictl` output before chasing control-plane issues.

---

## What CRI Does Not Cover

Keeping the boundaries clear saves debugging time:

| Concern | Mechanism |
|---------|-----------|
| Container / image lifecycle on node | **CRI** |
| Pod networking | **CNI** |
| Persistent volumes | **CSI** |
| Scheduling, desired state | **Kubernetes API** |
| Ingress, Services | **kube-proxy**, CNI, cloud LB |

CRI is deliberately narrow: it is the kubelet's runtime dial tone.

---

## Takeaways

- **CRI** is the gRPC interface between kubelet and container runtimes, split into `RuntimeService` and `ImageService`.
- **Sandboxes** model Pods; workload containers run inside the sandbox's namespaces.
- **containerd** and **CRI-O** are the mainstream implementations; Docker is no longer in the kubelet path.
- Operations and debugging flow through **`crictl`** and kubelet logs, not `docker ps` on modern nodes.
- CRI translates Kubernetes intent into **OCI** specs executed by **runc** (or alternatives).

Understanding CRI turns "my Pod is stuck" from a black box into a short checklist: image pull, sandbox, CNI, container start, probes—each with a clear RPC and log trail on the node.
