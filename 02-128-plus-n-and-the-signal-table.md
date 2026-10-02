# 128 plus N, and the signal table

A status above 128 usually means death by signal N, where N is the status minus 128, by a convention Bash and runc share and POSIX does not fix.

## Who uses 128 + N

Bash manual 3.7.5: "When a command terminates on a fatal signal whose number is N, Bash uses the value 128+N as the exit status." POSIX only says the shell "shall assign it an exit status greater than 128. The exit status shall identify, in an implementation-defined manner, which signal terminated the command", and shells "are permitted to assign an exit status greater than 255".

In a container no shell does this. runc v1.5.1 `libcontainer/utils/utils.go` sets `exitSignalOffset = 128` (line 13) and `ExitStatus` returns `exitSignalOffset + int(status.Signal())` when the process was signaled (line 20).

## The table

Scope: Linux on x86 and ARM. Every signal below has the same number in all four signal(7) columns.

| Signal | N | Status | Default |
|---|---|---|---|
| SIGHUP | 1 | 129 | Term |
| SIGINT | 2 | 130 | Term |
| SIGQUIT | 3 | 131 | Core |
| SIGILL | 4 | 132 | Core |
| SIGTRAP | 5 | 133 | Core |
| SIGABRT | 6 | 134 | Core |
| SIGFPE | 8 | 136 | Core |
| SIGKILL | 9 | 137 | Term |
| SIGSEGV | 11 | 139 | Core |
| SIGPIPE | 13 | 141 | Term |
| SIGALRM | 14 | 142 | Term |
| SIGTERM | 15 | 143 | Term |

Footnote: SIGBUS is 7 on x86/ARM (135) but 10 on Alpha/SPARC, MIPS and PARISC. SIGUSR1 is 10 on x86/ARM (138) but 30 on Alpha/SPARC and 16 on MIPS and PARISC. signal(7): "The signals SIGKILL and SIGSTOP cannot be caught, blocked, or ignored."

## The number is ambiguous

A program that calls `exit(137)` itself produces the same number. runc returns `status.ExitStatus()` for a normal exit (line 22), so the byte alone cannot separate the two.

## SIGPIPE in pipelines

With `set -o pipefail`, Bash returns the status of the rightmost command that failed. In `yes | head -1`, `head` exits 0 and `yes` keeps writing to a closed pipe, dies of SIGPIPE, and the pipeline reports 141.

Sources: Bash manual, POSIX V3_chap02, man7.org `signal.7`, github.com/opencontainers/runc/blob/v1.5.1/libcontainer/utils/utils.go#L13
