# Supervisor persistent sprint queue + autonomous backlog grooming -- design

Date: 2026-09-08
Status: draft (engineering detail). Covers GitHub issues #410 (mechanism) and #22 (decision-making).
Authors: planning pass for assignee Siddharth Deshpande (assigned by Akhil Kumar).

**Akhil review lives in `docs/proposals/`**, not here:

- Cover / how the system works: `docs/proposals/README.md`
- Approval plan for #410: `docs/proposals/issue-410-persistent-sprint-queue.md`
- Approval plan for #22: `docs/proposals/issue-22-autonomous-backlog-grooming.md`

This file is the longer internal design behind those two plans.

## 1. Why these two issues are one program of work

Issue #410 is the supervisor's missing execution primitive: a sprint that can
**exist before it runs**. Issue #22 is the missing decision-maker: the
backlog-groomer already knows *what* should be sprinted next, but today its
output is a report a human has to act on.

```
#22  groom backlog -> propose sprint sets -> POST queued sprint requests
#410 persist / order / wait / pause / resume / history-relaunch
     supervisor starts the next eligible sprint when a member frees
```

#410 is a hard dependency. #22's autonomous-queueing scope cannot be built
until a sprint can be created with zero members, survive a busy member as
WAITING rather than 409, and be reordered by a human. Do not start #22
implementation until #410 points 1-3 are merged (pause/resume and dashboard
history can land in parallel with the first #22 increment).

## 2. Current state (what already shipped)

### 2.1 Launch is launch-now-or-fail

`POST /api/sprints` in `packages/apra-fleet-se/src/supervisor/api.mjs`:

- `validateLaunchRequest` rejects `members.length === 0` with 400.
- `defaultMemberOverlapGuard` rejects any busy member with 409
  (`formatMemberConflict`), before `ledger.claim()`.
- A successful launch always `spawner.spawnSprint()` then `ledger.claim()`.
- `GET /api/sprints` lists **ledger-held** (live) sprints only.
- There is no persisted record of a sprint that has been requested but not
  started.

Existing tests that encode this contract (must be updated, not deleted):

- `packages/apra-fleet-se/test/supervisor-api.test.mjs` (empty members 400,
  overlap 409, relaunch gate 409)
- `packages/apra-fleet-se/test/reservation-interop-e2e.test.mjs`
- `packages/apra-fleet-se/docs/supervisor-openapi.yaml`
- `packages/apra-fleet-se/docs/supervisor-api.md`
- `packages/apra-fleet-se/fleet-sprint/skills/fleet-supervisor/SKILL.md`

### 2.2 Persistence is split on purpose -- keep it that way

- **Ledger** (`ledger.mjs` / `reservations.json`): who holds a reservation
  RIGHT NOW. `release()` deletes the entry. Do not store PENDING/WAITING
  here -- that would make "reserved" mean "queued", which is the speculative
  pre-reservation #410 explicitly forbids.
- **History** (`history.mjs` / `sprint-history.json`): append-only terminal
  events (finished, crashed, auto-released, aborted-by-restart, ...). Do not
  store live queue state here -- it is not an audit log of intent.
- **Child run-state** (`old_runs/<sprintId>.json` and the live run-state
  path): engine-owned cycle/progress/pause snapshot. Pause/resume already
  writes here via the workflow engine.

#410 needs a **third store**: the queue of sprints that exist but do not
hold a reservation.

### 2.3 Pause/resume primitive already exists (point 4 foundation)

