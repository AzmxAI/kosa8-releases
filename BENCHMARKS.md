# Performance baseline

`make bench` measures the targets docs/KOSA8_PRD.md commits to and prints
PASS/MISS per target. It needs guest assets and a real VM, so it is opt-in and
skips cleanly on a host that cannot run it.

## Baseline: Apple M-series, macOS 26.0.1, 2026-07-28

| Target | Measured | Verdict |
|---|---|---|
| FR-B1 idle footprint ≤ 400 MB | **64 MB** | pass, large headroom |
| FR-B2 first run after install ≤ 60 s | **1.9 s** | pass |
| FR-B2 container cold start ≤ 250 ms | **178 ms** | **pass** (was 883 ms) |
| FR-B3 bind-mount I/O ≥ 70% of native | **29%** of host native | **miss — see below** |
| FR-D1 snapshot 2 GB sandbox ≤ 2000 ms | **1795 ms** | pass |
| FR-D2 warm restore ≤ 1000 ms | **897 ms** (median of 3) | pass |

Five of six targets are met.

## Cold start: 883 ms → 178 ms

Profiling attributed the whole cost guest-side, then phase timing inside the
guest (`kosa8.trace=1`) pinned it exactly:

```
new-container  709ms      <- everything
new-task        12ms
cni-attach      23ms      <- the obvious suspect, and not the cause
task-start       1ms
```

`containerd.WithSnapshotter("native")` copies the entire image rootfs for every
container, so the cost scaled with image size — and alpine is about as small as
an image gets. The guest kernel already had overlay and containerd already
loaded the overlayfs snapshotter, so switching cost nothing and made the
operation copy-on-write:

```
new-container   52ms   (was 709)
```

## Bind-mount I/O: why 70% of native is the wrong target

The headline 29% is real but misleading, so the harness now measures the
ceiling too:

| Path | % of host native |
|---|---|
| Host native (APFS on NVMe) | 100% |
| **The VM's own block device** | **35%** |
| virtiofs bind mount | 29% |

The virtual machine cannot reach host-native throughput on *any* path: its own
virtio-blk disk tops out around 35%. That is the cost of the virtualization
boundary, and no file-sharing mechanism can avoid it.

Measured against what is actually achievable inside the VM, **virtiofs runs at
83% of the ceiling**. There is perhaps 17% left on the table; the other ~65% is
not virtiofs's to give.

So FR-B3 as written — 70% of *host native* — is arithmetically unreachable on
Apple's Virtualization.framework while the VM's own disk caps at 35%. The
target should be restated against the in-VM ceiling (where 83% is already close
to the 70% bar) or dropped in favour of an absolute MB/s figure. Chasing the
current wording means chasing a number the platform does not permit.

Apple's virtiofs host implementation is closed and exposes no DAX window or
cache-mode control to the guest, so the remaining 17% is not reachable by mount
options either.

## Notes on method

- Cold start is a median of 5, warm restore a median of 3: single samples
  flapped across the threshold and made the gate untrustworthy.
- Both sides of the I/O comparison use `conv=fsync`, so they measure durable
  writes rather than page cache. Without it the native figure measures memory
  and the ratio is meaningless.
- Idle footprint sums RSS for `kosa8d` and guest helpers. Guest RAM is mapped
  lazily by Virtualization.framework, so this reflects host memory actually
  resident.
- Re-run on the target hardware before publishing any performance claim; these
  are one machine, not a guarantee.
