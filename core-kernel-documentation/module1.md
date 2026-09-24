MODULE 1: Process Creation
Files: spawn.cpp, clone_stack.cpp, sync_protocol.cpp

What it does: Creates a new process via clone3() with CLONE_INTO_CGROUP and namespace flags. Manages the stack. Runs the 4-state sync protocol.

Internal dependencies: Module 10 (attributes), Module 14 (signals), Module 15 (retry/log)

External API: int corekernel_spawn(const ContainerArgs* args, ContainerHandle* out);

Traps:

1) clone_args version pinning (already handled with clone_args_cgroup)

2) EINTR on clone3 (already handled)

3) Stack guard page missing (already handled)

Sample code:

#define _GNU_SOURCE
#include <cerrno>
#include <cstdint>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <ctime>
#include <fcntl.h>
#include <poll.h>
#include <sched.h>
#include <signal.h>
#include <sys/mman.h>
#include <sys/syscall.h>
#include <sys/wait.h>
#include <unistd.h>

/* Architecture-aware fallback for SYS_clone3 */
#ifndef SYS_clone3
#if defined(__x86_64__) || defined(__s390x__) || defined(__powerpc64__)
#define SYS_clone3 435
#elif defined(__aarch64__)
#define SYS_clone3 434
#elif defined(__riscv)
#define SYS_clone3 437
#else
#warning "SYS_clone3 is not defined for this architecture. Please ensure your kernel headers are up to date."
#endif
#endif

/* Clone flags we use */
#ifndef CLONE_INTO_CGROUP
#define CLONE_INTO_CGROUP (1ULL << 33)
#endif

extern char **environ;

constexpr size_t STACK_SIZE = 1024 * 1024;
constexpr int SYNC_TIMEOUT_MS = 300000; /* 5 minutes for extreme filesystem latency */

/* ------------------------------------------------------------------ */
/*  FIX: Version-Pinned clone_args Struct                             */
/*                                                                    */
/*  We define our own struct containing exactly the fields up to and  */
/*  including 'cgroup' (introduced in Linux 5.7). This guarantees a   */
/*  stable 88-byte size regardless of the kernel headers used at      */
/*  compile time. If we used the system's struct clone_args and it    */
/*  had grown (e.g., shadow_stack in 6.x), deploying the binary to    */
/*  an older kernel would cause E2BIG.                                */
/* ------------------------------------------------------------------ */

struct clone_args_cgroup {
    uint64_t flags;
    uint64_t pidfd;
    uint64_t child_tid;
    uint64_t parent_tid;
    uint64_t exit_signal;
    uint64_t stack;
    uint64_t stack_size;
    uint64_t tls;
    uint64_t set_tid;
    uint64_t set_tid_size;
    uint64_t cgroup;       /* Last field we need (Linux 5.7) */
};

/* Compile-time assertion: this struct must be exactly 88 bytes on 64-bit */
static_assert(sizeof(clone_args_cgroup) == 88,
              "clone_args_cgroup must be exactly 88 bytes (11 x uint64_t)");

/* ------------------------------------------------------------------ */
/*  Strict 4-State Sync Protocol                                      */
/* ------------------------------------------------------------------ */

constexpr uint8_t SYNC_STATUS_SETUP_OK    = 0;
constexpr uint8_t SYNC_STATUS_ABOUT_EXEC  = 1;
constexpr uint8_t SYNC_STATUS_SETUP_FAIL  = 2;
constexpr uint8_t SYNC_STATUS_EXEC_FAIL   = 3;

/* ------------------------------------------------------------------ */
/*  Robust I/O helpers                                                */
/* ------------------------------------------------------------------ */

static int write_fully(int fd, const void *buf, size_t size)
{
    const char *p = static_cast<const char*>(buf);
    size_t left = size;
    while (left > 0) {
        ssize_t n = write(fd, p, left);
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        if (n == 0) return -1;
        if (static_cast<size_t>(n) > left) n = static_cast<ssize_t>(left);
        p += n;
        left -= static_cast<size_t>(n);
    }
    return 0;
}

