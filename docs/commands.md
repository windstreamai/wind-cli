# wind command reference

Every `wind` verb, what it does, its flags, and how it exits. For the guided walkthrough, see [Getting started](getting-started.md); for install steps, see [Install (macOS / Linux)](install.md) or [Install (Windows)](install-windows.md).

```text
wind — Wind Stream CLI

Usage:
  wind up <release> [--create]                  (Create the release branch off origin's default branch)
  wind up <release> <slug> [--ws <ref>] [--gh <issue>] [--create]   (Open a Wind Stream session in it)
  wind up current [<slug>]                      (The open release train — slug-less just prints it)
  wind up bump major|minor|patch [<slug>]       (Cut the next release, bumped from the latest)
  wind up feature <slug> [--ws <ref>] [--gh <issue>]  (Start a local feature session off the default branch)
  wind up continue <slug> [--ws <ref>] [--gh <issue>] (Reopen an existing feature/<slug> in a fresh worktree)
  wind up review --gh <pr-url|number> [<slug>] [--ws <ref>]  (Cut a review seat on a PR's head, anchored in the manifest)
  wind surf [<topic>]                           (Deliberate with Wind Stream in your terminal)
  wind down [--easy] [--inline] [--yes] [--message <text>]   (Retrospective for this session, written back)
      --easy    quick auto summary, approve/edit it (skips the interview)
      --inline  line-by-line rendering, no full-screen takeover (run it inside Claude Code)
      --yes     with --easy: record the generated summary without prompting
      --message with --easy: record the given text verbatim — supplying it implies acceptance
  wind eval publish <file>                      (Publish an eval envelope NDJSON file to Wind Stream)
  wind open [--print]                           (Open this session's Wind Stream page in the browser)
      --print   print the URL instead of opening it
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
wind up <release> <slug> [--ws <ref>] [--gh <issue>] [--create]
wind up current [<slug>]
wind up bump major|minor|patch [<slug>]
wind up feature <slug> [--ws <ref>] [--gh <issue>]
wind up continue <slug> [--ws <ref>] [--gh <issue>]
wind up review --gh <pr-url|number> [<slug>] [--ws <ref>]
```

