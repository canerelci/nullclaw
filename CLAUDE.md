# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Mandatory Reference

Read `AGENTS.md` before any code change. It is the authoritative engineering protocol covering architecture, naming conventions, anti-patterns, change playbooks, and validation requirements.

## Build & Test Commands

```bash
# Requires exactly Zig 0.15.2 (verify: zig version)
zig build                           # dev build
zig build -Doptimize=ReleaseSmall   # release build (target: <1 MB binary)
zig build test --summary all        # run all 3,371+ tests (must pass with 0 leaks)
zig fmt src/                        # format all source files
zig fmt --check src/                # check formatting (used by pre-commit hook)
```

Primary validation command is `zig build test --summary all` (project-wide). Individual files can still be run with `zig test <file>.zig` when needed.

### Build Flags

```bash
zig build -Dchannels=telegram,cli   # compile only specific channels (default: all)
zig build -Dengines=base,sqlite     # compile only specific memory engines (default: base,sqlite)
zig build -Dtarget=x86_64-linux-musl  # cross-compile for target triple
zig build -Dversion=2026.3.1        # override CalVer version string
```

Channel tokens: `all`, `none`, or comma-separated names (`cli`, `telegram`, `discord`, `slack`, `signal`, `matrix`, `web`, `nostr`, `irc`, `email`, `imessage`, `whatsapp`, `mattermost`, `lark`, `dingtalk`, `line`, `onebot`, `qq`, `maixcam`).

Engine tokens: `base`/`minimal` (enables `none`, `markdown`, `memory`, `api`), `sqlite`, `lucid`, `redis`, `lancedb`, `postgres`, `all`.

## Git Hooks

Activate once per clone:

```bash
git config core.hooksPath .githooks
```

- **pre-commit**: blocks if `zig fmt --check src/` fails
- **pre-push**: blocks if `zig build test --summary all` fails

## Project Overview

NullClaw is an autonomous AI assistant runtime written in Zig 0.15.2. Hard constraints: 678 KB binary, ~1 MB peak RSS, <2 ms startup. Every dependency and abstraction has a measurable size/memory cost. Only two external dependencies: vendored SQLite (with build-time SHA256 hash verification) and `websocket.zig` (pinned commit).

## Architecture

The entire codebase is **vtable-driven**. All major subsystems use `ptr: *anyopaque` + `vtable: *const VTable` for pluggable implementations. Extending NullClaw means implementing a vtable struct and registering it in the subsystem's factory (see `AGENTS.md` section 7 for playbooks).

**Critical ownership rule**: callers must OWN the implementing struct (local var or heap-alloc). Never return a vtable interface pointing to a temporary -- the pointer will dangle.

### Module Initialization Order

Defined in `src/root.zig`. Phases mirror deployment dependencies:

1. **Core**: `bus`, `config`, `util`, `platform`, `version`, `state`, `json_util`, `http_util`
2. **Agent**: `agent`, `session`, `providers`, `memory`
3. **Networking**: `gateway`, `channels`
4. **Extensions**: `security`, `cron`, `health`, `tools`, `identity`, `cost`, `observability`, `heartbeat`, `runtime`, `mcp`, `subagent`, `auth`, `multimodal`, `agent_routing`
5. **Hardware/Integrations**: `hardware`, `peripherals`, `rag`, `skillforge`, `tunnel`, `voice`

### Key Entry Points

- `src/main.zig` - CLI command routing (`agent`, `gateway`, `onboard`, `doctor`, `status`, `service`, `cron`, `channel`, `memory`, `skills`, `hardware`, `migrate`, `workspace`, `capabilities`, `models`, `auth`, `update`)
- `src/root.zig` - Module hierarchy and public API exports (also serves as library root)
- `src/config.zig` - JSON config loading (~30 sub-config structs from `config_types.zig`, loads from `~/.nullclaw/config.json`)
- `src/agent.zig` - Agent orchestration (delegates to `src/agent/root.zig`)
- `src/gateway.zig` - HTTP gateway server (rate limiting, pairing, webhooks)
- `src/daemon.zig` - Supervisor with exponential backoff for gateway mode

### Subsystem Directories

- `src/providers/` - AI model providers. 9 core implementations + 41+ OpenAI-compatible services via `compatible.zig`. Factory in `factory.zig`, single source of truth for provider URLs and auth styles.
- `src/channels/` - Messaging channels. Each implements `Channel.VTable` (`start`, `stop`, `send`, `name`, `healthCheck`). Factory in `root.zig`.
- `src/tools/` - Tool implementations. Each implements `Tool.VTable` (`execute`, `name`, `description`, `parameters_json`). Tools receive args as `JsonObjectMap` and return `ToolResult`. Factory in `root.zig`.
- `src/memory/` - Layered architecture: **engines** (SQLite, Markdown, LRU, Redis, PostgreSQL, LanceDB, Lucid, API, None) and **retrieval** (hybrid search, RRF, embeddings). Engines conditionally compiled via build flags.
- `src/security/` - Policy enforcement (`policy.zig`), pairing (`pairing.zig`), encrypted secrets (`secrets.zig`), sandbox backends (`landlock.zig`, `firejail.zig`, `bubblewrap.zig`, `docker.zig`, `detect.zig`).
- `src/agent/` - Agent loop internals: `dispatcher.zig` (tool call parsing), `compaction.zig` (history trimming), `prompt.zig` (system prompt builder), `memory_loader.zig` (context injection), `commands.zig` (agent-mode commands). Config defaults are `max_tool_iterations = 1000` and `max_history_messages = 100` (see `src/config_types.zig`).

