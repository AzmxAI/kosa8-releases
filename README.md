# kosa8 releases

Prebuilt binaries and packages for [kosa8](https://kosa8.com) — an AI-native
container platform with microVM isolation and snapshot/fork.

The source lives in a private repository; this repo carries the released
artifacts and the documents worth reading before you install.

## Install

Downloads are on the [releases page](https://github.com/AzmxAI/kosa8-releases/releases).

| Platform | Package | Contents |
|---|---|---|
| macOS (Apple Silicon) | `kosa8_<v>_darwin_arm64.zip` | `kosa8` + `kosa8d` — the full runtime |
| macOS (Intel) | `kosa8_<v>_darwin_amd64.zip` | `kosa8` CLI only |
| Linux | `.tar.gz`, `.deb`, `.rpm`, `.apk` (amd64/arm64) | `kosa8` + `kosa8-guestd` |
| Windows | `kosa8_<v>_windows_<arch>.zip` | `kosa8.exe` |

macOS builds are signed with a Developer ID and notarized by Apple. Windows
builds are Authenticode-signed.

**A non-macOS `kosa8` cannot run containers locally.** `kosa8d` needs
Virtualization.framework, so Linux and Windows builds are *remote clients* —
set `KOSA8_HOST` and `KOSA8_TOKEN` to drive a macOS worker. That is a
supported mode, not a degraded one, but it is not a local runtime.

## What to read first

- **[BENCHMARKS.md](BENCHMARKS.md)** — measured performance against every
  target the product commits to, including the two that are currently missed.
- **[COMMAND_COVERAGE.md](COMMAND_COVERAGE.md)** — all 48 `docker` commands
  mapped to kosa8, with honest status per command.

## Status

kosa8 is a pre-1.0 **developer preview**. Snapshot, restore, fork, egress
policy, audit and the Docker-compatible API all work and are covered by an
end-to-end test against real VMs. Container cold start and bind-mount
throughput both currently miss their targets — see BENCHMARKS.md for the
numbers rather than a claim.

## Verifying a download

Each release ships `checksums.txt`. On macOS you can also confirm the
signature directly:

```sh
codesign -dv kosa8d            # expect a Developer ID, not "adhoc"
spctl -a -vv kosa8d            # Gatekeeper's own verdict
```

---

## License

kosa8 is proprietary software. Copyright (c) 2026 kosa8. All rights reserved.
Use requires a valid kosa8 subscription or evaluation; see [LICENSE](LICENSE).
Each release ships `THIRD_PARTY_NOTICES.md` listing the third-party components
it contains and their licences.

Releases up to and including v0.4.2 were published under Apache-2.0; anyone who
received those versions keeps those rights to them. Every later release is under
the kosa8 licence.
