---
name: merge-queue
description: Merge every eligible pull request — approved via GitHub review, or with its linked Linear ticket carrying the `approved-to-merge` label, and CI green — rebasing each onto the repo's base branch in an order chosen to minimize conflicts, resolving mechanical rebase conflicts itself, resolving a short enumerated list of one-step-beyond-mechanical conflicts under a scoped intent review, and surfacing anything needing new logic or a judgment call to the human instead of guessing. Reads the host repo's `AGENTS.md` `## Dispatch` section for its Linear team, base branch, write window, and worktree directory, and stops short of any rebase, `gh pr ready`, or `gh pr merge` while that window is closed. Use when asked to run the merge queue, merge everything that's approved, or merge eligible PRs. Called by the dispatch skill for its own batch's approved tickets; equally fine invoked standalone against any repo's open PRs.
---

# Merge queue

Given one or more targets — a PR, a ticket id, or the literal "all
eligible PRs" — merge each into the base branch, in an order chosen to
minimize conflicts, as fast as CI allows. You do not write code, read
a diff, or invent a resolution to a real conflict yourself: a resolver
subagent handles each rebase, and stops instead of guessing when a
conflict needs a decision it cannot make from the diff.

If invoked with no target, **ask** which PR(s) or ticket(s) to merge —
never default to scanning the repo. "All eligible PRs" is a valid
explicit target: see "Finding eligible PRs" below.

## Repo configuration

Read the `## Dispatch` section of the host repo's `AGENTS.md` first:
it gives the Linear team key (so you can recognize `<KEY>-<n>` ticket
ids in PR titles and branch names), the **base branch** everything
rebases onto and merges into, the **worktree directory**
(`<runs-dir>` below), the **stop-list** — the paths and subjects
whose merge always waits for a human — and the **write window**, the
weekday hours, if any, during which a rebase, `gh pr ready`, or
`gh pr merge` is not allowed to happen (see the `dispatch` skill's
"The write window"). No `## Dispatch` section means the repo has not
opted into this pipeline — say so and stop; do not assume a base
branch.

## Eligibility

Never merge anything without one of these:

- **No linked Linear ticket**: the PR carries at least one **approved**
  GitHub review.
- **A linked ticket** (from the title's `<TICKET>: ...` convention or
  the branch name): that ticket carries the **`approved-to-merge`** label
  — added there directly by a human, or by you once you've confirmed
  the human cleared it. Three things count as cleared: they said so in
  chat, they pre-authorized that ticket at a `dispatch` plan gate, or
  the caller tells you `dispatch` is running this batch with its
  **merge gate released** — the human authorized the batch's merges at
  invocation. Take that last one only from the calling skill, stating
  it plainly, and only for tickets whose review came back ready. Never
  infer it from a ticket's state, and never assume it because a run
  looks unattended.

The label is the approval signal, not the status: an approved ticket
stays in `In Review` until it merges. A ticket sitting in `In Review`
without the label is **not** approved — that status is a reading gate,
and treating it as consent would merge things nobody agreed to. A
ticket carrying `needs-attention` is never eligible, whatever else is
true of it: something stopped, and a human has not looked yet.

A PR that hits the repo's **stop-list** needs the human's clearance
for that PR specifically — said in chat, or the label added by them.
A released merge gate does not clear it, and neither does a plan-gate
pre-authorization given before anyone knew what the diff would touch:
the point of the list is that these merges are decided after somebody
has seen the change. If the caller tells you a ticket is a stop-list
hit, that direct clearance is the only route left to eligible.

Either way, CI must show green on the PR's current head before you
queue it. That's a sanity check, not the real gate — rebasing needs a
fresh CI run regardless of what the pre-rebase head showed.

## Finding eligible PRs

For "all eligible PRs": list open PRs (`gh pr list --state open`) and
keep the ones meeting the rule above. For an explicit ticket or PR
target from a caller, just confirm it's eligible — don't second-guess
a target you were already given.

## Ordering

Use the same lightweight file-overlap approach `dispatch`'s wave
planning uses — file lists per PR (`gh pr diff <number> --name-only`),
never diff content. Order:

1. Serial chains first, in dependency order (one PR plainly building
   on another).
2. Then whatever overlaps least with what's already merged this run.
   The base branch moves after every merge, so re-evaluate before
   picking the next PR — don't compute the whole order up front and
   stick to it blindly.

Trivial overlap (both touch a registration list, an import block) is
not a reason to serialize — that's exactly what the rebase absorbs.

## Per PR

Step 1 is a Linear write, not a gated git write, so it always runs
first, for every PR — a chat or plan-gate clearance must get its label
before anything else, or a closed window strands it unlabeled and the
next sweep can't find it. Check the write window only after step 1,
before starting step 2 — the rebase itself creates commits, so this
has to happen before step 2, not just before the push. If it's closed,
don't start step 2 for this PR. The window won't reopen partway
through the queue, so run step 1 for every PR still waiting (in case
one of them just got cleared) and then report all of them as
`deferred` in one pass rather than discovering it again on each one;
see "On a deferred report" below and the `dispatch` skill's "The write
window".

