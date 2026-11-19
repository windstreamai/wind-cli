# Connect your coding agent to Wind Stream

Use this guide in Claude Code or Codex, in the person's current repository.
Connect using their Wind Stream identity and approved organization. Stop after
connection verification; creating a Brief or starting a build is a separate task.
Keep passwords, tokens, and credential files out of the conversation.

## 1. Check the existing setup

Identify the current coding agent and check for `wind`. Inspect the agent's Wind
MCP connection and any prepared-worktree binding without exposing credentials.
If the existing connection already works for the intended organization, keep it
and proceed to the native-agent check in step 5. Do not overwrite a custom MCP
entry, change another agent, or rebind existing work automatically.

## 2. Install Wind if needed

On macOS or Linux with Homebrew:

```sh
brew install windstreamai/tap/wind
```

Use `brew upgrade windstreamai/tap/wind` for an existing Homebrew installation.
For other installations, use the official instructions and release checksums:
https://github.com/windstreamai/wind-cli#install

Check `wind onboard --help` and `wind setup --help`. This flow requires the v0.5.0
onboarding commands and setup's paired `--connection` / `--organization` flags.
If the installed release lacks them, report the update requirement and stop.
Do not substitute token pasting or an unverified source build.

## 3. Get browser approval

Use production unless the person explicitly chose another deployment. From an
unbound directory, start a resumable operation:

```sh
wind onboard start --host https://app.windstream.ai --json
```

Save the returned operation ID. Show the returned `verification_uri` and
`user_code` to the person. They sign in and approve their organization in the
browser. If they need an account first, send them to
https://app.windstream.ai/signup; existing members can use
https://app.windstream.ai/login. Let them handle agreements, billing and consent.
Respect invitation, enrollment, and plan requirements.

After approval, resume the same operation, replacing OPERATION_ID:

```sh
wind onboard resume OPERATION_ID --host https://app.windstream.ai --json
```

When pending, wait at least `retry_after_seconds` before resuming. When
`organization_required`, have the person choose from the returned organizations
and resume with `--organization APPROVED_ORG_ID`. Never choose one arbitrarily.
An `authorized` response needs another resume for live MCP verification.

For denial, stop. For expiry, ask the person to approve a new operation. For
`uncertain`, have them review grants in Access before starting again; do not replay
an uncertain exchange. A PAT override or conflicting worktree pin is a blocker,
not permission to change authority. Never print the override's value.

## 4. Configure this coding agent

A `connected` response includes the approved connection and organization IDs.
Select the current agent and configure those exact IDs:

```sh
wind agent claude
# For Codex: wind agent codex
WIND_HOST=https://app.windstream.ai wind setup --connection CONNECTION_ID --organization ORGANIZATION_ID
```

In PowerShell, set `$env:WIND_HOST` for setup and restore it afterward. Keep the
same explicitly selected deployment throughout the task. This setup changes the
selected agent integration and Wind agent preference; it does not change OAuth
defaults or worktree bindings. Credentials stay in Wind's protected store.

Read setup's warnings: exit code zero alone is not success. Preserve conflicting
registrations and explain the supported repair before replacing anything. In
Codex, have the person review/trust Wind hooks from `/hooks` when requested.
Additional PAT-based workflows are not prerequisites for this connection task.

## 5. Verify through the agent

For a newly created connection, resume the operation once more and require fresh
MCP evidence (`verification.cached` is false) for the expected organization.
Then use the coding agent's native MCP connection to discover Wind tools and
make a fresh read-only request through the same connection, organization and
host. Do not create or upload work merely to test access.

An existing connection may have no onboarding operation ID; its fresh native MCP
request provides the live proof. Cached tool names or a saved registration do not.
If the host must reopen to load MCP changes, report configuration complete but
verification pending, and give the exact continuation step. Do not reauthorize
just to obtain an operation ID.

Report the connected agent, deployment, approved organization, and successful
check. If blocked, state what remains and the next action. Let the person choose
their first task after the connection works.
