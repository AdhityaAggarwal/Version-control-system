MODULE 5: PID 1 Reaper
Files: reaper.cpp, signals.cpp

What it does: The reaper loop that acts as PID 1 inside the container's namespace.

Sample code:

// ================================================================
// KILL ALL PROCESSES IN NAMESPACE
// ================================================================
static bool kill_all_processes_except_self() noexcept {
    pid_t self = getpid();

    for (int attempt = 0; attempt < 5; ++attempt) {
        DIR* dir = opendir("/proc");
        if (!dir) {
            if (attempt == 4) {
                reaper_log("kill_all_processes_except_self: opendir(/proc) failed\n");
                return false;
            }
            struct timespec ts = {0, 10000000};
            nanosleep(&ts, nullptr);
            continue;
        }

        struct dirent* entry;
        while ((entry = readdir(dir)) != nullptr) {
            if (entry->d_name[0] < '0' || entry->d_name[0] > '9') continue;
            char* endptr;
            errno = 0;
            long pid = strtol(entry->d_name, &endptr, 10);
            if (errno != 0 || endptr == entry->d_name || *endptr != '\0') continue;
            if (pid <= 1 || pid == self) continue;
            
            if (kill((pid_t)pid, SIGKILL) == -1) {
                if (errno != ESRCH) {
                    // Intentionally minimal
                }
            }
        }
        closedir(dir);
        struct timespec ts = {0, 10000000};
        nanosleep(&ts, nullptr);
    }

    struct timespec ts = {0, 50000000};
    nanosleep(&ts, nullptr);
    DIR* dir = opendir("/proc");
    if (!dir) return true;
    struct dirent* entry;
    while ((entry = readdir(dir)) != nullptr) {
        if (entry->d_name[0] < '0' || entry->d_name[0] > '9') continue;
        char* endptr;
        errno = 0;
        long pid = strtol(entry->d_name, &endptr, 10);
        if (errno != 0 || endptr == entry->d_name || *endptr != '\0') continue;
        if (pid <= 1 || pid == self) continue;
        
        if (kill((pid_t)pid, SIGKILL) == -1) {
            if (errno != ESRCH) {
                // Intentionally minimal
            }
        }
    }
    closedir(dir);
    return true;
}

// ================================================================
// ROBUST ELAPSED TIME CALCULATION
// ================================================================
static long calculate_elapsed_ms(const struct timespec& start, const struct timespec& current) noexcept {
    if (current.tv_nsec >= start.tv_nsec) {
        return (static_cast<long>(current.tv_sec) - static_cast<long>(start.tv_sec)) * 1000L +
               (static_cast<long>(current.tv_nsec) - static_cast<long>(start.tv_nsec)) / 1000000L;
    } else {
        return (static_cast<long>(current.tv_sec) - static_cast<long>(start.tv_sec) - 1L) * 1000L +
               (static_cast<long>(current.tv_nsec) + 1000000000L - static_cast<long>(start.tv_nsec)) / 1000000L;
    }
}