Shipped under the workflow-engine `apra-fleet-p2to` epic (Akhil's comment on
#410, 2026-09):

- Engine: `requestPause` / `requestResume`, activity-entry gate, pause guard.
- Runner: on `'paused'`, release fleet `member_reservation`; on resume,
  owner-checked re-reserve + git/dolt resync (`reReserveOnResume` in
  `packages/apra-fleet-se/bin/cli.mjs`).
- Viewer: `POST /pause` and `POST /resume` on the child.
- Supervisor today: dashboard Pause/Resume **proxies**
  `POST /sprints/:id/live/pause|resume`. The sprint stays on the ledger.
  Watchdog classifies a live-pid paused child as `paused` and does **not**
  auto-release the ledger reservation.

Point 4 of #410 is the **queue-facing wrapper**, not a rebuild: pause must
free the supervisor ledger so another queued sprint can start, persist enough
to resume (including on a different member), and re-enter WAITING rather
than demanding instant capacity.

### 2.4 Groomer already does the analysis half

`packages/apra-fleet-se/apra-pm/agents/backlog-groomer.md` plus
`agents/schemas/backlog-groomer-{input,output}.json`:

- Ready/urgent ranking, cohesive sprint sets, dedupe, quality bar,
  priority x age, structural deadlocks, landed-vs-closed.
- Full beads mutation authority, evidence-gated, never touches code.
- Output is JSON advice (`sprintSets`, `needsGrooming`, ...). No supervisor
  API client. No fleet capacity input. No cadence.

Dashboard today: Sprint Stack (running only) + Backlog tab + Launch form.
History is a per-sprint `GET /sprints/:id/history` page, not a browsable
list of the last 5 terminal sprints. Restart exists on **running** rows
(force-release then re-POST), not on past sprints.

## 3. Goals

### 3.1 Issue #410 (mechanism)

1. `POST /api/sprints` with omitted/`[]` members creates a persisted PENDING
   sprint (201, `state: "pending"`). No reservation, no spawn.
2. Named members that are busy become WAITING (201) unless the caller opts
   into fail-fast. When the full member union is free, the supervisor starts
   the sprint automatically. No pre-reservation while waiting.
3. Human-controllable durable ordering (priority + pin-next + createdAt),
   visible on `GET /api/sprints` in consume order, auditable.
4. `POST /api/sprints/:id/pause` releases ledger members; resume re-enters
   WAITING and may land on a different member from persisted git HEAD.
5. Dashboard: queue visible; last 5 terminal sprints + load more; relaunch
   from history; clone-and-amend on PENDING/WAITING/PAUSED; RUNNING not
   editable (API-enforced).

### 3.2 Issue #22 (decision-making)

1. Groomer gains capacity awareness (members, running, already queued).
2. Groomer turns quality-bar-passing sprint sets into queued POSTs
   (idempotent; never duplicates a pending/waiting/running set).
3. Supervisor runs the groomer on a cadence plus event triggers.
4. Delivery loop: on sprint terminal, next groom pass verifies
   landed-vs-closed and tracks velocity.

### 3.3 Non-goals

- Cross-machine scheduling beyond "start when the entire requested union is
  free" (#410 out of scope).
- Replacing beads, or teaching the groomer to touch code.
- Speculative member pre-reservation.
- In-place mutation of a RUNNING sprint's parameters.
- Rebuilding the engine pause primitive.

## 4. Product decisions (issue #22 open questions)

These are the recommended defaults so implementation is not blocked. They
are marked LOCKED-PROPOSAL: change them in review before coding #22, not
silently during implementation.

### D1. Autonomy: queue directly as PENDING, not a second "proposed" state

LOCKED-PROPOSAL: the groomer POSTs real PENDING sprints. There is no
separate proposed/draft state.

Rationale: #410 already gives the operator cancel, reorder, pin-next, and
pause. A second approval state would recreate the human handoff this issue
exists to remove. PENDING with zero members already cannot run, so a human
who wants a gate-before-run can simply not enable auto-assign (D2).

### D2. Member assignment: operator-opt-in auto-assign

LOCKED-PROPOSAL: groomer always queues PENDING (`members: []`). A supervisor
config flag `queue.autoAssignMembers` (default **false**) controls whether
the scheduler may bind a free eligible member onto the next PENDING item
when capacity appears.

- Default false: grooming becomes a durable queue the operator drains by
  assigning members (or pinning) in the dashboard. Safer for cost.
- When true: on member-free, scheduler binds one free member (or a
  configured default pool) onto the highest-priority PENDING item whose
  issue/branch/base still validate, then starts it if the union is free.

Groomer never picks machines. Capacity awareness only limits *how many*
sets it enqueues, not *who* runs them.

### D3. Cadence and triggers

LOCKED-PROPOSAL:

- Interval: `groomer.intervalMs` default 3600000 (1 hour). 0 disables the
  timer.
- Off-cadence triggers (all no-op if a groomer run is already in flight):
  - a sprint reaches a terminal watchdog status (finished/crashed/stopped)
  - a WAITING/PENDING count drops to 0 while at least one member is free
  - operator `POST /api/groom` (manual)
- P0 detection is the groomer's job on each pass (it already ranks by
  priority). Do not add a separate beads-file-watcher in v1.

