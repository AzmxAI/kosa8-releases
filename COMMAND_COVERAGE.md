# Docker command coverage

Every command in `docker --help` (48 total), mapped to kosa8. Status is what
is actually implemented and tested, not aspiration.

Legend: **✅ works** · **🟡 partial** · **⛔ not yet** (honest error, no fake success)

## Containers

| Docker | kosa8 | Status | Notes |
|---|---|---|---|
| `run` | `kosa8 run` | ✅ | auto-pulls, streams output, propagates exit code |
| `create` | `kosa8 create` | ✅ | `--name` supported |
| `start` | `kosa8 start` | ✅ | |
| `stop` | `kosa8 stop` | ✅ | SIGTERM then SIGKILL after 10s (dockerd semantics) |
| `kill` | `kosa8 kill` | ✅ | `-s SIGNAL` |
| `restart` | `kosa8 restart` | ✅ | stop + start |
| `pause` | `kosa8 pause` | ✅ | cgroup freezer via containerd |
| `unpause` | `kosa8 unpause` | ✅ | |
| `rm` | `kosa8 rm` | ✅ | `-f`; refuses running containers otherwise |
| `ps` | `kosa8 ps` | ✅ | merges docker-created and engine-native containers |
| `logs` | `kosa8 logs` | ✅ | stdout/stderr demultiplexed |
| `exec` | `kosa8 exec` | ✅ | exit codes propagated; `-t` for TTY |
| `run -i` / `-t` | `kosa8 run -i` / `-t` | ✅ | stdin streamed into the container; raw mode for TTY |
| `attach` | `kosa8 attach` | ✅ | replay + follow |
| `wait` | `kosa8 wait` | ✅ | |
| `top` | `kosa8 top` | 🟡 | lists PIDs (no full ps table) |
| `rename` | `kosa8 rename` | ✅ | |
| `inspect` | `kosa8 inspect` | ✅ | container inspect |
| `port` | `kosa8 port` | ✅ | with `run -p [HOST:]CONTAINER` |
| `stats` | `kosa8 stats` | ✅ | cgroup v2 CPU/memory/pids |
| `cp` | `kosa8 cp` | ✅ | both directions, via a temp rootfs mount |
| `diff` | `kosa8 diff` | ✅ | changes vs the image (A/C/D) |
| `commit` | `kosa8 commit` | ✅ | new layer via containerd diff, uncompressed so digest == diffID |
| `export` | `kosa8 export` | ✅ | container rootfs tar, `-o` |
| `update` | `kosa8 update` | ✅ | `--cpus`, `--memory`; verified against the cgroup |
| `checkpoint` | `kosa8 snapshot` / `fork` | ✅+ | **stronger than docker**: whole-VM state, forkable |

## Images

| Docker | kosa8 | Status | Notes |
|---|---|---|---|
| `pull` | `kosa8 pull` | ✅ | own registry client (token auth, multi-arch) |
| `images` | `kosa8 images` | ✅ | |
| `tag` | `kosa8 tag` | ✅ | |
| `rmi` | `kosa8 rmi` | ✅ | |
| `push` | `kosa8 push` | ✅ | blob upload + manifest PUT, push-scoped tokens |
| `build` | `kosa8 build` | ✅ | BuildKit (OCI worker); `-t`, `-f`, `--build-arg`, `.dockerignore` |
| `history` | `kosa8 history` | ✅ | from the image config |
| `save` / `load` | `kosa8 save` / `load` | ✅ | OCI archives, `-o` / `-i` |
| `import` | `kosa8 import` | ✅ | rootfs tar to a single-layer image, `-t` |
| `search` | `kosa8 search` | ✅ | Docker Hub search API |

## System

| Docker | kosa8 | Status | Notes |
|---|---|---|---|
| `version` | `kosa8 version` | ✅ | client + daemon |
| `info` | `kosa8 info` | ✅ | also `kosa8 system info` |
| `events` | `kosa8 events` | 🟡 | stream opens; no event payloads yet |
| `system prune` | `kosa8 system prune` | ✅ | removes exited containers |
| `login` / `logout` | `kosa8 login` / `logout` | ✅ | credentials at ~/.kosa8/credentials.json (0600) |
| `context` | — | ⛔ | single local daemon; no remote endpoints yet |
| `network` | `kosa8 network` | ✅ | create/ls/rm; each network its own bridge and /24 |
| `volume` | `kosa8 volume` | ✅ | create/ls/rm on the engine data disk |
| `plugin` | — | ⛔ | not planned |
| `manifest` | `kosa8 manifest inspect` | ✅ | remote manifest without pulling |
| `builder` | `kosa8 builder prune` | 🟡 | build + cache prune work; `du`/cache import-export not exposed |

## kosa8-only (no Docker equivalent)

| Command | What it does |
|---|---|
| `kosa8 sandbox create --egress deny\|--egress-allow HOST` | restrict what a sandbox's containers may reach on the network |
| `kosa8 sandbox create -v HOST:GUEST` | boot an isolated microVM with its own kernel, engine, and host mounts |
| `kosa8 sandbox run/exec/start/ps/pull/images` | drive a sandbox's private container engine |
| `kosa8 snapshot create/ls/restore/rm` | capture and restore **whole-VM state including running processes** |
| `kosa8 fork -n N` | clone a live sandbox into N independent VMs, processes intact |
| `kosa8 doctor` | verify the machine can run kosa8 |
| `kosa8 daemon start/status/stop` | manage kosa8d |
| `kosa8 agent run KIT` | run a coding agent in a sandbox that can only see one directory and reach the hosts its kit allows |
| `kosa8 agent kits / init` | list Sandbox Kits, or copy one to ~/.kosa8/kits to customize |
| `kosa8 audit ls / verify` | hash-chained record of every state-changing action, tagged human or agent |
| `kosa8 up` | detect a project, generate a Dockerfile, build and run it |
| `kosa8 mcp serve` | **run kosa8 as an MCP server** — 23 typed tools so AI agents drive the runtime |
| `kosa8 mcp tools` | list the tools exposed to agents |

## Docker CLI compatibility

The upstream `docker` CLI drives kosa8 directly — no kosa8-specific flags:

```
export DOCKER_HOST=unix://$HOME/.kosa8/run/kosa8.sock
docker run --rm alpine echo hi
```

Compat suite: **24/24 passing** (`make test-compat`), MCP suite **11/11**
(`make test-mcp`). Covered there: version,
info, run, stdout/stderr framing, exit codes, create→start→logs→inspect→rm,
ps -a, run -d, logs, exec, kill, and guest-container leak checks.

## Remaining

**45 of 48 implemented.** What is left, and why:

- **`context`** — kosa8 talks to one local daemon; remote endpoints are a
  Phase 2 concern (they pair with cloud burst)
- **`plugin`** — not planned; MCP is the extension model
- **`checkpoint`** — superseded by `kosa8 snapshot` / `fork`, which capture
  whole-VM state including running processes

Container `--network` selection is wired through the engine protocol; the
CLI flag for it is still to come.
