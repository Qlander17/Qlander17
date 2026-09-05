# GhostOS: Exit Code Zero Is Not Mission Success

GhostOS marked an unattended mission `COMPLETE` after the agent reported that it could not finish. The Claude Code process had exited successfully. No crash. No escalation. The work had not been done.

That incident is the engineering problem this case study is about. Autonomous orchestration needs typed mission outcomes, durable authorization state, and recovery semantics — not a subprocess wrapper that infers success from `ok=True`.

The falsified assumption: **a clean process exit is a completed mission.**

This document covers the parts of GhostOS — a private, local-first agentic operating system — that are inspectable without exposing private business logic: unattended execution, permission posture, outcome classification, concurrency leases, and the failures that forced those mechanisms into existence.

---

## The incident

An unattended pipeline mission needed public financial filings from a government API. The permitted fetch tool could not set the identification header the source requires, so every request was rejected. The agent refused to fabricate a result, ended on an open question, and the process exited cleanly. The launcher recorded `COMPLETE`.

Root cause: `classify_terminal_status()` treated a successful subprocess exit plus an empty escalation list as completion. It did not read what the agent's final answer said, and it did not inspect Claude Code's structured `permission_denials`. Nothing had crashed, so nothing looked wrong.

A later mission produced the same false-`COMPLETE` through a different semantic failure. The agent wrote an in-scope plan, then stopped to ask whether to proceed — the right move for an interactive session, the wrong move for a launch that already constituted permission to execute in-scope work. Again: `ok=True`, zero escalations, no permission denial. The earlier `BLOCKED_EXTERNAL_ACCESS` detector could not see it, because no tool had been denied.

Two different unfinished missions, one shared error: **control-plane success is not task success.**

---

## Typed outcomes, not process supervision

**The unit of autonomous-agent supervision is verified state transition, not process termination.**

The launcher now maps a finished `claude -p` invocation onto an explicit terminal status before it returns control. The current set, from `tools/ghostos_mission.py`:

| Status | What produces it |
|---|---|
| `COMPLETE` | Process ok, no escalations, no hard-denied external-access attempt, and no unresolved plan-mode question |
| `COMPLETE_WITH_ESCALATIONS` | Process ok, but one or more consequential actions were filed as durable operator requests (`chairman_escalation_requests`) rather than executed |
| `WAITING_FOR_CHAIRMAN` | Process not ok, and at least one escalation was filed — the objective depends on a human decision |
| `BLOCKED_EXTERNAL_ACCESS` | Process ok, no escalations, but Claude Code recorded a hard-denied attempt to reach a non-localhost network resource |
| `BLOCKED_AWAITING_INPUT` | Structured `GHOSTOS_MISSION_OUTCOME: BLOCKED_AWAITING_INPUT` marker, or (fallback) an unresolved plan-mode question in the final text |
| `FAILED_SAFE` | Process not ok, no escalation — error, crash, or timeout |
| `INVARIANT_CONFLICT` | Mission text contradicts a hard operator invariant (`chairman_invariants`); the mission is never launched |

The structured marker is checked first. The agent running under `overnight-safe` is instructed to end with exactly one of:

```
GHOSTOS_MISSION_OUTCOME: COMPLETE
GHOSTOS_MISSION_OUTCOME: BLOCKED_AWAITING_INPUT: <one-line reason>
```

A `COMPLETE` marker does not override a hard-denied external-access attempt. A mission that says it is done, but was blocked on a denied fetch, is still `BLOCKED_EXTERNAL_ACCESS`. If the model emits no marker, classification falls through to structured denials, then to a narrow textual fallback (`"want me to proceed"` and similar phrases, only when the final text also ends in a question mark). That fallback exists because the historical Article 3 transcript predates the marker and would otherwise still classify as `COMPLETE`.

Every terminal status also requires `reports/missions/<mission-id>/final-report.md` to exist before control returns. If the agent did not write one, the launcher synthesizes a labeled fallback from status, checkpoints, result text, and escalation IDs.

This is still a heuristic over a finished subprocess, not a mid-tool-call pause. `claude -p` is atomic. Resumption re-invokes the same Claude session id after a wait; it does not continue from the denied instruction.

