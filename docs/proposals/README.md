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
(`packages/apra-fleet-se/apra-pm/agents/backlog-groomer.md`). This is not a
component to be designed -- it is a working agent today, with **full beads
mutation authority**. On one operator's assigned beads it merges duplicates,
closes or defers stale items, reprioritizes, reparents and fixes bad edges
(every mutation evidence-gated), proposes cohesive sprint sets from
interdependencies, flags high-priority items too thin to act on, and
verifies landed-vs-closed. Its real deliverable is **beads mutations**, plus
a report.

What is missing is that **nothing invokes it programmatically**. `install.mjs`
copies the agent file; the schema tests validate its output contract; no
code dispatches it on a cadence or turns its proposed sprint sets into a
launch. It is run ad hoc by a human, who then fills in the Launch Sprint
form by hand.

That handoff -- not the analysis -- is the gap.

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
| Backlog-groomer agent | Shipped, with full beads mutation authority | Ranking, sprint sets, dedupe, quality bar, landed-vs-closed. Keep it. Extend it only where queueing needs it. Nothing invokes it programmatically -- that is the gap. |
| Supervisor launch API | Shipped as launch-now | We change it; we do not replace the supervisor. |
| Reservation ledger | Shipped | Still "who is reserved RIGHT NOW". Queued work must NOT live here, or waiting would look like reserved -- and the ledger feeds live scope re-expansion per launch, so extra rows cost real work. |
| Sprint history log | Shipped | Terminal events only (finished / crashed / released). Not a queue. |
| Engine pause/resume | Shipped (`requestPause` / `requestResume`) | A sprint parks at a clean git/dolt boundary and releases its *fleet* member reservations. But the supervisor still holds it on its own ledger, so the member is not free for the *next queued sprint*. #410 point 4 changes that -- and it is an invariant change, not a wrapper (see the #410 plan, section 3). |
| Dashboard Pause button | Shipped | Proxies to the child's `/pause`. Does not enqueue. |
| Dashboard Restart button | Shipped, destructive | Force-releases first, then prompts for anything it cannot recover. #410's relaunch work fixes this path rather than adding a parallel one. |

---

## 3. What we are asking Akhil to approve

Two plans, in order. Each plan has a **decision table** at the top. A
review is "AGREE / DISAGREE / NEEDS-DISCUSSION" on those rows, plus any
scope cut.

Recommended stance (also written into each plan):

1. Build **#410 first**, in two slices: (A) create/wait/order the queue,
   (B) pause-into-queue + relaunch/history.
2. Build **#22 after A**, so the groomer can POST PENDING sprints.
3. Groomer queues **work**, not machines (PENDING, `members: []`).
4. Keep today's fail-fast 409 unless the caller opts into `queue: true`,
   so Launch and Restart do not silently start waiting.
5. Do not auto-assign members until an operator turns that on (default off).
6. Cap autonomous queueing (`maxQueued`, optional daily USD).
7. Make the automatic grooming cadence **cheap by default**: a non-LLM
   backlog-change check decides whether an agent pass is worth running at
   all.

If those are wrong, it is cheaper to say so on the plan than after code.

### Round 2 (2026-09-17)

Akhil reviewed both plans on the PR. Every technical claim in that review
was re-checked against the code and holds; all points are accepted. The
material changes:

- **#410 D6** is restated as an invariant change (the watchdog deliberately
  never releases a paused sprint's reservation), with a concrete
  restart-recovery design for paused sprints and an explicit decision on the
  fleet-global reservation mirror.
- **#410 D7** is rescoped to fix the existing Restart button rather than
  build a second relaunch path.
- **#410 D4** gains a dual-source poll, so a member freed outside this
  supervisor cannot strand a waiting sprint.
- **#22 G5** keys idempotency off sorted bead ids, not an LLM-invented
  branch name.
- **#22 G7** (new) has the groomer absorb newly-ready beads into an existing
  queued sprint instead of proposing an overlapping new one.
- **#22 G2** replaces the hourly full-agent timer with a tiered cadence
  whose common case costs one `bd` call and zero tokens.

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
   retyping the form -- and Restart stops prompting you for fields it should
   already know.
6. After #22: the groomer keeps that queue filled on its own, without
   duplicating work already pending/waiting/running, without queueing items
   it already said were too thin to act on, and without burning an agent
   pass on a backlog that has not changed.

---

## 6. Please reply with

On each issue plan's decision table: AGREE / DISAGREE / NEEDS-DISCUSSION.

If DISAGREE, name the alternative. Both plans are currently **REVISED**
(round 2), with each plan carrying a response table showing how round-1
feedback was resolved. One open question needs an explicit answer: #410's
D6a proposes a short spike (can a paused run persist-and-exit resumably?)
before committing to the restart-recovery design.

Implementation of #22 stays blocked until #410 Slice A is agreed.
