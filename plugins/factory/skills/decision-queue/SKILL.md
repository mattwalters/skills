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
ticket (or unlinked PR) can carry more than one marker comment at
once — a deferred fixer's stop-list hold, a stuck `merge-queue`'s
escalation — so read each kind's **latest** comment separately
(`factory: escalation`, `factory: stop-list hold`, `factory: deferred`
— see the `dispatch` skill's "Escalation comments": only the latest
comment of *each kind* counts, never the latest of any kind across
them), then apply the one rule that decides which source applies and
whether it's still live: **the newest of the three latest comments is
the ticket's (or PR's) current record**, whatever the older two say
(see the `dispatch` skill's "Escalation comments"). There is no
per-source freshness test beyond that single comparison, and it
applies the same way to a ticket or an unlinked PR.

1. **Newest marker is an escalation, ticketed.** An open
   **`needs-attention`** label together with that escalation comment is
   a queue item (below), excluding `Done` and `Canceled` tickets — the
   label coming off is the human's own signal that they've looked, even
   before any new marker is posted, so an escalation whose ticket has
   already lost the label isn't a queue item. If a labelled ticket has
   no escalation comment at all, list it anyway, with one line saying
   no escalation was recorded rather than skipping it. This is why a
   stuck `merge-queue` deferral — which posts only the escalation,
   never a separate deferral note (see the `dispatch` skill's
   "Escalation comments") — is always a queue item here for as long as
   `needs-attention` stays on, whatever an older, now-superseded hold
   or deferral note on the same ticket says. The escalation comment can
   also carry a `Stop-list` line (see the `dispatch` skill's
   "Escalation comments") when a blocked, failed, or capped review
   result hit the stop-list; read it the same way you read `Entries` on
   a hold comment.
2. **Newest marker is a stop-list hold naming at least one entry,
   ticketed.** The team's `In Review` tickets with an open PR whose
   newest marker is a `factory: stop-list hold` comment with `Entries`
   not `none`, that carry neither **`approved-to-merge`** nor
   **`needs-attention`**. Every ready cycle posts a hold, `Entries:
   none` included (see `adversarial-review`'s "Recording the stop-list
   hold"), so among holds alone the latest is always the whole of the
   current truth: an `Entries: none` hold becoming the newest marker
   clears an earlier hit as surely as a later non-empty hold re-asserts
   it. A ticket whose newest marker is instead an escalation or a
   deferral note isn't this source: an escalation newer than the
   ticket's latest hold means a standalone re-review has already moved
   past that hold (that ticket is source 1 above if `needs-attention`
   is still open, an FYI line otherwise), and a deferral note newer
   than the latest hold means a deferred review never came back ready,
   so that ticket is the FYI deferred line below instead — neither is a
   merge-approval question.
3. **Newest marker is an escalation, or a stop-list hold naming at
   least one entry, on an open PR with no linked ticket.** Found the
   same way `adversarial-review` and `merge-queue` post them: on the PR
   itself, since there's no ticket to hold it. Whichever kind is newest
   decides the item's shape — escalation-shaped per source 1's item
   shape, or hold-shaped per source 2's — and either kind counts as
   live only while the PR stays open and has gained no newer
   *approving* GitHub review since: a later approving review clears an
   unlinked PR's marker the same way `approved-to-merge` clears a
   ticketed one, whatever the marker says, and the PR closing clears it
   too. This is what keeps a capped or stuck review's question from
   outliving its own answer: once a later cycle posts a newer marker —
   a ready cycle's hold, `Entries: none` included, or a fresh
   escalation of its own — that comment is what's newest, and the older
   one it followed stops being read at all, whether or not the PR has
   closed. None of that turns on the PR's commit history: a plain push
   with no marker, escalation, deferral, or approval of its own changes
   nothing here.

A ticket or PR that matches more than one of these is one item, not
two — it can't: the newest marker is a single comment, so only one
bullet's condition is ever the one that applies.

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

- **Deferred.** A ticket or unlinked PR whose newest marker (see "What
  goes in the queue" above) is a `factory: deferred` comment, and whose
  status and PR haven't moved since — nobody has restarted it yet. The
  deferral note itself names no restart invocation or PR number (see
  the `dispatch` skill's "The deferral note"); derive one instead from
  the note's `Stage` line plus the ticket id or its linked PR:
  `implement` → `implement-ticket <TICKET>`, `review-fixer` or
  `trivial-minors` → `adversarial-review <PR>`, `merge-resolver` or
  `merge` → `merge-queue <PR>`. `merge-queue`'s stuck sub-case posts
  only the escalation, never a separate deferral note (see the
  `dispatch` skill's "Escalation comments"), so it is never this line —
  it's a queue item above (source 1) for as long as `needs-attention`
  stays on. Once a restart posts anything at all, that new comment is
  the newest marker and this line stops applying on its own, whether or
  not the restart ever pushed a commit: a review that comes back ready
  without pushing still posts a fresh hold (`Entries: none` included —
  see `adversarial-review`'s "Recording the stop-list hold"), and that
  hold outranks the older deferral note the moment it posts, moving the
  ticket to the "Awaiting merge approval" line below instead.
- **Awaiting merge approval, or still in review.** A ticket that's
  `In Review` with an open PR, not carrying `needs-attention`, and
  whose newest marker (see "What goes in the queue" above) is either
  nothing at all or a stop-list hold naming no entries. A ticket isn't
  this line if its newest marker is a live escalation or a non-empty
  hold — those are queue items above instead, because clearing them
  takes a specific human call rather than just watching a review
  finish — and isn't this line if its newest marker is a deferral note —
  that's the Deferred line above instead, since that ticket isn't
  "still in review", it's stopped until a human restarts it. This is
  not a decision for the human either; it's here so the report accounts
  for every ticket a run touched, not only the ones stuck.

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