---

## Durable authorization is not an interactive prompt

The Claude agent decides *what* to do. A separate, deterministic layer decides *whether the named tool invocation is allowed*. The agent cannot add allow-list entries or switch profiles. What an allowed interpreter can do *inside* an auto-allowed `python3`/`node` call is a different boundary, taken up below.

Unattended missions run `claude -p --permission-mode dontAsk` against a hand-written profile (`overnight-safe`, `research-safe`, `development-safe`, `development-checkpoint`). `bypassPermissions` / `--dangerously-skip-permissions` is not used. `dontAsk` has no prompt surface: unmatched calls are skipped, not approved.

Every denied or allowed call resolves to one of three states:

- **AUTO-ALLOW** — an explicit allow-list entry. Runs immediately.
- **HARD-DENY** — out of scope for the profile. No escalation path. `git reset`, `sudo`, `rm`, credential-path `Read`/`Edit`, non-localhost `curl` as a Bash command, and similar.
- **HUMAN-ESCALATE** — denied by the profile, but listed in `ESCALATION_WORTHY_PATTERNS` (`git push`, `git commit`, `gh repo create/delete`, `gh api`, `npm publish`, `pip`/`npm install`, `sudo`, `chmod`, `chown`). The action does not run. A row is written to `chairman_escalation_requests` with justification, expected benefit, and material risks. Pending is not allowed. An escalation is not a dialog: the process continues, or reports that it is blocked, without waiting.

A second, independent `--allowedTools` / `--disallowedTools` layer on the CLI mirrors each profile's settings file, so a settings-parse mistake is not the only matcher.

Launch under `overnight-safe` injects an additional authorization into the system prompt: producing a plan and then executing in-scope work is already authorized by the launch. That text cannot override HARD-DENY or HUMAN-ESCALATE. Those remain enforced by the tool profile.

---

## Where enforcement actually happens — and where it does not

**Agent permissions must be reasoned about as reachable capabilities, not merely allowed command names.**

The safety claim that needs answering: **can an allowed Python interpreter reproduce behavior nominally denied by a provider command matcher?**

### DISCOVERED LIMITATION

**Yes, in part — and that was observed, not hypothesized.** The outer enforcement boundary at discovery time was Claude Code's command-string matcher (`dontAsk` + profile allow/deny + the `--allowedTools`/`--disallowedTools` echo), plus post-hoc classification of whatever denials Claude records. It was not an OS sandbox, not a network namespace, not seccomp, and not `execution_guard`.

`overnight-safe` auto-allows `Bash(python3 *)` and `Bash(node *)`. It denies `Bash(curl …)` except localhost GET, `Bash(wget *)`, `Bash(git push*)`, `Bash(rm *)`, and credential-path `Read`/`Edit`. The matcher sees the invocation string. It does not see what the interpreter does next.

That is not a hypothetical hole. After WebFetch could not set the `User-Agent` SEC.gov requires, and non-localhost `curl` was denied, the repair was `company-intelligence-pipeline/services/sec_edgar_client.py`: a stdlib `urllib.request` client invoked as a Python script. The profile was not loosened. Python was already allowed. `tools/unattended_ghostos.py` even lists `"http.client"` and `"urllib.request"` among external-fetch patterns — but those patterns only fire if the Bash command was *denied*. An allowed `python3` command never reaches them.

The same matcher-vs-interpreter gap applied to:

- `subprocess` from Python or Node invoking a binary whose *direct* Bash form is denied (`git push`, `curl`, `rm`)
- filesystem reads of paths denied to the `Read` tool (`.env`, `secrets/`, `~/.ssh`) via `open()`, or via allowed `Bash(cat *)` / `Bash(grep *)`
- nested `.env` / `credentials` files inside an otherwise-granted workspace tree
- network POST or arbitrary sockets, which the curl deny-list never sees

What the architecture already prevented, at the matcher layer:

- the agent invoking those denied commands *as Bash tool calls*
- unmatched tools running under `dontAsk` (fail closed: skip, do not approve)
- `Read`/`Edit` of listed credential paths through those tools
- HUMAN-ESCALATE actions executing without a durable queue row

