# GhostOS: A Governed, Autonomous AI Operations Case Study

A real engineering case study of the parts of GhostOS — a private, local-first agentic operating
system — that are genuinely interesting independent of the private business logic they support:
running AI-driven work unattended, safely, for hours at a time, with no human in the loop, and
recovering from the real failures that happen along the way.

## The problem

An AI coding agent (Claude Code) can do real, substantial work — but by default it stops and asks a
human before almost every consequential action. That's correct behavior for a supervised session,
and wrong behavior for work meant to run overnight: a permission prompt with nobody watching just
means the work silently stalls. The naive fix — `bypassPermissions`, skip all prompts — trades one
failure mode for a much worse one: an unattended agent with no safety rails at all.

The real engineering problem: run genuinely unattended AI work for hours, without ever removing the
safety boundary a human-supervised session has, and be honest — not silent — about every real
failure mode that shows up along the way.

## Architecture

### The deterministic/AI boundary

The Claude agent decides *what* to do; a separate, deterministic Python layer decides *whether it's
allowed*. The agent never has a way to grant itself permission — every tool call passes through a
permission profile the agent doesn't control. This is `dontAsk` mode plus a real, hand-written
allow/deny list per profile (`overnight-safe`, `research-safe`, etc.), not the agent's own judgment
about what's safe.

### The three-state permission model

Every tool call resolves to exactly one of three states:

- **AUTO-ALLOW** — an explicit, bounded allow-list entry (e.g. `Bash(python3 *)`, `Bash(git status*)`,
  `WebFetch`). Runs immediately, no human involved.
- **HARD-DENY** — out of scope for this profile, full stop. No escalation path, no workaround. Most
  denials are this: an unattended mission that tries `curl` to an external host, or `git push`,
  simply can't, by design.
- **HUMAN-ESCALATE** — a real, narrow set of actions (currently: `git push`, `git commit`,
  `gh repo create/delete`, `gh api`, `npm publish`, `pip`/`npm install`, `sudo`, `chmod`, `chown`) that
  are legitimate but consequential enough to need a real decision. These don't vanish into a denial —
  they become a durable, structured request in a database-backed queue, with a real justification,
  expected benefit, and material risks recorded, and the mission continues (or reports honestly that
  it's blocked) rather than hanging.

### Fail-closed by construction

The distinction that matters: a **hard-deny** and a **not-yet-decided escalation** are never treated
the same. An escalation-worthy action that hasn't been approved yet is still blocked — "pending" is
not "allowed." And an escalation is explicitly *not* an interactive dialog: it's written to durable
storage and the process continues without waiting for an answer, so one blocked branch never stalls
an entire multi-hour mission.

### Mission architecture, resumability, and rate-limit recovery

A supervisor process (`ghostos_overnight.py`) wraps each real Claude Code invocation: launch, run,
checkpoint, detect a rate limit if one occurs, persist state, wait, and resume the *same* Claude
session (`--session-id`/`--resume`) rather than starting over — verified empirically to preserve real
conversational continuity across the gap, not just restart cold. A mission that hits a real usage
limit mid-task doesn't fail; it waits with a bounded, capped backoff and picks the same conversation
back up once the limit resets.

### Durable, per-mission reporting

Every mission gets its own directory (`reports/missions/<mission-id>/`) with append-only logs
(`transcript.jsonl`, `stdout.log`, `stderr.log`), a real status snapshot, a checkpoint history, and a
final report — durable and UI-independent, so a mission's real record survives past any one terminal
window or chat session.

### Concurrency: resource leases, and a real foreground/background model

Once more than one unattended mission (or an unattended mission alongside a live foreground session)
became routine, file-level collision became a real risk. A small, path-prefix lease primitive lets a
mission declare which part of the filesystem it owns (a one-line marker in its own spec file); a
second mission or session checking for a conflicting active lease fails closed *before* it starts,
rather than racing the first mission's writes. Deliberately not a distributed lock manager — a
real, small, reviewable table, not new infrastructure.

This isn't just a design principle — it's the actual working pattern for how a human-driven
foreground session and an unattended background mission share this codebase day to day. A real,
in-use discipline: before touching a path an unattended mission might own, the foreground session
checks the real, current lease table (not a cached assumption) and treats an active lease as
read-only for that path until the mission finishes; once a mission completes and its lease releases,
foreground work resumes freely there. Reading is always safe; writing waits.

### Pre-authorized missions and structured outcomes

An unattended mission and a supervised interactive session don't actually need the same permission
posture for *planning* — a real problem this system originally got wrong. A mission would correctly
recognize a task needed a plan before implementation (following the same discipline an interactive
session follows), write a real, specific plan, and then stop to ask "want me to proceed on this
basis?" — exactly right for a human present to answer, and exactly wrong for an unattended run with
nobody there. The fix: launching a mission under the unattended profile now carries its own explicit,
injected authorization that launch itself already constitutes permission to move from plan into
execution for in-scope work — without touching the separate, much narrower set of actions (a public
push, an account mutation, external contact) that still always require a real, human-decided
escalation regardless of execution mode.

Closing that gap also needed a second, structural piece: a way for the launcher to actually tell the
difference between "genuinely stuck, correctly asking a real question" and "just being appropriately
cautious about something the launch itself already authorized." The mission is now asked to end with
a small, structured, machine-readable outcome marker rather than leaving the launcher to guess from
prose — checked first, before any older, fuzzier signal, with a narrow, deliberately conservative
text-pattern fallback for a mission that doesn't emit the marker at all.

### Transient failure recovery, distinct from a real usage limit

A rate limit (a real usage-window reset) and a transient server error (an overloaded backend that
will very likely succeed on retry moments later) are different problems needing different handling,
not one bucket. A real mission lost real, substantial completed work to exactly this conflation once:
a transient server error on the very first call, before any actual work began, was treated the same
as a hard failure and the mission gave up immediately rather than retrying — discarding a mission that
would very likely have succeeded a few minutes later. The fix: a separate, explicit detector for this
class of failure, with its own short, genuinely exponential backoff (distinct from a rate limit's
own longer, clock-time-aware wait), so a transient hiccup gets a real, bounded number of quick retries
before anything is given up on — and a real, added guarantee that any real work already durably
checkpointed before a transient failure survives the retry, never silently discarded.

## Real failures found and fixed (not invented, not idealized)

**The permission-prompt problem.** The starting problem this whole architecture exists to solve:
Claude Code defaults to asking a human before nearly every consequential action. Solved by building
the `dontAsk`/profile-based system above — not by disabling safety, by making safety declarative and
deterministic instead of interactive.

**A virtualenv-launcher failure.** A production script crashed with `ModuleNotFoundError` when run
from a fresh, non-activated shell — a dependency that was genuinely installed, just not on the
interpreter actually being used. Root cause, found by direct reproduction: `.venv/bin/python3` on
this machine is a *symlink* to the system Python, and the original venv-detection code compared
*resolved* paths — which silently defeats Python's actual venv-activation mechanism (confirmed
directly: `sys.prefix` differs depending on whether the *unresolved* symlink path or its resolved
target is what actually gets executed). Fixed by checking `sys.prefix` and always executing the
unresolved path. A regression test reproduces the exact symlink scenario so this can't silently come
back.

