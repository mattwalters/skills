---
name: decision-queue
description: Turn everything a dispatch run stopped on into a queue of answerable decisions — each blocked, failed, capped or stop-listed item stated as one question with its options and a recommendation, ordered by how much work an answer unblocks. Reads the host repo's `AGENTS.md` `## Dispatch` section for its Linear team, and reads Linear and the open PRs directly — it keeps no state of its own. Use when asked for a status report, what needs a decision, what is blocked, or what to look at first. Called by the dispatch skill in place of a plain status table; equally fine invoked standalone against a repo mid-run or the morning after one.
---

# Decision queue

Everything in this pipeline that stops, stops because a human has to
decide something. This skill renders those into a queue to answer, not
a report to read: one question per item, the options laid out, a
recommendation attached, ordered by how much other work the answer
releases.

You do not read diffs, CI logs, or PR threads to build it. The
subagent that stopped already wrote the decision down, as a `factory:
escalation` or `factory: stop-list hold` comment on the ticket or PR,
while it still had the context to do that cheaply; you are collecting
and ordering those comments, not reconstructing anything from scratch.
If an item's escalation comment recorded no options, say so in one
line rather than inventing them — a missing recommendation is a fact
about the run, and a guessed one is worse than the gap.

## Repo configuration

Read the `## Dispatch` section of the host repo's `AGENTS.md` first:
the **Linear team key**. No `## Dispatch` section means the repo has
not opted into the pipeline — say so and stop. This skill keeps no
state of its own and reads no run manifest, whether or not the repo's
section still declares one.

## What goes in the queue

Three sources, all of them read straight from Linear and GitHub:

1. The team's tickets carrying the **`needs-attention`** label,
   excluding `Done` and `Canceled`, each paired with its **latest**
   `factory: escalation` comment (see the `dispatch` skill's
   "Escalation comments" for the comment's shape and how to find the
   latest one). If a labelled ticket has no such comment, list it
   anyway, with one line saying no escalation was recorded rather than
   skipping it.
2. The team's `In Review` tickets whose **latest** `factory:
   stop-list hold` comment is on an open PR, and which don't yet carry
   **`approved-to-merge`** — once that label lands, the hold has been
   cleared and the ticket drops out.
3. Open PRs with no linked ticket that carry either marker comment,
   found the same way `adversarial-review` and `merge-queue` post them:
   on the PR itself, since there's no ticket to hold it.

A ticket or PR that matches more than one of these is one item, not
two.

## Ordering

Sort by **downstream unblock impact** — how much other work an answer
releases — not by when the item stopped. For each item count the
tickets blocked by it in Linear, the tickets serialized behind it in
the caller's wave plan (if the caller passed one — a standalone
invocation won't have one, and that term of the count is then zero),
and the open PRs that share files with it (`gh pr diff --name-only`,
never diff content — the same lightweight overlap check `dispatch`'s
wave planning and `merge-queue`'s ordering use). Ten tickets waiting on
one ruling belongs at the top even if it stopped an hour ago; a leaf
that stopped first thing belongs at the bottom.

Break ties by the cost of being wrong: at equal impact, a stop-list
item outranks an ordinary one.

State the count in the item itself, so the ordering is visible rather
than asserted.

## Item shape

One per item, in this shape, and nothing longer:

    ### <TICKET> — <the question, as a question>

    **Unblocks:** <n tickets: ids> · **PR:** <url> · **Stopped at:**
    <planning | implementing | review round n | merge>

    <One or two sentences on what was found. Not a narrative of the
    run.>

    - **A.** <option> — <consequence>
    - **B.** <option> — <consequence>

    **Recommended: <letter>.** <One sentence of why.>

`Stopped at` and the question both come straight from the escalation
comment: `Stopped at` is its `Stage` line, and the heading question is
built from its `Question` line — you are rendering what the comment
already says, not re-deriving it from the diff.

Two or three options. If the honest answer is that there are two and
one of them is "drop the ticket", say that — a queue that dresses
every decision up as interesting is a queue that stops being read.

The question in the heading is the point of the whole item. "WRIT-88 —
should the merge keep the old field name or take the rename?" is a
question someone can answer in the time it takes to read it. "WRIT-88
— blocked on a rebase conflict" is a report, and it makes them go
looking.

## FYI, below the queue

Under the queue, clearly separated from it: work the run discovered
but was not asked to do — bugs a subagent noticed outside its ticket's
scope, invariants that look stale, minors a review left open, findings
a fixer rebutted rather than fixed. These are not decisions and carry
no options. One line each.

Two further kinds of line belong here too, since neither is a decision
either:

- **Deferred.** A ticket whose latest `factory:` comment is
  `**factory: deferred**`, and whose status and PR haven't moved since
  — nobody has restarted it yet. Show the restart invocation the
  deferral note names (`implement-ticket <TICKET>`,
  `adversarial-review <PR>`, or `merge-queue <PR>`) so a human reading
  this doesn't have to go find it. Once a restart pushes something,
  the ticket's state moves and this line stops applying.
- **Awaiting merge approval, or still in review.** A ticket that's
  `In Review` with an open PR, carrying neither `needs-attention` nor a
  stop-list hold — it's simply ready and waiting, or still mid-review.
  This is not a decision for the human either; it's here so the report
  accounts for every ticket a run touched, not only the ones stuck.

If the human asks for any of them to be filed, file into **`Backlog`**,
never `Todo`. `Backlog` is where they park what they have not started,
and promoting to `Todo` is their act; a pipeline that files into its
own queue feeds itself. Never file one without being asked.

## Report

The queue, then the FYI list, then one line: how many items are
waiting, and how many tickets in total sit behind them.

If nothing is waiting, say that in one line and render no empty
headings. "Nothing needs you" is the most useful report this skill can
produce, and it should be one sentence long.
