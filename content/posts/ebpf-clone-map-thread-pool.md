+++
date = "2026-02-28"
title = "The Clone Map Pattern: Tracking Thread Ancestry in eBPF"
slug = "ebpf-clone-map-thread-pool"
tags = ["ebpf", "thread-pools", "context-propagation", "observability"]
type = "post"
+++

A recurring challenge in eBPF-based automatic instrumentation is this: the thread that does the interesting work is not the thread that has the context.

Your tracer registers a span under TID 1001. The work executes on TID 1002. The two events are causally related — but from the kernel's perspective, they're just two different threads. The clone map pattern is a simple, verifier-friendly way to bridge that gap by tracking thread ancestry directly in BPF.

## Why Thread Identity Is Hard in the Kernel

Linux exposes two identifiers via `bpf_get_current_pid_tgid()`:

- **TGID** (Thread Group ID): this is what user space calls the "PID". All threads in a process share one TGID.
- **TID** (Thread ID): the actual OS-level thread identifier. Each thread gets a unique TID.

When you write a BPF probe that fires on a syscall or tracepoint, you're executing in the context of a specific thread — a specific TID. That's all you know. You don't know who spawned this thread, what task it was given, or what request it's servicing.

This matters because modern applications almost never do work on the thread that initiates the work. They use thread pools, work queues, and executors. A Java HTTP server (Tomcat, Jetty, Netty) accepts connections on one thread and handles requests on a pool of workers. Python's `ThreadPoolExecutor`, C++ `std::async`, and most native applications follow the same model. The thread that registered context and the thread that does work are causally linked but not the same.

![Thread pool context problem — TID miss](/images/blog/ebpf-clone-map-thread-pool/ebpf-clone-map-problem.svg)

## The Core Problem (Generalized)

Any eBPF program that does per-thread context lookup — `bpf_map_lookup_elem(&ctx_map, &current_tid)` — will miss whenever work has been handed off to a different thread than the one that registered context.

This shows up in several real scenarios:

**Log-trace correlation:** An OpenTelemetry tracer instruments the request thread (TID 1001) with a trace ID. The application logs from a thread pool worker (TID 1002). The log capture probe fires on TID 1002, looks up `span_map[1002]` — empty. The log loses its trace context.

**Distributed tracing across eBPF programs:** You have one eBPF program that hooks network ingress and stamps an incoming request's TID with a request ID. A second eBPF program hooks a database syscall to track query latency. If the database call happens from a thread pool worker (different TID), the second program can't find the request ID the first one stored.

**Security and audit tools:** An eBPF-based auditor registers a file access policy per TID based on the process's current privilege state. Work handed off to a worker thread bypasses the policy check because the worker's TID has no registered state.

**Profiling attribution:** CPU profiling probes fire on whatever thread is scheduled. Attributing that CPU time to the right request requires knowing which request the thread is currently servicing — which means knowing its ancestry.

In all these cases, the fix is the same: track who spawned whom, then walk up the chain.

## The Clone Map Pattern

The key insight is that thread creation is a syscall — `clone()`. On Linux, `pthread_create` calls `clone()` under the hood, and so does virtually every threading primitive in every language runtime. `clone()` is universally hookable.

Attach a `kretprobe` on `sys_clone`. Every time a thread is created, record the parent→child TID relationship in an LRU BPF map. When a TID lookup misses, walk up the parent chain.

```c
// clone_map: records thread ancestry
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, u32);    // Child TID
    __type(value, u32);  // Parent TID
    __uint(max_entries, 10240);
} clone_map SEC(".maps");

#define MAX_PARENT_CHAIN_LEVELS 5
```

`BPF_MAP_TYPE_LRU_HASH` is intentional: thread pools constantly spawn and destroy workers. LRU automatically evicts stale entries when the map fills, with no manual cleanup needed.

## Populating clone_map via kretprobe

The `kretprobe/sys_clone` fires in the parent thread when `clone()` returns. The return value is the child TID.

