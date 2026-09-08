# Plan: autonomous backlog grooming into the sprint queue (GitHub #22)

- Status: PROPOSED (awaiting Akhil approval)
- Proposed by: Siddharth Deshpande
- Date: 2026-09-08
- Issue: https://github.com/Apra-Labs/apra-fleet/issues/22
- Depends on: https://github.com/Apra-Labs/apra-fleet/issues/410 (especially
  PENDING create + GET queue + human reorder). Plan:
  `docs/proposals/issue-410-persistent-sprint-queue.md`
- Cover / context: `docs/proposals/README.md`

This is the approval document for the **decision-making** half. Do not start
this implementation until #410 Slice A is AGREED (and, practically, merged).
Engineering breakdown:
`docs/superpowers/plans/2026-09-08-autonomous-backlog-grooming.md`.

The issue itself left three product questions open. The decision table
answers them so this plan can be approved as a whole.

---

## Decision table (please mark)

These are the issue's open questions, plus two that fall out of them.

| # | Decision | Proposal | Why |
|---|---|---|---|
| G1 | How much autonomy -- queue directly, or a human-approved "proposed" state? | **Queue directly as PENDING.** No second draft state. | #410 already gives cancel, reorder, pin-next. PENDING has no members so it cannot run until someone (or later auto-assign) binds a machine. A "proposed" state recreates the human handoff this issue exists to remove. |
| G2 | Cadence and off-cadence triggers | Timer default **1 hour** (`FLEET_SE_GROOMER_INTERVAL_MS`, 0 = off). Also: sprint reached terminal; operator `POST /api/groom`. No separate P0 file-watcher in v1 -- the groomer already ranks by priority every pass. | Cheap, explicit, no extra beads-watch machinery. |
| G3 | Budget guardrails | Supervisor-enforced: `maxQueued` default **3** (PENDING+WAITING). Optional `maxDailyBudgetUsd` (off if unset). Quality bar is separate and still blocks enqueue. | The LLM must not be the only thing stopping spend. Caps belong in the API. |
| G4 | Who picks members? | Groomer **never** picks machines. It always POSTs `members: []` (PENDING). `queue.autoAssignMembers` default **false**. Operators assign (or pin) in the dashboard. | Matches "groomer knows what, not which laptop". Auto-assign can be a follow-up once the queue is trusted. |
| G5 | Idempotency | Key = sorted issue roots + branch. Re-POST returns the existing PENDING/WAITING/RUNNING/PAUSED sprint (HTTP 200), not a duplicate. Finished sprints do not block a later re-queue (relaunch gate still applies). | Issue: "re-running the groomer must not enqueue duplicates". |
| G6 | Quality bar | Never queue a set that the same pass put in needsGrooming, needsVerification, or an unresolved deadlock. Enforced in the runner, not only in the prompt. | Issue: "never queue work that failed the groomer's own quality bar". |

---

## 1. What is happening today

The backlog-groomer is a **read-and-mutate-beads** agent. It does not write
product code. For one operator identity it already:

- Ranks ready / urgent work (priority, unmet `blocks`, unclosed children)
- Groups cohesive sprint sets (`blocks` chains, shared parent, subsystem)
- Sweeps duplicates; evidence-gated merge/close/defer
- Flags high-priority items that are too thin (no repro, no AC)
- Treats stale P1 and stale P3 as different findings
- Checks landed-vs-closed (children closed is not proof the work is on
  this branch)

Canonical contracts:

- `packages/apra-fleet-se/apra-pm/agents/backlog-groomer.md`
- `packages/apra-fleet-se/apra-pm/agents/schemas/backlog-groomer-input.json`
- `packages/apra-fleet-se/apra-pm/agents/schemas/backlog-groomer-output.json`

Output is JSON **advice** (`sprintSets`, `needsGrooming`, `actions`, ...).
Nothing in that JSON is a supervisor sprint. The agent has no idea how many
members are free or what is already queued.

Akhil's comment on #22: this analysis half is considered addressed; the
remaining idea is to **use that agent from the fleet-supervisor**.

The missing half in the issue body:

- Sprint planning should see **real capacity** (members, running, queued)
- Autonomous **queueing** on a cadence into the supervisor
- **Delivery tracking** when a queued sprint completes (landed-vs-closed,
  velocity), so the next pass starts from reality

---

## 2. Target pipeline

```
supervisor timer or POST /api/groom or "a sprint just finished"
        |
        v
  load capacity: free/busy members, queued count, running count, caps
        |
        v
  run backlog-groomer (same agent, new input fields)
        |
        +-- mutate beads when dry-run is false (already allowed, evidence-gated)
        +-- emit queueRequests[] for sets that pass the quality bar
        |
        v
  supervisor POSTs each request as PENDING (members: [], queue: true)
        |
        v
  #410 queue: operator (or later auto-assign) binds members
        --> WAITING --> RUNNING when free
```

