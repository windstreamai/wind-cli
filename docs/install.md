# Install wind (macOS / Linux)

`wind` is the Wind Stream CLI. Instead of launching your coding tool directly, you run `wind up` — it prepares the worktree and session manifest, then launches your selected coding agent. Claude Code and Codex have connected MCP, connect, and trace-return paths. Once you're installed and connected, [Getting started](getting-started.md) walks through the full loop.

> On Windows? See [install-windows.md](install-windows.md).

## Prerequisites

- Claude Code or Codex CLI installed and signed in
- `git`, and `gh` (the GitHub CLI) authenticated — used by `wind up` to work with your repos on GitHub; public binary downloads do not require a GitHub login
- A Wind Stream account with MCP access (token generated in step 2)

## 1. Install the binary

### Standard path

```bash
brew install windstreamai/tap/wind
```

Use Homebrew to install and update a Homebrew-owned copy. See the manual download option below when you need to install a release asset directly.

### Manual download

For a direct download, `gh` also supports authenticated access where required. Pick the asset for your machine — `wind-darwin-arm64` (Apple Silicon), `wind-darwin-x64` (Intel Mac), or `wind-linux-x64` (Linux). Not sure which? `uname -m` tells you: `arm64` → Apple Silicon, `x86_64` → Intel Mac or Linux x64.

```bash
gh release download <tag> --repo windstreamai/wind-cli -p 'wind-darwin-arm64' -p SHA256SUMS
shasum -a 256 -c <(grep darwin-arm64 SHA256SUMS)
mkdir -p "$HOME/.local/bin"
install_path="$HOME/.local/bin/wind"
test ! -e "$install_path" && test ! -L "$install_path" && install -m 755 wind-darwin-arm64 "$install_path"
```

Find the current `<tag>` with:

```bash
gh release view --repo windstreamai/wind-cli
```

If migrating a manual install to Homebrew, keep the existing copy until the new installation and agent configuration are verified. Inspect both copies and select the intended path as described below.

If macOS blocks the directly downloaded binary ("unverified developer"), clear the quarantine attribute:

```bash
xattr -d com.apple.quarantine "$HOME/.local/bin/wind"
```

Confirm it runs:

```bash
wind --version        # → the version you installed
```

The manual recipe requires an unused target; it does not replace another installation. Ensure `~/.local/bin` is on PATH if you choose that location. Manual copies have no native-installer receipt, so their ownership is reported as unverified.

### Inspect installations before switching sources

```bash
wind doctor --installations
wind doctor --installations --json
```

These commands inspect the running binary, PATH candidates and common installation locations without accessing credentials or changing configuration. They report native ownership only when the adjacent `wind.install.json` receipt matches the binary's checksum, or Homebrew ownership when a matching keg receipt is present. Other versions are read from receipts; unknown executables are never launched to determine a version. Shell aliases/functions and arbitrary off-PATH locations are outside this inventory.

The native installer refuses symlink targets and Homebrew `Cellar`/`Caskroom` destinations, including aliases to them. Use `brew upgrade wind` for a Homebrew-owned copy. To adopt a verified legacy **regular native file** without a matching receipt, explicitly set `WIND_REPLACE_UNVERIFIED=1` when running the native installer. This opt-in never bypasses symlink or package-directory protection, and does not authorize overwriting an unrelated receipt file. Use `WIND_INSTALL_DIR` to choose an unused native destination instead when ownership is uncertain.

New native installations write an ownership receipt and stage replacement files on the destination filesystem. A per-target installation lock prevents cooperating native installers from updating simultaneously. After interruption, verify no installer remains before removing a stale `.wind-install.lock` directory. A stale or malformed receipt is not silently trusted; inspect it before explicitly moving it aside or opting into replacement of an unverified binary.

The installer records `.wind-install.pending.json` before replacing the binary. On retry it verifies that record under the installation lock: if the binary matches the pending checksum, it finishes the receipt update; if the old binary still matches its existing receipt, it discards the pending record and retries the requested installation. A pending first install with neither a binary nor an existing receipt can also be retried. This handles process interruption; it does not guarantee recovery after power loss or filesystem damage.

Keep the pending record after an interrupted install and rerun the installer with the intended `WIND_VERSION` and `WIND_INSTALL_DIR`. If an unverified legacy replacement was interrupted before the binary changed, or files no longer match either valid receipt, recovery refuses even with `WIND_REPLACE_UNVERIFIED=1`. Inspect the files and preserve a copy before manually moving metadata aside. Do not remove a lock while an installer is still running.

