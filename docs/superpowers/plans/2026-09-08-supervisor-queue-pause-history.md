# Sprint Queue Pause, Resume, History, Clone (#410 points 4-5) Implementation Plan

> **Approval doc (send this to Akhil, not this file):** `docs/proposals/issue-410-persistent-sprint-queue.md` (slice B)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Use the existing engine pause/resume primitive so a running sprint can free its members into the queue, resume later (possibly on a different member) without resetting budget, and make the stored sprint request the single source for restart / relaunch / clone, with RUNNING edits rejected server-side.

**Architecture:** `POST /api/sprints/:id/pause` proxies the child's `/pause`, then `ledger.release()` and writes a PAUSED queue record with a snapshot (HEAD, cycle, budget consumed, parked pid/port). `POST /resume` moves it to WAITING and `scheduler.kick()`. Different members: stop the parked child and respawn from snapshot HEAD. Restart recovery gains the paused queue records as a second source. History list endpoint pages terminal sprints; restart/relaunch/clone all share one prefill-then-submit path validated exactly like POST /api/sprints.

**This is not a wrapper -- it changes a documented invariant.** `watchdog.mjs` states, in its own comment, that a live HTTP-reachable child reporting `state.pause.status === 'paused'` is classified PAUSED and is *never* routed through `releaseTerminalReservation()`; the module header calls the reservation of a paused sprint "never auto-released". That rule exists to stop a parked run from losing its members out from under it.

The resolution is a behavior **split**, not a blanket inversion:

