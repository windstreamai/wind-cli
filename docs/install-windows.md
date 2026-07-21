# Install wind (Windows)

`wind` is the Wind Stream CLI. Instead of launching your coding tool directly, you run `wind up` — it stands up a coding session (Claude Code today) that's already connected to Wind Stream: the worktree, the session manifest, the MCP, and the hooks are all wired up before you type a word. Once you're installed and connected, [Getting started](getting-started.md) walks through the full loop.

> On macOS or Linux? See [install.md](install.md). This guide is PowerShell-first.

## Prerequisites

- Claude Code installed
- `git`, and `gh` (the GitHub CLI) authenticated — needed to download the binary while the distribution repository is private, and used by `wind up` to work with your repos on GitHub
- A Wind Stream account with MCP access (token generated in step 2)
- **Windows Terminal** recommended — `wind surf` / `wind down` use a full-screen raw-mode TUI, which is happiest on ConPTY (Windows Terminal / modern PowerShell). Legacy `conhost` may not render or restore it cleanly.

## 1. Install the binary

### Standard path

```powershell
winget install WindStream.Wind
```

This becomes the standard path once the winget listing is live — the submission happens after the distribution flip and is manual, so it's **coming, not live yet**. Until then, use the private-phase path below.

### While this repository is private

During pipeline bring-up (and until winget catches up), `windstreamai/wind-cli` is private, so install with `gh` instead:

```powershell
gh release download <tag> --repo windstreamai/wind-cli --pattern 'wind-windows-x64.exe' --pattern 'SHA256SUMS' --clobber
Unblock-File .\wind-windows-x64.exe          # clears the "downloaded from the internet" block (mark-of-the-web)

# (optional) verify — compare against SHA256SUMS
(Get-FileHash .\wind-windows-x64.exe -Algorithm SHA256).Hash.ToLower()

# put it on PATH as wind.exe — a folder already on PATH, e.g. %USERPROFILE%\bin
mkdir $HOME\bin -Force
Move-Item .\wind-windows-x64.exe $HOME\bin\wind.exe -Force
```

Find the current `<tag>` with:

```powershell
gh release view --repo windstreamai/wind-cli
```

If `$HOME\bin` isn't on your PATH yet, add it once (then open a **new** terminal):

```powershell
setx PATH "$($env:PATH);$HOME\bin"
```

Confirm it runs — and that it's the version you just installed:

```powershell
wind -v          # the version you installed — a stale wind.exe earlier on PATH is the #1 trap
```

> **Two Windows gotchas:**
> - An unsigned `.exe` may trip **SmartScreen** on first run — choose **More info → Run anyway**.
> - You **can't overwrite a running `wind.exe`** (Windows locks it). If `Move-Item` fails with *"in use / access denied,"* close any running `wind`, or rename-then-replace:
>   ```powershell
>   Rename-Item $HOME\bin\wind.exe wind.old.exe   # Windows allows renaming a running exe
>   Move-Item .\wind-windows-x64.exe $HOME\bin\wind.exe
>   Remove-Item $HOME\bin\wind.old.exe
>   ```

## 2. Connect to Wind Stream

Installing the binary does not deploy the Wind Stream integration — `wind setup` does. It registers the session hooks, adds the Wind Stream MCP, deploys the `windstream-*` slash commands, and prompts for a token. Generate one in Wind Stream under **Settings → MCP Token**, paste it at the prompt, then press **Enter**:

```powershell
wind setup
```

Restart Claude Code afterward so it loads the MCP, the hooks, and the new `/windstream-connect` command.

## 3. Verify

`wind setup` prints a readiness readout — every line should be a green `✓`:

```
wind <version>
Wind Stream
  ✓ Connected (https://alpha.windstream.ai)
  ✓ stored: dpapi                 # token in the Windows credential store
  ✓ MCP registered
Claude Code
  ✓ Ready (you@example.com)
  ✓ Commands deployed             # /windstream-connect is now available
  ✓ Hooks installed
Git / GitHub
  ✓ git
  ✓ gh (2.x)
  ✓ authenticated (your-handle)
  ✓ repo scope
  ✓ release access
```

Then confirm the rest:

```powershell
wind -v                 # the version you installed — a stale wind.exe earlier on PATH is the #1 trap
claude mcp list         # windstream → ✓ (connected)
wind hook --selftest    # [ok] git spawn / manifest / token / api reachable
wind status             # readiness of installed coding tools
```

In Claude Code (after the restart) type `/` — **`/windstream-connect`** should be in the list. If it's missing: `wind setup` didn't run from this binary, or Claude Code hasn't been restarted.

Finally, smoke the TUI and confirm it **restores your terminal** on exit:

```powershell
wind surf "hello"       # full-screen UI opens (header + thread + composer)
```

Type a line, try `/menu` and `/open`, then `/exit` — the prompt should return **clean** (no leftover raw-mode, no garbled screen). `Ctrl-C` and `Ctrl-D` should also exit and leave the terminal usable.

## Use it

From a repo with a `release/<version>` branch, start a connected session:

```powershell
wind up <release> <slug>   # worktree off release/<release> + a connected Claude Code session
wind surf <topic>          # deliberate with Wind Stream in your terminal, any time
wind down                  # capture the retrospective for the session
```

See [Getting started](getting-started.md) for the full loop.

## Notes

- **Updating:** once winget is live, `winget upgrade WindStream.Wind`; until then, repeat the `gh` recipe above — replace `$HOME\bin\wind.exe` with the new build (`Move-Item … -Force`, or rename-then-replace if it's running), then re-run `wind setup` (idempotent) and restart Claude Code. Always check `wind -v` afterward.
- **Terminal host:** if the TUI renders garbled or won't restore under legacy `conhost`, switch to **Windows Terminal** (ConPTY).
- **Troubleshooting:** the readout above (from `wind setup`) is the health check — every `✓` line is confirmation, not just a status. If something's off, `wind doctor` bundles the same state into a diagnostics file for a bug report; it's a snapshot for support, not a replacement for the readout.
- **Non-default environments:** set `WIND_HOST` to your Wind Stream host before `wind setup` (default: `https://alpha.windstream.ai`).
