# Dispatch: the state-to-action routing table

This is the spec a deterministic dispatcher implements: given one Linear
ticket's current state, which factory stage (if any) fires for it. It
replaces a standing `orchestrate` run for a repo that adopts it. The
dispatcher selects nothing, prioritizes nothing and batches nothing. A
human has already made the selection decision by moving a ticket to
`Todo`, and every later step follows from the ticket's state.

It reuses the vocabulary the skills define and redefines none of it.
The statuses, the labels, the four comment markers (`factory:
escalation`, `factory: stop-list hold`, `factory: deferred`, `factory:
decision`) and what makes a decision **pending** are as `orchestrate`
states them in "Statuses and labels" and "Escalation comments". Where
this file and a `SKILL.md` disagree, the `SKILL.md` wins, and the
disagreement is a bug in this file.

The dispatcher itself is not part of this plugin. This file is what it
builds against.

## Inputs

Everything the table reads is in Linear or on GitHub, and is read fresh
each time a ticket is evaluated:

- the ticket's status and labels;
- whether its description has a `## Plan` section;
- its blocked-by relations, and the status of each blocker;
- its comments, for the newest marker of each of the four kinds;
- its linked PR, found by the `<TICKET>: ...` title convention or the
  branch name, and that PR's state, draft flag, and CI result on its
  current head;
- the repo's `## Orchestrate` section in `AGENTS.md`, for the write
  window and everything the stages themselves read.

Dependencies are Linear blocked-by relations only, never prose in a
description. A ticket is **unblocked** when every ticket blocking it is
`Done`. A blocker that is `Canceled`, or any other status, holds the
ticket. A canceled blocker means the thing it waited on is not coming,
so a human removes the relation or cancels this ticket too.

The **current record** is the newest of the four markers, skipping a
stale decision, exactly as "Escalation comments" defines it.

## Exclusions

These are checked before any row is checked. A ticket that matches one
of them fires nothing:

1. It is `Backlog`, `Done` or `Canceled`.
2. It carries `needs-attention`. A human is meant to look first, and
   that label comes off only when they answer, either by posting a
   `factory: decision` or by editing the ticket and clearing the label.
3. An action is already in flight for it (see "In-flight").

A pending `factory: decision` overrides nothing in this list. A human
who posts one clears `needs-attention` in the same step.

## The table

Rows are checked top to bottom, and the first match fires. Each row's
stage is invoked exactly as its own skill describes, against this one
ticket.

| # | When | Fires |
| - | ---- | ----- |
| D | The current record is a pending `factory: decision` | its `Next` line, verbatim |
| P | `Todo`, unblocked, no `## Plan` | `plan-ticket <TICKET>` |
| I | `Todo`, unblocked, has `## Plan`, carries `plan-approved`, no `research` | `implement-ticket <TICKET>` |
| R | `Todo`, unblocked, has `## Plan`, carries `plan-approved`, carries `research` | `research-ticket <TICKET>` |
| V | `In Progress`, no `research`, an open **draft** linked PR with CI green on its head, current record not `factory: deferred` | `adversarial-review <PR>` |
| M | `In Review`, no `research`, an open linked PR with CI green on its head, carries `approved-to-merge` | `merge-queue <PR>` |
| — | anything else | nothing |

Row D comes first because a pending decision is the human's explicit
instruction for this ticket, and it holds whatever the ticket's status
is. The status rows are disjoint with each other, so their order
matters only for reading.

What each row deliberately does not do:

- **P** ignores `plan-approved`. If a ticket with no plan carries it,
  `plan-ticket` removes it before spawning the planner. An approval
  of a plan that does not exist yet approves nothing.
- **I** and **R** are the plan gate. A written plan waits in `Todo` until
  the human reads it and adds `plan-approved`. The dispatcher removes
  `plan-approved` in the same step it fires I or R, so one approval
  starts one stage. A ticket that comes back to `Todo` later, such as a
  research ticket whose researcher stopped, needs approving again.
- **V** fires on `In Progress`, not `In Review`. A green
  `implement-ticket` leaves the ticket `In Progress` with a draft PR, and
  `adversarial-review` moves it to `In Review` when its cycle starts. So
  `In Review` means review has started, and it is never re-fired from
  status alone. A capped, blocked or failed review carries
  `needs-attention`. A review that came back ready has posted its
  `factory: stop-list hold` and sits in `In Review`, waiting on row M.
