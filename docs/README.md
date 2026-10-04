# wind — Wind Stream CLI

wind connects Wind Stream workflows to the AI coding tools you work in — Claude Code and Codex today. It brackets a piece of work: `wind up` stands up a connected session, `wind down` closes the loop with a retrospective, and the trace returns to Wind Stream automatically. It's built for developer productivity — wind absorbs the setup so you don't have to think about it.

## The loop

```
wind up <release> my-feature
  → worktree at ../worktrees/<repo>-<release>-my-feature, branched off release/<release>
  → Claude or Codex connects the worktree to a Wind Stream session
  → …do the work…
  → on session end, the hook returns a trace to Wind Stream
```

## Documentation

| Guide | What's in it |
|-------|--------------|
| [Install (macOS / Linux)](install.md) | Binary install, wind setup, verification |
| [Install (Windows)](install-windows.md) | PowerShell-first install, Windows gotchas |
| [Getting started](getting-started.md) | The session lifecycle: up → work → down |
| [Command reference](commands.md) | Every verb: synopsis, flags, behavior |
| [wind proxy](proxy.md) | Shadow one service behind another, score the parity window |
| [wind factory](factory.md) | The software factory: a Wind Stream brief becomes a gated, evidenced pull request handed to a reviewer |
| [What a trace contains](trace.md) | Everything the SessionEnd trace ships, listed up front |
| [Troubleshooting](troubleshooting.md) | Symptom-first fixes |

Issues and support: success@windstream.ai
