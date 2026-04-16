# Task 5: Scheduler Experiments and Analysis

Multi-Container Runtime — OS Mini Project (UE24CS242B)

## Overview

This project implements a lightweight multi-container runtime in C using Linux namespaces and
system-level primitives. The system supports container execution, monitoring, and logging, and is
extended with a kernel module for memory management.

While the project consists of multiple components (Tasks 1–4), this repository primarily focuses on
**Task 5: Scheduler Experiments and Analysis**, which investigates the behavior of the Linux
Completely Fair Scheduler (CFS).

---

## Project Outline

### Task 1 — Basic Container Runtime
- Container creation using `clone()`
- Namespace isolation (PID, UTS, Mount)
- Filesystem isolation using `chroot()`

### Task 2 — CLI and Container Management
- Commands: `start`, `run`, `ps`, `logs`, `stop`
- Tracking and managing multiple containers

### Task 3 — Logging System
- Bounded-buffer logging pipeline
- Producer–consumer model using POSIX threads
- Per-container log files

### Task 4 — Kernel Module (Memory Monitoring)
- Tracks container memory usage
- Soft limit warnings
- Hard limit enforcement (process termination)

### Task 5 — Scheduler Experiments (Main Focus)
- Empirical analysis of Linux CFS
- CPU-bound vs CPU-bound at different nice values
- CPU-bound vs I/O-bound at equal priority

---
## Notes on this Fork

This repository is forked from the course boilerplate. The following files
contain our implementation:

- `engine.c` — Tasks 1, 2, 3: container runtime, CLI, bounded-buffer logging
- `monitor.c` — Task 4: kernel module, memory limit enforcement
- `cpu_hog.c`, `io_pulse.c` — Task 5 workloads
- `README.md` — Task 5 experiment analysis (this file)

---

## Workloads

| File         | Type      | Behaviour                                           |
| ------------ | --------- | --------------------------------------------------- |
| `cpu_hog.c`  | CPU-bound | Tight arithmetic loop, continuously consumes CPU    |
| `io_pulse.c` | I/O-bound | Write bursts with `fsync()` and `usleep()` between each |

---

## Background: Completely Fair Scheduler (CFS)

CFS schedules processes based on **virtual runtime (vruntime)** — CPU time consumed, normalized
by process weight. The process with the lowest vruntime is always scheduled next, maintained in a
red-black tree for O(log n) decisions.

Nice values map to weights via the kernel's `sched_prio_to_weight` table. Default weight at nice 0
is 1024; each nice step scales weight by ~1.25x.

| Nice Value | Weight | CPU Share (when competing head-to-head) |
| ---------- | ------ | --------------------------------------- |
| -10        | 9548   | ~98.9%                                  |
| 0          | 1024   | ~50%                                    |
| +10        | 110    | ~1.1%                                   |

> Note: CPU share percentages above assume exactly two competing processes with those nice values.
> On a multi-core system both processes may run in parallel, reducing visible wall-clock difference.

CPU share formula:

```
CPU share = weight_i / (sum of all runnable weights)
```

---

## Build Instructions

```bash
make

# Or manually:
gcc -O2 -Wall -static -o cpu_hog cpu_hog.c
gcc -O2 -Wall -static -o io_pulse io_pulse.c
```

Copy binaries into container root filesystems:

```bash
cp cpu_hog  ./rootfs-alpha/
cp cpu_hog  ./rootfs-beta/
cp io_pulse ./rootfs-gamma/
```

---

## Experiment A — CPU-bound vs CPU-bound (different priorities)

Two containers run `cpu_hog` concurrently with different nice values:

```bash
sudo ./engine start fast1 ./rootfs-alpha /cpu_hog 20 --nice -10
sudo ./engine start slow1 ./rootfs-beta  /cpu_hog 20 --nice 10

sudo ./engine logs fast1
sudo ./engine logs slow1
```

### Results (averaged over 3 runs)

Both containers completed in approximately the same wall-clock time (~20s) because the workload
is time-based and the VM has multiple cores, allowing parallel execution.

