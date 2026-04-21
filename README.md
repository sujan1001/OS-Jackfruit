# Multi-Container Runtime

A lightweight Linux container runtime in C with a long-running supervisor and a kernel-space memory monitor.

---

## 1. Team Information

| Name | SRN |
|------|-----|
| Sujan S M | PES1UG24AM292 |
| Shryes M | PES1UG24AM272 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 VM with Secure Boot OFF. No WSL.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build

```bash
cd boilerplate
make
```

This produces: `engine`, `memory_hog`, `cpu_hog`, `io_pulse`, and `monitor.ko`.

To verify only user-space compilation (CI-safe, no sudo/kernel needed):

```bash
make -C boilerplate ci
```

### Run Environment Preflight

```bash
cd boilerplate
chmod +x environment-check.sh
sudo ./environment-check.sh
```

### Prepare Root Filesystem

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# One writable copy per container
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta

# Copy workload binaries into rootfs
cp memory_hog cpu_hog io_pulse ./rootfs-alpha/
cp memory_hog cpu_hog io_pulse ./rootfs-beta/
```

Do not commit `rootfs-base/` or `rootfs-*` to the repository.

### Load the Kernel Module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor   # device should appear
dmesg | tail                   # should show: [container_monitor] Module loaded
```

### Start the Supervisor (Terminal 1)

```bash
sudo ./engine supervisor ./rootfs-base
```

The supervisor prints `Supervisor ready. Control socket: /tmp/mini_runtime.sock` and stays alive.

### Use the CLI (Terminal 2)

```bash
# Start containers in the background
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /memory_hog --soft-mib 32 --hard-mib 64

# List all tracked containers
sudo ./engine ps

# View a container's log output
sudo ./engine logs alpha

# Stop a container (SIGTERM then SIGKILL after 3 s)
sudo ./engine stop alpha

# Run a container and block until it exits
sudo ./engine run gamma ./rootfs-alpha "/cpu_hog 10"
```

### Test Memory Limits

```bash
# memory_hog allocates 8 MiB/s; with hard limit 64 MiB it gets killed after ~8 allocations
sudo ./engine start memtest ./rootfs-alpha /memory_hog --soft-mib 32 --hard-mib 64
sleep 12
dmesg | tail -20               # SOFT LIMIT and HARD LIMIT events
sudo ./engine ps               # state should show hard_limit_killed
```

### Scheduler Experiment

```bash
# Run two cpu_hog instances simultaneously with different nice values
cp -a ./rootfs-base ./rootfs-lo
cp -a ./rootfs-base ./rootfs-hi
cp cpu_hog ./rootfs-lo/ && cp cpu_hog ./rootfs-hi/

sudo ./engine start lo ./rootfs-lo "/cpu_hog 30" --nice 0
sudo ./engine start hi ./rootfs-hi "/cpu_hog 30" --nice 15

# Compare progress by watching logs
watch -n1 'sudo ./engine logs lo | tail -3 && echo "---" && sudo ./engine logs hi | tail -3'
```

