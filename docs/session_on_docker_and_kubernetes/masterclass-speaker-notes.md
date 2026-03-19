# Master Class — Speaker Notes & Teaching Guide
## From Bare Metal to Kubernetes (2 × 90 min)

---

## SESSION 1: From Bare Metal to Virtualization

### Slide Timing Guide

| Slide(s) | Topic | Mins | Teaching Notes |
|-----------|-------|------|----------------|
| 1–2 | Title + Agenda | 2 | Set expectations: "By end of today, you'll understand everything between hardware and VMs" |
| 3–6 | Bare Metal | 15 | **Ask the room:** "Who has installed an OS on bare metal?" Start with what they know. Highlight the utilization problem — this motivates everything that follows. |
| 7–9 | What is Virtualization | 10 | History slide is a great storytelling moment. The Popek & Goldberg theorem is the "aha" — x86 wasn't virtualizable until 2005! |
| 10–13 | Hypervisor Core | 20 | **Key teaching point:** Explain Type 1 vs Type 2 with the building analogy: Type 1 = the building foundation itself, Type 2 = a room inside someone else's building. KVM slide is critical — "Linux IS the hypervisor" blows minds. |
| 14–17 | Reference Model | 15 | Use the traffic controller analogy for Dispatcher. Walk through the CR3 example step by step — this makes it concrete. Poll: "What happens when a VM tries to reboot?" |
| 18–21 | Virt vs Container | 10 | This is the bridge to Session 2. Key message: "In production, you use BOTH." The full stack diagram at the end is the money shot. |
| 22–23 | Takeaways + Q&A | 10 | Recap the 7 key points. Open discussion. |

---

### Key Stories & Analogies to Use

#### The Hotel Analogy (Virtualization)
> "Think of a hypervisor as a hotel building. Each VM is a complete hotel room with its own bathroom, kitchen, and bedroom. Guests are fully isolated — what happens in room 301 doesn't affect room 302. But each room takes significant space and resources."

#### The Apartment Analogy (Containerization — preview)
> "Containers are like apartments in a shared building. They share plumbing (kernel), electrical (CPU scheduler), and foundation (hardware). Each has its own locked door (namespaces) and utility meter (cgroups). Much more efficient, but the building superintendent (kernel) is a shared dependency."

#### The Traffic Controller (Dispatcher)
> "The dispatcher is a traffic controller at an intersection. It doesn't drive any car, it doesn't fuel any car — it just decides which lane each car goes to. Privileged instruction? Go to the Interpreter lane. Resource change? Go to the Allocator lane."

#### The x86 Problem — Tell It As a Mystery
> "In 1974, Popek and Goldberg proved that you CAN virtualize any architecture... IF sensitive instructions always trap. For 25 years, x86 couldn't do this — 17 instructions were sensitive but didn't trap. VMware's genius was binary translation: rewrite the guest code on-the-fly to replace those sneaky instructions with safe trapping versions. Then Intel said 'fine, we'll fix it in hardware' — VT-x in 2005."

---

### Common Questions & Answers

**Q: Is Docker a hypervisor?**
A: No. Docker is a container runtime that uses Linux kernel primitives (namespaces, cgroups). It doesn't run full operating systems — just isolated processes on a shared kernel.

**Q: Is KVM Type 1 or Type 2?**
A: This is debated! Technically it's Type 1 — the KVM module makes the Linux kernel itself into a hypervisor. But since you still have a full Linux userspace, some argue it's a "hybrid." The practical answer: it delivers Type 1 performance with Type 2 convenience.

**Q: Why can't I just use containers for everything?**
A: Containers share the host kernel — a kernel vulnerability affects ALL containers. VMs provide hardware-level isolation. In regulated environments (banking, defense, healthcare), VM isolation is often a compliance requirement. Also, you can't run Windows containers on a Linux kernel (native — WSL2 uses a VM).