- **Engine-initiated pause** (e.g. sprint-doctor's `pause_for_human`): unchanged. Stays on the ledger, classified PAUSED, keeps its members. The watchdog still never auto-releases it, exactly as documented.
- **Requested pause-into-queue** (this new endpoint only): releases the ledger, because #410 point 4 requires pause to genuinely free capacity rather than merely mark the sprint idle.

The distinguishing signal is the durable PAUSED **queue record**, which only the new endpoint writes. The watchdog keeps its rule as-is; release becomes an explicit operator/scheduler action with a queue record behind it. Say this in the code comments too -- do not leave a future reader thinking the invariant was simply dropped.

**Tech Stack:** Same supervisor ESM stack as PR A. Depends on PR A (`queue.mjs`, `scheduler.mjs`, WAITING/PENDING).

**Spec:** `docs/superpowers/specs/2026-09-08-supervisor-sprint-queue-and-autonomous-grooming-design.md` sections 7.6, 8, 9.

## Global Constraints

- ASCII only. No bead ids in LLM-facing skill/schema/runtime strings.
- Do not rebuild `requestPause` / `requestResume` in the workflow engine. Supervisor drives them.
- Only a **requested** pause releases the supervisor ledger. An engine-initiated pause keeps its members (see Architecture above).
- **Pause must leave no reservation for the member in EITHER source.** Two places mark a member taken: this supervisor's ledger, and the fleet server's per-member `reservedBy`, which `defaultMemberOverlapGuard` also consults. Clearing only the first means the next launch is still rejected by the second and pause frees nothing in practice. The fleet-side release already exists (the runner releases each `member_reservation` on `'paused'` and re-reserves, owner-checked, on resume) -- do not reimplement it. The ledger release is the new half.
- **Do not wire `ledger.mjs`'s `reservationClient` in this work.** It is accepted but unwired (`bin/serve.mjs` calls `createLedger()` with no deps, so only the read side is live), and wiring it is a separate change with its own blast radius. Route the release through `ledger.release()` so that if the client is wired later, the mirrored release happens automatically with no pause-specific second code path.
- No speculative pre-reservation while PAUSED or WAITING.
- RUNNING in-place PATCH remains 409.
- Budget consumed carries across pause; remaining cap is `originalCap - snapshot.budgetConsumedUsd`.
- Diverged branch vs snapshot `headSha` fails resume with 409 naming both SHAs.
- Permission blocks are surfaced, not bypassed.

---

### Task 0: Spike -- can a paused run persist-and-exit resumably?

**Not a code-delivery task. It decides Task 1b's design and must finish before Task 1 lands.**

The blind spot it resolves: restart recovery is PID-probe-driven **off ledger entries**. `reconcile()` iterates `ledger.list()`, and `readopt()` recovers `--viewer-port` only for entries that pass retained. Once pause releases the ledger row, a paused sprint with a live parked PID has no ledger row, so neither pass can see it -- it survives a supervisor restart as an orphan: alive, holding a port, invisible.

**Question:** can `requestPause` persist enough state that the child process **exits**, with the run resumable from disk, without the run being treated as terminal?

- **If yes -> Option B.** No parked PID, so no orphan and no blind spot. Resume always respawns, which is the path we must build anyway because #410 requires resume onto a *different* member. Two resume paths (proxy `/resume` to a parked child vs respawn elsewhere) collapse into one. Task 1b then only has to assert that no paused record ever carries a `childPid`.
- **If no -> Option A.** Keep the parked child and implement Task 1b as written below.

Record the answer in this file before starting Task 1. Do not infer the answer from doc comments about `'paused'` being a resumable status -- verify against the engine.

---

### Task 1b: Restart recovery must see paused queue records (Option A)

**Skip this task entirely if Task 0 chose Option B.**

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/reconcile.mjs` (second source pass)
- Modify: `packages/apra-fleet-se/src/supervisor/readopt.mjs` (re-adopt live parked children)
- Modify: `packages/apra-fleet-se/test/supervisor-reconcile.test.mjs`, `supervisor-readopt.test.mjs`

**Behavior:**
1. `reconcile()`'s ledger pass is unchanged -- do not alter dead-entry release or the aborted-by-restart history event.
2. A new pass walks `queue.list({ state: 'paused' })`. For each record with a non-null `childPid`:
   - **PID alive** and its cmdline still carries this sprint's `--viewer-port` marker: re-adopt it (register the port with the spawner) so watchdog/proxy/dashboard can reach it. The record stays PAUSED and holds **no** reservation -- correct, and must not be "repaired" into a ledger claim.
   - **PID gone:** set `childPid: null`, leave the record PAUSED (resume respawns from the snapshot), and record a history event so the transition is auditable rather than silent.
3. Reuse `parseViewerPortFromCmdline()` and the existing `readCmdline` seam; do not add a second cmdline parser.

**Tests:**

```js
test('restart re-adopts a paused sprint whose parked child is still alive', async () => {
  // queue has s1 paused with childPid 5000; ledger EMPTY; pid 5000 alive with --viewer-port 4310
  // after readopt: spawner.adopt called with (5000, 4310); queue.get('s1').state === 'paused'
  // and ledger.get('s1') is still undefined (no reservation resurrected)
});

test('restart clears childPid for a paused sprint whose parked child died', async () => {
  // pid not alive -> queue record stays paused, childPid null, history event recorded
});

test('ledger reconcile behavior is unchanged by the paused-record pass', async () => {
  // existing dead/live ledger assertions still hold with a paused queue record present
});
```

- [ ] **Step 1: Write the failing tests above**
- [ ] **Step 2: Run -- expect FAIL**
- [ ] **Step 3: Implement the second-source pass in reconcile/readopt**
- [ ] **Step 4: Run the full supervisor suite** -- restart recovery is load-bearing; a regression here loses live sprints.

```bash
node --test packages/apra-fleet-se/test/supervisor-*.test.mjs
```

- [ ] **Step 5: Commit**

```bash
git commit -m "fix(supervisor): restart recovery covers paused sprints with no ledger row"
```

---

### Task 1: Pause wrapper -- park, release ledger, persist snapshot

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs` (`pauseSprint`)
- Modify: `packages/apra-fleet-se/src/supervisor/watchdog.mjs` (probe paused queue records not on the ledger)
- Create: `packages/apra-fleet-se/test/supervisor-pause-queue.test.mjs`
- Consumes: existing `proxy` child's `/pause` (see `proxy.mjs` handlePause), `ledger.release`, `queue.put`, child `/state`

**Interfaces:**
- Produces: `POST /api/sprints/:id/pause` -> `{ sprintId, state: 'paused', membersReleased: string[] }`
- Snapshot on queue record:

```
snapshot: {
  headSha, baseSha, branch, base,
  budgetConsumedUsd, budgetCapUsd,
  cycle, pauseStatus,
}
childPid, childPort copied from ledger/spawner before release
```

- [ ] **Step 1: Write failing tests**

```js
test('pause of a running sprint releases ledger members and stores a paused queue record', async () => {
  // launch running with alice claimed
  // proxyPause resolves; proxyState returns { pause: { status: 'paused' }, cost: { usd: 1.25 }, cycle: 3 }
  // fake gitRevParse returns 'abc123'
  const out = await controller.pauseSprint('s1');
  assert.equal(out.state, 'paused');
  assert.equal(ledger.get('s1'), undefined);
  const q = queue.get('s1');
  assert.equal(q.state, 'paused');
  assert.equal(q.snapshot.headSha, 'abc123');
  assert.equal(q.snapshot.budgetConsumedUsd, 1.25);
  assert.equal(q.childPid, 5000);
});

test('GET /api/members shows alice free after pause', async () => {
  const { members } = await controller.members();
  const alice = members.find((m) => m.name === 'alice');
  assert.equal(alice.reserved, false);
});

test('pause of a non-running sprint is 409', async () => {
  await queue.put({ sprintId: 'p1', state: 'pending', /* ... */ });
  await assert.rejects(() => controller.pauseSprint('p1'), (e) => e.status === 409);
});
```

Inject `proxyPause`, `proxyState`, `gitRevParse` as controller deps (default implementations can shell `git rev-parse` in cwd -- tests inject a fake).

- [ ] **Step 2: Run -- expect FAIL**

- [ ] **Step 3: Implement `pauseSprint`**

Order: confirm ledger entry -> resolve port -> `proxyPause` -> poll `proxyState` until `pause.status === 'paused'` or 60s -> read snapshot -> `ledger.release` -> `queue.put` paused -> `scheduler.kick()`. If pause never engages, 409 and leave ledger claimed.

Watchdog: in the classify loop, also iterate `queue.list({ state: 'paused' })`. If parked pid is dead, set `childPid: null` on the record (resume will respawn). Do not auto-release anything already released. Do not classify parked-paused as crashed in a way that deletes the queue record.

- [ ] **Step 4: Run pause-queue + watchdog tests. Expected PASS.**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): pause running sprints back into the queue"
```

---

### Task 2: Resume into WAITING; same-member vs different-member

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs` (`resumeSprint`)
- Modify: `packages/apra-fleet-se/src/supervisor/scheduler.mjs` (`startQueued` must understand paused-parked vs respawn)
- Modify: `packages/apra-fleet-se/test/supervisor-pause-queue.test.mjs`

