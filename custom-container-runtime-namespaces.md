## 1. Project Title

Custom Container Runtime with Linux Namespaces

## 2. Difficulty

Mid-Level

### Rationale
This project dives into operating system virtualization and Linux kernel primitives. The developer must interact directly with low-level system calls (`clone`, `unshare`, `setns`) to isolate kernel namespaces (PID, mount, network, UTS, IPC, user) and set up cgroups for resource limitation. It demands a deep understanding of Linux process isolation, filesystem mounts (`pivot_root`), and low-level system programming.

## 3. Project Overview

The Custom Container Runtime is a lightweight command-line tool that isolates and runs arbitrary commands or root filesystems in a secure, sandboxed environment. Rather than relying on heavyweight orchestration tools like Docker, this project implements the core isolation primitives from scratch. It creates isolated process trees, mounts a minimal root filesystem, sets memory and CPU resource limits using cgroups v2, and executes isolated workloads safely.

## 4. Problem Statement

Modern containerization powers cloud infrastructure, yet tools like Docker conceal the underlying operating system mechanics behind friendly APIs. 
- Developers use containers daily without understanding how Linux kernel features like namespaces and cgroups achieve process isolation.
- Troubleshooting container escape vulnerabilities, resource starvation, or mount namespace issues requires an understanding of kernel-level virtualization.
- Building a minimal runtime unmasks the "magic" of containers, revealing that a container is simply a standard Linux process running under specific kernel constraints.

## 5. Proposed Solution

The proposed software is a CLI utility (e.g., `minicontainer run <rootfs> <cmd>`) that:
1. **Clones a Process**: Uses the `clone()` system call with isolation flags (`CLONE_NEWPID`, `CLONE_NEWNS`, `CLONE_NEWUTS`, `CLONE_NEWIPC`, `CLONE_NEWNET`) to spawn an isolated child process.
2. **Isolates Hostname & IPC**: Configures UTS and IPC namespaces so the container has its own hostname and IPC queues.
3. **Pivots Root Filesystem**: Uses `pivot_root` to switch the container's root filesystem to a minimal Linux distribution image (`rootfs`), making the host filesystem invisible.
4. **Applies Resource Limits**: Allocates cgroups v2 control groups to restrict CPU shares and maximum RAM usage.
5. **Executes Workload**: Runs the target executable within the isolated environment.

## 6. Project Goal

To build a functional Linux container runtime from scratch that successfully isolates processes, mounts a custom rootfs, restricts resource consumption, and executes commands without affecting the host operating system.

## 7. Core Workflow

```text
CLI Input: minicontainer run ───> Parse RootFS & Command ───> Invoke clone() with Flags
                                                                       │
                                      ┌────────────────────────────────┘
                                      ▼
                             [Child Process Context]
                                      │
                         1. Unshare / Namespace Isolation (PID, UTS, NS)
                         2. Mount procfs & pivot_root()
                         3. Apply cgroups v2 Limits
                                      │
                                      ▼
                            4. execve() Target Command
```

## 8. Functional Requirements

### Namespace Isolation
- **PID Namespace**: The containerized process must believe it is PID 1.
- **Mount Namespace**: Provide a private mount table so mount/unmount operations do not affect the host.
- **UTS Namespace**: Allow setting a unique container hostname.
- **Network / IPC Namespaces**: Isolate network interfaces and inter-process communication resources.

### Filesystem Sandboxing
- **RootFS Preparation**: Support mounting a provided directory (e.g., an Alpine Linux rootfs) as the container's root.
- **Pivot Root**: Safely switch root directories using `pivot_root` and unmount the old host root.
- **Virtual Filesystems**: Automatically mount `/proc` and `/sys` inside the container for inspection tools like `ps`.

### Resource Management (Cgroups v2)
- **Memory Limits**: Enforce a strict memory ceiling (e.g., max 100MB). If exceeded, the container process is terminated.
- **CPU Limits**: Restrict CPU weight or quota.