- **M** is only a pointer. `merge-queue`'s own "Eligibility" stays the
  final authority, including the stop-list rule that the human clear
  that PR specifically. The dispatcher never adds `approved-to-merge`.
  M does not exclude a deferred current record: an ordinary merge
  deferral keeps the PR eligible, as `merge-queue` already says.
- **Nothing** is free. A ticket that matches no row costs no agent.

## Gates under dispatch

The dispatcher has no modes, and nothing at invocation can release a
gate:

- **Selection** is the human moving a ticket to `Todo`. There is no
  batch, no cap and no pre-filter: the planner is the only judge of
  whether a ticket can be planned, and it reports one that cannot as
  `unplannable` (see "Plan and research").
- **Plan** is always held, per ticket, and released only by the human
  adding `plan-approved`. `hold-plan` adds nothing on top of that.
  `plan-approved` is the human approving that ticket's plan in person,
  which is the one thing that clears `hold-plan` today.
- **Merge** is always held, per ticket, and released only by the human
  adding `approved-to-merge` (or, for an unlinked PR, an approving
  GitHub review, which is outside this table). `hold-merge` adds nothing
  on top, since the dispatcher never adds `approved-to-merge` itself.

Every escalation rule still applies unchanged, because the stages
enforce them, not the dispatcher. A capped review never merges, a
blocked or failed ticket is never cleared, and a stop-list hit waits
for the human.

## Plan and research

Rows P and R invoke a stage skill, like every other row: P invokes
`plan-ticket <TICKET>`, and R invokes `research-ticket <TICKET>`. Each
skill owns its subagent, its status moves, its escalations and its
telemetry rows, so the dispatcher adds nothing around them, with one
exception. On a `planned` result, the ticket stays in `Todo` and waits
for `plan-approved`. Surface the report's `STOPLIST` line wherever the
dispatcher notifies the human, so they know at approval time which
merges will wait for them.

Either skill can report `skipped`, when the ticket changed between the
dispatcher's read and the stage's own check. Nothing was written, so the
dispatcher releases its claim and evaluates the ticket again on its next
read.

A research ticket never matches V or M, and a `factory: decision` is
never posted for one (see "Escalation comments"), so D cannot route it
either.

## In-flight

A ticket's status does not change while its planner runs, and an
implementer can turn its PR green a moment before it reports. So the
dispatcher holds a per-ticket claim from the moment it fires a row until
that stage returns, and it never fires a second row for a claimed
ticket. This is also the guard `orchestrate`'s "Escalation comments"
leaves to whatever runs a decision's `Next`: a pending decision can be
read again while its resume is still running.

If a claim is lost, for example because the dispatcher crashed, the row
may fire again. Every stage tolerates this. A planner replaces
`## Plan`, `implement-ticket` resets the worktree, and
`adversarial-review` starts again at round 1.

A ticket that is `In Progress` or `In Review`, unclaimed, and matches no
row is either waiting on a human or orphaned. A crashed implementer is
one example: `In Progress`, no green PR, no marker. The dispatcher
reports orphans to the human and never routes them.

## The write window

Rows I, V and M, and a D whose `Next` is one of those stages, lead to
gated writes. While the repo's write window is closed, the dispatcher
does not fire them, so they don't spend a stage only to defer. Rows P
and R are read-only work and fire whatever the window says. A stage that
defers anyway, because the window closed mid-stage, leaves a `factory:
deferred` note. As today, the human restarts it, either by invoking the
stage or by posting a `factory: decision`. M is the one exception: an
ordinary merge deferral stays eligible and is picked up again once the
window opens.

## Stage parameters

- **Review budget**: `adversarial-review`'s default of six rounds, which
  is the supervised budget. Without a mode, the dispatcher has no
  authority to grant ten.
- **Telemetry**: each fired row is its own run. Pass a run id per firing
  as `orchestrate`'s "Telemetry" describes.
- **Models and effort**: as `orchestrate`'s "Models and effort".

## What is gone

These are deliberate losses compared with `orchestrate`, so nobody goes
looking for them:

- **Waves.** No file-overlap serialization. Ordering comes from
  blocked-by relations only, and `merge-queue` absorbs conflicts at
  rebase time.
- **Judgment at pick time.** No "too vague", no "plainly a project", no
  priority reordering, no `Backlog` promotion proposals.
- **Released gates.** No mode releases plan or merge.

## Scope

- A ticket-less PR fires no row. `adversarial-review` and
  `merge-queue` still handle one when invoked standalone.
- Running `orchestrate` and a dispatcher against the same repo at the
  same time fires stages twice. Use one or the other per repo.
