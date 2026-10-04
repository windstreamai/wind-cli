# wind factory

A Wind Stream brief becomes a gated, evidenced pull request handed to a reviewer, with no one in the loop until the review.

You decide what to build in a Wind Stream session titled `FACTORY: …`, or you label an existing issue `factory-build` and `sprint-ready`. The factory takes it from there: a GitHub issue, a Copilot cloud agent started against the repository's `factory-build` branch, CI lifted and the agent nudged when it stalls, a verdict from the brief session on every pushed head (`VERDICT: OK` or `VERDICT: BLOCK — reason`, posted as `@copilot` so the agent reconciles), and finally one handoff comment per green, gate-OK head: the hold table of the checks the brief names, the no-human proof from GitHub's own event record, the `factory-ready` label, and a review request. A person merges. The receipt and the executor trace land in the session on close. The session is the ledger: brief, verdicts, and receipt all live in it.

```
   Wind Stream session "FACTORY: …"  /  issue labelled factory-build + sprint-ready
              │  wind factory intake · dispatch     (every 30 min; one claim at a time, behind an arming switch)
              ▼
   GitHub issue  →  Copilot cloud agent on factory-build        gh agent-task create --base factory-build --custom-agent windstream-issue
              │  wind factory sweep                  (every 10 min: ready, lift CI, nudge, hold, hand off)
              │  wind factory gate-sweep             (a verdict per unjudged head, scored on /evals)
              ▼
   factory-ready PR, reviewer requested  →  a person merges into factory-build  →  promotion to the release train, by a person
```

## Set it up

1. In the repository, run `wind factory install`. It writes the four thin triggers (`.github/workflows/windstream-{intake,gate,ci-triage,launch}.yml`), the Copilot hooks, the two custom agents, the repository instructions, the coding agent's MCP configuration, the issue template, and the two helper scripts the hooks fall back to inside the Copilot sandbox, where wind is not on PATH. It also adds `.factory/` to `.gitignore`, where the hooks keep per-session state. The triggers must live on the default branch, because GitHub only fires `schedule`, `workflow_dispatch`, and `workflow_run` from there; they check out `FACTORY_BASE_BRANCH` and run the verbs, which install this same wind version on every run. A repository that already runs an older factory under the same workflow names needs those disabled first: disabling is repository-wide by file name.
2. Give the factory an identity. The tokens belong to a **bot account** with write access and a Copilot seat, never to a person:
   - `WINDSTREAM_MCP_TOKEN` (Actions secret): the bot's Wind Stream MCP token.
   - `FACTORY_ASSIGN_TOKEN` (Actions secret): the bot's `gho_…` token from `gh auth login` on a machine signed in as the bot (`gh secret set FACTORY_ASSIGN_TOKEN --body "$(gh auth token)"`). `gh agent-task` refuses personal access tokens of either kind; this account's name appears on launches and BLOCK comments.
   - `COPILOT_MCP_WINDSTREAM_TOKEN`: Repo → Settings → Copilot → Coding agent → environment secret, beside the pasted `.github/copilot/mcp-servers.json`. The cloud agent does not read a project `.mcp.json`, and the install writes none: a project-scope `windstream` server shadows the one you configured at user scope.
3. `wind factory labels <owner/repo>` creates the labels: `factory`, `groomed`, `needs-decision`, `factory-build`, `sprint-ready`, `factory-claimed`, `factory-building`, `factory-failed`, `factory-ready`, `factory-stuck`, `factory-budget`. It does not create `ci-failure`, which triage files issues under: `gh label create ci-failure`.
4. Set the variables: `FACTORY_SHARE_IDS` and `FACTORY_MENTION_IDS` (Wind Stream user ids the factory shares its sessions with and mentions on every turn), `FACTORY_REVIEWERS` (GitHub logins the handoff requests review from).
5. Create `factory-build` from the default branch, synchronize the open release train into it, protect it with the repository's required test check, then set `FACTORY_BASE_BRANCH=factory-build`. Every launch targets it; intake, gate, and triage check it out. Nothing is guessed from branch names, and a missing branch stops a launch.
6. Keep `FACTORY_DISPATCH_ENABLED` unset through bootstrap. Set `FACTORY_CANARY_ISSUES` to an ordered list of issue numbers and `FACTORY_DISPATCH_LIMIT=1`. A non-empty canary list is a strict allow-list, so nothing else can launch. Arm with `FACTORY_DISPATCH_ENABLED=true` once the canaries, the required checks, and `wind factory health` are green; clear the canary list only after the sequence is green.
7. Optional telemetry: `WINDSTREAM_OTLP_URL=https://app.windstream.ai/api/v1/otlp`. Every factory job then emits one OpenTelemetry span; the gate adds the judge's scores and capture adds verdict-quality and loop-outcome facts, shown on `/evals` as the `factory` composite. Unset is the kill switch: no telemetry, no failure. A repository that authors its own eval charters under `api/evals/charters/` publishes them once per environment with `wind factory charters > f.ndjson && wind eval publish f.ndjson`.
8. Name your CI workflow: `workflows:` in `windstream-ci-triage.yml` and the variable `FACTORY_CI_WORKFLOW` the sweep lifts and compares against (both default `CI`).

`wind factory health` checks both credentials and distinguishes the failures (by hand it needs `FACTORY_ASSIGN_TOKEN` exported too, else it reports degraded; `WIND_TOKEN` or your stored wind token stands in for `WINDSTREAM_MCP_TOKEN`): exit 0 connected, 1 token expired, 2 unreachable, 3 MCP token unset, 4 degraded (MCP fine, but `FACTORY_ASSIGN_TOKEN` missing or not `gho_`, which silently disables gate auto-reconcile and cloud-agent launch). `wind factory selftest` runs the lane's eleven self-tests.

