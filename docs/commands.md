# wind command reference

Every `wind` verb, what it does, its flags, and how it exits. For the guided walkthrough, see [Getting started](getting-started.md); for install steps, see [Install (macOS / Linux)](install.md) or [Install (Windows)](install-windows.md).

```text
wind — Wind Stream CLI

Usage:
  wind up <release> [--create]                  (Create the release branch off origin's default branch)
  wind up <release> <slug> [--ws <ref>] [--create]   (Open a Wind Stream session in it)
  wind up current [<slug>]                      (The latest release — slug-less just prints it)
  wind up bump major|minor|patch [<slug>]       (Cut the next release, bumped from the latest)
  wind surf [<topic>]                           (Deliberate with Wind Stream in your terminal)
  wind down [--easy] [--inline]                 (Retrospective for this session, written back)
      --easy    quick auto summary, approve/edit it (skips the interview)
      --inline  line-by-line rendering, no full-screen takeover (run it inside Claude Code)
  wind connect <ws-session-ref> [--url <url>]   (Connect this repo to a Wind Stream session)
  wind setup                                    (Set up wind on this machine)
  wind auth <login|status|logout>               (Manage your Wind Stream token)
  wind doctor                                   (Capture diagnostics for a bug report)
  wind status                                   (Show readiness of installed coding tools)
  wind --version
```

Run `wind` with no arguments, `--help`, `-h`, or `help` to print this same block. An unrecognized command prints `unknown command: <cmd>` followed by the block, and exits 1.

## wind up

```bash
wind up <release> [--create]
wind up <release> <slug> [--ws <ref>] [--create]
wind up current [<slug>]
wind up bump major|minor|patch [<slug>]
```

`wind up` has two forms. `wind up <release>` just makes sure `release/<release>` exists — if it doesn't, wind creates it off the repo's default branch on the remote and pushes it (with a confirm prompt at an interactive terminal, or immediately with `--create`, which is what you need non-interactively — CI, scripts). Add a `<slug>` and it opens a session instead: wind creates a worktree off `release/<release>`, writes the session manifest into it, and launches Claude Code there, straight into `/windstream-connect` — the command that attaches to (or creates) the Wind Stream session behind the work.

- `<release>` — required. A concrete version, or one of the two aliases below. Resolves to `release/<release>`.
- `<slug>` — optional. A short name for this piece of work; supplying it switches `wind up` into session mode. Omit it to just ensure the release branch exists (or, with an alias, to query it).
- `--ws <ref>` — session mode only. Attach a specific, already-existing Wind Stream session instead of letting `/windstream-connect` create a new one.
- `--create` — create `release/<release>` non-interactively when it doesn't exist yet. At an interactive terminal wind asks first; this is the opt-in for when there's no one to ask.

In session mode, the worktree lands at `../worktrees/<repo>-<release>-<slug>` — a sibling of your repo, scoped by release so the same slug used across two releases doesn't collide — branched off `release/<release>`, with `.session-state.json` (the session manifest) written at its root.

### Release aliases: `current` and `bump`

`current` and `bump` name a release relative to what's already shipped, so you don't have to know or type an exact version. Both resolve against one live listing of origin's `release/*` branches and `v*` tags — they're strictly online, and only stable `vMAJOR.MINOR.PATCH` names count (a pre-release with an `-rc.N` suffix is ignored, so bump math never comes from an rc).

- `wind up current [<slug>]` — `current` is the highest stable version that still has a live release branch on origin. Slug-less, it's a query: `wind up current` just prints the resolved version and exits, no fetch beyond the listing, nothing created. Add a slug and it opens a session on that release, same as `wind up <release> <slug>`. If the latest version has already shipped (its release branch is gone, only the tag remains), `current` refuses and points you at `bump` instead.
- `wind up bump major|minor|patch [<slug>]` — resolves the latest stable version the same way, then reports the next one bumped by that level (`major` → `x+1.0.0`, `minor` → `x.y+1.0`, `patch` → `x.y.z+1`) and treats it exactly like `wind up <that-version>`: if the bumped release branch doesn't exist yet (the normal case — you're cutting it), wind creates and pushes it under the same confirm-or-`--create` gate as the concrete form. A slug opens a session there; slug-less just cuts (or confirms) the release.
- Typing the bump level bare (`wind up major`) doesn't guess — it redirects you to the real grammar (`did you mean: wind up bump major?`) rather than silently doing something else.
- Both aliases need the network to resolve (they list origin first); if origin can't be reached, `wind up` says so and asks you to pass a concrete version instead.

