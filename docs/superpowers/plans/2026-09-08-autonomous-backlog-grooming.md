# Autonomous Backlog Grooming Pipeline (#22) Implementation Plan

> **Approval doc (send this to Akhil, not this file):** `docs/proposals/issue-22-autonomous-backlog-grooming.md`

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the existing backlog-groomer agent a programmatic caller. The agent already grooms (with full beads mutation authority) and already proposes cohesive `sprintSets`; nothing in the repo invokes it on a cadence or turns its output into a sprint. This plan is mostly **wiring**, with exactly one change to what the agent itself decides: G7 absorption.

**Architecture:** Extend groomer input/output schemas and the agent markdown. Supervisor `groomer.mjs` runs a cheap non-LLM backlog-change check first, dispatches the agent only when something actually changed, supplies capacity + the current queue, then applies `absorb[]` to existing queued sprints and POSTs the rest as PENDING (`members: []`, `queue: true`) through the same controller humans use. Caps (`maxQueued`, optional daily budget, full-pass-per-day) are enforced in the controller. Events (sprint terminal, queue drained) are the primary triggers; the timer is a low-frequency safety net.

**Tech Stack:** Existing backlog-groomer agent + JSON schemas; supervisor HTTP; `node:test`. Depends on PR A (PENDING + idempotencyKey). Pause/history (PR B) not required for the first groomer increment.

**Spec:** `docs/superpowers/specs/2026-09-08-supervisor-sprint-queue-and-autonomous-grooming-design.md` sections 4 and 10. Product defaults: PENDING, opt-in auto-assign, `maxQueued=3`, tiered cadence with a 6h safety-net timer (round 2 replaced the original 1h full-pass timer).

## Global Constraints

- ASCII only. Never cite bead ids in LLM-facing prompt/skill/schema/runtime strings the runner adds.
- Groomer still never edits source code or runs git writes.
- Groomer never selects machines. It queues PENDING only.
- `queue.autoAssignMembers` default false (D2). Do not implement auto-assign in the first increment unless the dashboard assign-members path from PR A is already enough for operators.
- Idempotent: same `idempotencyKey` must not create a second PENDING/WAITING/RUNNING/PAUSED sprint.
- Quality bar: never enqueue items in needsGrooming / needsVerification / unresolved deadlock.
- Caps enforced in the supervisor even if the agent ignores them.
- **A cadence tick must not cost an agent turn by default.** The common case is a fingerprint comparison that dispatches nothing. A full-backlog pass is reserved for the daily floor, a large delta, an escalation, or an explicit operator request -- and is capped per day.
- **The agent emits no `branch` and no idempotency key.** The runner derives both. An LLM-invented branch string would make the key unstable across passes.
- **Never amend a RUNNING sprint.** Absorption targets WAITING/PENDING only; PATCH on RUNNING is 409 by design.
- `packages/apra-fleet-client` unchanged unless a fleet MCP tool is added (do not add MCP in this plan).
- PowerShell-safe: any command string the runner builds for a member must not use `$VAR`/`~/`/backticks. Runner-derived branch names are concrete strings.

---

### Task 1: Schema + agent contract for queueRequests and capacity

**Files:**
- Modify: `packages/apra-fleet-se/apra-pm/agents/schemas/backlog-groomer-input.json`
- Modify: `packages/apra-fleet-se/apra-pm/agents/schemas/backlog-groomer-output.json`
- Modify: `packages/apra-fleet-se/apra-pm/agents/backlog-groomer.md`
- Modify: `packages/apra-fleet-se/apra-pm/test/agent-schema-validation.test.mjs` (fixtures if required)

**Interfaces:**
- Input adds:

```json
"capacity": {
  "type": "object",
  "properties": {
    "membersFree": { "type": "array", "items": { "type": "string" } },
    "membersBusy": { "type": "array", "items": { "type": "string" } },
    "queuedCount": { "type": "integer" },
    "runningCount": { "type": "integer" },
    "maxQueued": { "type": "integer" }
  }
}
```

