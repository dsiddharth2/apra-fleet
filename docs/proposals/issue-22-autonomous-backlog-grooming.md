# Plan: autonomous backlog grooming into the sprint queue (GitHub #22)

- Status: REVISED (round 2, after Akhil review of 2026-09-17)
- Proposed by: Siddharth Deshpande
- Date: 2026-09-08, revised 2026-09-17
- Issue: https://github.com/Apra-Labs/apra-fleet/issues/22
- Depends on: https://github.com/Apra-Labs/apra-fleet/issues/410 Slice A
- Cover / context: `docs/proposals/README.md`
- Review thread: https://github.com/dsiddharth2/apra-fleet/pull/3

**Round 2 summary:** all three review points accepted. The framing now
separates "wire up the existing agent" from "change what the agent
produces" (section 1). The idempotency key drops `branch` and keys off
sorted bead ids (G5). A new decision row G7 covers absorbing newly-ready
beads into an existing WAITING sprint instead of always proposing a new
set. And G2 is replaced outright: the blind 1-hour full-agent timer is gone,
replaced by a tiered cadence whose common case costs one `bd` subprocess
call and zero tokens (section 4).

---

## Decision table

Rows marked **[r2]** changed in round 2. G7 is new.

| # | Decision | Proposal | Why |
|---|---|---|---|
| G1 | How much autonomy | **Queue directly as PENDING.** No second "proposed" approval state. | #410 gives cancel, reorder, pin. PENDING has no members so it cannot run until someone binds a machine. A proposed state recreates the handoff this issue removes. |
| G2 **[r2]** | Cadence | **Tiered, not a blind timer.** Tier 0: a cheap non-LLM backlog fingerprint check on each tick -- no change, no dispatch, zero tokens. Tier 1: delta groom over changed beads only, cheap model. Tier 2: full-backlog groom, rare (daily floor, or when the delta is large or Tier 1 flags ambiguity). Real events (sprint terminal, queue drained with a member free, `POST /api/groom`) are the primary triggers; the timer drops to a low-frequency safety net. | Review point 3 accepted. 24 full agent passes/day/operator is real money for mostly-empty passes. See section 4. |
| G3 | Budget guardrails | Supervisor-enforced: `maxQueued` default **3** (PENDING+WAITING). Optional `maxDailyBudgetUsd`. Plus (new in r2) a daily cap on Tier 2 passes. | The LLM must not be the only thing limiting spend. |
| G4 | Who picks members | Groomer **never** picks machines. Always POSTs `members: []`. `autoAssignMembers` default **false**. | Groomer knows what, not which laptop. |
| G5 **[r2]** | Idempotency key | Key = **sorted bead ids** of the sprint set. `branch` is dropped from the key. | Review point 1 accepted. The agent's output has no branch field today, so a branch-based key would depend on a string the LLM re-invents each pass -- unstable key, duplicate sprints. Bead ids are stable and already in `sprintSets[].beadIds`. |
| G6 | Quality bar | Never queue a set whose beads the same pass flagged in `needsGrooming` / `needsVerification` / an unresolved deadlock. Enforced in the runner, not just the prompt. | Issue: "never queue work that failed the groomer's own quality bar". |
| G7 **[r2] NEW** | Absorb into an existing WAITING sprint before proposing a new set | Each pass first reads the queue's **WAITING/PENDING** sprints and asks whether a new or newly-ready bead is cohesive with one of them (shared parent, `blocks`/related edge, same subsystem, continuation of the same work). If yes, amend that queued sprint's scope instead of creating an overlapping new set. **RUNNING is excluded** -- D7 in #410 forbids editing a live sprint. | Review point 2 accepted as its own row. See section 5. |

---

## 1. Framing correction: what exists vs what is new

The round-1 cover said "the groomer already analyzes beads", which
undersells it. Correcting, and splitting the scope three ways.

### Already real, shipped, not to be redesigned

