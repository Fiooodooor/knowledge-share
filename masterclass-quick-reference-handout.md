# Quick Reference — Bare Metal to Kubernetes
## Master Class Handout (1-pager per session)

---

## SESSION 1 CHEAT SHEET: Bare Metal → Virtualization

### The Stack
```
VMs → Hypervisor → Hardware
```

### Hypervisor Types
| | Type 1 (Bare Metal) | Type 2 (Hosted) |
|-|---------------------|-----------------|
| **Runs on** | Hardware directly | Host OS |
| **Performance** | Near-native | 10–30% overhead |
| **Use** | Production / Cloud | Dev / Test |
| **Examples** | ESXi, KVM, Hyper-V, Xen | VirtualBox, VMware Workstation |

### Hypervisor Reference Model (Popek & Goldberg, 1974)
```
DISPATCHER  → Entry point, routes traps
ALLOCATOR   → Manages resources for VMs
INTERPRETER → Emulates privileged instructions
```

### Hardware Virtualization Extensions
- **Intel VT-x / AMD-V** — CPU virtualization (VMENTER/VMEXIT)
- **Intel VT-d / AMD-Vi** — I/O virtualization (IOMMU, DMA isolation)
- **SR-IOV** — NIC hardware partitioning (virtual functions)

### Key Commands
```bash
# Check if CPU supports virtualization
grep -E 'vmx|svm' /proc/cpuinfo

# Check KVM availability
lsmod | grep kvm

# List VMs (libvirt)
virsh list --all
```

---

## SESSION 2 CHEAT SHEET: Containers → Kubernetes

### The Container "Recipe"
```
Container = Namespaces + cgroups + seccomp + capabilities + rootfs
```

### Linux Namespaces
| NS | Isolates | Flag |
|----|----------|------|
| mnt | Filesystems | CLONE_NEWNS |
| pid | Process IDs | CLONE_NEWPID |
| net | Network stack | CLONE_NEWNET |
| uts | Hostname | CLONE_NEWUTS |
| ipc | IPC resources | CLONE_NEWIPC |
| user | UID/GID | CLONE_NEWUSER |
| cgroup | cgroup view | CLONE_NEWCGROUP |

### Container Runtime Stack
```
HIGH: Docker / Podman / nerdctl     (UX: build, run, push)
MID:  containerd / CRI-O            (lifecycle: create, start, stop)
LOW:  runc / crun / kata / gVisor   (OCI: clone, unshare, execve)
```

### CRI = Container Runtime Interface
```
kubelet ←─ gRPC ─→ containerd or CRI-O ──→ runc/crun
```
Two services: **RuntimeService** (pod/container lifecycle) + **ImageService** (pull/list/remove)

### Kubernetes Components
```
CONTROL PLANE:                 WORKER NODE:
├── API Server (REST)          ├── kubelet (CRI client)
├── etcd (state store)         ├── container runtime
├── Scheduler (placement)      └── kube-proxy (networking)
└── Controller Manager (loops)
```

### Pod = Scheduling Unit
- Shares: network namespace, IPC, volumes
- Each pod gets its own IP
- "pause" container holds the namespace

### Docker vs Podman
| | Docker | Podman |
|-|--------|--------|
| Daemon | Yes (dockerd) | No |
| Root | Default | Rootless default |
| Build | docker build / buildx | podman build / buildah |
| K8s | via containerd | via CRI-O |

### Essential Commands
```bash
# Namespaces
unshare --pid --net --mount --fork /bin/bash
lsns
nsenter -t <PID> -n                  # enter network ns

# cgroups
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/cpu.max

# Docker / Podman
docker build -t myapp:1.0 .
docker run --rm -p 8080:80 myapp:1.0
podman pod create --name mypod

# Kubernetes
kubeadm init --pod-network-cidr=10.244.0.0/16
kubeadm join <cp>:6443 --token <tok>
kubectl get nodes
kubectl get pods -A

# CRI (on K8s node)
crictl pods
crictl ps
crictl info
crictl inspect <id>
```

### Dockerfile Best Practices
```dockerfile
# Multi-stage, pinned version, non-root
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

### Security Layers (defense in depth)
```
1. Namespaces        → visibility isolation
2. cgroups           → resource limits
3. seccomp           → syscall filtering
4. Capabilities      → fine-grained privileges
5. AppArmor/SELinux  → mandatory access control
6. Network Policies  → pod-to-pod firewall
7. Read-only rootfs  → immutable filesystem
8. Non-root user     → no UID 0
```
