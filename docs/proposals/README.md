# Supervisor queue + grooming -- cover note for review

- Status: PROPOSED
- Proposed by: Siddharth Deshpande (assignee)
- For: Akhil Kumar (issue author)
- Date: 2026-09-08
- GitHub: https://github.com/Apra-Labs/apra-fleet/issues/410 and https://github.com/Apra-Labs/apra-fleet/issues/22

This cover is the "what is going on" document. The two issue plans are the
artifacts to approve or send back:

1. `docs/proposals/issue-410-persistent-sprint-queue.md` -- **approve first**
2. `docs/proposals/issue-22-autonomous-backlog-grooming.md` -- approve second;
   it cannot ship until #410's queue exists

Engineering task breakdown (how to implement, file by file) lives under
`docs/superpowers/plans/`. That is not what to review here. Review the two
issue plans: the behavior, the defaults, and the sequencing.

---

## 1. The picture in one page

Apra Fleet runs **sprints**: an AI loop (plan, implement, review, ...) against
a beads backlog, dispatched onto **members** (machines / agents in the fleet).

Operators are not supposed to start sprints by invoking the sprint CLI.
They go through the **supervisor** (`packages/apra-fleet-se/bin/serve.mjs`,
default `http://localhost:8787`):

```
operator (or later, the groomer)
        |
        v
  supervisor HTTP API + dashboard
        |
        +-- reservation ledger: "who is busy RIGHT NOW"
        +-- spawn a sprint child when a launch is accepted
        |
        v
  fleet members actually do the work
```

Today that API is **launch now, or fail**:

- You must name at least one member. Empty list = HTTP 400.
- If that member is already on another sprint = HTTP 409. The request is
  thrown away. Nothing is remembered.
- The dashboard Sprint Stack only shows **running** sprints.
- When a sprint finishes, it disappears from the stack. History exists as a
  per-sprint page, not as a queue or a "run this again" list.

Separately, there is a **backlog-groomer** agent
(`packages/apra-fleet-se/apra-pm/agents/backlog-groomer.md`). It already
knows how to look at one person's assigned beads and say:

- these items are ready / urgent
- these three belong in one sprint
- these two are duplicates
- this P0 is too thin to act on

What it cannot do is **put that conclusion anywhere the supervisor will
run**. A human reads the report, then fills in the Launch Sprint form.

That human handoff is the whole gap.

```
TODAY

  groomer writes a report  -->  human launches a sprint  -->  it runs or 409s


TARGET (issue #22 on top of issue #410)

  groomer grooms on a cadence
        --> queues sprint requests into the supervisor
        --> supervisor runs the next eligible one when a member is free
        --> operator can still reorder, pause, cancel, or assign members
```

**#410 is the conveyor belt. #22 is the worker that loads it.**

Without #410, #22 still has nowhere to put a sprint except a message to a
human. That is why these are two GitHub issues, and why they are one program
of work.

---

## 2. What already shipped (so we do not rebuild it)

| Piece | Status | Meaning for this work |
|---|---|---|
| Backlog-groomer analysis | Shipped | Ranking, sprint sets, dedupe, quality bar. Keep it. Extend it. |
| Supervisor launch API | Shipped as launch-now | We change it; we do not replace the supervisor. |
| Reservation ledger | Shipped | Still "who is reserved RIGHT NOW". Queued work must NOT live here, or waiting would look like reserved. |
| Sprint history log | Shipped | Terminal events only (finished / crashed / released). Not a queue. |
| Engine pause/resume | Shipped (`requestPause` / `requestResume`) | A sprint can park at a clean git/dolt boundary and release *fleet* member reservations. The supervisor still keeps the sprint on its ledger, so the member is not free for the *next queued sprint*. #410 point 4 is the wrapper that actually frees capacity into a queue. |
| Dashboard Pause button | Shipped | Proxies to the child's `/pause`. Does not enqueue. |

Akhil's comment on #410 already said this: point 4 is a supervisor-queue
wrapper around pause, not a rebuild of the engine primitive.

---

## 3. What we are asking Akhil to approve

Two plans, in order. Each plan has a **decision table** at the top. A
review is "AGREE / DISAGREE / NEEDS-DISCUSSION" on those rows, plus any
scope cut.

Recommended stance (also written into each plan):

1. Build **#410 first**, in two slices: (A) create/wait/order the queue,
   (B) pause-into-queue + history/relaunch UI.
2. Build **#22 after A**, so the groomer can POST PENDING sprints.
3. Groomer queues **work**, not machines (PENDING, `members: []`).
4. Keep today's fail-fast 409 unless the caller opts into `queue: true`,
   so Launch and Restart do not silently start waiting.
5. Do not auto-assign members until an operator turns that on (default off).
6. Cap autonomous queueing (`maxQueued`, optional daily USD).

If those six are wrong, it is cheaper to say so on the plan than after code.

---

## 4. How Siddharth will contribute

After approval of #410 (at least slice A):

- Implement against `packages/apra-fleet-se/src/supervisor/` (API, new queue
  store, scheduler, dashboard) and the existing tests in
  `packages/apra-fleet-se/test/supervisor-*.test.mjs`.
- Keep `packages/apra-fleet-se/docs/supervisor-api.md`,
  `supervisor-openapi.yaml`, and
  `fleet-sprint/skills/fleet-supervisor/SKILL.md` in the same change.
- Not in v1 unless a later PR adds MCP tools: `packages/apra-fleet-client`.

#22 work is schema + groomer markdown + a supervisor runner that POSTs into
the #410 API. The analysis agent is not rewritten from scratch.

---

## 5. What "done" looks like for the operator

1. You can POST a sprint with no members. It sits in the dashboard as
   PENDING. Restarting the supervisor does not lose it.
2. You can POST a sprint with `queue: true` against a busy member. It sits
   as WAITING and starts when that member is actually free. Nothing is
   reserved while it waits.
3. You can pin, reorder, or cancel queued items. The list order is the
   order the supervisor will consume.
4. You can pause a long sprint so an urgent one can take the machine, then
   resume the first one later (including on a different member) without
   resetting its budget.
5. You can see the last few finished sprints and relaunch one without
   retyping the form.
6. After #22: the groomer can fill that queue on a timer, without duplicating
   work that is already pending/waiting/running, and without queueing items
   it already said were too thin to act on.

---

## 6. Please reply with

On each issue plan's decision table: AGREE / DISAGREE / NEEDS-DISCUSSION.

If DISAGREE, name the alternative. The plans stay PROPOSED until that
table is clean. Implementation of #22 stays blocked until #410 slice A is
agreed (and preferably merged).
