# Task 5: Scheduler Experiments and Analysis

Multi-Container Runtime — OS Mini Project (UE24CS242B)

## Overview

This project implements a lightweight multi-container runtime in C using Linux namespaces and system-level primitives. The system supports container execution, monitoring, and logging, and is extended with a kernel module for memory management.

While the project consists of multiple components (Tasks 1–4), this repository primarily focuses on **Task 5: Scheduler Experiments and Analysis**, which investigates the behavior of the Linux Completely Fair Scheduler (CFS).

---

## Project Outline

The project is divided into the following components:

### Task 1 — Basic Container Runtime

* Container creation using `clone()`
* Namespace isolation (PID, UTS, Mount)
* Filesystem isolation using `chroot()`

### Task 2 — CLI and Container Management

* Commands: `start`, `run`, `ps`, `logs`, `stop`
* Tracking and managing multiple containers

### Task 3 — Logging System

* Bounded-buffer logging pipeline
* Producer–consumer model using POSIX threads
* Per-container log files

### Task 4 — Kernel Module (Memory Monitoring)

* Tracks container memory usage
* Soft limit warnings
* Hard limit enforcement (process termination)

### Task 5 — Scheduler Experiments (Main Focus)

* Empirical analysis of Linux CFS
* CPU-bound vs CPU-bound scheduling
* CPU-bound vs I/O-bound scheduling

---

## Workloads

| File         | Type      | Behaviour                                           |
| ------------ | --------- | --------------------------------------------------- |
| `cpu_hog.c`  | CPU-bound | Tight loop, continuously uses CPU                   |
| `io_pulse.c` | I/O-bound | Performs write bursts with `fsync()` and `usleep()` |

---

## Background: Completely Fair Scheduler (CFS)

CFS schedules processes based on virtual runtime (vruntime), which represents CPU time consumed normalized by process weight.

Nice values are mapped to weights using the kernel’s scheduling table:

| Nice Value | Weight | CPU Share (2 processes) |
| ---------- | ------ | ----------------------- |
| -10        | 9548   | ~98.9%                  |
| 0          | 1024   | ~50%                    |
| +10        | 110    | ~1.1%                   |

CPU share formula:

```id="9l8c0m"
CPU share = weight / (sum of weights)
```

---

## Build Instructions

```bash id="6n4l3y"
make

# Or manually:
gcc -O2 -Wall -static -o cpu_hog cpu_hog.c
gcc -O2 -Wall -static -o io_pulse io_pulse.c
```

Copy binaries into container root filesystems:

```bash id="m8k2ps"
cp cpu_hog  ./rootfs-alpha/
cp cpu_hog  ./rootfs-beta/
cp io_pulse ./rootfs-gamma/
```

---

## Experiment A — CPU-bound vs CPU-bound (different priorities)

Two containers run `cpu_hog` concurrently:

```bash id="3y0d4k"
sudo ./engine start alpha ./rootfs-alpha /cpu_hog 20 --nice -10
sudo ./engine start beta  ./rootfs-beta  /cpu_hog 20 --nice 10

sudo ./engine logs alpha
sudo ./engine logs beta
```

### Results

| Container | Nice | Elapsed Time | CPU Share |
| --------- | ---- | ------------ | --------- |
| alpha     | -10  | ~19–20 s     | ~98.9%    |
| beta      | +10  | ~20 s        | ~1.1%     |

### Observation

The alpha container receives a significantly larger share of CPU time due to its higher weight. The beta container progresses more slowly due to limited CPU allocation.

On multi-core systems, both containers may complete in similar wall-clock time due to parallel execution. The scheduling difference becomes more visible under single-core contention.

---

## Experiment B — CPU-bound vs I/O-bound (equal priority)

```bash id="0zt9n3"
sudo ./engine start alpha ./rootfs-alpha /cpu_hog 20 --nice 0
sudo ./engine start gamma ./rootfs-gamma /io_pulse 20 --nice 0

sudo ./engine logs alpha
sudo ./engine logs gamma
```

### Results

| Container | Workload | Nice | Behaviour            |
| --------- | -------- | ---- | -------------------- |
| alpha     | cpu_hog  | 0    | Continuous CPU usage |
| gamma     | io_pulse | 0    | Periodic blocking    |

### Observation

The I/O-bound process frequently blocks and yields the CPU, allowing the CPU-bound process to utilize available CPU time. When the I/O-bound process wakes, it is scheduled promptly due to its lower virtual runtime, resulting in responsive execution.

---

## Analysis

### Weight-Based Scheduling

CFS distributes CPU time proportionally based on process weights derived from nice values.

* High-priority processes receive a larger share of CPU
* Low-priority processes still receive a guaranteed minimum share

### I/O vs CPU Behaviour

* CPU-bound processes utilize CPU continuously
* I/O-bound processes voluntarily yield CPU during blocking
* Wakeup scheduling ensures responsiveness

---

## CFS Properties Demonstrated

| Property              | Description                     |
| --------------------- | ------------------------------- |
| Proportional fairness | CPU allocation based on weights |
| Starvation freedom    | All processes receive CPU time  |
| Responsive scheduling | Fast wakeup for I/O-bound tasks |

---

## Conclusion

This project demonstrates that the Linux Completely Fair Scheduler achieves fairness through proportional CPU allocation while maintaining responsiveness for interactive workloads. The experiments validate theoretical scheduling behavior using practical container-based workloads.

---

## Repository

https://github.com/prathiknambiar/OS-Jackfruit