**Q: What about WSL2?**
A: WSL2 is actually a lightweight Hyper-V VM running a real Linux kernel. It's Type 1 virtualization (Hyper-V is bare metal, Windows runs in the root partition) with a great developer experience layer on top.

---

## SESSION 2: From Containers to Kubernetes

### Slide Timing Guide

| Slide(s) | Topic | Mins | Teaching Notes |
|-----------|-------|------|----------------|
| 1–2 | Title + Agenda | 2 | Quick recap of Session 1's key points before diving in |
| 3–8 | Linux Kernel Primitives | 15 | **DEMO OPPORTUNITY:** Run `unshare` live. Show `lsns`. Create a network namespace and show isolated `ip addr`. This is the most educational part — demystify the "magic" of containers. |
| 9–11 | Containerization | 10 | The layer diagram and OCI spec slide are key. Emphasize: "A container image is just a tarball of filesystem layers + metadata JSON." |
| 12–14 | Dockerfiles & Buildx | 15 | **LIVE CODING:** Write a Dockerfile together. Show the DO vs DON'T side by side. Multi-stage builds are the #1 practical takeaway. |
| 15–19 | Container Runtimes | 15 | The landscape diagram is the anchor slide. Key message: "Docker is 3 layers: dockerd → containerd → runc. Kubernetes skips dockerd." |
| 20–23 | Kubernetes Architecture | 15 | **Draw on whiteboard:** Start with "a user types kubectl apply" and trace the full path. The pod start sequence is the master slide. |
| 24–28 | CRI Deep Dive | 10 | The protobuf definitions make it concrete — CRI is just a gRPC API. CRI-O vs containerd comparison is a common team decision point. |
| 29–31 | Full Journey + Triage | 5 | The triage analogy lands well with mixed audiences. The full stack map is the synthesis of both sessions. |
| 32–33 | Takeaways + Q&A | 5 | End with the hands-on next steps. |

---

### Live Demo Script (Session 2)

#### Demo 1: Build a Container From Scratch (5 min)

```bash
# Show current namespaces
lsns

# Create a new PID + mount + UTS namespace
sudo unshare --pid --mount --uts --fork /bin/bash

# Inside the new namespace:
hostname container-demo
hostname  # shows "container-demo"

ps aux    # only shows processes in this namespace!
# PID 1 is our bash shell

# Exit and show host hostname is unchanged
exit
hostname  # still the original hostname
```

#### Demo 2: Network Namespace (5 min)

```bash
# Create a network namespace
sudo ip netns add demo-ns

# Show it's completely empty (no interfaces)
sudo ip netns exec demo-ns ip addr
# Only loopback, and it's DOWN

# Bring up loopback
sudo ip netns exec demo-ns ip link set lo up

# Create a veth pair (virtual ethernet cable)
sudo ip link add veth-host type veth peer name veth-ns

# Move one end into the namespace
sudo ip link set veth-ns netns demo-ns

# Assign IPs
sudo ip addr add 10.0.0.1/24 dev veth-host
sudo ip link set veth-host up
sudo ip netns exec demo-ns ip addr add 10.0.0.2/24 dev veth-ns
sudo ip netns exec demo-ns ip link set veth-ns up

# Ping across the namespace boundary!
ping -c 2 10.0.0.2

# Cleanup
sudo ip netns del demo-ns
```

#### Demo 3: cgroup resource limit (3 min)

```bash
# Create a cgroup with 50MB memory limit (cgroups v2)
sudo mkdir /sys/fs/cgroup/demo
echo "52428800" | sudo tee /sys/fs/cgroup/demo/memory.max

# Run a process in that cgroup
echo $$ | sudo tee /sys/fs/cgroup/demo/cgroup.procs

# Try to allocate more than 50MB → OOM killed!
python3 -c "x = ' ' * 60_000_000"
# Killed!
```

#### Demo 4: crictl basics (2 min, needs a K8s node)

