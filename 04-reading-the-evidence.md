# Reading the evidence

In containerd, OOMKilled comes from memory events (a TaskOOM event, or OOM metrics checked when the exit status is 137), while Completed and Error are just exit code 0 versus non-zero, and both the reason and Docker's OOMKilled flag have documented blind spots.

## containerd's reason strings

Everything in this chapter is scoped to containerd v2.3.5. `internal/cri/server/helpers.go` lines 53 to 59 define "Completed" (exit 0), "Error" (non-zero) and "OOMKilled". `container_status.go` lines 105 and 107 apply Completed or Error when no reason was set. `events.go` line 378 sets OOMKilled on a TaskOOM event, and line 194 re-checks OOM metrics when the exit status equals 137.

A comment in containerd's source (`events.go` lines 199 to 220) lists three caveats:

1. An OOM kill of an exec process or a child process can set OOMKilled even when the main process was not OOM killed.
2. The client may see Error first and OOMKilled later.
3. With the systemd cgroup driver, detection can be flaky or missing.

## Docker's flag

The Engine API `State.OOMKilled` (swagger.yaml at docker-v29.8.0, lines 6123 to 6126): "Whether a process within this container has been killed because it ran out of memory since the container was last started." Note "a process within". It is not about the main process alone.

## Kubernetes fields

ContainerStateTerminated has exitCode, reason, message, signal, startedAt and finishedAt. In `kubelet_pods.go` at v1.37.0 the conversion for an exited container (lines 2393 to 2401) copies ExitCode, Reason, Message, StartedAt, FinishedAt and ContainerID and does not set signal. In that file the word Signal appears only as StopSignal. The Kubernetes memory task page shows `exitCode: 137` together with `reason: OOMKilled`.

## Triage order

1. Read the reason, or `State.OOMKilled`.
2. Check the kernel log for the OOM lines (chapter 03).
3. Rebuild the stop timeline: who sent TERM, and when (chapter 05).
4. Check whether PID 1 handles SIGTERM (chapter 06).

Sources: github.com/containerd/containerd/blob/v2.3.5/internal/cri/server/events.go#L194, github.com/moby/moby/blob/docker-v29.8.0/api/swagger.yaml#L6123, kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/
