# Autonomous Backlog Grooming Pipeline (#22) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn backlog-groomer from an advice-only agent into the producer of queued sprints: capacity-aware, quality-bar-gated, idempotent POSTs to the supervisor queue, run on a cadence with delivery tracking.

**Architecture:** Extend groomer input/output schemas and the agent markdown. Supervisor `groomer.mjs` supplies capacity, invokes a `runGroomer(input)` seam, then POSTs `queueRequests` as PENDING (`members: []`, `queue: true`) through the same controller as humans. Caps (`maxQueued`, optional daily budget) are enforced in the controller. Timer + terminal-sprint trigger + `POST /api/groom`.

**Tech Stack:** Existing backlog-groomer agent + JSON schemas; supervisor HTTP; `node:test`. Depends on PR A (PENDING + idempotencyKey). Pause/history (PR B) not required for the first groomer increment.

**Spec:** `docs/superpowers/specs/2026-09-08-supervisor-sprint-queue-and-autonomous-grooming-design.md` sections 4 and 10. Product defaults D1-D4 (PENDING, opt-in auto-assign, 1h cadence, maxQueued=3).

## Global Constraints

- ASCII only. Never cite bead ids in LLM-facing prompt/skill/schema/runtime strings the runner adds.
- Groomer still never edits source code or runs git writes.
- Groomer never selects machines. It queues PENDING only.
- `queue.autoAssignMembers` default false (D2). Do not implement auto-assign in the first increment unless the dashboard assign-members path from PR A is already enough for operators.
- Idempotent: same `idempotencyKey` must not create a second PENDING/WAITING/RUNNING/PAUSED sprint.
- Quality bar: never enqueue items in needsGrooming / needsVerification / unresolved deadlock.
- Caps enforced in the supervisor even if the agent ignores them.
- `packages/apra-fleet-client` unchanged unless a fleet MCP tool is added (do not add MCP in this plan).
- PowerShell-safe: any command string the runner builds for a member must not use `$VAR`/`~/`/backticks. Groomer branch names are concrete strings.

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

- Output adds `queueRequests` and `delivery` as in the spec.

Agent markdown: new last step "Queue, do not advise-only" when `dry-run` is false AND capacity is present: emit `queueRequests` for sprint sets that pass the quality bar, sized so `queuedCount + new <= maxQueued`. Each request: `issue` (comma-joined bead ids of the set's roots -- use the beads' actual ids from `bd`, not invented ids), concrete `branch` (`feat/<short-ascii-slug>`), `base` (current default branch name from `git rev-parse --abbrev-ref origin/HEAD` or `main`), `goal`, `priority` (numeric from beads priority), `idempotencyKey` description (sorted issue roots + branch -- the runner will hash if the agent supplies the raw key string). `skippedReason` when a set is withheld.

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
    queueRequests: [{
      issue: 'iss-1', branch: 'feat/iss-1', base: 'main', goal: 'P1',
      priority: 2, idempotencyKey: 'iss-1|feat/iss-1', budget: 5,
    }],
    delivery: { verifiedClosed: [], stillOpen: [], atRisk: [], velocityNote: '0 finished in last 7d' },
  };
}
```

Runner logic (pure enough to unit test):

1. If `isRunning()`, return `{ status: 'busy' }` without starting a second agent.
2. Build capacity from `listMembers` + ledger + queue.
3. `output = await runGroomer({ identity, dry-run: false, mode: [...], capacity })`.
4. Filter `queueRequests`: drop if `skippedReason`; drop if issue is in output.needsGrooming or needsVerification ids; drop if structuralIssues deadlock mentions that id.
5. For each remaining, `launch({ ..., members: [], queue: true, origin: 'groomer', enforceQueueCaps: true, idempotencyKey })`. Collect 429 as skipped.
6. Write `<dataDir>/groomer-last.json`.
7. Velocity: count history events finished vs crashed in last 7d; pass into input or stamp `delivery.velocityNote` if the agent left it empty.

`POST /api/groom` -> 202 `{ status: 'started' }` or 200 `{ status: 'busy' }`.

Timer: `setInterval` in serve.mjs, `groomer.intervalMs` from `FLEET_SE_GROOMER_INTERVAL_MS` default 3600000, 0 disables. `unref()` the interval so it does not keep the process alive if tests boot serve.

Identity: `FLEET_SE_GROOMER_IDENTITY` required for autonomous runs; if unset, timer no-ops and POST /api/groom returns 400 naming the env var. Do not guess from `git config` in the supervisor (wrong user on the supervisor host).

- [ ] **Step 1: Write failing tests**

```js
test('run() POSTs pending sprints for quality-bar-passing queueRequests', async () => {});
test('run() does not launch a request whose issue is in needsGrooming', async () => {});
test('second run while first in flight returns busy', async () => {});
test('idempotent second run reuses sprint ids (launch reused:true)', async () => {});
test('POST /api/groom without identity config is 400', async () => {});
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

- [ ] **Step 1: Document env vars, POST /api/groom, D1-D4 defaults, idempotency, caps, quality bar.** Curl example with empty members already in PR A; add groom example.

- [ ] **Step 2: OpenAPI for POST /api/groom and 429 on POST /api/sprints.**

- [ ] **Step 3: Commit**

```bash
git commit -m "docs(supervisor): document autonomous grooming cadence and caps"
```

---

## Self-review vs issue #22

| Spec / issue | Task |
|---|---|
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