```bash
# List pods via CRI
sudo crictl pods

# List containers
sudo crictl ps

# Inspect a container
sudo crictl inspect <container-id>

# Pull an image via CRI
sudo crictl pull nginx:1.27

# Check runtime info
sudo crictl info
```

---

### Common Questions & Answers (Session 2)

**Q: Why did Kubernetes remove Docker support?**
A: Kubernetes never "used Docker" — it used containerd (inside Docker). The `dockershim` was a translation layer in kubelet that converted CRI calls to Docker API calls, which then called containerd anyway. Removing it: (a) eliminated a maintenance burden, (b) removed an unnecessary indirection layer, (c) let kubelet talk directly to containerd via CRI. Your container images still work — they're OCI standard.

**Q: Should we use CRI-O or containerd?**
A: Both are production-grade.
- **containerd** if: you want the most widely adopted option with the largest community (default for GKE, EKS, AKS kubeadm).
- **CRI-O** if: you want a minimal, K8s-only runtime with version-locked releases (default for OpenShift, Rancher).
Neither is "better" — it's organizational preference.

**Q: Are containers less secure than VMs?**
A: **Yes, by default.** Containers share the host kernel → a kernel exploit affects all containers. VMs have hardware isolation (VT-x, separate kernel per VM). However, container security can be hardened significantly with: seccomp profiles, AppArmor/SELinux, rootless containers, read-only rootfs, network policies, and tools like Falco. For maximum isolation, use Kata Containers (container UX, VM isolation).

**Q: What is a "pause" container?**
A: When CRI-O/containerd creates a pod, they first start a tiny "pause" container (literally does `pause()` syscall — sleeps forever). This container holds the pod's network namespace alive. When you add application containers to the pod, they join this existing namespace. If the app container crashes and restarts, the network namespace (and IP address) survive because the pause container is still running.

**Q: Can I run Kubernetes on bare metal?**
A: Absolutely — and many high-performance workloads do (no hypervisor overhead). Tools for bare metal K8s: kubeadm, k3s, Talos Linux, Flatcar Container Linux, Tinkerbell for provisioning, MetalLB for load balancing, Rook/Ceph for storage.

---

### Whiteboard Diagrams to Draw Live

#### 1. "What Happens When You Type `kubectl run nginx --image=nginx`"

Draw step by step:
1. kubectl → API Server (REST call)
2. API Server → etcd (store pod spec)
3. Scheduler watches → picks node
4. Scheduler → API Server (binding)
5. kubelet on chosen node watches → sees new pod
6. kubelet → CRI-O (RunPodSandbox)
7. CRI-O → runc (create pause container + namespaces)
8. CRI-O → CNI (setup networking, assign IP)
9. kubelet → CRI-O (CreateContainer, StartContainer)
10. CRI-O → runc (unshare + execve nginx)
11. kubelet → API Server (pod status: Running)

#### 2. "The Isolation Stack"

Draw as concentric security rings:
- Outer: Hardware (separate machines)
- Next: VMs (hypervisor isolation, separate kernels)
- Next: Containers (namespace + cgroup isolation, shared kernel)
- Next: Processes (standard OS isolation)
- Inner: Threads (shared address space)

Each ring = different cost/performance/isolation tradeoff.

---

## General Teaching Tips

1. **Start each concept with WHY, then HOW.** "Before we explain cgroups, let's understand why you need them — imagine 50 containers and one starts eating all the memory..."

2. **Use the "zoom in" technique.** Show the full stack diagram → "Today we're zooming into THIS layer."

3. **Every 15 minutes, interact.** Ask a question, run a demo, or do a quick poll. 90 minutes of pure slides = sleeping audience.

4. **The triage analogy works.** Your team likely knows medical triage from common knowledge. Map it: "immediate = critical pods, urgent = guaranteed QoS, standard = burstable, non-urgent = best-effort, deceased = evicted."

5. **End with hands-on homework.** Give specific commands to try. People remember what they do, not what they hear.
