---
name: adversarial-review
description: Run adversarial review rounds on one or more open pull requests — a fresh reviewer subagent each round, a fixer subagent addressing findings, capped at six rounds by default and stopped early when rounds stop making progress — and report which are ready to merge, meaning no major or medium findings left; a ready PR is marked ready for review (un-drafted) in the same step. Reviews against the host repo's own invariants, declared in its `AGENTS.md` `## Orchestrate` section, which also declares a write window that stops the fixer's commits and pushes, and this skill's own un-draft, short while it's closed. Labels and records a capped, blocked, or failed outcome, and a stop-list hold, on the linked ticket (or on the PR itself when none resolves). Use when asked to adversarially review a PR, run a review cycle, or review every open PR. Called by the orchestrate skill right after implement-ticket reports a ticket green; equally fine invoked standalone against any open PR, ticket-linked or not.
---

# Adversarial review

Given one or more targets — a PR, a ticket id, or the literal "all open
PRs" — run this cycle on each, independently: fresh reviewer subagent
→ if it finds anything that matters, fixer subagent → fresh reviewer
again, up to the round budget. You never read the diff, a test log, or
a CI log yourself; subagents do that and report back counts and one
line per finding.

If invoked with no target, **ask** which PR(s) or ticket(s) to review
— never default to scanning the repo. "All open PRs" is a valid
explicit target if the caller says it: expand it with
`gh pr list --state open` and run one independent cycle per PR.

## Round budget

**Six rounds per target** unless the caller says otherwise. That is a
hard cap: at six rounds with major or medium findings still open,
stop and report capped. Never run a seventh on your own judgment — a
human decides whether another cycle is warranted.

A caller running unattended can hand you a **discretionary budget**
instead — "up to ten rounds, your judgment", which is what the
`orchestrate` skill passes in semi and autonomous modes. Under a
discretionary budget the number is a ceiling, not a target.

Either way, the rule that ends a cycle early is the same one, and it
is not the round count.

## The progress rule

A round makes **progress** if it closes findings the round before it
raised, *and* whatever it raises is new ground. A round makes none if
it re-raises a finding an earlier round already raised, oscillates
between two shapes of the same fix, or trades a finding for an
equivalent new one.

**Stop after two consecutive rounds that make no progress**, whatever
budget you are under, and report capped with the ledger rows that show
it.

New findings on a later round are **not** by themselves a reason to
stop. A second or third reviewer reaching ground the first one didn't
is the cycle working as designed — it is most of what the adversarial
part buys, and a net-new major on round two is a catch, not a warning
sign. What is worth stopping for is a cycle that has stopped moving:
the same question, three shapes, three rounds running. Judge forward
progress on the ticket, not the arrival of findings that are new.

Two consecutive is deliberate. One round that circles is noise — a
reviewer landing on ground a previous one covered, a fixer taking the
long way to the same place. It takes a second one to tell a slow round
from a loop, and a cycle that ejects on the first is a cycle that
hands a human work it would have finished on its own.

## The ledger

You cannot apply the progress rule from memory, and the reviewers
cannot apply it at all: each one is deliberately fresh and knows
nothing of the rounds before it. That stays true — a reviewer told
what the last one found is no longer an independent read. The
comparison is yours to make, so keep the ledger yourself.

One row per finding: the round that raised it, its severity, its file,
its one-line failure scenario, and what the fixer did with it (fixed,
rebutted, or still open). Read it before every new round. A finding
whose file and failure scenario match a row already there is a
re-raise, whether or not the wording matches.

That comparison is the entire mechanism. Without the ledger,
"converging" is a feeling, and a cycle run on feelings stops either
too early or never.

Say in your report which budget you were running under and, whenever
you stop before the cap, why. A human reading "capped at 6 of 10,
rounds 4-6 kept re-raising the same nullability question, see ledger"
learns something; "capped" alone does not.

## Restarting a deferred cycle

