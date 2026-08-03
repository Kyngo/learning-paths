---
title: "Containers from the OS Perspective"
weight: 8
---

# Containers from the OS Perspective

Containers are not a single technology — they are an orchestration of several Linux kernel features that together provide process isolation. Understanding these primitives demystifies Docker and reveals what containers actually are: **isolated groups of processes sharing the host kernel**.

---

## Linux Namespaces

Namespaces partition kernel resources so that one set of processes sees one set of resources, and another set sees a different set.

| Namespace | Isolates | Flag | Effect |
|-----------|----------|------|--------|
| **PID** | Process IDs | `CLONE_NEWPID` | Process sees itself as PID 1 |
| **NET** | Network stack | `CLONE_NEWNET` | Own interfaces, IPs, routing tables |
| **MNT** | Mount points | `CLONE_NEWNS` | Own filesystem tree |
| **UTS** | Hostname/domain | `CLONE_NEWUTS` | Own hostname |
| **IPC** | IPC resources | `CLONE_NEWIPC` | Own message queues, semaphores |
| **USER** | UID/GID mappings | `CLONE_NEWUSER` | Root inside, unprivileged outside |
| **CGROUP** | Cgroup hierarchy | `CLONE_NEWCGROUP` | Own view of cgroup tree |
| **TIME** | System clocks | `CLONE_NEWTIME` | Own boot/monotonic time (Linux 5.6+) |

### Creating Namespaces

```bash
# Create a new PID + MNT + NET namespace and run bash inside
sudo unshare --pid --mount --net --fork bash

# Inside: PID 1, no network interfaces, isolated mounts
ps aux        # only see processes in this namespace
ip link       # only loopback
hostname      # still host's (need --uts to isolate)
```

### PID Namespace in Action

```
Host view:                Container view:
PID 1 (systemd)          PID 1 (nginx)      ← mapped to PID 4521 on host
PID 4521 (nginx)         PID 2 (worker)
PID 4522 (worker)
```

### User Namespaces — Rootless Containers

```bash
# Map container root (UID 0) to host UID 100000
unshare --user --map-root-user bash
id    # uid=0(root) — but only inside the namespace
```

This enables rootless containers: full "root" power inside, zero privilege on the host.

---

## Control Groups (cgroups)

Cgroups **limit, account for, and isolate** resource usage of process groups.

### cgroups v1 vs v2

| Feature | v1 | v2 |
|---------|----|----|
| Hierarchy | Multiple hierarchies (one per controller) | Single unified hierarchy |
| Controllers | Mounted independently | All under one tree |
| Delegation | Complex, error-prone | Clean delegation to unprivileged users |
| PSI (Pressure Stall Info) | Not available | Built-in |
| Adoption | Legacy (Docker historically) | Default in modern kernels (5.8+) |

### cgroups v2 Resource Control

```bash
# Create a cgroup
mkdir /sys/fs/cgroup/mycontainer

# Set memory limit to 256MB
echo 268435456 > /sys/fs/cgroup/mycontainer/memory.max

# Set CPU limit to 50% of one core
echo "50000 100000" > /sys/fs/cgroup/mycontainer/cpu.max

# Add a process
echo $PID > /sys/fs/cgroup/mycontainer/cgroup.procs
```

### Common Controllers

| Controller | Limits | Example |
|------------|--------|---------|
| `memory` | RAM + swap usage | `memory.max = 512M` |
| `cpu` | CPU time share | `cpu.max = 50000 100000` (50%) |
| `io` | Disk I/O bandwidth | `io.max = 8:0 rbps=1048576` |
| `pids` | Number of processes | `pids.max = 100` |
| `cpuset` | CPU core pinning | `cpuset.cpus = 0-3` |

---

## Linux Capabilities

Traditional Unix: either root (all power) or unprivileged (no power). Capabilities split root privileges into granular units:

| Capability | Grants |
|------------|--------|
| `CAP_NET_BIND_SERVICE` | Bind to ports < 1024 |
| `CAP_NET_RAW` | Use raw sockets (ping) |
| `CAP_SYS_ADMIN` | Mount filesystems, set hostname |
| `CAP_DAC_OVERRIDE` | Bypass file permission checks |
| `CAP_CHOWN` | Change file ownership |
| `CAP_SETUID` | Change process UID |

```bash
# Docker drops most capabilities by default
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# View capabilities of a process
cat /proc/$PID/status | grep Cap
```

Containers run with a **minimal capability set** — typically ~14 of 40+ available.

---

## Seccomp (Secure Computing)