`execution_guard` is a different control plane. It gates GhostOS-native external writes and spend (`controller/scheduler/runner.py`, Etsy publish, paid pipeline) against a global write lock and a spend ceiling. It does not wrap Claude Bash. ADR-0003 scopes it that way on purpose.

### THREAT MODEL

The unattended worker is not a hostile remote tenant. It is a local provider CLI that the operator also uses interactively, running a mission file under `dontAsk`. The relevant adversary is **transitive capability**: an allowed interpreter reconstructing a denied named invocation.

Two high-value reconstructions:

1. **Network.** `Bash(python3 *)` is allowed. Direct non-localhost `curl` is not. Python `urllib.request`, `socket.create_connection`, `subprocess curl`, and Node `net.connect` all sit outside the matcher.
2. **Secret files.** Landlock omit of `secrets/` and repo-root `.env` does not protect `pkg/deep/.env` if `pkg/` is granted `READ_FILE`. Landlock cannot subtract a filename pattern from an allowed tree.

Host constraints (measured, not assumed): AppArmor `kernel.apparmor_restrict_unprivileged_userns=1` blocks unshare/bwrap netns; systemd `--user` has only `cpu memory pids` controllers, so `PrivateNetwork=` / `IPAddressDeny=` are accepted and then silently inert. Landlock ABI 8 is available, including TCP bind/connect by port. Ubuntu security is not weakened to make a prettier sandbox.

### REMEDIATION

Capability policy (`controller/engines/capability_policy.py`) now names four interpreter TCP modes: `none`, `localhost_only`, `public_read`, `external_write`. Enforcement (`controller/engines/execution_sandbox.py`) is outside the interpreter:

- **Filesystem.** Recursive secret-free positive grants. Mixed directories are traverse-only; nested `.env`, `.env.*`, and `credentials` identities are omitted rather than denied after a parent grant. Landlock's inability to exclude by filename inside a granted tree is tested as a limitation, not papered over.
- **Network.** The provider CLI is not Landlock-net-restricted (it must reach its API). PATH-invoked `python3` / `node` / `curl` / `wget` are re-exec'd through a shim that stacks a second Landlock net layer. `overnight-safe` defaults to `none`. Research/SEC missions request `public_read` explicitly (`--network-mode public_read` or profile `research-public-read`).
- **Not used.** Deny-strings for `urllib` / `requests` / `socket`. bubblewrap. unprivileged netns. systemd IP filters. AppArmor changes.

`PUBLIC_READ` is TCP connect to ports 80 and 443. It is not HTTP-method filtering. POST over those ports still succeeds.

### REGRESSION TEST

`controller/engines/test_capability_boundary.py` attacks the sandboxed interpreter, not the matcher, with synthetic fixtures:

- Python direct HTTP (`urllib.request`) under `none` → blocked
- Python TCP socket under `none` → blocked
- Python `subprocess` curl under `none` → blocked
- Node `net.connect` under `none` → blocked
- Python `open` / `cat` / `grep` of nested `.env` → blocked
- `public_read` still blocks high-port TCP and allows 443
- Granting a parent tree with `READ_FILE` still leaks nested `.env` (Landlock limitation record)

### RESIDUAL RISK

- Absolute-path `/usr/bin/python3` spawned as the *first* exec from the unrestricted provider bypasses the PATH shim. Children of a PATH-invoked interpreter inherit the net layer.
- Landlock net is TCP-only. UDP and raw sockets are not covered.
- Landlock net is port-based, not IP-based. `localhost_only` is a local-service port class, not "destination 127.0.0.1".
- `PUBLIC_READ` does not distinguish GET from POST.
- Filename identities are not content inspection. A token in `config.yaml` is not omitted.
- Codex remains unwrapped by GhostOS Landlock (its targeted AppArmor profile needs userns for bwrap). It uses `--sandbox read-only` instead.
- The matcher layer is still present and still not a sandbox.