1. Add the `approved-to-merge` label to a ticket cleared any of the
   three ways above if it isn't already there (skip if the PR has no
   linked ticket).
   Leave the ticket's status alone; it stays `In Review` until it
   merges.
2. Set up the worktree at `<runs-dir>/<TICKET-or-PR#>` per the
   `dispatch` skill's "The one worktree rule" — reset it to the PR's
   current remote head if one's already there at that path (a caller
   may have left one from implementing or reviewing, an earlier
   resolver's deferral included), or create it fresh on the PR's branch
   if not. Either way this is a fresh attempt, not a continuation of
   whatever that worktree held.
3. Spawn a **fresh** resolver subagent with `prompts/resolver.md` —
   mid-tier model, high effort (see the `dispatch` skill's Models and
   effort table for the harness mapping) — filling in WORKTREE,
   BRANCH, BASE (the repo's base branch), PR, and WINDOW (the repo's
   write window). It rebases onto current `origin/<base>`, resolves
   the conflicts its two declared tiers cover, pushes, and watches CI
   under the same three-attempt rule as implementing, checking the
   window again before the rebase and before every push. It reports
   the highest **tier** it resolved at: `1` for purely mechanical, `2`
   for one of four enumerated cases a step beyond that, `none` if the
   rebase came out clean.
4. **`RESULT: green` with `TIER: 2`** → run the intent review before
   anything else. Spawn a fresh subagent with `prompts/intent-review.md`
   — strongest model available, high effort — giving it the PR, the
   worktree, and the resolved hunks the resolver named, with one
   question: did this resolution drop anyone's intent?

   A single pass over the resolved hunks, and no second round.
   `adversarial-review` already covered this branch's code quality;
   the only thing that has happened since is the rebase, so a finding
   here means the resolution is wrong, not that the code is.

   - **`RESULT: intact`** → carry on to step 5 and merge.
   - **`RESULT: dropped`** → treat it exactly like a blocked resolver:
     stop this PR, add `needs-attention` to the linked ticket (or, with
     no linked ticket, post on the PR instead), say which hunk and
     whose intent went missing, leave the PR and worktree as they are,
     move on to the next PR. In the same step as the label, post a
     `factory: escalation` comment (see the `dispatch` skill's
     "Escalation comments") with stage `merge`, result `dropped`, and
     the intent reviewer's `NOTES` and `OPTIONS` verbatim. Do not send
     the resolver back in for another try — the second attempt belongs
     to whoever answers the question.

   `TIER: 1` and `TIER: none` skip the intent review entirely and go
   straight to step 5. A mechanical resolution has nothing to
   second-guess, and paying a strong-model pass on every merge to
   insure against a class of mistake that tier is defined to exclude
   would slow the whole queue for nothing.

5. **`RESULT: green`**, intent review passed or not required → check
   the write window once more, immediately before merging — it can
   have closed since step 3 started, and `gh pr merge` is a gated
   write in its own right, not covered by the resolver's checks. If
   it's closed, stop here and report this PR `deferred` (see "On a
   deferred report"): the rebase went green, but nothing gets merged
   while the window is shut.

   Mark the PR ready and merge it:

       gh pr merge <number> --squash

   Don't use `gh pr merge`'s own `--delete-branch` flag: it tries to
   switch the local checkout to the base branch first, which fails
   whenever that branch is checked out in another worktree — which,
   in this pipeline, it always is.

   Check the window a third time before the branch delete below. If it
   closed after the merge went through but before you get here, the PR
   is already merged — don't undo that — just report it merged, note
   in NOTES that the remote branch was left in place, and skip only
   the `git push origin --delete` step below. Everything else in this
   step still happens: moving the ticket to `Done`, clearing
   `approved-to-merge`, and removing the worktree are local or Linear
   actions, not gated writes, so the window closing doesn't touch them.

   Otherwise, check whether the remote branch is still there, rather
   than assuming either way — some repos auto-delete a branch on merge
   and some don't:

       git ls-remote --heads origin <branch>

   If it still exists, delete it as a separate step:

       git push origin --delete <branch>

   The squash commit is all of the branch's content that survives, so
   a remote branch with nothing left to give is just clutter; but a
   `--delete` against a branch the repo already reaped just errors, so
   check first and say nothing more about it if it was already gone.

   Move the ticket to `Done` and remove its `approved-to-merge` label —
   the label described a queue the ticket has now left. A GitHub
   automation may race you to `Done`, which is harmless; the label is
   still yours to clear. Delete the worktree
   (`git worktree remove <path>`). Move to the next PR.
