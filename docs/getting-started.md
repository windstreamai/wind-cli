# Getting started with wind

A connected session is a coding session that already knows which Wind Stream session it belongs to — before you've typed a word. You don't launch Claude Code yourself: you run `wind`, and `wind` launches your coding tool (Claude Code today, others to come) already wired up — the worktree, the session manifest, the MCP, and the hooks are all in place. This guide walks the full loop, start to finish.

> Not installed yet? See [Install (macOS / Linux)](install.md) or [Install (Windows)](install-windows.md) first.

## The loop

```
wind up <release> my-feature
  → worktree at ../worktrees/<repo>-<release>-my-feature, branched off release/<release>
  → claude launches into /windstream-connect → attaches or creates a Wind Stream session
  → …do the work…
  → on session end, the hook returns a trace to Wind Stream
```

The sections below walk each step.

## Start a session: wind up

```bash
wind up <release> <slug>
```

creates a worktree off `release/<release>`, writes the session manifest into it, and launches Claude Code there, straight into `/windstream-connect` — the command that attaches to (or creates) the Wind Stream session behind the work.

If `release/<release>` doesn't exist yet, wind creates and pushes it first — with a confirm prompt at an interactive terminal, or immediately with `--create` (what you need non-interactively, e.g. in CI). Run the release-only form on its own to just take that step, with no session attached:

```bash
wind up <release> [--create]
```

Add `--ws <ref>` to attach a specific, already-existing Wind Stream session instead of letting `/windstream-connect` create a new one. Add `--gh <issue>` to record the GitHub issue the session is anchored to in the session manifest — the bare number or a pasted issue URL both work. No name for the work yet? Leave the slug off: with `--ws`/`--gh`, wind fetches the source and proposes one.

`wind up current <slug>` starts a session on the open release train, and `wind up bump patch <slug>` cuts the next one and starts there — so you don't have to look up or type an exact version; see [the command reference](commands.md#wind-up) for the full alias grammar.

Work that isn't on a release train has its own form: `wind up feature <slug>` opens the same kind of session off your repo's default branch — branch `feature/<slug>`, no release involved — and `wind up continue <slug>` reopens that branch later in a fresh worktree. Starting a feature works offline; `continue` does too, once the branch is local; see [Feature sessions](commands.md#feature-sessions-feature-and-continue).

Reviewing someone else's PR has its own form too: `wind up review --gh <pr-url|number> [<slug>]` cuts a `review/<slug>` seat on the PR's head, with the PR recorded as the manifest's anchor. This one needs the network up front, like the `current`/`bump` aliases — it's fetching the PR from GitHub; see [Review sessions](commands.md#review-sessions-review).

## During the work

Deliberate with Wind Stream from your terminal, any time:

```bash
wind surf <topic>
```

It opens a full-screen terminal UI (TUI) — continuing an existing Wind Stream conversation about that topic, or starting a new one. Type `/exit` to leave (`/quit` works too, as a silent alias) and drop back to your shell. Always give it a topic — bare `wind surf` currently just prints a reminder to add one.

Inside Claude Code itself, the same reach is available through the `windstream-*` slash commands that `wind setup` deployed there — `/windstream-connect` is the one that opened this session in the first place.

Want the session in your browser instead of the terminal? From anywhere in the worktree:

```bash
wind open
```

opens the Wind Stream session this work is attached to — it reads the session manifest, so you never have to fish the URL out of a file. Where a browser can't appear (SSH, CI), `wind open --print` prints the URL instead, ready to paste.

## Close the loop: wind down

```bash
wind down
```

opens the same kind of full-screen conversation, framed as a retrospective — Wind Stream asks what happened, you tell it straight, and the exchange writes back to the Wind Stream session this worktree is attached to. `/exit` when you're done.

You don't have to run it, though: the SessionEnd hook returns a trace of the session to Wind Stream automatically when the coding session ends, `wind down` or not. Running `wind down` adds your own account of the work on top of that trace — it doesn't replace it.

`wind down --easy` skips the interview for a quick auto summary instead — you just approve or edit it. For non-interactive runs (CI, scripts), add `--yes` to record that summary without asking, or `--message <text>` to record your own words verbatim instead (the argument is the retrospective — supplying it is the approval; for a file-backed one, `--message "$(cat retro.md)"`) — `wind down --easy` needs one of the two off a terminal, or it refuses rather than hang. A clean exit either way means your words are saved for good, whether delivered right away or spooled to sync on a later session start.

## Not in a wind worktree?

`wind up` is the on-ramp for release work, but you don't need a release worktree to talk to Wind Stream. From any repo:

```bash
wind connect <ws-session-ref> [--url <url>]
```

attaches the current repo to an existing Wind Stream session, ad hoc. It writes the same kind of manifest `wind up` does, just minimal — no release, no branch, only enough for the hooks to know where this session's trace goes when it ends.

## What's on disk

`.session-state.json`, at the root of the worktree, is the session manifest — the record of which Wind Stream session this work belongs to. `wind up` and `wind connect` write it, and `/windstream-connect` fills in the Wind Stream session id once it attaches; the hooks and `wind down` read it to know where things go. Don't edit it by hand — let wind manage it.