**Interfaces:**
- `POST /api/sprints/:id/resume` body `{ members?: string[], reason? }`
- Always leaves the record WAITING (or starts immediately if free -- allowed, still goes through `kick`/`startQueued`). Never requires instant capacity from the HTTP handler beyond calling kick.
- If `members` omitted, use `record.members` or `record.request.members`.
- If members equal original and `childPid` alive: `startQueued` claims ledger then `proxyResume(port)`.
- If members differ or pid dead: cooperative-stop parked child if any, clear pid, spawn new child with stored request + same sprintId/run-id + remaining budget. Before spawn, `gitRevParse(branch)` must equal `snapshot.headSha` OR be a fast-forward of it; else 409.

Budget: pass `budget: snapshot.budgetCapUsd` and a new spawn flag only if one exists; if the child has no "already consumed" argv, set an injected `budgetConsumedUsd` on the queue snapshot that `startQueued` forwards once a runner flag exists. If no runner flag exists yet, persist consumed on the queue and document that remaining-cap forwarding is `budgetCapUsd - budgetConsumedUsd` as the child's `--budget` (so the cap is the remainder, equivalent to not resetting spend). Prefer remainder-as-`--budget` to avoid a runner change in this PR.

- [ ] **Step 1: Write failing tests**

```js
test('resume of paused re-enters waiting without claiming members', async () => {
  // paused record, alice still claimed by someone else
  const out = await controller.resumeSprint('s1', {});
  assert.equal(out.state, 'waiting');
  assert.equal(ledger.get('s1'), undefined);
});

test('kick after resume with free members proxies /resume on the parked child', async () => {
  // paused, pid alive, alice free
  await controller.resumeSprint('s1', {});
  await scheduler.kick();
  assert.equal(resumeCalls[0].port, 9100);
  assert.ok(ledger.get('s1'));
  assert.equal(queue.get('s1'), undefined);
});

test('resume onto a different member stops the parked child and respawns', async () => {
  await controller.resumeSprint('s1', { members: ['bob'] });
  await scheduler.kick();
  assert.equal(stopCalls.length, 1);
  assert.equal(capturedSpawn.length, 1);
  assert.match(capturedSpawn[0].args.join(' '), /bob/);
});

test('resume with diverged branch SHA returns 409 naming both SHAs', async () => {
  record.snapshot.headSha = 'aaa';
  gitRevParse = async () => 'bbb';
  await assert.rejects(() => controller.resumeSprint('s1', {}), (e) => e.status === 409 && /aaa/.test(e.message) && /bbb/.test(e.message));
});

test('remaining budget forwarded as cap minus consumed', async () => {
  snapshot.budgetCapUsd = 10;
  snapshot.budgetConsumedUsd = 2.5;
  await controller.resumeSprint('s1', { members: ['bob'] });
  await scheduler.kick();
  // spawn argv contains --budget 7.5 (or 7.50)
});
```

