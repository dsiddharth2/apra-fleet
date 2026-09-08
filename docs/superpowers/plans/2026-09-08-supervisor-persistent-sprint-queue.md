# Persistent Sprint Queue (#410 points 1-3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the fleet supervisor a durable sprint queue: zero-member PENDING creation, opt-in WAITING instead of 409 when members are busy, deterministic pick-next when capacity appears, and human-controllable ordering.

**Architecture:** A new `queue.mjs` store holds PENDING/WAITING/BLOCKED records (never reservations). `launch()` validates as today then either enqueues or spawns. `scheduler.mjs` `kick()` walks the queue in documented order and starts the first fully-eligible WAITING sprint with no pre-reservation. Ledger and history stay in their current roles.

**Tech Stack:** Node.js ESM (`packages/apra-fleet-se`), `node:test`, existing `createLedger` / `createHistory` / `createSpawner` seams, atomic JSON via `renameWithRetry`.

**Spec:** `docs/superpowers/specs/2026-09-08-supervisor-sprint-queue-and-autonomous-grooming-design.md` (PR A). Pause/resume and dashboard history browser are PR B, not this plan.

## Global Constraints

- ASCII only in every file.
- Do not cite bead ids (`apra-fleet-XXXX`) in LLM-facing strings (prompts, skill copy, schema descriptions, runtime print). Fine in code comments and `docs/`.
- Ledger remains "who holds a reservation RIGHT NOW". Queue records MUST NOT call `ledger.claim()` until pick-next actually starts the sprint.
- Default busy-member behavior stays 409. Queueing requires `queue: true` or empty members (design D5).
- Validation of issue/branch/base still runs at enqueue time using `validateIssueId` / `validateBranchName` from `fleet-sprint/runner.js`.
- Relaunch gate still runs at enqueue AND again at execution.
- `packages/apra-fleet-client` is unchanged in this PR (no MCP tool schema change).
- Never wrap around a permission-layer block.
- Member-bound command strings (if any) must not use `$VAR`, `~/`, or backticks; resolve paths in JS.

---

### Task 1: Queue store (`queue.mjs`)

**Files:**
- Create: `packages/apra-fleet-se/src/supervisor/queue.mjs`
- Create: `packages/apra-fleet-se/test/supervisor-queue.test.mjs`
- Consumes: `packages/apra-fleet-se/src/supervisor/rename-with-retry.mjs`, `ledger.mjs` durability pattern
- Produces: `createQueue`, `compareQueueOrder`, `QUEUE_FILENAME`, `QUEUE_VERSION`, `emptyQueueDocument`

**Interfaces:**
- Consumes: `renameWithRetry(from, to)`
- Produces:

```js
export const QUEUE_VERSION = 1;
export const QUEUE_FILENAME = 'sprint-queue.json';
export function emptyQueueDocument() {
  return { version: QUEUE_VERSION, items: {} };
}
export function compareQueueOrder(a, b) { /* pinNext desc, priority desc, createdAt asc, sprintId asc */ }
export function createQueue(opts = {}) {
  // opts.filePath, opts.now
  return {
    start: async () => {},
    get: (sprintId) => record|undefined,
    list: (filter) => record[],          // filter.state optional string or string[]
    put: async (record) => record,       // upsert, atomic
    remove: async (sprintId) => boolean, // true if existed
    clearPinNextExcept: async (sprintId) => {},
  };
}
```

Queue record shape (normalize missing fields on load, same as `normalizeReservation`):

```
{ sprintId, state, request, members, issueRoots, priority, pinNext,
  createdAt, updatedAt, skippedCount, blockedReason, childPid, childPort,
  snapshot, idempotencyKey, origin, audit }
```

- [ ] **Step 1: Write the failing tests**

Create `packages/apra-fleet-se/test/supervisor-queue.test.mjs`:

