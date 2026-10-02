# The exit status byte

An exit status is one byte, 0 to 255, and 0 alone means success.

## Only 8 bits survive

POSIX `exit()` says that only "status & 0377" shall be "available from wait() and waitpid()". The POSIX shell chapter (2.8.2) adds that for C programs the value "is equal to the result of performing a modulo 256 operation". So C `exit(256)` is read as 0 and `exit(300)` as 44.

Linux `_exit(2)` puts it plainly: "The value status & 0xFF is returned to the parent process as the process's exit status", collected with the wait(2) family. POSIX.1-2024 also says the full value shall be available from `waitid()`. Do not count on that on Linux, where the man page above describes only the low byte.

## The shell builtin is different

`exit 256` in a script is not guaranteed to give 0. The POSIX `exit` special built-in says that for n of 256, or above 256 and not a signal status, "the wait status of the shell or subshell is unspecified". Bash truncates: `builtins/common.c` at bash-5.3, line 555, reads `status = sval & 255;`. So `bash -c 'exit 300'; echo $?` prints 44. That is Bash behaviour, not a POSIX promise.

## Reserved values

- 127: "If the command is not found, the exit status shall be 127." (POSIX 2.8.2)
- 126: "if the command name is found, but it is not an executable utility, the exit status shall be 126."
- 128: "An unrecoverable read error was detected by the shell while reading commands" (POSIX `exit`, application usage).
- Above 128: "A command was interrupted by a signal."
- 2: Bash builtins use it for incorrect usage (Bash manual 3.7.5).

## Both sides: docker run reuses the numbers

`docker run` gives 125, 126 and 127 its own meanings. 125 means "the error is with Docker daemon itself", 126 that "the specified contained command can't be invoked", 127 that "the contained command can't be found". Any other code is the container command's own.

Sources: pubs.opengroup.org/onlinepubs/9799919799 (functions/exit.html, utilities/V3_chap02.html), man7.org `_exit.2`, Bash manual, docs.docker.com/engine/containers/run/.