static ssize_t read_with_timeout(int fd, void *buf, size_t size, int timeout_ms)
{
    char *p = static_cast<char*>(buf);
    size_t left = size;
    
    struct timespec ts_start, ts_now;
    if (clock_gettime(CLOCK_MONOTONIC, &ts_start) == -1) return -1; 
    
    long long deadline_ns = static_cast<long long>(ts_start.tv_sec) * 1000000000LL + 
                            ts_start.tv_nsec + 
                            static_cast<long long>(timeout_ms) * 1000000LL;

    while (left > 0) {
        if (clock_gettime(CLOCK_MONOTONIC, &ts_now) == -1) return -1;
        
        long long now_ns = static_cast<long long>(ts_now.tv_sec) * 1000000000LL + ts_now.tv_nsec;
        int remaining_ms = static_cast<int>((deadline_ns - now_ns) / 1000000LL);
        if (remaining_ms <= 0) return -2;

        struct pollfd pfd;
        pfd.fd = fd;
        pfd.events = POLLIN;
        pfd.revents = 0;

        int ret = poll(&pfd, 1, remaining_ms);
        if (ret < 0) { 
            if (errno == EINTR) continue; 
            return -1; 
        }
        if (ret == 0) return -2; 
        
        ssize_t n = read(fd, p, left);
        if (n < 0) { 
            if (errno == EINTR) continue; 
            return -1; 
        }
        if (n == 0) break; 
        if (static_cast<size_t>(n) > left) n = static_cast<ssize_t>(left);
        p += n;
        left -= static_cast<size_t>(n);
    }
    return static_cast<ssize_t>(size - left);
}

/* ------------------------------------------------------------------ */
/*  Safe close and waitpid helpers                                    */
/* ------------------------------------------------------------------ */

static void safe_close(int *fd_ptr, const char *label)
{
    if (fd_ptr && *fd_ptr >= 0) {
        if (close(*fd_ptr) == -1) {
            fprintf(stderr, "close(%s): %s\n", label, strerror(errno));
        }
        *fd_ptr = -1;
    }
}

static pid_t safe_waitpid(pid_t pid, int *status, int options)
{
    pid_t ret;
    do { 
        ret = waitpid(pid, status, options); 
    } while (ret == -1 && errno == EINTR);
    return ret;
}

/* ------------------------------------------------------------------ */
/*  Parent-side: interpret the sync-pipe result                       */
/* ------------------------------------------------------------------ */