An explicit older `WIND_VERSION` can reinstall that version's checksum-verified binary. It does not roll back shared credentials, OS keys, queued data, or agent settings. Confirm the older version supports the existing credential format and protocol before using it; otherwise restore a compatible newer installation. A successful binary rollback is not evidence of OAuth downgrade compatibility.

### Shared OAuth custody across installations

Compatible installations use the same per-user OAuth store and SQLite lock, regardless of where their binaries are installed. The encrypted file retains connection IDs, organization choices, defaults, and any recorded uncertainty about a refresh. Removing a binary is separate from removing these shared credentials or their OS protection key.

An unsupported store format, a backend mismatch, or failed decryption must stop access before a replacement credential generation is written. Reinstall a compatible binary and restore access to the original OS credential store rather than deleting the credential file or creating replacement keys. Reinstalling a binary cannot recover a lost protection key. Do not restore an old credential snapshot as a rollback mechanism: its refresh token may already have rotated.

Custody acceptance checks compile separate worker executables from the production store and protection modules and use real OS protection with synthetic credentials. macOS keychain routing is explicitly redirected to a disposable keychain. These checks establish cross-process module behavior and refusal of unsupported format fixtures; they do not certify historical releases, packaged CLI integration, or a live provider's refresh behavior. Keep all concurrently used installations on a supported credential format.

### Select the installation used by your agent

After installing the intended copy, run setup from that installation and pass its stable path:

```bash
# Apple Silicon Homebrew example; use your actual prefix from `brew --prefix`.
/opt/homebrew/bin/wind setup --binary /opt/homebrew/bin/wind

# Native installer example, when this path has a matching native receipt.
"$HOME/.local/bin/wind" setup --binary "$HOME/.local/bin/wind"
```

Selection requires a verified native or Homebrew receipt and the same binary bytes as the running executable. It records the alias you supply, so use Homebrew's `bin/wind` path rather than its versioned `Cellar` path. A manually copied binary without a receipt must first be adopted by the native installer or replaced with a managed installation.

Setup saves this choice before repairing the selected agent's hooks and matching OAuth MCP registration. Review its readout and restart the agent; custom or conflicting registrations require manual repair. The connection, organization, and deployment pins remain unchanged. A saved choice is revalidated on later setup runs. If it disappears or changes ownership, setup stops; if builds differ after an upgrade, run setup from the intended installation itself. Shared settings updates are locked; retry a busy update. After a crash, verify no settings update is running before removing `~/.windstream/.config.lock`.

Selection does not edit PATH or uninstall either copy. Check `type -a wind` and `command -v wind` after changing your shell's PATH. Before removing a copy, select the surviving installation and verify its agent integrations. Remove Homebrew copies through Homebrew; do not delete or replace package-managed links by hand.

Credentials, OS protection keys, work manifests, and queued data remain shared per user. Binary removal is separate from deleting that data. Keep every concurrently used binary compatible with the encrypted OAuth format; copying or selecting an older binary does not downgrade credentials. Full cross-version custody and rollback acceptance is still being validated.

## 2. Connect to Wind Stream

Installing the binary doesn't connect it to Wind Stream — `wind setup` does:

```bash
wind auth login
wind agent codex   # only if Codex should be the default; Claude is the fallback
wind setup
```

Run `wind auth login` first: the terminal shows an address and a code, and you approve access in the browser. `wind setup` then registers a local Wind Stream MCP bridge for the selected agent, with no credential in the agent configuration, and uses the Codex-specific path when Codex is selected; otherwise it preserves the existing Claude setup. For Claude Code it also registers hooks and deploys the `windstream-*` slash commands; restart Claude Code afterward. For Codex it installs hooks and deploys `$windstream-connect` without changing Claude configuration; launch Codex once, open `/hooks`, and trust the Wind hooks when prompted. The hooks and uploads still use a token in this release: generate one in Wind Stream under **Settings → API Tokens** and store it with `wind auth login --token`.

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

- **Updating:** use `brew upgrade wind` for Homebrew, or re-run the native installer for a receipt-verified native installation. Inspect ownership before replacing a manual/legacy copy. Then run `wind setup` from the intended installed binary; setup repairs only matching OAuth authority and preserves conflicting or customized registrations. Restart Claude Code after its setup path; Codex reads its user MCP/skill configuration on the next launch.
- **Non-default environments:** set `WIND_HOST` to your Wind Stream host before `wind setup` (default: `https://app.windstream.ai`).