`wind up` opens work in one of three worlds: on a **release train** (`wind up <release> …`, the forms documented first below), **off the default branch** (`wind up feature …` / `wind up continue …`, [Feature sessions](#feature-sessions-feature-and-continue)), or **on a PR's head for review** (`wind up review …`, [Review sessions](#review-sessions-review)). Everything after the worktree is created is the same across all three — the manifest, the handoff into Claude Code, `/windstream-connect`, the hooks, `wind down` — except review seats differ; see the Review sessions caveat below.

In release mode, `wind up` has two forms. `wind up <release>` just makes sure `release/<release>` exists — if it doesn't, wind creates it off the repo's default branch on the remote and pushes it (with a confirm prompt at an interactive terminal, or immediately with `--create`, which is what you need non-interactively — CI, scripts). Add a `<slug>` and it opens a session instead: wind creates a worktree off `release/<release>`, writes the session manifest into it, and launches Claude Code there, straight into `/windstream-connect` — the command that attaches to (or creates) the Wind Stream session behind the work.

- `<release>` — required. A concrete version, or one of the two aliases below. Resolves to `release/<release>`.
- `<slug>` — optional. A short name for this piece of work; supplying it switches `wind up` into session mode. Omit it to just ensure the release branch exists (or, with an alias, to query it).
- `--ws <ref>` — session mode only. Attach a specific, already-existing Wind Stream session instead of letting `/windstream-connect` create a new one.
- `--gh <issue>` — session mode only. Record the GitHub issue this session is anchored to — the bare number (`94`) or a pasted issue URL, normalized to the number; it lands in the session manifest as `gh_issue`, where downstream tooling can pick it up (e.g. to stamp `Closes #N` on the session's PR). Structural check only — the issue isn't looked up on GitHub, so `wind up` works before the network does.
- `--create` — create `release/<release>` non-interactively when it doesn't exist yet. At an interactive terminal wind asks first; this is the opt-in for when there's no one to ask.

Don't have a name for the work yet? Leave the slug off and pass a binding flag: `wind up <release> --gh <issue>` (or `--ws <ref>`) fetches the referenced source's title and **proposes a session name** — confirm at the terminal and the session starts under it. Non-interactively wind never guesses: it prints the proposal and the exact re-run command instead. If no title is fetchable (offline, `gh` missing, unresolvable reference), wind says the flags went unused rather than dropping them silently.

In session mode, the worktree lands at `../worktrees/<repo>-<release>-<slug>` — a sibling of your repo, scoped by release so the same slug used across two releases doesn't collide — branched off `release/<release>`, with `.session-state.json` (the session manifest) written at its root.

### Release aliases: `current` and `bump`

`current` and `bump` name a release relative to what's already shipped, so you don't have to know or type an exact version. Both resolve against one live listing of origin's `release/*` branches and `v*` tags — they're strictly online, and only stable `vMAJOR.MINOR.PATCH` names count (a pre-release with an `-rc.N` suffix is ignored, so bump math never comes from an rc).

- `wind up current [<slug>]` — `current` is the open release train: the release branch whose version has no matching tag on origin yet. The tag is the ship marker — shipped release branches may linger on origin, and they don't count. Slug-less, it's a query: `wind up current` just prints the resolved version and exits, no fetch beyond the listing, nothing created. Add a slug and it opens a session on that release, same as `wind up <release> <slug>`. `current` refuses rather than guesses: if the latest version has shipped (its tag exists) it points you at `bump`; if more than one train is open above everything shipped (an early cut), it lists them and asks for a concrete version; an open branch below a shipped version is stale — it never answers, and never blocks the live train.
- `wind up bump major|minor|patch [<slug>]` — resolves the latest stable version the same way, then reports the next one bumped by that level (`major` → `x+1.0.0`, `minor` → `x.y+1.0`, `patch` → `x.y.z+1`) and treats it exactly like `wind up <that-version>`: if the bumped release branch doesn't exist yet (the normal case — you're cutting it), wind creates and pushes it under the same confirm-or-`--create` gate as the concrete form. A slug opens a session there; slug-less just cuts (or confirms) the release.
- Typing the bump level bare (`wind up major`) doesn't guess — it redirects you to the real grammar (`did you mean: wind up bump major?`) rather than silently doing something else.
- Both aliases need the network to resolve (they list origin first); if origin can't be reached, `wind up` says so and asks you to pass a concrete version instead.

Exit: 1 with a usage line for bad arguments, an invalid release name, running outside a git repo, an unresolvable release with no `--create` at a non-interactive terminal, a worktree that already has a live session in it, or (aliases) origin being unreachable or `current` refusing (latest shipped, multiple open trains, or a stale open branch). Declining the create prompt isn't an error — that exits 0. In release-only mode, success (the release already existed, or was just created) exits 0; the `current` query form (no slug) also exits 0 after printing the resolved version. In session mode, once the worktree and manifest are ready, `wind up` hands off to Claude Code and exits with that session's own exit code.

### Feature sessions: `feature` and `continue`

```bash
wind up feature <slug> [--ws <ref>] [--gh <issue>]
wind up feature [--ws <ref>] [--gh <issue>]
wind up continue <slug> [--ws <ref>] [--gh <issue>]
```

Not every piece of work rides a release train. `wind up feature <slug>` opens a session off the repo's default branch instead: branch `feature/<slug>`, worktree at `../worktrees/<repo>-feature-<slug>`, and a manifest with no release in it. Everything downstream is unchanged — the manifest, the handoff into Claude Code, `/windstream-connect`, the hooks, `wind down`. `feature` and `continue` are reserved words in the `<release>` position, the same way `current` and `bump` are, so a release can't be named either of them.

- `<slug>` — the name of the work; it becomes the branch (`feature/dark-mode`), the worktree directory, and the session title. Leave it off and pass `--ws <ref>` or `--gh <issue>` and wind proposes a name from that source's title, exactly as release mode does (confirm at the terminal; non-interactively it prints the proposal and the exact re-run instead of guessing). Bare `wind up feature` — no slug, no flags — has nothing to name the work from and is a usage error: `usage: wind up feature <slug> [--ws <ref>] [--gh <issue>]`.
- `--ws <ref>` / `--gh <issue>` — carry over from release mode unchanged, and land in the manifest the same way.
- `--create` doesn't apply here and is refused rather than ignored: `--create doesn't apply to feature mode — nothing is pushed to origin`. Feature mode pushes nothing to origin — the branch is local until you push it yourself.

**Starting works offline.** wind picks the base in three steps and tells you which one answered before it creates anything (`feature dark-mode -> base main (origin default)`): origin's default branch when origin is reachable, else the cached `origin/HEAD` from your last fetch (`cached origin HEAD — offline`), else local `main` (`local main — no origin`). If none of the three resolves, that's a named error — nothing is created.

**The name has to be free.** Before creating anything, wind checks the branch name against local, cached-remote, and (when origin is reachable) live origin refs. Taken exactly, it refuses and points at the recovery: `feature/dark-mode already exists — yours? wind up continue dark-mode; otherwise pick another slug`. Taken as a prefix — git can't have both a `feature/dark-mode` branch and `feature/dark-mode/*` branches — it refuses too: `feature/dark-mode collides with existing feature/dark-mode/* branches — pick another slug`. Either way nothing is created; pick another slug, or `continue` the branch that's already there.

`wind up continue <slug>` is the other half: it reopens an existing `feature/<slug>` in a fresh worktree — local if it's there, tracked from origin if it only exists there. No branch is created and no ref is minted; you get a fresh worktree, a fresh manifest, and a fresh session that re-attaches to the same Wind Stream session by title. Nothing to reopen is a named error (`feature/dark-mode not found locally or on origin — wind up feature dark-mode to start it`), and so is a branch that's already checked out somewhere else (`feature/dark-mode is already checked out at <path> — cd in to resume`).

Because an offline `wind up feature` can only check the refs it can see, connect re-checks origin once: if `feature/<slug>` turns out to exist on origin **and** that origin branch has diverged from yours (it isn't an ancestor of your local branch — a genuinely different branch wearing the same name), you get a non-fatal warning with the recovery: `warning: feature/dark-mode already exists on origin and this session's branch diverges from it — yours? you likely wanted wind up continue dark-mode. not yours? rename before pushing: git branch -m <new>`. A `continue` session, or a local branch that's simply ahead of origin, is not a divergence and stays silent — as does a check wind can't settle. The prefix case warns whenever it's found: `warning: feature/dark-mode collides with feature/dark-mode/* branches on origin — rename before pushing: git branch -m <new>`. Warnings only — your local work already exists, so wind never blocks on them.

Exit: 1 with a usage line for bad arguments (`usage: wind up feature <slug> [--ws <ref>] [--gh <issue>]`, `usage: wind up continue <slug>`), for `--create`, for a slug that collides, for a slug that normalizes to nothing, for no resolvable base, for running outside a git repo, or for a worktree that already has a live session in it — plus, for `continue`, a target that doesn't exist or is checked out elsewhere. Otherwise `wind up` hands off to Claude Code and exits with that session's own exit code.

### Review sessions: `review`

```bash
wind up review --gh <pr-url|number> [<slug>] [--ws <ref>]
```

`wind up review` opens a session anchored to an existing GitHub pull request instead of a slug you pick yourself: branch `review/<number>-<title>`, worktree at `../worktrees/<repo>-review-<slug>`, and the PR recorded in the session manifest so the session opens already knowing what it's reviewing. This form needs the network up front, like the `current`/`bump` aliases — it fetches the PR's details and head from GitHub before creating anything.

- `--gh <pr-url|number>` — required. A pasted PR URL (`https://github.com/acme/app/pull/907`) or a bare PR number (same-repo by construction — a bare number trusts the checkout you're standing in). A URL for a different repo is refused: `PR #907 belongs to acme/app, but this checkout is dan/app-fork — cd into the acme/app checkout and re-run`.
- `<slug>` — optional. Overrides the name half only; the branch always keeps the PR number as a prefix (`wind up review --gh 907 my-fix` → `review/907-my-fix`), so two reviews of the same PR can never collide and the PR stays identifiable from the branch name alone. Leave it off and wind proposes `<number>-<title>` from the PR's own title, confirming at the terminal — non-interactively it proceeds and logs what it derived instead of asking, since the name is a pure function of the PR and there's nothing to double-check by re-running.
- `--ws <ref>` — carries over unchanged, same as everywhere else.
- `--create` doesn't apply here either and is refused: `--create doesn't apply to review mode — nothing is pushed to origin`.

A PR that doesn't exist, that GitHub can't be reached to check, or that's already merged or closed is refused by name before anything is created: `PR #907 not found — check the number against the repo you're in`; `review needs the network — couldn't fetch PR #907 from GitHub` (plus a `gh` readiness hint naming the cause); `PR #907 is merged — review seats anchor open PRs`. The branch name has to be free the same way a feature branch does: taken and checked out elsewhere → `review/907-my-fix is already checked out at <path> — cd in to resume the seat`; taken with no worktree → `review/907-my-fix already exists — if it's yours locally, delete it (git branch -D review/907-my-fix) to re-cut the seat; if it only exists on origin, pick another slug`; taken as a prefix → `review/907-my-fix collides with existing review/907-my-fix/* branches — pick another slug`.

Exit: 1 with a usage line for bad arguments, for a missing or unparseable `--gh` value, for `--create`, for a cross-repo PR URL, an unreachable network, a PR that's missing/merged/closed, or a branch-name collision — every refusal creates nothing. Declining the terminal confirm isn't an error — that exits 0, same as everywhere else in `wind up`. Otherwise `wind up` hands off to Claude Code and exits with that session's own exit code.

**`wind down` is not yet review-aware:** its summary reads the branch's commits, which on a review seat are the reviewed PR author's — review-aware wind-down is planned.

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
wind down [--easy] [--inline] [--yes] [--message <text>]
```

`wind down` opens the same kind of full-screen conversation as `wind surf`, framed as a retrospective for this session: Wind Stream asks what happened, you tell it straight, and the exchange writes back to the Wind Stream session this worktree is attached to. Type `/exit` when you're done. You don't have to run it — the SessionEnd hook already returns a trace of the session to Wind Stream automatically when the coding session ends, `wind down` or not; running it adds your own account on top of that trace, it doesn't replace it.

- `--easy` — skip the interview. Wind Stream generates a short, accurate summary of the commits on this branch up front; you approve it as-is, edit it before it's recorded, or cancel and write nothing. At most one write, and it's yours to sign off on either way.
- `--inline` — render line by line instead of taking over the full screen. For running `wind down` inside another tool's own terminal (a Claude Code session, say), where a full-screen takeover would fight with the surrounding UI.
- `--yes` — with `--easy`, record the generated summary without prompting: no approve/edit/cancel menu, it just writes. Needs `--easy` — the full interview has no single summary for a flag to approve on your behalf.
- `--message <text>` — with `--easy`, skip the generated summary too: the argument itself is your retrospective, recorded verbatim (whitespace and all). There's no approve step because there's nothing to approve — you wrote the words, and supplying them is the acceptance. Also needs `--easy`. For a retrospective you keep in a file, let the shell hand the text over: `wind down --easy --message "$(cat retro.md)"`, or `--message "$(cat)"` to type/pipe it in. wind never treats the argument as a path, so a value that looks like a flag (`--message --yes`) or a bare `-` is refused up front rather than recorded as your words.

`--easy` and `--inline` are strictly orthogonal — one picks what gets generated (a quick summary instead of the interview), the other picks how it's drawn (line-by-line instead of full-screen). Either flag works with or without the other. `--yes` and `--message` both let the `--easy` path run unattended (CI, scripts) instead of waiting on a menu nobody is there to answer — `--yes` is the approval you'd have typed at that menu, `--message` skips the menu entirely by supplying the retrospective itself.

Exit: 1 if you're not authenticated (run `wind auth login` first), with a usage line for an unrecognized flag, for `--yes`/`--message` without `--easy`, or for a `--message` value that looks like a flag or is a bare `-`. `wind down --easy` also exits 1 at a non-interactive terminal if you give it neither `--yes` nor `--message` — it refuses up front rather than hang waiting on a menu that will never get an answer. Otherwise 0: a deliberate cancel writes nothing and exits 0, same as a worktree with no Wind Stream session attached (`wind down` says so and returns without opening the TUI) — unless the ask was non-interactive (`--yes` or `--message`), in which case asking to record with nothing to record into is a 1. With `--easy --yes` or `--message`, exit 0 means your words are durably saved — delivered right away, or spooled to sync on a later session start if Wind Stream can't be reached right now (offline, or your token needs a refresh); the one other 1 is a save that fails everywhere, locally and on Wind Stream both.

## wind eval

```bash
wind eval publish <file>
```

`wind eval publish` ships an eval-envelope NDJSON file to Wind Stream — eval results your harness wrote, one JSON event per line (`eval.charter`, `eval.run`, `eval.score`), delivered on the same lane wind's own traces ride, under the same stored token. What wind adds over a raw POST is provenance: each `eval.run` line is stamped with where the run came from — the bound Wind Stream session, branch, commit, release, repo, the git user as actor, the lane (`local` at a terminal, `ci` otherwise), and `wind-cli` as the runner. Stamping is fill-absent: a value the file already carries always wins, so a CI sender's own stamps are never overwritten — and every other line reaches Wind Stream byte-exact; wind parses only what it stamps. What it can't stamp it names rather than drops silently — `provenance absent: ws_session_id (no wind manifest)` — and publishes anyway: absent is valid, silent isn't. You don't need to be in a wind worktree, or even a git repo — each provenance field degrades independently.

- `<file>` — required. The envelope file: every non-blank line a JSON object, with exactly one eval family across its `eval.charter`/`eval.run` lines. A malformed line, a missing or mixed family, or a family that isn't slug-shaped is refused by name before anything is sent — as is a file over the pipe's limits (5,000 events, 8 MB), which needs splitting first.

Delivery is honest about which lane you're on. At an interactive terminal, a Wind Stream that can't be reached — offline, a server error, or a token that's missing, expired, or rejected — spools the batch under `~/.windstream/spools/evals/` and says so: `spooled, not delivered (<reason>)`, pending until the eval spool drain ships (tracked as windstream#943). Non-interactively (CI) there is no "pending": transient failures get a bounded number of synchronous retries and then fail hard, so a CI box is either delivered or failed — never left holding a batch its lifecycle can't flush.

Exit: 1 with a usage line for bad arguments; for an unreadable or malformed file, a missing/mixed/malformed family, or a file over the batch limits; when the pipe rejects the batch or any event in it (each failed event is itemized by line); when the acknowledgment doesn't reconcile as durable (reported, and treated as not delivered); and, non-interactively, for auth failures or for transient failures once the retries are exhausted. 2 when the batch spooled instead of delivering (interactive lane only) — pending, not lost, but not yet durable either. 0 means durably delivered: Wind Stream acknowledged every event committed.

## wind open

```bash
wind open [--print]
```

`wind open` opens the Wind Stream session this worktree is attached to in your browser — the "take me there" verb for when you're beside the session, not in it. It resolves the target from the session manifest: the exact URL recorded when the session was bound, or, for a manifest bound before URLs were recorded, the session id plus your Wind Stream host. Either way it echoes the URL it's opening, so even a box where no browser window can appear still shows you the link.

- `--print` — print the bare URL instead of opening a browser, nothing else on the line. For terminals where a browser can't reach you (SSH, CI), or for pasting the link somewhere.

Exit: 1 with a usage line for anything other than `--print`, if there's no session manifest here (run `wind up` or `wind connect` first), or if the manifest has no Wind Stream session bound yet (run `/windstream-connect` in your coding session to attach one). Otherwise 0 — with `--print` that means the URL was printed; without it, that the URL was handed to your system's opener (wind can't see whether a window actually appeared).

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

- **`wind hook <session-start|session-end|retro-drain>`** — invoked by the SessionStart/SessionEnd hooks `wind setup` registers with Claude Code. Reads the hook's JSON off stdin; at session start, when the worktree has a manifest, writes the session binding and emits an orientation block to stdout (the manifest contents and two Verify lines for the current branch and working directory) so every agent session starts with its Wind Stream context in view — without a manifest, session start does nothing. Only the manifest fields wind recognizes are relayed, and their values pass the same secret-redaction as wind's traces; anything dropped or unrenderable is called out in the block itself (a count, never the raw content), with the details in `wind trace`. At session end, uploads (or spools, if Wind Stream is unreachable) the session trace. Always exits 0 — a lifecycle hook must never crash your coding session, so every internal failure is swallowed rather than surfaced as a nonzero exit. `wind hook --selftest` runs a quick manual check instead (git, manifest, token, API reachability) and prints `[ok]`/`[FAIL]` lines — the one case here that exits 1, if a check fails. At session start it also drains the home-level spool queues (`~/.windstream/spools/`): undelivered traces flush in-hook, and pending `wind down` turns are handed to a detached `wind hook retro-drain` child — also plumbing, spawned automatically and lock-guarded, never something you run yourself — so they finish syncing at their own pace without delaying your session.
- **`wind bind <ws_session_id> [--url <url>]`** — run by `/windstream-connect` once it has resolved (or created) the Wind Stream session for this worktree, to persist that id — and optionally its URL — into `.session-state.json`. In a feature session it also re-checks the branch name against origin and may print a non-fatal warning (see [Feature sessions](#feature-sessions-feature-and-continue)); the exit code is unaffected either way. Exits 1 with a usage line for bad arguments, or if there's no manifest here yet (run `wind up` first); 0 on success.
- **`wind trace [--run <id>] [--json]`** — inspect wind's own trace output: the events each `wind` invocation logs while it runs. Bare `wind trace` prints the most recent run, pretty-printed; `--run <id>` picks an older one by matching its filename; `--json` prints raw JSON lines instead of the pretty form. Exits 0, or 1 if `--run <id>` matches nothing or an unknown flag is passed.