Exit: 1 with a usage line for bad arguments, an invalid release name, running outside a git repo, an unresolvable release with no `--create` at a non-interactive terminal, a worktree that already has a live session in it, or (aliases) origin being unreachable or `current` finding no release branch left to attach to. Declining the create prompt isn't an error — that exits 0. In release-only mode, success (the release already existed, or was just created) exits 0; the `current` query form (no slug) also exits 0 after printing the resolved version. In session mode, once the worktree and manifest are ready, `wind up` hands off to Claude Code and exits with that session's own exit code.

## wind surf

```bash
wind surf [<topic>]
```

`wind surf` opens a full-screen conversation with Wind Stream, right in your terminal — deliberate about a topic any time, not just mid coding-session. Give it a topic and it either continues an existing Wind Stream conversation that matches, or starts a new one. Type `/exit` to leave and drop back to your shell (`/quit` also works, as a silent alias).

- `<topic>` — optional. A short label for what you want to talk about.

Bare `wind surf` (no topic) doesn't open the TUI yet — it prints a pointer to add one and returns.

Exit: 1 if you're not authenticated (run `wind auth login` first), or with a usage line if you pass more than one argument or any flag (`wind surf` takes none). Bare `wind surf` isn't an error either — it exits 0 after printing the pointer. With a topic, `wind surf` exits 0 once you leave the TUI.

## wind down

```bash
wind down [--easy] [--inline]
```

`wind down` opens the same kind of full-screen conversation as `wind surf`, framed as a retrospective for this session: Wind Stream asks what happened, you tell it straight, and the exchange writes back to the Wind Stream session this worktree is attached to. Type `/exit` when you're done. You don't have to run it — the SessionEnd hook already returns a trace of the session to Wind Stream automatically when the coding session ends, `wind down` or not; running it adds your own account on top of that trace, it doesn't replace it.

- `--easy` — skip the interview. Wind Stream generates a short, accurate summary of the commits on this branch up front; you approve it as-is, edit it before it's recorded, or cancel and write nothing. At most one write, and it's yours to sign off on either way.
- `--inline` — render line by line instead of taking over the full screen. For running `wind down` inside another tool's own terminal (a Claude Code session, say), where a full-screen takeover would fight with the surrounding UI.

`--easy` and `--inline` are strictly orthogonal — one picks what gets generated (a quick summary instead of the interview), the other picks how it's drawn (line-by-line instead of full-screen). Either flag works with or without the other.

Exit: 1 if you're not authenticated (run `wind auth login` first), or with a usage line if you pass a flag other than `--easy`/`--inline`. 0 otherwise — including when this worktree isn't attached to a Wind Stream session at all, in which case `wind down` says so and exits without opening the TUI.

## wind connect

```bash
wind connect <ws-session-ref> [--url <url>]
```

`wind connect` attaches the current repo to an already-existing Wind Stream session, ad hoc — for repos `wind up` didn't set up. It writes the same kind of manifest `wind up` does, just minimal: no release, no branch, just enough for the SessionStart/SessionEnd hooks to know where this session's trace goes. Nothing uploads immediately: `connect` runs mid-session, so the transcript uploads to Wind Stream when the current Claude Code session ends.

- `<ws-session-ref>` — required. The Wind Stream session to attach to.
- `--url <url>` — optional. The session's canonical URL, stored alongside the reference.

Exit: 1 with a usage line for bad arguments, if you're not inside a git repo, or if a `.session-state.json` already exists here — `wind connect` never overwrites an existing manifest; a `wind up` session (or an earlier `wind connect`) already owns that directory. 0 on success.

## wind setup

```bash
wind setup
```