### Unload and Clean Up

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
# Ctrl+C the supervisor (or: kill $(pgrep -f "engine supervisor"))
dmesg | tail
sudo rmmod monitor
make clean
```

---
## 3. Demo Screenshots

### SS1. Supervisor startup + multi-container launch
<img width="559" height="322" alt="image" src="https://github.com/user-attachments/assets/2ea41eaf-4497-421a-b439-796c784bcf30" />

Supervisor starts and then tracks multiple containers concurrently.

### SS2. Metadata tracking (`engine ps`)
<img width="1201" height="347" alt="image" src="https://github.com/user-attachments/assets/b7f92e9b-ee84-4b30-a42f-f753bf0c3586" />

Container ID, host PID, state, and limits are visible in one listing.

### SS3. Logging pipeline
<img width="566" height="713" alt="image" src="https://github.com/user-attachments/assets/b9ddd04b-4206-44a2-a8e7-fddc87b52f46" />

Logs captured from container stdout/stderr are persisted and retrievable.

### SS4. CLI to supervisor IPC
<img width="576" height="727" alt="image" src="https://github.com/user-attachments/assets/1ab2b44f-a2cc-4204-9b3e-ccc1d4fe7a06" />

CLI client commands are issued from a separate terminal, and the supervisor returns a structured response over the Unix domain socket control channel.

### SS5. Soft-limit warning event
<img width="568" height="721" alt="image" src="https://github.com/user-attachments/assets/0dbd8313-cfb8-403d-a1cb-2a989a8cc161" />

`dmesg` records first threshold crossing at soft memory limit.

### SS6. Hard-limit enforcement
<img width="572" height="725" alt="image" src="https://github.com/user-attachments/assets/d4021ba0-734b-484f-8bc0-616d78ff901a" />

Container state shows kill outcome after exceeding configured limits.

### SS7. Scheduling experiment evidence
<img width="570" height="329" alt="image" src="https://github.com/user-attachments/assets/fba370b9-8fb4-4416-89e7-0c768e230b37" />

Different `nice` values produce visible CPU-share differences.

### SS8. Clean teardown
<img width="572" height="725" alt="image" src="https://github.com/user-attachments/assets/38302654-7daf-4bdb-bcb1-1a2b420aced9" />

No leftover supervisor/container zombies after shutdown.

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Each container is created with `clone(CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS | SIGCHLD)`. The PID namespace gives the container its own PID 1 so container processes cannot see or signal host processes. The UTS namespace allows each container to have its own hostname (set to the container ID via `sethostname`). The mount namespace isolates the filesystem view — changes to mounts inside the container do not propagate to the host.

After `clone()`, the child calls `chroot(rootfs)` to restrict its filesystem root to its assigned Alpine directory, then mounts `/proc` inside so tools like `ps` work correctly. `chroot` is simpler than `pivot_root` and sufficient for this project since the rootfs directories are not accessible to other containers anyway.

The host kernel still shares the network stack, time, and the user/group database with all containers. PID 1 in each container's namespace maps to a real host PID, so the supervisor can track and signal it directly.

### 4.2 Supervisor and Process Lifecycle

The supervisor is a long-running process that listens on a UNIX domain socket. It stays alive for the lifetime of all containers so it can: accept CLI commands, maintain container metadata, reap exited children, and own the logging pipeline. Without a persistent parent, child processes would become orphans and their exit status would be lost.

Container creation uses `clone()` rather than `fork()` because `clone()` lets us pass namespace flags to isolate the child's PID, UTS, and mount views in one call. The supervisor records the host PID returned by `clone()` so it can signal, wait for, and unregister each container from the kernel monitor.

`SIGCHLD` is handled with `SA_RESTART | SA_NOCLDSTOP`. The handler sets a flag; the actual `waitpid(-1, WNOHANG)` loop runs in the event loop to avoid async-signal-safety issues. This prevents zombie accumulation regardless of whether a container exits normally, is stopped, or is killed by the memory monitor.

Termination is classified as:
- `exited` — process called `exit()` normally
- `stopped` — supervisor sent SIGTERM via `engine stop` (`stop_requested` flag was set)
- `hard_limit_killed` — received SIGKILL and `stop_requested` was not set (kernel module enforcement)
- `killed` — received another signal without a stop request

### 4.3 IPC, Threads, and Synchronization

The project uses two separate IPC mechanisms:

**Path A — Logging (pipes):** Each container's stdout and stderr are redirected to the write end of a pipe. One producer thread per container reads from the read end and pushes `log_item_t` structs into a shared `bounded_buffer_t`. One consumer thread (the logger) pops from the buffer and appends to per-container log files.

The bounded buffer is protected by a `pthread_mutex_t` with two `pthread_cond_t` variables (`not_empty`, `not_full`). Without this, a producer and consumer could simultaneously read/write `head`, `tail`, and `count`, corrupting the buffer state. A condition variable is the right primitive here because both producer and consumer need to sleep while waiting — a spinlock would waste CPU.

The `shutting_down` flag allows clean drain: producers that find the buffer full during shutdown drop the item; the consumer exits only when the buffer is both empty and `shutting_down == 1`.

**Path B — Control (UNIX domain socket):** The CLI client connects, writes one `control_request_t`, reads one `control_response_t`, and closes. The supervisor accepts connections in `select()` with a 1-second timeout so it can also check the shutdown flag. The socket is distinct from the pipes so that control messages never mix with log data.

The `containers` linked list is protected by `metadata_lock`. Any path that reads or writes container state (CLI handlers, SIGCHLD reaper, monitor unregister) takes this lock. The lock is a `pthread_mutex_t` because list traversal may involve memory allocation and is not performance-critical.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the number of physical pages currently mapped into a process's address space. It does not include pages that have been swapped out, shared library pages counted multiple times, or pages allocated but never touched. It is an imperfect but practical proxy for actual memory pressure.

Soft and hard limits represent two different policies:
- **Soft limit:** advisory — log a warning when RSS first crosses it, but allow the container to continue. This gives operators early visibility before a container becomes a problem.
- **Hard limit:** enforcement — send SIGKILL when RSS crosses it. The process is terminated immediately because at this point continued growth threatens system stability.

The enforcement belongs in kernel space rather than user space because: a user-space polling loop cannot guarantee timely enforcement (it could be scheduled out), a misbehaving process could interfere with user-space monitoring, and kernel code runs with higher privilege and can directly access the `mm_struct` RSS count and send signals atomically.

The kernel module uses a mutex with `mutex_trylock()` in the timer callback. The callback runs in softirq context (timer BH) which cannot sleep, so we use `trylock` and skip the tick if the lock is held by an ioctl call. This is safe because a 1-second check interval is much coarser than any locking delay.

### 4.5 Scheduling Behavior

The Linux Completely Fair Scheduler (CFS) allocates CPU time proportional to each process's weight, which is derived from its `nice` value. A `nice` value of 0 (default) gives weight 1024; `nice 15` gives weight ~82 — roughly 12× less. In a two-container experiment where both run CPU-bound loops, CFS gives the lower-nice container proportionally more time slices. This shows up as a faster elapsed time and higher progress count in the logs.

From Experiment 1 (see Section 6), `lo` (nice 0) completed all 30 seconds of work in approximately **31 s wall-clock** while `hi` (nice 15) required approximately **48 s wall-clock** — a ratio of ~1.55×. The theoretical maximum from pure CFS weights is 12.5×, but the single VM had 2 vCPUs; when one container is scheduled out on one core the other can still run on the second, compressing the observed ratio toward 1. Kernel overhead and context-switch cost further reduce the gap.

I/O-bound workloads (`io_pulse`) spend most of their time blocked in `fsync()`. The scheduler marks them as interactive and gives them a short latency boost when they wake up, but since they are blocked most of the time they consume little CPU regardless of nice value. From Experiment 2, `cpu_hog` consumed **≈ 97% CPU** while `io_pulse` consumed **< 2% CPU** at the same nice value, confirming that the I/O workload's wall time (~12 s for 60 iterations at 200 ms sleep) was dictated entirely by its sleep interval, not by CPU contention.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` via `clone()`, with `chroot()` for filesystem isolation.
**Tradeoff:** `pivot_root` would be more secure (prevents `..` traversal out of the rootfs) but requires the rootfs to be a mount point and adds setup complexity.
**Justification:** `chroot` is sufficient when each container has its own rootfs copy and the host has no untrusted users.

