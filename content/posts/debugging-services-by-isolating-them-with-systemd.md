+++
date = '2025-08-25T12:01:33+01:00'
draft = false
title = 'Debugging Services by Isolating Them With Systemd'
+++

# Debugging Services by Isolating Them with systemd

A few days ago, I was trying to track down a bug in a service that was causing it to misbehave and occasionally crash. Despite having good test coverage and attaching a debugger, I was struggling to reproduce the issue reliably. After some thought, it occurred to me to try running the service in a container with systemd and progressively removing privileges to see if I could trigger the bug.

This post won't cover the Podman specifics, as the Red Hat developer blog has an excellent [article on running systemd in a container](https://developers.redhat.com/blog/2019/04/24/how-to-run-systemd-in-a-container) that I used as a starting point. Instead, I'll focus on the systemd unit file options that I found most useful for isolating and debugging services.

I'll quickly cover the options that I think require clarification, but the example unit file below has links to all the docs if you want to dig into what each option does in more detail. :)

Oh, and if you are wondering what the issue was, it turned out there were some I/O operations that were semi-order-dependent, but because they were wrapped in a guard clause, that code path would more often than not be skipped.

In the end, what I'm showing here didn't pinpoint the issue, but it got me thinking about what the real issue could be. If nothing else, it was a fun side quest where I got to learn a bit more about systemd unit files and what they can do. :)

## systemd Unit File Options for Debugging

Here are some of the systemd options I used to isolate the service:

-   `StartLimitIntervalSec` and `StartLimitBurst`: These two options work together to prevent a service from endlessly restarting. If the service stops or crashes repeatedly, it will stop being restarted after the number of restarts is greater than the `StartLimitBurst` value within the `StartLimitIntervalSec` period.

-   `NoNewPrivileges`: This option blocks the service process from gaining new privileges. The catch here is that it has no effect if another process (like `crontab`, `systemd-run`, or any random IPC) sends it a request that could cause the service to fork a new process.

-   `LockPersonality`: The systemd documentation for this says it "locks down the personality(2) system call," which is very clear... if you know what personalities are. I didn't. From what I can gather, it seems to be related to an address space feature where you can give a process a "persona" with a certain set of permissions. This option locks that persona so the process can't switch to a new one, which might have elevated privileges. I mostly included this as a "why not?" option, so I'm not entirely sure how it works in practice. I may dig into that at some point and write something up then. :)

-   `SystemCallFilter`: This is probably the most interesting option of the lot because it lets you either allow or deny access to a set of syscalls for the binary launched by the unit. This is done by using a kernel feature called `seccomp`. `seccomp` transitions a process into what it calls a "secure state" where the process can only use the `read`, `write`, `sigreturn`, and `exit` syscalls. If the process tries to make any other syscalls, it will either log the call or be killed, depending on how it's configured. `SystemCallFilter` uses three symbols to denote whether a syscall should be allowed, blocked, or to target a syscall group:

    -   `~`: Denies the specified syscall.
    -   `+`: Excludes a syscall from any filtering.
    -   `@`: Targets one of the predefined syscall groups instead of a single syscall. You can use both `~` and `+` with `@`. The list of groups is quite long, so you can find the full list in the docs linked in the example.

## Example systemd Unit File

Here is an example of a systemd unit file that uses these options:

```bash
# Manpage references:
# - systemd.unit (where StartLimit* belongs):
#   https://www.freedesktop.org/software/systemd/man/systemd.unit.html
# - systemd.exec (sandbox options, SystemCallFilter, SystemCallErrorNumber, NoNewPrivileges):
#   https://www.freedesktop.org/software/systemd/man/systemd.exec.html
# - systemd.resource-control (MemoryLimit/MemoryMax/MemoryHigh/TasksMax):
#   https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html
# - systemd-analyze (syscall-filter helper to list/inspect syscalls):
#   https://www.freedesktop.org/software/systemd/man/systemd-analyze.html
#
# Notes:
# - StartLimitIntervalSec and StartLimitBurst belong in [Unit] (see systemd.unit).
# - SystemCallFilter & SystemCallErrorNumber require systemd built with seccomp support;
#   test with: sudo systemd-analyze syscall-filter @known
# - If using cgroup v2, MemoryMax is preferable to MemoryLimit; keep MemoryLimit for
#   broader compatibility (see systemd.resource-control).

[Unit]
# See: systemd.unit (StartLimit* placement)
# https://www.freedesktop.org/software/systemd/man/systemd.unit.html
StartLimitIntervalSec=90s
StartLimitBurst=4

[Service]
# Basic behaviour & existing Exec lines are provided by packaged unit.
# Only sandbox/resource keys are set here to avoid editing the packaged file.

# On cgroup v2 you may prefer MemoryMax (see systemd.resource-control).
# https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html
MemoryLimit=100M

# See systemd.exec "NoNewPrivileges"
# https://www.freedesktop.org/software/systemd/man/systemd.exec.html
NoNewPrivileges=yes

# Filesystem / kernel surface reduction (see systemd.exec)
PrivateTmp=yes
PrivateDevices=yes
ProtectHome=yes
ProtectSystem=full
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes

# Prevent process from changing personality
# See systemd.exec "LockPersonality"
# https://www.freedesktop.org/software/systemd/man/systemd.exec.html
LockPersonality=yes

# Keep network namespace commented by default. Enable only if you want no network:
#PrivateNetwork=yes

# Keep KillMode as supplied by you (process). If you want systemd to kill whole
# cgroup on stop/failure, use KillMode=control-group (default behaviour).
#KillMode=process

# Syscall filtering (seccomp) — block common inspection syscalls and return EPERM.
# Confirm these names on your host with:
#   sudo systemd-analyze syscall-filter process_vm_readv
#   sudo systemd-analyze syscall-filter process_vm_writev
#   sudo systemd-analyze syscall-filter bpf
#   sudo systemd-analyze syscall-filter perf_event_open
#   sudo systemd-analyze syscall-filter ptrace
# See systemd.exec "SystemCallFilter" and "SystemCallErrorNumber":
# https://www.freedesktop.org/software/systemd/man/systemd.exec.html
#
# If your systemd supports syscall sets and you prefer sets, replace with e.g.
#   SystemCallFilter=@ipc ~ptrace ~perf_event_open
# Otherwise the explicit names below are valid on systems whose syscall list
# includes these entries (verify with systemd-analyze above).
SystemCallFilter=~ptrace ~process_vm_readv ~process_vm_writev
SystemCallFilter=~perf_event_open ~bpf
SystemCallErrorNumber=EPERM

# Task limit: keep "infinity" per original (see systemd.resource-control)
# https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html
TasksMax=infinity

SystemCallErrorNumber=EPERM

# Task limit: keep "infinity" per original (see systemd.resource-control)
# https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html
TasksMax=infinity
```

I'm always open to a good chat! If anything I've written resonates with you, if you have questions, or if you're interested in collaborating on something cool, please don't hesitate to reach out. I generally prefer asynchronous communication, as it helps me manage my focus and give thoughtful responses.

You can find me here:

*   [**Email**](mailto:me@gozes.dev)
*   [**Mastodon**](https://hachyderm.io/@gozes)
*   [**Bluesky**](https://bsky.app/profile/gozes.dev)
*   [**LinkedIn**](https://www.linkedin.com/in/juan-alberto-s-93490925)

Looking forward to connecting with you!
