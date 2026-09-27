---
name: dispatch
description: Batch-run the next tickets from Linear for the repo you are in. Picks 5–10 unblocked tickets from `Todo` by priority, stops for human approval on the selection, has each one planned into its ticket description, plans parallel vs serial execution, stops for approval again on the plans, then runs each ticket through the implement-ticket, adversarial-review, and merge-queue skills to a merged pull request. Its three gates — selection, plan, merge — are set by naming a mode at invocation: supervised, semi, or autonomous. Reads the host repo's `AGENTS.md` `## Dispatch` section for its Linear team, check command, base branch, write window, and worktree locations, and stops a stage short of any commit, push, or merge while that window is closed. Use when asked to run the queue, work the next tickets, dispatch a batch, or process Linear tickets in parallel. Do not use for a single ticket a human is already driving — use implement-ticket, adversarial-review, or merge-queue directly for that.
---

# Dispatch

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
against a repo no dispatch run is touching. Read their SKILL.md files
before changing what a stage does; change this file for how runs are
queued. These files call all five by their bare names throughout; in
Claude Code they arrive as the `factory` plugin, so invoke them
namespaced — `/factory:implement-ticket` and so on. See "Where this
skill lives".

## Repo configuration (read this first)

Nothing in this pipeline is hardcoded to a particular project. Every
repo-specific value comes from a `## Dispatch` section in the host
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
- **Run manifest** — the file this skill keeps its per-ticket state in.
- **Review invariants** — the properties a reviewer should be
  adversarial about in this repo. These are the substance of review;
  the reviewer prompt carries no invariants of its own.
- **Stop-list** — the paths and subjects whose merge always waits for
  a human, whatever mode the run is in. See "The stop-list".
- **Write window** — the weekday hours, if any, during which a
  commit, push, or merge is not allowed to happen. Required; a repo
  with none declares `none`. See "The write window".

If `AGENTS.md` has no `## Dispatch` section, this repo has not opted
in: say so and stop, rather than guessing a team key, a check command,
or a place to put worktrees. If the section exists but a field is
missing or marked as not yet filled in, say which field and stop
before the point where you would need it — never substitute a guess.

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

## Modes

Gates are the mechanism; modes are names for the three settings of
them worth having. Naming one at invocation — "dispatch in autonomous
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
keep going. The manifest records how each gate was passed: approved by
the human, or taken under a released gate.

The human can re-hold a released gate at any point mid-run, and
release one only by saying so. When in doubt mid-run, the gate is
held.

## The stop-list

Some changes stop for a human however the run is configured. The repo
declares which in its `## Dispatch` section — paths, plus subjects
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
of planning. Record both in the manifest.

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
gate the same way.

## The write window

A repo's `## Dispatch` section declares a **write window** — when
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

**Required field.** A `## Dispatch` section with no **Write window**
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
That is the whole record. There is no field-by-field state to preserve
beyond it, because nothing carries over — see "Restarting" below.

**Restarting.** There is no automatic resume, and `dispatch` never
restarts a deferred ticket itself — Phase 1 draws only from `Todo`
(see "Statuses and labels"). A deferred ticket sits where it stopped
until a human deliberately picks it back up, by directly re-invoking
whichever of `implement-ticket <TICKET>`, `adversarial-review <PR>`,
or `merge-queue <PR>` deferred — and that stage starts from the start,
with fresh budgets: a fresh three CI pushes, a fresh round count, a
fresh ledger. A run's end-of-run report names each deferred ticket
with that exact invocation (see "Status updates").

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

The canonical copy of `dispatch`, `implement-ticket`,
`adversarial-review`, `merge-queue`, and `decision-queue` lives once,
in the `mattwalters/skills` plugin marketplace repo, at
`plugins/factory/skills/<name>/`. There is deliberately no per-project
fork to keep in sync. Edit it there, and the change reaches every repo
once it merges and each repo's installed marketplace is updated.

Two mechanisms carry it into a project repo:

- **Claude Code** gets it as the `factory` plugin from the `mattwalters`
  marketplace. Each project repo declares the marketplace and enables
  the plugin in a tracked `.claude/settings.json`, so a clone wires
  itself up. A plugin namespaces its skills, so in Claude Code these
  five are invoked as `/factory:dispatch`, `/factory:implement-ticket`,
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

A repo opts in by having a `## Dispatch` section in its `AGENTS.md`.
That section, and nothing else, is what makes these skills applicable
to it.

## Statuses and labels

Only Linear's stock statuses are used, plus two workspace labels:

