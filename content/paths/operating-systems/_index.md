---
title: "Operating Systems"
weight: 30
bookCollapseSection: true
---

# Operating Systems

Understanding operating systems is fundamental to becoming an effective software engineer. The OS is the layer between your applications and hardware — it manages processes, memory, files, and I/O devices, providing the abstractions that all software relies on.

This learning path covers OS internals from processes and scheduling through memory management, file systems, and concurrency, culminating in how modern container technology builds directly on OS primitives.

## What You'll Learn

| Section | Topic | Key Concepts |
|---------|-------|--------------|
| 01 | Processes & Threads | Process model, states, PCB, context switching, thread models |
| 02 | CPU Scheduling | FCFS, SJF, Round Robin, multilevel queues, Linux CFS |
| 03 | Memory Management | Paging, TLB, page faults, replacement algorithms, thrashing |
| 04 | Virtual Memory | Multi-level page tables, swap, COW, NUMA, huge pages |
| 05 | File Systems | Inodes, journaling, ext4/XFS/Btrfs, VFS, FUSE |
| 06 | I/O Systems | Interrupts, DMA, I/O scheduling, epoll/io_uring, SSD vs HDD |
| 07 | Concurrency Primitives | Mutexes, semaphores, deadlock, classic problems |
| 08 | Containers from the OS Perspective | Namespaces, cgroups, seccomp, overlay FS, runtimes |

## Prerequisites

- **Programming fundamentals** — comfortable reading C or pseudocode
- **Basic computer architecture** — CPU, RAM, disk, bus concepts
- **Command-line proficiency** — navigating a Linux/Unix terminal
- **Data structures** — linked lists, queues, trees, hash tables

## Recommended Reading

- *Operating System Concepts* (Silberschatz, Galvin, Gagne) — the classic "Dinosaur Book"
- *Operating Systems: Three Easy Pieces* (Arpaci-Dusseau) — free online, excellent explanations
- *Linux Kernel Development* (Robert Love) — for Linux-specific internals
- *The Linux Programming Interface* (Michael Kerrisk) — systems programming reference

## How to Use This Path

Each section builds on previous ones. Processes and threads are the foundation; scheduling decides which process runs; memory management gives each process its own address space; virtual memory extends that abstraction; file systems provide persistent storage; I/O systems connect to hardware; concurrency primitives coordinate parallel execution; and containers tie all these OS mechanisms together into modern deployment units.

Work through them in order. Each section includes diagrams, pseudocode, and real-world examples from Linux.
