# Who sent the SIGKILL

Several different actors produce 137, and in one path the kubelet writes it without observing any signal.

## Senders

- The kernel OOM killer, when a cgroup or the whole system runs out of memory.
- `docker stop`, after its timeout runs out (chapter 05).
- `docker kill`, whose default signal is "SIGKILL".
- The kubelet and runtime after a pod's grace period: "The container runtime sends SIGKILL to any processes still running".
- A person or script running `kill -9`.
- The kernel, when a PID namespace's init dies: it "terminates all of the processes in the namespace via a SIGKILL signal" (pid_namespaces(7)).

## No sender at all

Kubernetes v1.37.0 `pkg/kubelet/kubelet_pods.go`, lines 2418 to 2420: when a container that was Running can no longer be found, the kubelet writes a terminated state itself, with Reason ContainerStatusUnknown in the default path, Message "The container could not be located when the pod was terminated" and `ExitCode: 137, // this code indicates an error`. The reason constant is in `pkg/kubelet/container/runtime.go` line 308. A second site at lines 2656 to 2659 uses "The container could not be located when the pod was deleted.  The container used to be Running", again with 137.

In both cases the kubelet writes 137 itself, without observing any signal. The real cause of death, if there was one, is unknown, so the number is a placeholder.

## What the kernel logs

`mm/oom_kill.c` at Linux v7.2 builds the message from a prefix and a fixed format (lines 945 and 1169 to 1170). A cgroup limit kill logs a line starting:

    Memory cgroup out of memory: Killed process <pid> (<comm>) total-vm:...

A system wide kill logs:

    Out of memory: Killed process <pid> (<comm>) total-vm:...

If one of those lines names your process near the exit time, the OOM killer was the sender. If neither appears, look at the other senders above.

Sources: docs.docker.com/reference/cli/docker/container/kill/, kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/, man7.org `pid_namespaces.7`, github.com/kubernetes/kubernetes/blob/v1.37.0/pkg/kubelet/kubelet_pods.go#L2420, github.com/torvalds/linux/blob/v7.2/mm/oom_kill.c#L945
