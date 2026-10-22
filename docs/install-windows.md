# Install wind (Windows)

`wind` is the Wind Stream CLI. Instead of launching your coding tool directly, you run `wind up` — it prepares the worktree and session manifest, then launches your selected coding agent. Claude Code and Codex have connected MCP, connect, and trace-return paths. Once you're installed and connected, [Getting started](getting-started.md) walks through the full loop.

> On macOS or Linux? See [install.md](install.md). This guide is PowerShell-first.

## Prerequisites

- Claude Code or Codex CLI installed and signed in
- `git`, and `gh` (the GitHub CLI) authenticated — used by `wind up` to work with your repos on GitHub; public binary downloads do not require a GitHub login
- A Wind Stream account with MCP access (token generated in step 2)
- **Windows Terminal** recommended — `wind surf` / `wind down` use a full-screen raw-mode TUI, which is happiest on ConPTY (Windows Terminal / modern PowerShell). Legacy `conhost` may not render or restore it cleanly.

## 1. Install the binary

### Direct download

Download `wind-windows-x64.exe` and `SHA256SUMS` from
[Releases](https://github.com/windstreamai/wind-cli/releases) into the same folder.
Once the repository is public, this needs no GitHub login. Until then, download
with the authenticated GitHub CLI:

```powershell
gh release download <tag> --repo windstreamai/wind-cli --pattern 'wind-windows-x64.exe' --pattern 'SHA256SUMS' --clobber
```

From the download folder (for either download method):

```powershell
Unblock-File .\wind-windows-x64.exe          # clears the "downloaded from the internet" block (mark-of-the-web)

# Verify before installing; stop if the checksum differs or is missing.
$expected = (Get-Content .\SHA256SUMS | Where-Object { $_ -match '  wind-windows-x64\.exe$' }) -replace '  .*$', ''
$actual = (Get-FileHash .\wind-windows-x64.exe -Algorithm SHA256).Hash.ToLower()
if (-not $expected -or $actual -ne $expected) { throw "wind checksum mismatch" }

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

### winget (pending)

The winget listing requires a separate submission and install verification after
the distribution repos become public. Use the direct download above until that
listing is live.

## 2. Connect to Wind Stream

Installing the binary does not deploy the Wind Stream integration — `wind setup` does. Choose Codex first if that is your agent; otherwise Claude Code remains the default. Setup prompts for a token and uses the Codex-specific path when Codex is selected; that path leaves Claude configuration unchanged. Generate a token in Wind Stream under **Settings → API Tokens**, paste it at the prompt, then press **Enter**:

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

- **Updating:** repeat the download and verification steps above — replace `$HOME\bin\wind.exe` with the new build (`Move-Item … -Force`, or rename-then-replace if it's running), then re-run `wind setup` (idempotent). Restart Claude Code after its setup path; Codex reads its user configuration on the next launch. Always check `wind -v` afterward.
- **Terminal host:** if the TUI renders garbled or won't restore under legacy `conhost`, switch to **Windows Terminal** (ConPTY).
- **Troubleshooting:** the readout above (from `wind setup`) is the health check — every `✓` line is confirmation, not just a status. If something's off, `wind doctor` bundles the same state into a diagnostics file for a bug report; it's a snapshot for support, not a replacement for the readout.
- **`wind factory`:** its verbs run the factory's bash and python scripts and need `bash` and `python3` on PATH (the scripts themselves use `gh`, `curl`, and `jq`) — run them from WSL (or let GitHub Actions run them). `wind factory install` is plain file writing and is expected to work from PowerShell, but has not been exercised there; run it from WSL anyway, so the hook scripts it writes keep their executable bit when committed.
- **Non-default environments:** set `WIND_HOST` to your Wind Stream host before `wind setup` (default: `https://app.windstream.ai`).
