# profiler

Independent function timing collectors, implemented in Encore using `std`.

```encore
import profiler::Profiler

static PROFILER_A: Profiler<H> = Profiler::new("parser")

@PROFILER_A.profile("parse_resume")
fn parse_resume(text: str) -> usize {
    ret text.byte_len()
}
```

`Profiler::new(name: str) -> Profiler<H>` explicitly allocates a heap node.
Every constructor call creates an independent collector, including calls with
the same name. Copies of one collector's handle share its state. A `static`
retains the collector for the lifetime of the process and initializes it once.
Requires a compiler with the static-initialization fix for issue #153 and
representation-preserving mutable node annotations (`mut T&`).

Use `@PROFILER_A.profile_async("label")` on async functions. The synchronous
and asynchronous decorators preserve arguments and return values, including
`Result::Err`. Methods and nested or recursive calls can also be decorated.

## Reading statistics

- `name()` returns the collector's name.
- `stats(label) -> Option[ProfileStats]` returns independent statistics for one
  label. It returns `None` until a call with that label completes, and after
  `reset()` until another call completes.
- `snapshot() -> Vec[ProfileStats]` returns an independent snapshot in the order
  labels first completed. Each entry exposes `label()`, `calls()`, `total_ns()`,
  `min_ns()`, `max_ns()`, `mean_ns()`, `p50_ns()`, `p95_ns()` and `p99_ns()`.
  The mean is an `f64`; the other durations are `u64` nanoseconds.
- `report_text() -> str` formats the same statistics without printing or resetting
  them. Pass this string to `println` or your logger.
- `reset()` requires a mutable collector handle. Previously obtained snapshots
  remain unchanged:

```encore
let before = PROFILER_A.snapshot()
let mut profiler = PROFILER_A
profiler.reset()
```

Read individual fields through their methods:

```encore
match PROFILER_A.stats("parse_resume") {
    Option::Some(stats) => {
        let label = stats.label()
        let calls = stats.calls()
        let total_ns = stats.total_ns()
        let min_ns = stats.min_ns()
        let max_ns = stats.max_ns()
        let mean_ns = stats.mean_ns()
        let p50_ns = stats.p50_ns()
        let p95_ns = stats.p95_ns()
        let p99_ns = stats.p99_ns()
    }
    Option::None => {}
}
```

Import `core::option::Option` when matching the lookup result.

Percentiles use the nearest-rank rule over all completed calls for that label:
sort the durations and select rank `ceil(count * percentage / 100)` (one-based).
For a single call, every percentile equals its duration. With fewer than 20
calls, p95 is the maximum; p50 for two calls is the smaller duration. Statistics
are computed from a sorted copy, so reading them does not change the collector.

## Measurement boundaries

Measurements use the monotonic `perf_counter_ns` clock. Async durations include
time suspended at `await`; nested durations include child calls. Only completed
calls contribute a sample. A dropped future, panic or process exit does not
record an unfinished call. Constructing an unpolled future records nothing.
Calls that complete after `reset()` contribute to the new statistics.

This version is for one OS thread, including cooperative async tasks on that
thread. Collector initialization does not make mutable statistics thread-safe.
Labels are explicit, and identical labels within one collector are combined.
Recording scans existing labels, so its cost is linear in the number of labels;
appending a duration is amortized constant time. Exact percentiles retain one
`u64` per completed call, plus vector capacity, until `reset()`. Memory therefore
grows with the number of completed calls. Reading a label with `n` calls copies
and sorts its durations in `O(n log n)` time using `O(n)` temporary storage.
The measured interval excludes recording and reporting overhead, although parent
measurements include child instrumentation.

See `example/` for a runnable example.
