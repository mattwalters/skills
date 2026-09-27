---
name: adversarial-review
description: Run adversarial review rounds on one or more open pull requests — a fresh reviewer subagent each round, a fixer subagent addressing findings, capped at six rounds by default and stopped early when rounds stop making progress — and report which are ready to merge, meaning no major or medium findings left. Reviews against the host repo's own invariants, declared in its `AGENTS.md` `## Dispatch` section, which also declares a write window that stops the fixer short of any commit or push while it's closed. Use when asked to adversarially review a PR, run a review cycle, or review every open PR. Called by the dispatch skill right after implement-ticket reports a ticket green; equally fine invoked standalone against any open PR, ticket-linked or not.
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
`dispatch` skill passes in semi and autonomous modes. Under a
discretionary budget the number is a ceiling, not a target.

A cycle resumed after a `deferred` report continues against the same
budget it started with — the rounds it already ran still count toward
the cap. Resuming never resets it.

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

## Repo configuration

Read the `## Dispatch` section of the host repo's `AGENTS.md` before
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
  predicted.
- **Write window** — the weekday hours, if any, during which a commit
  or push is not allowed to happen. Pass it down to the fixer. See the
  `dispatch` skill's "The write window".

Also take the base branch and the worktree directory (`<runs-dir>`)
from that section. No `## Dispatch` section means the repo has not
opted in: say so and stop. If the invariants are missing, you can
still review against the ticket brief and general correctness, but say
plainly in your report that this repo declared no invariants, so the
review was thinner than it should have been — don't paper over it.

## Resolving a target to a brief

A reviewer needs the ticket's brief — its description, `## Plan`
section if present — to check the PR against. Given a ticket id, find
its open PR. Given a PR, find its linked ticket from the title's
`<TICKET>: ...` convention or the branch name. If a PR has no
resolvable ticket (a human's PR with no Linear link), review it
against just this repo's own documents — `AGENTS.md`, its
`## Dispatch` invariants, and whatever convention documents it points
to — and say in your report that there was no ticket brief to check
against.

## Cycle, per target

1. Move the resolved ticket to `In Review`, if one resolved.
2. Spawn a **fresh** reviewer subagent with `prompts/reviewer.md` —
   strongest model available, high effort, every round, not just later
   ones (see the `dispatch` skill's Models and effort table for the
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
4. **Zero major and zero medium findings** → this target is ready,
   whatever minors are open. See "When a target is ready" below.
5. **Any major or medium finding** → spawn a fixer subagent with
   `prompts/fixer.md` in a worktree on the PR's branch — reuse one at
   `<runs-dir>/<TICKET>`
   if it already exists (a caller may have left one from implementing),
   otherwise make one:
   `git fetch origin && git worktree add <runs-dir>/<TICKET-or-PR#> origin/<branch>`.
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
the progress rule stops you first, leave the PR as it stands and
report it to whoever is waiting: the findings summary, what each round
fixed, and the ledger rows behind your read. `capped` is never a merge
signal — a capped target waits for a human whatever mode the caller is
in.

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
name that contradicts the one three lines above it. Spawn the fixer in
trivial-minors mode, let it push, and confirm CI is green. Skip this
pass entirely while the write window is closed — a fixer spawned into
a closed window would just come back `deferred` having pushed nothing.

**That pass is not a round and is not reviewed again.** Nothing about
it changes the verdict. A fix that would need a review round to be
trusted was not trivial, and the right move on one of those is to
leave the minor open and say so — which is exactly what the fixer is
told to do when it finds one. The pass is optional; skipping it costs
nothing.

## On a blocked or failed report

Blocked (from either subagent) means something about the target
doesn't match what the review expected and needs a human's judgment
call before this cycle can mean anything — the mismatch is already
commented on the PR, with the options the subagent saw and the one it
would pick. Relay those with the mismatch; they are what makes this
answerable by someone who has not read the diff. Failed means the
fixer never got CI green after three honest attempts — a mechanical
wall. Don't collapse the two when you relay this: say which one it was
and why. Either way, stop this
target's cycle, add the `needs-attention` label to the resolved ticket
if there is one (leaving its status where it is), and don't let it
block review of the others you were given.

## On a deferred report

The fixer stopped before a commit or push because the repo's declared
write window was closed (see the `dispatch` skill's "The write
window"). This is not blocked and not failed — nothing about the
target is wrong, the clock is. Stop this target's cycle, same as a
blocked or failed report, but don't add `needs-attention` and don't
touch the ticket's status. Post a comment on the PR — not thread
replies on individual findings, which would describe fixes nobody can
see yet — saying which round deferred, what's left undone, the
worktree path, when the window next opens, and the invocation that
resumes it. Tell whoever is waiting on this the same thing, along with
the ledger rows so far.

## Report, per target

    TICKET: <id or none>
    PR: <url>
    RESULT: ready | blocked | capped | failed | deferred
    ROUNDS: <n> of <budget, as used so far>
    LAST_ROUND_FINDINGS: <count by severity, or 0 — omit if blocked>
    OPEN_MINORS: <count, one line each, or none>
    STOPLIST: <the entries this diff hits, or none>
    OPTIONS: <for blocked: 2-3 options with consequences, and the pick>
    NOTES: <disputed findings, why capped, failed, or deferred, why you
           stopped where you did; for capped or deferred, the ledger
           rows behind that read>

`RESULT: ready` means no major or medium findings on the latest round
and CI green — it is not a merge, and it is not merge approval. This
skill never merges anything; a human (directly, or via the `dispatch`
skill's
merge queue) still approves that separately, and approval is what puts
the `approved-to-merge` label on the ticket. Never add that label
yourself.

A non-empty `STOPLIST` line means the caller holds this ticket's merge
gate for a human however the run is configured — including a run whose
merge gate was released at invocation. Report it whatever the result
was, including on a clean first round: it is the diff that decides
this, not the review.
