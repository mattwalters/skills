# Resolver brief

Fill in before spawning: WORKTREE (absolute path), BRANCH, BASE (the
repo's base branch, from its `## Dispatch` config), PR (url or
number), WINDOW (that config's write window). WORKTREE has already
been reset to the PR's current remote head before you were spawned —
treat this as a fresh attempt, whatever an earlier resolver on this PR
may have left behind.

---

## The write window

WINDOW is this repo's declared write window: the closed periods, if
any, during which the rebase, a commit-creating command, or a push
must not happen. If WINDOW is `none`, skip this section. If it names
no timezone, or can't be read unambiguously, stop now and report back
as blocked, the same way an unreadable BASE would.

Otherwise, before you run the rebase below, before `git rebase
--continue` if resolving a conflict needs it, before any commit you
make to fix a red CI run (see "Push and watch CI" below), and again
before every push (including every CI-retry push, not just the
first), run `TZ=<zone> date '+%u %H:%M'` fresh — never a reading from
earlier in this run — and check it against WINDOW.

If the window is closed at any of those points: don't run the rebase
(it creates a commit), don't run `--continue` (it stamps the rewritten
commit the same way), and don't push. If you are mid-rebase when it
closes, run `git rebase --abort` rather than leaving WORKTREE with a
rebase in progress — otherwise leave WORKTREE untouched. Stop exactly
where you are and report back `RESULT: deferred` instead of `blocked`
or `failed`. In NOTES, say whether the rebase ran, and, if you aborted
mid-conflict, which tier and files you'd resolved. There is no resume:
a later attempt on this PR starts over from a worktree reset to the
PR's remote head, not from here.

Rebase BRANCH onto current `origin/BASE` in WORKTREE. Work only there;
never touch another branch or worktree.

    git fetch origin && git rebase origin/BASE

If it completes without conflicts, skip to "Push and watch CI" below.

If it conflicts, resolve a file yourself only when the conflict falls
in one of the two tiers below. Everything else aborts. "I can see what
they probably meant" is not a tier.

## Tier 1 — mechanical

Both sides added unrelated content near the same lines: an import
block, a registration list, a switch or map gaining new entries,
adjacent but disjoint edits, a changelog or fixture appended to from
both sides. Keep both additions.

Nothing at this tier requires you to know what the code means.

## Tier 2 — one step beyond mechanical

Four cases, and only these four:

1. **Same function, disjoint regions.** Both sides edited one
   function, in regions that do not read or write the same values.
2. **A rename on one side, a use on the other.** One side renamed a
   symbol, field, or method; the other added or changed a use of the
   old name. Apply the rename to the new use.
3. **A signature change on one side, a call on the other.** One side
   added or reordered a parameter whose value at the new call site is
   unambiguous — a literal, a value already in scope, or the same
   default every other call site passes.
4. **A move on one side, an edit on the other.** One side moved code
   to another file or position unchanged; the other edited it in
   place. Apply the edit at the new location.

In all four, both sides' intent survives intact and you are applying a
transformation rather than making a choice. The moment you would have
to decide whose version is right, whether an edit still applies, or
what some value ought to be, it is not tier 2.

A tier-2 resolution gets a scoped review afterwards, so report which
of the four cases it was, in which files, at which hunks — precisely
enough that a reviewer finds them without asking you.

Once you've resolved the conflict at whichever tier applies, `git add`
the file and check the window again (see "The write window" above)
before running `git rebase --continue` — resolving can take a while on
its own, and `--continue` is what actually stamps the rewritten
commit's committer date. If the window closed while you were
resolving, don't run `--continue`: run `git rebase --abort` instead
and report `RESULT: deferred` as described above, noting the tier and
files in NOTES. If it's still open, run `git rebase --continue`, and
if another conflict follows, come back to this paragraph for it too.

## Everything else

If a conflict is neither tier 1 nor tier 2, or you are not confident
which it is:

    git rebase --abort

and stop. Report back as blocked with exactly which files and what the
conflict actually is. You are the only one who has looked at it and
you are about to be discarded, so spend three more lines: two or three
options for resolving it, the consequence of each, and which you would
pick. Whoever reads this rebuilds your context from nothing otherwise.

A wrong "mechanical" merge that silently drops one side's logic is
worse than surfacing it. A skipped merge costs somebody five minutes
in the morning; a quietly wrong one costs a day.

## Push and watch CI

Push the rebased branch (rebase rewrites history, so force with
lease):

    git push --force-with-lease origin HEAD:BRANCH

Then watch CI (`gh pr checks --watch`). If CI fails: read the failure,
fix it — checking the window before that commit too, same as any other
commit-creating command (see "The write window" above) — and push
again. Same rule as implementation: three pushes that reach CI; if the
third is still red, stop and report back as failed — a mechanical
wall, not a judgment call, so it's failed rather than blocked.

Report back to the orchestrator in exactly this shape — no diffs, no
logs:

    PR: <url or number>
    BRANCH: <name>
    RESULT: green | blocked | failed | deferred
    TIER: 1 | 2 | none — the highest tier you resolved at; n/a if you
          deferred before the rebase ran or aborted mid-rebase
    RESOLVED: <for tier 2: the case number, files and hunks; for tier 1:
              the files; "clean" if the rebase did not conflict; for a
              mid-rebase deferral, the tier and files you'd found
              before aborting>
    OPTIONS: <for blocked: 2-3 options with consequences, and your pick>
    NOTES: <the conflict if blocked, why it failed, or — for deferred —
           whether the rebase ran and, if a head was already pushed,
           whether its CI is green>
