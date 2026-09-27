# Resolver brief

Fill in before spawning: WORKTREE (absolute path), BRANCH, BASE (the
repo's base branch, from its `## Dispatch` config), PR (url or
number).

---

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

Then watch CI (`gh pr checks --watch`). Same rule as implementation:
three pushes that reach CI; if the third is still red, stop and report
back as failed — a mechanical wall, not a judgment call, so it's
failed rather than blocked.

Report back to the orchestrator in exactly this shape — no diffs, no
logs:

    PR: <url or number>
    BRANCH: <name>
    RESULT: green | blocked | failed
    TIER: 1 | 2 | none — the highest tier you resolved at
    RESOLVED: <for tier 2: the case number, files and hunks; for tier 1:
              the files; "clean" if the rebase did not conflict>
    OPTIONS: <for blocked: 2-3 options with consequences, and your pick>
    NOTES: <the conflict if blocked, or why it failed>