- `Todo` — the queue this skill draws new picks from, and the only
  one for that. A ticket sitting `In Progress` or `In Review` with a
  deferral comment is never picked up from here — see Phase 1 — and
  dispatch never re-picks it itself either way.
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
  teleported somewhere else.

Both are workspace labels, so they exist on every team automatically.
Move tickets and add labels yourself as they advance; a GitHub
automation may race you to `Done` on merge, which is harmless.

Take a label off when it stops being true: `needs-attention` comes off
when the ticket is picked back up, `approved-to-merge` comes off when the
ticket reaches `Done`. A stale label on a finished ticket is noise in
everyone's views.

**You never pick from `Backlog`** — see Phase 1.

## Context rules (non-negotiable)

- Never read a diff, a test log, or a CI log into your own context.
  Subagents summarize; you get ticket id, branch, PR URL, a verdict,
  and counts.
- Review findings live as comments on the pull request, never pasted
  into Linear or into your context beyond a one-line-per-finding
  summary.
- Keep a run manifest — a small file, one entry per ticket: status,
  worktree path, branch, PR URL, CI attempt count, review round count,
  stop-list hits at plan time and at review time, and how each gate it
  passed was passed. For anything that stopped, keep the options and
  recommendation its subagent reported — that is what `decision-queue`
  renders, and re-deriving it later costs a subagent that has the
  context this one had. Record the run's mode and gate configuration
  at the top. It goes where the repo's `## Dispatch`
  section says, next to the worktrees. Update it as things happen; it
  is what status updates and resumption read from.

## Phase 1 — Pick

Read the queue for the configured Linear team. Candidates are tickets
in `Todo`, sorted by priority.

Within `Todo`, skip anything blocked by an open ticket, anything
carrying the `needs-attention` label (a human is meant to look at that
first), anything too vague to implement without a human, and anything
whose scope is plainly a project rather than a change. Use judgment: a
slightly lower-priority ticket that unblocks others, or rounds out a
coherent batch, can jump the line. Pick 5–10 — or everything eligible
in `Todo` if it holds fewer. A short batch is a normal outcome, not a
reason to reach further.

A ticket sitting `In Progress` or `In Review` with a deferral comment
(see "The write window") is not picked up here — Phase 1 only draws
from `Todo`, and dispatch never restarts a deferred ticket itself, in
any mode. Restarting one is a human's deliberate act: they re-invoke
the stage directly against the ticket or PR (see "Restarting").

### `Backlog`

`Backlog` is where the human parks work they do not want started —
their own tickets, decisions they are still thinking through, things
deliberately set aside. Promoting a ticket to `Todo` is how they hand
it to dispatch. **You never pick a `Backlog` ticket into a batch**, and
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
estimate, and one line of why-now drawn from the description. Nothing
is planned yet, so there are no plan summaries here and no waves —
that is the next gate's job. Add, separately:

- **Proposed promotions from `Backlog`**, if you made any, clearly
  marked as not picked and not promoted, one line each on why they
  look runnable now.
- **What you skipped and why**, for anything a human would want to
  know about: blocked by an open ticket, carrying `needs-attention`,
  too vague, or too big.

