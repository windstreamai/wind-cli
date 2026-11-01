# wind factory

A Wind Stream brief becomes a gated, evidenced pull request handed to a reviewer, with no one in the loop until the review.

You decide what to build in a Wind Stream session titled `FACTORY: …`, or you label an existing issue `factory-build` and `sprint-ready` (with its session on a visible `Wind Stream session: session-…` line). The factory takes it from there: a GitHub issue, a Copilot cloud agent started against the repository's `factory-build` branch, CI lifted and the agent nudged when it stalls, a verdict from the brief session on every pushed head (`VERDICT: OK` or `VERDICT: BLOCK — reason`, posted as `@copilot` so the agent reconciles), and finally one handoff comment per green, gate-OK head: the hold table of the checks the brief names, the no-human proof from GitHub's own event record, the `factory-ready` label, and a review request. A person merges. The receipt and the executor trace land in the session on close. The session is the ledger: brief, verdicts, and receipt all live in it.

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
   - `FACTORY_ASSIGN_TOKEN` (Actions secret): the bot's `gho_…` token from `gh auth login` on a machine signed in as the bot (`gh secret set FACTORY_ASSIGN_TOKEN --body "$(gh auth token)"`). `gh agent-task` refuses personal access tokens of either kind; this account's name appears on everything the factory writes: launches, issues, verdicts, nudges, and handoffs. It is also the only account whose markers the factory believes (see [Who the factory believes](#who-the-factory-believes)).
   - `COPILOT_MCP_WINDSTREAM_TOKEN`: Repo → Settings → Copilot → Coding agent → environment secret, beside the pasted `.github/copilot/mcp-servers.json`. The cloud agent does not read a project `.mcp.json`, and the install writes none: a project-scope `windstream` server shadows the one you configured at user scope.
3. `wind factory labels <owner/repo>` creates the labels: `factory`, `groomed`, `needs-decision`, `factory-build`, `sprint-ready`, `factory-claimed`, `factory-building`, `factory-failed`, `factory-ready`, `factory-stuck`, `factory-budget`. It does not create `ci-failure`, which triage files issues under: `gh label create ci-failure`.
4. Set the variables: `FACTORY_SHARE_IDS` and `FACTORY_MENTION_IDS` (Wind Stream user ids the factory shares its sessions with and mentions on every turn), `FACTORY_REVIEWERS` (GitHub logins the handoff requests review from).
5. Create `factory-build` from the default branch, synchronize the open release train into it, protect it with the repository's required test check, then set `FACTORY_BASE_BRANCH=factory-build`. Every launch targets it; intake, gate, and triage check it out. Nothing is guessed from branch names, and a missing branch stops a launch.
6. Keep `FACTORY_DISPATCH_ENABLED` unset through bootstrap. Set `FACTORY_CANARY_ISSUES` to an ordered list of issue numbers and `FACTORY_DISPATCH_LIMIT=1`. A non-empty canary list is a strict allow-list, so nothing else can launch. Arm with `FACTORY_DISPATCH_ENABLED=true` once the canaries, the required checks, and `wind factory health` are green; clear the canary list only after the sequence is green.
7. Optional telemetry: `WINDSTREAM_OTLP_URL=https://app.windstream.ai/api/v1/otlp`. Every factory job then emits one OpenTelemetry span; the gate adds the judge's scores and capture adds verdict-quality and loop-outcome facts, shown on `/evals` as the `factory` composite. Unset is the kill switch: no telemetry, no failure. A repository that authors its own eval charters under `api/evals/charters/` publishes them once per environment with `wind factory charters > f.ndjson && wind eval publish f.ndjson`.
8. Name your CI workflow: `workflows:` in `windstream-ci-triage.yml` and the variable `FACTORY_CI_WORKFLOW` the sweep lifts and compares against (both default `CI`).

`wind factory health` checks both credentials and distinguishes the failures (by hand it needs `FACTORY_ASSIGN_TOKEN` exported too, else it reports degraded; `WIND_TOKEN` or your stored wind token stands in for `WINDSTREAM_MCP_TOKEN`, and is then sent to the host `WIND_HOST` names, like every other wind command): exit 0 connected, 1 token expired, 2 unreachable, 3 MCP token unset, 4 degraded (MCP fine, but `FACTORY_ASSIGN_TOKEN` missing or not `gho_`, which silently disables gate auto-reconcile and cloud-agent launch, and leaves the gate signing with the workflow token). `wind factory selftest` runs the lane's seventeen self-tests.

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

The gate and the receipt find the brief only through an explicit pointer: the intake marker on the issue, a `WS: session-…` trailer line in the PR description, or a `Wind Stream session: session-…` line. A session merely cited in prose is never treated as the brief. A pointer also has to come from someone the factory believes, because the gate sends the diff into that session and publishes the reply, and the receipt is written there, all as the factory's identity: see [Who the factory believes](#who-the-factory-believes).

## What runs where

| Verb | When | What it does |
|---|---|---|
| `intake` | every 30 min, or dispatched | `FACTORY:` sessions with a matching `Repo:` line become issues labelled `factory-build` + `sprint-ready`; it never launches |
| `dispatch` | right after intake, serialized per repository | claims one eligible issue (`factory-claimed`), launches it (`factory-building`), or records the failure (`factory-failed`); inert until armed |
| `launch <issue> [base] [agent]` | dispatch, triage, or dispatched by hand | `gh agent-task create` with the base from `base-branch` and the custom agent; refuses rather than land a PR on the wrong branch |
| `sweep` | every 10 min, as the factory login | a draft whose session died: `factory-budget` on a usage limit, else a resume nudge (three, then `factory-stuck`); a finished draft: ready + lift CI (not for a head that changes workflow files: a person approves that run); red CI: a nudge with what broke (three, then stuck); green + gate OK: hold, no-human proof, one handoff per head; `resume` clears the stops |
| `gate-sweep` / `gate <pr>` | after the sweep, or dispatched | for every open, non-draft PR of this repository whose head the factory has not judged: preflight the head, find the brief, attach the diff between the PR's base commit and that head and the touched files, ask the brief session; the verdict is a PR comment signed by the factory login and a `windstream-gate` commit status, scored for `/evals`; a BLOCK wakes the agent |
| `capture <pr> <event>` | on close of a same-repository PR | the receipt into the brief session, the verdict-quality and loop-outcome facts, and the executor trace read from the Copilot session log |
| `triage <run-id>` | a failed run of the named CI workflow on this repository's own `main` or `release/*` (a pull request's run, a fork's run, and other branch names are not triaged) | a `FACTORY: CI failure` session, an issue, and a launch against the failing branch (under the arming switch) |
| `health`, `labels`, `charters`, `selftest` | by hand | credentials, labels, the eval charters, the self-tests |
| `brief-ref`, `base-branch`, `hold`, `loop-facts`, `scores-parse`, `eligible`, `no-human`, `agent-log`, `otel`, `mcp` | the scripts call each other; useful by hand | the lane's pure parts: the brief pointer rule, base-branch resolution, the hold, the eval facts, the scores block, dispatch selection, the no-human proof, the session-log reader, the OTLP emitter, the raw MCP client |

GitHub holds every workflow run that Copilot's own activity triggers behind *Approve workflows to run*, and an Actions-bot `@copilot` mention wakes nothing; the sweep acts as the factory login so a user identity does both.

## Who the factory believes

The factory holds secrets and acts under its own identity on what it reads from GitHub, so five rules fix whose word it takes and what it will run. `trust.py` owns them, and every script reads through it.

- **Who signs.** With its own token (`FACTORY_ASSIGN_TOKEN`) the factory posts every marker comment and files every issue as that account, and only that account is believed. The workflow token is not enough to be the factory: every workflow run in the repository holds one, CI on a pull request's own head included, and they all post as `github-actions[bot]`. A factory without a token of its own can only post as `github-actions[bot]`, so that is who it believes then. That footing is weaker, and it is one more reason to give the factory its bot account.
- **Markers.** The factory keeps its state in marker comments: the gate's verdict, the handoff, the nudges, the stops. A marker counts only as the last thing in a comment the factory wrote, which is where the factory puts it. Typed by anyone else it changes nothing, and so does one that a comment of the factory's merely quotes (a model's reply, a check's name, a person's words in the handoff table): the gate still judges that head, and the handoff still waits for the gate. The verdict is the one in the gate's own heading, which the gate takes from the judge's first line and nowhere else. A reply that does not open with a verdict records none, and the next sweep asks again, three times at most; after that the gate says so on the pull request and stops asking until a push or a gate by hand. A verdict is for one head and one base branch: a pull request retargeted since is judged again. Text the factory quotes in its own comments (a reply, a check's name, a base branch an issue names, a person's words) cannot mention anyone: an `@name` in it is defused, so nothing quoted wakes the agent. A brief and a CI log reach the issue as written, since the agent reads them; the log sits in a code block, where a mention is not one.
- **Whose pull requests.** The sweep acts on pull requests the Copilot coding agent opened from the repository itself, matched by exact login and GitHub's bot flag. The gate judges same-repository pull requests only, and judges the diff between the two commits it names, so the verdict it signs for a head is about that head. When the diff or the touched files are longer than what the judge is shown, the judge is told and the verdict comment says it is partial. A pull request from a fork is outside the factory: never gated, never receipted.
- **The brief pointer.** Whoever writes a text chooses the session it names, and the gate sends the diff there as the factory. So: the factory's own intake marker, on a linked issue (`#n` in the description) that the factory filed, counts for any same-repository pull request. When the coding agent or the factory login opened the pull request, its own visible pointer counts next (a `WS:` trailer or a `Wind Stream session:` line), then the visible pointer of an issue it links (the same lines, or the issue form's *Wind Stream session* field). A hidden marker is only ever the factory's: one in an issue a person filed, or quoted into a pull request's description, is invisible to whoever labelled or reviewed that text, so it is not a pointer. Visible means a line of its own that a reader of the page sees without expanding or reading a quotation. That is not guessed from the markdown: the factory reads GitHub's own rendering of the body and counts only text inside elements known to display it (paragraphs, headings, lists, markdown tables, plain inline markup), and not under more than two nested elements that each shrink it. What is collapsed, quoted, a code block, an embedded snippet, or not shown by a browser at all is left out by default, and a pointer in a comment, a tag's attribute, an image, or a link definition is simply not in the rendering. A launch reads an issue the same way, and its `Base branch:` line under the same rule. The issue on which the factory announced a pull request's launch is that pull request's brief, whatever the pull request says it closes; a linked pull request is not an issue. When several factory-filed issues are linked and the factory announced the launch on none of them, none is the brief. The hold and the no-human proof read that same issue. A person's pull request is therefore gated only when it links an issue the factory filed, and an issue you file by hand needs its session on a visible line.
- **What it runs on its own authority.** GitHub holds the workflow runs an agent's push triggers behind *Approve workflows to run*, because the pull request may have changed the workflows themselves, and an approved run executes the pull request's version of them with the repository's secrets. The sweep approves those runs for the agent's pull requests, but not for a commit that changes anything under `.github/workflows/` or `.github/actions/`, an `action.yml` anywhere, or any file under a directory a workflow at that commit `uses: ./…` (a local action is its whole directory, not only its manifest; when those workflows cannot be read, the run stays held); it asks about the very commit whose runs it would approve, not about the pull request as it stands by then, and when it cannot read what the commit changes it does not guess. Either way it says so once on the pull request, and a person reads the change and approves the run under Actions. The factory continues from there. A repository whose CI holds no secrets can set `FACTORY_LIFT_WORKFLOW_CHANGES=true` to lift those too. CI-triage briefs often ask for a workflow change, so expect those pull requests to wait for that approval.

Two more reads follow from the pointer. The hold takes the brief's `Checks:` from the issue that carries the pull request's brief, not from whichever issue the pull request says it closes, so an agent that links a different issue, or none, does not choose its own acceptance checks. And an issue counts as "already filed", for intake and for triage, only when the factory filed it: an issue anyone opened with the right words does not block a brief or silence triage for a branch.

CI triage follows the same line: it acts on failed runs of this repository's own `main` and plainly named `release/*` branches, never on a pull request's run, because the branch name a run reports is whatever its pusher called the branch.

What this rests on: the bot account's tokens, and write access to the repository. Anyone who can push a workflow change that a person then approves, or who holds the bot's token, is the factory. Keep outside contributors on forks, keep `FACTORY_ASSIGN_TOKEN` out of every workflow but the factory's four, and treat it as you would a maintainer's credentials.

## Turning it off

Each trigger can be disabled without a revert: `gh workflow disable windstream-gate.yml` (and intake, launch, ci-triage); `gh workflow enable` restores it. Disabling is repository-wide and takes effect at once for scheduled and dispatched runs. `FACTORY_DISPATCH_ENABLED` unset stops launches without stopping the gate.

## Known limits (tracked: windstreamai/wind#330, #331)

- The installed hooks diff and receipt against `origin/main`, while every lane PR bases on `FACTORY_BASE_BRANCH`; run by hand, `wind factory no-human` needs `FACTORY_ASSIGN_LOGIN` to know which account is the factory's (the sweep always passes the token's own login; no built-in name is assumed). Whenever `FACTORY_ASSIGN_TOKEN` is set, the account the factory believes is the token's own as GitHub names it; `FACTORY_ASSIGN_LOGIN` stands in only when GitHub cannot be asked. Fixes to the lane land upstream in windstream `scripts/factory/` and re-sync here.
- The sweep lifts the *Approve workflows to run* hold on the agent's pull requests so their CI runs without a person. A lifted run executes the pull request's code (its tests, its build) with whatever the CI workflow hands its steps, so keep secrets the build does not need out of the CI workflow. Only the workflow definitions themselves are kept for a person's approval.
- The bash guard hook denies obviously destructive commands. It is a guard against accidents, not a sandbox: a determined command line gets past any pattern list.
- The hooks need the token in their own environment (the coding agent's `COPILOT_MCP_WINDSTREAM_TOKEN` env secret); wind's stored token serves verbs you run by hand, not the hooks.
- `triage` handles failed runs on this repository's own `main` and `release/*` only; a repository whose CI runs only on pull requests gets no triage.
- On Windows the verbs run under WSL; `install` from PowerShell is untested. On macOS, `intake` and `sweep` need Homebrew bash (4+) and coreutils (GNU `date`).
- After a wind upgrade, `install` moves the pin in a file whose template has not otherwise changed; a file written by an earlier wind from a since-changed template is reported `skipped` (#332 — provenance is the fix). The four workflows changed in 0.4.1 (inputs reach the shell through variables, `gh` is checksum-verified before it is installed, capture and triage refuse forks), so a repository installed by 0.4.0 takes them with `wind factory install --force`, then re-applies its own edits, such as the triage trigger's `workflows:` list.

## Things that cost a run before they were understood

- **A PAT does not start the agent.** `gh agent-task create` accepts only a `gho_` token stored by `gh auth login`. The launcher logs the token's prefix so a wrong kind is obvious.
- **Assigning Copilot to an issue does not select a custom agent.** Only the launcher (or the UI dropdown) does.
- **The cloud agent's sandbox cannot reach Wind Stream.** Hooks anchor the session id and order `ws_fetch` first; anything that must call Wind Stream deterministically runs in a workflow.
- **A hook that errors is a deny.** Every hook prints a decision on every path and exits 0.
- **Wind Stream cannot see a private repository.** The gate attaches the diff and the full post-change contents of every touched file at the verified head; a missing path is reported, never substituted from another branch.
- **A session cited in prose is not the brief.** Only explicit pointers count; the first canary sat ungated and unreceipted until the gate said so on the PR.