The claim this architecture now earns is **not** "sandboxed unattended worker". It is command policy plus Landlock filesystem allow-listing plus PATH-interpreter TCP modes. See `docs/transitive-capability-model.md` and `reports/capability-boundary-p0-hardening.md`.

---

## Cooperative leases, not mutual exclusion

Once an unattended mission and a foreground session shared the tree, path collision became a practical risk. `mission_workspace_leases` is a SQLite table of path-prefix claims. A mission declares a prefix in its spec; `ghostos_overnight.py` acquires it before launch and releases it on the way out. A conflicting active lease fails closed *before* the second mission starts.

The module states its own limit: enforcement is by convention. Writers are expected to call `check_path_conflict()` first. There is no OS file lock, no PID liveness check, no automatic stale-lease reap, and no distributed lock manager. Acquire is check-then-insert across separate connections — a time-of-check/time-of-use race remains possible.

This is single-host cooperative coordination among GhostOS writers. A non-cooperating process, a crash before release, or PID reuse is outside what the table can enforce.

Foreground discipline: treat an active lease as write-deferred for that prefix until release. Concurrent reads of ordinary workspace files are permitted. They are not a transactionally consistent multi-file snapshot, and they are not a grant to read credential paths. Mission logs under `reports/missions/<id>/` are append-mode for `transcript.jsonl`, `stdout.log`, and `stderr.log`; `status.json` is overwritten. Append-only is a write convention, not an immutable filesystem attribute.

---

## Recovery that had to be split

Rate-limit recovery and transient-server-error recovery were originally one bucket. A mission whose first API call failed with an overloaded-backend error was treated as a hard failure and abandoned, discarding work that a short retry would likely have finished.

They are now separate detectors in `tools/ghostos_overnight.py`:

- a usage-window rate limit waits with a clock-aware, capped backoff and resumes the same `--session-id`
- a transient 5xx/overloaded error uses a short exponential backoff, distinct from the rate-limit wait
- already-written checkpoints survive either retry

Session resume preserves conversational continuity across the gap; it is still a new subprocess, not a paused instruction pointer.

---

## Correct answers are not completion

A later, separate governed pipeline extended the same launcher discipline to multi-model missions: a builder model produces work, an independent reviewer model evaluates it, and a set of deterministic checks verify the resulting artifacts before the mission can close. One real run made both models right and the mission still not done.

The builder's analysis was complete and correct. The independent reviewer verified it in full and found the underlying judgment sound. Two required output files were fully authored — and staged at the wrong filesystem path, because a sandbox restriction blocked creating new files at the declared location. The reviewer's own verdict said so plainly, and still returned `REVISE`: the deterministic artifact-existence and hash checks had failed, and the arbitration is unconditional — a failed deterministic check overrides any reviewer verdict, regardless of how correct that verdict's own reasoning was. The mission ended `FAILED_SAFE`, exactly as designed.

The falsified assumption here is a second-order version of the first one in this document: not "process exit implies success," but **correct judgment implies completion.** Neither a person's nor a model's assessment that work is right is itself the state a governed pipeline is contracted to produce. Completion requires the artifacts to exist, at the declared paths, matching the declared hashes — verified mechanically, not argued for.

The same discipline applies one layer earlier. A separate incident: a reviewer call returned a genuine success response from the provider's own API — not a timeout, not an error — but the provider's own explicit answer field was empty. Evidence-binding refused to treat an empty result as a reviewable judgment and failed closed rather than let a downstream stage infer one. Transport success, model evaluation, a usable response, artifact production, and mission completion are five different states; a system that collapses any two of them will eventually report done on work that only reached the third.

**What would falsify this:** a governed pipeline that let a reviewer's qualitative override commute a failed deterministic check — if "the reviewer says it's fine" were ever sufficient to close a mission over a failed artifact check, this principle would be wrong. It is not sufficient here, on purpose (`controller/engines/test_review_arbitration.py`).

**The cost, stated plainly:** this trades recoverability for strictness. A mission whose substantive work is entirely done can sit terminated, pending a purely mechanical fix, rather than auto-resolving itself — a real, disclosed tradeoff, not a hidden one.

---

## Incident table