static pid_t check_child_exec(pid_t child_pid, int sync_pipe_rd)
{
    uint8_t status = 0;
    uint8_t next_status = 0;
    
    ssize_t n = read_with_timeout(sync_pipe_rd, &status, 1, SYNC_TIMEOUT_MS);
    
    if (n == -2) { fprintf(stderr, "Timeout waiting for child setup status\n"); goto fail_and_reap; }
    if (n != 1)  { fprintf(stderr, "Child died or I/O error before reporting setup status\n"); goto fail_and_reap; }

    if (status == SYNC_STATUS_SETUP_FAIL) {
        int child_errno = 0;
        n = read_with_timeout(sync_pipe_rd, &child_errno, sizeof(child_errno), SYNC_TIMEOUT_MS);
        safe_close(&sync_pipe_rd, "sync_pipe_rd");
        
        if (n == static_cast<ssize_t>(sizeof(child_errno))) {
            fprintf(stderr, "Child setup failed: %s\n", strerror(child_errno));
        } else if (n > 0) {
            fprintf(stderr, "Child setup failed: partial error code (%zd bytes)\n", n);
        } else if (n == 0) {
            fprintf(stderr, "Child setup failed: pipe closed prematurely\n");
        } else if (n == -2) {
            fprintf(stderr, "Child setup failed: timeout reading error code\n");
        } else {
            fprintf(stderr, "Child setup failed: I/O error\n");
        }
        
        safe_waitpid(child_pid, nullptr, 0);
        return -1;
    }

    if (status != SYNC_STATUS_SETUP_OK) {
        fprintf(stderr, "Protocol error: invalid setup status byte (%d)\n", status);
        goto fail_and_reap;
    }

    n = read_with_timeout(sync_pipe_rd, &next_status, 1, SYNC_TIMEOUT_MS);
    
    if (n == -2) { fprintf(stderr, "Timeout waiting for child exec status\n"); goto fail_and_reap; }
    if (n == 0)  { fprintf(stderr, "Child died before reaching execve()\n"); goto fail_and_reap; }
    if (n != 1)  { fprintf(stderr, "I/O error reading exec status\n"); goto fail_and_reap; }

    if (next_status == SYNC_STATUS_ABOUT_EXEC) {
        uint8_t exec_status = 0;
        n = read_with_timeout(sync_pipe_rd, &exec_status, 1, SYNC_TIMEOUT_MS);

        if (n == 0) {
            /* 
             * EOF after ABOUT_EXEC: execve() succeeded (O_CLOEXEC closed pipe).
             * KNOWN OS LIMITATION: The only way this is a false positive is if 
             * an external signal (like SIGKILL) killed the child in the microsecond 
             * between sigprocmask returning and the execve kernel trap.
             */
            safe_close(&sync_pipe_rd, "sync_pipe_rd");
            return child_pid;
        }
        if (n == -2) { fprintf(stderr, "Timeout waiting for execve() result\n"); goto fail_and_reap; }
        if (n == -1) { fprintf(stderr, "I/O error reading execve() result\n"); goto fail_and_reap; }

        if (exec_status == SYNC_STATUS_EXEC_FAIL) {
            int child_errno = 0;
            n = read_with_timeout(sync_pipe_rd, &child_errno, sizeof(child_errno), SYNC_TIMEOUT_MS);
            
            if (n == static_cast<ssize_t>(sizeof(child_errno))) {
                fprintf(stderr, "Child execve failed: %s\n", strerror(child_errno));
            } else if (n > 0) {
                fprintf(stderr, "Child execve failed: partial error code (%zd bytes), child died mid-write\n", n);
            } else if (n == 0) {
                fprintf(stderr, "Child execve failed: pipe closed before error code\n");
            } else if (n == -2) {
                fprintf(stderr, "Child execve failed: timeout waiting for error code\n");
            } else {
                fprintf(stderr, "Child execve failed: I/O error\n");
            }
            
            goto fail_and_reap;
        }

        fprintf(stderr, "Protocol error: unexpected exec status byte (%d)\n", exec_status);
        goto fail_and_reap;
    }

    fprintf(stderr, "Protocol error: unexpected status byte after setup OK (%d)\n", next_status);

fail_and_reap:
    safe_close(&sync_pipe_rd, "sync_pipe_rd");
    safe_waitpid(child_pid, nullptr, 0);
    return -1;
}

/* ------------------------------------------------------------------ */
/*  Public API                                                        */
/* ------------------------------------------------------------------ */

