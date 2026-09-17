# Plan: persistent sprint queue (GitHub #410)

- Status: REVISED (round 2, after Akhil review of 2026-09-17)
- Proposed by: Siddharth Deshpande
- Date: 2026-09-08, revised 2026-09-17
- Issue: https://github.com/Apra-Labs/apra-fleet/issues/410
- Unblocks: https://github.com/Apra-Labs/apra-fleet/issues/22
- Cover / context: `docs/proposals/README.md`
- Review thread: https://github.com/dsiddharth2/apra-fleet/pull/3

This is the approval document for the **mechanism**. It is not an
implementation checklist. Implementation detail lives in
`docs/superpowers/plans/2026-09-08-supervisor-persistent-sprint-queue.md`
(Slice A) and
`docs/superpowers/plans/2026-09-08-supervisor-queue-pause-history.md`
(Slice B).

**Round 2 summary:** all five review points accepted. Every technical claim
in the review was re-checked against the code and holds. The substantive
changes are: D6 is reframed as an invariant change rather than a wrapper and
now carries a concrete restart-recovery design (D6a) and an explicit
fleet-mirror decision (D9); D7 is rescoped to fix the existing Restart path
instead of building a parallel one; D4 gains a dual-source poll so an
externally-freed member cannot strand a WAITING sprint. Slice A (D1-D5) is
unchanged apart from D4's trigger.

---

## Decision table

Rows marked **[r2]** changed in round 2.