| Symptom | False assumption | Invariant introduced | Regression artifact | Residual risk |
|---|---|---|---|---|
| Mission exited 0; launcher wrote `COMPLETE`; agent had hit a denied external fetch and stopped on a question | Clean process exit + empty escalation list = finished work | Inspect Claude Code `permission_denials`; `BLOCKED_EXTERNAL_ACCESS` beats `ok=True` | `tools/test_ghostos_mission.py` (denial-payload classification) | A later-recovered denial can still flag the mission blocked — accepted, conservative |
| Mission wrote a plan, asked “want me to proceed?”, exited 0, classified `COMPLETE` | Interactive plan-mode rules are safe to reuse unattended | Launch under `overnight-safe` authorizes in-scope execution; `GHOSTOS_MISSION_OUTCOME` marker; `BLOCKED_AWAITING_INPUT` | `test_unattended_ghostos.py` marker/fallback tests; historical Article 3 transcript as the fallback fixture | Marker is cooperative; a model that neither emits it nor matches the narrow phrase list can still look complete |
| First-call overloaded error abandoned the mission | Rate-limit wait and transient 5xx are the same failure | Separate detectors; short exponential backoff for transient; checkpoints survive retry | overnight supervisor tests around `compute_transient_backoff_delay` | Classification depends on the provider's error shape remaining stable |
| Fresh-shell launch crashed `ModuleNotFoundError` despite packages being installed | Resolved-path identity (`Path.resolve()`) identifies the venv interpreter | Compare `sys.prefix`; `exec` the unresolved `.venv/bin/python3` symlink | `tools/test_venv_bootstrap.py` (`test_symlinked_venv_python_is_never_resolved_before_exec`) | Any future bootstrap that resolves the symlink first will silently undo venv activation |
| SEC EDGAR returned 403 to every permitted fetch | WebFetch is a general HTTP client | Policy-compliant stdlib client, invoked as allowed `python3`, without widening curl | `company-intelligence-pipeline/services/test_sec_edgar_client.py` | Historical exhibit of the interpreter gap. Later closed for PATH-invoked python3 by Landlock net mode `none`; SEC-style fetches now require explicit `public_read` |
| SEC EDGAR 10-K filing metadata (`form`/`fp`) mislabels some quarterly figures as annual | Filing-metadata tags (`form='10-K'`, `fp='FY'`) reliably describe fact duration | Duration-based (not label-based) validation: filter on real `period_start`/`period_end` span (350–380 days) rather than trusting `fp`/`form` alone | `company-intelligence-pipeline/README.md`, `company-intelligence-pipeline/analytics/00_views.sql` | Cross-filing restatement reconciliation is still a separate, unimplemented concern |
| A governed mission's builder and independent reviewer were both substantively correct, but two outputs were staged at the wrong path; the mission still ended `FAILED_SAFE` | A correct reviewer verdict is sufficient for completion | Deterministic artifact/hash checks arbitrate unconditionally over any reviewer verdict | `controller/engines/test_review_arbitration.py` | A fully correct mission can sit terminated pending a purely mechanical fix — accepted, disclosed |

---

## What this does not claim

No performance or scale numbers. This is a single-operator, single-host system.

No claim of a fully sandboxed unattended worker. Named-tool policy still bounds invocations, not every syscall. PATH-invoked python3/node/curl/wget under `none` cannot make outbound TCP; absolute-path first-exec from the provider, UDP, and HTTP POST on 80/443 under `public_read` remain residual. Nested `.env` identities are omitted from the Landlock allow-list; Landlock still cannot exclude a filename inside a granted tree.

No claim that leases exclude arbitrary writers, that mission logs are tamper-evident, or that concurrent readers see a consistent snapshot.

No claim that typed outcomes recover true agent intent. They recover structured signals and a conservative fallback. The private scoring, underwriting, and CRM logic this architecture happens to sit under is out of scope here.

The tests that lock these behaviors live next to the modules named above. Counts change as the suites grow; the invariant is that permission-denial handling is tested against captured denial payloads, the venv bug against an actual symlink layout, and false-`COMPLETE` against the transcript shape that produced it.
