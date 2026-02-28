+++
date = "2026-02-28"
title = "Tracking Log Origin Across Thread Pools in eBPF: The Clone Map Pattern"
slug = "ebpf-clone-map-thread-pool"
tags = ["ebpf", "thread-pools", "context-propagation", "observability"]
type = "post"
+++

Thread pools break a surprising number of eBPF-based observability tools. Not because the tools are badly designed — but because of a fundamental mismatch between how Linux models threads and how distributed tracing models work.

The fix is a pattern I've started calling the **clone map**: a lightweight BPF map that records the parent-child thread lineage established by `sys_clone`. It's simple, general-purpose, and useful well beyond log correlation. This post digs into the Linux thread model, why TID tracking matters for eBPF instrumentation, and several concrete use cases where the clone map pattern solves real problems.

## How Linux Models Threads

In Linux, there's no first-class "thread" concept at the kernel level. What you call a thread is just a `task_struct` that shares certain namespaces with its siblings — specifically the memory space (`CLONE_VM`), file descriptors (`CLONE_FILES`), and signal handlers.

When you call `pthread_create`, under the hood it calls `clone(2)` with a set of flags that produce this sharing:

```
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD | ...)
```

The result is a new `task_struct` with its own **TID** (Thread ID, also called `pid` in kernel space) but sharing the **TGID** (Thread Group ID, also called `tgid`, which userspace calls the process ID).

From eBPF, `bpf_get_current_pid_tgid()` returns a u64 where:
- Upper 32 bits = TGID (the process ID userspace sees)
- Lower 32 bits = TID (the specific thread ID)

For the main thread of a process, TID == TGID. For every other thread in the process, TID differs from TGID but TGID is shared.

This means: **TID is the finest-grained identity you can use to distinguish concurrent execution contexts in a process**. And that's exactly what makes thread pools painful.

## The Thread Pool Problem

Thread pools pre-spawn a fixed set of worker threads and reuse them across many requests. A typical web server:

1. Main thread accepts connections
2. Hands each request off to a thread pool worker (TID != main thread TID)
3. Worker executes the handler and emits logs, metrics, traces

In distributed tracing, a "span" tracks a unit of work. The span context (trace ID + span ID) needs to propagate with the work as it moves through threads. SDK-based tracing handles this via thread-local storage (Java's `ThreadLocal`, Python's `contextvars`) — the application explicitly propagates context when submitting work to the pool.

eBPF-based automatic instrumentation has no such luxury. You're operating below the application level. When your probe fires on a thread pool worker, you have its TID. But the span context may have been registered under the request thread's TID — or under no TID at all, if the worker was pre-spawned before any request arrived.

![Thread pool context problem — TID miss](/images/blog/ebpf-clone-map-thread-pool/ebpf-clone-map-problem.svg)

This isn't just a log correlation problem. Any eBPF tool that needs to associate a kernel event (syscall, network packet, scheduler event) with application-level context runs into the same wall.

## Why TID Ancestry Matters

Here's the key observation: **thread pools are hierarchical**. Workers are spawned by a pool manager, which was spawned by the application main thread. When work is dispatched to a worker, the context we want is associated with the *logical* owner of that work — often traceable up the spawn lineage.

Consider a few scenarios:

**Log-trace correlation.** Your log capture probe intercepts `write()` on a thread pool worker. The active span context lives in a BPF map keyed by the request handler thread's TID. The worker's TID misses. But the worker was spawned by a thread that *does* have span context, or its grandparent does.

**eBPF tracer ↔ log TID mismatch.** You have a Go tracer registering spans under goroutine-pinned OS thread TIDs. A separate log capture probe fires on a different OS thread (thread pool worker). The two TIDs don't match — even though they're doing the same logical work. Walking the spawn tree finds the common ancestor.

**Scheduler event attribution.** You want to attribute CPU scheduling events (`sched_switch`) to the request they're serving. Workers are interchangeable and don't have per-request TIDs. But their spawn parent does — and you can walk up to find the context.

**Network packet attribution.** A network packet is processed on a kernel receive thread, then handed off via a queue to an application thread pool worker. The worker's TID misses your application context map. Parent chain lookup finds the owning thread.

## The Clone Map

The solution is to track the parent-child TID relationship established at spawn time.