### D4. Budget guardrails

LOCKED-PROPOSAL: supervisor-enforced, not LLM-enforced.

- `groomer.maxQueued` default 3: refuse to enqueue above this many
  PENDING+WAITING items (running does not count).
- `groomer.maxDailyBudgetUsd` default null (disabled). When set, sum
  `budget` of running + queued created today; skip enqueue if the new
  item's budget would exceed the cap.
- Groomer still must not queue sets that failed its own quality bar
  (missing repro / AC / unverified closure) -- that is a producer-side
  filter, independent of the numeric caps.

### D5. Fail-fast vs queue on busy members (issue #410 point 2)

LOCKED-PROPOSAL: **opt-in queue, default fail-fast**.

- `members: []` / omitted -> PENDING (new; no existing caller sends this).
- Named members + overlap + `queue: true` -> WAITING 201.
- Named members + overlap + `queue` omitted/false -> today's 409.

Rationale: the issue's "Done when" prefers default WAITING, but also says
existing launch-now callers must not be silently changed. Dashboard Restart,
the Launch form, and every current 409 test are launch-now callers. Opt-in
`queue: true` preserves them. The groomer always sends `queue: true` (or
empty members). Document the flag in OpenAPI and the supervisor skill.

If review prefers default-queue, the same code path works; only the default
and the existing 409 tests change.

## 5. Architecture

### 5.1 New queue store (do not overload ledger or history)

New module: `packages/apra-fleet-se/src/supervisor/queue.mjs`

On-disk: `<FLEET_SE_DATA_DIR>/sprint-queue.json`

Same durability discipline as `ledger.mjs`: in-memory commit only after
atomic temp-file + `renameWithRetry`. Schema versioned. Reload on
`start()` so PENDING/WAITING/PAUSED survive supervisor restart.

A queue record holds the **full original request** (issue, branch, base,
goal, maxCycles, roleMap, budget, requirementsFile, allowMissingMembers,
overrideRelaunchGate, members, plus queue metadata).

```
QueueRecord
  sprintId          string     minted at enqueue, same <issue>-<uuid> shape
  state             pending | waiting | paused | blocked
  request           object     verbatim launch body (normalized)
  members           string[]   named members (empty for pending)
  issueRoots        string[]
  priority          number     default 0; higher runs first
  pinNext           boolean    default false; beats priority
  createdAt         ISO-8601
  updatedAt         ISO-8601
  skippedCount      number     times pick-next walked past this item
  blockedReason     string|null
  childPid          number|null  paused parked process, if any
  childPort         number|null
  snapshot          object|null  pause snapshot (see 8)
  idempotencyKey    string|null  groomer-supplied; see 11.3
  origin            "api" | "groomer" | "relaunch" | "clone"
  audit             { lastActor, lastReason } | null
```

States that hold a ledger reservation (RUNNING) are **not** queue records.
When pick-next starts a sprint it: validates, spawns, `ledger.claim()`,
deletes the queue record (or marks it consumed -- delete is simpler; the
launch metadata already lands on the ledger + later history).

### 5.2 Scheduler

New module: `packages/apra-fleet-se/src/supervisor/scheduler.mjs`

Pure-ish pick function + a `kick()` that the supervisor calls after every
capacity-changing event:

- watchdog `releaseTerminalReservation` succeeded
- `stopSprint` / force-release completed
- pause wrapper released the ledger
- `PATCH` assigned members onto a PENDING item
- member list changed (deleted/unregistered)
- queue reorder

`kick()` is serialized (one in-flight pick-next at a time). Re-entrant calls
set a dirty flag and run once more when the current kick finishes.

### 5.3 Controller split

Keep `createSprintController` as the HTTP facade. Internally split launch
into:

1. `validateLaunchRequest` (issue/branch/base always; members may be empty)
2. relaunch gate (still at enqueue time AND again at execution time)
3. `enqueueOrStart`:
   - no members -> write PENDING, return 201
   - members named, overlap, `queue !== true` -> 409 (today)
   - members named, overlap, `queue === true` -> write WAITING, return 201
   - members named, free -> existing spawn+claim path, return 201 with
     `state: "running"`