There is no resume. A cycle that comes back `deferred` is picked up
later by a human re-invoking this skill directly against the PR —
`orchestrate` never restarts a deferred cycle itself — and it starts over
at round 1 with a fresh round count and an empty ledger, the same as
any other call. The worktree reset this needs is "The worktree" rule
below, applied at whichever step first touches the worktree this time:
whatever the deferred fixer left uncommitted or unpushed there is
stale, and nothing about this call knows what it still covers.
Restarting a cycle
that had already used some of its round budget does spend that budget
again; that is the accepted cost of stopping dead at a window boundary
rather than carrying state across a context nothing keeps alive. See
the `orchestrate` skill's "The write window".

## Repo configuration

Read the `## Orchestrate` section of the host repo's `AGENTS.md` before
starting. Four fields matter most here:

- **Review invariants** — the properties this repo wants a reviewer to
  be adversarial about. This is the substance of the review; the
  reviewer prompt deliberately carries none of its own, because what
  is worth being hostile about is different in every repo. Quote the
  invariants into the reviewer's brief rather than paraphrasing them.
- **Check command** — what the fixer must pass locally before pushing.
- **Stop-list** — the paths and subjects whose merge always waits for
  a human. Every round's reviewer checks the diff against it and
  reports what it hits. This is not a review finding and says nothing
  about the change's quality; it routes the merge decision, and it is
  the check that catches a diff reaching files the plan never
  predicted. On a ready result, every cycle posts a `factory:
  stop-list hold` comment, `Entries: none` included (see "Recording
  the stop-list hold" below); on a blocked, failed, or capped result
  it goes on the `factory: escalation` comment's `Stop-list` line
  instead (see the `orchestrate` skill's "Escalation comments"); on a
  deferred result it goes on the deferral note's `Stop-list` line (see
  "On a deferred report" below). Neither a hold nor a deferral note is
  a label.
- **Write window** — the weekday hours, if any, during which a commit
  or push is not allowed to happen. Pass it down to the fixer. It also
  governs the one gated write this skill makes itself: the `gh pr
  ready` on a ready result (see "When a target is ready" below). See
  the `orchestrate` skill's "The write window".

Also take the base branch and the worktree directory (`<runs-dir>`)
from that section. No `## Orchestrate` section means the repo has not
opted in: say so and stop. If the invariants are missing, you can
still review against the ticket brief and general correctness, but say
plainly in your report that this repo declared no invariants, so the
review was thinner than it should have been — don't paper over it.

## The worktree

Whichever step is the first in a cycle to touch
`<runs-dir>/<TICKET-or-PR#>` — step 5's fixer on a major or medium
finding, or the "one inline pass" at trivial minors in "When a target
is ready" if the cycle reaches ready without step 5 ever running —
applies the `orchestrate` skill's "The one worktree rule" before making
its first edit there: reset a worktree already at that path (a caller
may have left one from implementing, or an earlier, now-stale attempt
at reviewing this same PR, ticketed or not, deferred or otherwise), or
create one fresh on the PR's branch if none exists. A later round or
pass within the same cycle keeps building in that same worktree
without resetting it again — it holds this cycle's own accumulating
work, not a leftover from somewhere else.

## Resolving a target to a brief

A reviewer needs the ticket's brief — its description, `## Plan`
section if present — to check the PR against. Given a ticket id, find
its open PR. Given a PR, find its linked ticket from the title's
`<TICKET>: ...` convention or the branch name. If a PR has no
resolvable ticket (a human's PR with no Linear link), review it
against just this repo's own documents — `AGENTS.md`, its
`## Orchestrate` invariants, and whatever convention documents it points
to — and say in your report that there was no ticket brief to check
against.

## Cycle, per target

1. Move the resolved ticket to `In Review`, if one resolved.
2. Spawn a **fresh** reviewer subagent with `prompts/reviewer.md` —
   strongest model available, high effort, every round, not just later
   ones (see the `orchestrate` skill's Models and effort table for the
   harness mapping) — filling in TICKET (or "none"), PR, ROUND, the
   repo's declared review invariants, and its stop-list. It reads the
   diff with hostile eyes, posts findings as PR review comments rated
   major/medium/minor, and returns counts, one line per finding, and
   the stop-list entries the diff hits.

   Write every finding it returns into the ledger before you do
   anything else with the round. That is what the next round's
   progress judgment reads.
3. **Reviewer reports blocked** — it found something that isn't a
   diff-level finding (the PR doesn't do what the brief describes, a
   diverged base, a serious pre-existing bug) — stop the cycle for this
   target right there. Don't spawn a fixer; there's nothing for it to
   fix. Go to "On a blocked or failed report" below.
4. **Zero major and zero medium findings** → before calling this round
   clean, check CI on the PR's current head (`gh pr checks`). Green:
   this target is ready, whatever minors are open — see "When a target
   is ready" below. Red: the round isn't clean — treat "CI red on
   head" as a medium finding and fold it into step 5 for the fixer.
   Pending: wait for it with `gh pr checks --watch`, then judge
   whatever it settles on. No checks reported at all: that is not
   green either — report `RESULT: blocked` with that as the reason and
   go to "On a blocked or failed report" below, same as a blocked
   reviewer report; don't call this round ready on the strength of an
   absent signal. This is a call you're making yourself, not something
   either subagent reported — see "On a blocked or failed report" for
   what that means for the escalation's `Options` line.
5. **Any major or medium finding** → spawn a fixer subagent with
   `prompts/fixer.md` in a worktree on the PR's branch. Apply "The
   worktree" rule above if this is the first thing in the cycle to
   touch it. On a later round within this same cycle, keep building in
   that same worktree without resetting it again — it holds this
   cycle's own accumulating work, not a leftover from somewhere else.
   Give it the repo's check command and write window (WINDOW) along
   with the findings — all of them, minors included, since a minor
   next to a major it is already editing around is cheap to take. The
   fixer addresses every finding, or rebuts one in the PR thread with a
   concrete reason and flags it in its report as disputed. It gets CI
   green again under the implementer's same three-attempt rule.

   Record what it did with each finding in the ledger — fixed,
   rebutted, still open — then run the next round with another fresh
   reviewer, and apply the progress rule to what that round comes back
   with.

   If the fixer instead reports **blocked** — a finding can't be
   addressed without a judgment call outside its brief — stop the cycle
   here too, same as a blocked reviewer report.

   If the fixer reports **deferred** — it stopped before a commit or
   push because the write window was closed — stop this target's cycle
   right there too, but treat it as neither blocked nor a failure: see
   "On a deferred report" below. Keep the ledger rows built so far in
   your report's NOTES, since the ledger itself doesn't survive past
   this context.

When the budget runs out with major or medium findings still open, or
the progress rule stops you first, leave the PR as it stands. If a
ticket resolved, label it `needs-attention`, leaving it in `In Review`,
and post a `factory: escalation` comment (see the `orchestrate` skill's
"Escalation comments") with stage `review round <n> of <budget>`,
result `capped`, the latest round's `STOPLIST` on its `Stop-list` line,
the ledger rows behind your read, and 2-3 options of your own — no
subagent has one to give here, so write them yourself:
for example, rule on the circling question and restart (fresh budget),
grant more rounds, or rescope, with your pick. If no ticket resolves,
post that comment on the PR instead. Either way, report it to whoever is
waiting: the findings summary, what each round fixed, and the ledger
rows behind your read. `capped` is never a merge signal — a capped
target waits for a human whatever mode the caller is in.

A clean round is a good round. Zero findings is an acceptable and
expected result for a small, correct change, not a sign the review was
skipped — and a round that finds only minors is a finished change, not
an unfinished review.

## When a target is ready

The exit condition is **no major and no medium findings**. It is not
zero findings. Do not chase zero: a competent adversarial reviewer
will find a minor on any diff of any size, so a cycle that exits only
at zero exits when the reviewer runs out of energy rather than when
the work is done — and it sends finished changes to a human over a
naming quibble, which is the one resource this whole pipeline is
built to spend carefully.

Open minors do not block ready. They stay as PR review comments, they
are listed in your report, and they are somebody's judgment call
later. They are not a reason to hold a correct change.

Before declaring ready you may take **one inline pass** at minors that
are trivially fixable — a wrong error string, a missed nil check, a
name that contradicts the one three lines above it. Apply "The
worktree" rule above first if step 5 never ran this cycle — a cycle
that reaches ready at round 1, or one restarted after a deferred
fixer, has never had this pass reset a worktree that may be stale or
missing. Then spawn the fixer in trivial-minors mode, let it push, and
confirm CI is green. Skip this pass entirely while the write window is
closed — a fixer spawned into a closed window would just come back
`deferred` having pushed nothing.

If it comes back `RESULT: deferred` instead — the window closed while
it was working — reset the worktree to the PR's remote head before
doing anything else: `git fetch origin && git reset --hard
origin/<branch>` in WORKTREE. That drops either kind of leftover the
same way — uncommitted edits, or a local commit the pass made but
didn't get to push — and leaves nothing for `merge-queue`'s rebase to
trip on.

The PR's current head, after that reset, decides the verdict:

- **CI is green there** (the pass either never pushed, so the head is
  unchanged, or it pushed and CI has already come back clean). Report
  `ready` with any minors the pass had touched left open — the pass was
  optional, and an unreviewed trivial edit that didn't make it costs
  nothing to drop.
- **CI is red, still running, or you can't tell.** Report this cycle
  `deferred`, not `ready` — `RESULT: ready` means CI green on the
  current head, not just findings resolved. See "On a deferred report".

**That pass is not a round and is not reviewed again.** Nothing about
it changes the verdict. A fix that would need a review round to be
trusted was not trivial, and the right move on one of those is to
leave the minor open and say so — which is exactly what the fixer is
told to do when it finds one. The pass is optional; skipping it costs
nothing.

Once the verdict is `ready` — after any trivial-minors pass above, and
after its reset-and-check-CI handling if that pass deferred — mark the
PR ready for review, immediately before posting the stop-list hold
(see "Recording the stop-list hold" below): check the write window per
the `orchestrate` skill's "The write window" (verify the zone once,
read the clock fresh — never reuse a reading from earlier in the
cycle). If it's open, check `gh pr view <PR> --json isDraft -q
.isDraft`; if that's `true`, run `gh pr ready <PR>`. If the window is
closed, skip the un-draft and still report `ready` — do **not** report
`deferred`: the review itself is finished and the hold must still be
posted; `merge-queue` un-drafts the PR at merge time instead. Say which
happened in the report's NOTES — `PR marked ready for review`, `PR was
already ready for review`, or `PR left draft: write window closed`. If
`gh pr ready` itself errors for another reason, likewise report
`ready` with the error in NOTES — it is not an escalation, and
`merge-queue` retries the un-draft when it gets there. This applies to
every target that ends ready, ticketed or not.

## Recording the stop-list hold

Once this cycle ends **ready** (see "When a target is ready" above),
post a `factory: stop-list hold` comment (see the `orchestrate` skill's
"Escalation comments") on the resolved ticket, or on the PR if none
resolved — **every ready cycle posts one**, whatever the target's final
`STOPLIST` line, from whichever round's reviewer ran last, says:
`Entries: <the entries it hit>` when non-empty, `Entries: none` when
it's empty. The latest hold is the entire current truth: an `Entries:
none` hold clears whatever an earlier hold recorded as surely as a
later non-empty hold re-asserts it, so post it every time, even when it
says the same thing an earlier hold already said. Post it last, as the
last act of the ready cycle — after any trivial-minors pass, if one
ran — but still built from the last review round's `STOPLIST`: a
trivial-minors pass only edits lines the reviewed diff already
changed (see "Trivial-minors mode" in `fixer.md`), so it can't reach a
new stop-list entry for that `STOPLIST` to have missed. Never add
`needs-attention` for this: a stop-list hold is not an escalation (see
the `orchestrate` skill's "The stop-list"), and this comment is the only
durable trace of it.

## On a blocked or failed report

Blocked, ordinarily, means a subagent (reviewer or fixer) found
something about the target that doesn't match what the review expected
and needs a human's judgment call before this cycle can mean anything —
the mismatch is already commented on the PR, with the options the
subagent saw and the one it would pick. Relay those with the mismatch;
they are what makes this answerable by someone who has not read the
diff. The one exception is step 4's no-checks-reported case: that
`blocked` is your own call, not a subagent's, since neither the
reviewer nor the fixer reported it — write 2-3 options yourself the
same way a capped result's are written (see the `orchestrate` skill's
escalation template), rather than inventing subagent options that were
never given or reaching for "none recorded", which is for a failed
result's mechanical wall, not this. Failed means the fixer never got CI
green after three honest attempts — a mechanical wall. Don't collapse
blocked and failed when you relay this: say which one it was and why.
Either way, stop this target's cycle, add the `needs-attention` label
to the resolved ticket if there is one (leaving its status where it
is), post a `factory: escalation` comment (see the `orchestrate` skill's
"Escalation comments") with stage `review round <n> of <budget>`,
result `blocked` or `failed`, this round's `STOPLIST` on its
`Stop-list` line, and the reviewer's or fixer's `OPTIONS` verbatim, or
your own written options for the no-checks-reported case (on the PR
itself if no ticket resolved), and don't let it block review of the
others you were given.

## On a deferred report

The fixer stopped before a commit or push because the repo's declared
write window was closed (see the `orchestrate` skill's "The write
window"). This is not blocked and not failed — nothing about the
target is wrong, the clock is. Stop this target's cycle, same as a
blocked or failed report, but don't add `needs-attention` and don't
touch the ticket's status. If a ticket resolved, post the short
deferral note the `orchestrate` skill's "The write window" defines (first
line `**factory: deferred**`), on the
ticket — the same rule `implement-ticket` follows on its own deferred
report. This is a stage-status note about the cycle pausing, not a
review finding about the diff, so it belongs where the ticket's status
already lives, not as a comment or thread reply on the PR. Post it with
stage `review-fixer` (or `trivial-minors`, for a deferral from that
pass): what's left undone, the worktree path, when the window next
opens, and a `Stop-list` line carrying the latest round's `STOPLIST`
(or `none`) — a deferred cycle never reaches the ready result that
would otherwise post a hold, so this line is the only durable record of
a hit that stalled here. For a ticketless PR
there is no private place to put that: skip the comment and rely on the
report below. Tell whoever is waiting on this the same thing either
way. There is no resume: a later call restarts this cycle from round 1
(see "Restarting a deferred cycle").

## Report, per target

    TICKET: <id or none>
    PR: <url>
    RESULT: ready | blocked | capped | failed | deferred
    ROUNDS: <n> of <budget, as used so far>
    LAST_ROUND_FINDINGS: <count by severity, or 0 — omit if blocked>
    OPEN_MINORS: <count, one line each, or none>
    STOPLIST: <the entries this diff hits, or none>
    OPTIONS: <for blocked, ordinarily the reviewer's or fixer's own 2-3
             options with consequences and their pick; for the
             no-checks-reported blocked case and for capped, the same
             shape but yours to write, since no subagent has one — e.g.
             rule on the circling question and restart (fresh budget),
             grant N more rounds, or rescope, with your pick>
    NOTES: <disputed findings, why capped, failed, or deferred, why you
           stopped where you did; for capped or deferred, the ledger
           rows behind that read>

`RESULT: ready` means no major or medium findings on the latest round
and CI green — it is not a merge, and it is not merge approval. It also
means the PR was marked ready for review (un-drafted), unless NOTES
says it was left draft — the write window was closed, or `gh pr ready`
itself errored; either way, un-drafting is a signal to readers that
review has finished, not merge approval, and merge eligibility is
unchanged (see `merge-queue`'s "Eligibility"). This skill never merges
anything; a human (directly, or via the `orchestrate` skill's merge
queue) still approves that separately, and for a ticketed PR, approval
is what puts the `approved-to-merge` label on the ticket. Never add
that label yourself.

A non-empty `STOPLIST` line means the caller holds this ticket's merge
gate for a human however the run is configured — including a run whose
merge gate was released at invocation. Report it whatever the result
was, including on a clean first round: it is the diff that decides
this, not the review.
