---
name: orchestrate
description: Batch-run the next tickets from Linear for the repo you are in. Picks 5–10 unblocked tickets from `Todo` by priority, stops for human approval on the selection, has each one planned into its ticket description, plans parallel vs serial execution, stops for approval again on the plans, then runs each ticket through the implement-ticket, adversarial-review, and merge-queue skills to a merged pull request. Its three gates — selection, plan, merge — are set by naming a mode at invocation: supervised, semi, or autonomous. A ticket carrying a `hold-plan` or `hold-merge` label has that gate held for it alone, whatever the mode. Reads the host repo's `AGENTS.md` `## Orchestrate` section for its Linear team, check command, base branch, write window, and worktree locations, and stops a stage short of any commit, push, or merge while that window is closed. Every stop any stage hits is recorded as a comment or label on its Linear ticket (or its PR, if none resolved), so nothing a run stops on depends on this session's own context surviving. Use when asked to run the queue, work the next tickets, orchestrate a batch, or process Linear tickets in parallel. Do not use for a single ticket a human is already driving — use implement-ticket, adversarial-review, or merge-queue directly for that.
---

# Orchestrate

You are the orchestrator. You pick a batch of tickets, get the picks
approved, have each one planned, get the plans approved, and then run
each ticket through the `implement-ticket`, `adversarial-review`, and
`merge-queue` skills. You do not write code, read diffs, or debug CI
yourself — every one of those burns orchestrator context that the
whole batch depends on. Subagents do the work; you route, count
rounds, and move Linear tickets.

This skill is the orchestration policy — picking, waving, modes,
gates, tracking. What happens to one ticket once it's picked lives in
the three skills it delegates to, each equally usable on its own:
`implement-ticket` for "just implement TEAM-200",
`adversarial-review` for "review every open PR", `merge-queue` for
"merge everything that's approved". A fifth, `decision-queue`, renders
whatever a run stopped on as decisions to answer; it reads state
rather than producing it, so it can be run mid-run, afterwards, or
against a repo no orchestrate run is touching. Read their SKILL.md files
before changing what a stage does; change this file for how runs are
queued. These files call all five by their bare names throughout; in
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
- **Check command** — the local command an implementer or fixer must
  run and pass before pushing.
- **Base branch** — what worktrees branch from and what PRs merge
  into.
- **Worktrees** — the directory isolated per-ticket worktrees go in
  (`<runs-dir>` throughout these skills).
- **Review invariants** — the properties a reviewer should be
  adversarial about in this repo. These are the substance of review;
  the reviewer prompt carries no invariants of its own.
- **Stop-list** — the paths and subjects whose merge always waits for
  a human, whatever mode the run is in. See "The stop-list".
- **Write window** — the weekday hours, if any, during which a
  commit, push, or merge is not allowed to happen. Required; a repo
  with none declares `none`. See "The write window".

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
a field is missing or marked as not yet filled in, say which field
and stop before the point where you would need it — never substitute
a guess.

Pass the relevant fields down to every skill and subagent you invoke.
They read the same section, but stating the values keeps a subagent
from having to go looking, and keeps one repo's config from leaking
into a run against another.

## Gates

Three points in a run stop for a human:

- **Selection gate** (Phase 2) — which tickets this batch will be,
  before any of them is planned.
- **Plan gate** (Phase 5) — the plans and the wave order, before any
  code is written.
- **Merge gate** (Phase 8) — per ticket, before it merges.

**By default all three are held.** Only the human can release one, at
invocation, and the ordinary way they do it is by naming a mode.

Two of the gates can also be held for one ticket alone, by a label the
human puts on it: `hold-plan` and `hold-merge`. See "Per-ticket
holds". There is no `hold-selection` — selection is a batch decision,
and a ticket that shouldn't be picked stays in `Backlog`.

## Modes

