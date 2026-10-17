# The factory, in brief

This is orientation for a session that is about to talk about factory runs:
the vocabulary, not the rulebook. Reading it asks for no action. After
reading it, wait for what the human asks next. It is not the source of truth
for any rule. Each skill's `SKILL.md`, linked below, is, and where this file
and a `SKILL.md` disagree, the `SKILL.md` wins. The factory is a set of
skills in the `factory` plugin. You can read the
[repository README](https://github.com/mattwalters/skills#readme) for how it
is installed.

## The skills

`orchestrate` is the policy layer. It takes tickets from Linear, plans them,
orders them into waves, holds the plan and merge gates, and tracks the batch
to merged pull requests. See
[orchestrate](skills/orchestrate/SKILL.md).

Five skills are the per-ticket stages, and each is also usable on its own.
[plan-ticket](skills/plan-ticket/SKILL.md) writes a ticket's `## Plan` into its
description. [research-ticket](skills/research-ticket/SKILL.md) answers a
planned `research` ticket with a `## Report`.
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

## Gates

A run stops for a human at two gates. The plan gate covers the plans and the
wave order, before any code is written. The merge gate is per ticket, before
it merges. Both are always held, and only the human clears one. There are no
modes: nothing said at invocation releases either gate, and a word that tries
to is not honoured. A run takes the tickets the human names, or else every
eligible `Todo` ticket, with no cap and no selection step. The full treatment
is in the "Gates" section and Phase 1 of
[orchestrate](skills/orchestrate/SKILL.md).

## Per-ticket holds

The older `hold-plan` and `hold-merge` labels add nothing now, since both
gates are always held, and the pipeline still never adds or removes them.

## Research tickets

A ticket carrying the `research` label has a report, not a pull request, as
its deliverable. It is taken and planned like any other. After the plan gate
`research-ticket` has a single researcher subagent answer it and write a `## Report` section into
the ticket's description. There is no review and no merge, and the ticket
rests in `In Review` until the human accepts the report by moving it to
`Done`. The human owns the label, and `approved-to-merge` does nothing
for such a ticket. When a
researcher reports `blocked` or `failed`, `research-ticket` (not the
researcher) moves the ticket back to `Todo`, labels it `needs-attention` and posts an
escalation with stage `researching`; clearing the label makes it eligible
again. See "Research tickets" in
[orchestrate](skills/orchestrate/SKILL.md).

## The stop-list and escalations

Each adopting repo declares a stop-list: the paths and subjects whose merge
always waits for a human. A ticket on it is not blocked. It is planned,
implemented and reviewed like any other, and only its merge is different: it
waits for the human to clear that pull request specifically, after seeing what
the diff touched.

An escalation is a different thing. It is reactive: a ticket came back
blocked, failed or capped because something stalled. A capped review never
merges, and a blocked or failed ticket is never cleared. Every blocked,
failed or capped stop, and every stop-list hold, lands on the Linear ticket,
or on the pull request if no ticket resolves, as a `factory: escalation` or
`factory: stop-list hold` comment, a label, or both.

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

Status is readiness. `Todo` is the only status orchestrate takes tickets
from. `In Progress` means a ticket is being implemented, or researched. `In Review` means a
pull request exists and is waiting on review or on its merge gate; for a
research ticket it means the report is written and awaiting acceptance. `Done`
means merged, or for a research ticket, accepted by the human.

`Backlog` is the human's parking lot. Orchestrate never takes from it,
never proposes promotions out of it, and never promotes out of it. Only a
human moves a ticket from `Backlog` to `Todo`.

The `needs-attention` label means a human is needed. The ticket keeps
whatever status it had, and the why is in its escalation comment. The
`approved-to-merge` label clears a ticket for the merge queue, without
changing its status. The `human-only` label marks a ticket as the human's
own: the plan, research and implement stages skip it, and neither orchestrate
nor a dispatcher takes or fires one. The `plan-ready` label marks a written
plan waiting for approval: the plan stage adds it, and the stage that starts on
the approved plan removes it, so a saved Linear view can list the waiting
plans. The two hold labels are described
above. See "Statuses and labels" and Phase 1 in
[orchestrate](skills/orchestrate/SKILL.md).

## Dispatch

A dispatcher is the alternative to a standing `orchestrate` run. It is
built outside this plugin. It reads each ticket's state and fires the
one stage that state calls for, with no batch and no selection, and
nothing at all when no row matches. Its plan gate is the human adding
the `plan-approved` label to a planned ticket. Its merge gate is
`approved-to-merge`. Both gates are always held, as in orchestrate. See
[DISPATCH.md](DISPATCH.md).

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
it, stops and says so rather than guessing. If a single required field is missing, the run stops before the point where
it is needed. One more field is optional: `Telemetry`, the path of a poster
command that records each stage attempt's outcome and each review finding's
severity. Absent means off, and a failing poster never stops or fails a run. The worked example is the `AGENTS.md` of whichever repo the
session is in. Nothing project-specific lives in the skills. See "Repo
configuration" and "The write window" in
[orchestrate](skills/orchestrate/SKILL.md).

## Worktrees

Each ticket gets one detached worktree, named for the ticket, in the
directory the repo declares. That directory is recommended to be outside the
repo, so no ancestor `AGENTS.md` or `CLAUDE.md` loads into a ticket's run,
and namespaced per repo, so two repos don't collide. A run uses the
installed plugin, never the files in the worktree it is editing.