Do not spawn on the PENDING/WAITING paths.

### 5.4 Wiring

`packages/apra-fleet-se/bin/serve.mjs` already constructs ledger, history,
spawner, watchdog, dashboard, sprint controller. Add:

- `createQueue({ filePath })` next to `createLedger`
- `createScheduler({ queue, ledger, controller, listMembers, history })`
- pass `queue` + `scheduler.kick` into the controller and watchdog
- register new routes (pause/resume/reorder/clone/groom)

## 6. State machine

```
                  POST members=[]
                        |
                        v
                     PENDING  ---------------- PATCH members
                        |                         |
                        |                         v
                        |                      WAITING
                        |                         |
                        |     entire union free   |
                        |     (pick-next)         |
                        +------------------------>+
                                                  |
                                                  v
                                               RUNNING  (ledger claim held)
                                                  |
                    +------------+----------------+----------------+
                    |            |                |                |
                    v            v                v                v
               COMPLETED      FAILED          STOPPED           PAUSED
               (history)      (history)       (history)     (queue record,
                                                             ledger released)
                                                                   |
                                                            POST /resume
                                                                   v
                                                                WAITING
```

Additional: WAITING whose named member is deleted/unregistered -> BLOCKED
(visible, not silent forever). Operator can PATCH members or DELETE.

Cancel: `DELETE /api/sprints/:id` for PENDING/WAITING/PAUSED/BLOCKED
(never started, or paused). RUNNING still uses cooperative stop /
force-release.

## 7. HTTP contract

All new/changed routes live in `api.mjs` + `registerSprintRoutes`, documented
in `supervisor-openapi.yaml` and `supervisor-api.md`. ASCII-only copy.

### 7.1 POST /api/sprints (changed)

Request additions:

- `members` optional. Empty/omitted -> PENDING.
- `queue` boolean, default false (see D5).
- `priority` number, default 0.
- `idempotencyKey` optional string.

Response 201 always includes `sprintId` and `state`:
`pending | waiting | running`.

PENDING/WAITING responses have `pid: null`, `port: null`. RUNNING keeps
today's pid/port.

Validation that still runs at creation (even for PENDING):

- issue ids, branch, base (same helpers as today)
- relaunch gate (deterministic prior terminal -> 409 unless
  `overrideRelaunchGate: true`)

Issue-scope overlap: PENDING/WAITING do **not** claim scope. Scope overlap
is re-checked at execution time. Two queued sprints may name the same root;
pick-next starts one, the other stays WAITING (or 409s at start and moves
to BLOCKED with the conflict message). This avoids speculative scope
reservation.

### 7.2 GET /api/sprints (changed)

Returns consume-order:

1. RUNNING (ledger), then
2. queue records sorted by the pick-next comparator (see 7.5)

Each item includes `state`, `priority`, `pinNext`, `createdAt`, and the
stored request. Query `?state=pending,waiting` filters. Query
`?history=1&limit=5&offset=0` pages terminal history for the dashboard
(point 5) without a second resource if we want one list endpoint; otherwise
add `GET /api/sprints/history?limit=5&offset=0`. Prefer a dedicated history
list to keep the live/queue payload small:

`GET /api/sprints/history?limit=5&offset=0` -> `{ sprints, total }`
newest-terminal-first from `history.mjs` + `old_runs/` metadata.

### 7.3 GET /api/sprints/:id (changed)

If queue has the id, return `{ sprintId, live: false, state, request, ... }`.
Else today's live-or-history behavior. 404 only if in none of
ledger / queue / history.

### 7.4 Ordering

`PATCH /api/sprints/:id`

Allowed on PENDING/WAITING/PAUSED/BLOCKED only. RUNNING -> 409
`sprint is running; pause first or clone`.

Body (all optional): `priority`, `pinNext`, `members`, `goal`, `budget`,
`roleMap`, `maxCycles`, `reason` (audit).

Assigning members on PENDING transitions it to WAITING (or RUNNING if
immediately free). Clearing members on WAITING returns it to PENDING.