```js
import { test, describe } from 'node:test';
import assert from 'node:assert';
import fsp from 'node:fs/promises';
import path from 'node:path';
import os from 'node:os';
import { createQueue, QUEUE_FILENAME, compareQueueOrder } from '../src/supervisor/queue.mjs';

async function tmpDir() {
  return fsp.mkdtemp(path.join(os.tmpdir(), 'queue-'));
}

function rec(over) {
  return {
    sprintId: 'issue-aaaa',
    state: 'pending',
    request: { issue: 'issue', branch: 'feat/x', base: 'main' },
    members: [],
    issueRoots: ['issue'],
    priority: 0,
    pinNext: false,
    createdAt: '2026-09-08T00:00:00.000Z',
    updatedAt: '2026-09-08T00:00:00.000Z',
    skippedCount: 0,
    blockedReason: null,
    childPid: null,
    childPort: null,
    snapshot: null,
    idempotencyKey: null,
    origin: 'api',
    audit: null,
    ...over,
  };
}

describe('queue store', () => {
  test('put then get survives process restart (reload from disk)', async () => {
    const dir = await tmpDir();
    const filePath = path.join(dir, QUEUE_FILENAME);
    const q1 = createQueue({ filePath, now: () => '2026-09-08T00:00:00.000Z' });
    await q1.start();
    await q1.put(rec({ sprintId: 's1' }));
    const q2 = createQueue({ filePath, now: () => '2026-09-08T00:00:00.000Z' });
    await q2.start();
    assert.equal(q2.get('s1').state, 'pending');
    assert.equal(q2.get('s1').request.branch, 'feat/x');
  });

  test('compareQueueOrder: pinNext beats priority; higher priority beats older createdAt', () => {
    const pin = rec({ sprintId: 'p', pinNext: true, priority: 0, createdAt: '2026-09-08T02:00:00.000Z' });
    const high = rec({ sprintId: 'h', priority: 10, createdAt: '2026-09-08T01:00:00.000Z' });
    const old = rec({ sprintId: 'o', priority: 0, createdAt: '2026-09-08T00:00:00.000Z' });
    const sorted = [old, high, pin].sort(compareQueueOrder);
    assert.deepEqual(sorted.map((r) => r.sprintId), ['p', 'h', 'o']);
  });

  test('list({ state: "waiting" }) does not return pending', async () => {
    const dir = await tmpDir();
    const q = createQueue({ filePath: path.join(dir, QUEUE_FILENAME) });
    await q.start();
    await q.put(rec({ sprintId: 'a', state: 'pending' }));
    await q.put(rec({ sprintId: 'b', state: 'waiting', members: ['alice'] }));
    assert.deepEqual(q.list({ state: 'waiting' }).map((r) => r.sprintId), ['b']);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `node --test packages/apra-fleet-se/test/supervisor-queue.test.mjs`

Expected: FAIL with `Cannot find module` for `queue.mjs`.

- [ ] **Step 3: Write minimal implementation**

Mirror `ledger.mjs`: `start()` mkdir + read-or-empty, `put`/`remove` write a temp file then `renameWithRetry`, commit in-memory only after the rename succeeds. `compareQueueOrder`:

```js
export function compareQueueOrder(a, b) {
  const pin = Number(Boolean(b.pinNext)) - Number(Boolean(a.pinNext));
  if (pin !== 0) return pin;
  const pri = (b.priority ?? 0) - (a.priority ?? 0);
  if (pri !== 0) return pri;
  const t = String(a.createdAt ?? '').localeCompare(String(b.createdAt ?? ''));
  if (t !== 0) return t;
  return String(a.sprintId).localeCompare(String(b.sprintId));
}
```

`list()` returns `Object.values(items)` filtered then sorted with `compareQueueOrder`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --test packages/apra-fleet-se/test/supervisor-queue.test.mjs`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/apra-fleet-se/src/supervisor/queue.mjs packages/apra-fleet-se/test/supervisor-queue.test.mjs
git commit -m "feat(supervisor): add durable sprint queue store"
```

---

### Task 2: PENDING creation (zero members)

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs` (`validateLaunchRequest`, `launch`, `listSprints`, `getSprint`, `createSprintController` deps)
- Modify: `packages/apra-fleet-se/bin/serve.mjs` (construct queue, inject)
- Create: `packages/apra-fleet-se/test/supervisor-api-queue.test.mjs`
- Modify: `packages/apra-fleet-se/test/supervisor-api.test.mjs` only if an existing empty-members test must stay 400 -- replace that expectation: empty members is now 201 pending.

