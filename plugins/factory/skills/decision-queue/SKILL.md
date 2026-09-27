---
name: decision-queue
description: Turn everything a dispatch run stopped on into a queue of answerable decisions — each blocked, failed, capped or stop-listed item stated as one question with its options and a recommendation, ordered by how much work an answer unblocks. Reads the host repo's `AGENTS.md` `## Dispatch` section for its Linear team and run manifest. Use when asked for a status report, what needs a decision, what is blocked, or what to look at first. Called by the dispatch skill in place of a plain status table; equally fine invoked standalone against a repo mid-run or the morning after one.
---

# Decision queue

Everything in this pipeline that stops, stops because a human has to
decide something. This skill renders those into a queue to answer, not
a report to read: one question per item, the options laid out, a
recommendation attached, ordered by how much other work the answer
releases.

You do not read diffs, CI logs, or PR threads to build it. The
subagent that stopped already wrote the decision down while it still
had the context to do that cheaply; you are collecting and ordering,
not reconstructing. If an item's escalation recorded no options, say
so in one line rather than inventing them — a missing recommendation
is a fact about the run, and a guessed one is worse than the gap.

## Repo configuration

Read the `## Dispatch` section of the host repo's `AGENTS.md` first:
the **Linear team key**, and the **run manifest** this reads its
in-flight state from. No `## Dispatch` section means the repo has not
opted into the pipeline — say so and stop.

## What goes in the queue

Four sources, all of them already recorded somewhere:

- Tickets carrying the **`needs-attention`** label. Every blocked and
  failed report anywhere in the pipeline leaves one.
- PRs whose review came back **capped**, which never merge on their
  own whatever mode the run was in.
- Tickets whose merge is held by the **stop-list**, waiting on a
  clearance no released gate can give.
- Anything the run manifest records as waiting at a held gate, if a
  run is still in flight.

A ticket that appears in more than one of those is one item, not two.

## Ordering

Sort by **downstream unblock impact** — how much other work an answer
releases — not by when the item stopped. For each item count the
tickets blocked by it in Linear, the tickets serialized behind it in
the current batch, and the tickets whose plans touch the same files.
Ten tickets waiting on one ruling belongs at the top even if it
stopped an hour ago; a leaf that stopped first thing belongs at the
bottom.

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
