---
name: research-ticket
description: Answer one or more planned `research`-labelled Linear tickets, using a fresh read-only researcher subagent per ticket that writes a `## Report` section into the ticket's description and reports reported, blocked or failed. Moves the ticket to `In Progress` while it works and to `In Review` on a report, where it waits for the human to accept it by moving it to `Done`; on blocked or failed, moves it back to `Todo` with the `needs-attention` label and a `factory: escalation` comment carrying the researcher's options. No worktree, commit, PR, review or merge. Reads the host repo's `AGENTS.md` `## Orchestrate` section for its Linear team and base branch. Also reads an optional `Telemetry` field; when it is set, posts one `research` row per ticket, and a failed post never blocks or changes the result. Skips, reporting it skipped, a ticket without the `research` label, without a `## Plan`, not in `Todo` or `In Progress`, or carrying `human-only`. Use when asked to research a ticket or answer a research ticket. Called by the orchestrate skill after its plan gate, and by a dispatcher for an approved research plan; equally fine invoked standalone against a single ticket.
---

# Research ticket

Given one or more ticket ids, answer each with a report written into the
ticket's own description, using a fresh researcher subagent.
Researchers are read-only, so spawn every target's researcher in the
same batch of tool calls. You do not read the material or write the
report yourself: the subagent does, and reports back.

A research ticket has a written report as its deliverable, not a diff.
There is no worktree, no branch, no PR, no review and no merge. See the
`orchestrate` skill's "Research tickets" for where these tickets sit in
the pipeline.

If invoked with no ticket id, ask which ticket(s) to research rather
than guessing.

## Repo configuration

Read the `## Orchestrate` section of the host repo's `AGENTS.md` before
starting. It gives the Linear team key and the **base branch**, the ref
the researcher reads the code at. Pass both down to the researcher
verbatim.

The section may also carry an optional **Telemetry** field, and the
caller may hand you a run id; see the `orchestrate` skill's
"Telemetry", which says how to resolve the run id when none is given.
Neither goes to the researcher, and a missing **Telemetry** is not a
missing field: it means telemetry is off.

No `## Orchestrate` section means the repo has not opted into this
pipeline: say so and stop. A field that is missing, or marked as not yet
filled in, means stop before the step that needs it rather than
inventing a value.

Research makes no commit, push or PR, so nothing it does is a gated
write, whatever the **write window** says. Editing a Linear description
or posting a comment never was one (see the `orchestrate` skill's "The
write window").

## Which tickets

Research a ticket only if all four hold:

- it carries the `research` label. Without it, its deliverable is a PR;
  point at `implement-ticket`.
- its description has a `## Plan` section. Without one, the researcher
  has no brief; point at `plan-ticket`.
- it is in `Todo` or `In Progress`. `In Review` means a report is
  already written and waiting on the human, and `Done` means they
  accepted it; researching again would overwrite that report. A human
  who wants it redone moves it back to `Todo` first.
- it does not carry `human-only`. That ticket is the human's own (see
  the `orchestrate` skill's "Statuses and labels").

Otherwise skip it with no status move, no label, no comment and no
telemetry row, and report `RESULT: skipped` with the reason in NOTES, so
whoever invoked you can account for it.

This skill does not check whether anyone approved the plan. Whoever
invokes it is the approval: `orchestrate` after its plan gate, a
dispatcher on `plan-approved`, or a human asking for it directly.

## Per ticket

1. Move the ticket to `In Progress`, unless it's already there. If it
   carries the `needs-attention` label from an earlier attempt, remove
   it: someone is on it again.
2. Spawn a researcher subagent with `prompts/researcher.md`, filling in
   TICKET, the team key and the base branch. Use the strongest
   reasoning model available at high effort (see the `orchestrate`
   skill's "Models and effort"). If **Telemetry** is set, record
   `started_at` first.
3. When the report arrives, post one `research` row with its `RESULT`
   word (`reported`, `blocked` or `failed`), then carry on with the
   handling below.

The researcher's contract, enforced by the prompt: answer what the
`## Plan` asks, reading the code at the base branch's remote ref. Write
the report into the ticket's description under a `## Report` heading
after `## Plan`. Replace an existing `## Report` and nothing else. Open
it with a 2–4 sentence summary, then the findings, each with its file
and line or URL, then the open questions.

## On a reported result

Move the ticket to `In Review`. The report is already in the
description, and the ticket rests there until the human accepts it by
moving it to `Done`. Nothing in the pipeline moves a research ticket to
`Done`, under any mode or gate.

## On a blocked or failed result

Blocked means the ticket's premise doesn't hold, or the question needs a
human's call before it can be answered. Failed means the research hit a
mechanical wall, such as an unreachable source or a denied tool. Keep
them distinct when you relay them.

Either way, the researcher has already commented the mismatch or wall
on the ticket. In the same step:

- move the ticket back to `Todo`, so that clearing the label makes it
  workable again;
- label it `needs-attention`;
- post a `factory: escalation` comment (see the `orchestrate` skill's
  "Escalation comments") with stage `researching`, the result, and the
  researcher's `OPTIONS` verbatim. A failed result that returned none
  records "none recorded".

A human answers it by editing the ticket, and its plan if the question
changed, and clearing `needs-attention`. It is never answered with a
`factory: decision`, since none of that marker's `Next` invocations
re-researches.

Tell whoever is waiting on this: the human if you were invoked
standalone, or your caller if a skill or dispatcher invoked you. Don't
let one ticket's trouble stop the others you were given.

## Report, per ticket

Relay the researcher's report upward, unchanged:

    TICKET: <id>
    RESULT: reported | blocked | failed | skipped
    SUMMARY: <the 2-4 sentence summary from the report>
    OPTIONS: <for blocked: 2-3 options with consequences, and its pick>
    NOTES: <the weakest part of the report, why blocked/failed, or why
           skipped>

A skipped ticket's report carries only TICKET, RESULT and NOTES.