**Interfaces:**
- Consumes: `createQueue` from Task 1; existing `generateSprintId`, `validateIssueId`, `validateBranchName`
- Produces: `launch({ issue, branch, base, members: [] })` -> `{ sprintId, state: 'pending', pid: null, port: null, issueRoots, members: [], goal }` status 201. No `spawner.spawnSprint`, no `ledger.claim`.

- [ ] **Step 1: Write the failing tests**

In `supervisor-api-queue.test.mjs` reuse `stores` / `recordingSpawner` / `createSprintController` pattern from `supervisor-api.test.mjs`:

```js
test('POST /api/sprints with members [] returns 201 pending, does not spawn or claim', async () => {
  const dir = await tmpDir();
  const { ledger, history } = await stores(dir);
  const captured = [];
  const queue = createQueue({ filePath: path.join(dir, QUEUE_FILENAME) });
  await queue.start();
  const controller = createSprintController({
    ledger, history, queue,
    spawner: recordingSpawner(captured),
    generateSprintId: () => 'issue-fixed',
    listMembers: async () => ({ members: [{ name: 'alice' }] }),
  });
  const result = await controller.launch({
    issue: 'issue', branch: 'feat/x', base: 'main', members: [], goal: 'P1',
  });
  assert.equal(result.state, 'pending');
  assert.equal(result.sprintId, 'issue-fixed');
  assert.equal(result.pid, null);
  assert.equal(captured.length, 0);
  assert.equal(ledger.list().length, 0);
  assert.equal(queue.get('issue-fixed').state, 'pending');
  assert.equal(queue.get('issue-fixed').request.goal, 'P1');
});

test('PENDING still 400s on a malformed issue id (validation is not deferred)', async () => {
  // same setup, issue: 'bad issue' with a space
  await assert.rejects(
    () => controller.launch({ issue: 'bad issue', branch: 'feat/x', base: 'main', members: [] }),
    (err) => err instanceof ApiError && err.status === 400 && err.field === 'issue',
  );
});
```

Also: omitted `members` (undefined) behaves like `[]`. `GET` list includes the pending item with `state: 'pending'`. `GET /:id` returns the stored request. Cancel via new `controller.cancelSprint(id)` or `DELETE` -- if DELETE is not wired yet, `queue.remove` coverage can wait for Task 4; at minimum `getSprint` must find it.

- [ ] **Step 2: Run tests to verify they fail**

Run: `node --test packages/apra-fleet-se/test/supervisor-api-queue.test.mjs`

Expected: FAIL -- `members must be a non-empty list` and/or `createSprintController requires` no `queue`.

- [ ] **Step 3: Write minimal implementation**

In `validateLaunchRequest`, delete the `members.length === 0` throw. Empty members is valid.

In `createSprintController`, require `deps.queue` with `get/list/put/remove` (or default a throwing stub only in tests that still inject everything -- prefer requiring it so serve.mjs must wire it).

In `launch()` after relaunch gate:

```js
if (members.length === 0 && memberUnion(members, roleMap, unreservable).length === 0) {
  const sprintId = generateSprintId(issue);
  await queue.put({
    sprintId,
    state: 'pending',
    request: { ...body, issue, branch, base, members: [], goal: body.goal ?? null },
    members: [],
    issueRoots,
    priority: Number.isFinite(body.priority) ? body.priority : 0,
    pinNext: false,
    createdAt: now(),
    updatedAt: now(),
    skippedCount: 0,
    blockedReason: null,
    childPid: null,
    childPort: null,
    snapshot: null,
    idempotencyKey: body.idempotencyKey ?? null,
    origin: body.origin ?? 'api',
    audit: null,
  });
  return { sprintId, state: 'pending', pid: null, port: null, logPath: null, issueRoots, members: [], goal: body.goal ?? null, buildVersionWarning: null };
}
```

Inject `now` the same way ledger does, or use `new Date().toISOString()`.

`listSprints()` concatenates ledger-running items (`state: 'running'`) with `queue.list()` already sorted.