```c
SEC("kretprobe/sys_clone")
int BPF_KRETPROBE(handle_clone_ret, long ret) {
    if (ret <= 0) {
        return 0;  // clone failed, or we're in the child (child sees 0)
    }

    u32 child_tid  = (u32)ret;
    u64 pid_tgid   = bpf_get_current_pid_tgid();
    u32 parent_tid = (u32)(pid_tgid & 0xFFFFFFFF);

    bpf_map_update_elem(&clone_map, &child_tid, &parent_tid, BPF_ANY);
    return 0;
}
```

One detail: `bpf_get_current_pid_tgid()` returns the *current* thread's identity. In a kretprobe on clone, we're executing in the parent thread that called `clone()`. The return value `ret` is the newly created child's TID. One probe, both sides of the relationship.

## Walking the Chain on a Lookup Miss

In whatever probe needs the lookup, fall back to parent-chain walking on a miss:

```c
u32 current_tid = (u32)(bpf_get_current_pid_tgid() & 0xFFFFFFFF);
void *ctx = bpf_map_lookup_elem(&ctx_map, &current_tid);

if (!ctx) {
    u32 tid = current_tid;
    for (int i = 0; i < MAX_PARENT_CHAIN_LEVELS; i++) {
        u32 *parent = bpf_map_lookup_elem(&clone_map, &tid);
        if (!parent) break;

        ctx = bpf_map_lookup_elem(&ctx_map, parent);
        if (ctx) break;

        tid = *parent;
    }
}
```

The compile-time constant bound (`MAX_PARENT_CHAIN_LEVELS = 5`) is what keeps the BPF verifier happy. It needs to prove the loop terminates. A runtime variable would be rejected. Five levels covers any realistic thread pool hierarchy: main → pool manager → worker → sub-task → nested sub-task. In practice it's usually two hops.

![Clone map solution — parent chain walk](/images/blog/ebpf-clone-map-thread-pool/ebpf-clone-map-solution.svg)

## How This Solves the Tracer / Log TID Mismatch

Concretely, in an OpenTelemetry-style auto-instrumentation system:

1. Network ingress probe fires on TID 1001. Stores `span_map[1001] = {trace_id, span_id}`.
2. Tomcat dispatches the request to a worker pool. Worker TID is 1002. `clone_map[1002] = 1001` was recorded when the pool was initialized.
3. Log capture probe fires on TID 1002. Looks up `span_map[1002]` — miss.
4. Walks `clone_map`: TID 1002 → parent 1001 → `span_map[1001]` — hit.
5. Log gets stamped with the correct trace context.

The chain walk happens in the eBPF probe itself, with no userspace coordination in the hot path. No locks, no ring buffer round-trips.

## Gotchas and Edge Cases

**TID reuse:** Linux can reuse TIDs after a thread exits. If a new thread gets the same TID as an old one, `clone_map` may have a stale parent entry. In practice this is rare within typical observability windows, but worth monitoring if you see spurious correlations.

**Goroutines:** Go's M:N scheduler multiplexes goroutines onto OS threads, and the mapping changes dynamically. TID-based tracking works for goroutines only when they're pinned with `runtime.LockOSThread()`. This pattern targets real OS threads — most useful for Java, Python, and native C/C++ applications.

**Eager cleanup:** LRU handles eviction automatically. For eager cleanup when a process exits, hook `sys_exit_group` and purge entries for all TIDs belonging to that TGID.

**Cross-process pools:** The pattern assumes the parent TID is meaningful for context lookup — i.e., the parent registered context in the same map. If work crosses process boundaries (different TGIDs), you'd need to key the map by TID+TGID pairs and scope lookups accordingly.

**Performance:** Two extra BPF map lookups per event on a miss path (clone_map + ctx_map with parent TID). Both are O(1) hash lookups. Negligible compared to the cost of any ring buffer write or perf event output.

## When NOT to Use This

If you control which TIDs get context registered — and there's at most one active context per process — use TGID keying instead. All threads in a process share a TGID. This is simpler and faster.

Use the clone map when:
- You're doing automatic instrumentation with no control over application code
- Multiple independent eBPF programs need to share context across thread hops
- Your target language uses real OS threads (not goroutines or async tasks)
- You need to correlate events that originate on different threads of the same request

---

If this is useful, I'm doing more work in this space at [odigos-io/odigos](https://github.com/odigos-io/odigos) — automatic, zero-code-change observability via eBPF.
