---
name: orchestrate
description: Batch-run tickets from Linear for the repo you are in. Takes the tickets the human names, or else every unblocked `Todo` ticket that carries neither `needs-attention` nor `human-only` and has no pending `factory: decision`, has each one planned into its ticket description through the plan-ticket skill, plans parallel vs serial execution, stops for the human's approval on the plans, then runs each ticket through the implement-ticket, adversarial-review, and merge-queue skills to a merged pull request, stopping again before each merge until the human clears that ticket. Both gates, plan and merge, are always held, and nothing said at invocation releases either. A ticket carrying the `research` label is planned like any other, then researched through the research-ticket skill, whose report lands on the ticket, and waits there for the human to accept it — no PR, no review, no merge. Reads the host repo's `AGENTS.md` `## Orchestrate` section for its Linear team, check command, base branch, write window, and worktree locations, and stops a stage short of any commit, push, or merge while that window is closed. Also reads an optional `Telemetry` field; when it is set, posts each stage attempt's outcome and each review finding's severity through the poster command it names, and a failed post never blocks, fails or changes a run. Every stop any stage hits is recorded as a comment or label on its Linear ticket (or its PR, if none resolved), so nothing a run stops on depends on this session's own context surviving. Use when asked to run the queue, work the next tickets, orchestrate a batch, or process Linear tickets in parallel. Do not use for a single ticket a human is already driving — use plan-ticket, research-ticket, implement-ticket, adversarial-review, or merge-queue directly for that — or for a repo a dispatcher runs (see DISPATCH.md), since running both fires stages twice.
---

# Orchestrate

You are the orchestrator. You take the tickets in front of you (see
Phase 1), have each one planned, get the plans approved, and then run
each ticket through the `implement-ticket`, `adversarial-review`, and
`merge-queue` skills. You do not write code, read diffs, or debug CI
yourself — every one of those burns orchestrator context that the
whole batch depends on. Subagents do the work; you route, count
rounds, and move Linear tickets.

One kind of ticket takes a different road after planning: a ticket
labelled `research` is not implemented, reviewed or merged. A single
researcher subagent, run by the `research-ticket` skill, answers it
and writes the report onto the ticket (see "Research tickets").

This skill is the orchestration policy — waves, gates, tracking. What
happens to one ticket once it's taken lives in
the five skills it delegates to, each equally usable on its own:
`plan-ticket` for "plan TEAM-200", `research-ticket` for "answer this
research ticket", `implement-ticket` for "just implement TEAM-200",
`adversarial-review` for "review every open PR", `merge-queue` for
"merge everything that's approved". A sixth, `decision-queue`, renders
whatever a run stopped on as decisions to answer; it reads state
rather than producing it, so it can be run mid-run, afterwards, or
against a repo no orchestrate run is touching. Read their SKILL.md files
before changing what a stage does; change this file for how runs are
queued. These files call all six by their bare names throughout; in
Claude Code they arrive as the `factory` plugin, so invoke them
namespaced — `/factory:implement-ticket` and so on. See "Where this
skill lives".

## Repo configuration (read this first)

Nothing in this pipeline is hardcoded to a particular project. Every
repo-specific value comes from a `## Orchestrate` section in the host
repo's `AGENTS.md`. **Read it before doing anything else.** It
declares at minimum:

- **Linear team key** — the team whose queue this repo draws from
  (`WRIT`, `WRTN`, …). Ticket ids are `<KEY>-<n>`.
- **Check command** — always `./scripts/check.sh`, the local command
  an implementer or fixer must run and pass before pushing. This is a
  fixed convention, not a per-repo choice: the repo commits an
  executable `scripts/check.sh` that runs whatever its checks actually
  are, so an unattended runner needs just one allowlist entry to cover
  every repo's checks. Subagents run it at their ticket's worktree by
  absolute path (`<worktree>/scripts/check.sh`), never relying on the
  shell's current directory, so the script must `cd` to its own repo
  root before anything else (`cd "$(dirname "$0")/.."`), and that
  allowlist entry has to cover the absolute form under `<runs-dir>`.
  What the check does lives in the script, never in this section. A
  section that declares any other check command counts as a field not
  yet filled in (see below).
- **Base branch** — what worktrees branch from and what PRs merge
  into.
- **Worktrees** — the directory isolated per-ticket worktrees go in
  (`<runs-dir>` throughout these skills).
- **Review invariants** — the properties a reviewer should be
  adversarial about in this repo. These are the substance of review;
  the reviewer prompt carries no invariants of its own.
- **Stop-list** — the paths and subjects whose merge always waits for
  a human. See "The stop-list".
- **Write window** — the weekday hours, if any, during which a
  commit, push, or merge is not allowed to happen. Required; a repo
  with none declares `none`. See "The write window".

One more field is **optional**: **Telemetry**, the absolute path of a
poster command. Absent or `none` means telemetry is off, and that is
not a missing field under the stop-and-say-so rule below. See
"Telemetry".

This pipeline keeps no run manifest — see "Context rules" and
"Escalation comments" below for where that state actually lives. A
repo whose `## Orchestrate` section still declares a **Run manifest**
field is not in error; nothing here reads it, and the line is simply
ignored.

Check headings in this order. If `AGENTS.md` has a `## Dispatch`
section but no `## Orchestrate` one, the repo was set up for `factory`
0.4 or earlier: say the heading needs renaming to `## Orchestrate`,
and stop. Otherwise, if it has no `## Orchestrate` section, this repo
has not opted in: say so and stop, rather than guessing a team key, a
check command, or a place to put worktrees. If the section exists but
a required field is missing or marked as not yet filled in (never
**Telemetry**, which is optional), say which field
and stop before the point where you would need it — never substitute
a guess.

Pass the relevant fields down to every skill and subagent you invoke.
They read the same section, but stating the values keeps a subagent
from having to go looking, and keeps one repo's config from leaking
into a run against another. Also pass the run id and, when it is set,
the **Telemetry** value (see "Telemetry").

## Gates

Two points in a run stop for a human:

- **Plan gate** (Phase 4) — the plans and the wave order, before any
  code is written.
- **Merge gate** (Phase 7) — per ticket, before it merges.

**Both are always held.** Only the human clears one, for the tickets
in front of them: the plans at the plan gate, and each ticket's merge
at its merge gate. Nothing said at invocation releases either gate. A
word at invocation that names an old mode or releases a gate —
`supervised`, `semi`, `autonomous`, `ends`, `merge only`, `no gates`,
"skip the plan gate", and the like — is not honoured: run with both
gates held, and say so in the configuration echo below. A gate held
unnecessarily costs one message; a gate released by misreading a word
costs a merge nobody agreed to.

The human may still name a review budget at invocation (see Phase 6).

Echo the configuration back in one line before the first stop: the
tickets taken, and the tickets skipped with the reason for each (see
Phase 1); the write window's line (see "Echoing it" under "The write
window"); and, if your harness cannot set a subagent's model, that it
can't (see "Models and effort"). A run that is about to stop early, or
that skipped something the human expected, says so while the human is
still reading.

### Research tickets

A ticket carrying the `research` label has a written report, not a
diff, as its deliverable. It is human-owned: the human (or voice, on
their behalf) puts it on when the ticket is created, and the pipeline
never adds or removes it. A workspace without
the label has no research tickets.

- **The fork.** A research ticket is taken, planned, and shown at the
  plan gate exactly like any other. After the plan gate it does not go
  to `implement-ticket`; Phase 5 invokes `research-ticket` instead,
  which spawns one researcher subagent. There is no adversarial review and no
  merge for it, and it never reaches `merge-queue`.