`getSprint(id)`: if `queue.get(id)` return that record before falling through to history.

Wire `createQueue` in `serve.mjs` next to `createLedger` (`path.join(dataDir, QUEUE_FILENAME)`).

- [ ] **Step 4: Run tests**

Run:

```
node --test packages/apra-fleet-se/test/supervisor-api-queue.test.mjs packages/apra-fleet-se/test/supervisor-api.test.mjs
```

Expected: new tests PASS. Existing tests PASS. If an old test asserted empty members -> 400, update it to the PENDING behavior and note the contract change in the test title.

- [ ] **Step 5: Commit**

```bash
git add packages/apra-fleet-se/src/supervisor/api.mjs packages/apra-fleet-se/bin/serve.mjs packages/apra-fleet-se/test/supervisor-api-queue.test.mjs packages/apra-fleet-se/test/supervisor-api.test.mjs
git commit -m "feat(supervisor): accept zero-member PENDING sprint creation"
```

---

### Task 3: WAITING on busy members (`queue: true`)

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs` (`launch`, overlap handling)
- Modify: `packages/apra-fleet-se/test/supervisor-api-queue.test.mjs`
- Leave: existing 409 tests in `supervisor-api.test.mjs` (default `queue` false)

**Interfaces:**
- Consumes: `defaultMemberOverlapGuard` (keep throwing 409)
- Produces: when `body.queue === true` and the guard would 409, catch `ApiError` status 409 field `members`, `queue.put` state `waiting`, return 201 `{ state: 'waiting', sprintId, members: union, pid: null }`. Scope-overlap 409s are NOT converted to WAITING (issue conflict is not "busy member"); they still reject at enqueue for named-member launches. PENDING has no members so the member guard is skipped.

- [ ] **Step 1: Write the failing tests**

```js
test('queue:true + busy member => 201 waiting, ledger unchanged, no spawn', async () => {
  await ledger.claim('other', { members: ['alice'], issueRoots: ['other'], childPid: 1, branch: 'feat/o', base: 'main', goal: null });
  const result = await controller.launch({
    issue: 'issue', branch: 'feat/x', base: 'main', members: ['alice'], queue: true,
  });
  assert.equal(result.state, 'waiting');
  assert.equal(captured.length, 0);
  assert.equal(queue.get(result.sprintId).state, 'waiting');
  assert.deepEqual(ledger.get('other').members, ['alice']);
});

test('busy member without queue:true still 409 (launch-now default)', async () => {
  await ledger.claim('other', { members: ['alice'], issueRoots: ['other'], childPid: 1, branch: 'feat/o', base: 'main', goal: null });
  await assert.rejects(
    () => controller.launch({ issue: 'issue', branch: 'feat/x', base: 'main', members: ['alice'] }),
    (err) => err instanceof ApiError && err.status === 409 && err.field === 'members',
  );
});

test('queue:true + free member still launches immediately (state running)', async () => {
  const result = await controller.launch({
    issue: 'issue', branch: 'feat/x', base: 'main', members: ['alice'], queue: true,
  });
  assert.equal(result.state, 'running');
  assert.equal(captured.length, 1);
  assert.ok(ledger.get(result.sprintId));
  assert.equal(queue.get(result.sprintId), undefined);
});
```

- [ ] **Step 2: Run tests -- expect FAIL** (waiting path does not exist)

- [ ] **Step 3: Implement**

```js
let overlapError = null;
try {
  await beforeLaunch({ members: union, issueRoots, membersList: membersListRaw });
} catch (err) {
  if (body.queue === true && err instanceof ApiError && err.status === 409 && err.field === 'members') {
    overlapError = err;
  } else {
    throw err;
  }
}
if (overlapError) {
  const sprintId = generateSprintId(issue);
  await queue.put({ /* waiting record with members: union */ state: 'waiting', members: union, ... });
  return { sprintId, state: 'waiting', pid: null, port: null, issueRoots, members: union, goal: body.goal ?? null };
}
// existing spawn + claim, then:
return { ..., state: 'running' };
```

Add `state: 'running'` to the existing success payload (backward compatible extra field). Update any strict deepEqual on launch result if tests break.

- [ ] **Step 4: Run**

```
node --test packages/apra-fleet-se/test/supervisor-api-queue.test.mjs packages/apra-fleet-se/test/supervisor-api.test.mjs packages/apra-fleet-se/test/reservation-interop-e2e.test.mjs
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): queue busy launches as WAITING when queue:true"
```

---

### Task 4: Scheduler pick-next + kick on capacity

**Files:**
- Create: `packages/apra-fleet-se/src/supervisor/scheduler.mjs`
- Create: `packages/apra-fleet-se/test/supervisor-scheduler.test.mjs`
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs` (extract `startQueued(record)` used by both launch-now and scheduler)
- Modify: `packages/apra-fleet-se/src/supervisor/watchdog.mjs` (`releaseTerminalReservation` calls `onCapacity()` after a successful release)
- Modify: `packages/apra-fleet-se/bin/serve.mjs`