`POST /api/sprints/:id/reorder` body `{ action: "front"|"back", reason }`
is sugar: front sets `pinNext: true` (and clears pinNext on any other
item -- at most one pin); back sets `priority` to min(existing)-1 and
clears pinNext.

Every successful PATCH/reorder records a history event
`queue-reordered` with actor (request header `X-Fleet-Actor` or `"api"`)
and reason.

### 7.5 Pick-next comparator (deterministic)

```
pinNext desc (true first; at most one record should be true)
priority desc
createdAt asc
sprintId asc   (tie-break, never random)
```

Document this in SKILL.md and supervisor-api.md. The dashboard list uses
the same sort function (`export function compareQueueOrder(a, b)` from
`queue.mjs`) so API and UI cannot drift.

Anti-starvation (multi-member): pick-next walks the sorted list and starts
the **first** record whose entire member union is free **right now**. It
does not HOL-block behind a WAITING sprint that cannot start. Each skip
increments `skippedCount`. No automatic priority bump (operator pins).
Log a warning when `skippedCount >= 3` and at least one named member was
free on that kick (starvation signal).

No partial starts. Two half-satisfied multi-member sprints never take
one member each.

### 7.6 Pause / resume / cancel / clone / relaunch

`POST /api/sprints/:id/pause`
- RUNNING only. Proxies child `/pause`, waits until child `state.pause.status
  === 'paused'` (timeout 60s -> 409 naming the stall).
- Then `ledger.release()`, writes queue record `state: paused` with snapshot
  (section 8). Members immediately appear free on `GET /api/members`.
- Kicks scheduler.

`POST /api/sprints/:id/resume`
- PAUSED only. Optional body `{ members, reason }`.
- Sets state WAITING (members from body or original request). Does not
  spawn immediately. Kicks scheduler.
- If `members` differ from the parked child's original union, the parked
  child is stopped (cooperative stop) and discarded; pick-next will spawn a
  **new** child from snapshot HEAD (section 8). Same-member resume proxies
  `/resume` on the parked process once members are claimed.

`DELETE /api/sprints/:id`
- PENDING/WAITING/PAUSED/BLOCKED: drop queue record; if a parked child
  exists, cooperative stop it. 200.
- RUNNING: 409, use stop.

`POST /api/sprints/:id/clone`
- PENDING/WAITING/PAUSED only. Body is a partial request overlay.
- Creates a **new** sprintId from the stored request merged with the overlay.
  Original untouched. 201.

`POST /api/sprints/:id/relaunch`
- Terminal history only. Prefills from the stored launch metadata on the
  history/ledger echo / `old_runs` + history events. Goes through the same
  `enqueueOrStart` as POST /api/sprints (so a now-invalid request still
  400/409s loudly). Default `queue: true` so relaunch of a busy fleet
  waits rather than 409. 201.

PATCH/clone of RUNNING: 409 API-side even if a client bypasses the UI.

## 8. Pause snapshot (point 4)

Enough to resume without redoing completed work, including on another
member:

From the child's `/state` + git at pause time, persist on the queue record:

- `cycle` / phase / per-role progress (already in run-state)
- `branch` and `headSha` (`git rev-parse <branch>`)
- `base` and `baseSha`
- `budgetConsumedUsd` (from run-state cost; must not reset on resume)
- `beadHeads` optional (run-state already has bead progress)
- `originalMembers`, `roleMap`
- `childPid` / `childPort` if the process is kept parked

Resume-on-different-member:

1. Stop the parked process if members changed or pid is dead.
2. New spawn with same `--run-id` / same branch, `overrideRelaunchGate` as
   stored. Runner already resyncs git + `bd dolt pull` on resume
   (`reReserveOnResume`).
3. If `git rev-parse <branch>` != snapshot `headSha` and the divergence is
   not fast-forward from snapshot, fail resume loudly (409) with both SHAs
   -- never silently re-run or reset the branch.
4. Budget cap forwarded is the **original cap**; consumed-so-far is the
   snapshot value so the remaining ceiling is `cap - consumed`.

Watchdog change: paused sprints that have left the ledger must still be
probed via `queue.list({ state: 'paused' })` so a dead parked pid is
detected (move to BLOCKED or drop pid and keep PAUSED as
resume-will-respawn). Do not treat parked-paused as stall-suspect.

## 9. Dashboard