`packages/apra-fleet-se/apra-pm/agents/backlog-groomer.md` is a working
agent with **full beads mutation authority**: it merges duplicates, closes
or defers stale items, reprioritizes, reparents and fixes bad edges -- every
mutation evidence-gated. It proposes cohesive sprint sets from
interdependencies, flags high-priority items whose content quality is too
low to act on, reports priority x age jointly, runs a structural-deadlock
scan, and does landed-vs-closed verification. Its real deliverable today is
**beads mutations**, plus a report.

Its output contract already exists too:
`agents/schemas/backlog-groomer-output.json` defines `sprintSets` as
`{ label, beadIds, reason }`, along with `needsGrooming`,
`needsVerification`, `duplicates`, `actions` and the rest. So the sprint-set
*shape* is not new scope -- one correction to the review on that narrow
point.

### What is genuinely missing (this issue)

Nothing in the repo invokes the agent programmatically. `install.mjs` copies
the file; the schema tests validate its contract. There is no dispatcher, no
cadence, and nothing that reads `sprintSets` and turns it into a launch. It
is dispatched ad hoc by a human today. That is the gap.

### Which G-rows are which

| Row | Kind |
|---|---|
| G1, G2, G3, G4 | **Wire the existing agent into a scheduler.** No change to what the agent decides. |
| G5 | **Runner-side.** Key computed from output the agent already emits (`beadIds`). |
| G6 | **Runner-side enforcement** of a bar the agent already applies. |
| G7 | **Changes what the agent produces.** It must read the queue and may output an amend-existing action, not only new sets. |

Additive agent-contract changes (new scope, called out rather than buried):

- Input gains `capacity` and `queuedSprints` (for G7).
- Output gains `queueRequests[]` and `absorb[]` (G7) and `delivery`.
- Output does **not** gain a `branch` field, because G5 no longer needs one.
  The runner derives the branch from the set's label/bead ids at POST time,
  so branch naming is deterministic supervisor-side rather than LLM-invented.

---

## 2. Target pipeline

```
trigger (sprint ended, queue drained, manual, or safety-net tick)
        |
        v
  Tier 0: cheap backlog fingerprint -- changed since last groom?    (no LLM)
        |  no -> stop here
        v
  load capacity + current WAITING/PENDING queue
        |
        v
  groom (Tier 1 delta, or Tier 2 full) -- the existing agent
        |
        +-- beads mutations (already its job)
        +-- absorb[]: add bead X to queued sprint S        (G7)
        +-- queueRequests[]: genuinely new sprint sets
        |
        v
  supervisor applies: PATCH queued sprint scope, or POST PENDING
        |
        v
  #410 queue -> operator binds members -> WAITING -> RUNNING
```

The agent never calls the API itself. The runner reads its JSON and calls
the same controller humans use, so validation, caps and idempotency live in
one place.

---

## 3. Groomer contract changes

**Input** (optional for a human "define my sprint" run; required for
autonomous runs):

- `capacity`: `{ membersFree, membersBusy, queuedCount, runningCount, maxQueued }`
- `queuedSprints`: `[{ sprintId, beadIds, label, state }]` -- WAITING/PENDING
  only, for G7
- `mode`: adds `delta` for Tier 1 (scope restricted to changed beads)

**Output** (new fields; existing ones unchanged):

- `queueRequests[]`: `{ setLabel, beadIds, goal, budget, priority, skippedReason? }`
- `absorb[]`: `{ sprintId, beadIds, reason }` (G7)
- `delivery`: `{ verifiedClosed, stillOpen, atRisk, velocityNote }`

No `branch` field, per G5.

---

## 4. G2 replaced: making the automatic cadence cheap

The review's objection: a full `backlog-groomer` dispatch is a real agent
turn over the whole assigned backlog. At 1 per hour that is ~24/day per
operator, most finding nothing, with no human check in the loop because G1
queues directly. Agreed. Replacing the timer with four mechanisms.