**Interfaces:**
- Consumes: `queue.list`, `queue.put`, `queue.remove`, `ledger.list/claim`, `listMembers`, `controller.startQueued` or injected `startSprint(record)`
- Produces:

```js
export function isUnionFree(union, { ledgerEntries, membersList }) { /* boolean */ }
export function createScheduler(deps) {
  return {
    kick: async () => ({ started: string|null, skipped: string[] }),
  };
}
```

`startQueued(record)` must re-run: relaunch gate, member overlap against **live** free set, issue-scope overlap, then spawn+claim, then `queue.remove(id)`. On overlap at start: leave record WAITING (member busy) or set BLOCKED (scope conflict / missing member). Never claim then fail spawn without `release`.

Member deleted: if every requested member is absent from `listMembers()`, set `state: 'blocked'`, `blockedReason: "member 'x' is not registered"`.

- [ ] **Step 1: Write failing tests** in `supervisor-scheduler.test.mjs`

```js
test('kick starts the highest-priority waiting sprint whose entire union is free', async () => {
  // queue: s-low priority 0 members [alice], s-high priority 5 members [alice]
  // ledger empty, alice free
  // startSprint records which id
  // kick -> started s-high, queue no longer has s-high, s-low remains waiting
});

test('kick does not start a waiting sprint if any requested member is busy (no pre-reservation)', async () => {
  // alice claimed by running r1; waiting wants alice
  // kick -> started null; waiting still waiting; ledger.list still [r1] only
});

test('multi-member waiting starts only when the entire union is free; skip to next eligible', async () => {
  // w-ab wants [alice, bob], w-c wants [carol]; alice free, bob busy, carol free
  // kick starts w-c, increments skippedCount on w-ab, does not claim alice for w-ab
});

test('waiting whose member is unregistered becomes blocked', async () => {
  // listMembers returns [carol] only; waiting wants [ghost]
  // kick -> blockedReason matches /not registered/
});

test('kick is serialized: overlapping kicks run once more if dirty', async () => {
  // optional if cheap; otherwise skip and document single-flight in implementation
});
```

- [ ] **Step 2: Run -- expect FAIL** (module missing)

- [ ] **Step 3: Implement `scheduler.mjs` + `startQueued` + watchdog hook**

`isUnionFree`:

```js
export function isUnionFree(union, { ledgerEntries, membersList }) {
  const busy = new Set();
  for (const r of ledgerEntries) for (const m of (r.members ?? [])) busy.add(m);
  const list = Array.isArray(membersList) ? membersList : (membersList?.members ?? []);
  const known = new Set(list.map((m) => (typeof m === 'string' ? m : m.name)));
  const reservedByServer = new Set(list.filter((m) => m && m.reservedBy && !m.unreservable).map((m) => m.name));
  for (const m of union) {
    if (busy.has(m) || reservedByServer.has(m)) return false;
    if (known.size > 0 && !known.has(m)) return false;
  }
  return union.length > 0;
}
```

Unregistered vs busy: if `known.size > 0 && !known.has(m)` the scheduler sets BLOCKED rather than treating as not-free (test above). Split: missing -> blocked; busy -> skip.

