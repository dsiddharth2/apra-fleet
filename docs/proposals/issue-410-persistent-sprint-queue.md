# Plan: persistent sprint queue (GitHub #410)

- Status: PROPOSED (awaiting Akhil approval)
- Proposed by: Siddharth Deshpande
- Date: 2026-09-08
- Issue: https://github.com/Apra-Labs/apra-fleet/issues/410
- Unblocks: https://github.com/Apra-Labs/apra-fleet/issues/22
- Cover / context: `docs/proposals/README.md`

This is the approval document for the **mechanism**. It is not an
implementation checklist. After this is AGREED, implementation follows
`docs/superpowers/plans/2026-09-08-supervisor-persistent-sprint-queue.md`
(slice A) and
`docs/superpowers/plans/2026-09-08-supervisor-queue-pause-history.md`
(slice B).

---

## Decision table (please mark)

| # | Decision | Proposal | Why |
|---|---|---|---|
| D1 | Where queued sprints live | New on-disk store `sprint-queue.json`, sibling of the reservation ledger. Ledger stays "reserved right now". History stays terminal events. | Putting PENDING on the ledger would make waiting look reserved (#410 forbids speculative pre-reservation). |
| D2 | Empty `members` | Valid. Creates PENDING. No spawn, no reservation. Issue/branch/base still validated at create time. | Groomer (#22) knows *what* to sprint, not *which machine*. |
| D3 | Busy named member | Default stays today's **409**. `queue: true` stores WAITING and returns 201. | Issue text wants WAITING-by-default *and* "do not silently change launch-now callers". Launch form + Restart are launch-now. Opt-in keeps them. |
| D4 | When a WAITING sprint starts | The moment a member actually frees, walk the queue in documented order, start the first sprint whose **entire** member set is free. Skip (do not block behind) one that cannot start. No partial starts. | Same all-or-nothing rule as today, just delayed. |
| D5 | Ordering | `priority` desc, then older first, plus a single "pin this next" that always wins. PATCH + reorder API. Audit who/why. Human override survives restart. | Issue point 3. |
| D6 | Pause (point 4) | Wrap existing engine pause. After the child is actually paused: release the **supervisor ledger** so another sprint can use the member. Resume re-enters WAITING (does not demand instant capacity). Different member: stop parked child, respawn from stored git HEAD. Diverged HEAD fails loudly. Budget remaining = original cap minus already spent. | Engine pause already exists; today the supervisor still holds the reservation, so pause does not help a queue. |
| D7 | Running sprints | Not editable in place. Clone/amend only for PENDING/WAITING/PAUSED. Relaunch from history creates a **new** sprint. API returns 409 if someone bypasses the UI. | Issue point 5. |
| D8 | Ship shape | Two PRs: **A** = D1-D5 (queue exists, dashboard shows it). **B** = D6-D7 (pause wrapper + history 5 / relaunch / clone). | #22 only needs A. B is large and independent enough to land second. |

If D3 should instead be "WAITING by default, `failIfBusy: true` to keep 409",
say so here. The code path is the same; only the default and existing 409
tests change.

---

## 1. What is happening today (grounded in code)

`POST /api/sprints` in `packages/apra-fleet-se/src/supervisor/api.mjs`:

1. Validate issue / branch / base / **non-empty members**.
2. Relaunch gate (do not blindly retry a deterministic failure).
3. Member overlap guard: any requested member already claimed -> **409**,
   request discarded.
4. Issue-scope overlap guard: same, for beads scope.
5. Mint a sprint id, **spawn a child process**, **claim the ledger**.

There is no persisted object for "this sprint should happen later".

`GET /api/sprints` lists ledger rows only (live reservations).

The dashboard Sprint Stack renders running watchdog states. Pause/Resume
buttons proxy to the child (`POST /sprints/:id/live/pause`). The watchdog
treats a live paused child as still occupying the ledger.

That is correct for "pause this one run and come back to it on the same
reservation". It is the wrong shape for "pause so something else in a queue
can run".

---

## 2. Target behavior (mapped to the issue)

### Point 1 -- PENDING (zero members)

`POST /api/sprints` with `members: []` or `members` omitted:

- 201, `{ sprintId, state: "pending" }`
- Survives supervisor restart
- Appears on `GET /api/sprints` with a distinct state
- `GET /api/sprints/:id` returns the full stored request (issue, branch,
  base, goal, maxCycles, roleMap, budget, ...)
- No member reservation; not in the reservation ledger
- Issue/branch/base validation still happens now, not at start time
- Can be cancelled/deleted without ever running

### Point 2 -- WAITING (busy member), then auto-start

With `queue: true` (see D3): busy member -> 201 `state: "waiting"` instead
of 409.

When a sprint ends, stops, pauses (slice B), or is force-released, the
supervisor **re-evaluates** WAITING items against members that are free
**at that instant**. Eligibility is not reserved in advance.

Also:

- If a named member is later unregistered, the item becomes BLOCKED with a
  visible reason (not silent forever).
- Multi-member / `roleMap`: start only when the **entire** union is free.
  Two sprints each needing half the same pair must not each grab one member.
  Anti-starvation: skip an unsatisfiable head-of-line item rather than wait
  on it; operator can pin to override.
- Operator can still PATCH members and kick a start once they are free.
- Relaunch gate and topology checks run **again at execution**, not only at
  enqueue.

Manual start: PATCH members onto PENDING, or pin + wait for kick, or (same
as today) POST with free members and no queue flag to launch immediately.

### Point 3 -- Human-controllable ordering

Each PENDING/WAITING item has `priority` (higher first) and optional
`pinNext` (at most one pin). Default tie-break: older `createdAt` first,
then `sprintId`.

APIs:

- `PATCH /api/sprints/:id` -- priority, pin, members, goal/budget/roleMap
  on non-running items
- `POST /api/sprints/:id/reorder` -- `{ action: "front" | "back" }`
- `GET /api/sprints` -- running first, then queue in **consume order**
- History event on every reorder (actor + reason)
- Dashboard Queue section shows that same order (not API-only)

### Point 4 -- Pause releases capacity; resume waits

`POST /api/sprints/:id/pause`:

- Only RUNNING
- Uses the existing child `/pause` (cooperative, clean git/dolt boundary)
- When the child reports paused: **release supervisor ledger**, persist a
  PAUSED queue record (cycle, branch HEAD, budget spent, parked pid if any)
- Members show free on `GET /api/members`
- Scheduler may start the next WAITING sprint

`POST /api/sprints/:id/resume`:

- PAUSED -> WAITING (optional new `members` list)
- Does not require capacity in that HTTP call
- Same member + parked process still alive: claim, then child `/resume`
- Different members or dead pid: stop parked process, spawn from snapshot
  HEAD (runner already re-reserves and `bd dolt pull`s on resume)
- If branch HEAD is not the snapshot SHA and not a fast-forward: 409 naming
  both SHAs
- Child `--budget` on respawn = original cap minus consumed (spend cap must
  not reset)

### Point 5 -- History browsing, relaunch, clone

- Dashboard: 5 most recent **terminal** sprints, Load more pages the rest
- Relaunch: new sprint, body copied from the stored request, still validated
- Clone-and-amend: PENDING/WAITING/PAUSED only; original untouched
- RUNNING: no edit/clone-as-edit in the UI; API 409

---

## 3. State machine

```
PENDING   created, no members, nothing reserved
   |  PATCH members
   v
WAITING   members named, not reserved, waiting for them to be free
   |  entire union free + this item wins ordering
   v
RUNNING   ledger claim held, child spawned
   |
   +--> COMPLETED / FAILED / STOPPED   (history)
   |
   `--> PAUSED  ledger released, snapshot kept  -->  WAITING on resume
```

WAITING + deleted member -> BLOCKED.

DELETE allowed on PENDING / WAITING / PAUSED / BLOCKED.
RUNNING uses today's stop / force-release.

---

## 4. What we will not do in this issue

- Grooming intelligence (which beads, when) -- that is #22
- Cross-machine scheduling fancier than "entire requested union is free"
- Pre-reserving a member for a WAITING sprint
- Rebuilding engine pause/resume
- In-place mutation of a live sprint's parameters

---

## 5. Main code touchpoints

| Area | Path |
|---|---|
| Launch API | `packages/apra-fleet-se/src/supervisor/api.mjs` |
| New queue store | `packages/apra-fleet-se/src/supervisor/queue.mjs` (create) |
| Pick-next | `packages/apra-fleet-se/src/supervisor/scheduler.mjs` (create) |
| Capacity events | `watchdog.mjs` (after auto-release), `serve.mjs` wiring |
| UI | `dashboard.mjs`, `launch-form.mjs` |
| Contract | `docs/supervisor-api.md`, `docs/supervisor-openapi.yaml`, `fleet-sprint/skills/fleet-supervisor/SKILL.md` |
| Tests | `packages/apra-fleet-se/test/supervisor-*.test.mjs` (existing 409 tests stay valid under D3) |

---

## 6. Slice plan (after approval)

**Slice A (needed by #22)**

1. Queue store, restart-safe
2. PENDING create + GET
3. WAITING via `queue: true`; default 409 unchanged
4. Scheduler kick when members free
5. PATCH / reorder / DELETE + audit
6. Dashboard Queue section; launch form allows empty members
7. OpenAPI + SKILL.md

**Slice B (can overlap #22 once A is merged)**

1. Pause API releases ledger + snapshot
2. Resume -> WAITING; different-member respawn; SHA and budget rules
3. History list endpoint; dashboard recent-5 + relaunch + clone
4. Pause button on the stack calls the new API so capacity is actually freed

---

## 7. Risks to accept in review

- **Watchdog vs pause:** today paused = still on the ledger. Slice B
  changes that. The parked process must still be probed from the queue
  list, or a dead pid is invisible.
- **Scope overlap at start, not enqueue:** two queued sprints may name the
  same issue root. The second becomes BLOCKED when it would start, rather
  than speculatively claiming scope while idle.
- **D3 default:** if product wants WAITING-by-default, say so before Slice A
  tests are written the other way.

---

## 8. Approval

Reply on this table (copy is in the Decision table at the top). Status
moves to AGREED when there are no DISAGREE rows, or to REVISED after
edits.
