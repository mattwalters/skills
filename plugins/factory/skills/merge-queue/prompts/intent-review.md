# Intent review brief

Fill in before spawning: PR (url or number), BRANCH, BASE (the repo's
base branch), WORKTREE (absolute path), and RESOLVED — the tier-2
case, files, and hunks the resolver reported.

---

A rebase of BRANCH onto BASE resolved conflicts in RESOLVED beyond the
purely mechanical. Read those hunks and answer one question:

**Did the resolution drop anyone's intent?**

That is the whole review. Both sides of each conflict meant something.
The question is only whether both meanings survived — not whether the
code is good, not whether the change is well designed, not whether you
would have written it this way. Those were settled by the adversarial
review this branch already passed. The only thing that has happened
since is the rebase.

Read each resolved hunk three ways: what BRANCH's side was doing
before the rebase, what BASE's side was doing, and whether the
resolved text still does both. Use `git log`, `git show`, and the PR's
own history to see the two sides as they were.

The failures worth catching:

- One side's edit silently absent from the result.
- A rename or signature change applied to some uses and not others.
- An edit reapplied at a location where it no longer does the same
  thing.
- Both sides present, but in an order or nesting that changes
  behavior.

Read the resolved hunks and only as much around them as judging those
requires. Do not review the rest of the diff. A finding elsewhere in
the PR is out of scope here: put it in one line under NOTES and do not
let it change the verdict.

There is no second round and no fixer behind you. `intact` sends this
PR to merge; `dropped` sends it to a human. Say which, and do not
hedge between them — if you cannot tell whether an intent survived,
that is `dropped`, and NOTES is where you say what you could not
determine.

For `dropped`, also give the decision its options, the same way any
other stopping subagent in this pipeline does: two or three ways to
resolve it — re-do the rebase by hand, restore the dropped side and
re-check the other, or something specific to what you found — with the
consequence of each, and which you would pick. Whoever reads this
report is deciding without having reopened the rebase themselves.

Report back in exactly this shape — no diffs, no logs:

    PR: <url or number>
    RESULT: intact | dropped
    CHECKED: <the hunks you read>
    OPTIONS: <for dropped: 2-3 options with consequences, and your pick>
    NOTES: <for dropped: which hunk, whose intent, and what is missing,
           precisely enough to act on without reopening the rebase>