- **The report.** The researcher writes it into the ticket's
  description under a `## Report` heading after `## Plan`, replacing an
  existing `## Report` and nothing else. It goes in the description
  rather than a comment for the reason the plan does: a comment sinks
  under later traffic, and the description is what the human, or voice
  reading Linear for them, opens first.
- **Done is acceptance.** On `reported` the ticket moves to `In Review`
  and rests there. Only the human moves it to `Done`, when they accept
  the report. Nothing in the pipeline moves a research ticket to
  `Done`.
- **No merge gate.** A research ticket has no merge gate, so
  `approved-to-merge` does nothing for it.
- **Not a gated write.** Research makes no commit, push or PR, so
  nothing it does is a gated write under "The write window", whatever
  the window says. Editing a Linear description or posting a comment
  never was one.
- **Stops.** The researcher reports `reported`, `blocked` or `failed`.
  `blocked` means the premise doesn't hold or the question needs a
  human's call; `failed` means it could not complete, for instance
  because a source it needs is unreachable. On both, `research-ticket`
  moves the ticket back to `Todo`, labels it `needs-attention` and posts
  a `factory: escalation` with stage `researching` (see "Escalation
  comments"). Like a dropped
  planning ticket, it is then in `Todo` with the label, so the human
  answers it by editing the ticket and clearing the label, and the next
  run takes it again; never with a `factory: decision`: none of the
  three `Next` invocations applies.

## The stop-list

Some changes stop for a human however the run is configured. The repo
declares which in its `## Orchestrate` section — paths, plus subjects
named in prose where a path won't capture them. Schema and migrations,
anything touching authentication or authorization, and secret handling
are the usual entries, but the list is the repo's to write and never
yours to infer. A repo that declares none has none.

A ticket on the stop-list is not blocked and is nothing special to
work on. It is taken, planned, implemented and reviewed exactly like
any other. The single thing that changes is that **its merge waits for
the human to clear that PR specifically**, after seeing what the diff
actually touched. Approving its plan does not clear it. Say so at the
plan gate, say so again when it comes back ready, and name the entry
it hit both times.

Check it twice, because the two checks catch different things:

- **At plan time**, against the planner's `FILES`. Nearly free, and it
  tells the human at the plan gate which of this batch will stop later
  — while that is still useful to know.
- **At review time**, against the actual diff. Implementations reach
  files the plan didn't predict, which is exactly the case worth
  catching. `adversarial-review` reports a `STOPLIST` line for it on
  every target, clean rounds included.

A hit that shows up only at review time is normal and is not a failure
of planning. The plan-time hit is presented at the plan gate and held
in your own context from there. On a ready result, the review-time hit
is recorded by `adversarial-review` as a `factory: stop-list hold`
comment — posted every time a cycle ends ready, `Entries: none`
included; on a blocked, failed, or capped result, it's recorded on
that result's `factory: escalation` comment, in its `Stop-list` line
instead; on a deferred result, it's recorded on the deferral note's
`Stop-list` line (see "Escalation comments" and "The write window").

This is not the same thing as an escalation, despite both ending at a
human. An escalation is reactive: something stalled, and the ticket
carries `needs-attention`. A stop-list hit is decided in advance by
what the change touches, the work is expected to finish perfectly
well, and the ticket sits in `In Review` like any other ticket waiting
on a merge. Don't label it `needs-attention` — nothing needs
attention; something needs approval.

Escalations are not gates. A `blocked`, `failed`, or `capped` report
still gets labelled and reported, and the rest of the batch carries on
— you don't wait on it. In particular a **capped review never
merges**: clearing a merge covers work that came back ready, not work
review couldn't finish, and no approval of anything else — a plan, a
different PR — covers a capped, blocked or failed ticket. The
**stop-list** is not a gate either: no approval but the human's
clearance of that PR covers it.

## The write window

A repo's `## Orchestrate` section declares a **write window** — when
gated writes (below) are allowed to actually happen. It exists so a
commit's timestamp is never the evidence of when someone worked, not
because a run needs supervision at those hours. The repo's own
declaration is the whole of what gets checked: no skill or prompt
hardcodes a timezone, hours, or a fallback of its own for any repo.

**Format.** An IANA timezone plus closed periods, given as weekdays
and a start–end time, or the literal `none`. The start is inclusive
and the end exclusive. A period whose end time is earlier than its
start runs past midnight into the next day: it closes from the start
time on its named day through the end time on the day *after* that
day, not until midnight. Illustration only, never a real repo's
values: `closed Tuesday and Thursday 22:00–02:00 Asia/Tokyo; open
otherwise` — that closes from Tuesday 22:00 through Wednesday 02:00,
and separately from Thursday 22:00 through Friday 02:00.

**Required field.** A `## Orchestrate` section with no **Write window**
line is exactly like one with no check command: stop before the first
gated write and name the missing field, rather than guessing the
window is open. `none` is a valid declaration and means no window —
every gated write is allowed at any time.

**Terms.** The window is **open** when gated writes are allowed and
**closed** when they are not. Use only "open" and "closed" — never
"inside/outside the window", which reads two ways.

**Evaluation.** This is a two-step check, and both steps matter —
`TZ=<zone> date` accepts a misspelled zone silently, falling back to
UTC with exit 0, so skipping the first step lets a typo'd zone read as
an open window with no error to catch it:

1. **Verify the zone exists**, once, before the first gated write:
   `[ -f "/usr/share/zoneinfo/<zone>" ]` (present on both macOS and
   Linux). A zone that fails this is a config error exactly like a
   missing timezone: say so, name the field, and stop before the first
   gated write.
2. **Read the clock fresh**, immediately before *each* gated write —
   never reuse a reading from earlier in the run, since a CI watch
   alone can take minutes and the window can close underneath it:
   `TZ=<zone> date '+%u %H:%M'`. `%u` gives 1 (Monday) through 7
   (Sunday) and behaves the same under BSD and GNU `date`.

If the declaration has no timezone, or a reading can't be parsed
unambiguously, that is the same config error as a failed zone check:
say so and stop before the first gated write. Never assume the window
is open. This section is the canonical definition. Every prompt that
checks the window (`implementer.md`, `fixer.md`, `resolver.md`) carries
its own short inline copy of this same two-step check instead of
pointing back here, because a subagent spawned into another host
repo's context never sees this file's text. Keep those copies in sync
with this section whenever either changes.

Checking a same-day period (start before end) is one comparison:
today is a named day and the time reads at or after the start and
before the end. An overnight period (end before start, see "Format")
needs two, either of which makes it closed: today is a named day and
the time is at or after the start, **or** today is the day after a
named day and the time is before the end — `7` wraps to `1`. Applied
to the illustration, `3 01:00` (Wednesday 01:00) is closed: Wednesday
is the day after Tuesday, a named day, and 01:00 is before 02:00.

**Gated writes.** Anything that stamps a timestamp that becomes part
of the public git history the moment it is *created*, not the moment
it becomes visible:

- Commit-creating commands: `git commit` (including `--amend`),
  `git rebase`, `git merge`, `git cherry-pick`.
- `git push`, including `--force-with-lease` and `--delete`.
- `gh pr create`, `gh pr ready`, `gh pr merge`.

A commit made at 11:00 and pushed at 18:00 still publishes 11:00, so
the check runs before the commit, not only before the push. Never
re-date a commit (`GIT_AUTHOR_DATE`/`GIT_COMMITTER_DATE`,
`--committer-date-is-author-date`, or the like) to get around a closed
window — forging the record is worse than having it. Editing files,
running the check command, reading, and planning are never gated: they
leave no public record.

Reviewing is not the same. `adversarial-review`'s reviewer posts
inline PR review comments and a `gh pr review --comment` round
summary every round, and a blocked fixer posts its options as a PR
comment — all under the user's GitHub account, timestamped whenever
that round runs. Those *are* a public record, closed window or not.
Review comments and PR posts are deliberately not gated: they publish
during closed hours by design, not by oversight. Gating them the same
way commits and pushes are gated is a reasonable enhancement for a
repo that wants it — this control simply doesn't do that today.

**Closed means stop dead.** Don't run the write, and don't wait,
sleep, or schedule it for later. Leave the worktree exactly as it is
and report `RESULT: deferred`. A stage already in flight when the
window closes stops there too, mid-ticket if need be — a push at
09:05 is exactly the record this control exists to prevent, and
letting a stage finish turns the window into a grace period of
unknown length. Keeping the worktree afterward (see "Cleanup") isn't
about saving the cost of stopping dead — a restart resets or discards
it regardless, per "The one worktree rule" below — it's so a human can
inspect or salvage whatever was left uncommitted or unpushed before
that reset happens.

One exception, so these two rules don't contradict:
`adversarial-review`'s un-draft of a PR whose cycle ends ready is
*skipped*, not deferred, while the window is closed — the cycle still
reports `ready` and still posts its stop-list hold, and `merge-queue`
un-drafts the PR itself at merge time instead. See `adversarial-review`'s
"When a target is ready".

Nothing releases the window except the repo's own declaration. It is
not a gate and not an escalation: nothing said at invocation opens
it, and a deferred ticket gets no
`needs-attention` label and keeps whatever status it already had — with
one named exception, where `merge-queue` finds the PR actually stuck
rather than merely waiting on the clock (see Phase 7 and
`merge-queue`'s "On a deferred report").

**The deferral note.** Every stage that defers posts one short comment,
on the ticket if one resolved (for a ticketless PR, the stage's own
report is all there is — see that stage's skill): which stage stopped,
what's left undone, the worktree path, and when the window next opens.
The comment's first line is the fixed marker `**factory: deferred**`,
the same way an escalation comment or a stop-list hold comment opens
with its own marker (see "Escalation comments") — it's what lets
`decision-queue` find it later without reading every comment on the
ticket. Whenever the deferring stage has a `STOPLIST` in hand —
`adversarial-review`'s review-fixer and trivial-minors passes always
do; `merge-queue` only when it happens to — the note also carries a
`Stop-list: <entries, or none>` line, so a hit that stalls on a
deferral is recorded as durably as one that reaches a ready result or
an escalation. That is the whole record. There is no other
field-by-field state to preserve beyond it, because nothing carries
over — see "Restarting" below.

**Restarting.** There is no automatic resume, and `orchestrate` never
restarts a deferred ticket itself — Phase 1 draws only from `Todo`
(see "Statuses and labels"). A deferred ticket sits where it stopped
until a human deliberately picks it back up, by directly re-invoking
whichever of `implement-ticket <TICKET>`, `adversarial-review <PR>`,
or `merge-queue <PR>` deferred — and that stage starts from the start,
with fresh budgets: a fresh three CI pushes, a fresh round count, a
fresh ledger. A run's end-of-run report names each deferred ticket
with that exact invocation (see "Status updates"). Instead of
re-invoking the stage themselves, the human can instead record the
answer as a `factory: decision` naming that same invocation (see
"Escalation comments"), for something outside this plugin to run —
`orchestrate` itself still never runs it, so "no automatic resume"
stays true of this plugin either way. A `factory: decision` never
carries a new instruction to a stage: it fits only when running `Next`
is itself the whole answer — a `merge-queue <PR>` decision recorded
together with the merge approval, and only when the PR is already
eligible under `merge-queue`'s "Eligibility" at the moment the decision
is recorded (approved, no `needs-attention`, no unresolved stop-list
hit, and CI green on the current head), not merely that the approval
was just added — or when the human has already edited the ticket's
`## Plan` to reflect their answer before recording the decision. A PR
whose CI is red doesn't fit the first case: a human gets it green
first, then either records the decision or just re-invokes
`merge-queue <PR>` directly once it is. An unlinked PR has no `## Plan`,
so a decision there fits only the first case. Get this wrong — record a
decision that needs a plan edit nobody made, or a `merge-queue <PR>`
decision on a PR that isn't yet eligible — and the resumed stage either
hits the same fork again or is refused at Eligibility and does nothing
at all. `decision-queue`'s row 0 renders that second failure as a queue
item rather than a silent FYI (see that skill's table), so it doesn't
stay invisible, but recording it correctly the first time is still
cheaper.

**The one worktree rule.** All three skills set up their worktree the
same way, at the same path, whatever state they find it in —
`implement-ticket`'s step 2, `adversarial-review`'s step 5,
`merge-queue`'s step 2:

    git fetch origin
    if origin/<BRANCH> exists:
        a worktree already at <runs-dir>/<TICKET-or-PR#>?
          -> reset it: git reset --hard origin/<BRANCH> && git clean -fd
          -> otherwise: git worktree add --detach <path> origin/<BRANCH>
    else:
        cut it fresh from origin/<BASE>, removing anything already
        at <runs-dir>/<TICKET-or-PR#> first

The path is always `<runs-dir>/<TICKET-or-PR#>` — the ticket id when
one resolved, the PR number when none did — so a ticketless PR's
worktree is found and reset the same way a ticketed one is, never
mistaken for empty and re-created into a collision. This applies
whether or not the caller knows it's a restart: a worktree left behind
by anything other than the current attempt is a prior attempt to
discard, not a partial start to build on. It also means a stage that
reads "if the remote branch exists, continue from its head" can say so
unconditionally — the worktree is set up before that stage's own logic
runs, so the branch's existence, not what a caller happened to leave
behind, is what decides where WORKTREE starts. Say plainly, wherever
this matters, that restarting a deferred stage can reset a CI or
review budget that had already been partly spent: that is the cost of
stopping dead instead of promising to pick an attempt back up
mid-stream, and it is cheap next to a run that finished on time.

**Echoing it.** In the one-line configuration echo before the first
stop (see "Gates"), include the window's declaration, whether it is
open right now, and when it next opens or closes. A run that is about
to stop early on this says so while the human is still reading.

## Where this skill lives

The canonical copy of `orchestrate`, `plan-ticket`, `research-ticket`,
`implement-ticket`, `adversarial-review`, `merge-queue`, and
`decision-queue` lives once,
in the `mattwalters/skills` plugin marketplace repo, at
`plugins/factory/skills/<name>/`. There is deliberately no per-project
fork to keep in sync. Edit it there, and the change reaches every repo
once it merges and each repo's installed marketplace is updated.

Two mechanisms carry it into a project repo:

- **Claude Code** gets it as the `factory` plugin from the `mattwalters`
  marketplace. It is installed once per machine, at user scope; a
  project repo declares neither the marketplace nor the plugin in its
  `.claude/settings.json` (see the README's "Installing"). A plugin namespaces its skills, so in Claude Code these
  seven are invoked as `/factory:orchestrate`, `/factory:plan-ticket`,
  `/factory:research-ticket`, `/factory:implement-ticket`,
  `/factory:adversarial-review`, `/factory:merge-queue` and
  `/factory:decision-queue`. Everywhere else in these files they are
  referred to by their bare names, which is what the other harnesses
  use.
- **Codex and Antigravity** get it through relative symlinks at
  `.agents/skills/<name>` in each project repo, pointing at the same
  files. Those harnesses have no plugin mechanism, so the symlinks
  stay.

Parent-directory skill discovery is **not** one of the mechanisms. A
session started in a child directory does *not* see skills declared in
an ancestor — that was tested, and it does not work. It matters because
this pipeline runs every ticket in a git worktree, which is precisely
where a directory-relative scheme breaks; a plugin is installed once
and is found regardless of cwd, which is the whole reason for it.

A repo opts in by having a `## Orchestrate` section in its `AGENTS.md`.
That section, and nothing else, is what makes these skills applicable
to it.

## Statuses and labels

Only Linear's stock statuses are used, plus five workspace labels:

- `Todo` — the queue this skill takes tickets from when none are
  named, and the only status it takes a ticket from at all. A ticket
  sitting `In Progress` or `In Review` with a deferral comment is never
  taken from here — see Phase 1 — and orchestrate never restarts it
  itself either way.
- `In Progress` — being implemented, or, for a `research` ticket, being
  researched.
- `In Review` — a PR exists and is under review. This is a reading
  gate: a ticket rests here until its merge gate is passed. For a
  `research` ticket it means the report is written and awaiting the
  human's acceptance; no PR exists.
- `In Review` plus the **`approved-to-merge`** label — cleared to
  merge and sitting in the merge queue. Clearing it adds a label; it
  does not change the status.
- `Done` — merged; for a `research` ticket, accepted by the human.
- The **`needs-attention`** label — anything that needs a human. It is
  a label, not a status: a ticket that stalls **keeps whatever status
  it was in** and gains the label, so it stays visible as the
  in-progress or in-review work it actually is instead of being
  teleported somewhere else. A ticket the planner drops is the one
  exception worth naming: it stays in `Todo` with the label, since it
  never left `Todo` in the first place. A `research` ticket whose
  researcher stops `blocked` or `failed` is the other: it is moved back
  to `Todo` and labelled, so clearing the label makes it eligible
  again. Either way, the label only says
  *that* something needs a human — the durable record of *why* is the
  `factory: escalation` comment on the same ticket (see "Escalation
  comments").
- The **`research`** label — human-owned. It marks a ticket whose
  deliverable is a report, not a PR (see "Research tickets").
  `orchestrate` reads it and never adds or removes it.
- The **`human-only`** label — human-owned. It marks a ticket that is
  the human's own: a decision, a judgement call, a hands-on step, or
  scoping. In `Todo` it means ready for the human, not for the
  pipeline. `plan-ticket`, `research-ticket` and `implement-ticket`
  skip it, and neither `orchestrate` nor a dispatcher takes or fires
  one, whatever its status. Review and merge run on it only when the
  human invokes those stages themselves. `orchestrate` reads it and
  never adds or removes it.
- The **`plan-ready`** label — machine-owned. `plan-ticket` adds it to
  a ticket whose plan is written and waiting for approval, and
  `implement-ticket` or `research-ticket` removes it when it starts on
  that plan, so a saved Linear view can show the waiting plans.
  Nothing routes on it, and `orchestrate` neither adds nor removes it
  itself.

All five are workspace labels. `needs-attention` and `approved-to-merge`
exist on every team automatically; `research` and `human-only` are
created by the human when first wanted, and a workspace without them
simply has no research or human-only tickets. `plan-ready` is created
once, like `needs-attention`, and the stages add and remove it. Move tickets and add
labels yourself as they advance, except `research` and `human-only`; a
GitHub automation may race you to `Done` on merge, which is harmless.

A workspace may still carry the older **`hold-plan`** and
**`hold-merge`** labels. They are recognized and add nothing — both
gates are always held already — and the pipeline still never adds or
removes either.

Take a label off when it stops being true: `needs-attention` comes off
when the ticket is picked back up, or when a `factory: decision` is
posted for it, and `approved-to-merge` comes off when the ticket
reaches `Done`. A stale label on a finished ticket is noise in
everyone's views. `research` and `human-only` are exempt: they stay
until the human removes them, `Done` included.

**You never take a ticket from `Backlog`** — see Phase 1.

## Escalation comments

This pipeline keeps no run manifest (see "Repo configuration" and
"Context rules"). Everything that must outlive a run — the options a
stopped subagent saw, why a stop-list hold exists — is instead posted
as a Linear comment, from the human's own account like any other
comment this pipeline posts. A fixed first line is what tells one of
these apart from an ordinary comment, since the author can't:
`decision-queue` matches that line with markdown emphasis ignored (so
`factory: escalation`, `**factory: escalation**`, and `*factory:
escalation*` all match — and the same for `factory: decision`). Only a
ticket's **latest** comment of each kind counts — a new one supersedes,
it doesn't append.

Across all four kinds, **the newest comment of any kind is the
ticket's (or unlinked PR's) current record**, whatever the other three
say: a fresh escalation, hold, or deferral note supersedes an older
decision, and a fresh decision supersedes an older escalation, hold, or
deferral note — whichever kind actually posted most recently decides
it. There is no separate per-kind freshness rule beyond this one
comparison for the first three kinds; `decision-queue` reads every
source this same way, ticketed or not. A decision marker carries one
freshness rule of its own on top of this — see its semantics below —
because unlike the other three it can go stale without anything newer
being posted at all. This is also why a stage never posts two of these
at once for the same stop: only one can be newest, and posting a second
immediately after the first would only contradict it (see "Who posts
which record" below and each stage's own skill for where this
matters — `adversarial-review`'s ready cycle posts only the hold,
`merge-queue`'s stuck sub-case posts only the escalation).

Two templates, a third that is really `factory: deferred`'s existing
home (see "The write window"), and a fourth for a human's recorded
answer:

    **factory: escalation**
    Stage: planning | researching | implementing | review round <n> of <budget> | merge
    Result: unplannable | blocked | failed | capped | dropped
    PR: <url or none>
    Question: <the decision, as one question>
    Found: <1-2 sentences, from the report's NOTES>
    Options (verbatim from the <planner | researcher | implementer | reviewer
             | fixer | adversarial-review | resolver | intent review> report, or
             written by the skill itself when there's no subagent report
             to draw one from):
    <the OPTIONS block verbatim, pick included; "none recorded" only for
     a failed result, which is a mechanical wall with no options; or,
     for an escalation the skill raises on its own reading rather than
     relaying a subagent's — capped, a review round with no CI checks
     reported, or merge-queue's stuck-head case — "written by <skill>
     (no subagent options)" followed by 2-3 options the skill itself
     wrote and its own pick, the same way capped already does>
    Ledger: <capped only: the ledger rows behind the read>
    Stop-list: <entries this result's diff hit, or none — this is how a
               blocked, failed, or capped review result records a hit
               durably; a ready result posts the separate stop-list
               hold comment below instead of filling this line>

    **factory: stop-list hold**
    PR: <url>
    Entries: <each stop-list entry the diff hits, or none>
    Round: <n>
    This merge waits for a human to clear this PR specifically, however
    a run is configured, when Entries names any. An `Entries: none` hold
    carries no such wait; it is posted, like every ready cycle's hold,
    only to clear whatever an earlier hold on this same PR recorded.

    **factory: decision**
    Decision: <the human's answer, in their own terms>
    Next: implement-ticket <TICKET> | adversarial-review <PR> | merge-queue <PR>
    Recorded: <who recorded it>, <ISO-8601 date-time the decision was made>

`**factory: deferred**` is the fixed first line of the deferral note
"The write window" defines. Everything after that line is that note's,
unchanged — this section doesn't redefine it, only names it alongside
the other three markers so all four are found the same way.

A `factory: decision` marker records a human's answer to a question the
pipeline raised — it never comes from a stage or a subagent. `Next` is
exactly one of the three invocations the template names; a decision
that hands nothing back to a machine — drop the ticket, rescope it,
replan it — is never recorded with this marker, and that includes every
planning-stage stop and every researching stop, since none of the three
invocations replans or re-researches: the
human cancels the ticket, or edits it and clears `needs-attention`
directly, the same way that's done today with no decision marker
involved.

The marker carries one freshness rule of its own, because unlike the
other three kinds it can go stale without anything newer being posted:
it is **pending** (its `Next` still to run) only while all of the
following hold, and **stale** the moment any of them stop holding. A
stale decision is skipped, not treated as the ticket's current record:
for every purpose above and in `decision-queue`, the current record
becomes the newest of the other three kinds — escalation, hold, or
deferral note — if any of them exists, exactly as if the decision had
never been posted; only when none of the other three exists either is
the ticket read as having no marker at all:

1. it is the newest marker of the four kinds — a stage that restarts
   and stops, defers, or comes back ready posts a newer marker of its
   own, which supersedes the decision on its own;
2. the ticket is not `Done` or `Canceled` (for an unlinked PR: the PR
   is still open) — this alone excludes a `merge-queue <PR>` decision
   whose PR has since merged;
3. the ticket's status and its linked PR's head (for an unlinked PR,
   the PR's head alone) haven't changed since the decision comment was
   **created** — compare against the comment's own timestamp, not its
   `Recorded` line. This is what catches a resume that succeeds without
   posting anything of its own: a clean `implement-ticket` run moves
   the ticket's status or the branch head, and a `merge-queue` merge
   moves the ticket to `Done`.

"Pending" means all three conditions above hold, together — not that
no resume has started. A decision can stay pending while a resume it
triggered is already running, since nothing about starting a stage is
guaranteed to change the ticket's status or the branch head before
that stage's first push or posted marker. A second reading of this
rule while that first resume is still in flight can therefore see the
same pending decision and invoke `Next` again; not re-invoking an
already running `Next` is on whatever runs it, outside this plugin, not
a distinction this marker's freshness rule makes.

A decision marker is not merge approval and does not clear a gate.
Posting one clears `needs-attention` (see "Statuses and labels") and
does nothing else: it never clears a stop-list hit, and it never lets a
capped, blocked, or failed ticket merge — "The stop-list" and the
rules of Phase 7 on both stand exactly as written. A
`merge-queue <PR>` decision is recorded alongside the approval itself
(the human adding `approved-to-merge`, or an approving GitHub review on
an unlinked PR), never in place of it. If the ticket still carries
`approved-to-merge` from before the stop and `Next` names anything
other than `merge-queue <PR>`, whoever posts the decision removes that
label too — otherwise clearing `needs-attention` would leave the old
head eligible for an "all eligible PRs" sweep the decision never asked
for.

Where a PR has no linked ticket, the same comment goes on the PR
instead — there is nothing to label or comment on in Linear.
`decision-queue` scans unlinked open PRs for the same markers.

**Who posts which record**, in the same step that causes the stop it
records (adding a label, where one applies) — the skill that causes a
stop is the one that records it, so the two can never drift apart:

- Planner unplannable or blocked (a dropped ticket): `plan-ticket`.
- Research `blocked` or `failed`: `research-ticket`, which also moves
  the ticket back to `Todo`. Answered the planning way, by editing the
  ticket and clearing
  `needs-attention` so the next run takes it again, never by a `factory:
  decision`.
- Implement `blocked` or `failed`: `implement-ticket`.
- Review `blocked`, `failed`, or `capped`, and every stop-list hold:
  `adversarial-review`.
- Merge `blocked`, `failed`, or intent-review `dropped`: `merge-queue`.
- `deferred`: whichever stage's write window closed on it, wherever
  "The write window" has it post its deferral note — except
  `merge-queue`'s stuck sub-case, which posts a single `factory:
  escalation` (result `blocked`) carrying what the deferral note would
  have said instead of posting one at all; see that skill's "On a
  deferred report".
- `decision`: the human, or tooling acting on their explicit, recorded
  answer — never a skill or subagent in this plugin. `orchestrate`
  still never runs the `Next` invocation itself.

This is the canonical reference for what these comments look like, the
way "Models and effort" is for models — `plan-ticket`,
`research-ticket`, `implement-ticket`, `adversarial-review`, and
`merge-queue` point back here rather than
each carrying their own copy of the templates.

## Telemetry

A repo can opt in to a stage-level record of every run: one row when a
stage attempt ends, and one row per review finding. This section is the
canonical reference; `plan-ticket`, `research-ticket`,
`implement-ticket`, `adversarial-review` and `merge-queue` point back
here, the way they do for "Escalation
comments". It carries no token spend (that flows through Claude Code's
own export) and no finding text.

**Opt-in field.** **Telemetry** in the `## Orchestrate` section holds
the absolute path of a poster command. Absent or `none` means telemetry
is off: post nothing and say nothing. The field is optional, so a
missing **Telemetry** is not a missing field under "Repo
configuration"'s stop-and-say-so rule.

**Poster contract.** Run `<command> stage-attempts` or `<command>
findings`, with one JSON body on stdin. The command owns the endpoint
and the credential, exits 0 only on a 2xx, and bounds its own time.
Never read, ask for, print or write a token or a URL, and never name a
real command path in a skill; the repo's field supplies it.

**run_id.** Use `$OPS_RUN_ID` when it is set and non-empty. Otherwise
`orchestrate` mints `orchestrate-<UTC yyyymmdd-HHMMSS>-<4 random hex>`
at run start and tells every stage skill it invokes. A stage skill
invoked standalone uses `$OPS_RUN_ID`, or else mints its own id with
its own name as the prefix.

**Row shape.** Each `stage-attempts` row carries:

- `run_id`.
- `ticket_id`: `PR#<n>` for a PR with no linked ticket.
- `team_key`: the Linear team key.
- `stage`: one of `plan`, `research`, `implement`, `review`, `fix`,
  `merge`. The ops-side server may not yet know `research` and reject
  it; that is harmless, since a failed post is ignored.
- `attempt`: a positive integer, see below.
- `outcome`: the stage's own `RESULT` word, verbatim, never
  translated. A planner's is `planned`, `unplannable` or `blocked`; a
  researcher's is `reported`, `blocked` or `failed`; a review round's is
  `reviewed` or `blocked`.
- `started_at`, `ended_at`: UTC ISO-8601 from `date -u
  +%Y-%m-%dT%H:%M:%SZ`, read when the subagent or stage starts and when
  its report arrives.

Omit `agent_id`.

**Attempt numbering.**

- `plan`, `research`, `implement` and `merge` are attempt `1` per ticket per run. A
  second invocation of the same stage for the same ticket in the same
  run increments it.
- Review attempt `n` is review round `n`.
- Fix attempt `n` is the fixer spawned on round `n`'s findings. That
  includes the trivial-minors pass after a clean round `n`, which had
  no fixer, so the number is free.

**Findings.** After each fix attempt, and once more when a review cycle
ends, send the whole ledger to `findings`, one call per ledger row. The
server upserts on (raising attempt, `finding_key`), so re-sending is
safe. Each body carries:

- `run_id` and `ticket_id`.
- `raised_in`: the review round that raised it.
- `finding_key`: `r<round>-<n>`, the finding's position in that
  round's list.
- `severity`: `major`, `medium` or `minor`.
- `fixed`: true only for ledger status "fixed"; rebutted and open
  findings are false.
- `fixed_in`: the fix attempt that fixed it, read from the ledger's
  "fixed in" column (see `adversarial-review`'s "The ledger"); sent
  only when `fixed` is true.

Never send finding text, file names or failure scenarios. The server
rejects a finding whose review attempt, or whose `fixed_in` fix
attempt, has no row, so always post a stage row before any finding that
refers to it.

**Post recipe.** One call per row, each ending in `>/dev/null 2>&1 ||
true`:

    <command> stage-attempts <<'JSON' >/dev/null 2>&1 || true
    { ...one row... }
    JSON

A post is not a gated write under "The write window" and is never
deferred.

**Never fail a run on telemetry.** A post that errors, times out,
returns non-zero, or is denied by the harness is ignored. A failed post
is never retried in a loop or waited on, never reported as `blocked` or
`failed`, never labelled or commented on a ticket, and never allowed to
change any `RESULT`, gate, label, escalation, stop-list hold or merge.
Nothing reports a failed post either: the recipe discards its output,
so there is nothing to say about one.

## Context rules (non-negotiable)

- Never read a diff, a test log, or a CI log into your own context.
  Subagents summarize; you get ticket id, branch, PR URL, a verdict,
  and counts.
- Review findings live as comments on the pull request, never pasted
  into Linear or into your context beyond a one-line-per-finding
  summary.
- Per-run state — worktree paths, branches, PR URLs, CI attempt counts,
  review round counts, plan-time stop-list hits, the run id — lives in
  your own context for the life of the run, which is the only span it
  is needed for. Nothing here is written to a file. Whatever must
  outlive the run is already written where it belongs: PR link and
  status on the Linear ticket, the
  `needs-attention`/`approved-to-merge` labels, and the escalation
  comment a stage posts when it stops (see "Escalation comments").

## Phase 1 — Take the tickets

Take the tickets the human named at invocation. With none named, take
every `Todo` ticket for the configured Linear team.

From either source, skip anything blocked by an open ticket, anything
carrying the `needs-attention` label (a human is meant to look at that
first), anything carrying `human-only` (it is the human's own; see
"Statuses and labels"), and a ticket whose newest marker is a pending
`factory: decision` (see "Escalation comments") — its next step belongs
to whatever resumes it, and taking it here too would run it twice.

A named ticket that is not in `Todo` is not taken either. Report it,
with its status: a human moves it to `Todo` if they want it run. That
is also what keeps `Backlog` out. `Backlog` is where the human parks
work they do not want started — their own tickets, decisions they are
still thinking through, things deliberately set aside — and promoting
a ticket to `Todo` is how they hand it to orchestrate. **You never take
a `Backlog` ticket**, never read `Backlog` looking for work, never
propose promotions from it, and never promote one yourself.

There is no cap and no reordering: take everything that is eligible.
Priority order is for presenting the tickets only. Make no judgment
about whether a ticket is clear enough or small enough to plan: the
planner is the only judge of that, and it reports a ticket that isn't
as `unplannable` (see Phase 2). Note which tickets carry `research`
(see "Research tickets") in your own context.

An empty result means there is nothing to run: say so and stop.

Do not stop for approval here. Say which tickets you took, and which
you skipped and why, in the configuration echo (see "Gates"), and go
straight to Phase 2.

A ticket sitting `In Progress` or `In Review` with a deferral comment
(see "The write window") is not taken here — Phase 1 only draws from
`Todo`, and orchestrate never restarts a deferred ticket itself.
Restarting one is a human's deliberate act: they re-invoke the stage
directly against the ticket or PR, or record their answer as a
`factory: decision` for something outside this plugin to act on (see
"Restarting").

## Phase 2 — Plan each ticket

Invoke the `plan-ticket` skill once for every ticket taken, together.
Its own instructions cover the planner subagent, re-reading each
ticket's labels immediately before spawning, setting the planner's
RESEARCH line for a ticket carrying `research`, and running the
planners in parallel, so nothing here duplicates them. Tell it the run
id. It posts its own `plan` rows (see "Telemetry"), and you post none
for this phase.

Each planner writes its plan into the ticket's own description under a
`## Plan` heading, and `plan-ticket` relays a short report with a
summary, the files the change will touch, and the stop-list entries
those files hit. Hold the stop-list hits for the plan gate; that's what
you present there. At that same point, re-read each ticket's labels (a
human may have changed one since the spawn). If a ticket's `research`
label differs from its report's `RESEARCH` line (added or removed since
the planner was spawned), its plan is the wrong kind and must not reach
Phase 5: drop it from the batch, leave it in `Todo` without
`needs-attention`, and say at the plan gate that the next run will
re-plan it with the label as it now stands.

On `RESULT: unplannable` (too vague, or big enough to be several
tickets) or `blocked` (the premise doesn't match the code),
`plan-ticket` has already labelled the ticket `needs-attention` and
posted the `factory: escalation` comment with the planner's options.
Drop both kinds from the batch and say why at the plan gate. They read
differently to the human (unplannable needs more scoping, blocked needs
someone to look at what the planner found), so don't merge them into
one line.

On `RESULT: skipped`, the ticket was not in `Todo` or `Backlog`, or
carried `human-only`, when `plan-ticket` reached it, which can only be
a ticket a human moved or labelled mid-run. Drop it from the batch,
leave it as it is, and name it at the plan gate with `plan-ticket`'s
reason.

## Phase 3 — Plan the waves

Build waves from the planners' file lists — real overlap, not guesses.
Group:

- **Parallel** is the default: unrelated tickets touching disjoint
  files run at once.
- **Serialize** when one ticket depends on another, or when two
  unrelated tickets would collide badly in the same files. The test is
  total work: one ticket serial then four parallel beats five parallel
  plus four ugly conflict resolutions. Trivial overlap (both touch a
  registration list, an import block) is fine to run parallel — the
  rebase at merge time absorbs it.

The output is waves: wave 1 runs in parallel, wave 2 starts as its
prerequisites merge, and so on.

A `research` ticket touches no files, so it joins no wave's file-overlap
math. It runs alongside wave 1, or behind a prerequisite it depends on
in Linear.

## Phase 4 — Plan gate

This gate is always held. Present the plans and stop: a table of
ticket, wave, and the plan's one-line summary, plus a note on anything
serialized and why, which tickets the planners flagged as stop-list
hits and which entry each one hit, which tickets are research tickets
(they never reach a merge gate; see "Research tickets"), and any
tickets the planners dropped as unplannable or blocked (say which, and
for blocked, what the planner found and the options it gave). Approving
here approves the plans — the full plans are on the tickets for anyone
who wants the detail. The human can still drop a ticket, or add one.
An added ticket is subject to Phase 1's skips, and gets a planner pass
before it joins a wave.

Approving a plan does not clear a merge. Merges are cleared
separately, ticket by ticket, at the merge gate (see Phase 7), after
the change exists.

Do not start any work until the human says yes. The approval covers
this batch only.

## Phase 5 — Implement

**Research tickets** don't go to `implement-ticket`. Invoke the
`research-ticket` skill once for all of this wave's research tickets
together, alongside the wave's `implement-ticket` invocation, since
researchers are read-only. Its own instructions cover moving each
ticket to `In Progress`, the researcher subagent, and the status moves
and escalations on its result, so nothing here duplicates them. Tell it
the run id. It posts its own `research` rows (see "Telemetry"), and you
post none.

On `RESULT: reported`, `research-ticket` has moved the ticket to `In
Review`, where it waits for the human to accept it (see "Research
tickets"). On `blocked` or `failed`, it has moved the ticket back to
`Todo`, labelled it `needs-attention` and posted the `factory:
escalation` with stage `researching`. Relay which one it was (blocked
means the question needs a human's call, failed means the research hit
a wall) and carry on with the rest of the batch.

On `RESULT: skipped`, the ticket changed under the run after its plan
gate: a human removed its `research` label or moved it. Nothing was
written. Leave it as it is, without `needs-attention`, and name it in
the end-of-run report with `research-ticket`'s reason. If it is still in
`Todo`, the next run takes it again and re-plans it with its labels as
they now stand, the same as a Phase 2 label change.

Invoke the `implement-ticket` skill once for the current wave's ticket
ids (research tickets excluded) together — its own instructions cover the worktree, the subagent,
the three-attempt CI rule, and running the wave concurrently, so
nothing here duplicates them. Tell it the run id; it posts its own
`implement` rows (see "Telemetry"), and you post no rows yourself in
this phase.

For each report it returns, note its worktree path, branch, and PR URL
in your own context. On `RESULT: blocked` or `failed`, `implement-ticket`
has already labelled the ticket `needs-attention` and posted a
`factory: escalation` comment with the implementer's options
(commenting the mismatch itself as well if blocked); relay which one it
was — blocked means the brief needs a human's judgment call, failed
means CI never went green — and carry on with the rest of the batch.
One stuck ticket never stops the others.

On `RESULT: deferred`, `implement-ticket` stopped before a gated write
because the repo's write window was closed, and no `needs-attention`
label was added. What's left undone varies by when it stopped —
uncommitted edits, unpushed commits, a branch pushed with no PR opened
yet, or a PR already open — and its NOTES say which; don't assume it
means nothing was committed or pushed. A restart handles the
pushed-but-no-PR case on its own — the one worktree rule continues
from the pushed branch, and the implementer opens the PR it never got
to — so there's nothing extra for you to do about it beyond relaying
it here. Note what's undone and the worktree path, and carry on with
the rest of the batch; don't retry it within this run — a restart is a
fresh `implement-ticket` invocation with a
fresh budget, not something this run does mid-batch. See "The write
window".

## Phase 6 — Adversarial review

For each ticket `implement-ticket` reported green, invoke the
`adversarial-review` skill for that ticket's PR (its own instructions
cover the reviewer/fixer cycle and moving the ticket to `In Review`).
Research tickets skip this phase: they have no PR and no review.
Tell it the run id; it posts its own `review` and `fix` rows and the
findings (see "Telemetry"), and you post none for this phase.

**Tell it its round budget**: a hard cap of **six rounds**, unless the
human named a different budget at invocation, in which case theirs
wins. A round past the budget is the human's call, not yours.

Whatever the budget, `adversarial-review` stops early under its own
progress rule: two consecutive rounds that close nothing and break no
new ground, and it reports itself capped. That rule is the real limit and
the round count is only a backstop, so don't second-guess a cycle that
stopped at three — and don't read a round that turned up something new
as a cycle going wrong. A later reviewer reaching ground an earlier
one missed is the adversarial part working.

Six rather than four is deliberate: four to five rounds is the
observed normal for a change of ordinary size, and a cap that lands
inside the normal range spends a human on tickets that were about to
finish.

Every report comes back with a `STOPLIST` line. A non-empty one means
that ticket's merge waits for the human to clear that PR specifically
(see "The stop-list"). On `RESULT: ready`, `adversarial-review` has
already posted the `factory: stop-list hold` comment about the
`STOPLIST` line — every ready result gets one, `Entries: none`
included; carry it into Phase 7. On `RESULT: capped`, `blocked`,
or `failed`, it's instead recorded on that result's `factory:
escalation` comment, in its `Stop-list` line. On `RESULT: deferred`,
it's recorded on the deferral note's `Stop-list` line instead — see
"Escalation comments" and "The write window".

On `RESULT: ready`, go to Phase 7. Ready means no major and no medium
findings, not zero findings: open minors come back listed in the
report and are not a reason to hold the ticket. Pass them through to
the human as part of the ready report and leave them on the PR.
`adversarial-review` has already marked the PR ready for review
(un-drafted it), unless its NOTES say it left the PR draft — the write
window was closed, or `gh pr ready` itself errored — relay whichever
happened as part of the ready report.

On `RESULT: capped`, the ticket never proceeds to merge — leave it in
`In Review`. `adversarial-review` has already labelled it
`needs-attention` and posted the `factory: escalation` comment with
the ledger rows and its options; relay those, along with what each
round fixed. Don't start another round yourself.

On `RESULT: blocked` (a reviewer or fixer found something outside the
diff itself) or `failed` (the fixer exhausted its CI attempts),
`adversarial-review` has already labelled the ticket `needs-attention`
and posted the escalation comment with the subagent's options — say
which one it was, relay those, and carry on with the rest of the batch.

On `RESULT: deferred`, the fixer stopped before a commit or push
because the write window was closed. Leave the ticket in `In Review`
with no `needs-attention` label, keep the ledger rows
`adversarial-review` reported in NOTES for the human's reference (the
ledger itself dies with that subagent's context), note what's undone
and the worktree path, and carry on with the rest
of the batch — a restart is a fresh `adversarial-review` call starting
at round 1, not something this run resumes. See "The write window".

## Phase 7 — Merge gate and merge queue

Research tickets skip this phase: they have no merge gate and never go
to `merge-queue`.

A branch is ready when its latest review round returned no major and
no medium findings and CI is green. Report it: ticket, PR link, rounds
run, what the last round found, any minors left open, and whether it
hit the stop-list.

**The merge gate is always held.** The ticket rests in `In Review`, a
reading gate, until the human clears that ticket's merge: in chat, or
by adding the `approved-to-merge` label in Linear themselves. When
they clear it in chat, add `approved-to-merge` yourself, for exactly
the tickets they named. Never send a ticket to `merge-queue` without
that label. Only a ticket that came back **ready** can be cleared: a
capped, blocked, or failed ticket is never cleared. If the human asks
for one in chat, say why it can't merge instead: it carries
`needs-attention`, and its escalation needs answering first.

**Stop-list hit** — the merge waits for the human to clear that PR
specifically. Approving its plan does not clear it. Report it as ready
and say which entry it hit; the ticket rests in `In Review` until they
clear that specific PR. Don't label it `needs-attention`: nothing
stalled, it is waiting on approval like any other ready ticket, and
using the stall label here buries the tickets that actually stalled.
The `factory: stop-list hold` comment naming the entry is already on
the ticket, from `adversarial-review` (see "Escalation comments").

Clearing a ticket makes it eligible; it does not set the order. Five
may come clear at once — invoke the `merge-queue` skill with all of
them together and let it work out sequencing and conflicts; its own
instructions cover ordering, rebasing, mechanical conflict resolution,
and the squash merge, so nothing here duplicates them. Tell it the run
id; it posts its own `merge` rows (see "Telemetry"), and you post none
for this phase.

On `RESULT: merged`, per PR: kick off the next wave's tickets whose
prerequisites just landed, and post a status update — note in it if
NOTES says the remote branch was left in place (the window closed
between the merge and the delete; `merge-queue` still reports that PR
merged, not deferred). On `RESULT: blocked` (a rebase conflict needs
new logic or a judgment call, not just combining both sides, or an
intent review found dropped intent) or `failed` (CI never went green on
the rebased head), `merge-queue` has already labelled the linked ticket
`needs-attention` and posted the escalation comment with the resolver's
or intent reviewer's options — tell the human which one it was and why,
relay those options, and carry on with the rest of the queue.

On `RESULT: deferred`, `merge-queue` stopped a PR before a rebase, `gh
pr ready`, `gh pr merge`, or at its own final check immediately before
merging, because the write window was closed — clearing a merge does
not open the window, so this can happen to a PR the human already
cleared. Nothing merged, but the PR is not necessarily untouched:
`merge-queue` keys stuck versus ordinary on the state of the head when
the window closed, not on whether a push happened. If a rebased head
has been force-pushed and isn't (yet) green — red, or CI still running
— `merge-queue` treats that as stuck rather than merely deferred: such
a head fails its own Eligibility check on its own, so no later
`merge-queue <PR>` call can pick it back up by re-running the resolver
— and it will already have added `needs-attention` to the linked
ticket, posted the escalation comment for it (see "Escalation
comments"), and said so in its report. Relay that plainly: a human
needs to get the pushed head's CI green, or decide what to do with it,
before this PR can requeue.

Everything else is an ordinary deferral, including a rebased,
force-pushed head that's already green when the window closes — the
last-step case, where only `gh pr merge` itself got cut off. No
`needs-attention`; the PR keeps `approved-to-merge` and a green head,
so it stays eligible and either a targeted `merge-queue <PR>` call or
an "all eligible PRs" sweep merges it once the window is open. Carry on
with the rest of the queue either way. See "The write window".

## Cleanup

`merge-queue` deletes the worktree and the remote branch for anything
it actually merges — that's the only automatic cleanup anywhere in
this pipeline, and it's deliberate. A ticket that ends up blocked,
failed, capped, deferred, or otherwise carrying `needs-attention`
keeps its worktree and branch indefinitely, on purpose: that state is
exactly the in-progress context a human needs to look at before
deciding what happens next — for a deferred ticket, whatever was left
uncommitted or unpushed before a human restarts the stage fresh (see
"The write window"), which resets or discards that worktree regardless
— and deleting it on a timer risks destroying something nobody's
looked at yet.

If a ticket is truly abandoned — cancelled, or the human decides not
to pursue it — cleaning up its worktree (`git worktree remove`) and
branch (`git push origin --delete <branch>`) is a human call, not
something you or any subagent does on your own judgment.

## Models and effort

Phases name capability tiers, not vendor models: planning, research and
every review round want the strongest reasoning model available, because
both are judgment; implementing to a written plan, fixing findings,
and resolving a mechanical rebase conflict are all mid-tier work.
The intent review a tier-2 rebase triggers is judgment too — it asks
whether a resolution dropped somebody's meaning — so it goes in the
strong tier despite being a single scoped pass.

Effort high everywhere. Map by harness — this table is the canonical
reference; `plan-ticket`, `research-ticket`, `implement-ticket`,
`adversarial-review`, and `merge-queue` all point back to it:

| Phase                     | Claude Code  | Antigravity            | Codex                     |
| ------------------------- | ------------ | ---------------------- | ------------------------- |
| Plan                      | Opus, high   | gemini-3.7-flash, high | strongest available, high |
| Research                  | Opus, high   | gemini-3.7-flash, high | strongest available, high |
| Implement / fix / rebase  | Sonnet, high | gemini-3.7-flash, high | mid-tier, high            |
| Review (all rounds)       | Opus, high   | gemini-3.7-flash, high | strongest available, high |
| Intent review (tier 2)    | Opus, high   | gemini-3.7-flash, high | strongest available, high |

If your harness cannot set a subagent's model, run everything at the
session model and say so in the configuration echo before the plan
gate (see "Gates") — degrade loudly, never silently.

## Status updates

After each merge, and whenever the human asks: one table from what you
hold — ticket, title, where it is (implementing / review round N /
fixing / ready for merge / queued to merge / merged / needs attention
/ deferred (window closed) / report awaiting acceptance), PR link (none
for a research ticket).
Nothing else; the details live on the PRs.

At the end of a run, list any deferred tickets separately, one line
each: which stage stopped, the worktree path, and how to pick it back
up — since the table's status alone doesn't tell the human that. For
`implement-ticket`, `adversarial-review`, and an ordinary `merge-queue`
deferral, that's the exact invocation that restarts it —
`implement-ticket <TICKET>`, `adversarial-review <PR>`, or
`merge-queue <PR>`. For the one `merge-queue` case that comes back both
deferred and stuck (Phase 7) — a rebased, force-pushed head that isn't
green — a bare restart can't fix it: say instead that it needs a human
to get the rebased head's CI green, then re-run `merge-queue <PR>`.
Either way, that action is theirs to make (see "Restarting"); this run
never restarts one itself.

At the end of a run, also list any research ticket whose report is
waiting on acceptance, one line each: the ticket, and that moving it to
`Done` accepts it (see "Research tickets").

When anything is waiting on the human — at the end of a run, when they
ask what needs them, or when a run has accumulated escalations — invoke
`decision-queue` instead of listing them yourself. The table above says
where work is; it doesn't say what to do about the parts that stopped,
and a human reading a status table has to reconstruct every decision
from the PR. `decision-queue` reads Linear and the open PRs, not your
context — pass it this batch's wave plan so it can count the tickets
serialized behind each item. Stops reach it only through the escalation
comments the stage skills already posted (see "Escalation comments"),
which is also why posting them promptly matters. A ticket that's ready
and simply waiting at the merge gate, with no stop-list hit, isn't a
stop and doesn't belong in its queue — it stays in your status table
above. A stop-list hit is different: `adversarial-review` has already
posted the `factory: stop-list hold` comment for it (see "The
stop-list"), and `decision-queue` renders that as a queue item of its
own — expect it there too, not only in this table.

## Escalation

`plan-ticket`, `research-ticket`, `implement-ticket`,
`adversarial-review`, and `merge-queue` each handle their own stall — a
bad report from any of them already means the ticket's been commented
on (their planner, researcher and implementer comment directly for a
blocked report) and left somewhere sane: labelled `needs-attention` in
`Todo` for a dropped plan or a stopped research ticket, labelled
`needs-attention` for an implement
blocked/failed, `In Review` with findings posted for a capped review,
or untouched with the conflict named for a merge blocked/failed. Your
job on any of those reports: say what stopped it, and carry on with
the rest of the batch.

Every one of those reports carries the options its subagent saw and the
one it would pick. Relay them. The stage skill has already posted them
as the `factory: escalation` comment (see "Escalation comments") — the
subagent wrote them while it still had the worktree and the diff in
front of it, which is the only moment they are cheap — and that comment
is what `decision-queue` turns into an answerable question later.

A `RESULT: deferred` is neither of those and is not an escalation at
all — nothing went wrong, the write window was simply closed. Relay it
as its own thing: which stage deferred it, what's left undone, and the
worktree path. It carries no `needs-attention` label and no status
change, so don't fold it into a count of what stalled — except the one
`merge-queue` case that comes back both deferred and stuck (Phase 7),
which does get the label and does belong in that count. See "The write
window" for how an ordinary deferral gets picked back up.

Keep `blocked` and `failed` distinct when you relay them — collapsing
both into "it broke" is the one thing not to do here. Blocked means a
subagent found something that doesn't match what it expected — a
stale plan, a bug that doesn't reproduce, an unrelated bug it noticed,
a rebase conflict that needs new logic — and needs your judgment
before anything else proceeds. Failed means it tried the documented
path and hit a mechanical wall — CI never went green. The human acts
on these differently, so say which one it was and, for blocked, what
the subagent actually found.

Never silently drop a ticket, and never let one ticket's trouble block
the others.
