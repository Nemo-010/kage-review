# kage vs pi vs opencode vs kimi-code

kage is the best-engineered of the four and the least productized. Its Rust core, layered crates, Lua capability sandbox, hardened HTTP path, plain JSONL sessions, and clean fmt/clippy/test gates beat the TypeScript trio on construction. It matches them on providers, MCP, ACP, skills, subagents and compaction, and it is the only one of the four that can act as an ACP *client* as well as a server. It trails on everything that spreads a tool: source-only install with no releases, no web search, no LSP, no background or scheduled tasks, no todo or plan tools, no provider OAuth, no Windows build and no plugin marketplace. Default-allow permissions plus unsandboxed `bash` make the shipped posture much weaker than the code underneath it. Good choice for a hackable single binary with real extension isolation. Not a daily driver until distribution and the default posture are fixed.

**Scope.** CLI, engine, tools, providers, sessions, permissions, MCP, ACP, extensibility, config, docs, tests, distribution. GUI and desktop shells are excluded by request.

---

## Contents

- [Snapshot](#snapshot)
- [Architecture](#architecture)
- [Standing per dimension](#standing-per-dimension)
- [Builtin tools](#builtin-tools)
- [Providers and auth](#providers-and-auth)
- [Sessions, permissions, protocols](#sessions-permissions-protocols)
- [Extensibility](#extensibility)
- [Issues in priority order](#issues-in-priority-order)
- [Verdict](#verdict)
- [Method and limits](#method-and-limits)

---

## Snapshot

| | kage | pi | opencode | kimi-code |
| --- | --- | --- | --- | --- |
| Repo | QaidVoid/kage | earendil-works/pi | anomalyco/opencode | moonshotai/kimi-code |
| Pin reviewed | `d0f535a` | `d6af72e` | `696f41b` | `be7d5f5` |
| Created | 2026-05-09 | 2025-08-09 | 2025-04-30 | 2026-05-22 |
| Stars (2026-09-26) | 0 | 109,381 | 210,081 | 7,670 |
| Forks | 0 | 13,900 | 27,775 | 1,256 |
| Language | Rust 2024, Lua vendored | TypeScript monorepo | TypeScript, Go bits | TypeScript (pnpm) |
| Source size | 145,557 lines | 379,654 lines | 683,229 lines | 803,374 lines |
| Install | source only, Rust 1.87 + C compiler | npm + standalone binaries | script, npm, brew, scoop, choco, pacman, mise | script, npm, no Node needed |
| Releases | none | yes | yes | yes |
| Platforms | Linux, macOS, WSL | Linux, macOS, Windows | Linux, macOS, Windows | Linux, macOS, Windows |
| Tests (approx) | 2,511 `#[test]` | ~7,000 | ~3,700 | ~1,100 |
| License | MIT | MIT | MIT | MIT |
| Status | pre-1.0, single maintainer | community, auto-close triage | large community | vendor led (Moonshot AI) |

---

## Architecture

kage splits into 11 crates with strict downward layering, documented in `docs/reference/architecture.md`. The loop is fully synchronous: no tokio, no async traits, no `Pin<Box<dyn Stream>>`. Frontends (the TUI, `kage rpc` for editors, and `-p` print mode) all drive one dispatcher through the same event-out / command-in channel pair, with every event carrying a session id and a per-session sequence number. That is the cleanest agent-core boundary of the four.

pi is close behind: separate `agent`, `ai`, `tui` and `coding-agent` packages, though `coding-agent` carries more weight than kage's equivalent. opencode is an Effect-based service graph with a client/server split: powerful and testable, highest conceptual load. kimi-code is a many-package monorepo (`agent-core-v2`, `kaos`, `kap-server`, `kosong`, `minidb`, a deliberately forked `pi-tui`) with clear cut lines and a disciplined upstream-sync process.

---

## Standing per dimension

Legend. **Better** = kage ahead of every rival named in that row. **Good** = kage level with the named rivals. **Bad** = behind at least one named rival. **Worse** = behind with structural weight, not a small gap. Where a cell says unverified, that rival's position was not confirmed, so no claim is made against them there.

| Dimension | Verdict | Against whom | Note |
| --- | --- | --- | --- |
| Construction quality | **Better** | pi, opencode, kimi-code | Rust, `unsafe_code = "forbid"`, pedantic clippy, layered crates, 2,511 tests |
| Extension isolation | **Better** | pi, opencode, kimi-code | per-plugin Lua `_ENV` + capability jail + instruction watchdog; the others run extensions in-process with full trust |
| Session format | **Better** | opencode, kimi-code; level with pi | append-only JSONL, `cat`/`rg` friendly, fsync + flock, fork/clone/resume/search |
| Config trust | **Better** | pi, opencode, kimi-code | project config ignored until `kage trust`, pinned to exact risky values, edits re-ask |
| ACP | **Better** | opencode, kimi-code; ahead of pi | only one with both directions: serves editors and drives another ACP agent as a provider |
| Fetch hardening | **Better** | pi, opencode, kimi-code on depth | SSRF checks re-run on every DNS answer; gaps remain (see issues) |
| One engine, multiple frontends | **Good** | level with opencode, pi; plus over kimi-code | TUI, ACP rpc and `-p` share one dispatcher and sequenced events |
| Providers | **Good** | level with pi, opencode, kimi-code | 20 catalog ids; 4 native client impls, rest OpenAI-compatible |
| MCP | **Good** | level with opencode, kimi-code | tools, resources, prompts, OAuth; pi has none |
| Agents / subagents | **Good** | level with pi, opencode, kimi-code | `agent` tool, depth cap, cancel tree, one reply channel per agent |
| Compaction | **Good** | level with pi, opencode, kimi-code | 0.8 threshold, hook-overridable summary |
| Permissions model | **Good** | level with opencode, kimi-code; pi differs by design | allow/ask/deny, deny-first globs, per-server MCP actions, session approvals |
| CI hygiene | **Good** | level on basics; ahead on two gates | fmt, clippy `-D warnings`, os matrix everywhere; Lua-type drift + ASCII gates are kage-only |
| Streaming / retry | **Good** | level with all three | typed stop reasons, transient retry honouring `retry_after`, doom-loop steering |
| Provider login UX | **Bad** | pi, opencode, kimi-code | API keys only; OAuth record shape exists but no provider flow |
| Code intelligence | **Bad** | opencode; pi and kimi-code unverified | no LSP, no symbols, no diagnostics tool |
| Web search | **Bad** | opencode, kimi-code | fetch only, no search tool |
| Background + scheduled work | **Bad** | opencode, kimi-code | no background tasks, no cron |
| Plan tracking | **Bad** | opencode, kimi-code | no todo, no plan mode, no question tool |
| Distribution | **Bad** | pi, opencode, kimi-code | no binaries, no install script, no package managers |
| Windows | **Bad** | pi, opencode, kimi-code | not supported; README says Linux, macOS, WSL |
| Plugin discovery | **Bad** | kimi-code, opencode, pi | 7 example plugins, no registry, no index, no marketplace |
| Docs toolchain | **Bad** | pi, opencode, kimi-code | docs preview requires bun + vitepress despite a non-Node product |
| Security defaults | **Worse** | opencode, kimi-code; pi differs by design | default-allow builtins incl. `bash`, `confine_paths` off, `scrub_env` empty |
| Sandbox story | **Worse** | pi, opencode, kimi-code | `[sandbox]` section removed in `e02897c` with nothing behind it; pi documents container patterns, opencode ships `packages/containers` |
| Maturity risk | **Worse** | pi, opencode, kimi-code | pre-1.0, one maintainer, 0 stars, 0 issues, no releases |

---

## Builtin tools

| Tool | kage | pi | opencode | kimi-code |
| --- | --- | --- | --- | --- |
| read | yes | yes | yes | yes + media |
| write | yes | yes | yes | yes + append mode |
| edit + diff | yes | yes | yes + apply_patch | yes |
| bash / shell | yes, 120 s default, 100 KB cap, opt-in env scrub | yes + powershell | yes | yes + background |
| grep | yes | yes | yes | yes |
| find / glob | yes | yes | yes (glob) | yes |
| ls | yes | yes | n/a (via glob/bash) | n/a |
| web fetch | yes, SSRF hardened | no | yes | yes |
| web search | no | no | yes | via host injection |
| LSP / symbols / diagnostics | no | no | yes | no |
| todo tracking | no | no | yes | yes |
| plan mode | no | no | yes | yes |
| question prompt | no | no | yes | yes |
| background tasks | no | no | yes | yes |
| scheduled (cron) tasks | no | no | no | yes |
| subagents | yes | via extensions | yes | yes + swarm |
| skill loading | yes (SKILL.md) | yes (Agent Skills spec) | yes | yes |
| image input | yes (paste, drag, `:attach`) | yes | yes | yes + video |

kage's whole registry is `crates/kage-tools/src/builtin/mod.rs`: read, write, edit, bash, grep, find, ls, web_fetch, plus the per-run `agent` tool. That is the complete builtin surface.

---

## Providers and auth

| | kage | pi | opencode | kimi-code |
| --- | --- | --- | --- | --- |
| Provider ids | 20 catalog ids | 46 provider files, widest | models.dev driven | Moonshot first + compatibles |
| Native client impls | anthropic, openai, openai-responses, gemini | per-provider files | via SDK | via SDK |
| Model catalog refresh | yes, `kage models refresh` | yes | yes | yes |
| Regional plan split | yes (ZAI global/CN, Xiaomi 4 regions) | yes | varies | varies |
| API key auth | yes, env or `auth login` | yes | yes | yes |
| Provider OAuth login | no | yes | yes | yes (device flow) |
| Thinking-level mapping | per family | varies | varies | varies |
| Credential file mode | `0600` `auth.json` | varies | varies | varies |

kage's provider auth is API-key only: `kage auth login` prompts for a key (`crates/kage-cli/src/auth.rs`, `run_login`). The store and the consumer both understand OAuth records (`Credential::Oauth`, `AuthStore::access_token`), and MCP server tokens use that shape in a separate `mcp-auth.json`, but no provider OAuth login flow is wired.

---

## Sessions, permissions, protocols

| | kage | pi | opencode | kimi-code |
| --- | --- | --- | --- | --- |
| Session storage | JSONL, header first line | JSONL + sqlite backend | sqlite / Effect stores | backends + minidb |
| resume / clone / fork / search | yes | yes | yes | yes |
| Permission actions | allow / ask / deny + session approvals | no builtin gate, container patterns | rules + wildcards | rules + approval dialogs |
| Builtin default | **allow** | n/a (all allowed) | ask-leaning | ask-leaning |
| Unknown MCP tool default | ask | n/a, no MCP | varies | varies |
| Path confinement | opt-in, default off | via container | varies | varies |
| MCP tools/resources/prompts/OAuth | yes | no MCP support | yes | yes |
| ACP server (editors drive it) | yes | via RPC, not ACP | yes | yes |
| ACP client (drives other agents) | **yes, unique** | no | no | no |
| Print `-p` + `--json` event stream | yes | yes | yes | yes |
| `doctor` diagnostics | yes | varies | varies | varies |

---

## Extensibility

| | kage | pi | opencode | kimi-code |
| --- | --- | --- | --- | --- |
| Language | Lua 5.4 (vendored) | TypeScript | TypeScript | TS plugins + skills |
| Isolation | per-plugin `_ENV` + capability grants | full host permissions | full host permissions | trust level shown, no jail |
| Capabilities | `session_write`, `exec`, `env`, `net`, `crypto` | none | none | none |
| Hooks / autocmds | ~25 events, patterns + groups | session/agent hooks | plugin hooks | lifecycle external hooks |
| Skills (`SKILL.md`) | yes, user + project + `.agents` | yes (Agent Skills spec) | yes | yes |
| Marketplace / registry | no | docs + packages | docs + discovery | yes, versioned index |
| Slash commands | yes | yes | yes | yes |
| Typed authoring stubs | yes, `kage.lua` drift-checked in CI | yes (TS types) | yes | yes |

---

## Issues in priority order

| # | Sev | Issue | Evidence | Fix |
| --- | --- | --- | --- | --- |
| 1 | High | Default-allow permissions, including `bash` | `PermissionAction::Allow` is the serde default and `PermissionsConfig::check` returns `Allow` for a missing entry | flip the fallback for `Risk::Exec` / `Risk::Write`, or gate first run behind an explicit consent |
| 2 | High | Provider API keys inherit into `bash` children | `scrub_env` defaults empty; `kage init` writes `scrub_env = []` with the useful list commented out | ship a default scrub list (`*_API_KEY`, `*_TOKEN`, `*_SECRET`, `*_PASSWORD`) |
| 3 | High | No OS sandbox behind the permission gate | `bash.rs` says a backend "can land later"; `[sandbox]` removed in `e02897c` | add Landlock/seccomp or `bwrap`, or document containerization as pi does |
| 4 | High | No releases or binaries | README is `cargo build` + symlink; `gh release list` empty | versioned archives + checksums + install script, then package managers |
| 5 | Med | No read-before-write or whole-file change detection | `write` has only an `overwrite` flag; `edit` re-reads and exact-matches; `read` returns `structured: None`, no hash or mtime | per-session path -> (mtime, size, hash) map set by `read`, checked by `write`/`edit` |
| 6 | Med | Concurrent mutations are not serialized | only `bash` returns `ExecMode::Sequential`; `write`/`edit` do not | mark both sequential or add a per-path mutation queue (pi has `file-mutation-queue.ts`) |
| 7 | Med | SSRF denylist has holes | `is_unsafe` misses `0.0.0.0/8`, `100.64/10`, `198.18/15`, `192.88.99/24`, `240/4`; v6 misses `2002::/16`, `2001::/32`, `64:ff9b::/96` | extend the ranges or use a vetted crate |
| 8 | Med | `resolve_under` blind to symlinks in the unresolved tail | documented; `confine_paths` on builtin tools does not re-verify after parent creation | re-canonicalize after open, or resolve per component with `openat`/`O_NOFOLLOW` |
| 9 | Med | Blocking plugin host calls are unbounded | watchdog counts VM instructions only; a `net` plugin can loop on `kage.http` | add a per-entry wall-clock deadline |
| 10 | Low | Retry backoff has no jitter | `1u64 << (attempt - 1).min(5)` in `retry_backoff` | add jitter so fleet-wide 429s do not resynchronize |
| 11 | Low | No provider OAuth login | `run_login` prompts for a key; `Credential::Oauth` is only fed by MCP auth | wire provider OAuth or state the rejection |
| 12 | Low | No loop iteration cap | `LoopConfig` documents no cap as intentional; with default-allow tools it is also unbounded spend | add a high configurable ceiling with a clear message |
| 13 | Low | Docs toolchain needs bun | `docs/package.json` + `bun.lock` for a non-Node product | plain static build or explicitly document the contributor requirement |
| 14 | Low | No telemetry stance | no telemetry code found in the tree | state "no telemetry" as a guarantee |
| 15 | Info | ACP subagents track a draft spec | `kage-cli/src/rpc/mod.rs` and `kage-acp/src/acp.rs` cite draft RFD PR #1992 | pin the draft version and track ratification |

### Detail on the two that matter most

**1. Default-allow.** `crates/kage-core/src/permissions.rs` sets `PermissionAction::Allow` as `#[default]`, and `check` returns `Allow` outright when a tool has no `[permissions.tools.<name>]` entry. `bash` is registered and allowed. Out of the box, a prompt injection in a fetched page, a dependency's README, or a tool result is arbitrary unsandboxed execution with zero prompts. pi has the same exposure but documents it loudly and tells you to containerize; opencode and kimi-code ask by default. kage is the only one of the four that ships a permission system and leaves it effectively off.

**5. No read-before-write.** `write` gates on `overwrite: true` only. `edit` re-reads the file, applies splices and atomically writes, with no record that the model ever read it, no mtime check and no content hash. Exact-match semantics are partial cover: a changed match region fails loudly, but a change anywhere else is invisible, and `read` records nothing that could catch it later. Kimi-code rejects writes and edits when the file was not read first or changed on disk; opencode keeps snapshots. kage clobbers.

---

## Verdict

| Dimension | Winner |
| --- | --- |
| Engineering quality per line | **kage** |
| Overall product today | opencode, then pi |
| Distribution and UX | kimi-code |
| Security defaults | opencode / kimi-code |
| Plugin isolation | **kage** (only entrant) |
| Ecosystem and momentum | opencode |
| Feature breadth | opencode |

- **Best engineered per line: kage.** Crate layering, loop/frontend separation, the plugin capability model, atomic-write hygiene and redacted credentials are better than anything comparable in the other three. If you read one of these to learn how to structure an agent, read kage.
- **Best overall product today: opencode, then pi.** Feature depth, ecosystem, install story and momentum are not close.
- **Best distribution/UX bet: kimi-code**, for the single-binary install and the swarm/background/plan surface.
- **kage's gap is the last mile, not code quality.** Defaults that assume a trusted single expert user, no releases, and a feature set that stops at the core loop. Issues 1 to 4 decide whether it stays a well-written personal tool or becomes something other people run.

---

## Method and limits

Read directly from source and shipped docs after cloning all four repositories. No build, no benchmarks, no model calls, no TUI interaction: no Rust toolchain was available in the review environment, so kage was not compiled and its test suite was not run. Every claim about kage is traceable to a file in the tree at `d0f535a`; every claim about a rival at the pin in the Snapshot table.

Files read for kage: README, `docs/guide`, `docs/plugins`, `docs/reference/architecture.md`, `crates/kage-tools/src/builtin/*`, `crates/kage-tools/src/{path,ssrf,tool,registry}.rs`, `crates/kage-core/src/{permissions,trust,fsutil,config,skills,agents}.rs`, `crates/kage-loop/src/{config,run,doom,dispatch}.rs`, `crates/kage-session/src/writer.rs`, `crates/kage-plugin/src/{capabilities,watchdog,host}.rs`, `crates/kage-cli/src/{auth,permissions,runtime_env,init}.rs`, `crates/kage-acp/src/{lib,client,acp}.rs`, `crates/kage-mcp/src/oauth.rs`, `crates/kage-cli/src/rpc/mod.rs`, the CI workflow, and `git log`.

Rivals were read at the same breadth: pi tools/extensions/docs/security/windows, opencode tool/auth/permission/agent/lsp/background/mcp/acp dirs, kimi-code tool contract/marketplace/hooks/skills/sessions/ACP.

**Known gap in this review:** the SSRF denylist analysis and the plugin watchdog analysis are source-only; neither was exercised against a live target. The `is_unsafe` holes are read from the match arms, and the watchdog claim from the `CREDITS_KEY` instruction budget. Treat those two as high-confidence but not proven.