- Input also adds `queuedSprints` (round 2, G7): `[{ sprintId, label, beadIds, state }]`, WAITING/PENDING only. RUNNING sprints are deliberately excluded -- they cannot be amended (PATCH RUNNING is 409), so showing them to the agent would only invite a proposal the API rejects.
- Input also adds `mode: "full" | "delta"` and, for `delta`, `changedBeadIds` (round 2, cheap cadence).
- Output adds `queueRequests`, `absorb` and `delivery` as in the spec.

**Framing (round 2): most of this task is wiring, not redesign.** The agent already emits `sprintSets` as `{ label, beadIds, reason }` and already applies the quality bar, dedupe, and cohesion grouping. What is genuinely new to the agent's own behavior is exactly one thing: G7 absorption (reading `queuedSprints` and preferring to amend an existing queued set over proposing an overlapping new one). Everything else here is a caller-side contract addition. Do not rewrite the agent's grooming heuristics.

Agent markdown: new last step "Queue, do not advise-only" when `dry-run` is false AND capacity is present:

1. **Absorb first (G7).** For each new or newly-ready bead, test cohesion against each entry in `queuedSprints` using the grouping signals the agent already uses -- shared parent, an explicit `blocks`/related edge, same subsystem, or a natural continuation of the same work. On a match emit an `absorb` entry `{ sprintId, beadIds, reason }` instead of a new sprint set. Absorption obeys the same quality bar: a bead that failed it is not smuggled into a good sprint.
2. **Then propose what is left** as `queueRequests`, sized so `queuedCount + new <= maxQueued`. Each request: `issue` (comma-joined bead ids of the set's roots -- the beads' actual ids from `bd`, never invented), `goal`, `priority` (numeric from beads priority), `beadIds` (sorted). `skippedReason` when a set is withheld.

**No `branch` field (round 2, G5).** The agent does not emit a branch and the idempotency key does not use one. A branch string invented fresh by the LLM on each pass would make the key unstable and produce duplicate sprints for the same work -- and absorption (G7) needs a stable identity to check against. The key is the **sorted bead ids** of the set; the runner derives the branch name deterministically at POST time from the set label and bead ids.

Do not tell the agent to call curl itself if `dry-run` is true. The supervisor runner performs the POST (Task 3) so the agent stays beads+git-read and schema-out.

- [ ] **Step 1: Write/adjust schema fixtures so validation tests fail on missing new optional properties (optional fields should still pass). Add one valid fixture that includes `queueRequests`.**

- [ ] **Step 2: Run** `node --test packages/apra-fleet-se/apra-pm/test/agent-schema-validation.test.mjs` -- expect FAIL until schema+fixture match.

- [ ] **Step 3: Edit schemas and backlog-groomer.md Step 9 / responsibilities.** Keep default dry-run true. Document capacity as optional for human "define my sprint", required for autonomous runner.

- [ ] **Step 4: Re-run schema tests. Expected PASS.**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(groomer): add capacity input and queueRequests output contract"
```

---

### Task 2: Controller idempotencyKey + maxQueued + daily budget caps

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs`
- Modify: `packages/apra-fleet-se/src/supervisor/queue.mjs` (`findByIdempotencyKey`)
- Modify: `packages/apra-fleet-se/bin/serve.mjs` (read env `FLEET_SE_GROOMER_MAX_QUEUED`, `FLEET_SE_GROOMER_MAX_DAILY_BUDGET_USD`)
- Modify: `packages/apra-fleet-se/test/supervisor-api-queue.test.mjs`

**Interfaces:**
- **Key derivation (round 2, G5):** `idempotencyKey = sha256(beadIds.slice().sort().join(','))`, computed **by the runner** from `queueRequests[].beadIds`. The agent never supplies the key and never supplies a branch. Rationale: the agent's output has no stable branch field, so a branch-derived key would change every pass and duplicate sprints for identical work. Sorting makes the key order-independent so a re-proposed set in a different order still collides.
- `queue.findByIdempotencyKey(key)` scans PENDING/WAITING/PAUSED plus ledger-running metadata (store `idempotencyKey` on claim extras or keep a side index on the queue document `consumedKeys` for running ids). Simplest: persist `idempotencyKey` on the queue record until start, then write it onto the ledger reservation (add optional field, normalize null) so a running sprint still collides.