Gates are the mechanism; modes are names for the three settings of
them worth having. Naming one at invocation — "orchestrate in autonomous
mode" — carries the whole authorization, so nobody has to recite which
gates are held every time:

| Mode           | Selection | Plan     | Merge    |
| -------------- | --------- | -------- | -------- |
| **supervised** | held      | held     | held     |
| **semi**       | released  | held     | held     |
| **autonomous** | released  | released | released |

- **supervised** is the default and what you run when no mode is
  named. Every decision is the human's.
- **semi** hands them the one decision that is expensive to reverse —
  the plan — and runs to green without them, stopping before merge.
  You pick the batch and you say what you picked; they approve the
  plans; review and fixing run unattended; each merge waits.
- **autonomous** decides everything including merge, and escalates
  only as the pipeline's own stop conditions require.

Modes are presets over the gates, not a replacement for them. The
table below still holds, a mode simply expands into it, and a human
who wants a combination no mode names can still spell it out:

| Invocation                | Gates held             |
| ------------------------- | ---------------------- |
| (default), `supervised`   | selection, plan, merge |
| `semi`                    | plan, merge            |
| `ends`                    | selection, merge       |
| `merge only`              | merge                  |
| `autonomous`, `no gates`  | none                   |

`ends` and `semi` are not the same thing and the difference matters:
`ends` holds selection and releases the plan, `semi` does the
opposite. Both are legitimate — they disagree about which of the two
is worth a human's attention — so take the one that was actually said
rather than the one you'd have picked.

If an invocation names a mode *and* adjusts a gate — "autonomous, but
hold merge" — the adjustment wins. Spelling gates out without a mode
works as it always did.

### Per-ticket holds

A ticket carrying the `hold-plan` or `hold-merge` label in Linear has
that one gate held for it, whatever mode the run was invoked in:

- **`hold-plan`** — the ticket is planned as usual, the plan is written
  into its description, and then its plan gate is held for that ticket
  even if the run's plan gate is released.
- **`hold-merge`** — the ticket's merge gate is held even if the run's
  merge gate is released, with the same effect as a stop-list hit,
  triggered by the ticket rather than by the paths the diff reached.

A hold label beats the mode, beats a gate spelled out at invocation
("no gates" does not clear it), and beats a plan-gate
pre-authorization. Only the human clears it, for that ticket: in chat
during the run, or — for `hold-merge` — by adding `approved-to-merge`
or approving the PR on GitHub. Removing the label clears it for future
runs too. No released gate anywhere clears it.

Other tickets are unaffected. In autonomous mode a batch with one
`hold-plan` ticket runs the rest to merge and stops once, on that one
plan.

The pipeline never adds or removes either label. They are the human's
standing instruction for that ticket and persist across runs. A
workspace that doesn't have them simply has no held tickets.

Name the hold wherever a gate names its stop-list hits: "SKL-n is held
at plan by label", "SKL-n is held at merge by label". When any picked
ticket carries one, the one-line configuration echo before the first
gate says so and names both labels.

If a word at invocation is neither one of the three modes nor a gate
spelled out, it is not a mode: **run supervised** and say that you
did. A gate held unnecessarily costs one message; a gate released by
misreading a word costs a merge nobody agreed to.

Echo the configuration back in one line before the first gate — the
mode, which gates that holds, and which it releases — so a run that is
about to merge without asking says so while the human is still
reading.

A released gate is not a skipped decision. Make the call yourself,
state it in one line with the reasoning you would have presented, and
keep going. State how each gate was passed — approved by the human, or
taken under a released gate — in that one-line call and again in the
phase's status update; the transcript is the record, not a file.

The human can re-hold a released gate at any point mid-run, and
release one only by saying so. When in doubt mid-run, the gate is
held.

## The stop-list

Some changes stop for a human however the run is configured. The repo
declares which in its `## Orchestrate` section — paths, plus subjects
named in prose where a path won't capture them. Schema and migrations,
anything touching authentication or authorization, and secret handling
are the usual entries, but the list is the repo's to write and never
yours to infer. A repo that declares none has none.