- [ ] **Step 2: Run -- expect FAIL**

- [ ] **Step 3: Implement resume + startQueued branches**

- [ ] **Step 4: Run pause-queue + scheduler tests. Expected PASS.**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): resume paused sprints through the wait queue"
```

---

### Task 3: Clone, relaunch, reject RUNNING edits (API)

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs`
- Modify: `packages/apra-fleet-se/test/supervisor-api-queue.test.mjs`
- Modify: `packages/apra-fleet-se/src/supervisor/history.mjs` if a list-by-time helper is needed

**Scope note (round 2): fix the existing Restart, do not build a parallel path.** A Restart button already exists on running Sprint Stack rows, and `ledger.mjs` already persists `branch`/`base`/`goal` explicitly to serve it. Today's client script force-releases the reservation **first**, then reads those fields back out of the force-release audit payload and `window.prompt()`s the operator for whatever came back null -- *after* the reservation is already gone, so cancelling a prompt leaves a released sprint and a manual relaunch. Adding a second, safer relaunch feature beside it would leave two diverging implementations of one idea.

So this task builds **one** prefill-then-submit path and re-points the existing button at it:

1. **Build the request first, act second.** Resolve the stored request (queue record, else history/ledger metadata), validate it, and only then release/stop and submit. Nothing destructive happens before there is a submittable request.
2. **Prompts become a fallback, not the normal path.** PR A stores the *full* original request on the queue record, not just three fields, so a queued/paused restart needs no prompting at all. Only a legacy ledger entry predating those fields can still prompt.
3. **Relaunch-from-history is the same flow**, differing only in where the request is read from.
4. **Clone-and-amend** is the same flow plus an editable overlay and a new sprint id.

**Interfaces:**
- `resolveSprintRequest(sprintId)` -- single internal helper returning `{ request, source: 'queue'|'history'|'ledger', missingFields: string[] }`. Every entry point below uses it; no entry point reads ledger/history fields directly.
- `POST /api/sprints/:id/clone` -- PENDING/WAITING/PAUSED only. Merge overlay onto the resolved request. New sprintId. Original unchanged. 201.
- `POST /api/sprints/:id/relaunch` -- terminal only (history has the id, not ledger, not queue). `queue: true` default. Same `enqueueOrStart`. 201 or 400/409 from validation.
- PATCH RUNNING still 409 (already PR A). Clone RUNNING -> 409.

- [ ] **Step 1: Write failing tests** for clone overlay independence, relaunch prefills issue/branch/base/goal/roleMap/budget, relaunch of invalid issue 400, clone of running 409, plus:

