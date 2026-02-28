+++
date = "2026-02-28"
title = "Why bpf_ktime_get_ns() Is Useless for Log Timestamps (and What to Use Instead)"
slug = "ebpf-ktime-monotonic-wallclock"
tags = ["ebpf", "observability", "timestamps", "debugging"]
type = "post"
+++

When you write your first eBPF log capture tool, timestamping log events feels trivial. Call `bpf_ktime_get_ns()`, store the result in your event struct, ship it out the ring buffer. Done.

Then you try to correlate those logs with traces and discover the timestamps are nonsense — off by days, weeks, or the entire uptime of the machine. The numbers are technically precise; they're just measuring the wrong thing.

This post traces `bpf_ktime_get_ns()` all the way down to the kernel clock source, explains three approaches to getting usable wall-clock time in eBPF, and builds a decision tree for which one to use.

## What `bpf_ktime_get_ns()` Actually Returns

`bpf_ktime_get_ns()` is a thin wrapper around the kernel's `ktime_get_ns()`, which operates on the `CLOCK_MONOTONIC` domain.

`CLOCK_MONOTONIC` is defined in POSIX as a clock that:
- Never goes backward
- Has no defined relationship to wall-clock time
- Starts at an arbitrary point (typically system boot, but not guaranteed by the spec)

In practice on Linux, it starts at boot and runs forward. After 47 days of uptime, `bpf_ktime_get_ns()` returns roughly `4,060,800,000,000,000` nanoseconds. That's not a Unix timestamp — it's a duration since boot.

If you try to use it as a Unix timestamp, you'll get dates in 1970 (for short uptimes) or wildly wrong values for any non-trivial uptime. Correlating with traces that use `CLOCK_REALTIME`-based timestamps (which is what every tracing SDK, OpenTelemetry, and Jaeger uses) is simply not possible without conversion.

There are a few other BPF time helpers worth knowing:

| Helper | Clock domain | Notes |
|---|---|---|
| `bpf_ktime_get_ns()` | `CLOCK_MONOTONIC` | No wall-clock relation |
| `bpf_ktime_get_coarse_ns()` | `CLOCK_MONOTONIC_COARSE` | Lower precision, faster |
| `bpf_ktime_get_boot_ns()` | `CLOCK_BOOTTIME` | Includes time suspended |
| `bpf_ktime_get_tai_ns()` | `CLOCK_TAI` | Kernel 5.16+, atomic time, no leap smearing |
| `bpf_ktime_get_real_ns()` | `CLOCK_REALTIME` | Wall-clock, **kernel 5.16+** |

`bpf_ktime_get_real_ns()` is the obvious answer — it's `CLOCK_REALTIME` directly from BPF. But it requires kernel 5.16+, released in January 2022. If you need to support older kernels (enterprise Linux distros, many cloud providers), you need a different approach.

## Three Approaches

### Approach 1: BPF-Side Timestamp with Offset Calibration

Load time: snapshot both the monotonic clock and wall-clock time from userspace, compute an offset, store it in a BPF map. The probe adds the offset to `bpf_ktime_get_ns()` to approximate wall-clock time.

```go
// At BPF load time
func computeKtimeOffset() int64 {
    // Sample multiple times and take the tightest pair
    // to minimize the race between the two reads
    var minGap int64 = math.MaxInt64
    var bestOffset int64

    for i := 0; i < 10; i++ {
        before := time.Now().UnixNano()
        ktime := readKtimeNs()  // read via /proc/timer_list or a one-shot BPF call
        after := time.Now().UnixNano()

        gap := after - before
        if gap < minGap {
            minGap = gap
            // Approximate: ktime was read at the midpoint
            bestOffset = (before+after)/2 - ktime
        }
    }
    return bestOffset
}
```

Store `bestOffset` in a BPF map (a single-entry array map works). In the probe:

```c
u64 ktime_ns = bpf_ktime_get_ns();
u64 *offset  = bpf_map_lookup_elem(&ktime_offset_map, &zero);
u64 wall_ns  = offset ? ktime_ns + *offset : ktime_ns;
```

This is what Falco does, and what several Cilium features use. It works — with caveats.

**Why it drifts.**

NTP and the kernel's `adjtime()` can slew `CLOCK_REALTIME` by up to 500 parts per million (ppm) to correct for clock drift. Over 24 hours at maximum slew, that's ~43 milliseconds of accumulated error. The offset you computed at load time becomes stale.

Virtualized environments make this worse. With `kvm-clock` as the clock source, the guest's monotonic clock is disciplined by the hypervisor, not by NTP. The relationship between `CLOCK_MONOTONIC` and `CLOCK_REALTIME` in the guest can drift independently.

