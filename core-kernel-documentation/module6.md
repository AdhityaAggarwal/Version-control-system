MODULE 6: Cgroup Placement
Files: cgroup_placement.cpp

What it does: Atomic placement via CLONE_INTO_CGROUP. Fallback for kernels without clone3.

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



Module 6 Reference Notes
What this code does, in order:
1) Creates an O_CLOEXEC pipe for the sync protocol.
2) Opens the target cgroup v2 directory (O_RDONLY | O_DIRECTORY | O_CLOEXEC) to obtain a file descriptor.
3) Allocates a 1 MB stack with a 4 KB PROT_NONE guard page at the bottom.
4) Populates the version-pinned clone_args_cgroup struct with CLONE_INTO_CGROUP, namespace flags, and the cgroup FD.
5) Calls clone3() via raw syscall() to place the child atomically into the target cgroup at birth.
6) In the child: runs the 4-state sync protocol, then execve.
7) In the parent: closes unused FDs, waits for child's status via check_child_exec(), unmaps the stack.


What this code does NOT do:
1) Does not open the target cgroup for limits — that's Module 7 (limits) done by the parent after seeing SETUP_OK.
2) Does not verify the child actually landed in the cgroup — that would require reading /proc/<pid>/cgroup. Useful for debugging but not done here.
3) Does not create the cgroup — the caller (Daemon) is responsible for creating /sys/fs/cgroup/<slice>/<id>.scope before calling spawn_container.\
4) Does not apply fallback for older kernels — no fork() + cgroup.procs path is included here.
5) Does not use namespace flags beyond CLONE_NEWPID | CLONE_NEWNS — the caller should extend this.


How Module 6 connects to the rest of CoreKernel:

Daemon (Go)
   │
   │ 1. Create cgroup directory: /sys/fs/cgroup/<slice>/<id>.scope
   │ 2. exec() of corekernel-spawn binary with cgroup_path as argument
   ▼
corekernel-spawn (C++ binary, single-threaded)
   │
   ├── Module 6: spawn_container(cgroup_path, argv)  ← this file
   │       │
   │       ├── opens cgroup_fd
   │       ├── calls clone3(CLONE_INTO_CGROUP | CLONE_NEWPID | CLONE_NEWNS)
   │       │
   │       ├── CHILD: 
   │       │     ├── writes SETUP_OK
   │       │     ├── calls Module 3 (UID/GID maps)   [in bootstrap]
   │       │     ├── calls Module 4 (mount ops)       [in bootstrap]
   │       │     ├── calls Module 8 (capabilities)    [in bootstrap]
   │       │     ├── calls Module 9 (seccomp)         [in bootstrap]
   │       │     ├── writes ABOUT_EXEC
   │       │     └── execve()
   │       │
   │       └── PARENT:
   │             ├── reads SETUP_OK
   │             ├── calls Module 7 (cgroup limits)  ← write cpu.max, memory.max
   │             ├── reads ABOUT_EXEC
   │             ├── waits for EOF (execve success)
   │             └── returns child_pid
   │
   └── Module 5: run_reaper(child_pid)                ← after spawn returns



Traps already handled:

Trap                                        --->	Handling
clone_args version mismatch	                --->    clone_args_cgroup with static_assert(88)
EINTR on clone3                             --->	do { ret = syscall(...); } while (ret == -1 && errno == EINTR)
Stack guard page missing                    --->	mmap + mprotect(PROT_NONE) on the bottom page
Stack alignment                             --->	(child's stack pointer is set by kernel to stack + STACK_SIZE)
SIGPIPE during sync writes                  --->	Ignored during Phase 1, restored before execve
Signal mask inheritance	                    --->    Blocked during sync, unblocked before execve
Parent cannot tell if execve succeeded      --->	O_CLOEXEC pipe + EOF detection
execve failure with reason                  --->	Child writes EXEC_FAIL + errno before _exit
cgroup_fd leak on failure                   --->	safe_close on all exit paths
Stack leak on failure                       --->	munmap(stack_base, total_stack_size) on all error paths
SYS_clone3 unavailable                      --->	Architecture-aware fallback constant
CLONE_INTO_CGROUP undefined                 --->	Fallback define (1ULL << 33)



Traps NOT handled in this file (design decisions or future work):

Trap                                                --->	Where It Should Be Handled
Kernel doesn't support clone3 (< 5.3)               --->	Future: fallback to fork() + cgroup.procs write
Kernel doesn't support CLONE_INTO_CGROUP (< 5.7)    --->	Future: fallback path with traditional two-step placement
cgroup v1 only (no v2)                              --->	Daemon should detect and reject; CLONE_INTO_CGROUP requires v2
cgroup path doesn't exist                           --->	Daemon creates it before calling spawn_container
cgroup not empty (needs cgroup.subtree_control)     --->	Daemon configures parent cgroup hierarchy
cgroup.type = threaded                              --->	Daemon must ensure the target cgroup is not in threaded mode
Cgroup limits applied after execve                  --->	The parent writes limits after SETUP_OK and before ABOUT_EXEC — this is the entire reason for the 4-state protocol
Parent death while child sets up                    --->	PR_SET_PDEATHSIG set by child bootstrap (Module 10)
PID reuse race on kill                              --->	Future: integrate Module 11 (pidfd)


Compare with the fallback (unused):

Two-step (racy):
  fork() ──────► child runs ~100µs outside cgroup ──────► write(pid, cgroup.procs)
                     ⚠️  RACE WINDOW  ⚠️

CLONE_INTO_CGROUP (atomic):
  clone3(...) ─────► child born inside cgroup ──────► child runs
                        ✅  NO RACE



What the caller (Daemon) must do before calling spawn_container:
1) Create the cgroup directory: mkdir("/sys/fs/cgroup/<slice>/<id>.scope", 0755).
2) Enable the required controllers in the parent cgroup: echo "+cpu +memory +io +pids" > /sys/fs/cgroup/<slice>/cgroup.subtree_control.
3) Ensure the target cgroup has no child cgroups (cgroup v2 rule: a cgroup with domain controllers can only have processes if it has no child cgroups).
4) Pass the full path to spawn_container.


What the caller must do after spawn_container returns:
1)If it returns a valid PID: apply cgroup limits by calling Module 7 (set_cpu_max, set_memory_max, etc.). Wait — the parent inside spawn_container should have already applied them between SETUP_OK and ABOUT_EXEC. If the current implementation does not do that, the caller must do it. This is a known gap in the current version — the sync protocol has hooks for it, but the actual limit writes are not in this file.
2) Call Module 5 (run_reaper) with the returned PID.
3) On run_reaper return: clean up the cgroup directory (rmdir).



Next step for this module: Wire Module 7 into the parent's path between SETUP_OK and ABOUT_EXEC, so limits are enforced before the child execves.
