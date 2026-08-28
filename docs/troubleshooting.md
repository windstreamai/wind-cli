# Troubleshooting wind

Symptom-first: find what you're seeing, read why, apply the fix. If nothing here matches, skip to [Still stuck?](#still-stuck).

## `windstream` missing from `claude mcp list`

Installing the binary doesn't connect it to Wind Stream — `wind setup` does, and Claude Code only picks up a newly-registered MCP after a restart. If `claude mcp list` doesn't show `windstream`, either `wind setup` hasn't run from this binary yet, or it has but Claude Code hasn't been restarted since.

Fix: run `wind setup`, then restart Claude Code. See [Install](install.md#2-connect-to-wind-stream) or [Install (Windows)](install-windows.md#2-connect-to-wind-stream) for what it does, and re-check with `claude mcp list`.

## `/windstream-connect` not in the `/` list

Same root causes as above, applied to the slash commands: `wind setup` deploys `windstream-*` commands to Claude Code, and a session that was already open won't see them until it restarts.

Fix: run `wind setup` — it's idempotent, safe to re-run any time — then restart Claude Code and type `/` again. See [wind setup](commands.md#wind-setup) for what it deploys.

## `windstream` or `$windstream-connect` missing in Codex

Codex setup is selected through the persisted agent preference. If setup ran while Claude was the default, it correctly configured Claude instead and left Codex alone.

Fix: run `wind agent codex`, then `wind setup`. Confirm `codex mcp list` shows an enabled `windstream` entry backed by `WINDSTREAM_TOKEN`, and invoke `$windstream-connect` in a fresh Codex session. Setup never writes the token into Codex configuration; `wind up` supplies it to the Codex child process.

## Codex exits with “did not report a trace spool”

Codex requires explicit trust for installed lifecycle hooks. Without it, the coding session still runs and connects, but SessionEnd does not produce the trace-ready notification the parent `wind` CLI is waiting for.

Fix: open `/hooks` in Codex, review and trust the Wind SessionStart/SessionEnd handlers, then resume or start the worktree again. If the hooks are trusted but the note persists, inspect `wind trace`; any successfully written trace remains safe under `~/.windstream/spools/traces/` for a later SessionStart retry.

## `wind -v` shows an unexpected version

If `wind -v` (or `wind --version`) doesn't print the version you just installed, another `wind` earlier on your `PATH` is shadowing the one you meant to run — the most common trap right after an update.

Fix: find every `wind` on `PATH` — `which wind` (macOS / Linux) or `where wind` (Windows) — and remove or reorder so the one you just installed wins. On macOS, always install with `sudo install` onto a fresh path rather than overwriting the binary in place; a running binary replaced in place is killed by macOS. See the fresh-inode note under [Install → Notes](install.md#notes). On Windows, a running `wind.exe` can't be overwritten at all — rename it, move the new build in, then delete the old one; see the rename-then-replace steps under [Install (Windows) → 1. Install the binary](install-windows.md#1-install-the-binary).

## Terminal garbled or not restored after `wind surf` / `wind down`

`wind surf` and `wind down` are full-screen, raw-mode terminal UIs. They render and restore cleanly on ConPTY (Windows Terminal, modern PowerShell, any macOS/Linux terminal) but not reliably on a legacy console host (`conhost` on older Windows setups) — the screen can come back garbled, or the prompt can stay in raw mode after you leave.

Fix: run wind from Windows Terminal instead of legacy `conhost`. Any of `/exit`, `Ctrl-C`, or `Ctrl-D` should leave the TUI and restore your terminal cleanly — if one leaves it stuck, try another. See [Install (Windows) → Notes](install-windows.md#notes) and [During the work](getting-started.md#during-the-work).

## Token problems (401s, "not authenticated")

Everything that talks to Wind Stream — the CLI, the MCP, the session-end hook — uses the one token `wind auth` manages. A 401, or a command reporting you're not authenticated, means that token is missing, expired, or was revoked.

Fix: run `wind auth status` to see whether a token is stored and whether Wind Stream still considers it live. If not, generate a fresh one in Wind Stream under **Settings → MCP Token** and run `wind auth login` — it also refreshes the selected MCP registration. Claude's stored header is replaced; Codex's environment-backed entry is reconciled without storing the token there. See [wind auth](commands.md#wind-auth).

## Wrong or unreachable environment

Every request the CLI and the MCP send — trace uploads, `wind down`, MCP calls — goes to the host `WIND_HOST` resolves to (default `https://alpha.windstream.ai`). If it's set to the wrong host, or set for the CLI but not picked up by the MCP, things fail or land somewhere you don't expect.

Fix: set `WIND_HOST` to your Wind Stream host, then re-run `wind setup` so the MCP registration is refreshed to match — CLI and MCP need to agree on the same host. See [Install → Notes](install.md#notes) (non-default environments).

## Still stuck?

- `wind doctor` — writes the existing Claude-oriented diagnostics bundle to a file for a bug report. For Codex, pair `wind status`, `codex mcp list`, `/hooks`, and `wind trace` until doctor gains an agent-specific view. Secrets are redacted in the bundle. See [wind doctor](commands.md#wind-doctor).
- `wind hook --selftest` — checks the session-hook path end to end (git, manifest, token, API reachability) and prints `[ok]`/`[FAIL]` per check. See [Plumbing](commands.md#plumbing).
- Still no luck: **success@windstream.ai**.
