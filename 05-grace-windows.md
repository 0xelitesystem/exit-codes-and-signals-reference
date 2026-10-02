# Grace windows

Every graceful stop is a TERM, a timer, then a KILL, so a slow or deaf process turns an intended 143 or 0 into 137.

## Docker

`docker stop`: "The main process inside the container will receive SIGTERM, and after a grace period, SIGKILL." The default "is 10 seconds for Linux containers, and 30 seconds for Windows containers". `--stop-timeout` sets it per container, and with -1 "the daemon waits indefinitely". The STOPSIGNAL instruction or `--stop-signal` changes the first signal.

## Kubernetes

From the pod lifecycle page, section Termination of Pods:

- "The default terminationGracePeriodSeconds setting is 30 seconds."
- If a preStop hook exists and the grace period is not 0, the kubelet runs the hook first. If it is still running at expiry, "the kubelet requests a small, one-off grace period extension of 2 seconds".
- Then "The kubelet triggers the container runtime to send a TERM signal to process 1 inside each container", and containers receive it "at different times and in an arbitrary order".
- Sidecars get TERM after the last main container has fully terminated, in reverse order.
- At expiry, the runtime sends SIGKILL to anything still running.

With no stop signal in the image, the runtime default is "SIGTERM for both containerd and CRI-O". The `lifecycle.stopSignal` field behind the ContainerStopSignals gate is, as of 2026-10-01, "Alpha since Kubernetes v1.33; disabled by default". A grace period of zero "indicates stop immediately via the kill signal". A probe can carry its own terminationGracePeriodSeconds, which overrides the pod value on probe failure.

## Outcomes

| What the app does with TERM | Status |
|---|---|
| Handles it and exits 0 | 0 |
| Takes the default action (only possible when it is not PID 1, e.g. behind docker run --init) | 143 |
| Is PID 1 with no handler | 137 (see chapter 06) |
| Ignores it, or runs past the grace period | 137 |

A 137 after a stop therefore often means "too slow" or "not listening", not "out of memory".

Sources: docs.docker.com/reference/cli/docker/container/stop/, docs.docker.com/reference/cli/docker/container/run/, kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/, kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/
