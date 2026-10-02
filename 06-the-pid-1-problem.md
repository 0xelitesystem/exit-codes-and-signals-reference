# The PID 1 problem

PID 1 in a container ignores any signal it has no handler for, except SIGKILL and SIGSTOP from outside, so an app with no SIGTERM handler rides out docker stop and dies at the timeout with 137.

## The kernel rule

pid_namespaces(7): a process in an ancestor namespace can "send signals to the "init" process of a child PID namespace only if the "init" process has established a handler for that signal", and "SIGKILL or SIGSTOP are treated exceptionally: these signals are forcibly delivered when sent from an ancestor PID namespace."

## Docker's warning, and its limit

The Dockerfile reference says shell form runs under `/bin/sh -c`, "which does not pass signals", so the executable "will not be the container's PID 1" and "doesn't receive a SIGTERM from docker stop".

That is not universal. When `/bin/sh` is Bash, Bash replaces itself with the last simple command of a `-c` string. See `should_optimize_fork` (line 106) and `should_suppress_fork` (line 125) in `builtins/evalstring.c` at bash-5.3. The optimisation needs no traps, no redirections and a simple command. `bash -c 'cat /proc/$$/cmdline'` prints cat's own command line, because cat took over the shell's PID. Other shells were not checked here.

So do not guess. Inside the container, run:

    cat /proc/1/cmdline

## Being PID 1 is not enough

Even when your app is PID 1, the kernel rule above still applies. With no SIGTERM handler, TERM is dropped, the timer runs out, and SIGKILL arrives.

## Fixes

- Handle SIGTERM in the app and exit cleanly.
- Use exec form, or start shell form with `exec` (the Dockerfile reference: "remember to start it with exec").
- `docker run --init`: "Run an init inside the container that forwards signals and reaps processes". The bundled docker-init "is backed by tini".

Sources: man7.org/linux/man-pages/man7/pid_namespaces.7.html, docs.docker.com/reference/dockerfile/, docs.docker.com/reference/cli/docker/container/run/, git.savannah.gnu.org/cgit/bash.git/plain/builtins/evalstring.c?h=bash-5.3