### Tier 0 -- non-LLM change detection (the common case)

The supervisor **already** shells `bd list --json --limit 0` once per
dashboard render (`backlog.mjs`'s `bdListAllBeadsRaw()`). One more call on a
tick is negligible, and needs no new dependency.

Each tick: fetch, filter to the operator's assignee, and compute a
fingerprint over the fields that decide whether grooming has anything to do:

- bead count
- per-bead `id`, `status`, `priority`
- newest `created_at` in the set

If the fingerprint equals the one stored from the last successful groom:
**stop, dispatch nothing, spend nothing.** Log a line so a quiet day is
visible rather than looking broken.

`id`/`status`/`priority`/`created_at` are all present on
`bd list --json` rows today (`normalizeBead()` reads exactly these, plus the
parent edge). If `updated_at` turns out to be available too, it can be folded
in to catch description edits; the fingerprint above already catches
everything that changes *actionability*, which is what triggers grooming.

Staleness is the one thing a fingerprint misses, because "bead got older"
changes no field. Handled by the Tier 2 daily floor below, not by re-running
the agent hourly.

### Tier 1 -- delta groom, cheap model

Fingerprint changed: hand the agent only the beads that are new or changed
since the last successful groom watermark, plus the current queue (G7), on a
**cheaper model tier**. Most real ticks are "two beads landed" and do not
need a whole-backlog re-read.

Escalates to Tier 2 when the agent reports ambiguity (a duplicate candidate
it cannot resolve from the delta alone, or a set whose cohesion depends on
beads outside the delta).

### Tier 2 -- full pass, rare and capped

Whole assigned backlog, stronger model. Triggered by: a daily floor (once per
day, so staleness and priority x age are still evaluated), a large delta,
Tier 1 escalation, or an explicit operator request. Capped per day via G3 so
escalation cannot loop.

### Trigger rebalance

Events become primary, the timer becomes a safety net:

| Trigger | Tier |
|---|---|
| Sprint reached terminal | Tier 0, then 1 (a finished sprint is a real change signal, and this is where landed-vs-closed matters) |
| Queue drained while a member is free | Tier 0, then 1 |
| `POST /api/groom` | Tier 2 (operator asked explicitly) |
| Safety-net timer, default **6h** | Tier 0 (usually stops there) |
| Daily floor | Tier 2 |

Default interval moves from 1h to 6h and, crucially, a tick's default
outcome is a fingerprint comparison rather than an agent dispatch. Expected
steady-state cost on a quiet day: a handful of `bd` calls and zero tokens.

---

## 5. G7: absorb before proposing

Today's groomer proposes sets from the backlog alone. Once a queue exists, a
bead filed after a sprint was queued but clearly belonging to it should join
that sprint, not spawn a near-duplicate one competing for the same files.

Rules:

1. Before proposing new sets, read `queuedSprints` (WAITING/PENDING only).
2. For each new or newly-ready bead, test cohesion against each queued
   sprint's `beadIds` using the signals the agent already uses for grouping:
   shared parent, an explicit `blocks`/related edge, same subsystem, or a
   natural continuation of the same work.
3. Cohesive: emit `absorb[]` instead of a new `queueRequests[]` entry.
4. The runner applies it by PATCHing that sprint's scope (the #410 Slice A
   PATCH already exists for non-running sprints).
5. **RUNNING is never a target.** #410's D7 forbids in-place edits of a live
   sprint, and the API enforces it. A bead cohesive only with a running
   sprint waits for the next queued set.
6. Absorption respects the same quality bar (G6). A thin bead is not smuggled
   into a good sprint.

This is also why G5's key matters: absorption needs a stable identity for a
queued set to check against. Sorted bead ids give that; an LLM-invented
branch string would not.

---

## 6. Out of scope for v1

- Auto-assign members (G4 default false)
- A "proposed sprints" approval inbox (G1)
- A metrics product; velocity stays a one-line note
- Rewriting grooming heuristics -- extend the existing agent
- Absorbing into RUNNING sprints (G7 rule 5)

