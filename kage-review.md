# kage: a comparative review against pi, opencode and kimi-code

**Subject:** [QaidVoid/kage](https://github.com/QaidVoid/kage) @ `d0f535a`
**Compared with:** [earendil-works/pi](https://github.com/earendil-works/pi), [anomalyco/opencode](https://github.com/anomalyco/opencode), [moonshotai/kimi-code](https://github.com/moonshotai/kimi-code)
**Reviewed:** 2026-09-26
**Scope:** CLI / agent core only. GUI and desktop shells are intentionally out of scope.

> Method: all four repositories were cloned and read from source and shipped docs. No build was run (no Rust toolchain available in the review environment); every claim below is traceable to a file in the tree.

---

## Contents

1. [Snapshot](#1-snapshot)
2. [Architecture](#2-architecture)
3. [Feature matrix](#3-feature-matrix)
4. [Where kage is better](#4-where-kage-is-better)
5. [Where kage is good / at parity](#5-where-kage-is-good--at-parity)
6. [Where kage is bad / weaker](#6-where-kage-is-bad--weaker)
7. [Where kage is worst](#7-where-kage-is-worst)
8. [Issues to address](#8-issues-to-address)
9. [Verdict](#9-verdict)

---

## 1. Snapshot

| | kage | pi | opencode | kimi-code |
|---|---|---|---|---|
| Language | Rust 2024, 11 crates + xtask | TypeScript | TypeScript | TypeScript |
| Source size | 145,557 lines | 379,654 lines | 683,229 lines | 803,374 lines |
| Created | 2026-05-09 | 2025-08-09 | 2025-04-30 | 2026-05-22 |
| Stars / forks | 0 / 0 | 109,381 / 13,900 | 210,081 / 27,775 | 7,670 / 1,256 |
| Releases | none | v0.87.1 | v1.18.32 | 2.1.1 |
| Install | build from source | npm | many package managers + curl | curl single binary |
| Contributors (recent window) | 1 | 19 | bots + ~10 | ~8 |
| Tests (approx) | 2,511 `#[test]` | ~7,000 | ~3,700 | ~1,100 |
| License | MIT | MIT | MIT | MIT |

kage is the smallest, newest and least adopted of the four, but it is a complete product rather than a prototype. It ships an agent loop, a TUI, ACP in both directions, an MCP client and server, Lua plugins with a capability model, JSONL sessions, subagents, compaction, print/JSON modes, documentation, a man page and five CI jobs.

---

## 2. Architecture

**kage.** Strict downward crate layering, documented in `docs/reference/architecture.md`. Fully synchronous: no tokio, no async traits. The agent loop is a pure event emitter (`LoopEvent`) and every frontend (TUI, `kage rpc`, `-p`) drives it through the same event-out / command-in channel pair. This is the cleanest agent-core boundary of the four.

**pi.** Close behind: separate `agent`, `ai`, `tui` and `coding-agent` packages, but `coding-agent` is large and carries more responsibility than kage's equivalent.

**opencode.** An Effect-based service graph with a client/server split. Powerful and highly testable, but the highest conceptual load of the four.

**kimi-code.** A many-package monorepo (`agent-core-v2`, `kaos`, `kap-server`, `kosong`, `minidb`, a deliberately forked `pi-tui`). Clear cut lines, maintained with a disciplined upstream-sync process documented in `packages/pi-tui/UPSTREAM.md`.

---

## 3. Feature matrix

| Feature | kage | pi | opencode | kimi-code |
|---|---|---|---|---|
| Extensibility | Lua, sandboxed, capability-gated | TS extensions, full trust | TS/JS plugins, full trust | TS plugins + marketplace, forks pi-tui |
| Permission rules | allow / ask / deny, globs | none (docs say containerize) | allow / ask / deny, wildcards, `--auto` | modes: manual / ask-when-needed / never-ask |
| Default for exec | **allow** | n/a, all allowed | ask | ask |
| OS sandbox | none (documented future) | none (docs recommend container) | none | none |
| Project trust | yes, pinned to exact risky values | yes | yes | yes |
| Subagents | yes (`agent` tool, depth cap, cancel tree) | via extensions | yes (task + subagent permissions) | yes + AgentSwarm |
| Parallel tools | opt-in flag | n/a | yes | yes |
| MCP | client + server + resources + prompts + OAuth | no | yes | yes |
| ACP | agent + client (stacks another agent as a provider) | no | yes | yes (acp-server) |
| LSP / formatters | no | no | yes | no |
| Image input | yes (paste, drag, `:attach`, clipboard) | yes | yes | yes + video |
| Todo / plan mode | no | no | yes | yes |
| Background tasks | no | no | yes | yes |
| Session store | JSONL append-only, fsync + flock, fork/clone/compact/search | JSONL, tree/fork | DB | minidb |
| Print / JSON / RPC | yes / yes / ACP | yes / yes / rpc | yes / yes / server + SDK | yes / yes / server API |

---

## 4. Where kage is better

- **Smallest safe-to-read core.** 145k lines with a documented crate graph. opencode is 4.7x larger with an Effect runtime and a server. For a project whose stated goal is "small enough to read", kage delivers.

- **A real plugin sandbox.** Lua with `os.execute`, `io`, `load`, `dofile`, `debug`, `require` and `package` stripped, an instruction watchdog, per-plugin `_ENV`, and coarse named capabilities (`session_write`, `exec`, `env`, `net`, `crypto`) that must be granted in config *and* requested by the plugin at load (`crates/kage-plugin/src/capabilities.rs`). pi, opencode and kimi-code all run extensions in-process with full user privileges. kage is the only one that attempts a boundary at all.

- **Lowest memory and startup floor.** A single Rust binary, a synchronous loop, no Node or Bun runtime and no JIT warmup. kimi-code advertises "milliseconds" but still pays for a JS runtime; kage has no runtime at all.

- **ACP in both directions.** `kage rpc` serves editors, and `AcpProvider` drives another ACP agent as a provider so kage's own loop can stack on top of `claude-code-acp`, `goose`, `gemini` or another kage (`crates/kage-acp/src/client.rs`). It advertises no `fs`/`terminal` capabilities upstream and never auto-approves the upstream agent's tools. No other tool in this set does this.

- **Credential and filesystem hygiene.** `atomic_write_private` (0600), mode preservation on atomic replace, `Debug` redaction of every secret, `flock` plus torn-tail repair on session files, opt-in `confine_paths`, and an SSRF guard that re-checks every DNS hop (`ssrf::guarded_agent`) rather than only the initial URL. pi and opencode are good here too; kage is consistently deliberate across the whole tree.

- **Config-driven customization.** `init.lua` exposes keymaps, autocmds, prompt slots, themes, autocompletion, roughly 25 events, and plugin state persistence without shipping a TypeScript package. Lower friction than pi/opencode extensions for the common case.

- **Trust pinned to values, not a boolean.** Project trust stores the exact risky tables and agent files that were approved, so any later edit invalidates it and re-prompts (`crates/kage-core/src/trust.rs`). Stronger than a plain trust flag.

- **Test density.** 2,511 tests across 145k lines, including behavioural tests for permission precedence, trust invalidation, SSRF, symlink escape and torn-tail repair. The highest tests-per-line of the four.

---

## 5. Where kage is good / at parity

Provider breadth (models.dev-backed catalog; OpenAI, Responses, Anthropic, Gemini, DeepSeek, Z.ai, OpenRouter and custom endpoints), streaming with typed stop reasons, transient retry honouring the provider's `retry_after`, doom-loop steering, compaction with hook-overridable summaries, fork/clone/resume/search, skills plus `AGENTS.md`, MCP resources/prompts/OAuth, print and JSON event modes, themes, keymaps, vim mode.

Documentation is unusually complete for a 0.1 release: 21 markdown pages, editor guides for Zed and Neovim, and an architecture reference that matches the code.

---

## 6. Where kage is bad / weaker

- **No releases, no binaries.** Build-from-source only, one contributor, zero users. Every competitor installs in one command. This is the biggest practical gap and it is not technical.

- **Feature surface lags the mature three.** No LSP diagnostics, no formatters, no git or checkpoint integration, no todo or plan mode, no background task manager, no web search tool, no session sharing and no HTTP server/SDK. kage does the core agent loop very well and then stops.

- **Coarser permission model than opencode.** Per-tool globs matched against a single string; no structured per-path or per-argument rules and no inverse `--auto` mode. kimi-code's mode system (manual / ask-when-needed / never-ask / plan) is more usable.

- **Lua is a smaller ecosystem than TypeScript.** Sandboxing is a real win, but pi/opencode/kimi plugins can `import` npm. kage plugins cannot use libraries beyond the embedded stdlib, cannot `require` and cannot spawn threads.

- **No async.** A virtue for readability, a cost for high-concurrency subagent fan-out. kimi-code's swarm ramps to unbounded concurrency; kage's agent count is a configured cap.

---

## 7. Where kage is worst

- **Default security posture.** `PermissionAction::Allow` is the default and an unconfigured tool is allowed (`PermissionsConfig::check` returns `Allow` for a missing entry in `crates/kage-core/src/permissions.rs`). `bash` is registered and allowed. Out of the box, a prompt injection in a fetched page, a README or a dependency's source is arbitrary unsandboxed code execution with zero prompts. pi has the same exposure but documents it loudly and tells you to containerize; opencode and kimi-code ask by default. kage is the only one of the four that has a permission system and ships it effectively off.

- **Provider credentials reach the shell.** `BashTool` inherits the full parent environment, and `BashConfig::scrub_env` defaults to empty; `kage init` even writes `scrub_env = []` with the useful list commented out (`crates/kage-cli/src/init.rs`). A model or an injection can run `env` and read `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` when those come from the environment. `scrub_env` also cannot be a boundary on Linux: the child runs as the same UID and can read `/proc/<ppid>/environ`.

- **No read-before-write or external-change detection.** `edit` and `write` read and replace the file with no record that the model ever read it and no mtime or content check (`crates/kage-tools/src/builtin/edit.rs`). kimi-code explicitly rejects writes and edits when the file was not read first or changed on disk; opencode keeps snapshots. kage will silently clobber a concurrent edit.

- **Bus factor.** One author, no issues, no releases. If this is meant to be used by anyone other than its author, that is the risk to solve first.

---

## 8. Issues to address

Ordered by how much they matter.

### HIGH

1. **Default to `ask` for consequential tools.**
   Flip the fallback for `Risk::Exec` and `Risk::Write` (or gate the first run behind an explicit consent step) and make `[permissions]` opt-*out* rather than opt-in.
   *Files:* `crates/kage-core/src/permissions.rs` (`PermissionAction` default, `check`).

2. **Do not hand provider secrets to `bash`.**
   Ship a default scrub list (`*_API_KEY`, `*_TOKEN`, `*_SECRET`, `*_PASSWORD`) instead of `scrub_env = []` in `kage init`, and reword the `scrub_env` doc comment: it hides names from the child environment, it does not stop same-UID `/proc/<ppid>/environ` reads.
   *Files:* `crates/kage-cli/src/init.rs`, `crates/kage-tools/src/builtin/bash.rs`, `crates/kage-core/src/config.rs`.

3. **Add a sandbox backend, or say plainly that there is none.**
   `bash.rs` says an OS-level isolation backend "can land later". Until then the README should carry the same containerization warning pi's `security.md` does. Landlock/seccomp or a `bwrap` wrapper would turn the permission system into an actual boundary rather than a speed bump.

### MEDIUM

4. **Read-before-write tracking for `edit` and `write`.**
   Keep a per-session map of path to `(mtime, size, hash)` populated by `read`; reject an edit if the file changed since, and reject a write over an unread existing file.
   *Files:* `crates/kage-tools/src/builtin/{read,write,edit}.rs`.

5. **Widen the SSRF denylist.**
   `is_unsafe` misses IPv4 `0.0.0.0/8` (only the unspecified address), `100.64.0.0/10` CGNAT, `198.18.0.0/15` benchmarking, `192.88.99.0/24` and `240.0.0.0/4`; IPv6 misses `2002::/16` (6to4), `2001::/32` (Teredo) and `64:ff9b::/96` (NAT64), all of which can encode or reach an internal IPv4.
   *File:* `crates/kage-tools/src/ssrf.rs`.

6. **Close the `resolve_under` tail gap under `confine_paths`.**
   A symlink planted in a not-yet-existing tail is invisible to the check, and there is a TOCTOU window between resolution and `atomic_write`. Re-canonicalize the final path after opening, or resolve per component with `openat`/`O_NOFOLLOW`.
   *Files:* `crates/kage-tools/src/path.rs`, `crates/kage-core/src/fsutil.rs`.

7. **Bound blocking plugin host calls.**
   The watchdog counts VM instructions only, so a plugin with the `net` capability can loop on `kage.http` indefinitely, each call internally timed out. Add a per-entry wall-clock deadline alongside the instruction budget.
   *File:* `crates/kage-plugin/src/watchdog.rs`.

### LOW

8. **Add jitter to provider retry backoff.** `1u64 << (attempt - 1).min(5)` is deterministic, so concurrent instances retry in lockstep on a 429.
   *File:* `crates/kage-loop/src/run.rs` (`retry_backoff`).

9. **Consider a configurable loop iteration cap.** The config documents "no iteration cap" as intentional; with a default-allow toolset it is also an unbounded spend. A high ceiling with a clear message would not affect normal use.

10. **Ship releases.** Tagged binaries for Linux and macOS (and WSL), plus a `cargo install`-able path. Nothing else on this list gets users; this does.

---

## 9. Verdict

| Dimension | Winner |
|---|---|
| Engineering quality per line | **kage** |
| Overall product today | opencode, then pi |
| Distribution and UX | kimi-code |
| Security defaults | opencode / kimi-code |
| Plugin isolation | **kage** (only entrant) |
| Ecosystem and momentum | opencode |

- **Best engineered per line: kage.** The crate layering, the loop/frontend separation, the plugin capability model and the credential/atomic-write hygiene are better than anything comparable in the other three. If you read one of these to learn how to structure an agent, read kage.

- **Best overall product today: opencode, then pi.** Feature depth, ecosystem, install story and momentum are not close.

- **Best distribution/UX bet: kimi-code**, for the single-binary install and the swarm/background/plan surface.

- **kage's gap is not code quality.** It is the last mile: defaults that assume a trusted single expert user, no releases, and a feature set that stops at the core loop. Issues 1, 2 and 10 decide whether it stays a well-written personal tool or becomes something other people run.

---

<sub>Report generated for QaidVoid. GUI and desktop shells excluded by request.</sub>