// ================================================================
// RUN_REAPER
// ================================================================
static int run_reaper(pid_t child_pid) noexcept {
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGCHLD);
    sigaddset(&mask, SIGTERM);
    sigaddset(&mask, SIGINT);
    sigaddset(&mask, SIGQUIT);
    sigaddset(&mask, SIGHUP);
    sigaddset(&mask, SIGUSR1);
    sigaddset(&mask, SIGUSR2);

    if (sigprocmask(SIG_BLOCK, &mask, nullptr) == -1) {
        reaper_log("run_reaper: sigprocmask failed\n");
        return 1;
    }

    int sfd = signalfd(-1, &mask, SFD_CLOEXEC);
    if (sfd == -1) {
        int saved_errno = errno;
        char msg[256];
        safe_snprintf(msg, sizeof(msg), "run_reaper: signalfd failed: %s\n",
                      strerror(saved_errno));
        reaper_log(msg);
        return 1;
    }

    bool child_died = false;
    int child_exit_code = 0;
    bool sigterm_sent = false;
    struct timespec start_time = {};

    while (true) {
        while (true) {
            int status;
            pid_t dead_pid = waitpid(-1, &status, WNOHANG);
            if (dead_pid <= 0) break;

            if (dead_pid == child_pid) {
                child_died = true;
                if (WIFEXITED(status)) child_exit_code = WEXITSTATUS(status);
                else if (WIFSIGNALED(status)) child_exit_code = 128 + WTERMSIG(status);
                else child_exit_code = 0;
            }
        }

        if (child_died) {
            kill_all_processes_except_self();
            while (true) {
                int status;
                pid_t p = waitpid(-1, &status, WNOHANG);
                if (p <= 0) break;
            }
            close(sfd);
            return child_exit_code;
        }

        struct pollfd pfd;
        pfd.fd = sfd;
        pfd.events = POLLIN;

        int timeout_ms = -1;
        if (sigterm_sent) {
            struct timespec current_time;
            if (clock_gettime(CLOCK_MONOTONIC, &current_time) != 0) {
                reaper_log("run_reaper: clock_gettime failed, force-killing container\n");
                kill(child_pid, SIGKILL);
                kill(-child_pid, SIGKILL);
                kill_all_processes_except_self();
                while (true) {
                    int status;
                    pid_t p = waitpid(-1, &status, WNOHANG);
                    if (p <= 0) break;
                    if (p == child_pid) {
                        if (WIFEXITED(status)) child_exit_code = WEXITSTATUS(status);
                        else if (WIFSIGNALED(status)) child_exit_code = 128 + WTERMSIG(status);
                    }
                }
                close(sfd);
                return child_exit_code;
            }
            
            long elapsed_ms = calculate_elapsed_ms(start_time, current_time);
            if (elapsed_ms >= 10000) {
                kill(child_pid, SIGKILL);
                kill(-child_pid, SIGKILL);
                kill_all_processes_except_self();
                while (true) {
                    int status;
                    pid_t p = waitpid(-1, &status, WNOHANG);
                    if (p <= 0) break;
                    if (p == child_pid) {
                        if (WIFEXITED(status)) child_exit_code = WEXITSTATUS(status);
                        else if (WIFSIGNALED(status)) child_exit_code = 128 + WTERMSIG(status);
                    }
                }
                close(sfd);
                return child_exit_code;
            }
            timeout_ms = 10000 - elapsed_ms;
            if (timeout_ms < 0) timeout_ms = 0;
        }

        int poll_ret = poll(&pfd, 1, timeout_ms);
        if (poll_ret == -1) {
            if (errno == EINTR) continue;
            close(sfd);
            return 1;
        }

        if (poll_ret == 0) {
            kill(child_pid, SIGKILL);
            kill(-child_pid, SIGKILL);
            kill_all_processes_except_self();
            while (true) {
                int status;
                pid_t p = waitpid(-1, &status, WNOHANG);
                if (p <= 0) break;
                if (p == child_pid) {
                    if (WIFEXITED(status)) child_exit_code = WEXITSTATUS(status);
                    else if (WIFSIGNALED(status)) child_exit_code = 128 + WTERMSIG(status);
                }
            }
            close(sfd);
            return child_exit_code;
        }

        struct signalfd_siginfo fdsi;
        ssize_t ret = read(sfd, &fdsi, sizeof(fdsi));
        if (ret == -1) {
            if (errno == EINTR) continue;
            close(sfd);
            return 1;
        }
        if (ret != sizeof(fdsi)) {
            close(sfd);
            return 1;
        }

        int sig = fdsi.ssi_signo;
        if (sig == SIGCHLD) {
            continue;
        }

        if (sig == SIGTERM || sig == SIGINT || sig == SIGQUIT || sig == SIGHUP) {
            if (!sigterm_sent) {
                sigterm_sent = true;
                if (clock_gettime(CLOCK_MONOTONIC, &start_time) != 0) {
                    reaper_log("run_reaper: clock_gettime failed, force-killing container\n");
                    kill(child_pid, SIGKILL);
                    kill(-child_pid, SIGKILL);
                    kill_all_processes_except_self();
                    while (true) {
                        int status;
                        pid_t p = waitpid(-1, &status, WNOHANG);
                        if (p <= 0) break;
                        if (p == child_pid) {
                            if (WIFEXITED(status)) child_exit_code = WEXITSTATUS(status);
                            else if (WIFSIGNALED(status)) child_exit_code = 128 + WTERMSIG(status);
                        }
                    }
                    close(sfd);
                    return child_exit_code;
                }
            }
            if (kill(child_pid, sig) == -1 && errno != ESRCH) {
                char msg[256];
                safe_snprintf(msg, sizeof(msg), "run_reaper: kill(%d, %d) failed: %s\n",
                              child_pid, sig, strerror(errno));
                reaper_log(msg);
            }
            if (kill(-child_pid, sig) == -1 && errno != ESRCH) {
                char msg[256];
                safe_snprintf(msg, sizeof(msg), "run_reaper: kill(-%d, %d) failed: %s\n",
                              child_pid, sig, strerror(errno));
                reaper_log(msg);
            }
            continue;
        }

        if (kill(child_pid, sig) == -1 && errno != ESRCH) {
            char msg[256];
            safe_snprintf(msg, sizeof(msg), "run_reaper: kill(%d, %d) failed: %s\n",
                          child_pid, sig, strerror(errno));
            reaper_log(msg);
        }
        if (kill(-child_pid, sig) == -1 && errno != ESRCH) {
            char msg[256];
            safe_snprintf(msg, sizeof(msg), "run_reaper: kill(-%d, %d) failed: %s\n",
                          child_pid, sig, strerror(errno));
            reaper_log(msg);
        }
    }
}


Module 5 Reference Notes