| # | Decision | Proposal | Why |
|---|---|---|---|
| D1 | Where queued sprints live | New on-disk store `sprint-queue.json`, sibling of the reservation ledger. Ledger stays "reserved right now". History stays terminal events. | Confirmed in review: the ledger feeds `scope-overlap.mjs`, which live-re-expands every entry's bead subtree per launch attempt. PENDING rows there would add real cost, not just muddy semantics. |
| D2 | Empty `members` | Valid. Creates PENDING. No spawn, no reservation. Issue/branch/base still validated at create time. | Groomer (#22) knows *what* to sprint, not *which machine*. |
| D3 | Busy named member | Default stays today's **409**. `queue: true` stores WAITING and returns 201. | Issue wants WAITING-by-default *and* "do not silently change launch-now callers". Launch form + Restart are launch-now callers. Opt-in keeps them. |
| D4 **[r2]** | When a WAITING sprint starts | Event **plus poll**. Events: watchdog auto-release, stop, force-release, pause, member assignment. Poll: re-evaluate the WAITING set on the existing watchdog tick, using the **dual-source** free check (local ledger + the fleet server's `reservedBy`), not a ledger-only list. | Review point 5 accepted. `defaultMemberOverlapGuard` is deliberately dual-source; the only trigger I had proposed (watchdog auto-release) is purely local, so a member freed by an external reservation source would strand a WAITING sprint forever. |
| D5 | Ordering | `priority` desc, then older first, plus a single "pin this next" that always wins. PATCH + reorder API. Audit who/why. Human override survives restart. | Issue point 3. |
| D6 **[r2]** | Pause (point 4) | **Not a wrapper -- an invariant change, stated as such.** Pause releases the supervisor ledger so capacity actually moves to the queue. Resume re-enters WAITING. Different member: respawn from stored git HEAD. Diverged HEAD fails loudly. Budget remaining = original cap minus spent. | Review point 1 accepted. See section 3. |
| D6a **[r2]** | Restart recovery for paused sprints | Reconcile/readopt gains a **second source**: the queue's PAUSED records, not just ledger rows. Parked-PID lifecycle is specified explicitly. A spike first checks whether pause can persist-and-exit resumably, which would remove the parked PID (and the blind spot) altogether. | Review point 2 accepted -- this was named as a risk with no design. See section 4. |
| D7 **[r2]** | Relaunch / Restart | **Fix the existing Restart path; do not build a parallel one.** Today's Restart force-releases first, then prompts for unrecoverable branch/base/goal. Re-point it at the stored queue/history request so it prefills instead of prompting, and stops being destructive-before-validating. | Review point 4 accepted. See section 6. |
| D8 | Ship shape | Two PRs: **A** = D1-D5 (queue exists, dashboard shows it). **B** = D6, D6a, D7, D9. | #22 only needs A. |
| D9 **[r2]** | Fleet-global reservation mirror on pause | Explicit: pause must leave **no** reservation for that member in **either** source. Today the runner already releases the fleet-side reservation on `'paused'`; the supervisor ledger release is the new half. `ledger.mjs`'s `reservationClient` seam stays unwired in this work, and if it is ever wired, `ledger.release()` drives the mirror release automatically -- consistent, no second code path. | Review point 3 accepted -- this was implicit and should not have been. See section 5. |

Open for Akhil: D6a's primary-vs-spike choice (section 4) is the one place
I am proposing a decision that depends on an unknown. If you would rather
commit to the parked-PID design without the spike, say so.

---

## 1. What is happening today (re-verified for round 2)

`POST /api/sprints` (`packages/apra-fleet-se/src/supervisor/api.mjs`):

1. Validate issue / branch / base / **non-empty members** (`members.length === 0` -> 400).
2. Relaunch gate.
3. Member overlap guard -> **409**, request discarded. Dual-source: local ledger and the fleet server's own `reservedBy`.
4. Issue-scope overlap guard.
5. Mint sprint id, **spawn child**, **claim ledger**.

There is no persisted object for "this sprint should happen later", and no
queue, priority, or ordering concept anywhere in `src/supervisor` today.
D1-D5 are greenfield.

---

## 2. Target behavior (points 1-3)

Unchanged from round 1 except D4's trigger.

### Point 1 -- PENDING (zero members)

201 `{ sprintId, state: "pending" }`; survives restart; listed distinctly;
`GET /api/sprints/:id` returns the full stored request; **no** reservation;
issue/branch/base validated now, not at start; cancellable before running.

### Point 2 -- WAITING, then auto-start

With `queue: true`, a busy member yields 201 `state: "waiting"` instead of
409. Nothing is reserved while waiting.

Eligibility is evaluated against members free **at that instant**, via the
dual-source check (D4). Start only when the **entire** member union is free;
no partial starts. An unsatisfiable head-of-line item is skipped, not waited
on; the operator can pin to override. A named member that is later
unregistered moves the item to BLOCKED with a visible reason. The relaunch
gate and topology checks run **again at execution**, not only at enqueue.

### Point 3 -- Ordering

`priority` desc, `pinNext` (at most one) wins, tie-break oldest first then
`sprintId`. `PATCH /api/sprints/:id`, `POST /api/sprints/:id/reorder`,
`GET /api/sprints` in consume order, history event per reorder, and the
dashboard Queue section reads the same comparator.

---

## 3. D6 restated honestly: this changes a deliberate invariant

Round 1 called pause "a thin wrapper around the existing primitive". That
was wrong, and the review is right to push on it.

`watchdog.mjs` documents the current rule in its own comment: a live,
HTTP-reachable child whose engine reports `state.pause.status === 'paused'`
is classified `PAUSED`, falls through the same `pidAlive` branch as
running-healthy, and is therefore **never** routed through
`releaseTerminalReservation()` -- that only runs in the PID-gone branch.
The module header states the same thing as an invariant: a paused sprint's
"reservation is never auto-released".

That invariant exists for a good reason. It protects a paused run from
having its members taken while it is parked and expects to continue.

D6 deliberately inverts it, because the invariant is exactly what makes
pause useless for a queue: if a paused sprint keeps its member, pausing
frees nothing and an urgent queued sprint still cannot run. The whole point
of #410 point 4 is that pause "must genuinely free the member/resources, not
merely mark the sprint idle".

So the honest framing, which will be written into the code comments too:

- The watchdog keeps classifying `PAUSED` (no change to classification).
- What changes is **who releases the reservation**. Not the watchdog -- the
  new explicit `POST /api/sprints/:id/pause` path releases it, after the
  child confirms it is actually paused.
- The watchdog's "never auto-release a paused sprint" rule stays true as
  written: it still never auto-releases one. The release becomes an operator
  (or scheduler) action with a durable queue record behind it.

That distinction matters: an unrequested pause (engine-initiated, e.g.
sprint-doctor's `pause_for_human`) must **not** silently lose its members.
Only an explicit pause-into-queue request releases. This is a behavior split
the round-1 doc did not make, and it is now part of the design.

---

## 4. D6a: the restart blind spot, with a design

**The problem, as the review states it:** restart recovery
(`reconcile.mjs` -> `readopt.mjs`) is PID-probe-driven **off ledger entries**.
`reconcile()` iterates `ledger.list()`; `readopt()` then recovers each
retained entry's `--viewer-port` from the live process's command line. Once
pause releases the ledger row, a paused sprint with a live parked PID has no
ledger row, so neither pass can see it. The process would survive the
restart as an orphan: alive, holding a port, invisible to the supervisor.

Round 1 listed this as a risk and stopped there. Two ways to close it.

### Option A (primary): give restart recovery a second source

Reconcile and readopt stop being ledger-only:

1. `reconcile()` keeps its ledger pass unchanged.
2. A new pass walks `queue.list({ state: 'paused' })`. For each record with
   a non-null `childPid`:
   - PID alive and cmdline still carries the sprint's `--viewer-port`
     marker: re-adopt exactly as a running child is re-adopted (register the
     port with the spawner) so the dashboard, watchdog and proxy can reach
     it. The record stays PAUSED. It holds no reservation, which is correct.
   - PID gone: clear `childPid` on the record and leave it PAUSED. Resume
     will respawn from the snapshot. Record a history event so the
     transition is auditable rather than silent.
3. The watchdog likewise probes paused queue records, so a parked child that
   dies mid-pause is noticed rather than assumed alive.

This is bounded and certain to work, at the cost of teaching two recovery
modules a second input.

### Option B (spike first): no parked process at all

If pause can persist enough state and let the child **exit** -- with the run
resumable from disk -- then there is no parked PID, no orphan, and no blind
spot. Resume always respawns, which is the path we must build anyway because
#410 requires resume onto a **different** member.

This is attractive because it collapses two resume paths (proxy `/resume` to
a parked child vs respawn elsewhere) into one. The engine design notes
discuss `'paused'` as a resumable status, but I have not verified that a
paused run can be persisted, exited, and resumed without treating the run as
terminal. I am not going to claim it works on the strength of a doc comment.

**Proposal:** run a short spike at the start of Slice B. If persist-and-exit
resume works, take Option B and drop the parked-PID handling entirely. If
not, take Option A. Either way the blind spot is closed **before** D6 ships,
which is the review's actual requirement.

---

## 5. D9: which reservation sources pause must clear

The review is right that round 1 left this implicit. Making it explicit.

There are two places a member can be marked as taken:

1. **The supervisor's own ledger** (`reservations.json`).
2. **The fleet server's per-member `reservedBy`**, which
   `defaultMemberOverlapGuard` also consults, precisely so a reservation
   made by some other route still blocks a launch.

If pause clears only the first, the next sprint is rejected by the second,
and pause frees nothing in practice. So: **pause must leave no reservation
for that member in either source.**

Current reality, verified:

- The fleet-side release **already happens**: on `'paused'`, the runner
  releases each member's `member_reservation`, and re-reserves (owner-checked)
  on resume. That is shipped behavior, not new work.
- The supervisor-ledger release is the new half, and is what D6 adds.
- `ledger.mjs` accepts an optional `reservationClient` that would mirror
  claim/release onto the fleet server, but `bin/serve.mjs` calls
  `createLedger()` with no deps, so that bridge is inert today; only the
  read side is live.

**Decision:** this work does **not** wire `reservationClient`. Doing so is a
separate change with its own blast radius, and the runner already covers the
fleet side on pause. But the design is deliberately compatible: because the
release goes through `ledger.release()`, wiring the client later makes the
mirror release happen automatically, with no second pause-specific code
path. If we wired the client *without* this compatibility in mind, pause
would need its own mirror logic -- that is the trap this row avoids.

Slice B will assert the end state in a test: after pause, the member appears
free in `GET /api/members`, which resolves both sources.

---

## 6. D7 rescoped: fix Restart, do not clone it

The review is correct that a "relaunch from history" feature would sit
beside an existing Restart button and the two would drift.

What exists today: a Restart button on running Sprint Stack rows whose
client script force-releases the reservation **first**, then reads
`branch`/`base`/`goal`/`members`/`issueRoots` back out of the force-release
audit payload, and `window.prompt()`s the operator for anything that came
back null -- after the reservation is already gone. Cancelling a prompt
leaves the operator with a released sprint and a manual relaunch. The ledger
persists `branch`/`base`/`goal` explicitly for this control.

So the fix is the same feature, done in the right order:

1. **Build the request first, act second.** Read the stored request (queue
   record, or history/ledger metadata for a terminal sprint), prefill it, and
   only then release/stop and submit. Nothing destructive happens until
   there is a valid request to submit.
2. **Prompts become a fallback, not the normal path.** With the queue record
   storing the full original request (D2 stores every field, not just three),
   a restart of a queued/paused sprint needs no prompting at all. Only a
   legacy ledger entry predating those fields can still prompt.
3. **Same code path for "relaunch a past sprint".** A terminal sprint's
   stored request goes through the identical prefill-then-submit flow, which
   is what the issue's point 5 asks for. One implementation, two entry
   points, rather than an old destructive path and a new safe one.
4. **Clone-and-amend** (PENDING/WAITING/PAUSED) is the same prefill flow with
   an editable overlay and a new sprint id; the original is untouched.
5. **RUNNING stays non-editable**, enforced in the API (409), not just hidden
   in the UI.

Net effect on scope: D7 is no longer "add a history browser with a relaunch
button". It is "make the stored request the single source for restart,
relaunch and clone, and fix the existing button to use it", plus the
recent-5 history list the issue asks for.

---

## 7. State machine

```
PENDING   created, no members, nothing reserved
   |  PATCH members
   v
WAITING   members named, not reserved, waiting for them to be free
   |  entire union free (dual-source check) + wins ordering
   v
RUNNING   ledger claim held, child spawned
   |
   +--> COMPLETED / FAILED / STOPPED   (history)
   |
   `--> PAUSED  ledger released, snapshot kept  -->  WAITING on resume
```

WAITING + deleted member -> BLOCKED. DELETE allowed on PENDING / WAITING /
PAUSED / BLOCKED; RUNNING uses stop / force-release.

Engine-initiated pause (not requested through the queue API) stays on the
ledger and is classified PAUSED exactly as today -- see section 3.

---

## 8. Out of scope

- Grooming intelligence (that is #22)
- Cross-machine scheduling beyond "entire requested union is free"
- Pre-reserving a member for a WAITING sprint
- Rebuilding the engine pause/resume primitive
- In-place mutation of a live sprint's parameters
- Wiring `ledger.mjs`'s `reservationClient` (D9)

---

## 9. Main code touchpoints

| Area | Path |
|---|---|
| Launch API | `packages/apra-fleet-se/src/supervisor/api.mjs` |
| New queue store | `src/supervisor/queue.mjs` (create) |
| Pick-next + poll | `src/supervisor/scheduler.mjs` (create) |
| Capacity events | `watchdog.mjs` (auto-release hook, paused-record probe), `serve.mjs` |
| Restart recovery (D6a) | `reconcile.mjs`, `readopt.mjs` |
| UI | `dashboard.mjs` (Restart script rework, Queue section), `launch-form.mjs` |
| Contract | `docs/supervisor-api.md`, `docs/supervisor-openapi.yaml`, `fleet-sprint/skills/fleet-supervisor/SKILL.md` |
| Tests | `test/supervisor-*.test.mjs` (existing 409 tests stay valid under D3) |

---

## 10. Slice plan

**Slice A (D1-D5; unblocks #22)**

1. Queue store, restart-safe
2. PENDING create + GET
3. WAITING via `queue: true`; default 409 unchanged
4. Scheduler: event kick **plus** dual-source poll on the watchdog tick
5. PATCH / reorder / DELETE + audit
6. Dashboard Queue section; launch form allows empty members
7. OpenAPI + SKILL.md

**Slice B (D6, D6a, D7, D9)**

0. **Spike:** can a paused run persist-and-exit resumably? Picks Option A or
   B in section 4.
1. Pause API: release ledger after the child confirms paused; snapshot;
   explicit split from engine-initiated pause
2. Restart recovery covers paused records (or spike removed the need)
3. Resume -> WAITING; different-member respawn; SHA and budget rules
4. Rework the existing Restart script to prefill-then-act; reuse it for
   relaunch-from-history and clone-and-amend
5. History list endpoint + dashboard recent-5
6. Assert member is free in `GET /api/members` after pause (D9 end state)

---

## 11. Risks

- **D6 inverts a documented invariant** (section 3). Mitigated by splitting
  requested-pause from engine-initiated pause, and by saying so in the code.
- **Restart recovery blind spot** (section 4). Must be closed before D6
  ships; spike decides how.
- **Dual-source poll cost.** The poll reuses the watchdog's existing tick and
  the existing `listMembers` read rather than adding a new loop.
- **D3 default.** If product wants WAITING-by-default, say so before Slice A
  tests are written the other way.

---

## Akhil's Response -- 2026-09-17 (round 1)

| # | Item | Verdict | Resolution |
|---|---|---|---|
| 1 | D6 is not a wrapper; it inverts watchdog's never-release-a-paused-sprint rule. Name it. | ACCEPTED | Section 3 rewritten. Also splits requested-pause from engine-initiated pause so the invariant still holds for the latter. |
| 2 | Restart recovery is ledger-driven; a released paused sprint becomes invisible to reconcile/readopt. Needs a design, not a risk bullet. | ACCEPTED | New D6a + section 4: second recovery source, or a spike that removes the parked PID entirely. Closed before D6 ships. |
| 3 | `ledger.mjs`'s `reservationClient` bridge is unwired; pause's effect on the fleet-global mirror is implicit. | ACCEPTED | New D9 + section 5. Not wiring it; documenting that the runner already releases fleet-side, and that `ledger.release()` keeps the design mirror-compatible. |
| 4 | D7 overlaps the existing destructive Restart button; scope it as a fix, not a parallel path. | ACCEPTED | Section 6: one prefill-then-act flow serving restart, relaunch and clone. |
| 5 | D4's only trigger is local watchdog auto-release; an externally-freed member would strand a WAITING sprint. | ACCEPTED | D4 now event **plus** dual-source poll on the watchdog tick, reusing the existing guard's two sources. |

No DISAGREE rows. Status stays REVISED until Akhil confirms round 2,
specifically D6a's spike-first proposal.
