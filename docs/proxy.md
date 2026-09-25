# wind proxy

Shadow one service behind another and let Wind Stream score how far apart they are.

You have two versions of a service: the one you run today and the one that should replace it. Point your application at `wind proxy` instead. It answers every request from one of them (the **serve** upstream), quietly asks the other (the **shadow** upstream) the same question, and counts where they differ. Wind Stream scores the window on /evals as `<service>-window-parity`: how many calls diverged, how much of the traffic was actually compared, and whether enough calls were seen to mean anything.

## One command, one file

```
wind proxy init id3 --serve http://localhost:7001 --shadow http://localhost:7002 --route "GET /tags/:id"
wind proxy run
```

`init` registers the service with your organization and writes `wind-proxy.json`. `run` serves on `http://localhost:8787` (change `port` in the file). Point your app there.

```json
{
  "service": "id3",
  "serve": "http://localhost:7001",
  "shadow": "http://localhost:7002",
  "routes": ["GET /tags/:id"],
  "sample": 1,
  "window": "unwindowed-2026-09-16",
  "bars": { "coverage_min": 0.99, "volume_floor": 12 },
  "port": 8787
}
```

The rest of the file (policy, eval name, registration time) is stamped by the platform at `init`. Don't edit those by hand; run `wind proxy init --force` to re-register.

## What reaches Wind Stream, and what does not

Evidence contains counters, route templates (never concrete paths), and the names of fields that differed. Divergence fixtures and explicitly pushed sets can also contain request and response content when Wind Stream custody is enabled.

When two answers differ, the proxy keeps a **divergence fixture**: the request, both responses, and the list of fields that differed. Where that fixture goes is your organization's **custody policy**, set by your admin and applied automatically:

| Mode | Where the fixture goes |
|---|---|
| `windstream` | Your Wind Stream organization, as an asset you can attach to the Outcome that closes the divergence |
| `local` | A directory on the machine running the proxy (`./wind-proxy-divergences` unless the policy names one) |
| `none` | Nowhere; only the field names and counts are kept |

The proxy never loosens the policy on its own. If the policy says `windstream` and the platform is unreachable, the fixture is not written to disk as a fallback; the divergence still counts.

## Record the truth, then replay it

A parity window needs the same questions asked twice. A **set** holds them: a directory `<slug>-set/` with `manifest.json`, the `inputs/` the entries are about, and the `expected/` answers the truth gave. Record the service you trust, then replay the set against the candidate. Push and select an immutable platform version to have the platform score that replay like live traffic.

```
wind proxy record id3 --transport exec --command "winamp-tags {input}" --inputs ./tags/ --generator "WinAmp 5.666"
wind proxy replay id3-set --against http://localhost:7002 --no-evidence
```

Over http, `record` asks `--from` each line of a paths file (`GET /tags/abc` or just `/tags/abc`). Over exec, it runs the command once per input file. `wind proxy run --record id3` captures what a running proxy serves. Only `GET` and `HEAD` are recorded or replayed unless you pass `--unsafe`.

Replay verifies every payload's sha256 first and refuses a set that has been touched. Where it finds a difference it prints the fields side by side, expected against observed, with your organization's redaction rules applied, and writes the differing entries to `<slug>-divergences-set/` with what the candidate answered. Replay that set to reproduce them: each entry is asked again and reported as still diverging or converged. Record and replay beside the same `wind-proxy.json`, so both sides see the same redaction rules and route templates. A set recorded under redaction rules is compared as recorded; if you add a rule later, record again.

A plain paths file replays too. It carries no expectations, so `wind proxy replay smoke.paths --against http://localhost:8787` only drives the running proxy, which does the comparing. That is the clean-room window: after it flushes, /evals shows the result.

## Push, pull and replay an immutable version

`record` creates a local set. With Wind Stream custody enabled, push it to your
organization and use the returned set and version IDs for scored replay:

```sh
wind proxy push id3-set --config wind-proxy.json
wind proxy pull <set-id> --version <version-id> --out ./downloaded-id3
wind proxy replay --set <set-id> --version <version-id> --against http://localhost:7002 --service id3 --window release-check
```

Push requires a registered config and the current custody policy. It preserves
original manifest bytes and verifies payload digests. Scored platform replay also
requires a config so it can refresh the selected service's registration and policy. Rerun the same command after
an interrupted upload: stable request identities resume the same version, including
a lost finalization response. A changed policy revision requires a new upload under
the current policy. A finished upload is not human approval.

Pull writes to a new directory and refuses to overwrite an existing path. It checks
the manifest and every payload before writing. Replay also verifies everything
before candidate requests and records the exact version in evidence and evaluation
provenance. There is no `latest` selector. A concurrent upload cannot change the
version being replayed. Choose another registered `--service` and a fresh `--window`
to compare the same version against another service; this does not rewrite your
config's service.

Limits are 1 MiB for the manifest, 16 MiB per payload, 256 MiB total payload bytes,
1,000 entries and 3,000 payload paths. Absolute paths, traversal and symlinked local
payloads are refused. HTTP and exec sets retain their existing transport behavior.
Standalone local set replay uses `--no-evidence` when a config is present; publishing
set-backed evidence requires push followed by `--set` and `--version`.

## Evidence recovery

Each evidence batch has one frozen body and idempotency key. Network retries and
spool drains reuse both, so a lost successful response does not double the counters.
New traffic enters a separate batch. Proxy spool format 2 stores this identity with
the evidence in a private file; `wind proxy run` drains it for the configured host.

A conflict or malformed acknowledgement retains the evidence. Legacy format 1
entries have no safe retry identity: the CLI leaves them in place and warns you to
reconcile their delivery state before removal. Never assign them a fresh identity
without confirming whether they were accepted. Required evidence that remains
undelivered at shutdown produces exit status 2, including spooled evidence.


## Safe by default

Only `GET` and `HEAD` are shadowed unless `shadow_unsafe_methods` is `true` in the file, because replaying a write against a second system has side effects. The shadow never delays or changes a served response. If the shadow upstream is down, calls are still served; they just are not compared, and the coverage number on /evals says so.

## When nothing may touch disk

In the `none` custody mode the proxy keeps the most recent divergence fixtures in memory (two hundred by default) behind endpoints that answer only to the machine it runs on. `wind proxy replay --divergence <id> --against <url>` fetches one and reproduces it there and then, so a regulated deployment can still look at a divergence without a fixture ever being written.

## Divergences are yours to judge

The proxy counts and keeps evidence. Framing a divergence as an Outcome is a human act:

```
wind eval outcome frame id3-window-parity --statement "..." --exit-condition "..."
```

Closing it happens on the platform.
