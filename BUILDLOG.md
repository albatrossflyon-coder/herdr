# BUILDLOG — herdr (this fork)

Fork of `herdrdev/herdr` at `albatrossflyon-coder/herdr`, created 2026-08-21. This file tracks *our* changes/investigation against the upstream project — herdr itself is third-party, not our code, but the fork is ours to develop against before upstreaming.

## What this is
Terminal workspace/multiplexer for coordinating multiple AI coding agents in panes, with a real CLI + socket API for cross-agent orchestration (`agent.prompt`, `agent.wait`, `pane.report_agent`). Local clone: `C:\Repos\herdr` (origin: upstream `ogulcancelik/herdr`, resolves to `herdrdev/herdr`; `fork` remote: `albatrossflyon-coder/herdr`).

## 2026-08-21 (TIC 2) — Goose agent support: built, tested, committed, pushed

Added real `Agent::Goose` support (Rust enum variant + `ALL`/`SCREEN_MANIFEST_AGENTS` membership + label/executable/alias lookup + sidebar/sound config + a `goose.toml` detection manifest using patterns captured live from a running Goose session — working state = screen containing `(Ctrl+C to interrupt)`, idle = `Enter to send`). Followed the minimal `Agent::Amp` template (screen-manifest only, no deeper lifecycle integration), since Goose has no session-restore equivalent yet.

**Verified:** `cargo build` clean, `cargo test` 3265/3266 passing (the 1 failure is a pre-existing flaky test — `workspace::tests::generated_workspace_ids_are_short_base32_handles`, random base32 ID length assertion, confirmed unrelated to this change by running it in isolation 3x with different random pass/fail outcomes).

**Committed and pushed, landing independently verified** (not just a clean exit code): commit `e113d03` on branch `feature/goose-agent-support`, pushed to the fork, remote SHA confirmed matching local via `gh api repos/albatrossflyon-coder/herdr/branches/feature/goose-agent-support`.

**Not yet done:** opening the actual PR against upstream `herdrdev/herdr` — held for Chris's explicit go-ahead since it's a public action against someone else's repo, not decided automatically.

**Real build gotchas hit, worth remembering for next time building this from source:**
- `/mnt/c/...` (Windows-mounted path) fails Zig's build cache with `AccessDenied` on rename — build on WSL2's native filesystem instead (`~/src/herdr-build`).
- Copying the repo Windows→WSL2 via `rsync` produces a huge false "everything modified" diff from CRLF/LF line-ending differences alone — check one file's real diff before panicking.
- The vendored `libghostty-vt` dependency needs Zig **≥0.15.2** specifically; Ubuntu's `apt` package (0.14.1) fails with a ZON-import syntax error. Downloaded the official `zig-x86_64-linux-0.15.2` tarball directly (no sudo) and pointed the build at it via the `ZIG` env var.
- `sudo apt-get install zig` hangs invisibly through both the Bash tool and a `!`-prefixed command — neither gives sudo a real controlling terminal. Has to be run in a genuinely separate terminal window by Chris directly.

## 2026-08-21 (TIC 2, later) — OpenCode/Kilo Bun crash: real root cause found, herdr-specific trigger still open