A ticket on the stop-list is not blocked and is nothing special to
work on. It is picked, planned, implemented and reviewed exactly like
any other. The single thing that changes is that **its merge gate is
held even in autonomous mode**, and no released gate anywhere can
clear it — only the human, for that ticket, after seeing what the diff
actually touched. Say so at the plan gate, say so again when it comes
back ready, and name the entry it hit both times.

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
still gets labelled and reported in an autonomous run — you just don't
wait on it. In particular a **capped review never merges**, whatever
the merge gate is set to: a released merge gate authorizes merging
work that came back ready, not work review couldn't finish. The
**stop-list** is not a gate either, and it overrides a released merge
gate the same way. So does a `hold-merge` label, triggered by the
ticket rather than the diff — see "Per-ticket holds".

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
not a gate and not an escalation: no mode, no released gate, and no
plan-gate pre-authorization opens it, and a deferred ticket gets no
`needs-attention` label and keeps whatever status it already had — with
one named exception, where `merge-queue` finds the PR actually stuck
rather than merely waiting on the clock (see Phase 8 and
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

**Echoing it.** When you echo the mode back in one line before the
first gate (see "Modes"), include on the same line the window's
declaration, whether it is open right now, and when it next opens or
closes. A run that is about to stop early on this says so while the
human is still reading, the same way a run about to merge without
asking does.

## Where this skill lives

The canonical copy of `orchestrate`, `implement-ticket`,
`adversarial-review`, `merge-queue`, and `decision-queue` lives once,
in the `mattwalters/skills` plugin marketplace repo, at
`plugins/factory/skills/<name>/`. There is deliberately no per-project
fork to keep in sync. Edit it there, and the change reaches every repo
once it merges and each repo's installed marketplace is updated.

Two mechanisms carry it into a project repo:

- **Claude Code** gets it as the `factory` plugin from the `mattwalters`
  marketplace. It is installed once per machine, at user scope; a
  project repo may declare the marketplace in its tracked
  `.claude/settings.json` so Claude Code knows where it lives, but
  never pins the plugin there (see the README's "Installing"). A plugin namespaces its skills, so in Claude Code these
  five are invoked as `/factory:orchestrate`, `/factory:implement-ticket`,
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

Only Linear's stock statuses are used, plus four workspace labels:

- `Todo` — the queue this skill draws new picks from, and the only
  one for that. A ticket sitting `In Progress` or `In Review` with a
  deferral comment is never picked up from here — see Phase 1 — and
  orchestrate never re-picks it itself either way.
- `In Progress` — being implemented.
- `In Review` — a PR exists and is under review. This is a reading
  gate: a ticket rests here until its merge gate is passed.
- `In Review` plus the **`approved-to-merge`** label — cleared to
  merge and sitting in the merge queue. Clearing it adds a label; it
  does not change the status.
- `Done` — merged.
- The **`needs-attention`** label — anything that needs a human. It is
  a label, not a status: a ticket that stalls **keeps whatever status
  it was in** and gains the label, so it stays visible as the
  in-progress or in-review work it actually is instead of being
  teleported somewhere else. A ticket the planner drops is the one
  exception worth naming: it stays in `Todo` with the label, since it
  never left `Todo` in the first place. Either way, the label only says
  *that* something needs a human — the durable record of *why* is the
  `factory: escalation` comment on the same ticket (see "Escalation
  comments").

- The **`hold-plan`** and **`hold-merge`** labels — human-owned. The
  human puts one on a ticket to hold that ticket's plan or merge gate
  whatever the run's mode (see "Per-ticket holds"). `orchestrate` reads
  them and never adds or removes either.

All four are workspace labels. `needs-attention` and `approved-to-merge`
exist on every team automatically; the hold labels are created by the
human when first wanted, and a workspace without them simply has no
held tickets. Move tickets and add labels yourself as they advance,
except the hold labels; a GitHub automation may race you to `Done` on
merge, which is harmless.

Take a label off when it stops being true: `needs-attention` comes off
when the ticket is picked back up, or when a `factory: decision` is
posted for it, and `approved-to-merge` comes off when the ticket
reaches `Done`. A stale label on a finished ticket is noise in
everyone's views. The hold labels are exempt: they stay until the human
removes them, `Done` included.

**You never pick from `Backlog`** — see Phase 1.

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
    Stage: planning | implementing | review round <n> of <budget> | merge
    Result: unplannable | blocked | failed | capped | dropped
    PR: <url or none>
    Question: <the decision, as one question>
    Found: <1-2 sentences, from the report's NOTES>
    Options (verbatim from the <planner | implementer | reviewer | fixer
             | adversarial-review | resolver | intent review> report, or
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
    This merge waits for a human to clear this PR specifically, whatever
    mode a run is in, when Entries names any. An `Entries: none` hold
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
planning-stage stop, since none of the three invocations replans: the
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

A decision marker is not merge approval and not a released gate.
Posting one clears `needs-attention` (see "Statuses and labels") and
does nothing else: it never clears a stop-list hit, and it never lets a
capped, blocked, or failed ticket merge under a released gate — "The
stop-list" and Phase 8's rules on both stand exactly as written. A
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

- Planner unplannable or blocked (a dropped ticket): `orchestrate`, in
  Phase 3.
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
way "Models and effort" is for models — `implement-ticket`,
`adversarial-review`, and `merge-queue` point back here rather than
each carrying their own copy of the templates.

## Context rules (non-negotiable)

- Never read a diff, a test log, or a CI log into your own context.
  Subagents summarize; you get ticket id, branch, PR URL, a verdict,
  and counts.
- Review findings live as comments on the pull request, never pasted
  into Linear or into your context beyond a one-line-per-finding
  summary.
- Per-run state — worktree paths, branches, PR URLs, CI attempt counts,
  review round counts, plan-time stop-list hits, how each gate was
  passed, the run's mode and gate configuration — lives in your own
  context for the life of the run, which is the only span it is needed
  for. Nothing here is written to a file. Whatever must outlive the run
  is already written where it belongs: PR link and status on the
  Linear ticket, the `needs-attention`/`approved-to-merge` labels, and
  the escalation comment a stage posts when it stops (see "Escalation
  comments").

## Phase 1 — Pick

Read the queue for the configured Linear team. Candidates are tickets
in `Todo`, sorted by priority.

Within `Todo`, skip anything blocked by an open ticket, anything
carrying the `needs-attention` label (a human is meant to look at that
first), anything too vague to implement without a human, anything
whose scope is plainly a project rather than a change, and a ticket
whose newest marker is a pending `factory: decision` (see "Escalation
comments") — its next step belongs to whatever resumes it, and picking
it here too would run it twice. Use judgment: a
slightly lower-priority ticket that unblocks others, or rounds out a
coherent batch, can jump the line. Pick 5–10 — or everything eligible
in `Todo` if it holds fewer. A short batch is a normal outcome, not a
reason to reach further.

Note which picks carry `hold-plan` or `hold-merge` (see "Per-ticket
holds"), in your own context. A hold label is not a reason to skip a
ticket.

A ticket sitting `In Progress` or `In Review` with a deferral comment
(see "The write window") is not picked up here — Phase 1 only draws
from `Todo`, and orchestrate never restarts a deferred ticket itself, in
any mode. Restarting one is a human's deliberate act: they re-invoke
the stage directly against the ticket or PR, or record their answer as
a `factory: decision` for something outside this plugin to act on (see
"Restarting").

### `Backlog`

`Backlog` is where the human parks work they do not want started —
their own tickets, decisions they are still thinking through, things
deliberately set aside. Promoting a ticket to `Todo` is how they hand
it to orchestrate. **You never pick a `Backlog` ticket into a batch**, and
you never promote one yourself.

What you may do, at the selection gate and only there, is *propose*
promotions. If `Todo` yields **five or more** eligible tickets, don't
read `Backlog` at all — there is enough to run. If it yields fewer,
read `Backlog` and look for tickets that are plainly runnable now:
concrete enough to implement, unblocked, scoped as a change rather
than a project, and not obviously the human's own to decide — nothing
that reads as an open question, a direction call, or a ticket they
assigned themselves. When a ticket is borderline, leave it out; the
cost of a missed suggestion is nothing and the cost of nagging the
human about their own parked work is that they stop reading the list.

Propose enough to bring the batch to five or so, ranked, each with one
line on why it looks runnable. They are proposals, not picks: the
human promotes them, or tells you to. Only then do they join the
batch, and a promoted ticket gets a planner pass like any other.

If the selection gate is **released**, there is nobody to propose to:
leave `Backlog` untouched, and let a thin `Todo` be a short batch. An
empty `Todo` means there is nothing to run — say so and stop.

## Phase 2 — Selection gate

Present the picks and stop: a table of ticket, title, priority,
estimate, and one line of why-now drawn from the description, flagging
any pick that carries a hold label. Nothing
is planned yet, so there are no plan summaries here and no waves —
that is the next gate's job. Add, separately:

- **Proposed promotions from `Backlog`**, if you made any, clearly
  marked as not picked and not promoted, one line each on why they
  look runnable now.
- **What you skipped and why**, for anything a human would want to
  know about: blocked by an open ticket, carrying `needs-attention`,
  too vague, too big, or carrying a pending `factory: decision` (its
  resume belongs to whatever runs `Next`).

The human can drop picks, add tickets, promote something you proposed
(or promote something you didn't), or reorder priorities. Then they
approve the selection.

Do not spawn a planner, and do not move any ticket, until they do.
The approval covers this selection only.

If this gate is released, say in one line which tickets you took and
what you passed over, naming any pick that carries a hold label, and go
straight to Phase 3.

## Phase 3 — Plan each ticket

For each pick, spawn a planner subagent with `prompts/planner.md` —
strongest reasoning model available (see Models and effort below):
planning is scope judgment. Planners are read-only, so run them all in
parallel.

Each planner writes its plan into the ticket's own description under a
`## Plan` heading — replacing an existing `## Plan` section, touching
nothing above it — and returns a short report with a summary and the
files the change will touch. The description is the brief every later
subagent reads, so a plan there is read by construction; never put a
plan in a comment, where it sinks under later traffic.

Each planner also checks the files it expects to touch against the
repo's stop-list and reports what they hit. Hold it for the plan gate;
that's what you present there. At that same point, re-read each
ticket's labels (a human may have added one since Phase 1) and record
which carry `hold-plan` or `hold-merge`. The planner prompt is
unchanged; it knows nothing of the labels.

A planner that finds a ticket too vague, or big enough to be several
tickets, reports it unplannable instead of guessing. A planner that
finds the ticket's premise doesn't match the code — the bug it
describes doesn't reproduce, the feature already exists — reports it
blocked instead, and has already commented the mismatch on the ticket.
Drop both kinds from the batch and say why at the plan gate; they
read differently to the human (unplannable needs more scoping,
blocked needs someone to look at what the planner found) so don't
merge them into one line.

Both come back with options and a pick attached. Label the dropped
ticket `needs-attention` and post a `factory: escalation` comment
(see "Escalation comments") with stage `planning`, result `unplannable`
or `blocked`, and the planner's `OPTIONS` verbatim — that is what turns
"WRIT-88 is unplannable" into a question somebody can answer in one
line, and the planner that wrote it is gone. The planner's own comment
on the ticket stays as is; this is in addition to it.

## Phase 4 — Plan the waves

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

A `hold-plan` ticket joins a wave only once its plan is approved (see
Phase 5), and a ticket that depends on it is serialized behind it.

## Phase 5 — Plan gate

Present the plans and stop: a table of ticket, wave, and the plan's
one-line summary, plus a note on anything serialized and why, which
tickets the planners flagged as stop-list hits and which entry each
one hit, which tickets are held at plan by label and which at merge by
label (see "Per-ticket holds"), and any tickets the planners dropped as
unplannable or blocked (say which, and for blocked, what the planner
found and the options it gave). Approving here approves the
plans — the full plans are on the tickets for anyone who wants the
detail. The human can still drop a ticket, or add one (an added ticket
gets a planner pass before it joins a wave).

Merges are cleared separately, ticket by ticket, at the merge gate —
though the human can pre-authorize a particular ticket's merge here if
they say so. A pre-authorization does **not** cover a stop-list
ticket, even one flagged right here: the list exists so that those
merges are decided after somebody has seen the diff, and nothing at
plan time has seen it yet. Nor does it cover a `hold-merge` ticket, for
the same reason.

Do not start any work until the human says yes. The approval covers
this batch only.

If this gate is released, say in one line what the waves are, any
stop-list hits, any tickets held at merge by label, and anything the
planners dropped, and go straight to Phase 6 with every ticket *not*
carrying `hold-plan`. Present each held plan — ticket, the plan's
one-line summary, "held at plan by label" — and wait on those alone; a
ticket whose plan the human approves joins the next wave. If the run
otherwise finishes with a held plan still unanswered, the run ends
there: the ticket stays in `Todo` with its plan written, not labelled
`needs-attention` (it is not stalled), and the end-of-run report names
it. The next run will pick it again and re-plan it, overwriting that
plan, until the human approves it in chat or removes the label.

## Phase 6 — Implement

Invoke the `implement-ticket` skill once for the current wave's ticket
ids together — its own instructions cover the worktree, the subagent,
the three-attempt CI rule, and running the wave concurrently, so
nothing here duplicates them.

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

## Phase 7 — Adversarial review

For each ticket `implement-ticket` reported green, invoke the
`adversarial-review` skill for that ticket's PR (its own instructions
cover the reviewer/fixer cycle and moving the ticket to `In Review`).

**Tell it its round budget**, which follows the mode:

- **supervised** → a hard cap of **six rounds**. A seventh is the
  human's call, not yours.
- **semi** and **autonomous** → **up to ten rounds at the reviewer's
  discretion**. Ten is a ceiling, not a target.

Either way `adversarial-review` stops early under its own progress
rule: two consecutive rounds that close nothing and break no new
ground, and it reports itself capped. That rule is the real limit and
the round count is only a backstop, so don't second-guess a cycle that
stopped at three — and don't read a round that turned up something new
as a cycle going wrong. A later reviewer reaching ground an earlier
one missed is the adversarial part working.

Six rather than four is deliberate: four to five rounds is the
observed normal for a change of ordinary size, and a cap that lands
inside the normal range spends a human on tickets that were about to
finish.

The human can name a different budget at invocation; theirs wins.

Every report comes back with a `STOPLIST` line. A non-empty one holds
that ticket's merge gate for the human whatever the mode. At the same
point, re-read the ticket's labels: a `hold-merge` label holds its
merge gate for the human whatever the mode, exactly like a non-empty
`STOPLIST`. Nothing is posted for it — no comment, no `needs-attention`;
the label is the record. On
`RESULT: ready`, `adversarial-review` has already posted the `factory:
stop-list hold` comment for it — every ready result gets one, `Entries:
none` included; carry it into Phase 8. On `RESULT: capped`, `blocked`,
or `failed`, it's instead recorded on that result's `factory:
escalation` comment, in its `Stop-list` line. On `RESULT: deferred`,
it's recorded on the deferral note's `Stop-list` line instead — see
"Escalation comments" and "The write window".

On `RESULT: ready`, go to Phase 8. Ready means no major and no medium
findings, not zero findings: open minors come back listed in the
report and are not a reason to hold the ticket. Pass them through to
the human as part of the ready report and leave them on the PR.
`adversarial-review` has already marked the PR ready for review
(un-drafted it), unless its NOTES say it left the PR draft — the write
window was closed, or `gh pr ready` itself errored — relay whichever
happened as part of the ready report.

On `RESULT: capped`, the ticket does not proceed to merge under any
gate configuration — leave it in `In Review`. `adversarial-review` has
already labelled it `needs-attention` and posted the `factory:
escalation` comment with the ledger rows and its options; relay those,
along with what each round fixed. Don't start another round yourself.

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

## Phase 8 — Merge gate and merge queue

A branch is ready when its latest review round returned no major and
no medium findings and CI is green. Report it: ticket, PR link, rounds
run, what the last round found, any minors left open, and whether it
hit the stop-list.

**Merge gate held** — the ticket rests in `In Review`, a reading gate,
until the human clears that ticket's merge: in chat, by adding the
`approved-to-merge` label in Linear themselves, or by having
pre-authorized it at the plan gate. Any of the three counts. Never
send a ticket to `merge-queue` without one of them.

**Merge gate released** — the human authorized these merges at
invocation, for this batch only. Say which tickets you are clearing
and add the `approved-to-merge` label yourself, then proceed. Tell
`merge-queue` plainly that the merge gate was released for this run,
so it is not left inferring where the approval came from. This
authority covers tickets that came back **ready** and nothing else: a
capped, blocked, or failed ticket is never cleared under it. It never
covers a `hold-merge` ticket either: don't add `approved-to-merge` to
one, and don't include it when telling `merge-queue` the merge gate
was released. Re-check the ticket's labels immediately before adding
`approved-to-merge` under a released gate.

**Stop-list hit** — the merge waits for the human whatever the mode,
including a run whose merge gate was released at invocation, and
including a ticket pre-authorized at the plan gate. Report it as ready
and say which entry it hit; the ticket rests in `In Review` until they
clear that specific PR. Don't label it `needs-attention`: nothing
stalled, it is waiting on approval like any other ready ticket, and
using the stall label here buries the tickets that actually stalled.
The `factory: stop-list hold` comment naming the entry is already on
the ticket, from `adversarial-review` (see "Escalation comments").

**Hold-merge label** — the merge waits for the human whatever the
mode, the same as a stop-list hit, including under a released merge
gate and for a ticket pre-authorized at the plan gate. Report it as
ready and say "held at merge by label"; the ticket rests in `In Review`
until the human clears it (adds `approved-to-merge`, approves in chat,
or approves the PR on GitHub — or removes the label). Don't label it
`needs-attention`, for the same reason as above.

Clearing a ticket makes it eligible; it does not set the order. Five
may come clear at once — invoke the `merge-queue` skill with all of
them together and let it work out sequencing and conflicts; its own
instructions cover ordering, rebasing, mechanical conflict resolution,
and the squash merge, so nothing here duplicates them.

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
merging, because the write window was closed — a released merge gate
does not open the window, so this can happen on a batch whose merges
were already authorized. Nothing merged, but the PR is not necessarily
untouched: `merge-queue` keys stuck versus ordinary on the state of the
head when the window closed, not on whether a push happened. If a
rebased head has been force-pushed and isn't (yet) green — red, or CI
still running — `merge-queue` treats that as stuck rather than merely
deferred: such a head fails its own Eligibility check on its own, so no
later `merge-queue <PR>` call can pick it back up by re-running the
resolver — and it will already have added `needs-attention` to the
linked ticket, posted the escalation comment for it (see "Escalation
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

Phases name capability tiers, not vendor models: planning and every
review round want the strongest reasoning model available, because
both are judgment; implementing to a written plan, fixing findings,
and resolving a mechanical rebase conflict are all mid-tier work.
The intent review a tier-2 rebase triggers is judgment too — it asks
whether a resolution dropped somebody's meaning — so it goes in the
strong tier despite being a single scoped pass.

Effort high everywhere. Map by harness — this table is the canonical
reference; `implement-ticket`, `adversarial-review`, and `merge-queue`
all point back to it:

| Phase                     | Claude Code  | Antigravity            | Codex                     |
| ------------------------- | ------------ | ---------------------- | ------------------------- |
| Plan                      | Opus, high   | gemini-3.7-flash, high | strongest available, high |
| Implement / fix / rebase  | Sonnet, high | gemini-3.7-flash, high | mid-tier, high            |
| Review (all rounds)       | Opus, high   | gemini-3.7-flash, high | strongest available, high |
| Intent review (tier 2)    | Opus, high   | gemini-3.7-flash, high | strongest available, high |

If your harness cannot set a subagent's model, run everything at the
session model and say so at the selection gate — degrade loudly, never
silently. In a mode where the selection gate is released, say it in
the line you echo the mode back with.

## Status updates

After each merge, and whenever the human asks: one table from what you
hold — ticket, title, where it is (implementing / review round N /
fixing / ready for merge / queued to merge / merged / needs attention
/ deferred (window closed)), PR link.
Nothing else; the details live on the PRs.

At the end of a run, list any deferred tickets separately, one line
each: which stage stopped, the worktree path, and how to pick it back
up — since the table's status alone doesn't tell the human that. For
`implement-ticket`, `adversarial-review`, and an ordinary `merge-queue`
deferral, that's the exact invocation that restarts it —
`implement-ticket <TICKET>`, `adversarial-review <PR>`, or
`merge-queue <PR>`. For the one `merge-queue` case that comes back both
deferred and stuck (Phase 8) — a rebased, force-pushed head that isn't
green — a bare restart can't fix it: say instead that it needs a human
to get the rebased head's CI green, then re-run `merge-queue <PR>`.
Either way, that action is theirs to make (see "Restarting"); this run
never restarts one itself.

At the end of a run, also list any ticket still held at plan or at
merge by label, one line each, with what clears it: for a plan hold,
the human's approval in chat or removing `hold-plan`; for a merge
hold, `approved-to-merge`, an approving GitHub review, or removing
`hold-merge`.

A run in semi or autonomous mode reports more, not less, because
nobody is watching it happen: post a status update at the end of each
phase, so the transcript reads as a record of what was decided on the
human's behalf.

When anything is waiting on the human — at the end of a run, when they
ask what needs them, or when a run in any mode has accumulated
escalations — invoke `decision-queue` instead of listing them
yourself. The table above says where work is; it doesn't say what to
do about the parts that stopped, and a human reading a status table
has to reconstruct every decision from the PR. `decision-queue` reads
Linear and the open PRs, not your context — pass it this batch's wave
plan so it can count the tickets serialized behind each item. Stops
reach it only through the escalation comments the stage skills already
posted (see "Escalation comments"), which is also why posting them
promptly matters. A ticket that's ready and simply waiting at a *held*
merge gate, with no stop-list hit, isn't a stop and doesn't belong in
its queue — it stays in your status table above. A stop-list hit is
different: `adversarial-review` has already posted the `factory:
stop-list hold` comment for it (see "The stop-list"), and
`decision-queue` renders that as a queue item of its own — expect it
there too, not only in this table.

## Escalation

The planner, `implement-ticket`, `adversarial-review`, and
`merge-queue` each handle their own stall — a bad report from any of
them already means the ticket's been commented on (planner and
implement-ticket comment directly for a blocked report) and left
somewhere sane: labelled `needs-attention` for an implement
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
`merge-queue` case that comes back both deferred and stuck (Phase 8),
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
