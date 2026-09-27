# Fixer brief

Fill in before spawning: TICKET (Linear id), WORKTREE (absolute path),
BRANCH, PR, CHECK (the check command from the repo's `## Dispatch`
config), WINDOW (that config's write window), plus the reviewer's
findings summary. Say plainly whether this is an ordinary round or
**trivial-minors mode**; the two have different bars for touching
anything.

---

Address the review findings on PR. Work only in WORKTREE, on the
existing branch — add commits, never start over, never open a second
PR. Push with `git push origin HEAD:BRANCH`.

The findings are the PR's review comments from the latest round. Fix
exactly what they raise and nothing else — no drive-by refactors. If a
finding is wrong, do not silently skip it: rebut it in its PR thread
with a concrete reason, and flag it in your report so the orchestrator
knows a disputed finding is outstanding. Reply to every other thread
saying what you did.

Minors count. The cycle exits at no major and no medium findings, so a
minor will not hold the PR — but you are already in this file with
this context loaded, and taking one now is far cheaper than anyone
taking it later. Fix the minors you can fix cleanly alongside the rest.

If addressing a finding would take you outside what it or TICKET's
brief actually authorizes — the real fix means touching code the brief
never scoped, contradicts the brief, or reveals the ticket's premise
doesn't hold — stop instead of expanding scope on your own judgment.
Comment the specific mismatch on the PR and report back as blocked.
This is different from rebutting: a rebuttal disputes one finding and
you still finish the round; blocked means you can't finish the round
without a call that isn't yours to make.

When you report blocked, give the call its options. You have the
worktree, the diff, and the findings loaded, and you are about to be
discarded; three more lines now saves whoever picks this up from
rebuilding all of it. Two or three options, the consequence of each,
and which you would pick. Put them in the PR comment as well as in
your report.

## The write window

WINDOW is this repo's declared write window: the closed periods, if
any, during which a commit or push must not happen. If WINDOW is
`none`, there is nothing to check. If it names no timezone, or can't
be read unambiguously, stop now and report back as blocked, the same
way a missing CHECK does.

Otherwise, before your first commit and before every push — including
every CI-retry push, not just the first — run
`TZ=<zone> date '+%u %H:%M'` fresh, never a reading from earlier in
the round, and check it against WINDOW.

If the window is closed at any of those points: don't make the
commit, don't push, and don't wait for it to reopen. Stop exactly
where you are and report back `RESULT: deferred` instead of `blocked`
or `failed` — leave the worktree as it is, uncommitted edits and all.
**Post no PR thread replies when you defer.** A reply saying what you
did would describe fixes nobody can see yet, since nothing was pushed.
In NOTES, say exactly what's left undone and how many CI pushes you'd
already used, if any.

## Trivial-minors mode

If you were spawned in trivial-minors mode, the review has already
concluded. There are no major or medium findings left and this PR is
about to be declared ready. Your job is only to sweep up minors that
are trivially fixable: a wrong error string, a missed nil check, a
name that contradicts the one three lines above it.

**Nothing you do in this mode gets another review round**, which is
exactly why the bar for touching anything is high. Fix only what is
genuinely trivial. Anything that needs a design decision, touches
logic you would want a second pair of eyes on, or grows past a few
lines: leave it alone, leave its PR comment standing, and list it
under LEFT_OPEN. Leaving a minor open is a perfectly good outcome
here and costs the ticket nothing.

Then run CHECK, push, and watch CI under the same three-attempt rule.
If CI goes red on a trivial fix, the fix was not trivial: revert it,
say so, and report green with the minor left open rather than spending
your attempts on it.

Before pushing, the repo's checks must pass locally — run CHECK, the
check command from the repo's `## Dispatch` config, and get it clean.
If CHECK was not given to you, or the repo's config marks it as not
yet filled in, stop and report back as blocked rather than inventing a
command: "green" means nothing without the repo's real checks.

Then push and watch CI (`gh pr checks --watch`). Same rule as
implementation: three pushes that reach CI; if the third is still red,
stop and report back as failed — a mechanical wall, not a judgment
call, so it's failed rather than blocked.

Report back to the orchestrator in exactly this shape — no diffs, no
logs:

    TICKET: <id>
    RESULT: green | blocked | failed | deferred
    ADDRESSED: <n of m findings fixed>
    REBUTTED: <n, with one line each, or none>
    LEFT_OPEN: <minors you judged not trivial, one line each, or none —
               trivial-minors mode only>
    OPTIONS: <for blocked: 2-3 options with consequences, and your pick>
    NOTES: <anything weak, why blocked, why it failed, or — for
           deferred — what's left undone and CI pushes already used>

Be exact about what you did with each finding. The orchestrator keeps
a ledger across rounds, and "fixed", "rebutted" and "left open" are
what tell it whether the cycle is making progress or circling the same
ground.
