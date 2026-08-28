# Install wind (macOS / Linux)

`wind` is the Wind Stream CLI. Instead of launching your coding tool directly, you run `wind up` — it prepares the worktree and session manifest, then launches your selected coding agent. Claude Code and Codex have connected MCP, connect, and trace-return paths; Antigravity remains launcher-only. Once you're installed and connected, [Getting started](getting-started.md) walks through the full loop.

> On Windows? See [install-windows.md](install-windows.md).

## Prerequisites

- Claude Code or Codex CLI installed and signed in
- `git`, and `gh` (the GitHub CLI) authenticated — needed to download the binary while the distribution repository is private, and used by `wind up` to work with your repos on GitHub
- A Wind Stream account with MCP access (token generated in step 2)

## 1. Install the binary

### Standard path

```bash
brew install windstreamai/tap/wind
```

This becomes the standard path at the public launch — while the distribution repository is private, use the `gh` path below.

### While this repository is private

During pipeline bring-up, `windstreamai/wind-cli` is private, so install with `gh` instead of `brew`. Pick the asset for your machine — `wind-darwin-arm64` (Apple Silicon), `wind-darwin-x64` (Intel Mac), or `wind-linux-x64` (Linux). Not sure which? `uname -m` tells you: `arm64` → Apple Silicon, `x86_64` → Intel Mac or Linux x64.

```bash
gh release download <tag> --repo windstreamai/wind-cli -p 'wind-darwin-arm64' -p SHA256SUMS
shasum -a 256 -c <(grep darwin-arm64 SHA256SUMS)
chmod +x wind-darwin-arm64 && sudo install wind-darwin-arm64 /usr/local/bin/wind
```

Find the current `<tag>` with:

```bash
gh release view --repo windstreamai/wind-cli
```

If macOS blocks the binary ("unverified developer"), clear the quarantine attribute:

```bash
xattr -d com.apple.quarantine /usr/local/bin/wind
```

Confirm it runs:

```bash
wind --version        # → the version you installed
```

## 2. Connect to Wind Stream

Installing the binary doesn't connect it to Wind Stream — `wind setup` does:

```bash
wind agent codex   # only if Codex should be the default; Claude is the fallback
wind setup
```

It prompts for a token and uses the Codex-specific path when Codex is selected; otherwise it preserves the existing Claude setup. Generate one in Wind Stream under **Settings → MCP Token**, paste it at the prompt, then press **Enter**. For Claude Code it registers hooks, adds the MCP, and deploys the `windstream-*` slash commands; restart Claude Code afterward. For Codex it registers the native remote MCP, installs hooks, and deploys `$windstream-connect` without changing Claude or Antigravity configuration; launch Codex once, open `/hooks`, and trust the Wind hooks when prompted.

## 3. Verify

```bash
wind hook --selftest    # [ok] lines
wind status             # readiness of installed coding tools
claude mcp list         # Claude: windstream → ✓ (connected)
codex mcp list          # Codex: windstream is enabled with bearer env WINDSTREAM_TOKEN
```

`wind setup` itself prints a readiness readout when it runs — every line should show a green ✓.

## Use it

From a repo with a `release/<version>` branch, start a connected session:

```bash
wind up <release> <slug>   # worktree off release/<release> + a connected agent session
wind agent codex           # make Codex the default launcher; re-run wind setup after changing it
wind surf <topic>          # deliberate with Wind Stream in your terminal, any time
wind down                  # capture the retrospective for the session
```

See [Getting started](getting-started.md) for the full loop.

## Notes

- **Updating:** `brew upgrade wind` (standard path); while the repo is private, re-run the `gh` recipe above — `sudo install` again (never overwrite the binary in place; macOS kills a running binary replaced in place), then re-run `wind setup` (idempotent). Restart Claude Code after its setup path; Codex reads its user MCP/skill configuration on the next launch.
- **Non-default environments:** set `WIND_HOST` to your Wind Stream host before `wind setup` (default: `https://alpha.windstream.ai`).
