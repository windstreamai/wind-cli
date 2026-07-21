# wind — Wind Stream CLI

wind connects Wind Stream workflows to the AI coding tools you work in — Claude Code today, others to come. It brackets a piece of work: `wind up` stands up a connected session, `wind down` closes the loop with a retrospective, and the trace returns to Wind Stream automatically. It's built for developer productivity — wind absorbs the setup so you don't have to think about it.

## The loop

```
wind up <release> my-feature
  → worktree at ../worktrees/<repo>-<release>-my-feature, branched off release/<release>
  → claude launches into /windstream-connect → attaches or creates a Wind Stream session
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
| [Troubleshooting](troubleshooting.md) | Symptom-first fixes |

Issues and support: success@windstream.ai