## Write a brief

Start a session, title it `FACTORY: <task>`, and put the brief in the first turn:

```
Repo: <owner/repo>
Brief: what to build, and why — one task per brief
Decisions already made:
  - what is settled and must not be reopened
Acceptance:
  - checks that must hold
Checks: API Tests, Frontend Tests + Build
```

`Repo:` must match the repository, or intake skips the brief by design. `Checks:` names the CI jobs the hold must see green at the head before the handoff; a named check that never ran, or was skipped, is not green, and a check that is red here but was green on the base branch's last run is a regression. `Base branch:` is an explicit exception to `FACTORY_BASE_BRANCH`. The factory's sessions are owned by the bot and shared with `FACTORY_SHARE_IDS`; a brief you create yourself in the UI is yours alone until you add the bot as a participant, and without that the gate cannot read it or write the receipt. `wind factory brief "<task>"` creates the session under the bot's identity, shares it, and prints the link; you then write the brief as a participant.

The gate and the receipt find the brief only through an explicit pointer: the intake marker on the issue, a `WS: session-…` trailer line in the PR description, or a `Wind Stream session: session-…` line. A session merely cited in prose is never treated as the brief.

## What runs where

| Verb | When | What it does |
|---|---|---|
| `intake` | every 30 min, or dispatched | `FACTORY:` sessions with a matching `Repo:` line become issues labelled `factory-build` + `sprint-ready`; it never launches |
| `dispatch` | right after intake, serialized per repository | claims one eligible issue (`factory-claimed`), launches it (`factory-building`), or records the failure (`factory-failed`); inert until armed |
| `launch <issue> [base] [agent]` | dispatch, triage, or dispatched by hand | `gh agent-task create` with the base from `base-branch` and the custom agent; refuses rather than land a PR on the wrong branch |
| `sweep` | every 10 min, as the factory login | a draft whose session died: `factory-budget` on a usage limit, else a resume nudge (three, then `factory-stuck`); a finished draft: ready + lift CI; red CI: a nudge with what broke (three, then stuck); green + gate OK: hold, no-human proof, one handoff per head; `resume` clears the stops |
| `gate-sweep` / `gate <pr>` | after the sweep, or dispatched | preflight the head, find the brief, attach the diff and the touched files, ask the brief session; the verdict is a PR comment and a `windstream-gate` commit status, scored for `/evals`; a BLOCK wakes the agent |
| `capture <pr> <event>` | on close | the receipt into the brief session, the verdict-quality and loop-outcome facts, and the executor trace read from the Copilot session log |
| `triage <run-id>` | any failed run of the named CI workflow on `main` or `release/*` (other branch names are not triaged) | a `FACTORY: CI failure` session, an issue, and a launch against the failing branch (under the arming switch) |
| `health`, `labels`, `charters`, `selftest` | by hand | credentials, labels, the eval charters, the self-tests |
| `brief-ref`, `base-branch`, `hold`, `loop-facts`, `scores-parse`, `eligible`, `no-human`, `agent-log`, `otel`, `mcp` | the scripts call each other; useful by hand | the lane's pure parts: the brief pointer rule, base-branch resolution, the hold, the eval facts, the scores block, dispatch selection, the no-human proof, the session-log reader, the OTLP emitter, the raw MCP client |

GitHub holds every workflow run that Copilot's own activity triggers behind *Approve workflows to run*, and an Actions-bot `@copilot` mention wakes nothing; the sweep acts as the factory login so a user identity does both.

## Turning it off

Each trigger can be disabled without a revert: `gh workflow disable windstream-gate.yml` (and intake, launch, ci-triage); `gh workflow enable` restores it. Disabling is repository-wide and takes effect at once for scheduled and dispatched runs. `FACTORY_DISPATCH_ENABLED` unset stops launches without stopping the gate.

## Known limits (tracked: windstreamai/wind#330, #331)

- The installed hooks diff and receipt against `origin/main`, while every lane PR bases on `FACTORY_BASE_BRANCH`; the default bot login (`windstreambot-one`) and the charter owner are baked into `sweep.sh` and `no-human.py`. These are the lane's own values, carried as shipped; fixes land upstream in windstream `scripts/factory/` and re-sync here.
- The hooks need the token in their own environment (the coding agent's `COPILOT_MCP_WINDSTREAM_TOKEN` env secret); wind's stored token serves verbs you run by hand, not the hooks.
- `triage` handles failed runs on `main` and `release/*` only.
- On Windows the verbs run under WSL; `install` from PowerShell is untested. On macOS, `intake` and `sweep` need Homebrew bash (4+) and coreutils (GNU `date`).
- After a wind upgrade, `install` moves the pin in a file whose template has not otherwise changed; a file written by an earlier wind from a since-changed template is reported `skipped` (#332 — provenance is the fix).

## Things that cost a run before they were understood

- **A PAT does not start the agent.** `gh agent-task create` accepts only a `gho_` token stored by `gh auth login`. The launcher logs the token's prefix so a wrong kind is obvious.
- **Assigning Copilot to an issue does not select a custom agent.** Only the launcher (or the UI dropdown) does.
- **The cloud agent's sandbox cannot reach Wind Stream.** Hooks anchor the session id and order `ws_fetch` first; anything that must call Wind Stream deterministically runs in a workflow.
- **A hook that errors is a deny.** Every hook prints a decision on every path and exits 0.
- **Wind Stream cannot see a private repository.** The gate attaches the diff and the full post-change contents of every touched file at the verified head; a missing path is reported, never substituted from another branch.
- **A session cited in prose is not the brief.** Only explicit pointers count; the first canary sat ungated and unreceipted until the gate said so on the PR.