### Supervisor Architecture
**Choice:** Single-process supervisor with a `select()` event loop and a 1-second timeout.
**Tradeoff:** `select()` does not scale to thousands of file descriptors and the 1-second timeout adds latency for signal processing.
**Justification:** The project manages at most a handful of containers. `select()` is simpler to reason about than `epoll()` and keeps the event loop easy to audit.

### IPC / Logging
**Choice:** UNIX domain socket for control (Path B); pipes per container for logging (Path A); single shared bounded buffer with one consumer thread.
**Tradeoff:** A single consumer thread is a bottleneck if many containers produce output simultaneously. Per-container threads would be faster but harder to shut down cleanly.
**Justification:** Logging latency is not critical; correctness and clean shutdown are. One consumer thread with a condition-variable-gated buffer is straightforward to reason about and join on shutdown.

### Kernel Monitor
**Choice:** `mutex` for list protection; `mutex_trylock()` in timer callback; 1-second check interval.
**Tradeoff:** `trylock` means a tick is skipped if the lock is contended. A process could transiently exceed its hard limit for up to 1 extra second.
**Justification:** Memory limits are coarse-grained; a 1-second window is acceptable. A spinlock would be unsafe in ioctl context (which can sleep on `copy_from_user`).

### Scheduling Experiments
**Choice:** Compare two `cpu_hog` containers with `nice 0` vs `nice 15`; compare `cpu_hog` vs `io_pulse` at the same priority.
**Tradeoff:** Results are noisy without pinning processes to specific CPUs.
**Justification:** The experiments demonstrate the scheduler's weight-based fairness and I/O-vs-CPU scheduling behavior clearly without requiring cgroup or CPU affinity setup.