Wire `createWatchdog({ ..., onCapacity: scheduler.kick })` and call it at the end of `releaseTerminalReservation` only when `released === true`. Also call `kick` at end of successful `launch` WAITING/PENDING paths? Not required for PENDING; for WAITING a kick on another sprint ending is the path. Call kick once at serve startup after readopt/reconcile so a WAITING record from disk can start if members are free.

- [ ] **Step 4: Run**

```
node --test packages/apra-fleet-se/test/supervisor-scheduler.test.mjs packages/apra-fleet-se/test/supervisor-watchdog.test.mjs packages/apra-fleet-se/test/supervisor-api-queue.test.mjs
```

Expected: PASS. Watchdog tests that inject a ledger without `onCapacity` must still work (`onCapacity` optional no-op).

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): start WAITING sprints when members become free"
```

---

### Task 5: Human ordering API + audit + cancel

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs` (`patchSprint`, `reorderSprint`, `cancelSprint`, `registerSprintRoutes`)
- Modify: `packages/apra-fleet-se/src/supervisor/history.mjs` (add `QUEUE_REORDERED`, `QUEUE_CANCELLED`)
- Modify: `packages/apra-fleet-se/test/supervisor-api-queue.test.mjs`
- Modify: `packages/apra-fleet-se/test/supervisor-history.test.mjs` if HISTORY_EVENTS exhaustiveness is tested

**Interfaces:**
- `PATCH /api/sprints/:id` body `{ priority?, pinNext?, members?, reason? }`
- `POST /api/sprints/:id/reorder` body `{ action: "front"|"back", reason? }`
- `DELETE /api/sprints/:id`
- RUNNING patch -> 409
- `pinNext: true` clears pin on every other queue record (`queue.clearPinNextExcept`)
- history.record `{ event: 'queue-reordered', sprintId, reason, actor }`

- [ ] **Step 1: Write failing tests**

```js
test('PATCH priority changes consume order on GET list', async () => { /* two pending, patch one to 10, list queue ids [high, low] */ });
test('POST reorder action front sets pinNext and clears the previous pin', async () => {});
test('PATCH members on pending transitions to waiting (or running if free)', async () => {});
test('PATCH a running sprint returns 409', async () => {});
test('DELETE pending removes the record and is gone from GET', async () => {});
test('reorder writes a history event with reason', async () => {});
```

- [ ] **Step 2: Run -- expect FAIL**

- [ ] **Step 3: Implement routes**

```js
supervisor.route('PATCH', '/api/sprints/:id', ...);
supervisor.route('POST', '/api/sprints/:id/reorder', ...);
supervisor.route('DELETE', '/api/sprints/:id', ...);
```

After PATCH members onto a pending item, call `scheduler.kick()`.

- [ ] **Step 4: Run** the api-queue + history tests. Expected PASS.

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): add queue reorder, patch, and cancel APIs"
```

---

### Task 6: Dashboard queue section + optional members on launch form

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/dashboard.mjs`
- Modify: `packages/apra-fleet-se/src/supervisor/launch-form.mjs`
- Modify: `packages/apra-fleet-se/test/supervisor-dashboard.test.mjs`
- Modify: `packages/apra-fleet-se/test/supervisor-launch-form.test.mjs`

**Interfaces:**
- `renderQueueHtml(records)` -- consume order, shows state/priority/pin, buttons data attributes for patch/reorder/delete
- `renderIndexPageHtml(views, backlogHtml, launchFormHtml, queueRecords)`
- Launch form: members may be empty; checkbox `queue if busy` -> `queue: true` in `buildLaunchRequestBody`

Pause/resume wrapper, recent-5 history, clone/relaunch UI are **out of this task** (PR B).

- [ ] **Step 1: Write failing tests**

```js
test('queue section renders pending/waiting in consume order', () => {
  const html = renderQueueHtml([
    { sprintId: 'a', state: 'waiting', priority: 5, pinNext: false, issueRoots: ['i'] },
    { sprintId: 'b', state: 'pending', priority: 0, pinNext: false, issueRoots: ['j'] },
  ]);
  assert.ok(html.indexOf('a') < html.indexOf('b'));
  assert.match(html, /waiting/);
});

test('launch form allows empty members (queue as pending)', () => {
  const body = buildLaunchRequestBody({ issue: 'i', branch: 'feat/x', base: 'main', members: [] });
  assert.deepEqual(body.members, []);
});
```