`sys_clone` is the syscall used for all thread creation on Linux (including Go's `runtime.newproc` path for goroutines when they pin an OS thread, and `pthread_create`). A `kretprobe` on `sys_clone` fires in the parent after the child is created. At that point:
- `bpf_get_current_pid_tgid()` gives you the parent's TID
- The return value `ret` is the child's TID (the parent sees the child's PID; the child sees 0)

This gives you both sides of the relationship in one probe:

```c
SEC("kretprobe/sys_clone")
int BPF_KRETPROBE(handle_clone_ret, long ret) {
    if (ret <= 0) return 0;  // failed clone, or we're the child

    u32 child_tid  = (u32)ret;
    u32 parent_tid = (u32)(bpf_get_current_pid_tgid() & 0xFFFFFFFF);

    bpf_map_update_elem(&clone_map, &child_tid, &parent_tid, BPF_ANY);
    return 0;
}
```

The map itself:

```c
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, u32);    // Child TID
    __type(value, u32);  // Parent TID
    __uint(max_entries, 10240);
} clone_map SEC(".maps");

#define MAX_PARENT_CHAIN_LEVELS 5
```

`BPF_MAP_TYPE_LRU_HASH` is the right choice: thread creation and destruction is frequent, and you want automatic eviction of stale entries rather than manual cleanup. Each entry is 8 bytes; 10240 entries covers ~10K live threads.

## Walking the Parent Chain

In any probe where you need context and the direct TID lookup fails:

```c
u32 tid = (u32)(bpf_get_current_pid_tgid() & 0xFFFFFFFF);
struct span_context *ctx = bpf_map_lookup_elem(&span_map, &tid);

if (!ctx) {
    // Walk up the parent chain
    u32 current = tid;
    for (int i = 0; i < MAX_PARENT_CHAIN_LEVELS; i++) {
        u32 *parent = bpf_map_lookup_elem(&clone_map, &current);
        if (!parent) break;

        ctx = bpf_map_lookup_elem(&span_map, parent);
        if (ctx) break;

        current = *parent;
    }
}
```

The bounded `for` loop with a compile-time constant is what the BPF verifier requires. A runtime-variable bound would be rejected. `MAX_PARENT_CHAIN_LEVELS = 5` is enough for any real thread pool hierarchy: spawner → pool manager → worker → sub-task → sub-sub-task. In practice, it's usually 1–2 hops.

![Clone map solution — parent chain walk](/images/blog/ebpf-clone-map-thread-pool/ebpf-clone-map-solution.svg)

## Gotchas

**TID reuse.** Linux reuses TIDs after threads exit. If a stale clone_map entry maps a recycled TID to the wrong parent, you can get spurious context attribution. LRU eviction helps but doesn't eliminate this. In practice, TID reuse within a typical observability window is rare. For high-churn environments (thousands of thread spawns/second), add eager cleanup via a `sys_exit` hook.

**Goroutines.** Go's M:N scheduler multiplexes goroutines onto OS threads (Ms). Goroutines don't have stable TIDs unless the goroutine calls `runtime.LockOSThread()`. For goroutine-level context in Go, you need a different approach (e.g., reading goroutine ID from the G struct via uprobe). The clone map works for Java, Python, C/C++, and any language using real POSIX threads.

**Process fork.** `fork()` also goes through `clone()` (without `CLONE_THREAD`). Your kretprobe will record fork relationships too. This is usually harmless — you can filter by TGID if you only want intra-process thread relationships.

**Performance.** Two to five additional O(1) hash lookups per event. Negligible. The dominating cost is always the ring buffer write.

## When NOT to Use This

If you control all instrumentation in a single BPF object and all threads in a process share one logical context (e.g., a single-request-per-process model), keying by TGID instead of TID is simpler and avoids the chain walk entirely.

The clone map earns its keep when: you're doing automatic instrumentation with no application code changes, you need to correlate events across independently developed BPF programs (different instrumentors for different languages), or your target application dispatches work across OS threads in ways that break TID-based context lookup.

---

The clone_map pattern is part of the eBPF instrumentation infrastructure in [odigos-io/ebpf-core](https://github.com/odigos-io/odigos) — automatic, zero-code-change observability via eBPF.