**When it's acceptable.**

For coarse observability — dashboards, alerting, log search with second-level granularity — a few milliseconds of drift is fine. If you need sub-millisecond timestamp accuracy and must support kernels before 5.16, this is the best you can do from BPF.

You can mitigate drift by periodically re-calibrating the offset from userspace (every few minutes) and updating the BPF map. The probe always reads the latest value.

### Approach 2: `bpf_ktime_get_real_ns()` (Kernel 5.16+)

If your minimum kernel is 5.16, this is the clean solution. It reads `CLOCK_REALTIME` directly from BPF — the same clock source that `time.Now()` uses in Go, `System.currentTimeMillis()` in Java, and every tracing SDK.

```c
u64 wall_ns = bpf_ktime_get_real_ns();
```

No offset, no drift, no calibration. The timestamp is directly comparable to trace spans.

The only gotcha: `CLOCK_REALTIME` can go backward if NTP makes a step correction (as opposed to a slew). For monotonic ordering of events within a single capture session, combine it with `bpf_ktime_get_ns()` for ordering and `bpf_ktime_get_real_ns()` for the display timestamp.

### Approach 3: Timestamp in Userspace at Ring Buffer Consumption

For log capture specifically, skip BPF-side timestamping entirely. Emit the log event from the ring buffer without a timestamp, and call `time.Now()` in userspace when the event is consumed.

```go
func (lc *LogCollector) processEvents(ctx context.Context) {
    for {
        record, err := lc.reader.Read()
        // ...

        // Timestamp here, not in BPF
        now := time.Now()

        var event LogEvent
        binary.Read(bytes.NewReader(record.RawSample), binary.LittleEndian, &event)

        rec.SetTimestamp(now)
        rec.SetObservedTimestamp(now)
        // ...
    }
}
```

![Timestamp comparison — BPF-side vs userspace](/images/blog/ebpf-ktime-monotonic-wallclock/ebpf-ktime-comparison.svg)

**Why this is correct for logs.**

The ring buffer is a shared memory region between kernel and userspace. When `bpf_ringbuf_submit()` is called in the probe, the event is immediately visible to userspace readers. Ring buffer consumption latency on a non-saturated system is typically 10µs–500µs.

For log-trace correlation, trace spans have millisecond-scale durations. A sub-millisecond timestamp error is noise. What you actually want semantically is "when was this log observed by the collection pipeline" — which is precisely what `time.Now()` at consumption time gives you. This matches the OTel log data model's `observed_timestamp` field exactly.

The approach also sidesteps the 5.16 kernel requirement entirely and requires zero BPF map reads in the hot path.

## When BPF-Side Timestamps Are the Right Answer

Everything above assumes you're timestamping for display and correlation. Sometimes you need BPF-side timestamps for a different reason: **measuring duration between two kernel events**.

```c
// In a kprobe on tcp_sendmsg
u64 send_ts = bpf_ktime_get_ns();
bpf_map_update_elem(&send_times, &sock_cookie, &send_ts, BPF_ANY);

// In a kprobe on tcp_ack received
u64 *send_ts = bpf_map_lookup_elem(&send_times, &sock_cookie);
u64 rtt_ns = bpf_ktime_get_ns() - *send_ts;
```

Here, `bpf_ktime_get_ns()` is exactly right. You're computing a delta between two monotonic readings — the absolute value doesn't matter, only the difference. Using `bpf_ktime_get_real_ns()` here would introduce NTP slew into your latency measurements, which is worse.

The rule: **monotonic for deltas, real-time for display**.

## Decision Tree

```
Do you need wall-clock time (correlate with traces, display to humans)?
├── Yes
│   ├── Kernel >= 5.16?
│   │   ├── Yes → bpf_ktime_get_real_ns()
│   │   └── No → Offset calibration (re-calibrate every few minutes)
│   └── Is sub-ms accuracy required?
│       ├── No → time.Now() at ring buffer consumption (simplest, correct)
│       └── Yes → bpf_ktime_get_real_ns() or offset calibration
└── No (measuring duration between kernel events)
    └── bpf_ktime_get_ns() — monotonic delta, no conversion needed
```

For log capture on any kernel: **time.Now() at consumption**. You get wall-clock accuracy, no kernel version requirement, no BPF map overhead, and semantics that match what the OTel log spec actually calls for.

---

The timestamping approach described here is part of the log capture implementation in [odigos-io/ebpf-core](https://github.com/odigos-io/ebpf-core).
