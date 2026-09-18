# Install wind (Windows)

`wind` is the Wind Stream CLI. Instead of launching your coding tool directly, you run `wind up` — it prepares the worktree and session manifest, then launches your selected coding agent. Claude Code and Codex have connected MCP, connect, and trace-return paths; Antigravity remains launcher-only. Once you're installed and connected, [Getting started](getting-started.md) walks through the full loop.

> On macOS or Linux? See [install.md](install.md). This guide is PowerShell-first.

## Prerequisites

- Claude Code or Codex CLI installed and signed in
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

Installing the binary does not deploy the Wind Stream integration — `wind setup` does. Choose Codex first if that is your agent; otherwise Claude Code remains the default. Setup prompts for a token and uses the Codex-specific path when Codex is selected; that path leaves Claude and Antigravity configuration unchanged. Generate a token in Wind Stream under **Settings → MCP Token**, paste it at the prompt, then press **Enter**:

```powershell
wind agent codex   # optional; omit for Claude Code
wind setup
```

Restart Claude Code after its setup path. For Codex, launch it once, open `/hooks`, and trust the Wind lifecycle hooks when prompted; the native MCP and `$windstream-connect` skill load on launch.

## 3. Verify

`wind setup` prints a readiness readout — every line should be a green `✓`. The default Claude Code path looks like this:

```
wind <version>
Wind Stream
  ✓ Connected (https://app.windstream.ai)
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

With Codex selected, the middle of that readout is Codex-specific instead:

```
Wind Stream
  ✓ Codex MCP registered
Codex CLI
  ✓ Ready (codex-cli <version>)
  ✓ Connect skill deployed
  ✓ Lifecycle hooks installed
```

Hook installation is not the trust decision: open `/hooks` in Codex once and approve the Wind handlers.

Then confirm the rest (use the MCP command for your selected agent):

```powershell
wind -v                 # the version you installed — a stale wind.exe earlier on PATH is the #1 trap
claude mcp list         # windstream → ✓ (connected)
codex mcp list          # windstream → enabled, bearer env WINDSTREAM_TOKEN
wind hook --selftest    # [ok] git spawn / manifest / token / api reachable
wind status             # readiness of installed coding tools
```

In Claude Code (after the restart) type `/` — **`/windstream-connect`** should be in the list. In Codex, invoke **`$windstream-connect`** and use `/hooks` to confirm the Wind hooks are trusted. If the selected connect entry is missing, re-run `wind setup` from the installed binary.

Finally, smoke the TUI and confirm it **restores your terminal** on exit:

```powershell
wind surf "hello"       # full-screen UI opens (header + thread + composer)
```

Type a line, try `/menu` and `/open`, then `/exit` — the prompt should return **clean** (no leftover raw-mode, no garbled screen). `Ctrl-C` and `Ctrl-D` should also exit and leave the terminal usable.

## Use it

From a repo with a `release/<version>` branch, start a connected session:

```powershell
wind up <release> <slug>   # worktree off release/<release> + a connected agent session
wind agent codex           # make Codex the default launcher; re-run wind setup after changing it
wind surf <topic>          # deliberate with Wind Stream in your terminal, any time
wind down                  # capture the retrospective for the session
```

See [Getting started](getting-started.md) for the full loop.

## Notes

- **Updating:** once winget is live, `winget upgrade WindStream.Wind`; until then, repeat the `gh` recipe above — replace `$HOME\bin\wind.exe` with the new build (`Move-Item … -Force`, or rename-then-replace if it's running), then re-run `wind setup` (idempotent). Restart Claude Code after its setup path; Codex reads its user configuration on the next launch. Always check `wind -v` afterward.
- **Terminal host:** if the TUI renders garbled or won't restore under legacy `conhost`, switch to **Windows Terminal** (ConPTY).
- **Troubleshooting:** the readout above (from `wind setup`) is the health check — every `✓` line is confirmation, not just a status. If something's off, `wind doctor` bundles the same state into a diagnostics file for a bug report; it's a snapshot for support, not a replacement for the readout.
- **Non-default environments:** set `WIND_HOST` to your Wind Stream host before `wind setup` (default: `https://app.windstream.ai`).