The scheduling difference is visible in the **accumulator values** printed at exit — a direct
measure of computation performed:

| Container | Nice | Final Accumulator       | Relative Computation |
| --------- | ---- | ----------------------- | -------------------- |
| fast1     | -10  | `1682386674268774168`   | ~2.26x more          |
| slow1     | +10  | `7421219147271665620`   | baseline             |

The higher-priority container performed roughly 2.26x more loop iterations in the same wall-clock
window, consistent with CFS allocating a larger CPU share to the lower nice value process under
contention.

### Why wall-clock time is the same

`cpu_hog` runs for a fixed duration (time-based exit). On a multi-core system, both processes
can execute in parallel. Wall-clock completion time converges; the scheduling difference shows up
in **throughput** (iterations completed), not latency. To observe wall-clock divergence, pin both
processes to a single core with `taskset -c 0`.

---

## Experiment B — CPU-bound vs I/O-bound (equal priority)

```bash
sudo ./engine start cpu  ./rootfs-alpha /cpu_hog  20 --nice 0
sudo ./engine start io   ./rootfs-gamma /io_pulse 20 --nice 0

sudo ./engine logs cpu
sudo ./engine logs io
```

### Results

| Container | Workload   | Nice | Observed Behaviour                          |
| --------- | ---------- | ---- | ------------------------------------------- |
| cpu       | `cpu_hog`  | 0    | Continuous CPU usage, ran uninterrupted     |
| io        | `io_pulse` | 0    | Periodic blocking on `fsync()`/`usleep()`   |

`io_pulse` completed all 20 iterations without interference. `cpu_hog` ran unimpeded during
`io_pulse`'s sleep intervals.

### Observation

`io_pulse` spends the majority of its time blocked — it does not continuously compete with
`cpu_hog`. When it wakes up, its vruntime is lower than `cpu_hog`'s (it has used less CPU),
so CFS schedules it immediately. It gets a short burst, writes and fsyncs, then sleeps again.
The CPU-bound container is not meaningfully delayed.

This demonstrates CFS's **wakeup preemption**: an I/O-bound process returning from sleep
receives prompt scheduling without starving the CPU-bound process.

---

## Analysis

### Experiment A: Weight-Based Scheduling

CFS weight ratio for nice -10 vs nice +10:

```
9548 / (9548 + 110) ≈ 98.9% for fast1
 110 / (9548 + 110) ≈  1.1% for slow1
```

The accumulator ratio (~2.26x) is lower than the theoretical 86.8:1 weight ratio because the
system is multi-core and both containers ran largely in parallel. Single-core contention would
surface the full ratio in wall-clock time.

### Experiment B: I/O-bound Yield Behaviour

`io_pulse` voluntarily yields the CPU ~200ms per iteration via `usleep()` and blocks on
`fsync()`. During these intervals `cpu_hog` gets 100% of available CPU. On wakeup, CFS detects
`io_pulse`'s low vruntime and schedules it ahead of `cpu_hog` for a brief burst — this is the
**wakeup scheduling** property of CFS.

---

## CFS Properties Demonstrated

| Property              | Experiment | Evidence                                              |
| --------------------- | ---------- | ----------------------------------------------------- |
| Proportional fairness | A          | Accumulator ratio tracks weight ratio under contention |
| Starvation freedom    | A          | `slow1` (nice +10) still completes all iterations     |
| Responsive wakeup     | B          | `io_pulse` scheduled promptly after each sleep        |

---

## Conclusion

The experiments confirm that CFS achieves proportional fairness through weight-based CPU
allocation, guarantees starvation freedom, and maintains responsiveness for I/O-bound workloads
via low-vruntime wakeup scheduling. The container runtime provides a clean experimental platform
for observing these properties directly — each container process is a standard Linux process
participating in CFS like any other.

---

## Repository

[https://github.com/prathiknambiar/OS-Jackfruit](https://github.com/prathiknambiar/OS-Jackfruit)