`packages/apra-fleet-se/src/supervisor/dashboard.mjs`

Sprints tab becomes three stacked sections, still inside `tab-sprints`:

1. **Sprint Stack** -- RUNNING (today). Edit/amend controls disabled
   (no clone button on running rows; Restart remains a stop+relaunch of a
   *new* id, which is not in-place edit).
2. **Queue** -- PENDING/WAITING/PAUSED/BLOCKED in `compareQueueOrder`.
   Controls: pin next, move front/back, edit (opens amend overlay), clone,
   cancel, assign members (PENDING). Resume on PAUSED.
3. **Recent sprints** -- 5 most recent terminal, "Load more" fetches
   `GET /api/sprints/history?limit=5&offset=N`. Each row: relaunch.

Header stats banner: `N running | M waiting | P pending`.

Live-refresh loop already polls; extend the payload so queue + recent
history refresh without a full page load (same `supervisor-dashboard-live-refresh`
test pattern).

Launch form: members field becomes optional; empty submits PENDING. A
"queue if busy" checkbox maps to `queue: true`.

## 10. Issue #22 -- groomer pipeline

### 10.1 Capacity-aware producer

Extend `backlog-groomer-input.json`:

- `capacity` object (required for autonomous mode, optional for human
  "define my sprint"): `{ membersFree, membersBusy, queuedCount,
  runningCount, maxQueued }`.
- `queueUrl` optional (supervisor base URL). When present and `dry-run` is
  false, the groomer POSTs queue requests as its last step.

Extend `backlog-groomer-output.json`:

- `queueRequests`: array of `{ issue, branch, base, goal, budget,
  priority, idempotencyKey, beadIds, skippedReason? }`.
- `delivery`: `{ verifiedClosed, stillOpen, atRisk, velocityNote }`.

Groomer quality bar (never queue if):

- bead is in `needsGrooming` (thin P0/P1, no repro, no AC)
- bead is in `needsVerification` (landed-vs-closed unverified)
- structural deadlock unresolved
- set would exceed `maxQueued - queuedCount`

Branch naming: resolve in JavaScript/agent text to an explicit
`feat/<short-topic>` string -- never shell-expand `$VAR` or `~` (member
shells may be PowerShell).

### 10.2 Idempotency

`idempotencyKey = sha256(sorted(issueRoots).join(',') + '|' + branch)`.

`POST /api/sprints` with a key that already exists on a
PENDING/WAITING/RUNNING/PAUSED record returns 200 with the existing
`sprintId` (not 201, not a duplicate). Terminal records do not block a
new key -- a finished sprint may be re-queued after the next groom pass
if the beads are still open (relaunch gate still applies).

### 10.3 Supervisor groomer runner

New module: `packages/apra-fleet-se/src/supervisor/groomer.mjs`

- `POST /api/groom` starts one groomer dispatch if none is running.
- Timer in `serve.mjs` using D3.
- Dispatch: spawn the installed `backlog-groomer` agent the same way
  operators already invoke it (Claude agent file under
  `packages/apra-fleet-se/apra-pm/agents/`). Exact spawn adapter is an
  implementation task; the test seam is `runGroomer(input) -> outputJson`.
- After output: for each `queueRequests` entry without `skippedReason`,
  POST `/api/sprints` with `members: []`, `queue: true`, the key, origin
  `groomer`. Caps in D4 are enforced in the controller, not only in the
  agent.
- Persist last groom result under `<dataDir>/groomer-last.json` for the
  dashboard (timestamp, notes, queued ids, skipped reasons).

Do not cite bead ids (`apra-fleet-XXXX`) in any LLM-facing prompt text
the runner adds; the agent already sees beads via `bd` itself.

### 10.4 Delivery tracking

On each groom pass (including the post-terminal trigger):