---

## 7. Main code touchpoints

| Area | Path |
|---|---|
| Agent + schemas | `apra-pm/agents/backlog-groomer.md`, `agents/schemas/backlog-groomer-*.json` |
| Fingerprint + tiers | `src/supervisor/groomer.mjs` (create) |
| Cheap bead fetch | `src/supervisor/backlog.mjs` (`bdListAllBeadsRaw`, reuse) |
| Caps + idempotency | `src/supervisor/api.mjs`, `queue.mjs` |
| Wiring | `bin/serve.mjs` |
| Dashboard last-groom card | `src/supervisor/dashboard.mjs` |
| Schema tests | `apra-pm/test/agent-schema-validation.test.mjs` |

---

## 8. Slice plan (after #410 Slice A)

1. Schema + agent markdown: `capacity`, `queuedSprints`, `queueRequests`, `absorb`, `delivery`
2. Idempotency key (sorted bead ids) + caps on `launch()`
3. Tier 0 fingerprint + watermark store; tick that usually stops at Tier 0
4. Runner: Tier 1/2 dispatch, quality-bar filter, POST PENDING, apply `absorb[]`
5. Triggers: sprint-terminal, queue-drained, `POST /api/groom`, safety-net timer, daily floor
6. Dashboard last-groom card (timestamp, tier, queued ids, skipped reasons)
7. Docs: SKILL.md, OpenAPI, supervisor-api.md

Follow-ups, named not silent: production agent spawn adapter;
`autoAssignMembers`.

---

## 9. Risks

- **Fingerprint too coarse or too sensitive.** Too coarse misses work; too
  sensitive re-dispatches constantly. Mitigation: fingerprint only fields
  that change actionability, and log every decision so the ratio is
  measurable before trusting it.
- **Delta grooming hides context.** A duplicate is only visible against the
  whole backlog. Mitigation: Tier 1 escalates on ambiguity, and the Tier 2
  daily floor guarantees a full pass.
- **Unconfigured identity** must no-op, never groom the wrong backlog.
- **Absorption growing a sprint without bound** (G7). Mitigation: the same
  size judgment the agent already applies, plus the operator can always
  detach a bead from a queued sprint via PATCH.
- **Spawn adapter.** `POST /api/groom` needs a fakeable seam so the queueing
  path is testable without a live LLM.

---

## Akhil's Response -- 2026-09-17 (round 1)

| # | Item | Verdict | Resolution |
|---|---|---|---|
| 1 | Framing undersells the existing agent; be explicit about "wire it up" vs "change what it produces". Its real deliverable is beads mutations. | ACCEPTED | Section 1 splits exactly that, with a per-row table. One narrow correction: `sprintSets {label, beadIds, reason}` already exists in `backlog-groomer-output.json`, so that shape is not new scope -- but the rest holds, and nothing invokes the agent programmatically today. |
| 2 | G5's key depends on a branch the LLM invents fresh each pass; key off sorted bead ids instead. | ACCEPTED | G5 changed. `branch` is dropped from the key and from the agent's output entirely; the runner derives it deterministically at POST time. |
| 3 | Groomer should absorb newly-ready beads into existing WAITING sprints before proposing new sets. Own decision row. RUNNING excluded. | ACCEPTED | New G7 + section 5, including why this reinforces the G5 key change. |
| 4 | A 1-hour full-agent timer is expensive for mostly-empty passes, worse because G1 has no approval gate. Spec cheap techniques. | ACCEPTED | G2 replaced. Section 4: non-LLM fingerprint pre-check (reusing the `bd list --json` call the supervisor already makes), delta grooming, cheaper model tier, event-primary triggers, timer to 6h, Tier 2 capped with a daily floor. |

No DISAGREE rows. Status stays REVISED pending Akhil's round-2 confirmation.