**Confirmed root cause** (via Pi's investigation): both OpenCode and Kilo Code bundle Bun, and Bun's JavaScriptCore JIT emits AVX512 instructions this CPU (Intel Meteor Lake, no AVX512) can't execute, once code tiers up (~2-3s into a session) — real upstream issue `oven-sh/bun#34215`. Crash is `SIGILL`, happens only when the TUI initializes (interactive mode), not in headless/one-shot mode.

**Real fix found and corrected mid-session:** the actual working env var is `BUN_JSC_useJIT=0` (a JavaScriptCore-specific option) — **not** `NODE_OPTIONS=--jitless`, which is a V8/Node.js flag that Pi's first report incorrectly cited as confirmed-working; it does not apply to Bun's JSC engine at all. Confirmed the real fix via the actual `oven-sh/bun#34215` comment thread, not guessed.

**Wrapper scripts installed** (`~/.opencode/bin/opencode` wrapping the renamed `opencode-real` binary, `~/.npm-global/bin/kilo` wrapping the real `@kilocode/cli` bin) that export `BUN_JSC_useJIT=0` before exec-ing the real binary. **Confirmed working standalone** in a plain bash shell — real 8+ second run, exit 0, no crash, well past the crash threshold.

**Root cause of the herdr-specific trigger, found by Pi + Hermes independently:** the wrapper scripts (`~/.opencode/bin/opencode`, `~/.npm-global/bin/kilo`) set `BUN_JSC_useJIT=0` *inside* the wrapper before exec-ing the real binary, so the var never appears in herdr's own pane environment — it's set one process-exec too late to matter for anything herdr itself might read/log, and more importantly this makes herdr's own env the more robust place to guarantee it, independent of which wrapper (if any) PATH resolves to. Both agents converged on the same fix: set `BUN_JSC_useJIT=0` directly in herdr's `apply_pane_launch_env` (`src/pane.rs`), unconditionally for every pane. This is a no-op for non-Bun agents and only affects Bun's own JIT-heavy paths, which barely matters for an interactive CLI.

**Fix applied and fully verified 2026-08-21 (TIC 2):**
- `src/pane.rs` — one line added to `apply_pane_launch_env`: `cmd.env("BUN_JSC_useJIT", "0")`, with a comment citing `oven-sh/bun#34215`.
- `vuln-hunter scan_diff` (deep_review) → **no findings**.
- `cargo test` → **3265/3266 passing** (same pre-existing flaky test as the Goose change above, confirmed unrelated).
- **Live crash-rate verification** (the real test, since the crash was intermittent at <10% before the fix): launched OpenCode through the actual `herdr agent start --kind opencode` path **11 times** and Kilo through `herdr agent start --kind kilo` **9 times**, all against the freshly rebuilt dev binary. **20/20 clean — 0 crashes**, with genuine rendered TUIs confirmed each time (OpenCode 1.18.21, Kilo 7.4.23), not silent failures.
- Chris's explicit precondition ("make sure they're working properly before you commit and push") is satisfied by this result.

**Real correction made mid-session, worth remembering:** started a `claude`-kind pane (`claude-fixer`) to help with this without first flagging that it requires real Anthropic credentials (OAuth or a paid API key) to function at all — unlike every other agent in this roster (Pi/Hermes/Goose all run on free/cheap CLIProxyAPI models). Chris caught this and correctly pointed out the same thing happened in the original 2026-08-19 pilot too (that session's own log shows a `claude-implementer` pane was started already-logged-in, without ever calling out that it consumed real usage). Going forward: always state upfront what account/credentials an agent-start actually runs on, before starting it, not after it hits an auth wall.

## Not yet done
- [ ] Local-only herdr Connector (an MCP server exposing herdr's socket API as tools, registered directly with Claude Desktop, no public exposure) — scoped and agreed (local-first, confirmed by Chris), but not started; got sidetracked into the OpenCode/Kilo crash fix work instead.
- [ ] The PR against upstream `herdrdev/herdr` for the Goose work — commit is ready and pushed to the fork, PR itself not opened.
- [x] OpenCode/Kilo herdr-PTY-specific crash trigger — fixed (`BUN_JSC_useJIT=0` in `apply_pane_launch_env`), scanned clean, tested, 20/20 live launches with no crash. Committed and pushed to `feature/goose-agent-support` on the fork. PR against upstream not yet opened.
- [ ] Qwen Code (`QwenLM/qwen-code`) — real herdr-native agent kind already exists (`Agent::Qwen`, no code change needed unlike Goose), cloned and security-scanned clean (vuln-hunter: no exploitable findings, safe to install). **Not yet installed** — flagged to install next session.
