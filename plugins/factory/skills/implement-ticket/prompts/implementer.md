# Implementer brief

Fill in before spawning: TICKET (Linear id), WORKTREE (absolute path),
BRANCH (use Linear's suggested branch name for the ticket), BASE (the
repo's base branch, from its `## Dispatch` config), CHECK (that
config's check command), and WINDOW (that config's write window).

---

Implement TICKET, nothing more.

Work only in WORKTREE. It is a detached worktree at `origin/BASE`; it
shares the git directory with other concurrent work, so never check out
a branch by bare name — stay detached and push with
`git push origin HEAD:BRANCH`.

Read TICKET in Linear. Its description — the `## Plan` section if it
has one, otherwise the whole of it — is the brief.

Read the repo's `AGENTS.md`, including its `## Dispatch` section, and
then any convention documents `AGENTS.md` points to (often something
like `VISION.md` or `ARCHITECTURE.md`, but follow what this repo
actually names). Those documents are the fence around this project. A
document this repo doesn't have is not an error — skip it and move on.
`AGENTS.md`'s `## Dispatch` section also lists the invariants this
repo's reviewers hold changes to; read them now rather than
rediscovering them in review.

If the brief is too vague to implement, or conflicts with those
documents, stop: comment on TICKET saying why and report back as
blocked. Do not invent your own brief.

Stop and report back as blocked, the same way, if what you find in the
code doesn't match the brief's assumptions — it describes a function,
file, or behavior that's since changed or never existed, a concurrent
ticket already did this, or implementing it as written would plainly
break something the brief doesn't mention. Also stop as blocked if you
notice a real bug outside the ticket's scope — comment what you found
on TICKET; don't fix it as a drive-by and don't silently ignore it. In
every case: comment the specific mismatch on TICKET, don't guess your
way past it.

When you do, give the decision its options. You have the worktree and
have just read the surrounding code; you are about to be discarded,
and whoever picks this up rebuilds that context from nothing unless
you spend three more lines now. Two or three options for what to do,
the consequence of each, and which you would pick. Put them in the
comment on TICKET as well as in your report. A blocked report that
names the mismatch and stops there is half a report.

Implement the change. Match the surrounding code. Before pushing, the
repo's checks must pass locally — run CHECK, the check command from
the repo's `## Dispatch` config, and get it clean. If CHECK was not
given to you, or the repo's config marks it as not yet filled in, stop
and report back as blocked: "green" has no meaning without it, and
inventing a command that happens to pass is worse than stopping.

## The write window

WINDOW is this repo's declared write window: the closed periods,
if any, during which a commit, a push, or `gh pr create` must not
happen. If WINDOW is `none`, skip this section — there is nothing to
check. If it names no timezone, or can't be read unambiguously, stop
now and report back as blocked, the same way a missing CHECK does.

Otherwise, before every commit-creating command — the first commit,
any later fix or CI-repair commit, and any `--amend`, not just the
first one you happen to make — before every push, and before
`gh pr create`, run `TZ=<zone> date '+%u %H:%M'` fresh — never reuse
an earlier reading — and check it against WINDOW. Do this again before
every CI-retry push and every CI-fix commit, not just the first one: a
CI watch can take minutes on its own, long enough for the window to
close underneath it.

If the window is closed at any of those points: don't make the
commit, don't push, don't open the PR, and don't wait for it to
reopen. Stop exactly where you are, leave the worktree as it is —
uncommitted edits and all — and report back `RESULT: deferred`
instead of `blocked` or `failed`. In NOTES, say exactly what's left
undone (uncommitted edits, unpushed commits, no PR yet). This is not a
mismatch with the brief and not a stall; it needs no comment on
TICKET. There is no resume: a deferred ticket is picked up later by a
fresh implementer run, starting over with a fresh three-push budget,
not by continuing this attempt.

## Commit, push, and open the PR

Commit your change (checking the window fresh first, as above, for
this commit and any later one), then push and open a **draft** PR
(`gh pr create --draft`) titled `TICKET: <short description>` — the
title becomes the squash-merge subject on BASE, so the ticket id must
be in it. Watch CI with `gh pr checks --watch`.

If CI fails: read the failure, fix it — checking the window before
that commit too — and push again. You get **three pushes that reach
CI**. If the third is still red, the problem is systemic, not a
typo — stop and report back as failed rather than thrashing. Unlike
blocked, failed means you tried the documented path and hit a
mechanical wall, not a mismatch that needs a judgment call.

Report back to the orchestrator in exactly this shape, and keep it
under ~20 lines — no diffs, no logs:

    TICKET: <id>
    RESULT: green | blocked | failed | deferred
    PR: <url, if one was opened>
    BRANCH: <name>
    SUMMARY: <2-3 sentences: what changed, where>
    FILES: <paths touched>
    OPTIONS: <for blocked: 2-3 options with consequences, and your pick>
    NOTES: <anything you know is weak, why blocked, why it failed, or —
           for deferred — what's left undone>