## 9. Non-Functional Requirements

- **Security**: The container must prevent privilege escalation and block unauthorized access to host filesystems.
- **Performance**: Container startup overhead must be negligible (under 50ms).

## 10. Main Entities / Data Model

### ContainerConfig
- **RootFSPath**: String.
- **Command**: Array of Strings (e.g., `["/bin/sh"]`).
- **MemoryLimitBytes**: Integer.
- **CpuQuota**: Integer.

## 11. System Components

- **CLI Parser**: Parses arguments and configuration.
- **Namespace Spawner**: Executes system calls (`clone`, `unshare`).
- **Filesystem Setup**: Handles `pivot_root` and mounting `/proc`.
- **Cgroups Manager**: Configures cgroups v2 control limits.

## 12. Important Technical Challenges

### The `pivot_root` Sequence
- **Challenge**: Switching root filesystems securely without leaving leaks or permission errors is notoriously finicky. The old root must be made a slave mount before pivoting, and unmounted correctly.
- **Concepts**: Mount propagation, `MS_PRIVATE`, `pivot_root`.

### PID 1 Responsibilities
- **Challenge**: In a new PID namespace, the container process becomes PID 1. PID 1 is responsible for reaping orphan zombie processes. If the container process spawns child processes and exits, zombie processes can accumulate if PID 1 doesn't handle signals correctly.
- **Concepts**: Process lifecycle, reaping zombies, signal handling.

## 13. Suggested Technology Areas

- **Language**: Go (with `syscall` package) or C/C++ (direct system calls).
- **Environment**: Linux operating system (since Linux namespaces and cgroups are Linux kernel features).

## 14. Skills and Knowledge Gained

### Operating Systems
- Low-level Linux kernel primitives (namespaces, cgroups v2).
- System calls (`clone`, `pivot_root`, `unshare`, `mount`).
- Process isolation and containerization architectures.

## 15. Recommended Development Phases

1. **Phase 1 - Process Cloning**: Write a basic program that uses `clone()` to spawn a child process in a new PID and UTS namespace.
2. **Phase 2 - Filesystem Root Isolation**: Prepare an Alpine rootfs folder and implement `pivot_root` and `/proc` mounting.
3. **Phase 3 - Cgroups Integration**: Configure cgroups v2 directories to limit memory usage for the spawned process.
4. **Phase 4 - CLI Tool Integration**: Combine all components into a clean CLI tool (`run <rootfs> <cmd>`).

## 16. Testing Requirements

- **Functional Testing**: Run `ps aux` inside the container and verify that only container processes are visible (PID 1 active). Verify that modifying files inside the container does not affect the host filesystem.
- **Resource Limits Testing**: Run a memory-bomb script inside the container and verify that the kernel terminates it once it hits the cgroups memory limit.

## 17. Security Considerations

- **Privileged Execution**: Container runtimes require root privileges to create namespaces and cgroups. Ensure input paths are strictly validated to prevent accidental host destruction.
- **Capabilities**: Drop unnecessary Linux capabilities (`CAP_SYS_ADMIN`, `CAP_NET_ADMIN`) inside the container.

## 18. Possible Extensions

- **Container Networking (veth pairs)**: Create virtual ethernet pairs (`veth`) to give the container its own IP address and internet access via NAT on the host.
- **Image Tarball Extraction**: Automatically download and extract OCI container image tarballs.

## 19. Learning Questions

- How do Linux namespaces differ from cgroups in terms of container isolation?
- Why is `pivot_root` preferred over `chroot` for container filesystem isolation?
- What happens to zombie processes in a container if PID 1 fails to reap them?

## 20. Completion Criteria

- [ ] CLI successfully spawns an isolated process using namespaces.
- [ ] RootFS is pivoted correctly, and host files are inaccessible.
- [ ] `/proc` is mounted correctly inside the container.
- [ ] Cgroups v2 successfully restricts memory usage.
- [ ] Container terminates cleanly without leaving orphaned host processes.