```js
// launch():
if (key) {
  const existing = queue.findByIdempotencyKey(key) || ledger.findByIdempotencyKey(key);
  if (existing) return { ...existingView, state: existing.state, reused: true }; // HTTP 200 not 201
}
if (origin === 'groomer' || body.enforceQueueCaps === true) {
  const queued = queue.list({ state: ['pending', 'waiting'] }).length;
  if (queued >= maxQueued) throw new ApiError(429, `groomer.maxQueued (${maxQueued}) reached`, 'queue');
}
```

Register route: if `result.reused` then `sendJson(res, 200, result)` else 201.

Daily budget: if `maxDailyBudgetUsd` is a number, sum `request.budget` (or reservation.budget) of items whose `createdAt` is today UTC plus running. If adding `body.budget` exceeds, 429.

- [ ] **Step 1: Write failing tests** for reuse 200, second POST no extra queue row, 429 at maxQueued=1, 429 when daily budget exceeded, caps not applied to a normal human launch without origin groomer / enforceQueueCaps.

- [ ] **Step 2: Run -- expect FAIL**

- [ ] **Step 3: Implement findByIdempotencyKey, optional ledger field, 200-vs-201 in registerSprintRoutes, cap checks.**

- [ ] **Step 4: Run api-queue + ledger tests. Expected PASS.**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): idempotent queue keys and groomer enqueue caps"
```

---

### Task 3: `groomer.mjs` runner + POST /api/groom + timer

**Files:**
- Create: `packages/apra-fleet-se/src/supervisor/groomer.mjs`
- Create: `packages/apra-fleet-se/test/supervisor-groomer.test.mjs`
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs` or a small `registerGroomerRoutes`
- Modify: `packages/apra-fleet-se/bin/serve.mjs`
- Modify: `packages/apra-fleet-se/src/supervisor/dashboard.mjs` (last-groom card)
- Modify: `packages/apra-fleet-se/src/supervisor/watchdog.mjs` or scheduler: on successful `releaseTerminalReservation`, call `groomer.requestRun('sprint-terminal')`

**Interfaces:**

```js
export function createGroomerController(deps) {
  // deps: runGroomer, launch, listMembers, queue, ledger, history, now, identity, caps, dataDir
  return {
    run: async (reason) => ({ status, queued, skipped, notes, reused }),
    isRunning: () => boolean,
  };
}
```

`runGroomer(input)` in production is an injected seam (`deps.runGroomer`). The default adapter in this increment throws a clear "groomer spawn adapter not configured" error so the HTTP/timer path is testable without a live LLM. A follow-up wires the installed `backlog-groomer` agent file. Tests inject:

```js
async function runGroomer(input) {
  assert.ok(input.capacity);
  return {
    status: 'OK',
    identity: input.identity,
    notes: 'test',
    sprintSets: [{ label: 'set-a', beadIds: ['iss-1'], reason: 'ready' }],
    needsGrooming: [],
    needsVerification: [],
    absorb: [],
    queueRequests: [{
      issue: 'iss-1', beadIds: ['iss-1'], goal: 'P1', priority: 2, budget: 5,
    }],
    delivery: { verifiedClosed: [], stillOpen: [], atRisk: [], velocityNote: '0 finished in last 7d' },
  };
}
```

Note the fixture carries **no `branch` and no `idempotencyKey`** -- per G5 the runner derives both (branch deterministically from label+bead ids, key from sorted bead ids).

**Tier 0: cheap non-LLM change detection (round 2, G2).** The default outcome of a cadence tick must be "spend nothing", not "dispatch an agent". The supervisor already shells `bd list --json --limit 0` once per dashboard render via `bdListAllBeadsRaw()` in `backlog.mjs`; reuse that exact call -- no new dependency, no new bd invocation shape.

```js
export function backlogFingerprint(rows, identity) {
  const mine = rows.filter((r) => r.assignee === identity);
  const parts = mine
    .map((r) => `${r.id}:${r.status}:${r.priority ?? ''}`)
    .sort();
  const newest = mine.reduce((max, r) => (r.created_at > max ? r.created_at : max), '');
  return sha256(`${mine.length}|${newest}|${parts.join(',')}`);
}
```

