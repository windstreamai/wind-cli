# wind — the Wind Stream CL

Binary distribution of **wind**, the Wind Stream CLI. `wind` connects Wind
Stream workflows to your AI coding tools like Claude Code, Antigravity, and Codex. 
`wind` connects work to your GitHub integration and makes sure workstreams stay together.
It sets up the session, and returns traces and surface your retrospective to Wind Stream.

This repository is for the released binaries
([Releases](../../releases)) only and is written by Wind Stream's publish pipeline.

## Install

> During the pipeline bring-up this repository is private and installs go
> through `gh`; see the instructions on the current release. With the public
> launch, the package-manager channels take over:

**macOS / Linux (Homebrew)**

```bash
brew install windstreamai/tap/wind
```

**Windows (winget)**

```powershell
winget install WindStream.Wind
```

After any install: `wind setup` deploys the Wind Stream integration (hooks,
MCP, `windstream-*` commands).

## Docs

Full documentation lives in [docs/](docs/README.md) — install, getting started, the command reference, and troubleshooting.


## License

Proprietary — see [EULA.md](EULA.md), governed by the
[Wind Stream Terms of Service](https://windstream.ai/terms-of-service).

Third-party notices: [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
