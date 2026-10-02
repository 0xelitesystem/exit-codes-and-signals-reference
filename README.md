# Exit Codes and Signals

What Linux exit codes 126, 127, 137, 139 and 143 mean, which signal each 128+N code encodes, and why the number alone cannot tell an OOM kill from a stop timeout, a kill -9, or a code the kubelet wrote itself.

A plain-language reference for anyone staring at `exited with code 137` in Docker or Kubernetes. It reads the number the way the shell, the container runtime and the kubelet each produce it, quotes the primary source for every claim, and gives a short order of checks for finding out what actually happened. Scope: Linux on x86 and ARM.

## The core idea

Exit code 137 means only that the process ended with status 128 + 9, which is how Bash and container runtimes such as runc report death by SIGKILL, and it does not record who sent the signal. The kernel OOM killer is one sender. A stop timeout is another: Docker and Kubernetes both send TERM, wait, then send KILL. Kubernetes carries a separate reason field, set by the container runtime, and the kubelet can even write 137 itself for a container it can no longer find.

## What is inside

- [01-the-exit-status-byte.md](01-the-exit-status-byte.md) why only 0 to 255 survives, what 126, 127 and 128 are reserved for, and how docker run reuses them.
- [02-128-plus-n-and-the-signal-table.md](02-128-plus-n-and-the-signal-table.md) the 128 + N convention, where it comes from, and a signal table that holds on every Linux architecture.
- [03-who-sent-the-sigkill.md](03-who-sent-the-sigkill.md) every actor that produces 137, the path where the kubelet writes it without observing a signal, and the kernel's OOM log lines.
- [04-reading-the-evidence.md](04-reading-the-evidence.md) the reason field, Docker's OOMKilled flag, their known blind spots, and a triage order.
- [05-grace-windows.md](05-grace-windows.md) the TERM, timer, KILL sequence in Docker and Kubernetes, with the defaults.
- [06-the-pid-1-problem.md](06-the-pid-1-problem.md) why PID 1 drops SIGTERM without a handler, and the three fixes.

## How to use this

- A container exited 137 and someone says "memory". Run the triage order in chapter 04 before raising the limit.
- A deploy keeps ending pods with 137 instead of 0. Read chapters 05 and 06: the app is probably not handling SIGTERM, or it is not PID 1.
- A script returns 141 or 130 and you want the signal. Subtract 128 and look it up in chapter 02.
- `exit 256` in a script did not fail. Chapter 01 explains the byte.

## The stance

The exit code is a byte. Read it as a signal number, never as a diagnosis. Before blaming any one layer, put each layer's reading side by side: the exit code, the runtime's reason, the kernel log and the stop timeline.

## Sources

Every quote and line number was checked on 2026-10-01 against these pinned sources.

- POSIX.1-2024 Shell and Utilities: https://pubs.opengroup.org/onlinepubs/9799919799/utilities/V3_chap02.html
- POSIX.1-2024 exit(): https://pubs.opengroup.org/onlinepubs/9799919799/functions/exit.html
- Linux man pages: https://man7.org/linux/man-pages/man2/_exit.2.html, https://man7.org/linux/man-pages/man7/signal.7.html, https://man7.org/linux/man-pages/man7/pid_namespaces.7.html
- Bash Reference Manual, Edition 5.3: https://tiswww.case.edu/php/chet/bash/bashref.html
- Bash 5.3 source: https://git.savannah.gnu.org/cgit/bash.git/plain/builtins/common.c?h=bash-5.3 and https://git.savannah.gnu.org/cgit/bash.git/plain/builtins/evalstring.c?h=bash-5.3 (line numbers read from the bash-5.3 release tarball, https://ftp.gnu.org/gnu/bash/bash-5.3.tar.gz)
- Docker docs: https://docs.docker.com/reference/cli/docker/container/stop/, https://docs.docker.com/reference/cli/docker/container/kill/, https://docs.docker.com/reference/cli/docker/container/run/, https://docs.docker.com/engine/containers/run/, https://docs.docker.com/reference/dockerfile/
- Kubernetes docs: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/, https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/, https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/
- runc v1.5.1: https://github.com/opencontainers/runc/blob/v1.5.1/libcontainer/utils/utils.go#L13
- containerd v2.3.5: https://github.com/containerd/containerd/blob/v2.3.5/internal/cri/server/events.go#L194, https://github.com/containerd/containerd/blob/v2.3.5/internal/cri/server/helpers.go#L53, https://github.com/containerd/containerd/blob/v2.3.5/internal/cri/server/container_status.go#L105
- Kubernetes v1.37.0: https://github.com/kubernetes/kubernetes/blob/v1.37.0/pkg/kubelet/kubelet_pods.go#L2420, https://github.com/kubernetes/kubernetes/blob/v1.37.0/pkg/kubelet/container/runtime.go#L308
- Linux v7.2: https://github.com/torvalds/linux/blob/v7.2/mm/oom_kill.c#L945
- Moby docker-v29.8.0: https://github.com/moby/moby/blob/docker-v29.8.0/api/swagger.yaml#L6123

This is an independent project. It is not affiliated with or endorsed by Docker, the Kubernetes project, containerd, runc, the GNU Project or The Open Group. Product names are used only to identify the software being described.

## License

MIT. Copyright 0xelitesystem 2026.
