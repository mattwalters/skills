# The factory, in brief

This is orientation for a session that is about to talk about factory runs:
the vocabulary, not the rulebook. Reading it asks for no action. After
reading it, wait for what the human asks next. It is not the source of truth
for any rule. Each skill's `SKILL.md`, linked below, is, and where this file
and a `SKILL.md` disagree, the `SKILL.md` wins. The factory is a set of
skills in the `factory` plugin. You can read the
[repository README](https://github.com/mattwalters/skills#readme) for how it
is installed.

## The five skills

`orchestrate` is the policy layer. It picks tickets from Linear, plans them,
orders them into waves, holds or releases the gates, and tracks the batch to
merged pull requests. See
[orchestrate](skills/orchestrate/SKILL.md).

Three skills are the per-ticket stages, and each is also usable on its own.
[implement-ticket](skills/implement-ticket/SKILL.md) takes one ticket to a
CI-green draft pull request.
[adversarial-review](skills/adversarial-review/SKILL.md) runs fresh-reviewer
rounds on a pull request until nothing major or medium is left, or until it
gives up. [merge-queue](skills/merge-queue/SKILL.md) merges the pull requests
that are approved and green.

[decision-queue](skills/decision-queue/SKILL.md) is a reader. It looks at
whatever a run stopped on and turns each item into a question with options
and a recommendation. It keeps no state of its own, so it works mid-run,
afterwards, or against a repo nothing is running in.

[new-project](skills/new-project/SKILL.md) sits outside the pipeline. It is
the interactive setup skill for a brand-new repo and team, and it needs a
human at a terminal.

In Claude Code the skills are invoked namespaced, as in
`/factory:orchestrate`.

## Gates and modes

A run stops for a human at up to three gates. The selection gate decides
which tickets make up the batch. The plan gate covers the plans and the wave
order, before any code is written. The merge gate is per ticket, before it
merges. All three are held by default, and only a human releases one. The
ordinary way is to name a mode when invoking the run.

Supervised holds all three gates, and it is the default when no mode is
named. Semi releases selection and holds plan and merge, so the run picks the
batch and says what it picked, the human approves the plans, review and
fixing run unattended, and each merge waits. Autonomous releases all three.
A human can also spell out a combination of gates that no mode names, and an
adjustment spoken with a mode, such as autonomous but hold merge, wins over
the mode. A word at invocation that is neither a mode nor a gate spelled out
is not a mode, and the run goes supervised and says so.

A released gate is not a skipped decision. The run makes the call itself and
states it in a line. The full treatment is in the "Gates" and "Modes"
sections of [orchestrate](skills/orchestrate/SKILL.md).

## Per-ticket holds

A ticket can carry the `hold-plan` or `hold-merge` label. Each holds that one
gate for that one ticket, whatever the mode, including a run with no gates.
They are the human's labels: the pipeline never adds or removes them. A
`hold-merge` ticket is cleared to merge when the human adds
`approved-to-merge`. Other tickets in the batch are unaffected. See
"Per-ticket holds" in [orchestrate](skills/orchestrate/SKILL.md).

## The stop-list and escalations

Each adopting repo declares a stop-list: the paths and subjects whose merge
always waits for a human. A ticket on it is not blocked. It is planned,
implemented and reviewed like any other, and only its merge gate stays held,
even in autonomous mode.

An escalation is a different thing. It is reactive: a ticket came back
blocked, failed or capped because something stalled. A capped review never
merges, and a blocked or failed ticket is never cleared under a released
gate. Every blocked, failed or capped stop, and every stop-list hold, lands on
the Linear ticket, or on the pull request if no ticket resolves, as a
`factory: escalation` or `factory: stop-list hold` comment, a label, or both.

A closed write window is the quiet exception, and ordinarily not an escalation.
The stage stops dead and reports deferred. On a ticket it leaves one
`factory: deferred` comment, no label, and the status unchanged, so a ticket
can sit in `In Review` overnight looking untouched. On a pull request with no
ticket it posts nothing, and the stage's own report is the only record. A
human restarts a deferred stage by hand. The one exception is the merge
queue's stuck head: if the window closes after a rebased head was pushed but
before its checks are green, that is an escalation, with the
`needs-attention` label and a `factory: escalation` comment, on the pull
request if there is no ticket. Re-running the merge queue will not help,
since it refuses a head that is not green. See "The stop-list", "Escalation
comments" and "The write window" in
[orchestrate](skills/orchestrate/SKILL.md).

## Statuses and labels

Status is readiness. `Todo` is the only queue orchestrate draws new picks
from. `In Progress` means a ticket is being implemented. `In Review` means a
pull request exists and is waiting on review or on its merge gate. `Done`
means merged.

`Backlog` is the human's parking lot. Orchestrate never picks from it and
never promotes out of it, though at a held selection gate it may propose
promotions. Only a human moves a ticket from `Backlog` to `Todo`.

The `needs-attention` label means a human is needed. The ticket keeps
whatever status it had, and the why is in its escalation comment. The
`approved-to-merge` label clears a ticket for the merge queue, without
changing its status. The two hold labels are described above. See "Statuses
and labels" in [orchestrate](skills/orchestrate/SKILL.md), and the
"`Backlog`" part of its Phase 1.

## Adopting the factory

A repo opts in with a `## Orchestrate` section in its own `AGENTS.md`. The
section declares seven things: the Linear team key, the check command, the
base branch, the worktrees directory, the review invariants, the stop-list,
and the write window. The write window is required, and `none` means there
is none. The check command is fixed: it is always `./scripts/check.sh`, a
script the repo commits that runs whatever its checks actually are, and a
section that declares anything else counts as not filled in. Reviewers bring no invariants of their own, so the review
invariants are what a review is adversarial about in that repo.

If the section is missing, every skill except `new-project`, which creates
it, stops and says so rather than guessing. If a single field is missing, the run stops before the point where
it is needed. The worked example is the `AGENTS.md` of whichever repo the
session is in. Nothing project-specific lives in the skills. See "Repo
configuration" and "The write window" in
[orchestrate](skills/orchestrate/SKILL.md).

## Worktrees

Each ticket gets one detached worktree, named for the ticket, in the
directory the repo declares. That directory is recommended to be outside the
repo, so no ancestor `AGENTS.md` or `CLAUDE.md` loads into a ticket's run,
and namespaced per repo, so two repos don't collide. A run uses the
installed plugin, never the files in the worktree it is editing.
