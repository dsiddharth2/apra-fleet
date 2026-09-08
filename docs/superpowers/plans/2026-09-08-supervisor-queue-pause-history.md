# Sprint Queue Pause, Resume, History, Clone (#410 points 4-5) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wrap the existing engine pause/resume primitive so a running sprint can free its members into the queue, resume later (possibly on a different member) without resetting budget, and expose history browsing plus clone/relaunch on the dashboard with RUNNING edits rejected server-side.

**Architecture:** `POST /api/sprints/:id/pause` proxies the child's `/pause`, then `ledger.release()` and writes a PAUSED queue record with a snapshot (HEAD, cycle, budget consumed, parked pid/port). `POST /resume` moves it to WAITING and `scheduler.kick()`. Different members: stop the parked child and respawn from snapshot HEAD. History list endpoint pages terminal sprints; clone/relaunch go through the same enqueue validation as POST /api/sprints.

**Tech Stack:** Same supervisor ESM stack as PR A. Depends on PR A (`queue.mjs`, `scheduler.mjs`, WAITING/PENDING).

**Spec:** `docs/superpowers/specs/2026-09-08-supervisor-sprint-queue-and-autonomous-grooming-design.md` sections 7.6, 8, 9.

## Global Constraints

- ASCII only. No bead ids in LLM-facing skill/schema/runtime strings.
- Do not rebuild `requestPause` / `requestResume` in the workflow engine. Supervisor wraps them.
- Pause MUST release the supervisor ledger (today it does not). Fleet-side reservation release on `'paused'` already exists in the runner; do not duplicate it incorrectly.
- No speculative pre-reservation while PAUSED or WAITING.
- RUNNING in-place PATCH remains 409.
- Budget consumed carries across pause; remaining cap is `originalCap - snapshot.budgetConsumedUsd`.
- Diverged branch vs snapshot `headSha` fails resume with 409 naming both SHAs.
- Permission blocks are surfaced, not bypassed.

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

**Interfaces:**
- `POST /api/sprints/:id/clone` -- PENDING/WAITING/PAUSED only. Merge overlay onto `record.request`. New sprintId. Original unchanged. 201.
- `POST /api/sprints/:id/relaunch` -- terminal only (history has the id, not ledger, not queue). Rebuild body from history launch metadata (ledger used to persist branch/base/goal/members/issueRoots on claim; history events carry issueRoots/members; prefer the most complete of history event + old_runs). `queue: true` default. Same `enqueueOrStart`. 201 or 400/409 from validation.
- PATCH RUNNING still 409 (already PR A). Clone RUNNING -> 409.

- [ ] **Step 1: Write failing tests** for clone overlay independence, relaunch prefills issue/branch/base/goal/roleMap/budget, relaunch of invalid issue 400, clone of running 409.

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
- Buttons: Relaunch (history), Clone (queue rows), no Clone/Edit on running rows (Restart may remain as today's stop+new-launch).
- Pause button on running rows should call `POST /api/sprints/:id/pause` (new API), not only `/sprints/:id/live/pause`, so ledger release happens. Keep live proxy for the child's own viewer.

- [ ] **Step 1: Write failing dashboard tests** for 5 rows, load-more marker, running row has no clone button, paused row has clone + resume, history row has relaunch. API test for default limit 5.

- [ ] **Step 2: Run -- expect FAIL**

- [ ] **Step 3: Implement list endpoint + HTML + switch Pause script to the queue-facing route.** Update OpenAPI/SKILL with pause/resume/clone/relaunch/history examples.

- [ ] **Step 4: Run dashboard + api-queue + pause-queue tests. Expected PASS.**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(supervisor): browse sprint history and wrap pause in the queue API"
```

---

## Self-review vs spec points 4-5

| Spec | Task |
|---|---|
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