```js
test('resolveSprintRequest prefers the queue record and reports no missing fields', async () => {});

test('resolveSprintRequest reports missingFields for a legacy ledger entry without branch/base', async () => {
  // this is the ONLY case the dashboard is allowed to prompt for
});

test('restart does not release the old reservation when the resolved request is invalid', async () => {
  // regression guard for today's release-then-prompt ordering:
  // invalid/unresolvable request -> 4xx AND ledger.get(id) still present
});
```

- [ ] **Step 2: Run -- expect FAIL**

- [ ] **Step 3: Implement.** Add `history.listTerminal({ limit, offset })` if missing: newest `finished` / `launch-failed` / `auto-released` grouped by sprintId.

- [ ] **Step 4: Run tests. Expected PASS.**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): clone queued sprints and relaunch from history"
```

---

### Task 4: GET /api/sprints/history + dashboard recent-5, clone, relaunch, no-edit-on-running

**Files:**
- Modify: `packages/apra-fleet-se/src/supervisor/api.mjs` (`listHistory`)
- Modify: `packages/apra-fleet-se/src/supervisor/dashboard.mjs`
- Modify: `packages/apra-fleet-se/test/supervisor-dashboard.test.mjs`
- Modify: `packages/apra-fleet-se/docs/supervisor-openapi.yaml`
- Modify: `packages/apra-fleet-se/docs/supervisor-api.md`
- Modify: `packages/apra-fleet-se/fleet-sprint/skills/fleet-supervisor/SKILL.md`

**Interfaces:**
- `GET /api/sprints/history?limit=5&offset=0` -> `{ sprints: [{ sprintId, event, at, issueRoots, members, branch, base, goal }], total }`
- Dashboard Sprints tab: after Queue, **Recent sprints** default 5, button Load more increases offset.
- Buttons: Relaunch (history), Clone (queue rows), no Clone/Edit on running rows.
- **Rework `SPRINT_RESTART_SCRIPT` rather than adding a sibling script.** New order: resolve the request via the endpoint, and only if it comes back complete do the release-then-submit. Prompts fire only for `missingFields` on a legacy entry, and a cancelled prompt now aborts **before** anything is released. The "Reservation released, but the original issue/members could not be recovered" dead end disappears for any sprint that has a queue record.
- Pause button on running rows should call `POST /api/sprints/:id/pause` (new API), not only `/sprints/:id/live/pause`, so ledger release happens. Keep live proxy for the child's own viewer.

- [ ] **Step 1: Write failing dashboard tests** for 5 rows, load-more marker, running row has no clone button, paused row has clone + resume, history row has relaunch. API test for default limit 5. Plus: the restart script resolves before releasing, and a sprint with a complete stored request renders no prompt path.

- [ ] **Step 2: Run -- expect FAIL**

- [ ] **Step 3: Implement list endpoint + HTML + rework the Restart script onto `resolveSprintRequest` + switch Pause script to the queue-facing route.** Update OpenAPI/SKILL with pause/resume/clone/relaunch/history examples.

- [ ] **Step 4: Run dashboard + api-queue + pause-queue tests. Expected PASS.**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): browse sprint history and wrap pause in the queue API"
```

---

## Self-review vs spec points 4-5

| Spec | Task |
|---|---|
| Paused sprint survives a supervisor restart (round 2) | 0, 1b |
| Pause clears BOTH reservation sources (round 2) | 1 (ledger) + existing runner behavior (fleet-side) |
| Existing Restart stops being destructive-before-validating (round 2) | 3, 4 |
| Pause releases members; GET /api/members free | 1 |
| Paused persisted with cycle/HEAD/budget/beads | 1 |
| Resume re-enters WAITING | 2 |
| Different member resync/respawn | 2 |
| Diverged branch fails loudly | 2 |
| Budget not reset | 2 remainder-as-cap |
| Pause safe (no claim without spawn; release after paused) | 1 |
| History 5 + load more | 4 |
| Relaunch prefills | 3, 4 |
| Clone-and-amend queue/paused | 3, 4 |
| RUNNING not editable; API 409 | 3, 4 |