---

## 6. Scheduler Experiment Results

### Experiment 1: CPU-bound containers with different nice values

| Container | nice value | Duration (s) | Iterations completed |
|-----------|-----------|--------------|---------------------|
| lo        | 0         | 31           | 30                  |
| hi        | 15        | 48           | 30                  |

**Setup:** Both containers run `/cpu_hog 30` simultaneously. `lo` has `--nice 0`, `hi` has `--nice 15`.

**Expected:** `lo` finishes noticeably faster or completes more loop iterations in the same wall-clock window because CFS assigns it ~12× more weight.

**Observation:** `lo` finished in 31 s wall-clock; `hi` finished in 48 s wall-clock — a 1.55× ratio. The theoretical CFS weight ratio is 1024/82 ≈ 12.5×, but the 2-vCPU VM allows the lower-priority container to run in parallel most of the time, compressing the observed difference. The gap is still clear and consistent with nice-value scheduling.

### Experiment 2: CPU-bound vs I/O-bound at same priority

| Container | workload   | CPU% observed | wall time |
|-----------|-----------|---------------|-----------|
| cpu       | cpu_hog   | 97%           | 31 s      |
| io        | io_pulse  | < 2%          | 12 s      |

**Setup:** Both containers run simultaneously at `nice 0`. `cpu` runs `/cpu_hog 30`, `io` runs `/io_pulse 60 200`.

**Expected:** `cpu` dominates CPU usage. `io` finishes within expected wall time because its blocking time is spent in `fsync()`, not competing for CPU.

**Observation:** `cpu_hog` held the CPU at ~97% for the full 30 s. `io_pulse` ran 60 iterations in ~12 s (60 × 200 ms), consuming negligible CPU. Its wall time was determined entirely by the 200 ms sleep between `fsync()` calls, not by scheduling competition with the CPU-bound container.

### Analysis

CFS uses a red-black tree of virtual runtimes to ensure no runnable process is starved. Lower nice value → lower virtual time increment per real tick → more frequent scheduling. Higher nice value → larger increment → less frequent. This matches the OS scheduling goal of proportional fairness.

I/O-bound processes benefit from the CFS "sleeper fairness" mechanism: a process that was blocked gets a virtual runtime boost when it wakes so it can run sooner. This keeps interactive workloads responsive even under CPU pressure.
