# Task 5: Scheduler Experiments and Analysis

Multi-Container Runtime — OS Mini Project (UE24CS242B)

---

## Overview

This repository contains the Task 5 contribution to the Multi-Container Runtime project: Scheduler Experiments and Analysis.

The runtime launches container processes using Linux `clone()` with isolated PID, UTS, and mount namespaces. From the kernel's perspective, each container is treated as a normal Linux process and is scheduled by the Completely Fair Scheduler (CFS).

This task demonstrates CFS behaviour through controlled experiments using CPU-bound and I/O-bound workloads.

---

## Workloads

| File         | Type      | Behaviour                                           |
| ------------ | --------- | --------------------------------------------------- |
| `cpu_hog.c`  | CPU-bound | Tight loop, continuously uses CPU                   |
| `io_pulse.c` | I/O-bound | Performs write bursts with `fsync()` and `usleep()` |

---

## Background: Completely Fair Scheduler (CFS)

CFS schedules processes based on virtual runtime (vruntime), which represents CPU time used, normalized by process weight.

Nice values are mapped to weights using the kernel’s `sched_prio_to_weight` table:

| Nice Value | Weight | CPU Share (2 processes) |
| ---------- | ------ | ----------------------- |
| -10        | 9548   | ~98.9%                  |
| 0          | 1024   | ~50%                    |
| +10        | 110    | ~1.1%                   |

CPU share formula:

```
CPU share = weight / (sum of weights)
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

Two containers run `cpu_hog` for 20 seconds concurrently:

```bash
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

The alpha container (nice -10) receives a significantly larger share of CPU time due to its higher weight. The beta container progresses much more slowly because it is allocated a very small portion of CPU time.

On multi-core systems, both containers may complete in similar wall-clock time due to parallel execution. The difference becomes more evident under single-core contention.

---

## Experiment B — CPU-bound vs I/O-bound (equal priority)

Run one CPU-bound and one I/O-bound container:

```bash
sudo ./engine start alpha ./rootfs-alpha /cpu_hog 20 --nice 0
sudo ./engine start gamma ./rootfs-gamma /io_pulse 20 --nice 0

sudo ./engine logs alpha
sudo ./engine logs gamma
```

### Results

| Container | Workload | Nice | Behaviour                       |
| --------- | -------- | ---- | ------------------------------- |
| alpha     | cpu_hog  | 0    | Continuous CPU usage            |
| gamma     | io_pulse | 0    | Periodic blocking (I/O + sleep) |

### Observation

The I/O-bound process frequently blocks and is removed from the run queue, allowing the CPU-bound process to use most of the CPU.

When the I/O process wakes up, it is scheduled quickly due to its lower vruntime, executes briefly, and then blocks again. This results in high responsiveness without significantly impacting the CPU-bound process.

---

## Analysis

### Experiment A: Weight-Based Scheduling

CFS distributes CPU time proportionally based on weights derived from nice values.

* alpha (nice -10) receives approximately 98.9% CPU
* beta (nice +10) receives approximately 1.1% CPU

This demonstrates proportional fairness under CPU contention.

---

### Experiment B: I/O vs CPU Behaviour

* I/O-bound processes yield CPU voluntarily during blocking
* CPU-bound processes utilize available CPU time continuously
* Upon wakeup, I/O-bound processes are scheduled quickly due to low vruntime

---

### CFS Properties Demonstrated

| Property              | Demonstration                            |
| --------------------- | ---------------------------------------- |
| Proportional fairness | CPU share based on weights               |
| Starvation freedom    | Low-priority processes still receive CPU |
| Responsive wakeup     | I/O-bound processes scheduled quickly    |

---

## Conclusion

These experiments confirm that the Linux Completely Fair Scheduler distributes CPU time proportionally based on process weights while maintaining responsiveness for I/O-bound workloads. CPU-bound processes compete strictly based on priority, while I/O-bound processes benefit from frequent blocking and fast wakeup scheduling.