`id`, `status`, `priority` and `created_at` are all present on `bd list --json` rows today (`normalizeBead()` reads exactly these plus the parent edge). If `updated_at` proves available, fold it in to also catch description edits; the fields above already catch everything that changes a bead's *actionability*, which is what grooming acts on. Fingerprint equal to the stored watermark -> **return `{ status: 'unchanged' }`, dispatch nothing.** Log one line so a quiet day looks deliberate rather than broken.

Staleness is the one signal a fingerprint cannot see (age changes no field), which is why the Tier 2 daily floor below exists instead of an hourly full pass.

Runner logic (pure enough to unit test):

1. If `isRunning()`, return `{ status: 'busy' }` without starting a second agent.
2. **Tier 0.** Compute `backlogFingerprint`. If unchanged and the trigger is not an explicit operator request and the Tier 2 daily floor is not due, return `{ status: 'unchanged' }`. No tokens spent.
3. Decide tier: `delta` (changed beads only, cheap model) by default; `full` (whole backlog, stronger model) when the daily floor is due, the delta exceeds a threshold, an operator asked explicitly, or a prior delta pass escalated. Enforce the per-day Tier 2 cap so escalation cannot loop.
4. Build capacity from `listMembers` + ledger + queue, and `queuedSprints` from WAITING/PENDING records (G7; RUNNING excluded).
5. `output = await runGroomer({ identity, dry-run: false, mode, changedBeadIds, capacity, queuedSprints })`.
6. **Apply `absorb[]` first (G7):** for each entry, PATCH that queued sprint's scope. Reject (and log) any entry targeting a RUNNING sprint rather than passing it to an API that will 409.
7. Filter `queueRequests`: drop if `skippedReason`; drop if a bead is in `needsGrooming` / `needsVerification`; drop if a `structuralIssues` deadlock mentions it.
8. For each remaining, derive branch + `idempotencyKey`, then `launch({ ..., members: [], queue: true, origin: 'groomer', enforceQueueCaps: true, idempotencyKey })`. Collect 429 as skipped.
9. Write `<dataDir>/groomer-last.json` including the new fingerprint watermark, the tier used, and the trigger reason.
10. Velocity: count history events finished vs crashed in last 7d; stamp `delivery.velocityNote` if the agent left it empty.

`POST /api/groom` -> 202 `{ status: 'started' }` or 200 `{ status: 'busy' }`. An explicit operator request skips Tier 0 and runs Tier 2 -- a human asking is itself the signal.

**Triggers (round 2): events primary, timer as safety net.**

| Trigger | Tier entered |
|---|---|
| Sprint reached terminal | Tier 0, then delta |
| Queue drained while a member is free | Tier 0, then delta |
| `POST /api/groom` | full (operator asked) |
| Safety-net timer | Tier 0 (usually stops there) |
| Daily floor | full |

Timer: `setInterval` in serve.mjs, `FLEET_SE_GROOMER_INTERVAL_MS` **default 21600000 (6h, was 1h)**, 0 disables. `unref()` it. The interval change is safe precisely because a tick's default outcome is now a fingerprint comparison rather than an agent dispatch. Tier 2 cap via `FLEET_SE_GROOMER_MAX_FULL_PASSES_PER_DAY` (default 1, which is the daily floor).

Identity: `FLEET_SE_GROOMER_IDENTITY` required for autonomous runs; if unset, timer no-ops and POST /api/groom returns 400 naming the env var. Do not guess from `git config` in the supervisor (wrong user on the supervisor host).

- [ ] **Step 1: Write failing tests**