What this code does, in order:
1) Blocks all relevant signals (SIGCHLD, SIGTERM, SIGINT, SIGQUIT, SIGHUP, SIGUSR1, SIGUSR2) via sigprocmask.
2) Creates a signalfd that receives those signals instead of using async handlers.
3) Enters the main loop:
    Step 1: Drains all zombies with waitpid(-1, WNOHANG). If the main child is reaped, records its exit code and breaks out.
    Step 2: If the main child died, kills all remaining processes and returns.\
    Step 3: Polls the signalfd with a timeout (infinite unless a fatal signal has been forwarded).
    Step 4: On SIGCHLD, loops back to Step 1.
    Step 5: On SIGTERM/SIGINT/SIGQUIT/SIGHUP, starts the 10-second grace period (once), then forwards the signal to both the child PID and the child's process group.
    Step 6: On other signals, forwards them to both the child PID and the process group.
    Step 7: If the grace period expires (or clock_gettime fails), sends SIGKILL to the child and its group, kills all processes in the namespace, and returns.



What this code does NOT do:
1) Does not create the child — that's Module 1.
2) Does not configure the child's environment — that's Modules 3, 4, 7, 8, 9.
3) Does not use pidfd — that's Module 11 (future improvement).
4) Does not reap grandchildren via PR_SET_CHILD_SUBREAPER behavior — that's actually the caller's job (the parent process must set the subreaper flag before spawning, or the reaper itself will not receive SIGCHLD for orphaned grandchildren).

How Module 5 connects to the rest of CoreKernel:

Daemon (Go)
   │
   │ exec() of C++ helper binary
   ▼
corekernel-spawn (C++ binary, single-threaded)
   │
   ├── Module 1: spawn_container() → returns child_pid
   │
   └── Module 5: run_reaper(child_pid)  ← this file
           │
           ├── uses Module 14 (signals)  for signal mask setup
           ├── uses Module 15 (retry/log) for safe logging
           ├── calls kill_all_processes_except_self() on teardown
           └── returns child exit code to the helper's parent (Daemon)


Traps already handled:

Trap                                                            --->	Handling
SIGCHLD coalescing (kernel sends one for multiple deaths)	    --->    while (waitpid(-1, WNOHANG) > 0) drains all
SIGCHLD arriving before reaper enters poll                      --->	waitpid(-1, WNOHANG) runs before every poll call
signalfd read interrupted by signal                             --->	if (errno == EINTR) continue;
poll interrupted by signal                                      --->	if (errno == EINTR) continue;
clock_gettime failure                                           --->	Force-kill immediately, log, return
Time calculation overflow (tv_nsec borrow)                      --->	calculate_elapsed_ms handles negative borrow case
kill targeting a dead process                                   --->	errno != ESRCH check, otherwise silent
Grace period never expires because no SIGTERM sent              --->	timeout_ms = -1 means block indefinitely
Child calls setsid() and escapes process group                  --->	We send SIGKILL to both the PID and the process group
Child forked grandchildren that survived kill(-pgid)            --->	kill_all_processes_except_self() walks /proc
Grandchildren spawned after the final /proc scan                --->	5 attempts with 10 ms sleep, plus a final straggler pass
Reaper itself killed by an unblocked signal	                    --->    All relevant signals are blocked via sigprocmask
Zombie accumulation if reaper exits early                       --->	kill_all_processes_except_self() drains before returning
kill(-child_pid) when child is PID 1 of namespace               --->	PID 1 cannot receive signals from within its namespace with default disposition, but kill(child_pid, SIGKILL) from the parent namespace is allowed. Both calls are made.


Traps NOT yet handled in this file (handled elsewhere in CoreKernel):
1) Parent death signal: Set by the child before execve (PR_SET_PDEATHSIG), so if the reaper dies, the child does too. This is Module 10.
2) PID reuse race on kill(child_pid): Module 5 uses raw kill(). Module 11 (pidfd_send_signal) would fix this if it were integrated. Right now, the reaper relies on the child PID being valid, which is safe because the reaper is the direct parent and holds an unreaped child.
3) cgroup.kill for atomic teardown: Module 7 provides this. Right now, kill_all_processes_except_self walks /proc. If cgroup v2 + Linux 5.14+ is available, writing 1 to cgroup.kill is atomic and faster. Consider integrating this.
4) Signal forwarding to grandchildren in different sessions: Forwarding to -child_pid only reaches the child's process group. If a grandchild called setsid(), it's in a different group. kill_all_processes_except_self() catches them at teardown, but during the grace period they won't receive forwarded signals.


What the reaper does NOT do (design decisions):
1) It does not send SIGKILL immediately on SIGTERM — it honors a 10-second grace period.
2) It does not distinguish between SIGTERM and SIGINT — both start the grace period.
3) It does not forward SIGKILL — the kernel already delivers it to the target process; the reaper cannot block it.
4) It does not use pidfd — that's a future improvement.
5) It does not call cgroup.kill — that requires Module 7 to be integrated, and the current implementation uses /proc walking as a fallback.


Next step for this module: Integrate pidfd (Module 11) so that kill(child_pid, sig) becomes pidfd_send_signal(pidfd, sig) — eliminating the theoretical PID reuse race if the child is reaped but the PID is still referenced.