- Re-run landed-vs-closed for recently terminal issue roots.
- `velocityNote` is a short string (e.g. "3 finished / 1 crashed in last
  7d") computed from `history.mjs`, not a new metrics product.

## 11. Testing strategy

Follow existing supervisor test style (`node:test` + injected
ledger/spawner/history, temp dirs). New files:

- `packages/apra-fleet-se/test/supervisor-queue.test.mjs` -- store
  durability, sort, restart reload
- `packages/apra-fleet-se/test/supervisor-scheduler.test.mjs` -- pick-next,
  skip busy, multi-member all-or-nothing, member-deleted -> BLOCKED,
  no pre-reservation
- `packages/apra-fleet-se/test/supervisor-api-queue.test.mjs` -- PENDING
  201, WAITING vs 409/`queue` flag, PATCH running 409, clone, relaunch,
  idempotencyKey
- `packages/apra-fleet-se/test/supervisor-pause-queue.test.mjs` -- pause
  releases ledger; members free; resume WAITING; different-member respawn
  seam; budget not reset
- Dashboard unit tests in `supervisor-dashboard.test.mjs` for queue
  section, recent-5, disabled edit on running
- Groomer schema tests already in
  `packages/apra-fleet-se/apra-pm/test/agent-schema-validation.test.mjs`
  -- extend fixtures
- `packages/apra-fleet-se/test/supervisor-groomer.test.mjs` -- caps,
  idempotency, skip quality-bar sets (fake `runGroomer`)

Keep existing 409 overlap tests green under D5 (default fail-fast).

## 12. Implementation sequence

Ship as two PRs (or two stacked PRs), not one megamerge.

**PR A -- #410 points 1-3 (queue exists)**
Queue store, PENDING, WAITING + opt-in flag, scheduler kick, ordering
API, GET list in consume order, OpenAPI + SKILL.md + supervisor-api.md.
Dashboard Queue section (without pause wrapper / history browser).

**PR B -- #410 points 4-5**
Pause/resume wrapper, watchdog parked-pid probe, dashboard recent-5 +
relaunch + clone-and-amend, RUNNING edit rejected.

**PR C -- #22**
Groomer schema/skill, idempotency, caps, `POST /api/groom` + timer,
delivery notes, dashboard last-groom card.

Do not start PR C until PR A is merged (or stacked on A).

## 13. Risks

- **Watchdog vs pause-release:** today paused => stay on ledger. Releasing
  the ledger while a child is parked is the behavior change that makes
  the queue useful. Missed probe of the parked pid is the failure mode;
  scheduler + watchdog must list paused queue records.
- **Scope overlap at start, not enqueue:** two queued sprints can name the
  same root; the second BLOCKED at start. Document this; do not silently
  drop.
- **Default queue vs fail-fast:** D5 chooses fail-fast. If product wants
  default WAITING, it is a one-line default plus test updates.
- **Groomer spawn in a headless supervisor:** the runner seam must be
  fakeable; do not block PR A on LLM dispatch.
- **apra-fleet-client:** supervisor HTTP is not an MCP tool. No client
  package change unless a later change adds fleet MCP tools for queueing.
  If we add MCP wrappers, update `packages/apra-fleet-client` in the same
  PR (repo rule).

## 14. Files (expected blast radius)

Create:

- `packages/apra-fleet-se/src/supervisor/queue.mjs`
- `packages/apra-fleet-se/src/supervisor/scheduler.mjs`
- `packages/apra-fleet-se/src/supervisor/groomer.mjs` (PR C)
- tests listed in section 11

Modify:

- `packages/apra-fleet-se/src/supervisor/api.mjs`
- `packages/apra-fleet-se/src/supervisor/dashboard.mjs`
- `packages/apra-fleet-se/src/supervisor/launch-form.mjs`
- `packages/apra-fleet-se/src/supervisor/watchdog.mjs`
- `packages/apra-fleet-se/src/supervisor/history.mjs` (new event names)
- `packages/apra-fleet-se/bin/serve.mjs`
- `packages/apra-fleet-se/docs/supervisor-api.md`
- `packages/apra-fleet-se/docs/supervisor-openapi.yaml`
- `packages/apra-fleet-se/fleet-sprint/skills/fleet-supervisor/SKILL.md`
- `packages/apra-fleet-se/apra-pm/agents/backlog-groomer.md`
- `packages/apra-fleet-se/apra-pm/agents/schemas/backlog-groomer-input.json`
- `packages/apra-fleet-se/apra-pm/agents/schemas/backlog-groomer-output.json`

High blast radius on `createSprintController.launch` and
`defaultMemberOverlapGuard` -- every launch-now test depends on them.
Scheduler `kick` from watchdog release is the other high-risk seam.