### Dependency Direction

Concrete implementations depend inward on vtable interfaces, config, and util. Never import across subsystems (e.g., provider code must not import channel internals).

## Config System

Config loads from `~/.nullclaw/config.json`. Runtime behavior is then adjusted by `NULLCLAW_*` environment overrides (see `Config.applyEnvOverrides()` in `src/config.zig`). Types are defined in `src/config_types.zig` and re-exported from `src/config.zig`.

`Config.load()` heap-allocates an internal `ArenaAllocator`. Always call `defer cfg.deinit()` to free. In tests, wrap in a parent arena:

```zig
var arena = std.heap.ArenaAllocator.init(std.testing.allocator);
defer arena.deinit();
var cfg = try Config.load(arena.allocator());
defer cfg.deinit();
```

Key config sections: `models.providers` (API keys/endpoints), `agents` (named agent configs), `channels` (per-channel settings), `memory` (backend/search/lifecycle), `gateway` (port/host/pairing), `security` (sandbox/audit/autonomy), `autonomy` (level/limits/allowlists), `runtime` (native/docker/wasm).

## Zig 0.15.2 API Gotchas

- `std.io.getStdOut()` does NOT exist. Use `std.fs.File.stdout()`.
- HTTP client: `std.http.Client.fetch()` with `std.Io.Writer.Allocating`.
- Child processes: `std.process.Child.init(argv, allocator)`, `.Pipe` (capitalized).
- `ArrayListUnmanaged`: init with `.empty`, pass allocator to every method.
- `ChaCha20Poly1305.decrypt`: use stack buffer then `allocator.dupe()` (heap buffer segfaults on macOS).
- `SQLITE_TRANSIENT` in auto-translated C code: use `SQLITE_STATIC` (null) instead.
- When unsure about API, search `src/` for existing usage rather than guessing.

## Testing Conventions

- All tests use `std.testing.allocator` (leak-detecting GPA). Every allocation must be freed with `defer`.
- Use `builtin.is_test` guards to skip side effects (spawning processes, opening browsers, real hardware I/O). Return mock data instead (e.g., `return "test-refreshed-token"`).
- Tests must be deterministic and reproducible across macOS and Linux.
- Vendored SQLite hashes are validated at build time.
- Use `std.testing.tmpDir(.{})` with `defer tmp.cleanup()` for file-based test fixtures.
- Contract tests in `src/memory/engines/contract_test.zig` verify all memory backends satisfy the same vtable invariants. Follow this pattern when adding new backends.
- Test helpers (e.g., `TestHelper` structs with `dummyConfig()` / `initTestChannel()`) are defined within each module. Prefer this pattern over shared test utilities.
- Test naming: `subject_expected_behavior` (e.g., `"sendUrl constructs correct URL"`).

## Versioning

CalVer format: `YYYY.M.D` (e.g., `v2026.2.26`). Defined in `build.zig.zon`.

## CI

Tests run on Ubuntu (x86_64), macOS (aarch64), and Windows (x86_64). Release builds target 7 platforms including linux-riscv64. Docker images published to ghcr.io (linux/amd64, linux/arm64).

## Docker

Multi-stage build: Alpine builder with Zig, then minimal Alpine runtime. Runs as non-root (uid 65534) by default. Use `--target release-root` for root access.

```bash
docker-compose --profile gateway up   # HTTP gateway daemon
docker-compose --profile agent up     # interactive agent
```

## Nix

`flake.nix` provides a dev shell with Zig and ZLS. Activate with `direnv allow` (uses `.envrc`).

## License

MIT License.

---

# Fork Ownership — `canerelci/nullclaw` (for Pryva)

> Everything above this line is **upstream** (`nullclaw/nullclaw`) project guidance, inherited by
> this fork and kept essentially intact. Everything below is **fork-specific** ownership for the
> Pryva product. Treat them as two layers: upstream engineering protocol (above) + Pryva fork
> discipline (below). When they conflict on a Pryva concern, the fork layer wins.

## What this repo is

- This is **`canerelci/nullclaw`** (origin), a fork of **`nullclaw/nullclaw`** (upstream). Working
  directory: `~/dev/opensource/nullclaw`, branch `main`.