The groomer remains the intelligence. The supervisor remains the only
process allowed to create sprints. The agent should not curl the API
itself; the runner reads `queueRequests` and calls the same
`createSprintController.launch()` humans use. That keeps validation,
caps, and idempotency in one place.

---

## 3. Groomer contract changes (small)

**Input** (optional for a human "define my sprint" run; required for the
autonomous runner):

- `capacity`: `{ membersFree, membersBusy, queuedCount, runningCount, maxQueued }`

**Output** (new fields, old fields kept):

- `queueRequests[]`: `{ issue, branch, base, goal, budget, priority, idempotencyKey, beadIds, skippedReason? }`
- `delivery`: `{ verifiedClosed, stillOpen, atRisk, velocityNote }`

Branch names are concrete strings (`feat/short-topic`), never `$VAR` or
`~/` (member shells may be PowerShell).

Capacity use: do not emit more new queueRequests than
`maxQueued - queuedCount`. Prefer higher-priority sets. This is "the
capacity dimension" the issue asked for, without making the groomer a
scheduler.

---

## 4. Supervisor runner

New module `packages/apra-fleet-se/src/supervisor/groomer.mjs`:

- `POST /api/groom` -- one run if idle; 200 `{ status: "busy" }` if a run
  is already in flight
- Interval from env; identity from `FLEET_SE_GROOMER_IDENTITY` (required
  for autonomous runs; do not guess `git config` on the supervisor host)
- After output: filter by quality bar; POST PENDING with `origin: groomer`
  so caps apply; persist `groomer-last.json` for the dashboard
- On sprint terminal (watchdog release): request a groom run so
  landed-vs-closed happens against what just finished
- Velocity note: count finished vs crashed in last 7 days from sprint
  history if the agent did not fill it

**v1 spawn adapter:** injectable `runGroomer(input) -> outputJson`. Tests
fake it. Production wiring of the actual Claude/agent binary is a
follow-up if the first merge only proves the queueing path. The plan is
honest about that: the **contract and the POST path** are the #22
deliverable; a flaky LLM spawn should not block merging the pipeline.

---

## 5. What we will not do in v1

- Auto-assign members (G4 default false). Dashboard assign/pin from #410
  is the drain path.
- A "proposed sprints" approval inbox (G1).
- A new metrics product. Velocity is a one-line note on the last-groom
  card.
- Teaching the groomer to pick roleMaps or specific laptops.
- Rewriting grooming heuristics from scratch; extend the existing agent.

Still in scope as **extensions of the existing agent**, not new products:
stale x priority (already there), XL-leaf as decomposition signal
(already judged as S/M/L/XL), living backlog mutations (already
evidence-gated). The new work is capacity, queueing, cadence, delivery
loop.

---

## 6. Main code touchpoints

| Area | Path |
|---|---|
| Agent + schemas | `packages/apra-fleet-se/apra-pm/agents/backlog-groomer.md`, `agents/schemas/backlog-groomer-*.json` |
| Caps + idempotency | `packages/apra-fleet-se/src/supervisor/api.mjs`, `queue.mjs` |
| Runner | `packages/apra-fleet-se/src/supervisor/groomer.mjs` (create), `bin/serve.mjs` |
| Dashboard last-groom | `packages/apra-fleet-se/src/supervisor/dashboard.mjs` |
| Schema tests | `packages/apra-fleet-se/apra-pm/test/agent-schema-validation.test.mjs` |

---

## 7. Slice plan (after #410 Slice A)

1. Schema + agent markdown for `capacity` / `queueRequests` / `delivery`
2. Idempotency key + maxQueued / daily budget on `launch()`
3. Groomer runner + `POST /api/groom` + timer + terminal trigger
4. Dashboard last-groom card
5. Docs: SKILL.md, OpenAPI, supervisor-api.md
6. Follow-up (separate, called out, not silent): production agent spawn
   adapter; optional `autoAssignMembers`

---

## 8. Risks

- **Unconfigured identity:** timer must no-op, not groom the wrong
  person's beads.
- **Duplicate sprints:** wrong idempotency key (e.g. unstable branch
  names) would re-queue forever. Key includes branch; groomer must reuse
  a stable branch per set.
- **LLM ignores the quality bar:** runner filter is mandatory.
- **Spawn adapter:** if we cannot invoke the agent from `serve.mjs` in v1,
  `POST /api/groom` still needs a fakeable seam so the POST path is
  tested. Shipping a timer that always throws is worse than shipping the
  API + a documented "adapter not configured".

---

## 9. Approval

This plan is blocked on #410 D1-D5. You can still AGREE the G-table now
so #22 implementation can start the moment Slice A merges.

Reply AGREE / DISAGREE / NEEDS-DISCUSSION on G1-G6.
