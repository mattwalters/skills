---
name: decision-queue
description: Turn everything an orchestrate run stopped on into a queue of answerable decisions — each blocked, failed, capped or stop-listed item stated as one question with its options and a recommendation, ordered by how much work an answer unblocks. Reads the host repo's `AGENTS.md` `## Orchestrate` section for its Linear team, and reads Linear and the open PRs directly — it keeps no state of its own. Use when asked for a status report, what needs a decision, what is blocked, or what to look at first. Called by the orchestrate skill in place of a plain status table; equally fine invoked standalone against a repo mid-run or the morning after one.
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

Read the `## Orchestrate` section of the host repo's `AGENTS.md` first:
the **Linear team key**. No `## Orchestrate` section means the repo has
not opted into the pipeline — say so and stop. This skill keeps no
state of its own and reads no run manifest, whether or not the repo's
section still declares one.

## What goes in the queue

Everything is read straight from Linear and GitHub. A ticket (or
unlinked PR) can carry more than one marker comment at once — a
deferred fixer's stop-list hold, a stuck `merge-queue`'s escalation —
so read each kind's **latest** comment separately (`factory:
escalation`, `factory: stop-list hold`, `factory: deferred`, `factory:
decision` — see the `orchestrate` skill's "Escalation comments": only
the latest comment of *each kind* counts, never the latest of any kind
across them), then apply the one rule that decides which of the four a
row below is reading: **the newest of the four latest comments is the
ticket's (or PR's) current record**, whatever the older three say (see
the `orchestrate` skill's "Escalation comments") — except that a stale
`factory: decision` drops out of this comparison entirely, per that
same rule, leaving the newest of the other three, or no marker at all
if none of them exists, as the current record instead. A `factory:
decision` marker carries one freshness test beyond that single
comparison — the pending/stale rule in the `orchestrate` skill's
"Escalation comments". That rule decides two things: whether a
decision is pending enough to match row 0 below, and, once it's stale,
that it drops out of the newest-marker comparison for every row below,
not only row 0 — row 3 applies a different test of its own, the
deferral's status-and-head check, not this one. It applies the same way
to a ticket or an unlinked PR.

Check every ticket, and every unlinked open PR, against this table, top
to bottom. The first row that matches decides where it lands; every
ticket and every unlinked open PR matches exactly one row. This applies
only to tickets that aren't already `Done` or `Canceled`, and, for
unlinked PRs, only to open ones.

| # | Condition | Goes to |
|---|---|---|
| 0 | Newest marker is a *pending* `factory: decision` (see the `orchestrate` skill's "Escalation comments"), whether or not `needs-attention` is still on | FYI — Decided, naming its `Next` invocation (see "FYI, below the queue"). One exception: if `Next` is `merge-queue <PR>` and the PR isn't yet eligible under `merge-queue`'s "Eligibility" — `needs-attention` is still on, the ticket carries no **`approved-to-merge`** (unlinked PR: no GitHub review approving the PR — and, when the PR's latest `factory: stop-list hold` comment names at least one entry, none newer than that hold), or its checks aren't green at the current head — it's a **queue item** instead, fixed-shaped (see "Item shape") — the guard that keeps a stop-list hold, which a newer decision would otherwise supersede, from dropping out of the queue, and that keeps a decision `Next` can never run from sitting silent as an FYI |
| 1 | Has an open **`needs-attention`** label | Queue item — if the current record (the newest-marker rule above, which skips a stale decision) is a `factory: escalation` comment, escalation-shaped, built from it as usual; if it's a hold or a deferral instead, the question is "`needs-attention` is still on, but the latest record is a <hold\|deferral>: clear the label?"; if there is no marker at all — including when the only marker on the ticket is a stale decision — list it anyway with one line saying no escalation was recorded |
| 2 | The current record (the newest-marker rule above, which skips a stale decision) is a `factory: stop-list hold` naming at least one entry (`Entries` not `none`), the ticket is `In Review` with an open PR, and it carries no **`approved-to-merge`** | Queue item, hold-shaped (fixed shape — see "Item shape") |
| 3 | Newest marker is a `factory: deferred` comment, and the ticket's status and PR head haven't changed since it was posted | FYI — Deferred, with a restart invocation derived from its `Stage` line (see "FYI, below the queue") |
| 4 | `In Review`, or `In Progress` with an open PR — everything else lands here: a hold naming no entries, an escalation whose ticket has already lost the label, no marker at all (including when the only marker on the ticket is a stale decision), or `approved-to-merge` sitting on a green PR waiting for `merge-queue` | FYI — implemented and awaiting review, still in review, awaiting merge approval, queued to merge, or (a `research` ticket in `In Review`, which has no PR) research report awaiting acceptance |
| 5 | Anything else | Not listed |

Row 0 sits above row 1 on purpose: a pending decision means the
question row 1 would otherwise ask has already been answered, and
re-asking it is the failure this ordering exists to prevent.
`needs-attention` still being on is noted on the FYI line itself
("`needs-attention` still on"), not turned back into a question —
except when `Next` is `merge-queue <PR>`, where the label alone is what
row 0's exception turns into a queue item instead: `merge-queue`'s
Eligibility refuses any ticket carrying it, so `Next` can never run
while it's on.

The label coming off row 1 is the human's own signal that they've
looked, even before any new marker is posted — an escalation whose
ticket has already lost the label falls through to row 4, not a queue
item. This is why a stuck `merge-queue` deferral — which posts only the
escalation, never a separate deferral note (see the `orchestrate` skill's
"Escalation comments") — is always row 1 for as long as
`needs-attention` stays on, whatever an older, now-superseded hold or
deferral note on the same ticket says. The escalation comment can also
carry a `Stop-list` line (see the `orchestrate` skill's "Escalation
comments") when a blocked, failed, or capped review result hit the
stop-list; read it the same way you read `Entries` on a hold comment.

Every ready review cycle posts a hold, `Entries: none` included (see
`adversarial-review`'s "Recording the stop-list hold"), so among holds
alone the latest is always the whole of the current truth: an `Entries:
none` hold becoming the newest marker clears an earlier hit as surely
as a later non-empty hold re-asserts it. That is exactly what row 2's
"newest marker" test relies on to decide whether a hold is still live,
and it's why an escalation or deferral note that has since become the
newest marker takes a ticket out of row 2 without row 2 needing to say
so — the row above or below it will already have matched first.

For an unlinked open PR, found the same way `adversarial-review` and
`merge-queue` post them — on the PR itself, since there's no ticket to
hold it — read the same table with three rows adjusted, since there's
no ticket label to read. Row 0's "pending" test already reads the PR's
head alone when there's no linked ticket (see the `orchestrate` skill's
"Escalation comments"). Its exception clause has no `needs-attention` to
read — there's no ticket to carry it — and otherwise reads, for an
unlinked PR: no GitHub review approving the PR, or its checks aren't
green at the current head — matching `merge-queue`'s Eligibility, which
for an unlinked PR asks only for an approving review and a green head —
except that when the PR's latest `factory: stop-list hold` comment
names at least one entry, that approval must also be newer than the
hold, the same "newer than" test rows 1 and 2 below apply to their own
markers. With no hold at all, or a hold naming no entries, any
approving review satisfies the guard.

- **Row 1** reads "newest marker is an escalation, and no GitHub review
  approving the PR is newer than it" in place of `needs-attention`.
- **Row 2** keeps its `Entries` test but reads "and no GitHub review
  approving the PR is newer than it" in place of "carries no
  `approved-to-merge`" — an approving review clears an unlinked PR's
  hold the same way `approved-to-merge` clears a ticketed one.
- **Row 4** becomes "any other open unlinked PR carrying a marker."

None of this turns on the PR's commit history: a plain push with no
marker, escalation, deferral, or approval of its own changes nothing
here.

## Ordering

Sort by **downstream unblock impact** — how much other work an answer
releases — not by when the item stopped. For each item count the
tickets blocked by it in Linear, the tickets serialized behind it in
the caller's wave plan (if the caller passed one — a standalone
invocation won't have one, and that term of the count is then zero),
and the open PRs that share files with it (`gh pr diff --name-only`,
never diff content — the same lightweight overlap check `orchestrate`'s
wave planning and `merge-queue`'s ordering use). Ten tickets waiting on
one ruling belongs at the top even if it stopped an hour ago; a leaf
that stopped first thing belongs at the bottom.

Break ties by the cost of being wrong: at equal impact, a stop-list
item outranks an ordinary one — that includes a row 1 item whose
escalation comment's `Stop-list` line is non-`none`, and a row 0
not-yet-eligible item that names a superseded hold's `Entries`, not
only a row 2 hold.

State the count in the item itself, so the ordering is visible rather
than asserted.

## Item shape

One per item, in this shape, and nothing longer:

    ### <TICKET> — <the question, as a question>

    **Unblocks:** <n tickets: ids> · **PR:** <url> · **Stopped at:**
    <planning | researching | implementing | review round n | merge>

    <One or two sentences on what was found. Not a narrative of the
    run.>

    - **A.** <option> — <consequence>
    - **B.** <option> — <consequence>

    **Recommended: <letter>.** <One sentence of why.>

`Stopped at` and the question both come straight from the escalation
comment: `Stopped at` is its `Stage` line, and the heading question is
built from its `Question` line — you are rendering what the comment
already says, not re-deriving it from the diff.

A stop-list hold item — row 2, ticketed or unlinked-PR form — has no
`Stage`, `Question`, or `Options` to read; the hold comment carries only
`PR`, `Entries`, and `Round`. Build it in this fixed shape instead,
every time: **Stopped at** is `ready, waiting on stop-list merge
approval`; the heading question is `Merge PR <n>? It touches:
<Entries>`; the options are always **A.** merge and **B.** don't merge
yet — leaves it queued here. Option A's consequence depends on which
form of row 2 matched: for a ticketed item it's "clears the hold and
lets it proceed" (adding `approved-to-merge` is what clears it); for an
unlinked PR it's "approve the PR on GitHub, which clears the hold and
lets it proceed" — an unlinked PR's hold clears on an approving GitHub
review, not a label. Never invent a `Stage` or extra options for this
kind of item; this fixed shape is the whole of what a hold comment
gives you.

Row 0's not-yet-eligible exception — a pending `factory: decision`
naming `merge-queue <PR>` that fails `merge-queue`'s Eligibility on its
own terms — is fixed-shaped, in whichever of the three forms it fails
(name more than one in the sentences below if more than one is true):

- **`needs-attention` still on.** **Stopped at** is `decided, but
  needs-attention is still on`; the heading question is `A decision says
  merge PR <n>, but needs-attention is still on: clear it?`; the options
  are always **A.** clear the label — lets `merge-queue` pick it up, and
  **B.** leave it on — leaves it queued here.
- **No merge approval.** **Stopped at** is `decided, waiting on merge
  approval`; the heading question is `A decision says merge PR <n>, but
  it has no merge approval: approve it?`; the options are always
  **A.** approve it — adding `approved-to-merge` (unlinked PR: an
  approving GitHub review) clears the guard and lets `merge-queue` pick
  it up, and **B.** don't approve yet — leaves it queued here.
- **Checks not green.** **Stopped at** is `decided, blocked on CI`; the
  heading question is `A decision says merge PR <n>, but its checks
  aren't green: what now?`; the options are always **A.** get CI green
  on the current head, then let the next `merge-queue <PR>` pick it up
  — leaves it queued here until then, and **B.** drop the decision — the
  human answers some other way instead.