**A false-"COMPLETE" status bug.** A real unattended mission hit a hard external-access block (see
below), correctly refused to fabricate a result, and ended by asking an open question — and the
launcher recorded it as `COMPLETE` anyway. Root cause: the terminal-status classifier only checked
whether the underlying process exited cleanly and whether any formal escalation had been filed —
both looked "clean" here purely because nothing crashed, with zero awareness of what the agent's own
final answer actually said. Fixed with a new, *structurally* detected status (reading Claude Code's
own real permission-denial data, not string-matching the agent's prose) for exactly this case, plus a
launcher-level guarantee that a real final report always exists before a mission returns control —
previously, a report only existed if the agent itself remembered to write one.

**A real external-access blocker, root-caused rather than routed around.** A pipeline mission needed
to fetch public financial filings from a government API that requires a compliant identification
header; the tool available for external fetches inside the sandbox can't set custom headers, so every
request was correctly rejected by the source server. Investigated against the API's own published
access policy rather than assumed; fixed with a small, dependency-free HTTP client built specifically
to satisfy that policy (proper identification, real rate limiting, retry/backoff) — invoked as a
plain script rather than by loosening the sandbox's own access rules, which stayed exactly as
restrictive as before.

**A filing-metadata bug, found by the system itself.** Once real external data started flowing
through a pipeline, a subtler bug surfaced: a metadata field that looked like it reliably marked an
annual figure did not — some values tagged as annual were quietly quarterly numbers, which would
have silently corrupted every multi-year trend calculated downstream. Found by validating each
record's own real date range instead of trusting its label, fixed with duration-based filtering, and
locked in with a regression test.

**A second, different false-"COMPLETE" — a mission correctly stalling on a question nobody could
answer.** A real, later mission correctly followed the interactive-session plan-first discipline: it
wrote a real, specific, in-scope plan and ended by asking whether to proceed — precisely the pattern
the pre-authorization fix above now closes at the source. But that fix landed *after* this mission
ran, and at the time, an exit with no crash and no filed escalation looked "clean" by the same
narrow definition the original false-COMPLETE bug used — a real, related, but genuinely distinct
failure mode from the earlier external-access case (a real, satisfied plan-mode rule instead of a
real permission denial). Root-caused, and the historical record corrected once understood, rather
than left standing as an inaccurate "COMPLETE."

**A transient server error mistaken for a hard failure.** A real mission's very first API call failed
with a transient, server-side "overloaded" error — a condition explicitly usually temporary — and,
because that failure class wasn't yet distinguished from a genuine hard failure, the mission gave up
immediately rather than retrying, discarding a mission that had not yet done any real work but very
likely would have succeeded moments later. Fixed with a dedicated detector for this specific failure
class, separate from the real-usage-limit path (which needs a much longer, clock-aware wait, not a
quick retry), and a real, tested guarantee that any work already durably checkpointed before a
transient failure is never discarded by the retry that follows it.

## Test architecture

No mocked business logic standing in for real behavior where it matters: permission-denial handling
is tested against real, previously-observed denial payloads, not synthetic guesses; the venv bug's
regression test reproduces the actual symlink structure, not an approximation; the false-COMPLETE
fix's regression tests are built directly from the real failed mission's own captured transcript.
Real counts, not invented for this document: the core decision-engine test suite and the tooling test
suite both run in the low thousands of tests combined, verified directly, not estimated.

## What this deliberately does not claim

No performance/scale metrics are cited here because none have been measured against real external
load — this is a single-operator system, not a claim of production scale. No implication that any of
this runs without human oversight where it matters: every consequential action still resolves through
the three-state model above, and a human still decides every escalation. This case study describes
the governance and reliability architecture; it does not describe, and never will describe here, the
private business logic (opportunity scoring, underwriting weights, CRM data) that architecture
happens to run on top of.