The human can drop picks, add tickets, promote something you proposed
(or promote something you didn't), or reorder priorities. Then they
approve the selection.

Do not spawn a planner, and do not move any ticket, until they do.
The approval covers this selection only.

If this gate is released, say in one line which tickets you took and
what you passed over, and go straight to Phase 3.

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
repo's stop-list and reports what they hit. Record it on the ticket's
manifest entry; it is what you present at the plan gate.

A planner that finds a ticket too vague, or big enough to be several
tickets, reports it unplannable instead of guessing. A planner that
finds the ticket's premise doesn't match the code — the bug it
describes doesn't reproduce, the feature already exists — reports it
blocked instead, and has already commented the mismatch on the ticket.
Drop both kinds from the batch and say why at the plan gate; they
read differently to the human (unplannable needs more scoping,
blocked needs someone to look at what the planner found) so don't
merge them into one line.

Both come back with options and a pick attached. Keep them in the
manifest verbatim — they are what turns "WRIT-88 is unplannable" into
a question somebody can answer in one line, and the planner that wrote
them is gone.

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

## Phase 5 — Plan gate

Present the plans and stop: a table of ticket, wave, and the plan's
one-line summary, plus a note on anything serialized and why, which
tickets the planners flagged as stop-list hits and which entry each
one hit, and any tickets the planners dropped as unplannable or
blocked (say which, and for blocked, what the planner found and the
options it gave). Approving here approves the
plans — the full plans are on the tickets for anyone who wants the
detail. The human can still drop a ticket, or add one (an added ticket
gets a planner pass before it joins a wave).

Merges are cleared separately, ticket by ticket, at the merge gate —
though the human can pre-authorize a particular ticket's merge here if
they say so. A pre-authorization does **not** cover a stop-list
ticket, even one flagged right here: the list exists so that those
merges are decided after somebody has seen the diff, and nothing at
plan time has seen it yet.

Do not start any work until the human says yes. The approval covers
this batch only.

If this gate is released, say in one line what the waves are, any
stop-list hits, and anything the planners dropped, and go straight to
Phase 6.

## Phase 6 — Implement

Invoke the `implement-ticket` skill once for the current wave's ticket
ids together — its own instructions cover the worktree, the subagent,
the three-attempt CI rule, and running the wave concurrently, so
nothing here duplicates them.

For each report it returns, update that ticket's manifest entry
(worktree path, branch, PR URL). On `RESULT: blocked` or `failed`,
`implement-ticket` has already labelled the ticket `needs-attention`
(commenting the mismatch itself if blocked); relay which one it was —
blocked means the brief needs a human's judgment call, failed means CI
never went green — and carry on with the rest of the batch. One stuck
ticket never stops the others.

On `RESULT: deferred`, `implement-ticket` stopped before a gated write
because the repo's write window was closed, and no `needs-attention`
label was added. What's left undone varies by when it stopped —
uncommitted edits, unpushed commits, a branch pushed with no PR opened
yet, or a PR already open — and its NOTES say which; don't assume it
means nothing was committed or pushed. A restart handles the
pushed-but-no-PR case on its own — the one worktree rule continues
from the pushed branch, and the implementer opens the PR it never got
to — so there's nothing extra for you to do about it beyond relaying
it here. Record what's undone and the worktree path on the manifest
entry and carry on with the rest of the batch; don't retry it within
this run — a restart is a fresh `implement-ticket` invocation with a
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
that ticket's merge gate for the human whatever the mode — record it
on the manifest entry and carry it into Phase 8.

On `RESULT: ready`, go to Phase 8. Ready means no major and no medium
findings, not zero findings: open minors come back listed in the
report and are not a reason to hold the ticket. Pass them through to
the human as part of the ready report and leave them on the PR.

On `RESULT: capped`, the ticket does not proceed to merge under any
gate configuration — leave it in `In Review`, label it
`needs-attention`, and report exactly what `adversarial-review`
reported: findings summary, what each round fixed, and the ledger rows
behind its read. Don't start another round yourself.

On `RESULT: blocked` (a reviewer or fixer found something outside the
diff itself) or `failed` (the fixer exhausted its CI attempts), say
which one it was, relay the options the subagent reported, and carry
on with the rest of the batch.

On `RESULT: deferred`, the fixer stopped before a commit or push
because the write window was closed. Leave the ticket in `In Review`
with no `needs-attention` label, keep the ledger rows
`adversarial-review` reported in NOTES for the human's reference (the
ledger itself dies with that subagent's context), record what's undone
and the worktree path on the manifest entry, and carry on with the rest
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
capped, blocked, or failed ticket is never cleared under it.

**Stop-list hit** — the merge waits for the human whatever the mode,
including a run whose merge gate was released at invocation, and
including a ticket pre-authorized at the plan gate. Report it as ready
and say which entry it hit; the ticket rests in `In Review` until they
clear that specific PR. Don't label it `needs-attention`: nothing
stalled, it is waiting on approval like any other ready ticket, and
using the stall label here buries the tickets that actually stalled.

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
new logic or a judgment call, not just combining both sides) or
`failed` (CI never went green on the rebased head), tell the human
which one it was and why — `merge-queue` already left the PR and
worktree as they were — and carry on with the rest of the queue.

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
linked ticket and said so in its report. Relay that plainly: a human
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

After each merge, and whenever the human asks: one table from the
manifest — ticket, title, where it is (implementing / review round N /
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
the same manifest and renders those as ordered questions with the
options their subagents already recorded. Keeping the manifest's
options fields filled in is what makes that possible.

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

Every one of those reports carries the options its subagent saw and
the one it would pick. Relay them and keep them in the manifest — the
subagent wrote them while it still had the worktree and the diff in
front of it, which is the only moment they are cheap, and they are
what `decision-queue` turns into an answerable question later.

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