pid_t spawn_container(const char *cgroup_path, char *const argv[])
{
    int sync_pipe[2];
    if (pipe2(sync_pipe, O_CLOEXEC) == -1) { 
        perror("pipe2"); 
        return -1; 
    }

    int cgroup_fd = open(cgroup_path, O_RDONLY | O_DIRECTORY | O_CLOEXEC);
    if (cgroup_fd == -1) {
        perror("open cgroup");
        safe_close(&sync_pipe[0], "pipe_rd"); 
        safe_close(&sync_pipe[1], "pipe_wr");
        return -1;
    }

    /* Allocate stack with a hardware-enforced PROT_NONE guard page */
    long page_size = sysconf(_SC_PAGESIZE);
    if (page_size <= 0) page_size = 4096;
    size_t total_stack_size = STACK_SIZE + static_cast<size_t>(page_size);

    void *stack_base = mmap(nullptr, total_stack_size, PROT_READ | PROT_WRITE,
                            MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
    if (stack_base == MAP_FAILED) {
        perror("mmap stack");
        safe_close(&cgroup_fd, "cgroup_fd");
        safe_close(&sync_pipe[0], "pipe_rd"); 
        safe_close(&sync_pipe[1], "pipe_wr");
        return -1;
    }

    /* Guard page at the lowest address (stacks grow down) */
    if (mprotect(stack_base, static_cast<size_t>(page_size), PROT_NONE) == -1) {
        perror("mprotect guard page");
        if (munmap(stack_base, total_stack_size) == -1) {
            perror("munmap stack (cleanup)");
        }
        safe_close(&cgroup_fd, "cgroup_fd");
        safe_close(&sync_pipe[0], "pipe_rd"); 
        safe_close(&sync_pipe[1], "pipe_wr");
        return -1;
    }

    /* The actual usable stack starts after the guard page */
    void *stack = static_cast<char*>(stack_base) + page_size;

    /* 
     * FIX: Use our version-pinned struct instead of the system's struct clone_args.
     * This guarantees a stable 88-byte size regardless of compile-time headers.
     */
    clone_args_cgroup ca{}; 
    ca.flags       = CLONE_INTO_CGROUP | CLONE_NEWPID | CLONE_NEWNS;
    ca.cgroup      = static_cast<uint64_t>(cgroup_fd);
    ca.stack       = static_cast<uint64_t>(reinterpret_cast<uintptr_t>(stack)); 
    ca.stack_size  = STACK_SIZE;
    ca.exit_signal = SIGCHLD;

    long ret;
    do {
        /* Pass the size of our pinned struct, NOT sizeof(struct clone_args) */
        ret = syscall(SYS_clone3, &ca, sizeof(ca));
    } while (ret == -1 && errno == EINTR);

    if (ret == -1) {
        perror("clone3");
        safe_close(&cgroup_fd, "cgroup_fd");
        safe_close(&sync_pipe[0], "pipe_rd"); 
        safe_close(&sync_pipe[1], "pipe_wr");
        if (munmap(stack_base, total_stack_size) == -1) {
            perror("munmap stack (cleanup)");
        }
        return -1;
    }

    if (ret == 0) {
        /* ---------------- CHILD PATH ---------------- */
        safe_close(&sync_pipe[0], "pipe_rd");

        /* PHASE 1: Before SETUP_OK - errors report SETUP_FAIL */
        struct sigaction sa_old_pipe{}, sa_ign_pipe{};
        sa_ign_pipe.sa_handler = SIG_IGN;
        sigemptyset(&sa_ign_pipe.sa_mask);
        sa_ign_pipe.sa_flags = 0;
        if (sigaction(SIGPIPE, &sa_ign_pipe, &sa_old_pipe) == -1) {
            int err = errno;
            uint8_t fail_status = SYNC_STATUS_SETUP_FAIL;
            (void)write_fully(sync_pipe[1], &fail_status, 1);
            (void)write_fully(sync_pipe[1], &err, sizeof(err));
            _exit(EXIT_FAILURE);
        }

        uint8_t status = SYNC_STATUS_SETUP_OK;
        if (write_fully(sync_pipe[1], &status, 1) != 0) {
            _exit(EXIT_FAILURE);
        }

        /* PHASE 2: After SETUP_OK, before ABOUT_EXEC - errors cause EOF */
        sigset_t mask, old_mask;
        sigfillset(&mask);
        if (sigprocmask(SIG_BLOCK, &mask, &old_mask) == -1) {
            _exit(EXIT_FAILURE);
        }

        status = SYNC_STATUS_ABOUT_EXEC;
        if (write_fully(sync_pipe[1], &status, 1) != 0) {
            _exit(EXIT_FAILURE);
        }

        /* PHASE 3: After ABOUT_EXEC - errors report EXEC_FAIL */
        if (sigaction(SIGPIPE, &sa_old_pipe, nullptr) == -1) {
            int err = errno;
            status = SYNC_STATUS_EXEC_FAIL;
            (void)write_fully(sync_pipe[1], &status, 1);
            (void)write_fully(sync_pipe[1], &err, sizeof(err));
            _exit(EXIT_FAILURE);
        }

        if (sigprocmask(SIG_SETMASK, &old_mask, nullptr) == -1) {
            int err = errno;
            status = SYNC_STATUS_EXEC_FAIL;
            (void)write_fully(sync_pipe[1], &status, 1);
            (void)write_fully(sync_pipe[1], &err, sizeof(err));
            _exit(EXIT_FAILURE);
        }

        execve(argv[0], argv, environ);

        int err = errno;
        status = SYNC_STATUS_EXEC_FAIL;
        (void)write_fully(sync_pipe[1], &status, 1);
        (void)write_fully(sync_pipe[1], &err, sizeof(err));
        _exit(EXIT_FAILURE);
    }

    /* ---------------- PARENT PATH ---------------- */
    safe_close(&cgroup_fd, "cgroup_fd");
    safe_close(&sync_pipe[1], "pipe_wr");

    pid_t result = check_child_exec(static_cast<pid_t>(ret), sync_pipe[0]);
    
    if (munmap(stack_base, total_stack_size) == -1) {
        perror("munmap stack");
    }
    
    return result;
}



Module 1 Reference Notes
What this code does, in order:
1) Creates a O_CLOEXEC pipe for the sync protocol.
2) Opens the target cgroup directory.
3) Allocates a 1 MB stack with a 4 KB guard page at the bottom.
4) Populates the version-pinned clone_args_cgroup struct.
5) Calls clone3() with CLONE_INTO_CGROUP | CLONE_NEWPID | CLONE_NEWNS.
6) In the child: runs the 4-state sync protocol, then execve.
7) In the parent: closes unused FDs, calls check_child_exec(), unmaps the stack.



