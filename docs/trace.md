# What a trace contains

When a connected coding session ends, wind's SessionEnd hook returns a trace
of the session to Wind Stream. This page lists everything a trace carries —
listed here up front, so nothing in a trace is a discovery later.

A trace is only sent from a worktree with a wind session manifest (created by
`wind up` or `wind connect`). Sessions in unconnected directories send
nothing. If Wind Stream is unreachable, the trace is spooled locally under
`~/.windstream/` and delivered on a later session start.

## The transcript

The full session transcript, compressed: every prompt you typed, every
response, and every tool call and result from the session. This is the body
of the trace — the metadata below exists so common questions don't require
reading it.

## Metadata

| Field | What it is |
|-------|------------|
| `model` | The model at session start (legacy — see `provenance.models` for the authoritative per-model record) |
| `cwd` | The worktree path the session ran in |
| `started_at` / `ended_at` | Session start and end timestamps |
| `git_log` | The last 50 commit subject lines of the branch, including author names as they appear in commit subjects' context |

## Provenance (`metadata.provenance`)

A versioned block answering "what produced this work and what did it take":

**Harness and models**

- Harness name and version (e.g. `claude-code 2.1.243`)
- Per-model token counts — input, output, cache writes, cache reads — keyed
  by exact model id, plus turn counts per model. These reflect the main
  session only; work delegated to subagents is not yet included.
- Service tier and speed (e.g. fast mode) per turn, aggregated

**Tools**

- Tool names and call counts (e.g. `Bash: 16`), and the names of MCP servers
  used during the session

**Session shape**

- Duration, end reason, turn count, and how many subagents were dispatched

**Identity**

- Your git identity as configured locally: `git config user.name` and
  `git config user.email`

**Billing context**

- How the seat is connected — subscription or unknown — with the raw billing
  type, seat tier when present, organization name and id, and account id and
  email from the local Claude Code login state. Wind ships this raw context
  so your Wind Stream organization can reason about usage and cost; wind
  itself computes no prices.
- Whether an Anthropic API key was present: wind's SessionEnd hook detects
  whether `ANTHROPIC_API_KEY` is set in its environment and ships **a boolean
  only — never the key, its length, or any derivation of its value**. This
  exists because a machine can carry an old subscription login while a
  session actually bills to an API key; shipping both raw signals lets your
  organization classify correctly.

**Environment**

- wind's own version, operating system and architecture, the terminal app
  (when the terminal identifies itself), and whether the session ran in CI

## Degradation

Every provenance field is best-effort: anything unreadable is omitted or
reported as `unknown`, and the trace ships regardless. A field's absence
never blocks the session or the upload.

## Where it goes

The trace attaches to the Wind Stream session your worktree is bound to, in
your organization. Its visibility is that session's visibility.