`wind setup` connects this machine to Wind Stream: it registers the SessionStart/SessionEnd hooks, deploys the `windstream-*` slash commands to `~/.claude/commands`, prompts for your Wind Stream token (skipping the prompt if one's already stored), and registers the Wind Stream MCP with Claude Code. It's idempotent — re-run it any time (after a token rotates, to pick up newly deployed commands, after moving machines) and it only touches what's actually changed.

No arguments or flags.

It prints a readiness readout when it's done — Wind Stream, Claude Code, and Git/GitHub, each line marked with a check or a warning — so you can see at a glance what's connected and what still needs attention.

Exit: always 0. A missing precondition (git not installed, `gh` not authenticated, Claude Code not signed in, no token yet) prints a warning in the readout, not a failure — `wind setup` always does as much as it can and tells you what's left.

## wind auth

```bash
wind auth login
wind auth status
wind auth logout
```

`wind auth` manages the Wind Stream token everything else here uses — the CLI, the MCP, the SessionEnd hook. `login` reads a token (pasted at a prompt, or piped via stdin) and stores it, then refreshes the Wind Stream MCP registration in Claude Code so it isn't left pointing at a stale token. `status` reports whether the stored token is actually live — checked directly against Wind Stream, not just whether one exists locally — and where it's stored. `logout` removes the stored token.

- one of `login`, `status`, `logout` — required.

Exit: 1 with a usage line if you omit the subcommand or pass anything else. `login` exits 1 if you don't enter a token, 0 otherwise — even if the MCP refresh fails, which only warns. `status` exits 1 if no token is stored, 0 if one is, whether it turns out to still be live or not (the liveness verdict prints either way). `logout` always exits 0.

## wind doctor

```bash
wind doctor
```

`wind doctor` captures a diagnostics bundle for a bug report: platform, whether the token, MCP registration, deployed commands, and hooks are actually present — probed live, not cached — plus Claude Code and git/`gh` readiness. It writes the bundle as JSON to your home directory and prints the same grouped readiness readout `wind setup` does, so you can read the diagnosis without opening the file. Secrets are redacted in the written file — share it freely on a bug report.

No arguments or flags.

Exit: always 0.

## wind status

```bash
wind status
```

`wind status` reports readiness of the coding tools wind knows how to connect — today, Claude Code and Antigravity: whether each is installed, signed in, and its version. It's distinct from `wind setup`'s readout (wind's own connection to Wind Stream — hooks, MCP, commands) and from `wind doctor`'s diagnostics bundle (a captured, shareable snapshot for a bug report) — `status` is a quick, live probe of the tools themselves.

No arguments or flags.

Exit: always 0.

## wind --version

```bash
wind --version
```

Prints the installed version and exits. `-v` and `version` are aliases.

Exit: always 0.

## wind --help

```bash
wind --help
```

Prints the usage block reproduced at the top of this reference. `-h`, `help`, and running `wind` with no arguments at all are aliases.

Exit: always 0.

## Plumbing

You don't run these — wind's own hooks and the `/windstream-connect` command do. Documented here for completeness, and because `wind trace` is genuinely useful to reach for yourself when you want to see what a run actually did.

- **`wind hook <session-start|session-end>`** — invoked by the SessionStart/SessionEnd hooks `wind setup` registers with Claude Code. Reads the hook's JSON off stdin, writes the session binding at start, and uploads (or spools, if Wind Stream is unreachable) the session trace at end. Always exits 0 — a lifecycle hook must never crash your coding session, so every internal failure is swallowed rather than surfaced as a nonzero exit. `wind hook --selftest` runs a quick manual check instead (git, manifest, token, API reachability) and prints `[ok]`/`[FAIL]` lines — the one case here that exits 1, if a check fails.
- **`wind bind <ws_session_id> [--url <url>]`** — run by `/windstream-connect` once it has resolved (or created) the Wind Stream session for this worktree, to persist that id — and optionally its URL — into `.session-state.json`. Exits 1 with a usage line for bad arguments, or if there's no manifest here yet (run `wind up` first); 0 on success.
- **`wind trace [--run <id>] [--json]`** — inspect wind's own trace output: the events each `wind` invocation logs while it runs. Bare `wind trace` prints the most recent run, pretty-printed; `--run <id>` picks an older one by matching its filename; `--json` prints raw JSON lines instead of the pretty form. Exits 0, or 1 if `--run <id>` matches nothing or an unknown flag is passed.