What will be added by the caller (Daemon or a wrapper):
-> Additional namespace flags: CLONE_NEWNET, CLONE_NEWUSER, CLONE_NEWUTS, CLONE_NEWIPC, CLONE_NEWCGROUP.
-> The actual ContainerArgs struct with argc, envp, rootfs_path, host_uid, host_gid, etc.
-> The full container_init bootstrap that runs inside the child before execve — that's where the mount setup, UID/GID maps, device nodes, and capabilities go.
-> The reaper loop that runs in the parent after spawn_container returns.



What Module 1 does NOT do:
-> Mount operations (pivot_root, mount, /proc, /sys, /dev) — that's Module 4.
-> UID/GID maps — that's Module 3.
-> Capability or seccomp installation — that's Modules 8 and 9.
-> The reaper loop — that's Module 5.
-> Signal forwarding — that's Module 14.


How Module 1 connects to the rest of CoreKernel:

Daemon (Go)
   │
   │ exec() of C++ helper binary
   ▼
corekernel-spawn (C++ binary, single-threaded)
   │
   ├── Module 1: spawn_container()     ← this file
   │       │
   │       ├── uses Module 10 (attrs)  for prctl setup in child
   │       ├── uses Module 14 (signals) for signal mask reset
   │       ├── uses Module 15 (retry)   for EINTR handling
   │       └── uses Module 12 (FD mgmt) for close_range
   │
   └── Module 5: run_reaper()          ← after spawn returns
           │
           └── handles child exit, signals, timeouts

Traps handled:

Trap                                             --->	Handling
clone_args version mismatch                      --->    clone_args_cgroup with static_assert(88)
EINTR on clone3                                  --->	do { ret = syscall(...); } while (ret == -1 && errno == EINTR)
Stack guard page missing                         --->	mmap + mprotect guard page
SIGPIPE killing the child during sync writes     --->	Ignored during Phase 1 and Phase 3, restored before execve
Signal mask inherited by child                   --->	sigprocmask block/unblock around the sync protocol
Parent never knows if execve succeeded           --->	O_CLOEXEC pipe + EOF detection
execve failure                                   --->	Child writes EXEC_FAIL + errno before _exit
Resource leak on clone3 failure                  --->	safe_close on all FDs and munmap on the stack



Traps NOT yet handled in this file (handled elsewhere in CoreKernel):
1) Parent death (PR_SET_PDEATHSIG) — actually set in Module 10, called from the child bootstrap inside container_init.
2) Cgroup limit writes — done by the parent after seeing SYNC_STATUS_SETUP_OK.
3) Mount setup — done in the child bootstrap.
4) Reaper loop — done after spawn_container returns.
