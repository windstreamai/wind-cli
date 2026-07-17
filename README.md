# wind — official distribution

Binary distribution of **wind**, the Wind Stream CLI. `wind` connects Wind
Stream workflows to the AI coding tools you work in — it brackets a piece of
work, sets up the session, and returns the results to Wind Stream.

This repository contains **no source code**. It carries the released binaries
([Releases](../../releases)) and is written by Wind Stream's publish pipeline —
every file and release here is a generated artifact of a tag in the private
source repository. Please don't open pull requests against generated content;
issues and support: success@windstream.ai.

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

## License

Proprietary — see [EULA.md](EULA.md), governed by the
[Wind Stream Terms of Service](https://windstream.ai/terms-of-service).
Third-party notices: [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
