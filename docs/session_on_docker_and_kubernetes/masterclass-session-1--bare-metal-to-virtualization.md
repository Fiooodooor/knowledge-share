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
  table { font-size: 0.8em; }
  th { background: #16213e; color: #00d4ff; }
  td { background: #0f3460; }
  blockquote { border-left: 4px solid #7b68ee; background: #16213e; padding: 10px 20px; }
  a { color: #ffd93d; }
  .columns { display: flex; gap: 2em; }
  .col { flex: 1; }
---

<!-- _class: lead -->

# 🖥️ Session 1
## From Bare Metal to Virtualization
### Master Class — Infrastructure Foundations

**Duration:** 90 minutes
**Level:** Intermediate → Advanced

---

# 📋 Session 1 — Agenda

| # | Topic | Time |
|---|-------|------|
| 1 | Bare Metal OS Deployment | 15 min |
| 2 | What is Virtualization? | 10 min |
| 3 | The Hypervisor — Core Concepts | 20 min |
| 4 | Type 1 vs Type 2 Hypervisors | 15 min |
| 5 | Hypervisor Reference Model Deep Dive | 10 min |
| 6 | Virtualization vs Containerization — Preview | 10 min |
| 7 | Q&A / Discussion | 10 min |

---

<!-- _class: lead -->

# Part 1
## Bare Metal OS Deployment

---

# What is "Bare Metal"?

> **Bare metal** = software running directly on hardware **without** any intervening virtualization layer.

```
┌─────────────────────────────┐
│       Application(s)        │
├─────────────────────────────┤
│     Operating System        │
│   (Linux / Windows / BSD)   │
├─────────────────────────────┤
│   Hardware (CPU, RAM, NIC,  │
│     Storage, GPU, etc.)     │
└─────────────────────────────┘
```

- **1:1 relationship** — one OS owns the entire machine
- Full, unmediated hardware access
- Maximum performance, minimum abstraction

---

# Bare Metal — Characteristics

### ✅ Advantages
- **Maximum performance** — no virtualization overhead
- **Full hardware access** — DMA, IOMMU, SR-IOV, GPU passthrough native
- **Deterministic latency** — critical for HPC, real-time, HFT
- **Simpler debugging** — no hypervisor layer to reason about

### ❌ Disadvantages
- **Low utilization** — typical server uses only 5–15% of capacity
- **Slow provisioning** — hours to days (PXE boot, Kickstart, Preseed)
- **No isolation** — a rogue process can crash the entire machine
- **Scaling = buying hardware** — no elasticity

---

# Bare Metal Provisioning Methods

| Method | Description | Use Case |
|--------|-------------|----------|
| **PXE/iPXE + Kickstart** | Network boot → automated install | Data center fleet |
| **Cloud-Init** | First-boot config injection | Cloud bare metal (e.g., AWS i3.metal) |
| **Ironic (OpenStack)** | Bare metal as a service | Private cloud |
| **MAAS (Canonical)** | Metal as a Service | Ubuntu-centric DC |
| **Tinkerbell (Equinix)** | Declarative bare metal workflow | Edge / hybrid |
| **Manual ISO Install** | USB/DVD boot + manual steps | Lab / dev |

> **Key insight:** Bare metal provisioning is fundamentally slower than VM or container creation. This drove the industry toward virtualization.

---

# Linux on Bare Metal — Key Subsystems

```
User Space
├── systemd (PID 1, service management)
├── Applications & daemons
├── Shared libraries (glibc, libssl, ...)
│
Kernel Space
├── Process Scheduler (CFS / EEVDF)
├── Memory Management (page tables, NUMA, hugepages)
├── Virtual File System (VFS)
├── Network Stack (netfilter, tc, XDP, eBPF)
├── Device Drivers (NIC, storage, GPU)
├── Security Modules (SELinux, AppArmor, seccomp)
│
Hardware
├── CPU (rings 0-3, VMX extensions)
├── RAM (DDR4/5, NUMA nodes)
├── NIC (queues, RSS, offloads)
├── Storage (NVMe, SATA, HBA)
└── IOMMU, SR-IOV, PCIe topology
```

---

# The Problem Bare Metal Couldn't Solve

### Scenario: A company has 50 physical servers

```
Server 1:  Web App     → 8% CPU utilization
Server 2:  Database    → 12% CPU utilization
Server 3:  CI Runner   → 3% avg, 90% peak (bursts)
Server 4:  Mail Server → 5% CPU utilization
...
Server 50: Monitoring  → 2% CPU utilization
```

**Average utilization: ~8%** → 92% of purchased compute is **wasted**

### 💡 The question that launched an industry:

> *"Can we run multiple isolated workloads on one physical machine?"*

**Answer: Yes — Virtualization.**

---

<!-- _class: lead -->

# Part 2
## What is Virtualization?

---

# Virtualization — Definition

> **Virtualization** is the creation of a **virtual (rather than physical)** version of something — servers, storage, networks, or operating systems — using a software abstraction layer.

```
┌──────────┐ ┌──────────┐ ┌──────────┐
│   VM 1   │ │   VM 2   │ │   VM 3   │
│ (Ubuntu) │ │(Windows) │ │(FreeBSD) │
│ App A    │ │ App B    │ │ App C    │
├──────────┤ ├──────────┤ ├──────────┤
│ Guest OS │ │ Guest OS │ │ Guest OS │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │             │             │
┌────┴─────────────┴─────────────┴────┐
│         HYPERVISOR (VMM)            │
├─────────────────────────────────────┤
│         Physical Hardware           │
└─────────────────────────────────────┘
```

---

# A Brief History of Virtualization

| Year | Milestone |
|------|-----------|
| **1967** | IBM CP-40 — first hypervisor on System/360 Model 67 |
| **1972** | IBM VM/370 — commercial virtual machine OS |
| **1998** | VMware founded — x86 virtualization via binary translation |
| **1999** | VMware Workstation 1.0 released |
| **2003** | Xen hypervisor open-sourced (paravirtualization) |
| **2005–06** | Intel VT-x and AMD-V — hardware-assisted virtualization |
| **2007** | KVM merged into Linux kernel (2.6.20) |
| **2008** | Microsoft Hyper-V released |
| **2010s** | Cloud era — EC2, GCE, Azure all built on hypervisors |
| **2020s** | Lightweight VMMs: Firecracker (AWS Lambda), Cloud Hypervisor |

> **The x86 trap-and-emulate problem** (Popek & Goldberg, 1974) wasn't solved until VMware's binary translation (1999) and Intel VT-x (2005).

---

# Why Virtualization? — The Core Value

### Before Virtualization (Physical Servers)
- 1 app = 1 server
- 5–15% average CPU utilization
- Weeks to provision new servers
- Hardware lock-in

### After Virtualization
- **Many apps on 1 server** → 60–80% utilization
- **Minutes to create new VMs** → agility
- **Hardware abstraction** → portability
- **Snapshots & live migration** → disaster recovery
- **Isolation** → security boundaries between tenants

---

<!-- _class: lead -->

# Part 3
## The Hypervisor — Core Concepts

---

# Hypervisor — Definition

> A **hypervisor** (also called **Virtual Machine Monitor — VMM**) is software, firmware, or hardware that creates and runs virtual machines by **separating a computer's software from its hardware**.

### What it does:
1. **Partitions** physical resources (CPU, memory, I/O) among VMs
2. **Isolates** VMs from each other
3. **Emulates** or **paravirtualizes** hardware for guest OSes
4. **Schedules** VM execution on physical CPUs
5. **Intercepts** privileged instructions from guest kernels

### The Contract:
> Each VM **believes** it has exclusive access to dedicated hardware. The hypervisor maintains this **illusion** while sharing the real hardware.

---

# Types of Hypervisors — Overview

```
         TYPE 1 (Bare Metal)              TYPE 2 (Hosted)
     ┌────────┐  ┌────────┐         ┌────────┐  ┌────────┐
     │  VM 1  │  │  VM 2  │         │  VM 1  │  │  VM 2  │
     │Guest OS│  │Guest OS│         │Guest OS│  │Guest OS│
     └───┬────┘  └───┬────┘         └───┬────┘  └───┬────┘
         │           │                   │           │
  ┌──────┴───────────┴──────┐     ┌──────┴───────────┴──────┐
  │   TYPE 1 HYPERVISOR     │     │   TYPE 2 HYPERVISOR     │
  │   (runs ON hardware)    │     │   (runs ON host OS)     │
  ├─────────────────────────┤     ├─────────────────────────┤
  │     Physical Hardware   │     │     Host OS (Linux,     │
  └─────────────────────────┘     │     Windows, macOS)     │
                                  ├─────────────────────────┤
                                  │   Physical Hardware     │
                                  └─────────────────────────┘
```

---

# Type 1 — Bare Metal Hypervisor

> Runs **directly on hardware** — it **is** the operating system (or functionally replaces it).

### Examples:
| Hypervisor | Vendor | Notes |
|------------|--------|-------|
| **VMware ESXi** | Broadcom | Industry standard for enterprise |
| **Microsoft Hyper-V** | Microsoft | Built into Windows Server |
| **KVM** | Linux/Red Hat | Kernel module — Linux IS the hypervisor |
| **Xen** | Linux Foundation | Used by AWS EC2 (legacy instances) |
| **Proxmox VE** | Proxmox | KVM + LXC, open-source |
| **Firecracker** | AWS | MicroVM for Lambda/Fargate |
| **bhyve** | FreeBSD | Native FreeBSD hypervisor |

---

# Type 1 — Characteristics

### ✅ Pros
- **Near-native performance** — minimal overhead (1–5%)
- **Strong isolation** — thin attack surface
- **Hardware-assisted** — leverages VT-x/AMD-V, VT-d, SR-IOV
- **Scalable** — run hundreds of VMs per host
- **Live migration** — move VMs between hosts with zero downtime

### ❌ Cons
- **Complex to set up** — dedicated infrastructure
- **Requires compatible hardware** — VT-x/AMD-V, IOMMU
- **Management overhead** — needs vCenter, oVirt, Proxmox UI, etc.
- **Expensive licensing** (VMware vSphere, Microsoft Datacenter)

### 🎯 Use Cases
Enterprise data centers, cloud providers, production workloads, multi-tenant hosting

---

# Type 2 — Hosted Hypervisor

> Runs **as an application** on top of a conventional operating system.

### Examples:
| Hypervisor | Platform | Notes |
|------------|----------|-------|
| **Oracle VirtualBox** | Cross-platform | Free, open-source |
| **VMware Workstation** | Windows/Linux | Commercial, feature-rich |
| **VMware Fusion** | macOS | Workstation equivalent for Mac |
| **Parallels Desktop** | macOS | Best macOS integration |
| **QEMU** | Cross-platform | Emulator + virtualizer |
| **GNOME Boxes** | Linux | Simple QEMU frontend |

---

# Type 2 — Characteristics

### ✅ Pros
- **Easy to install** — just another application
- **No dedicated hardware** — runs on your laptop/desktop
- **Great host–guest integration** — shared folders, clipboard, drag & drop
- **Snapshots** — quick state save/restore
- **Perfect for development, testing, malware analysis**

### ❌ Cons
- **Performance degradation** — overhead from host OS layer
- **Security degradation** — host OS compromise → all VMs compromised
- **Resource contention** — competes with host OS and other apps
- **Not suitable for production** — no live migration, limited HA

### 🎯 Use Cases
Development, testing, learning, malware sandboxing, running legacy apps

---

# KVM — A Special Case (Type 1.5?)

```
┌──────────────────────────────────────────────┐
│              User Space (Linux)               │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ QEMU/VM1 │ │ QEMU/VM2 │ │ Normal Apps  │ │
│  └────┬─────┘ └────┬─────┘ └──────────────┘ │
│       │             │                         │
├───────┴─────────────┴─────────────────────────┤
│          Linux Kernel + KVM module            │
│  ┌─────────────────────────────────────────┐  │
│  │  KVM: /dev/kvm — hardware virt. API     │  │
│  │  QEMU: device emulation in userspace    │  │
│  │  virtio: paravirtualized I/O drivers    │  │
│  └─────────────────────────────────────────┘  │
├───────────────────────────────────────────────┤
│     Hardware (VT-x / AMD-V, VT-d, SR-IOV)   │
└───────────────────────────────────────────────┘
```

> **KVM turns Linux itself into a Type-1 hypervisor.** Each VM is a regular Linux process managed by QEMU. The kernel's scheduler, memory manager, and driver stack are reused — no separate hypervisor OS.

---

# Type 1 vs Type 2 — Comparison Matrix

| Aspect | Type 1 (Bare Metal) | Type 2 (Hosted) |
|--------|---------------------|------------------|
| **Runs on** | Hardware directly | Host operating system |
| **Performance** | Near-native (1–5% overhead) | Moderate (10–30% overhead) |
| **Security** | Strong isolation | Host compromise = game over |
| **Use case** | Production, cloud, DC | Dev, test, sandbox |
| **Boot time** | Seconds (microVMs) to minutes | Minutes (host + hypervisor + VM) |
| **Management** | vCenter, oVirt, Proxmox | GUI application |
| **Live migration** | ✅ Yes | ❌ No |
| **Cost** | $$$ (licenses + dedicated HW) | $ (free or cheap) |
| **Examples** | ESXi, KVM, Hyper-V, Xen | VirtualBox, VMware Workstation |

---

<!-- _class: lead -->

# Part 4
## Hypervisor Reference Model
### (Popek & Goldberg, 1974)

---

# The Three Pillars of a Hypervisor

> **Popek & Goldberg (1974)** defined the formal requirements for virtualizable architectures and the **three core modules** that coordinate to emulate hardware:

```
         ┌─────────────────────────────────────┐
         │         VIRTUAL MACHINE             │
         │     (guest OS + applications)       │
         └──┬───────────┬──────────────┬───────┘
            │           │              │
            ▼           ▼              ▼
   ┌────────────┐ ┌──────────┐ ┌─────────────┐
   │ DISPATCHER │ │ALLOCATOR │ │ INTERPRETER  │
   │            │ │          │ │              │
   │ Entry point│ │ Resource │ │  Privileged  │
   │ Routes     │ │ manager  │ │  instruction │
   │ traps to   │ │ for VMs  │ │  emulation   │
   │ handlers   │ │          │ │              │
   └────────────┘ └──────────┘ └─────────────┘
```

---

# Module 1: DISPATCHER

> The **entry point** of the VMM. All traps from guest VMs arrive here first.

### How it works:
1. Guest VM executes a **sensitive instruction** (privileged or behavior-sensitive)
2. Hardware **traps** to the hypervisor (via VT-x VMEXIT or ring transition)
3. Dispatcher receives the trap
4. Dispatcher **examines** the trap reason
5. Dispatcher **routes** to either the **Allocator** or the **Interpreter**

### Modern implementations:
- **KVM:** Linux kernel trap handler → `kvm_handle_exit()` 
- **ESXi:** VMM world trap handler
- **Xen:** Hypercall handler + trap dispatch

> Think of the Dispatcher as a **traffic controller** — it doesn't do the work, it decides **who** does.

---

# Module 2: ALLOCATOR

> Decides and manages **what physical resources** each VM gets.

### Responsibilities:
- **CPU scheduling** — which VM runs on which physical core, for how long
- **Memory allocation** — how much RAM each VM gets, shadow/nested page tables
- **I/O assignment** — virtual devices mapped to physical or emulated devices
- **Resource limits** — prevent one VM from starving others

### Triggered when:
A guest instruction **changes the machine's resource mapping** — e.g., setting up new page tables, accessing a new I/O port, changing interrupt vectors.

### Modern implementations:
- **KVM:** Linux CFS/EEVDF scheduler + KSM memory dedup + cgroups
- **ESXi:** DRS (Distributed Resource Scheduler) + memory ballooning
- **Xen:** Credit/Credit2 scheduler + Xen grant tables

---

# Module 3: INTERPRETER

> Contains **interpreter routines** that emulate privileged instructions.

### Responsibilities:
- Execute an **equivalent safe sequence** when a guest tries a privileged operation
- Emulate hardware behavior **without giving the guest real hardware access**
- Maintain the **virtual CPU state** (virtual registers, flags, control registers)

### Examples of interpreted instructions:
| Guest Instruction | What Interpreter Does |
|--|--|
| `LGDT` (load GDT) | Updates virtual GDT, maintains shadow GDT |
| `MOV CR3` (page table switch) | Updates shadow/nested page tables |
| `IN/OUT` (I/O port access) | Routes to virtual device emulator |
| `HLT` (halt CPU) | Deschedules vCPU, wakes on virtual interrupt |
| `INVLPG` (TLB invalidation) | Flushes relevant shadow TLB entries |

---

# Reference Model in Action — Full Flow

```
Guest VM: MOV to CR3 (switch page tables)
    │
    ▼ VMEXIT (hardware trap)
┌──────────┐
│DISPATCHER │──── Trap reason: CR3 write
└──┬───┬───┘
   │   │
   │   ▼ (resource change detected)
   │ ┌──────────┐
   │ │ALLOCATOR │──── Update VM's memory mapping
   │ └──┬───────┘     Track new page table base address
   │    │
   │    ▼ (privileged instruction)
   ▼ ┌───────────┐
     │INTERPRETER│──── Emulate CR3 load safely
     └──┬────────┘     Update nested/shadow page tables
        │              Flush relevant TLB entries
        ▼
    VMENTER (resume guest)
```

---

# Popek & Goldberg — The Formal Theorem

### Three types of instructions in an ISA:

1. **Privileged instructions** — cause a trap when executed in user mode
2. **Sensitive instructions:**
   - **Control-sensitive** — change system configuration (e.g., I/O, page tables)
   - **Behavior-sensitive** — behave differently depending on privilege level

### The Theorem (1974):

> *A virtual machine monitor may be constructed for any conventional third-generation computer if the set of **sensitive instructions** is a **subset** of the set of **privileged instructions**.*

### The x86 Problem:
x86 had **17 sensitive but non-privileged instructions** (e.g., `SGDT`, `SIDT`, `POPF`) — they didn't trap! Solutions:
- **Binary translation** (VMware, 1998) — rewrite guest code on-the-fly
- **Paravirtualization** (Xen, 2003) — modify guest OS to use hypercalls
- **Hardware VT-x/AMD-V** (2005–06) — new CPU mode with proper trapping

---

# Hardware-Assisted Virtualization (VT-x)

```
┌─────────────────────────────────────────┐
│              VMX Operation              │
│                                         │
│  ┌─────────┐         ┌───────────────┐ │
│  │VMX root │◄──VMEXIT──│VMX non-root│ │
│  │(host/   │          │  (guest VM)  │ │
│  │hyperv.) │──VMENTER──►│             │ │
│  └─────────┘         └───────────────┘ │
│                                         │
│  VMCS (Virtual Machine Control Struct.) │
│  ┌─────────────────────────────────────┐│
│  │ Guest state area (regs, CR, EFER)  ││
│  │ Host state area (return state)     ││
│  │ VM-execution controls (what traps) ││
│  │ Exit reason + qualification        ││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
```

- **VMLAUNCH/VMRESUME** → enter guest (VMX non-root)
- **VMEXIT** → trap back to host (VMX root) on sensitive operations
- **VMCS** → per-vCPU control structure (what to trap, guest/host state)

---

# Key Benefits of Hypervisors — Summary

### 🔧 Efficiency
Maximizes hardware utilization — run **multiple** virtual servers on **one** physical machine (60–80% utilization vs. 5–15% bare metal)

### 🔒 Isolation & Security
If one VM crashes or is compromised, others remain **unaffected** — strong security boundary (especially Type 1)

### 🔄 Flexibility
Run **different operating systems** (Linux, Windows, BSD) simultaneously on the same hardware

### 💰 Cost Savings
Reduces physical hardware count → lower **energy, cooling, space, and maintenance** costs

### ☁️ Cloud Foundation
Hypervisors are the **foundation** of modern cloud services — AWS (Xen → Nitro/Firecracker), Google Cloud (KVM), Azure (Hyper-V)

---

<!-- _class: lead -->

# Part 5
## Virtualization vs. Containerization
### (Preview for Session 2)

---

# The Two Paradigms

```
      VIRTUALIZATION                    CONTAINERIZATION

┌──────┐ ┌──────┐ ┌──────┐     ┌──────┐ ┌──────┐ ┌──────┐
│App A │ │App B │ │App C │     │App A │ │App B │ │App C │
├──────┤ ├──────┤ ├──────┤     ├──────┤ ├──────┤ ├──────┤
│Bins/ │ │Bins/ │ │Bins/ │     │Bins/ │ │Bins/ │ │Bins/ │
│Libs  │ │Libs  │ │Libs  │     │Libs  │ │Libs  │ │Libs  │
├──────┤ ├──────┤ ├──────┤     └──┬───┘ └──┬───┘ └──┬───┘
│GuestOS││GuestOS││GuestOS│        │        │        │
└──┬───┘ └──┬───┘ └──┬───┘   ┌────┴────────┴────────┴────┐
   │        │        │       │     Container Runtime      │
┌──┴────────┴────────┴────┐  │   (Docker/containerd/CRI-O)│
│       Hypervisor        │  ├────────────────────────────┤
├─────────────────────────┤  │    Host OS Kernel (shared) │
│   Hardware              │  ├────────────────────────────┤
└─────────────────────────┘  │    Hardware                │
                             └────────────────────────────┘
```

---

# Side-by-Side Comparison

| Aspect | Virtualization | Containerization |
|--------|---------------|-----------------|
| **Isolation** | Full OS per VM (strong) | Shared kernel (process-level) |
| **Resource Usage** | Heavy — each VM has full OS | Lightweight — shared kernel |
| **Performance** | 1–5% overhead (Type 1) | Near-native (<1% overhead) |
| **Startup Time** | Seconds to minutes | Milliseconds to seconds |
| **Image Size** | GBs (full OS image) | MBs (only app + deps) |
| **Portability** | Less portable (OS-specific) | Highly portable (OCI images) |
| **Density** | 10–100 VMs per host | 100–1000+ containers per host |
| **Security** | Strong (hardware isolation) | Weaker (kernel shared) |
| **Ecosystem** | VMware, Hyper-V, KVM | Docker, Kubernetes, Podman |
| **Best For** | Multi-OS, legacy, strong isolation | Microservices, CI/CD, cloud-native |

---

# When to Use Which?

### Choose **Virtualization** when:
- Running **different operating systems** on one host
- Need **strong security isolation** (multi-tenant, compliance)
- Running **legacy applications** not designed for containers
- Need **full kernel control** (custom kernel modules, drivers)
- Compliance requires **hardware-level separation**

### Choose **Containerization** when:
- Building **microservices architectures**
- Need **fast scaling** (autoscaling, burst capacity)
- Want **CI/CD pipeline integration** (build → test → deploy)
- Need **maximum resource density** (cost optimization)
- Building **cloud-native applications**

### 🔑 Real-world answer: **Both.** Containers typically run *inside* VMs in production.

---

# The Full Stack in Production

```
┌────────────────────────────────────────────────────┐
│              YOUR APPLICATIONS                     │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│   │Container │ │Container │ │Container │  ...      │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘          │
│        └──────┬──────┘            │                │
│        ┌──────┴───────────────────┴──────────┐     │
│        │   Kubernetes / Container Orchestrator │    │
│        ├─────────────────────────────────────┤     │
│        │   Container Runtime (containerd/CRI-O)│   │
│        ├─────────────────────────────────────┤     │
│        │   Linux Kernel (namespaces, cgroups) │    │
│        └─────────────────────────────────────┘     │
│                    ┌──────────┐                     │
│                    │    VM    │ ← VM per K8s node   │
│                    └────┬─────┘                     │
│               ┌─────────┴──────────┐                │
│               │   Hypervisor       │                │
│               │ (ESXi/KVM/Hyper-V) │                │
│               ├────────────────────┤                │
│               │ Physical Hardware  │                │
│               └────────────────────┘                │
└────────────────────────────────────────────────────┘
```

**Next session:** We'll deep-dive into what's inside that container & Kubernetes layer →

---

<!-- _class: lead -->

# 🧠 Session 1 — Key Takeaways

1. **Bare metal** gives maximum performance but wastes resources and is slow to provision
2. **Hypervisors** solve this by multiplexing hardware across isolated VMs
3. **Type 1** (bare metal) hypervisors are for production; **Type 2** (hosted) for dev/test
4. The **Dispatcher → Allocator → Interpreter** triad is the universal hypervisor model
5. **Hardware-assisted virtualization** (VT-x/AMD-V) solved x86's virtualization gap
6. **Containers ≠ replacement for VMs** — they complement each other
7. In production: apps in **containers**, containers in **VMs**, VMs on **hypervisors**

---

# 📖 Session 1 — Recommended Reading

- Popek & Goldberg, *"Formal Requirements for Virtualizable Third Generation Architectures"* (1974)
- Intel SDM Volume 3, Chapter 23-33 — VMX architecture
- Smith & Nair, *"Virtual Machines: Versatile Platforms for Systems and Processes"* (2005)
- [The Borg Paper (Google)](https://research.google/pubs/pub43438/) — large-scale cluster management
- [Firecracker: Lightweight Virtualization for Serverless](https://www.usenix.org/conference/nsdi20/presentation/agache)
- [KVM Architecture Overview](https://www.linux-kvm.org/page/Documents)

---

<!-- _class: lead -->

# ❓ Questions & Discussion
## (10 minutes)

### Discussion prompts:
1. *When would you choose bare metal over VMs?*
2. *Why did cloud providers build on Type 1 hypervisors instead of containers alone?*
3. *What's the security difference between VM isolation and container isolation?*

---

<!-- _class: lead -->

# See you in Session 2!
## From Containers to Kubernetes
### 🐳 → ☸️