Seccomp filters restrict which **syscalls** a process can make:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    { "names": ["read", "write", "exit", "brk"], "action": "SCMP_ACT_ALLOW" }
  ]
}
```

Docker's default seccomp profile blocks ~44 dangerous syscalls including:
- `reboot`, `kexec_load` — system destruction
- `mount`, `umount2` — filesystem manipulation
- `ptrace` — process debugging/injection
- `keyctl` — kernel keyring access

---

## Overlay Filesystems

Containers use **overlay filesystems** to layer a writable layer on top of read-only image layers:

```
┌─────────────────────────────┐
│  Container Layer (writable) │  ← upperdir
├─────────────────────────────┤
│  App Layer (read-only)      │  ← lowerdir
├─────────────────────────────┤
│  Runtime Layer (read-only)  │  ← lowerdir
├─────────────────────────────┤
│  Base OS Layer (read-only)  │  ← lowerdir
└─────────────────────────────┘
         merged view ← what the container sees
```

```bash
mount -t overlay overlay \
  -o lowerdir=/base:/runtime:/app,upperdir=/container,workdir=/work \
  /merged
```

### Copy-on-Write (CoW)

- **Read:** File served from lowest layer that has it
- **Write:** File copied up to writable layer, then modified
- **Delete:** Whiteout file created in upper layer

This enables sharing base layers across hundreds of containers with minimal disk usage.

---

## Container Runtimes

### Architecture Stack

```
┌──────────────────────────────────┐
│  Docker CLI / Podman / nerdctl   │  User interface
├──────────────────────────────────┤
│  Docker Engine / Podman daemon   │  Container management
├──────────────────────────────────┤
│  containerd / CRI-O             │  Container lifecycle (OCI runtime)
├──────────────────────────────────┤
│  runc / crun / gVisor / kata    │  Low-level runtime (spawns container)
├──────────────────────────────────┤
│  Linux Kernel                    │  namespaces, cgroups, seccomp, etc.
└──────────────────────────────────┘
```

| Runtime | Role | Details |
|---------|------|---------|
| **runc** | OCI reference runtime | Spawns the container process using kernel primitives |
| **containerd** | Daemon managing container lifecycle | Image pull, storage, networking, supervises runc |
| **CRI-O** | Kubernetes-native runtime | Lighter alternative to containerd for k8s |
| **gVisor** | User-space kernel | Intercepts syscalls for extra isolation |
| **Kata Containers** | MicroVM runtime | Each container in a lightweight VM |

### What `docker run` Actually Does

```
1. Docker CLI → sends request to Docker daemon
2. Daemon → instructs containerd to create container
3. containerd → pulls image layers, sets up overlay FS
4. containerd → calls runc with OCI bundle (config.json + rootfs)
5. runc → clone() with CLONE_NEWPID|CLONE_NEWNS|CLONE_NEWNET|...
6. runc → sets up cgroups, capabilities, seccomp
7. runc → pivot_root to container rootfs
8. runc → exec() the container entrypoint
9. runc → exits (containerd supervises the running container)
```

---

## VMs vs Containers

| Aspect | Virtual Machine | Container |
|--------|----------------|-----------|
| Isolation level | Hardware (hypervisor) | OS (kernel namespaces) |
| Kernel | Guest OS has own kernel | Shares host kernel |
| Boot time | Seconds to minutes | Milliseconds |
| Image size | GBs (full OS) | MBs (just app + deps) |
| Overhead | 5-15% CPU, GBs RAM | < 1% CPU, MBs RAM |
| Security boundary | Strong (separate kernel) | Weaker (shared kernel) |
| Density | 10s per host | 100s-1000s per host |
| Use case | Multi-tenant, different OSes | Microservices, CI/CD |

### When to Choose What

| Requirement | Best Choice |
|-------------|-------------|
| Maximum isolation (multi-tenant) | VM or Kata Containers |
| Fast startup, high density | Containers |
| Different OS from host | VM |
| Same OS, application isolation | Containers |
| Regulatory compliance requiring HW separation | VM |

---

## Putting It All Together

A running container is simply a Linux process with:

1. **Namespaces** — isolated view of PIDs, network, filesystem, users
2. **Cgroups** — resource limits (CPU, memory, I/O)
3. **Capabilities** — reduced root privileges
4. **Seccomp** — restricted syscalls
5. **Overlay FS** — layered, copy-on-write filesystem
6. **Pivot root** — new root filesystem

```bash
# The "manual container" — no Docker needed:
unshare --pid --net --mount --uts --ipc --fork \
  --mount-proc chroot /my-rootfs /bin/sh
```

---

## Key Takeaways

1. Containers are **not VMs** — they are isolated processes sharing the host kernel.
2. **Namespaces** provide isolation (what a process can see); **cgroups** provide limits (what it can use).
3. **User namespaces** enable rootless containers — root inside, unprivileged outside.
4. **Overlay FS** and copy-on-write make image layers efficient and shareable.
5. The container runtime stack is layered: CLI → daemon → containerd → runc → kernel.
6. Security is defense-in-depth: namespaces + cgroups + capabilities + seccomp together.
7. VMs provide stronger isolation at higher cost; containers provide density and speed at reduced isolation.