6. **`RESULT: blocked`** — the resolver hit a conflict whose
   resolution needed a judgment call it could not make from the diff:
   not tier 1, not one of tier 2's four cases, or it could not tell
   which. Its report carries the options it saw
   and its own pick; relay those, they are what makes this answerable.
   Stop this
   PR's merge, add the `needs-attention` label to the linked ticket
   (leaving its status where it is, or posting on the PR itself if none
   is linked), tell the human exactly what the resolver found (files,
   the nature of the conflict), and leave the
   PR and worktree as they are. In the same step as the label, post a
   `factory: escalation` comment (see the `dispatch` skill's
   "Escalation comments") with stage `merge`, result `blocked`, and the
   resolver's `OPTIONS` and `NOTES` verbatim. Move on to the next PR —
   one blocked merge never stalls the rest of the queue.
7. **`RESULT: failed`** — CI never went green on the rebased head after
   three attempts. Handle it like blocked for queue purposes (stop
   this one, label it — or post on the PR if none is linked — post the
   escalation comment with the resolver's `OPTIONS` and `NOTES`
   verbatim, tell the human, move on), but say which one it
   was — they read differently: blocked needs a decision, failed hit a
   mechanical wall.
8. **`RESULT: deferred`** — the resolver stopped before the rebase or a
   push because the write window closed. See "On a deferred report".

## On a deferred report

Whether it came from the window check at the top of this section, from
a resolver's `RESULT: deferred`, or from step 5's own check
immediately before merging, this is not blocked and not failed —
nothing about the PR is wrong, the clock is (except the one stuck
sub-case below, which is deferred *and* needs a human). Leave
the ticket's status and the `approved-to-merge` label exactly as they
are either way. Leave the PR and worktree as they stand and move to
the next PR, unless the whole queue was deferred at once because the
window was already closed when this PR's turn came, in which case
there is no next PR to move to.

For each deferred PR, record it in your report's NOTES, and, if a
ticket is linked, post the short deferral note the `dispatch` skill's
"The write window" defines (first line `**factory: deferred**`) —
stage `merge-resolver` for a deferral at
or before the rebase, or `merge` for one at step 5's final pre-merge
check — saying what's left undone, the worktree path, and when the
window next opens. Ordinarily there is no resume: a later call re-runs
this PR from step 2 above, which resets the worktree to the PR's
remote head before spawning a fresh resolver. The one exception is the
stuck sub-case below, where a later call can't get that far.

What decides stuck versus ordinary is the state of the head when the
window closes, not whether a push happened. A rebased head that's been
force-pushed and isn't (yet) green — red, or CI still running — is
stuck rather than merely deferred: say explicitly that the rebase was
pushed, and whether the head is red or its CI unverified. That head
cannot satisfy this skill's own Eligibility check on its own — CI must
show green before a PR is even queued — so no later call, targeted or
swept, can pick it back up by re-running the resolver; `merge-queue
<PR>` would reject it at Eligibility before the resolver ever ran.
Don't say a restart will fix it. Instead, add `needs-attention` to the
linked ticket (leaving its status where it is), or post on the PR
itself if none is linked, and, in that same step, post a `factory:
escalation` comment (see the `dispatch` skill's "Escalation comments")
with stage `merge`, result `blocked`, and `Found` explaining the stuck
head (pushed, and red, still running, or unverified) — this carries
`needs-attention` like any other labelled stop, so it gets the same
record. Say so in your report alongside the usual deferral note: a
human has to get the pushed head's CI green, or decide what to do with
it, before this PR can requeue at all.

Everything else is an ordinary deferral — including a rebased,
force-pushed head that's already green when the window closes, which
is exactly what step 5's final pre-merge check defers: the rebase
succeeded and CI passed, only `gh pr merge` itself got cut off. That
gets no label and is picked back up the normal way, by targeting the
PR directly. It's also not invisible to a sweep: it still carries
`approved-to-merge` and a green head, so the next "all eligible PRs"
sweep picks it up too and merges it once the window is open.

## Report

One line per PR as you go, plus a final table if you merged more than
one: PR, ticket, `RESULT` (merged | blocked | failed | deferred), the
resolver's `TIER` and whether an intent review ran, and one line of
why for anything not merged.

For anything not merged, carry the resolver's or intent reviewer's
options and pick through verbatim — they're also in the `factory:
escalation` comment you already posted (see the `dispatch` skill's
"Escalation comments"), which is what lets `decision-queue` render this
as a question somebody can answer instead of a conflict somebody has to
go read.

This skill never plans, implements, or reviews the change itself — it
only merges what's already eligible, and the one review it does run is
scoped to the rebase. See `implement-ticket` and `adversarial-review`
for the stages before this one, `decision-queue` for what happens to
the PRs it stops on, and `dispatch` for the full pipeline.
