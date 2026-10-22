# wind — the Wind Stream CLI

Binary distribution of **wind**, the Wind Stream CLI. `wind` connects Wind
Stream workflows to your AI coding tools like Claude Code and Codex.
`wind` connects work to your GitHub integration and makes sure workstreams stay together.
It sets up the session, and returns traces and surfaces your retrospective to Wind Stream.

This repository is for the released binaries
([Releases](../../releases)) only and is written by Wind Stream's publish pipeline.

## Install

**macOS / Linux (Homebrew)** — available once the distribution repos are public:

```bash
brew install windstreamai/tap/wind
```

**Windows** — download `wind-windows-x64.exe` and `SHA256SUMS` from
[Releases](https://github.com/windstreamai/wind-cli/releases), then follow the
[Windows install guide](docs/install-windows.md) to verify and install it.
The winget listing is pending; public downloads do not require a GitHub login.

While the distribution repos are private, use the authenticated `gh` instructions
on the current release.

After any install: `wind setup` deploys the Wind Stream integration (hooks,
MCP, `windstream-*` commands).

## Docs

Full documentation lives in [docs/](docs/README.md) — install, getting started, the command reference, and troubleshooting.

## License

Proprietary — see [EULA.md](EULA.md), governed by the
[Wind Stream Terms of Use](https://windstream.ai/legal/tos).

Third-party notices: [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
