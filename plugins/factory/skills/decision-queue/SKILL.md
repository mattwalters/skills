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

Three sources, all of them read straight from Linear and GitHub. A
ticket can carry more than one marker comment at once — a deferred
fixer's stop-list hold, a stuck `merge-queue`'s deferral and
escalation — so read each kind's **latest** comment separately
(`factory: escalation`, `factory: stop-list hold`, `factory: deferred`
— see the `dispatch` skill's "Escalation comments": only the latest
comment of *each kind* counts, never the latest of any kind across
them), and decide which source applies with this fixed order, so the
result never depends on which of two comments a stage happened to post
second:

1. An open **`needs-attention`** label with its latest `factory:
   escalation` comment always wins, whatever else is posted — this is
   a queue item (below), excluding `Done` and `Canceled` tickets. If a
   labelled ticket has no such comment, list it anyway, with one line
   saying no escalation was recorded rather than skipping it. This is
   why a stuck `merge-queue` deferral — which also carries an
   escalation and the label (see that skill's "On a deferred report")
   — is always a queue item here and never the FYI deferred line
   below, whichever of its two comments happened to post second. The
   escalation comment can also carry a `Stop-list` line (see the
   `dispatch` skill's "Escalation comments") when a blocked, failed, or
   capped review result hit the stop-list; read it the same way you
   read `Entries` on a hold comment.
2. Otherwise, the team's `In Review` tickets with an open PR whose
   latest `factory: stop-list hold` comment names at least one entry
   (`Entries` is not `none`), that carry neither
   **`approved-to-merge`** nor **`needs-attention`**. A hold is live
   only if it is the newest of the ticket's latest hold, latest
   escalation, and latest deferral note — any newer escalation or
   deferral note supersedes it, the same way `approved-to-merge`
   clears it, whatever either one says. Every ready cycle posts a
   hold, `Entries: none` included (see `adversarial-review`'s
   "Recording the stop-list hold"), so among holds alone the latest is
   always the whole of the current truth: an `Entries: none` hold
   clears an earlier hit as surely as a later non-empty hold
   re-asserts it — but a newer escalation or deferral note still beats
   it regardless. A ticket whose latest escalation or deferral note is
   newer than its latest hold isn't this source: a newer escalation
   means a standalone re-review has already moved past that hold (that
   ticket is source 1 above if `needs-attention` is still open, the
   FYI list otherwise), and a newer deferral note means a deferred
   review never came back ready, so that ticket is the FYI deferred
   line below instead — neither is a merge-approval question.
3. Open PRs with no linked ticket that carry either marker comment,
   found the same way `adversarial-review` and `merge-queue` post
   them: on the PR itself, since there's no ticket to hold it. Same
   shape as source 2, applied to the PR in place of a ticket: a hold
   counts as live only while it names at least one entry and is the
   newest of the PR's latest hold, latest escalation, and latest
   deferral note — any newer escalation or deferral note supersedes
   it, whatever either one says, the same way a later approving GitHub
   review does; the PR closing clears it too. None of that turns on
   the PR's commit history: a plain push with no marker, escalation,
   deferral, or approval of its own changes nothing here. Every ready
   cycle posts a hold, `Entries: none` included (see
   `adversarial-review`'s "Recording the stop-list hold"), so among
   holds alone a live one here never goes stale for want of a fresh
   comment.

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
item outranks an ordinary one — that includes a source 1 item whose
escalation comment's `Stop-list` line is non-`none`, not only a
source 2 or source 3 hold.

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

A stop-list hold item — source 2, or a source 3 item carrying that
marker — has no `Stage`, `Question`, or `Options` to read; the hold
comment carries only `PR`, `Entries`, and `Round`. Build it in this
fixed shape instead, every time: **Stopped at** is `ready, waiting on
stop-list merge approval`; the heading question is `Merge PR <n>? It
touches: <Entries>`; the options are always **A.** merge and **B.**
don't merge yet — leaves it queued here. Option A's consequence is
source-specific: for a source 2 item it's "clears the hold and lets it
proceed" (adding `approved-to-merge` is what clears it); for a source 3
item it's "approve the PR on GitHub, which clears the hold and lets it
proceed" — an unlinked PR's hold clears on an approving GitHub review,
not a label. Never invent a `Stage` or extra options for this kind of
item; this fixed shape is the whole of what a hold comment gives you.

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

- **Deferred.** A ticket whose latest `factory: deferred` comment is
  the source that applies under "What goes in the queue" above (no
  open `needs-attention` escalation, and no stop-list hold that beats
  it), and whose status and PR haven't moved since — nobody has
  restarted it yet. The deferral note itself names no restart
  invocation or PR number (see the `dispatch` skill's "The deferral
  note"); derive one instead from the note's `Stage` line plus the
  ticket id or its linked PR: `implement` → `implement-ticket
  <TICKET>`, `review-fixer` or `trivial-minors` → `adversarial-review
  <PR>`, `merge-resolver` or `merge` → `merge-queue <PR>`. A
  `merge-queue` deferral that also carries `needs-attention` is the
  stuck sub-case — that one is a queue item above (source 1), never
  this line. Once a restart pushes something, the ticket's state moves
  and this line stops applying.
- **Awaiting merge approval, or still in review.** A ticket that's
  `In Review` with an open PR, not carrying `needs-attention`, with no
  live stop-list hold under source 2's rule above, and not already
  covered by the Deferred line above — it's simply ready and waiting,
  or still mid-review. A marker from an earlier, now-resolved stop
  doesn't keep it off this line; only a currently live hold, an open
  escalation, or a live deferral note does. A stop-list hold is not
  this line even though it is also "waiting" in a sense — it's a queue
  item (source 2) above, because clearing it takes a specific human
  call rather than just watching a review finish. Nor is a live
  deferral: that ticket isn't "still in review", it's stopped until a
  human restarts it, which is exactly what the Deferred line already
  says. This is not a decision for the human either; it's here so the
  report accounts for every ticket a
  run touched, not only the ones stuck.

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
