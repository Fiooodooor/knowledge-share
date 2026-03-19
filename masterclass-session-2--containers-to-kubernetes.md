---
marp: true
theme: default
paginate: true
backgroundColor: #1a1a2e
color: #e0e0e0
style: |
  section {
    font-family: 'Segoe UI', Arial, sans-serif;
  }
  h1 { color: #00d4ff; }
  h2 { color: #7b68ee; }
  h3 { color: #ff6b6b; }
  strong { color: #ffd93d; }
  code { background: #16213e; color: #00d4ff; padding: 2px 6px; border-radius: 4px; }
  table { font-size: 0.78em; }
  th { background: #16213e; color: #00d4ff; }
  td { background: #0f3460; }
  blockquote { border-left: 4px solid #7b68ee; background: #16213e; padding: 10px 20px; }
  a { color: #ffd93d; }
  pre { font-size: 0.72em; }
---

<!-- _class: lead -->

# 🐳 Session 2
## From Containers to Kubernetes
### Master Class — Container Orchestration & CRI

**Duration:** 90 minutes
**Level:** Intermediate → Advanced
**Prerequisite:** Session 1 (Bare Metal to Virtualization)

---

# 📋 Session 2 — Agenda

| # | Topic | Time |
|---|-------|------|
| 1 | Linux Kernel Primitives for Containers | 15 min |
| 2 | Containerization Deep Dive | 10 min |
| 3 | Dockerfiles & Image Building | 15 min |
| 4 | Container Runtimes: Docker, Podman, Buildx, CRI-O | 15 min |
| 5 | Kubernetes Architecture | 15 min |
| 6 | CRI — Container Runtime Interface | 10 min |
| 7 | Bare Metal to K8s Cluster — Full Journey | 5 min |
| 8 | Q&A / Discussion | 5 min |

---

<!-- _class: lead -->

# Part 1
## Linux Kernel Primitives
### The Foundation Containers Are Built On

---

# Containers Are NOT a Kernel Feature

> There is **no "container" system call** in Linux. A "container" is a **user-space concept** built from combining multiple independent kernel primitives.

### The building blocks:

| Primitive | Purpose | Year |
|-----------|---------|------|
| **chroot** | Filesystem isolation | 1979 |
| **Namespaces** | Resource visibility isolation | 2002–2016 |
| **cgroups** (v1/v2) | Resource limits & accounting | 2007/2016 |
| **seccomp-bpf** | System call filtering | 2012 |
| **Capabilities** | Fine-grained privilege control | 1999 (POSIX) |
| **AppArmor / SELinux** | Mandatory access control | 2003/2000 |
| **OverlayFS** | Layered filesystem (image layers) | 2014 |

> **A container = namespaces + cgroups + seccomp + capabilities + rootfs**

---

# Linux Namespaces — Visibility Isolation

> Namespaces make a process **think it's alone** on the system.

| Namespace | Flag | Isolates | Since |
|-----------|------|----------|-------|
| **Mount** | `CLONE_NEWNS` | Filesystem mount points | 2.4.19 (2002) |
| **UTS** | `CLONE_NEWUTS` | Hostname and domain name | 2.6.19 (2006) |
| **IPC** | `CLONE_NEWIPC` | System V IPC, POSIX MQs | 2.6.19 (2006) |
| **PID** | `CLONE_NEWPID` | Process IDs (init = PID 1) | 2.6.24 (2008) |
| **Network** | `CLONE_NEWNET` | Network stack, interfaces, routes | 2.6.24 (2008) |
| **User** | `CLONE_NEWUSER` | UID/GID mappings (rootless) | 3.8 (2013) |
| **Cgroup** | `CLONE_NEWCGROUP` | cgroup root visibility | 4.6 (2016) |
| **Time** | `CLONE_NEWTIME` | Boot and monotonic clocks | 5.6 (2020) |

```bash
# Create a process with new PID + NET + MNT namespaces:
unshare --pid --net --mount --fork /bin/bash
```

---

# Network Namespace — Deep Dive

> Each **network namespace** has its own complete, isolated network stack.

```
┌─────────────────── Host Network Namespace ───────────────────┐
│                                                               │
│  eth0 (physical NIC)    docker0 (bridge)      veth-host-1    │
│  10.0.0.5               172.17.0.1            ─────────┐     │
│                              │                          │     │
│                              │                     ┌────┤     │
│                              │                     │veth│     │
│                              │                     │pair│     │
│                              │                     └────┤     │
│                                                         │     │
│  ┌──────────── Container Network Namespace ──────────┐  │     │
│  │                                                   │  │     │
│  │  eth0 (veth-container-1)     lo (loopback)        │  │     │
│  │  172.17.0.2                  127.0.0.1            │  │     │
│  │                                                   │  │     │
│  │  Routing table:   default via 172.17.0.1          │  │     │
│  │  iptables rules:  (independent)                   │  │     │
│  │  /proc/net/...:   (container's own view)          │  │     │
│  │                                                   │  │     │
│  └───────────────────────────────────────────────────┘  │     │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

# PID Namespace — Process Isolation

```
Host PID Namespace:
  PID 1: systemd
  PID 100: dockerd
  PID 200: containerd
  PID 500: containerd-shim → (container process)
  PID 501: nginx (from host's view)

Container PID Namespace:
  PID 1: nginx  ← same process, different PID!
  PID 2: nginx worker
  PID 3: nginx worker

  # Inside container:
  $ ps aux
  PID  USER  COMMAND
  1    root  nginx: master process
  2    nginx nginx: worker process
  3    nginx nginx: worker process
  # Can't see host processes = isolation
```

> Container's PID 1 = the entrypoint process.
> If PID 1 dies → the container stops.
> Signal handling for PID 1 is special (no default SIGTERM handler).

---

# cgroups — Resource Limits & Accounting

> **Control Groups** limit, account for, and isolate **resource usage** (CPU, memory, I/O, network).

### cgroups v2 hierarchy (modern):

```
/sys/fs/cgroup/
├── system.slice/              ← system services
├── user.slice/                ← user sessions
└── kubepods.slice/            ← Kubernetes pods
    ├── kubepods-burstable.slice/
    │   └── kubepods-burstable-pod<UID>.slice/
    │       ├── cri-containerd-<ID>.scope/
    │       │   ├── cpu.max          → "100000 100000" (100% of 1 CPU)
    │       │   ├── memory.max       → "536870912" (512 MiB)
    │       │   ├── memory.current   → "234881024" (current usage)
    │       │   ├── io.max           → "8:0 rbps=104857600" (100MB/s)
    │       │   └── pids.max         → "1024"
```

### Key cgroup controllers:
| Controller | Manages |
|-----------|---------|
| `cpu` | CPU time (shares, quota, period) |
| `memory` | Memory limit, swap, OOM behavior |
| `io` | Block I/O bandwidth, IOPS |
| `pids` | Max number of processes |
| `cpuset` | Pin to specific CPUs/NUMA nodes |

---

# seccomp-bpf — System Call Filtering

> **seccomp** restricts which **system calls** a process can make.

```
Default Docker seccomp profile blocks ~44 of ~330+ syscalls:

BLOCKED (dangerous):                  ALLOWED (safe):
├── reboot()                         ├── read() / write()
├── kexec_load()                     ├── open() / close()
├── mount() / umount2()              ├── mmap() / mprotect()
├── swapon() / swapoff()             ├── socket() / connect()
├── init_module() / delete_module()  ├── fork() / clone()
├── acct()                           ├── execve()
├── settimeofday()                   ├── getpid() / getuid()
├── syslog()                         ├── stat() / fstat()
├── ptrace() *                       └── ... (most normal ops)
└── bpf() *
```

> **Defense in depth:** Even if an attacker escapes namespace isolation, seccomp blocks dangerous kernel interactions.

---

# Capabilities — Fine-Grained Privileges

> Linux splits the old **root / non-root** binary into **~40 individual capabilities**.

```
Traditional model:        Capabilities model:
  UID 0 = ALL power         CAP_NET_BIND_SERVICE → bind < 1024
  UID !0 = no power         CAP_NET_RAW          → raw sockets
                            CAP_SYS_ADMIN        → mount, bpf, ...
                            CAP_SYS_PTRACE       → ptrace
                            CAP_DAC_OVERRIDE     → bypass file perms
                            CAP_CHOWN            → change file owner
                            ... (~40 total)
```

### Docker default capability set (whitelist):
```
GRANTED (14):                       DROPPED (everything else):
  CAP_CHOWN                          CAP_SYS_ADMIN
  CAP_DAC_OVERRIDE                   CAP_NET_RAW (dropped by default now)
  CAP_FSETID                         CAP_SYS_PTRACE
  CAP_FOWNER                         CAP_SYS_MODULE
  CAP_NET_BIND_SERVICE               CAP_SYS_RAWIO
  CAP_SETGID / CAP_SETUID            CAP_SYS_TIME
  CAP_KILL / CAP_AUDIT_WRITE         ...
  CAP_SETPCAP / CAP_SETFCAP
  CAP_MKNOD / CAP_NET_RAW (*)
```

---

<!-- _class: lead -->

# Part 2
## Containerization Deep Dive

---

# What is a Container?

> A **container** is an **isolated process** (or group of processes) running on a shared kernel, with its own filesystem, network, and process view.

```
    Container = isolated process on shared kernel

    ┌─ Namespace isolation ──────────────────────┐
    │                                             │
    │  PID namespace  → own PID 1                 │
    │  NET namespace  → own eth0, routes, iptables│
    │  MNT namespace  → own root filesystem       │
    │  UTS namespace  → own hostname              │
    │  USER namespace → own uid mapping           │
    │                                             │
    │  + cgroup limits (CPU, mem, I/O)            │
    │  + seccomp filter (syscall whitelist)       │
    │  + capabilities (fine-grained privs)        │
    │  + OverlayFS (layered rootfs)               │
    │                                             │
    └─────────────────────────────────────────────┘
         │
         ▼
    Host Linux Kernel (shared by ALL containers)
```

---

# Container Images — Layered Filesystem

> A container image is a **stack of read-only layers** plus a thin read-write layer on top.

```
┌─────────────────────────────────────────────┐
│  Writable Container Layer (ephemeral)       │  ← changes here
├─────────────────────────────────────────────┤
│  Layer 4: COPY app.py /app/                 │  ← your code
├─────────────────────────────────────────────┤
│  Layer 3: RUN pip install flask             │  ← dependencies
├─────────────────────────────────────────────┤
│  Layer 2: RUN apt-get install python3       │  ← runtime
├─────────────────────────────────────────────┤
│  Layer 1: Ubuntu 24.04 base image           │  ← base OS
└─────────────────────────────────────────────┘

Storage driver: OverlayFS (overlay2)
  - Lower layers: read-only, shared between containers
  - Upper layer: read-write, container-specific (copy-on-write)
  - Merged view: union mount presented to the container
```

> **Efficiency:** 100 containers from the same image share the **same base layers** — only the writable layer is unique per container.

---

# OCI Standards — The Universal Contract

> The **Open Container Initiative (OCI)** defines open standards so images and runtimes are interchangeable.

### Three specifications:

| Spec | Defines | Key Points |
|------|---------|------------|
| **Image Spec** | Image format & layout | Layers, manifests, config JSON |
| **Runtime Spec** | How to run a container | `config.json` → namespaces, mounts, hooks |
| **Distribution Spec** | How to push/pull images | Registry API (Docker Hub, GHCR, ECR) |

```
OCI Image Manifest:
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": { "digest": "sha256:abc...", "size": 7023 },
  "layers": [
    { "digest": "sha256:def...", "size": 32654 },   ← base
    { "digest": "sha256:ghi...", "size": 16724 },   ← deps
    { "digest": "sha256:jkl...", "size": 73109 }    ← app
  ]
}
```

---

<!-- _class: lead -->

# Part 3
## Dockerfiles & Image Building

---

# Dockerfile — Anatomy

```dockerfile
# ─── Build stage ──────────────────────────────────
FROM golang:1.22-alpine AS builder
WORKDIR /src

# Cache dependencies separately from source code
COPY go.mod go.sum ./
RUN go mod download

# Copy source and build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# ─── Production stage ─────────────────────────────
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /app/server /server

EXPOSE 8080
USER nonroot:nonroot

ENTRYPOINT ["/server"]
```

### Key principles:
- **Multi-stage builds** → small final images (no compiler in production)
- **Layer caching** → put rarely-changing layers first (`go.mod` before source)
- **Non-root user** → security best practice
- **Distroless base** → minimal attack surface (no shell, no package manager)

---

# Dockerfile Best Practices

### DO ✅
```dockerfile
# Pin versions for reproducibility
FROM python:3.12.3-slim-bookworm

# Combine RUN commands to reduce layers
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# Copy dependency file first for caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Use .dockerignore to exclude unnecessary files
# Run as non-root
USER 1000:1000
```

### DON'T ❌
```dockerfile
FROM ubuntu:latest           # unpinned tag
RUN apt-get update           # separate from install (cache bug)
RUN apt-get install python3  # missing -y, missing cleanup
COPY . .                     # copies everything including .git
USER root                    # running as root in production
```

---

# Multi-Architecture Builds with `docker buildx`

```bash
# Create a multi-platform builder
docker buildx create --name mybuilder --use --bootstrap

# Build for multiple architectures simultaneously
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  --tag myregistry/myapp:1.0.0 \
  --push \
  .
```

### How it works:

```
docker buildx build --platform linux/amd64,linux/arm64
         │
         ▼
┌─────────────────────────────────────────┐
│  BuildKit (Moby buildkitd)              │
│                                         │
│  amd64: native build on x86_64 host    │
│  arm64: QEMU emulation (or remote)      │
│                                         │
│  → Multi-arch manifest (fat manifest)   │
│    ├── linux/amd64 → sha256:abc...      │
│    └── linux/arm64 → sha256:def...      │
└─────────────────────────────────────────┘
         │
         ▼
    Registry auto-selects correct image
    based on client architecture
```

---

<!-- _class: lead -->

# Part 4
## Container Runtimes
### Docker, Podman, containerd, CRI-O

---

# The Container Runtime Landscape

```
                                HIGH-LEVEL
                          (Image mgmt + Build + API)
                    ┌──────────────────────────────────┐
                    │  Docker Engine  │  Podman  │ Nerd │
                    │  (dockerd)      │          │(nerdctl)
                    └───────┬─────────┴────┬─────┴──┬──┘
                            │              │        │
                        MID-LEVEL                    
                    (Container lifecycle management)  
                    ┌───────┴──────────────┴────────┴──┐
                    │   containerd       │    CRI-O     │
                    │                    │              │
                    │  (Docker's core)   │  (K8s native)│
                    └───────┬────────────┴──────┬───────┘
                            │                   │
                        LOW-LEVEL (OCI Runtime)  
                    ┌───────┴───────────────────┴───────┐
                    │  runc   │  crun   │  kata  │ gVisor│
                    │ (ref    │ (fast,  │(micro  │(user  │
                    │  impl)  │  C)     │ VMs)   │space) │
                    └──────────────────────────────────┘
```

---

# Docker Engine — Architecture

```
                     docker CLI
                        │
                   REST API (unix socket / TCP)
                        │
                        ▼
               ┌─────────────────┐
               │    dockerd      │  ← Docker daemon
               │  (Docker Engine)│     Image builds, networking,
               │                 │     volumes, orchestration
               └────────┬────────┘
                        │ gRPC
                        ▼
               ┌─────────────────┐
               │   containerd    │  ← Container runtime
               │                 │     Lifecycle, snapshots,
               │                 │     image pull/push
               └────────┬────────┘
                        │ OCI runtime exec
                        ▼
               ┌─────────────────┐
               │   runc          │  ← OCI runtime
               │                 │     clone() + execve()
               │                 │     namespaces + cgroups
               └─────────────────┘
```

> **Docker = dockerd + containerd + runc** — three layers.
> Kubernetes talks to **containerd directly**, skipping dockerd.

---

# Podman — Daemonless Alternative

```
                     podman CLI
                        │
                    (direct, no daemon)
                        │
                        ▼
               ┌─────────────────┐
               │    Podman       │  ← Library (libpod)
               │   (no daemon!)  │     Fork + exec model
               │                 │     Rootless by default
               └────────┬────────┘
                        │
                        ▼
               ┌─────────────────┐
               │   conmon        │  ← Container monitor
               │                 │     Holds stdio, exit code
               └────────┬────────┘
                        │
                        ▼
               ┌─────────────────┐
               │ crun (or runc)  │  ← OCI runtime
               └─────────────────┘
```

### Key differences from Docker:
- **No daemon** — each `podman` call is a direct process
- **Rootless by default** — runs entirely in user namespaces
- **Pod-native** — `podman pod` directly models K8s pods
- **Docker CLI compatible** — `alias docker=podman` works
- **systemd integration** — `podman generate systemd`

---

# Docker vs Podman vs Buildah vs Skopeo

| Tool | Role | Daemon? | Root? |
|------|------|---------|-------|
| **Docker** | Build + Run + Push/Pull | Yes (dockerd) | Yes (default) |
| **Podman** | Run containers (+ build) | No | No (rootless) |
| **Buildah** | Build OCI images | No | No (rootless) |
| **Skopeo** | Copy/inspect images between registries | No | No |
| **Buildx** | Docker multi-arch builder (BuildKit) | Yes (BuildKit) | Yes |

### The Red Hat container toolchain:
```
Building images:     Buildah (or Podman build)
Running containers:  Podman
Moving images:       Skopeo
Kubernetes runtime:  CRI-O
OCI runtime:         crun (or runc)
```

### The Docker toolchain:
```
Building images:     docker build (or docker buildx)
Running containers:  docker run (dockerd → containerd → runc)
Moving images:       docker push/pull
Kubernetes runtime:  containerd
OCI runtime:         runc
```

---

# OCI Runtimes — The Lowest Layer

| Runtime | Language | Key Feature |
|---------|----------|-------------|
| **runc** | Go | Reference implementation, Docker default |
| **crun** | C | 2× faster startup, lower memory, Podman default |
| **Kata Containers** | Go | Runs each container in a **lightweight VM** (hardware isolation) |
| **gVisor (runsc)** | Go | User-space kernel — intercepts syscalls (Google) |
| **Firecracker** | Rust | MicroVM backend for AWS Lambda / Fargate |
| **youki** | Rust | Rust rewrite of runc |

### Security spectrum:

```
Less isolation ◄──────────────────────────────► More isolation
   (faster)                                      (slower)

  runc/crun         gVisor            Kata / Firecracker
  namespaces        user-space        micro-VMs
  + cgroups         kernel            + hardware isolation
  (shared kernel)   (syscall proxy)   (separate kernel per container)
```

---

<!-- _class: lead -->

# Part 5
## Kubernetes Architecture

---

# What is Kubernetes?

> **Kubernetes (K8s)** is an open-source **container orchestration platform** that automates deployment, scaling, and management of containerized applications.

### What K8s manages:
- **Scheduling** — which node runs which container
- **Scaling** — horizontal pod autoscaling (HPA)
- **Networking** — service discovery, load balancing, ingress
- **Storage** — persistent volumes, storage classes
- **Self-healing** — restart failed containers, reschedule on healthy nodes
- **Rolling updates** — zero-downtime deployments
- **Config & secrets** — centralized configuration management

### What K8s is NOT:
- Not a PaaS (no app-level framework)
- Not a CI/CD system (but integrates with them)
- Not a VM orchestrator (that's OpenStack, vSphere)

---

# Kubernetes Architecture — The Big Picture

```
┌─────────────────── Control Plane ───────────────────┐
│                                                      │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ API      │  │ etcd         │  │ Scheduler     │  │
│  │ Server   │  │ (consensus   │  │ (where to     │  │
│  │ (REST)   │  │  store)      │  │  place pods)  │  │
│  └────┬─────┘  └──────────────┘  └───────────────┘  │
│       │                                              │
│  ┌────┴─────────────────┐  ┌──────────────────────┐  │
│  │ Controller Manager   │  │ Cloud Controller Mgr │  │
│  │ (reconciliation      │  │ (cloud-specific:     │  │
│  │  loops)              │  │  LB, nodes, routes)  │  │
│  └──────────────────────┘  └──────────────────────┘  │
└──────────────────────────────────────────────────────┘
              │ kubectl / API calls
              ▼
┌─────────────── Worker Nodes ────────────────────────┐
│  Node 1              Node 2              Node 3     │
│  ┌──────────┐       ┌──────────┐       ┌────────┐  │
│  │ kubelet  │       │ kubelet  │       │kubelet │  │
│  │ CRI ─────┤       │ CRI ─────┤       │CRI ───┤  │
│  │ runtime  │       │ runtime  │       │runtime │  │
│  │ kube-    │       │ kube-    │       │kube-   │  │
│  │  proxy   │       │  proxy   │       │ proxy  │  │
│  └──────────┘       └──────────┘       └────────┘  │
└─────────────────────────────────────────────────────┘
```

---

# Control Plane Components

### API Server (`kube-apiserver`)
- **Front door** for all K8s operations
- RESTful API — all communication goes through here
- Authentication, authorization (RBAC), admission control
- Stores/retrieves state from etcd

### etcd
- **Distributed key-value store** (Raft consensus)
- Single source of truth for all cluster state
- All K8s objects stored here (pods, services, secrets, configs)

### Scheduler (`kube-scheduler`)
- Decides **which node** a new pod should run on
- Considers: resources, affinity, anti-affinity, taints, tolerations

### Controller Manager (`kube-controller-manager`)
- Runs **reconciliation loops** (controllers)
- Deployment controller, ReplicaSet controller, Node controller, etc.
- Watches desired state vs actual state → takes corrective action

---

# Worker Node Components

### kubelet
- **Agent** running on every node
- Receives pod specs from API server
- Calls the **Container Runtime** via **CRI** to start/stop containers
- Reports node and pod status back to control plane
- Manages liveness/readiness probes

### Container Runtime
- Actually runs the containers
- Must implement the **CRI (Container Runtime Interface)**
- Options: **containerd**, **CRI-O**, (Docker via cri-dockerd — deprecated)

### kube-proxy
- Manages network rules on each node
- Implements Kubernetes **Services** (ClusterIP, NodePort, LoadBalancer)
- Modes: iptables (default), IPVS (high-performance), nftables (new)

---

# The Pod — Kubernetes' Atomic Unit

```
┌─────────────────── Pod ──────────────────────┐
│                                               │
│  Shared:                                      │
│  ├── Network namespace (same IP, localhost)   │
│  ├── IPC namespace                            │
│  ├── Volumes (shared storage)                 │
│  └── (optionally) PID namespace               │
│                                               │
│  ┌─────────────┐  ┌──────────────────┐        │
│  │ Container 1 │  │ Container 2      │        │
│  │ (main app)  │  │ (sidecar/proxy)  │        │
│  │             │  │                  │        │
│  │ Port 8080   │  │ Port 15001       │        │
│  └─────────────┘  └──────────────────┘        │
│                                               │
│  ┌─────────────────────────────┐              │
│  │ Init Container(s)          │ ← run first  │
│  │ (setup, migration, etc.)   │   then exit  │
│  └─────────────────────────────┘              │
│                                               │
│  Pod IP: 10.244.1.5 (all containers share)    │
│  Node: worker-02                              │
└───────────────────────────────────────────────┘
```

> All containers in a pod share the **same network namespace** — they can reach each other on `localhost`.

---

<!-- _class: lead -->

# Part 6
## CRI — Container Runtime Interface
### The Bridge Between kubelet and Containers

---

# Why CRI Exists

### The Docker Problem (pre-CRI):

```
Before CRI (K8s < 1.5):

  kubelet ──── Docker-specific code ──── dockerd ──── containerd ──── runc

  Problems:
  ✗ Kubelet was hardcoded to Docker's API
  ✗ Adding a new runtime = modifying kubelet source code
  ✗ Docker daemon had features K8s didn't need (build, swarm, etc.)
  ✗ Extra layer of indirection (kubelet→dockerd→containerd→runc)
```

### The CRI Solution (K8s 1.5+, stable 1.26+):

```
After CRI:

  kubelet ──── CRI (gRPC) ──── containerd ──── runc
                    or
  kubelet ──── CRI (gRPC) ──── CRI-O     ──── runc/crun

  Benefits:
  ✓ kubelet is runtime-agnostic
  ✓ Any CRI-compliant runtime works
  ✓ Direct path — no unnecessary Docker daemon
  ✓ Kubernetes removed dockershim in v1.24
```

---

# CRI — The gRPC Interface

> CRI defines a **gRPC protocol** with two services:

### 1. RuntimeService — Container Lifecycle
```protobuf
service RuntimeService {
    // Sandbox (pod-level) operations
    rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse);
    rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse);
    rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse);

    // Container operations
    rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse);
    rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);
    rpc StopContainer(StopContainerRequest) returns (StopContainerResponse);
    rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse);

    // Exec / Attach / Port-forward
    rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse);
    rpc Exec(ExecRequest) returns (ExecResponse);
    rpc Attach(AttachRequest) returns (AttachResponse);
}
```

### 2. ImageService — Image Management
```protobuf
service ImageService {
    rpc PullImage(PullImageRequest) returns (PullImageResponse);
    rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse);
    rpc ListImages(ListImagesRequest) returns (ListImagesResponse);
    rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse);
}
```

---

# CRI-O — Purpose-Built for Kubernetes

> **CRI-O** is a lightweight CRI implementation that does **one thing**: run containers for Kubernetes.

```
┌───────────────── kubelet ──────────────────┐
│                                             │
│  "I need a pod with nginx:1.27 container"  │
│                                             │
└──────────────────┬──────────────────────────┘
                   │ CRI gRPC (unix socket)
                   ▼
┌───────────────── CRI-O ───────────────────┐
│                                            │
│  1. Pull image from registry               │
│  2. Create pod sandbox (pause container)   │
│  3. Set up networking (CNI plugin call)    │
│  4. Create container in sandbox            │
│  5. Invoke OCI runtime (runc/crun)         │
│  6. Monitor via conmon                     │
│                                            │
│  Scope: ONLY what K8s needs               │
│  No build, no docker CLI, no swarm        │
│                                            │
└──────────────────┬─────────────────────────┘
                   │ OCI spec
                   ▼
┌───────────────── runc / crun ─────────────┐
│  clone() → unshare() → pivot_root()       │
│  → execve() the container entrypoint      │
└────────────────────────────────────────────┘
```

---

# CRI-O vs containerd — Comparison

| Feature | CRI-O | containerd |
|---------|-------|------------|
| **Purpose** | K8s-only CRI runtime | General-purpose container runtime |
| **Versioning** | Matches K8s versions (1.30.x) | Independent releases |
| **Scope** | Minimal — CRI + OCI | Broader — CRI + Docker + others |
| **Used by Docker?** | No | Yes (Docker's core runtime) |
| **Default in** | OpenShift, SUSE Rancher | GKE, EKS, AKS, kubeadm default |
| **Image pull** | containers/image library | Own image pull implementation |
| **Networking** | CNI plugins | CNI plugins |
| **Storage** | containers/storage | Own snapshotter framework |
| **Build images?** | No | No (but nerdctl can) |
| **Configuration** | Drop-in config files | TOML config |
| **OCI runtimes** | runc, crun, Kata, gVisor | runc, gVisor, Kata |

> **Rule of thumb:** CRI-O = lean & K8s-only. containerd = versatile & widely adopted.

---

# The Full Pod Start Sequence

```
kubectl apply -f pod.yaml
    │
    ▼
API Server → stores in etcd
    │
    ▼
Scheduler → assigns to Node-2
    │ (binding written to etcd)
    ▼
kubelet on Node-2 (watches API server)
    │
    ▼ CRI: RunPodSandbox()
Container Runtime (CRI-O / containerd)
    │
    ├── 1. Create pod sandbox (pause container with new namespaces)
    ├── 2. Call CNI plugin → allocate IP, set up veth pair
    ├── 3. CRI: CreateContainer() → prepare rootfs (overlay mount)
    ├── 4. CRI: StartContainer() → invoke OCI runtime
    │       └── runc/crun: clone(NEWNS|NEWPID|NEWNET|...)
    │                      → pivot_root() → execve("nginx")
    ├── 5. conmon monitors container stdio + exit
    └── 6. kubelet reports pod status → API server → etcd

Total time: 1-3 seconds (for a cached image)
```

---

<!-- _class: lead -->

# Part 7
## Bare Metal to K8s — The Full Journey

---

# The Complete Evolution

```
LEVEL 0: BARE METAL
┌──────────────────────────┐
│  App A     App B    App C│  1 server = 1 (or few) apps
│  ════════════════════════│  Low utilization, slow provisioning
│  Host OS (Linux)         │
│  Physical Hardware       │
└──────────────────────────┘
         │ need isolation + better utilization
         ▼
LEVEL 1: VIRTUALIZATION
┌──────────────────────────┐
│ ┌──VM──┐ ┌──VM──┐       │  Hardware-level isolation
│ │App A │ │App B │  ...   │  Full OS per workload
│ │OS    │ │OS    │        │  Minutes to provision
│ └──────┘ └──────┘        │
│ Hypervisor (KVM/ESXi)    │
│ Physical Hardware        │
└──────────────────────────┘
         │ need faster scaling + less overhead
         ▼
LEVEL 2: CONTAINERIZATION
┌──────────────────────────┐
│ ┌────┐ ┌────┐ ┌────┐    │  Process-level isolation
│ │ A  │ │ B  │ │ C  │... │  Shared kernel
│ └────┘ └────┘ └────┘    │  Seconds to provision
│ Container Runtime        │
│ Host OS (Linux Kernel)   │
│ (VM or Bare Metal)       │
└──────────────────────────┘
         │ need orchestration at scale
         ▼
LEVEL 3: KUBERNETES
┌──────────────────────────┐
│ K8s Control Plane        │  Automated scheduling, scaling,
│ ┌─Node─┐ ┌─Node─┐       │  self-healing, networking,
│ │Pod Pod│ │Pod Pod│ ...  │  service discovery, config mgmt
│ │CRI   │ │CRI   │       │  Declarative desired-state model
│ └──────┘ └──────┘        │
│ Infrastructure (VMs/BM)  │
└──────────────────────────┘
```

---

# Building a K8s Cluster from Bare Metal Linux

### Step-by-step:

```
1. PREPARE THE NODE (bare metal Linux)
   ├── Disable swap:           swapoff -a
   ├── Load kernel modules:    overlay, br_netfilter
   ├── Set sysctl:             net.bridge.bridge-nf-call-iptables = 1
   │                           net.ipv4.ip_forward = 1
   └── Install container runtime (choose one):
       ├── containerd (+ CNI plugins)
       └── CRI-O

2. INSTALL KUBERNETES COMPONENTS
   ├── kubeadm  (cluster bootstrapper)
   ├── kubelet  (node agent)
   └── kubectl  (CLI client)

3. INITIALIZE CONTROL PLANE (first node)
   └── kubeadm init --pod-network-cidr=10.244.0.0/16

4. INSTALL CNI PLUGIN (pod networking)
   └── kubectl apply -f calico.yaml  (or Cilium, Flannel, ...)

5. JOIN WORKER NODES
   └── kubeadm join <control-plane>:6443 --token <token> ...

6. VERIFY
   └── kubectl get nodes  →  Ready, Ready, Ready
```

---

# The Network Stack — What Actually Happens

```
Internet
    │
    ▼
┌─────────────── Physical NIC (eth0) ─────────────┐
│  IP: 10.0.0.100                                  │
│                                                  │
│  kube-proxy (iptables/IPVS rules)                │
│  ├── NodePort 30080 → Service ClusterIP → Pod IP │
│  └── LoadBalancer → External IP → Pods           │
│                                                  │
│  ┌──── CNI (Calico/Cilium) ─────────────────┐   │
│  │  Pod Network: 10.244.0.0/16              │   │
│  │                                           │   │
│  │  ┌── Pod A ──┐     ┌── Pod B ──┐         │   │
│  │  │ eth0      │     │ eth0      │         │   │
│  │  │10.244.1.5 │←───→│10.244.1.6 │         │   │
│  │  └───────────┘     └───────────┘         │   │
│  │     veth             veth                 │   │
│  │       └──── bridge/tunnel/eBPF ────┘      │   │
│  └───────────────────────────────────────────┘   │
│                                                  │
│  Cross-node: VXLAN / IPIP / BGP / WireGuard      │
└──────────────────────────────────────────────────┘
```

---

# Analogy: Medical Triage → Kubernetes Scheduling

> **Medical Triage** determines the **priority** of patient admission based on urgency.
>
> **Kubernetes Scheduling** determines the **priority** and **placement** of pods based on resource needs and constraints.

| Medical Triage | Kubernetes Scheduling |
|----------------|----------------------|
| **Immediate** (Red) — life-threatening | **PriorityClass: system-critical** — must run first |
| **Urgent** (Orange) — serious but stable | **Guaranteed QoS** — resources reserved |
| **Standard** (Yellow) — can wait | **Burstable QoS** — requests < limits |
| **Non-urgent** (Green) — minor | **BestEffort QoS** — no guarantees |
| **Deceased** — no treatment | **Evicted/Preempted** — killed for higher priority |
| **Available beds** determine placement | **Available node resources** determine scheduling |
| **Specialist wards** (cardio, neuro) | **Node affinity / taints** (GPU node, high-mem) |

> Both systems: **assess → classify → prioritize → assign resources** under constraints.

---

# Summary — The Full Stack Map

```
Layer 7: APPLICATION
  │  Your microservices, APIs, frontends, ML models
  │
Layer 6: ORCHESTRATION
  │  Kubernetes (scheduling, scaling, networking, self-healing)
  │  Helm charts, Operators, GitOps (ArgoCD/Flux)
  │
Layer 5: CONTAINER RUNTIME INTERFACE (CRI)
  │  kubelet ← gRPC → containerd / CRI-O
  │
Layer 4: CONTAINER RUNTIME
  │  containerd, CRI-O, Podman (standalone)
  │
Layer 3: OCI RUNTIME
  │  runc, crun, Kata Containers, gVisor
  │
Layer 2: LINUX KERNEL PRIMITIVES
  │  Namespaces, cgroups, seccomp, capabilities, OverlayFS
  │
Layer 1: OPERATING SYSTEM
  │  Linux (Ubuntu, RHEL, Flatcar, Talos)
  │
Layer 0: INFRASTRUCTURE
     Bare Metal  or  Virtual Machines (Type 1 hypervisor)
```

---

# 🧠 Session 2 — Key Takeaways

1. **Containers are NOT a kernel feature** — they're built from namespaces + cgroups + seccomp + capabilities
2. **Network namespaces** give each container its own full network stack
3. **OCI standards** ensure images and runtimes are interchangeable
4. **Multi-stage Dockerfiles** with non-root users are the gold standard
5. **Docker ≠ the only option** — Podman (daemonless), Buildah, CRI-O are production-proven
6. **CRI** decoupled Kubernetes from Docker — any CRI-compliant runtime works
7. **CRI-O** is purpose-built for K8s; **containerd** is more general-purpose
8. **The real-world stack**: App → K8s → CRI → containerd/CRI-O → runc → Linux kernel → VM → Hypervisor → Hardware

---

# 📖 Recommended Reading & Resources

### Specifications
- [OCI Image Spec](https://github.com/opencontainers/image-spec)
- [OCI Runtime Spec](https://github.com/opencontainers/runtime-spec)
- [CRI API protobuf](https://github.com/kubernetes/cri-api)

### Books
- Brendan Burns, *"Kubernetes: Up and Running"* (3rd ed., O'Reilly)
- Liz Rice, *"Container Security"* (O'Reilly) — Linux primitives deep-dive
- Michael Hausenblas, *"Learning Modern Linux"* (O'Reilly)

### Hands-on
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) — Kelsey Hightower
- [Play with Kubernetes](https://labs.play-with-k8s.com/) — browser-based lab
- `unshare` + `nsenter` — build a container from scratch in 20 commands

---

<!-- _class: lead -->

# ❓ Questions & Discussion

### Discussion prompts:
1. *Why did Kubernetes remove Docker support (dockershim)?*
2. *When would you choose CRI-O over containerd?*
3. *What stops a container from escaping to the host?*
4. *How does the triage analogy map to your team's deployment priorities?*

---

<!-- _class: lead -->

# Thank You!

### 🖥️ Session 1: Bare Metal → Virtualization
### 🐳 Session 2: Containers → Kubernetes

> *"You can't build cloud-native without understanding what's beneath the clouds."*

### Next steps for your team:
1. Run `unshare --pid --net --mount --fork /bin/bash` on a test box
2. Build a multi-stage Dockerfile for one of your services
3. Try `kubeadm init` on a spare node
4. Explore `crictl` to interact with CRI directly