Whichever form applies: if the ticket's latest `factory: stop-list
hold` comment names at least one entry, name its `Entries` in the one
or two sentences above the options too — "It touches: <Entries>", the
same line a row 2 item gives — this is exactly the hold this guard
exists to keep visible (see "Ordering" for how it's weighted). Never
invent a `Stage` or extra options for this kind of item either; the
decision comment's `Decision` and `Recorded` lines are context for the
one or two sentences above the options, not a source of more of them.

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

Three further kinds of line belong here too, since none of them is a
decision either:

- **Decided.** Row 0: a ticket or unlinked PR whose newest marker (see
  "What goes in the queue" above) is a *pending* `factory: decision`
  comment, and whose `Next` isn't the not-yet-eligible `merge-queue
  <PR>` case row 0 turns into a queue item instead (see the table and
  "Item shape"). One line: `<TICKET> — decided: <Decision, one line> →
  <Next>`, plus "`needs-attention` still on" when the label hasn't come
  off yet — true only for `implement-ticket` or `adversarial-review`
  `Next`, since a `merge-queue <PR>` decision with the label still on
  is the queue item above, not this line. It's not a queue item because
  the question it would ask has already been answered — the reason row
  0 sits above row 1 (see the paragraph just under the table) — so this
  line exists only to account for it in the report, the same reason the
  "awaiting merge approval" line below exists.
- **Deferred.** Row 3: a ticket or unlinked PR whose newest marker (see
  "What goes in the queue" above) is a `factory: deferred` comment. The
  deferral note itself names no restart invocation or PR number (see
  the `orchestrate` skill's "The deferral note"); derive one instead from
  the note's `Stage` line plus the ticket id or its linked PR:
  `implement` → `implement-ticket <TICKET>`, `review-fixer` or
  `trivial-minors` → `adversarial-review <PR>`, `merge-resolver` or
  `merge` → `merge-queue <PR>`. `merge-queue`'s stuck sub-case posts
  only the escalation, never a separate deferral note (see the
  `orchestrate` skill's "Escalation comments"), so it is never this line —
  it's row 1 for as long as `needs-attention` stays on. Once a restart
  posts anything at all, that new comment is the newest marker and a
  ticket stops matching row 3 on its own, whether or not the restart
  ever pushed a commit: a review that comes back ready without pushing
  still posts a fresh hold (`Entries: none` included — see
  `adversarial-review`'s "Recording the stop-list hold"), and that hold
  is what row 2 or row 4 next reads, moving the ticket to a queue item
  or the "awaiting merge approval" line below instead. A stopped restart
  posts a newer marker of its own, which is read the same way. A human
  recording a `factory: decision` after a deferral is read the same
  way too, in one clause: the decision is now the newest marker, so the
  ticket leaves row 3 for row 0 and this line for the Decided line
  above. A restart that succeeds without posting anything — a clean
  `implement-ticket` run or a `merge-queue` merge, neither of which
  leaves a marker behind — is what row 3's status-and-PR-head test is
  for: the deferral comment is still the newest marker, but the
  ticket's status or its PR's head has moved since it posted, so the
  ticket falls through to row 4 or row 5 instead of sitting here
  forever.
- **Implemented and awaiting review, still in review, awaiting merge
  approval, queued to merge, or research report awaiting acceptance.**
  A `research` ticket in `In Review` has no PR: its report is on the
  ticket and the human accepts it by moving it to `Done` (see the
  `orchestrate` skill's "Research tickets"), so it lands here as
  "research report awaiting acceptance", not as a queue item. Its
  escalations are row 1 items like any other. Row 4: a ticket that's `In Review`,
  or `In Progress` with an open PR, and didn't match row 0, 1, 2, or 3
  — whatever's left once those are ruled out: no marker at all
  (including when the only marker on the ticket is a stale decision), a
  stop-list hold naming no entries, an escalation whose ticket has
  already lost `needs-attention`, or `approved-to-merge` sitting on a
  green PR still waiting for `merge-queue` to pick it up. The
  `In Progress`-with-an-open-PR form is what a resumed `implement-ticket
  <TICKET>` decision leaves behind on a clean run — that stage never
  moves the ticket to `In Review` itself, so without this form it would
  fall through to row 5 and disappear. This is not a decision for the
  human either; it's here so the report accounts for every ticket a run
  touched, not only the ones stuck.

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
