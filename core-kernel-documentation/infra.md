┌────────────────────────────────────────────────────────────────────────────┐
│                         PUBLIC C API (extern "C")                          │
│                                                                            │
│   corekernel_spawn()   corekernel_join_ns()   corekernel_mount_overlay()   │
│   corekernel_set_cpu_max()   corekernel_kill_cgroup()   corekernel_reap()  │
└────────────────────────────────────┬───────────────────────────────────────┘
                                     │
┌────────────────────────────────────▼───────────────────────────────────────┐
│                          INTERNAL MODULE INTERFACES                        │
│                                                                            │
│   M1 (spawn)  ─────calls────►  M10 (attrs), M14 (signals), M15 (retry)     │
│   M1          ─────calls────►  M12 (fd mgmt)                               │
│   M2 (ns)     ─────calls────►  M12 (fd mgmt), M15 (retry)                  │
│   M3 (idmap)  ─────calls────►  M15 (retry)                                 │
│   M4 (mount)  ─────calls────►  M2 (ns), M12 (fd mgmt), M15 (retry)         │
│   M5 (reaper) ─────calls────►  M14 (signals), M15 (retry)                  │
│   M6 (cgroup) ─────calls────►  M12 (fd mgmt), M15 (retry)                  │
│   M7 (limits) ─────calls────►  M6 (cgroup fd), M15 (retry)                 │
│   M8 (caps)   ─────calls────►  M15 (retry)                                 │
│   M9 (seccomp)────calls────►  M15 (retry)                                  │
│   M11 (pidfd) ─────calls────►  M15 (retry)                                 │
│   M13 (time)  ─────calls────►  M15 (retry)                                 │
└────────────────────────────────────────────────────────────────────────────┘
