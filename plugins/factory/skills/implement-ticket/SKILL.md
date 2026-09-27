---
name: implement-ticket
description: Take one or more Linear tickets from Todo/In Progress through a CI-green draft PR, using a fresh implementer subagent per ticket in an isolated git worktree. Reads the host repo's `AGENTS.md` `## Dispatch` section for its Linear team, check command, base branch, write window, and worktree directory, and stops the implementer short of any commit or push while that window is closed. Use when asked to implement a specific ticket, pick up a ticket, or "just do the implementation" without also running review or merge. Called by the dispatch skill once per ticket in a wave; equally fine invoked standalone against a single ticket.
---

# Implement ticket

Given one or more ticket ids, take each through a CI-green draft PR
using a fresh implementer subagent. Independent tickets run
concurrently — spawn every target's subagent in the same batch of tool
calls. You do not write code, read diffs, or debug CI yourself: the
subagent does, and reports back.

If invoked with no ticket id, ask which ticket(s) to implement rather
than guessing.

## Repo configuration

Read the `## Dispatch` section of the host repo's `AGENTS.md` before
starting: it gives the Linear team key, the **check command** the
implementer must pass locally before pushing, the **base branch**
worktrees are cut from, the **write window** during which a commit or
push is not allowed to happen (see the `dispatch` skill's "The write
window"), and the **worktree directory** (`<runs-dir>` below). Pass
all five down to the implementer verbatim.

No `## Dispatch` section means the repo has not opted into this
pipeline — say so and stop. A field that is missing, or marked as not
yet filled in, means stop before the step that needs it rather than
inventing a value; a check command or a write window in particular is
never to be guessed at — the whole "green" verdict rests on the one,
and a legal control rests on the other. `none` is a valid write-window
declaration and is not the same as a missing one.

## Per ticket

1. Move the ticket to `In Progress` in Linear, unless it's already
   there (a caller may have moved it during its own planning/approval
   step). If it carries the `needs-attention` label from an earlier
   run, remove it — someone is on it again.
2. Make an isolated worktree in the configured worktree directory:
   `git fetch origin && git worktree add --detach <runs-dir>/<TICKET> origin/<base-branch>`.
   Never let two tickets share one worktree. If a worktree already
   exists at `<runs-dir>/<TICKET>` — left by an earlier attempt that
   came back `deferred` — this is a fresh attempt with fresh budgets,
   not a continuation: reset it per the `dispatch` skill's "The write
   window" ("Restarting") before doing anything else — `git fetch
   origin && git reset --hard origin/<branch>` if the ticket already
   has an open PR, or remove it and cut a fresh one from
   `origin/<base-branch>` if it doesn't.
3. Spawn an implementer subagent with `prompts/implementer.md`, filling
   in TICKET, WORKTREE, BRANCH (Linear's suggested branch name for the
   ticket), BASE (the base branch), CHECK (the check command), and
   WINDOW (the write window) — mid-tier model, high effort (Sonnet on
   Claude Code; see the `dispatch` skill's Models and effort table for
   other harnesses).

The implementer's contract, enforced by the prompt: implement the
ticket — its `## Plan` section if the description has one, otherwise
the whole description — match the surrounding code, pass the repo's
declared check command locally, push, open a **draft** PR titled
`<TICKET>: <short description>` (the squash-merge subject line, so the
ticket id must be in it), and watch CI. It gets three pushes that
reach CI; a third red run means something systemic, and it reports
back failed instead of thrashing.

It also stops and reports back **blocked**, instead of guessing, if
what it finds doesn't match the brief: a stale assumption, a
concurrent ticket that already did this, or a real bug outside the
ticket's scope. Blocked and failed are not the same thing — blocked
means the brief needs a human's judgment call; failed means CI never
went green after three honest attempts. Keep that distinction when you
relay it upward; don't collapse both into one generic "it broke."

## On a blocked or failed report

For blocked, the implementer has already commented the specific
mismatch on the ticket, along with the options it saw and the one it
would pick; relay those upward with the mismatch, since they are what
makes the decision answerable by someone who hasn't read the code. For
failed, it hasn't commented — do that yourself. Either way: add the `needs-attention` label to the ticket —
leave its status where it is, the label is what flags it — and tell
whoever is waiting on this: the human if you were invoked standalone,
or your caller if a skill invoked you. Don't retry past the
implementer's own three-attempt budget, and don't let one stuck ticket
stop the others you were given.

## On a deferred report

The implementer stopped before a gated write — a commit, a push, or
`gh pr create` — because the repo's declared write window was closed
(see the `dispatch` skill's "The write window"). Nothing is wrong:
don't label the ticket, and leave its status exactly where it was.
Post the short deferral note the `dispatch` skill's "The write window"
defines, on the ticket, naming stage `implement`: what's left undone,
the worktree path, and when the window next opens. Tell whoever is
waiting on this the same thing. There is no resume — a later attempt,
whether run by a later `dispatch` batch or a human re-invoking this
skill directly, starts over from step 1 with a fresh three-push budget.

## Report, per ticket

Relay the implementer's report upward, unchanged:

    TICKET: <id>
    RESULT: green | blocked | failed | deferred
    PR: <url, if one was opened>
    BRANCH: <name>
    SUMMARY: <2-3 sentences: what changed, where>
    FILES: <paths touched>
    OPTIONS: <for blocked: 2-3 options with consequences, and its pick>
    NOTES: <anything known weak, why blocked, or why it failed>

`RESULT: green` means CI is green on a draft PR — it says nothing
about review. This skill never runs review and never merges; see
`adversarial-review` and, for the full pipeline, `dispatch`.
