+++
date = "2026-02-28"
title = "Tracking Log Origin Across Thread Pools in eBPF: The Clone Map Pattern"
slug = "ebpf-clone-map-thread-pool"
tags = ["ebpf", "thread-pools", "context-propagation", "observability"]
type = "post"
+++

When you build an eBPF-based log capture tool, you quickly run into a wall: thread pools.

The basic design is simple. You attach a tracepoint on `sys_enter_write`, intercept log output before it hits the filesystem, look up the calling thread's active span context from a BPF map, and attach that context to the log record. Automatic log-trace correlation with zero application changes.

It works great — until your application uses a thread pool.

## The Problem

Most real-world applications don't do work on the thread that received the request. They hand it off to a pool of pre-spawned worker threads. A Java HTTP server accepts the connection on one thread and executes your handler on a Tomcat thread pool worker. A Go HTTP server goroutines-out the handler (less relevant — goroutines share the same OS TID within a P), but Java, Python's ThreadPoolExecutor, Node.js worker threads, and most native apps all exhibit this pattern.

Here's what that does to your TID-based span lookup:

![Thread pool context problem — TID miss](/images/blog/ebpf-clone-map-thread-pool/ebpf-clone-map-problem.svg)

Your instrumentation registered the active span under the request thread's TID (say, 1001). The thread pool picks a worker (TID 1002) to handle the request. When that worker calls `write()` to emit a log line, your eBPF probe fires, calls `bpf_get_current_pid_tgid()` to get TID 1002, looks it up in `span_map` — and finds nothing. The log is emitted without any trace context. You're back to square one.

This isn't a niche case. In any Java or Python service doing non-trivial work, it's the default.

## The Obvious Approach and Why It Breaks

The first instinct is to propagate span context into the thread pool itself. Have the instrumentation push the active span into each worker thread's map entry when work is dispatched. This is exactly what Java's `ThreadLocal` and MDC (Mapped Diagnostic Context) do in SDK-based approaches.

But you're doing automatic instrumentation — no SDK, no code changes. You don't control the dispatch. And from BPF, you don't get a hook when user-space hands work to a thread pool. There's no "work dispatched to thread" event in the kernel.

The other approach: put a uprobe on `pthread_create` or Java's thread dispatch code. But that requires per-language uprobes with symbol resolution, which is fragile and doesn't generalize.

## The Clone Map Pattern

Here's the key insight: when work is handed to a thread pool, those workers were originally *spawned* by something. The pool's threads were created via `clone()` (on Linux, that's what `pthread_create` ultimately calls). And `clone()` is a syscall — it's universally hookable.

The idea: attach a `kretprobe` on `sys_clone`. Every time a new thread is created, record the parent→child TID relationship in an LRU BPF map. Then, when a TID lookup in the span map fails, walk up the parent chain until you find an ancestor with an active span.

```c
// clone_map.h

struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, u32);    // Child TID
    __type(value, u32);  // Parent TID
    __uint(max_entries, 10240);
} clone_map SEC(".maps");

// Walk up to 5 levels of parent chain. Covers most practical thread pool
// hierarchies while keeping the BPF verifier happy.
#define MAX_PARENT_CHAIN_LEVELS 5
```

`BPF_MAP_TYPE_LRU_HASH` is the right choice here. Thread creation and destruction is frequent in long-running services. You want the map to automatically evict stale entries when it fills up, without any manual cleanup. LRU handles that.

Max entries at 10240 covers ~10K live threads simultaneously. Size this based on your workload; each entry is 8 bytes (two u32s).

## Populating the Map from kretprobe

The `kretprobe/sys_clone` fires when `clone()` returns. The return value *is* the child TID (in the parent process; the child sees 0). Read it from the BPF context:

```c
SEC("kretprobe/sys_clone")
int BPF_KRETPROBE(handle_clone_ret, long ret) {
    if (ret <= 0) {
        return 0;  // clone failed or we're in the child
    }

    u32 child_tid = (u32)ret;
    u64 pid_tgid  = bpf_get_current_pid_tgid();
    u32 parent_tid = (u32)(pid_tgid & 0xFFFFFFFF);

    bpf_map_update_elem(&clone_map, &child_tid, &parent_tid, BPF_ANY);
    return 0;
}
```

One subtlety: `bpf_get_current_pid_tgid()` returns the *current* thread's pid_tgid — which in a kretprobe on clone is the *parent* thread making the clone call. The return value (`ret`) is the child TID. So this single probe call gives us both sides of the relationship.

## Walking the Chain in the Log Capture Probe

In your `sys_enter_write` tracepoint, after a span map miss, walk the clone_map:

```c
// In your sys_enter_write handler, after a miss on span_map:
u32 current_tid = (u32)(bpf_get_current_pid_tgid() & 0xFFFFFFFF);
struct span_context *span = NULL;

// Try direct lookup first
span = bpf_map_lookup_elem(&span_map, &current_tid);

// On miss, walk up the parent chain
if (!span) {
    u32 tid = current_tid;
    for (int i = 0; i < MAX_PARENT_CHAIN_LEVELS; i++) {
        u32 *parent = bpf_map_lookup_elem(&clone_map, &tid);
        if (!parent) break;

        span = bpf_map_lookup_elem(&span_map, parent);
        if (span) break;

        tid = *parent;
    }
}

// Use span (may still be NULL if no ancestor has a span)
if (span) {
    // attach trace_id and span_id to log event
}
```

The bounded `for` loop with a compile-time constant (`MAX_PARENT_CHAIN_LEVELS = 5`) is what satisfies the BPF verifier. The verifier needs to prove the loop terminates. A runtime-variable bound would get rejected. Five levels is enough for any thread pool hierarchy I've encountered: spawner → pool manager → worker → sub-task → sub-sub-task. In practice, it's usually two hops.

![Clone map solution — parent chain walk](/images/blog/ebpf-clone-map-thread-pool/ebpf-clone-map-solution.svg)

## Gotchas and Edge Cases

**Short-lived threads:** If a thread pool worker is destroyed and a new worker is spawned with the same TID (TID reuse is possible on Linux), the clone_map could have a stale parent entry. LRU eviction helps, but this is a real edge case. In practice, TID reuse collisions within a typical observability window are rare. If you see spurious span correlations, this is worth investigating.

**Goroutines:** Go's M:N scheduler means goroutines don't always correspond to OS threads. For Go-native instrumentation, the TID-based approach works when goroutines are scheduled on their own OS thread (via `runtime.LockOSThread()`), but it won't work generally for goroutines. This pattern is most useful for Java and native threads.

**Cleanup:** LRU handles eviction automatically when the map fills. But if you want to clean up eagerly when a process exits, you can hook `sys_exit_group` and delete entries for all TIDs belonging to that TGID.

**Performance:** Two additional BPF map lookups per log event (clone_map, then span_map again with parent). Both are O(1) hash lookups. The overhead is negligible compared to the cost of the ringbuf write.

## When NOT to Use This

If all your instrumentation is in a single BPF object and you control which TIDs get spans registered, you can often avoid this entirely by registering span context under the TGID (process ID) instead of the TID. All threads in a process share the same TGID. This only works if you have one active span per process at a time — fine for simple sequential services, wrong for anything concurrent.

The clone map pattern is the right tool when: you're doing automatic instrumentation with no control over application code, you're correlating across multiple independent BPF programs, and your target application uses real OS threads (not green threads/goroutines) for concurrency.

---

If this is useful, I'm doing more work in this space at [odigos-io/ebpf-core](https://github.com/odigos-io/ebpf-core) — automatic, zero-code-change observability via eBPF.
