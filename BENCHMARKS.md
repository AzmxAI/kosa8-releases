# Performance baseline

`make bench` measures the targets docs/KOSA8_PRD.md commits to and prints
PASS/MISS per target. It needs guest assets and a real VM, so it is opt-in and
skips cleanly on a host that cannot run it.

Until this harness existed the performance requirements (FR-B1..B3, FR-D1,
FR-D2) were assertions with no measurement behind them. They are now measured.

## Baseline: Apple M-series, macOS 26.0.1, 2026-07-28

| Target | Measured | Verdict |
|---|---|---|
| FR-B1 idle footprint ≤ 400 MB | **61 MB** | pass, with large headroom |
| FR-B2 first run after install ≤ 60 s | **4.4 s** | pass |
| FR-B2 container cold start ≤ 250 ms | **862 ms** (median of 5) | **miss — 3.4× over** |
| FR-B3 bind-mount I/O ≥ 70% of native | **20%** (1.0 GB/s vs 4.8 GB/s) | **miss** |
| FR-D1 snapshot 2 GB sandbox ≤ 2000 ms | **886 ms** | pass |
| FR-D2 warm restore ≤ 1000 ms | **676 ms** (median of 3) | pass |
| sandbox boot (2 GB, informational) | 576 ms | — |

## Reading the two misses

**Cold start (862 ms vs 250 ms).** This is the whole `kosa8 run` path against
an already-booted engine VM with the image cached: CLI start, unix-socket
round trip, vsock request, containerd create+start, and streaming the exit
back. The 250 ms target is OrbStack-class and is not met. Closing it is
container-start-path work (task pre-warming, avoiding a fresh vsock connection
per operation), not a bug fix.

**Bind-mount I/O (20% of native).** 256 MiB written with `conv=fsync` on both
sides, so both figures are durable-write throughput rather than page cache —
without that flag the native side measures memory and the ratio is meaningless.
virtiofs sustains ~1.0 GB/s against ~4.8 GB/s native on this disk. Candidate
work: DAX window, queue depth, and mount options.

Both are real, quantified product gaps. Neither is a defect in the code that
exists; they are performance engineering still to be done, and the PRD should
not claim these targets are met until this table says so.

## Notes on method

- Cold start is a median of 5 and warm restore a median of 3: single samples
  flapped across the 1 s threshold and would have made the gate untrustworthy.
- Idle footprint sums RSS for `kosa8d` and any guest helper processes. Guest
  RAM is mapped lazily by Virtualization.framework, so this reflects host
  memory actually resident, which is what the target is about.
- Re-run on the target hardware before publishing any performance claim; these
  numbers are one machine, not a guarantee.