- It is the **ONLY place NullClaw (NCW) is modified for Pryva** — never clone or edit NCW anywhere
  else. The Pryva tenant image builds NCW **from this fork** (see `NULLCLAW_SHA` pin rule below).
- Pryva uses NCW as a lightweight "doer" agent: the OpenClaw SOUL is the **thinker** (decides what),
  an NCW specialist agent is the **doer** (calls MCP tools, returns strict JSON). This fork carries
  the commits that enable that split (`--tools`/`--list-tools` gating, `NULLCLAW_TRACE_FILE`).

## Sibling fork: OCW (OpenClaw)

Pryva consumes **two** `canerelci` forks in the same tenant image — this one (NCW, the doer) and
**OpenClaw / OCW** (`canerelci/openclaw`, the TypeScript orchestrator/pipeline + SOUL thinker). They
have **independent** image pins (`NULLCLAW_SHA` / `OPENCLAW_SHA`) and build **separately** (law 1).
OCW **owns the flow** (mints ids at `message_received`/`before_agent_start`, threads
`PRYVA_FLOW_ID`/`X-Flow-Id`); NCW is **backend-driven, NOT an OCW subagent** — its completion
re-enters the parent flow via the gateway `sessions.send` + `pryvaFlowId` seam (`ncw_completion`),
which is exactly why `NULLCLAW_TRACE_FILE` unifies NCW internals into the OCW flow trace. OCW can also
abort running NCW mid-flight (`/pipeline/steer-check`). Full relationship:
`.claude/memory/ocw-relationship.md`.

## ⛔ HARD RULE — the `NULLCLAW_SHA` pin (owner-mandated)

The Pryva assistant image (`~/dev/source/midmen` → `infra/docker-user/Dockerfile`) builds NCW pinned
at `ARG NULLCLAW_SHA=<commit>` (with `ARG NULLCLAW_REF=main`). The Dockerfile `git clone`s this fork
and **asserts the cloned commit equals the pin — it FAILS the build if the fork moved without a bump**.
Therefore: **a fork change is NOT in any tenant until `NULLCLAW_SHA` is bumped to the new fork HEAD
AND the image is rebuilt.** A local checkout / working tree is NOT the built image.

- After committing+pushing a fork change meant for tenants: state the new fork HEAD SHA and that
  midmen's `NULLCLAW_SHA` must be bumped to it before rebuild.
- Before claiming a fork change is "live" in a tenant: verify the built image (`nullclaw --version`
  inside the container), not the working tree.
- This side and the midmen agent both own this check — don't assume the other did it.

## The three laws (NCW build discipline)

1. **OCW and NCW builds are handled separately.** When this fork changes, produce a fresh NCW build
   of both archs and retire the old one. Never silently ship a stale NCW build alongside an OCW bump.
2. **The Pryva image MUST consume this prepared fork build** — reproducible Dockerfile build from
   `github.com/canerelci/nullclaw`, never a hand-baked or checked-in binary as the shipped artifact.
3. **When the image is built, DOUBLE-CHECK it was built from the correct sources** — verify, never
   assume. (Owner: *"Bir kez bile atlamanı istemiyorum."*)

Full checklist (both-arch rebuild, `nullclaw-bin` copy-back, `NULLCLAW_SHA` bump, image verify gate,
upstream rebase recipe): **`.claude/rules/fork-build.md`**.

## Validation (required before any fork commit)

```bash
zig version                       # MUST be exactly 0.15.2
zig build test --summary all      # all tests pass, 0 leaks (this is the gate)
zig fmt --check src/              # formatting (pre-commit hook enforces this)
zig build -Doptimize=ReleaseSmall # release build compiles clean (both archs for releases)
```

No stubs/placeholders/TODOs — deliver full working code, verify live before reporting done. Match the
vtable-driven architecture and naming in `AGENTS.md`. Read `AGENTS.md` before any code change.

## Do-not

- Do **not** push to upstream `nullclaw/nullclaw` — our changes live only in `canerelci/nullclaw`.
- Do **not** modify NCW anywhere but this repo.
- Do **not** hand-bake a binary into the Pryva image as the shipped artifact.
- Do **not** edit the upstream content above the banner — keep the fork layer separate.

## Where the detail lives

- `.claude/rules/fork-build.md` — NCW build/pin/rebase discipline + checklists.
- `.claude/skills/` — `ncw-fork-build` (ship a change), `ncw-rebase-upstream` (take an upstream
  release), `pryva-specialist-fork` (what the Pryva commits wired + thinker/doer split).
- `.claude/memory/MEMORY.md` — index of fork state (canonical path, commits ahead, pin, how Pryva
  invokes, validation).
- The Pryva/midmen side: `~/dev/source/midmen/.claude/skills/nullclaw/` + `ncw-agent/` + memory
  `ocw_ncw_fork_is_canonical` (the cross-repo source of truth for the fork-canonical incident).