Extend `renderIndexPageHtml` test: page contains `id="sprint-queue"`.

- [ ] **Step 2: Run dashboard + launch-form tests -- expect FAIL**

- [ ] **Step 3: Implement HTML + client script**

Follow existing `SPRINT_STOP_SCRIPT` pattern: delegated clicks POST/PATCH fetch, inline result div. Do not use the kill/force-release route for cancel of a pending item -- DELETE `/api/sprints/:id`.

Header stats: waiting/pending counts.

`createDashboard` must receive `listQueue: () => queue.list()`.

- [ ] **Step 4: Run**

```
node --test packages/apra-fleet-se/test/supervisor-dashboard.test.mjs packages/apra-fleet-se/test/supervisor-launch-form.test.mjs packages/apra-fleet-se/test/supervisor-dashboard-live-refresh.test.mjs
```

Expected: PASS. Update live-refresh tests if the poll payload shape grew (include `queue`).

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): show sprint queue on the dashboard"
```

---

### Task 7: Docs contract (OpenAPI, supervisor-api.md, SKILL.md)

**Files:**
- Modify: `packages/apra-fleet-se/docs/supervisor-openapi.yaml`
- Modify: `packages/apra-fleet-se/docs/supervisor-api.md`
- Modify: `packages/apra-fleet-se/fleet-sprint/skills/fleet-supervisor/SKILL.md`

- [ ] **Step 1: Update OpenAPI**

- `POST /api/sprints`: `members` not required; add `queue`, `priority`, `idempotencyKey`; 201 schema includes `state` enum `pending|waiting|running`; `pid`/`port` nullable.
- `GET /api/sprints`: items include `state`, `priority`, `pinNext`, `createdAt`.
- Add PATCH `/api/sprints/{id}`, POST `/api/sprints/{id}/reorder`, DELETE `/api/sprints/{id}`.
- Fix the stale "KNOWN GAP" note that says `issue` is single-id only (already wrong vs ymf.1) while you are in the file -- only if you touch that block; do not drive-by the rest of the yaml.

- [ ] **Step 2: Update supervisor-api.md**

Document: enqueue vs spawn; D5 fail-fast default; pick-next comparator; no pre-reservation; kick triggers; PATCH running 409.

- [ ] **Step 3: Update SKILL.md**

Show curl examples:

```
# queue work without choosing a member
curl -s -X POST http://localhost:8787/api/sprints \
  -H 'content-type: application/json' \
  -d '{"issue":"ISSUE","branch":"feat/topic","base":"main","members":[],"goal":"P1"}'

# wait if busy
curl -s -X POST http://localhost:8787/api/sprints \
  -H 'content-type: application/json' \
  -d '{"issue":"ISSUE","branch":"feat/topic","base":"main","members":["alice"],"queue":true}'
```

Never mention bead ids in the skill.

- [ ] **Step 4: Run a docs-sensitive test if one exists** (openapi drift). If none, skip.

- [ ] **Step 5: Commit**

```bash
git commit -m "docs(supervisor): document persistent sprint queue API"
```

---

## Self-review (this plan vs spec PR A)

| Spec requirement | Task |
|---|---|
| PENDING zero members, persisted, no reservation | 2 |
| GET lists pending distinguishable by state | 2, 5 |
| GET :id returns full stored request | 2 |
| Cancel without running | 5 |
| Validation at creation | 2 |
| WAITING on busy with queue:true; default 409 | 3 |
| Auto-start when free; no pre-reservation | 4 |
| Multi-member all-or-nothing; skip ineligible | 4 |
| Member deleted -> blocked | 4 |
| Safety gates at execution | 4 (`startQueued`) |
| Priority/rank + reorder API + durable + audit | 5 |
| GET in consume order | 1 sort + 2 list |
| Dashboard queue visible | 6 |
| OpenAPI/SKILL | 7 |
| Pause/resume, history 5, clone, relaunch | **not this plan (PR B)** |