```js
test('run() POSTs pending sprints for quality-bar-passing queueRequests', async () => {});
test('run() does not launch a request whose issue is in needsGrooming', async () => {});
test('second run while first in flight returns busy', async () => {});
test('idempotent second run reuses sprint ids (launch reused:true)', async () => {});
test('POST /api/groom without identity config is 400', async () => {});

// round 2 -- cheap cadence (G2)
test('unchanged backlog fingerprint dispatches no agent at all', async () => {
  // runGroomer is a spy that fails the test if called
  // second run() with identical bd rows -> { status: 'unchanged' }, spy never invoked
});
test('a new bead changes the fingerprint and triggers a delta pass', async () => {
  // second run() sees one extra row -> runGroomer called with mode 'delta'
  // and changedBeadIds === [the new id]
});
test('a status or priority change alone changes the fingerprint', async () => {});
test('explicit POST /api/groom runs a full pass even when the fingerprint is unchanged', async () => {});
test('the daily full-pass cap is enforced across repeated escalations', async () => {});

// round 2 -- idempotency key (G5)
test('the key is derived from sorted beadIds, so reordered beadIds collide', async () => {
  // two runs emitting ['b','a'] then ['a','b'] -> second is reused:true, one sprint total
});

// round 2 -- absorption (G7)
test('absorb[] amends an existing WAITING sprint instead of creating a new one', async () => {
  // queue has waiting w1 with beadIds ['a']; output absorb [{ sprintId: 'w1', beadIds: ['b'] }]
  // -> w1 scope now covers a and b; no new sprint created
});
test('absorb targeting a RUNNING sprint is rejected by the runner, not sent to the API', async () => {});
```

- [ ] **Step 2: Run -- expect FAIL**

- [ ] **Step 3: Implement runner, routes, serve wiring, dashboard card showing last groom timestamp + notes + queued ids (not as a beads lecture -- just sprintIds).**

Production `runGroomer`: if no adapter is ready, implement a function that throws a clear "groomer spawn adapter not configured" and keep the HTTP/timer working with the seam. Do not block tests. A follow-up can wire `claude` / installed agent invocation; file a remaining issue in the PR description rather than silently shipping a no-op timer.

- [ ] **Step 4: Run groomer + dashboard tests. Expected PASS.**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): run backlog-groomer into the sprint queue"
```

---

### Task 4: Docs + skill for autonomous grooming

**Files:**
- Modify: `packages/apra-fleet-se/fleet-sprint/skills/fleet-supervisor/SKILL.md`
- Modify: `packages/apra-fleet-se/docs/supervisor-api.md`
- Modify: `packages/apra-fleet-se/docs/supervisor-openapi.yaml`
- Modify: `packages/apra-fleet-se/apra-pm/docs/sprint-workflow.md` only if it already describes groomer; otherwise skip.

- [ ] **Step 1: Document env vars, POST /api/groom, D1-D4 defaults, idempotency, caps, quality bar.** Curl example with empty members already in PR A; add groom example. Document the tiered cadence explicitly -- operators need to know that a tick usually costs nothing, what makes a pass escalate to a full agent run, and which knobs (`FLEET_SE_GROOMER_INTERVAL_MS`, `FLEET_SE_GROOMER_MAX_FULL_PASSES_PER_DAY`, `FLEET_SE_GROOMER_MAX_QUEUED`, `FLEET_SE_GROOMER_MAX_DAILY_BUDGET_USD`) bound spend.

- [ ] **Step 2: OpenAPI for POST /api/groom and 429 on POST /api/sprints.**

- [ ] **Step 3: Commit**

```bash
git commit -m "docs(supervisor): document autonomous grooming cadence and caps"
```

---

## Self-review vs issue #22

| Spec / issue | Task |
|---|---|
| Cheap automatic cadence (round 2) | 3 Tier 0 fingerprint, delta mode, 6h safety net, capped full passes |
| Absorb into existing queued sprints (round 2, G7) | 1 contract, 3 apply |
| Stable idempotency identity (round 2, G5) | 2 sorted-beadIds key |
| Capacity awareness | 1 input, 3 runner supplies it |
| Cadence + triggers | 3 timer, terminal kick, POST /api/groom |
| Queue approved sets via supervisor API | 3 |
| Human override ordering | already PR A; groomer only sets priority |
| Quality bar | 1 agent + 3 filter |
| Idempotent | 2 |
| Delivery landed-vs-closed + velocity | 1 delivery field, 3 velocityNote fallback |
| Budget guardrails | 2 caps |
| Grooming intelligence extensions | agent markdown already has stale P x age, dedupe, XL leaf; only extend where queueing needs it |
| Auto-assign members | explicitly deferred (D2 default false) |

Remaining work after this plan (file as follow-ups, do not silently expand):

- Production spawn adapter for the backlog-groomer agent (Claude/CLI).
- `queue.autoAssignMembers` if operators want hands-off member binding.
- Dashboard charts for velocity beyond a one-line note.
