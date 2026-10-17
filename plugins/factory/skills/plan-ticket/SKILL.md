---
name: plan-ticket
description: Plan one or more Linear tickets into their own descriptions, using a fresh read-only planner subagent per ticket that writes a `## Plan` section and reports planned, unplannable or blocked, with the files the change will touch, the stop-list entries they hit, and whether it wrote a research plan. Skips a ticket that is not in `Todo` or `Backlog`, reporting it skipped. Removes `needs-attention` from a ticket it picks back up. Records an unplannable or blocked ticket itself, with the `needs-attention` label and a `factory: escalation` comment carrying the planner's options. Removes a stale `plan-approved` label before replanning. Reads the host repo's `AGENTS.md` `## Orchestrate` section for its Linear team, base branch, check command, review invariants and stop-list. Also reads an optional `Telemetry` field; when it is set, posts one `plan` row per ticket, and a failed post never blocks or changes the result. Use when asked to plan a ticket, write a plan, or replan one. Called by the orchestrate skill for every ticket in a batch, and by a dispatcher for a `Todo` ticket with no plan; equally fine invoked standalone against a single ticket.
---

# Plan ticket

Given one or more ticket ids, write a plan into each ticket's own
description using a fresh planner subagent. Planners are read-only, so
spawn every target's planner in the same batch of tool calls. You do not
read the code or write the plan yourself: the subagent does, and reports
back.

If invoked with no ticket id, ask which ticket(s) to plan rather than
guessing.

## Repo configuration

Read the `## Orchestrate` section of the host repo's `AGENTS.md` before
starting. It gives the Linear team key, the **base branch**, the
**check command**, the **review invariants** and the **stop-list**. Pass
all five down to the planner verbatim. The planner needs the check
command and the invariants so the plan says how the change will be
held, and it needs the stop-list to report which entries the change
hits.

The section may also carry an optional **Telemetry** field, and the
caller may hand you a run id; see the `orchestrate` skill's
"Telemetry", which says how to resolve the run id when none is given.
Neither goes to the planner, and a missing **Telemetry** is not a
missing field: it means telemetry is off.

No `## Orchestrate` section means the repo has not opted into this
pipeline: say so and stop. A field that is missing, or marked as not yet
filled in, means stop before the step that needs it rather than
inventing a value.

Planning writes nothing to the repo, so the **write window** does not
apply. Editing a Linear description or posting a comment is never a
gated write (see the `orchestrate` skill's "The write window").

## Which tickets

Plan a ticket in `Todo` or `Backlog`. A ticket in any other status is
already being worked, or is finished, and a new plan would change the
brief out from under whatever is working on it. Skip it with no label,
no comment and no telemetry row, and report `RESULT: skipped` with the
reason in NOTES, so whoever invoked you can account for it.

## Per ticket

1. Read the ticket's labels immediately before spawning; a human may
   have changed one since whoever invoked you last looked. If it carries
   `needs-attention`, remove it: someone is on it again. If it carries
   `plan-approved`, remove that too. That label approved the plan this
   run is about to replace, and an approval of a plan nobody has read
   approves nothing.
2. Spawn a planner subagent with `prompts/planner.md`, filling in
   TICKET and the configuration above. Use the strongest reasoning
   model available at high effort (see the `orchestrate` skill's
   "Models and effort"), because planning is scope judgment. Fill in
   the brief's optional RESEARCH line if and only if the ticket carries
   `research`, so the planner writes a research plan: the questions to
   answer and where to look, with `FILES: none` and `STOPLIST: none`.
   If **Telemetry** is set, record `started_at` first.
3. When the report arrives, post one `plan` row with its `RESULT` word
   (`planned`, `unplannable` or `blocked`), then carry on with the
   handling below.

The planner's contract, enforced by the prompt: write the plan into the
ticket's description under a `## Plan` heading. Replace an existing
`## Plan` section and nothing else, and leave everything above it
alone. Open the plan with a 2–4 sentence summary a human can approve or
reject on. Report the files the change will touch and the stop-list
entries they hit. A plan goes in the description, never in a comment,
because the description is the brief every later subagent reads.

## On a planned report

Leave the ticket's status and labels as they are. The plan is on the
ticket, and approving it is not this skill's job: `orchestrate` does
that at its plan gate, and under a dispatcher the human does it by
adding `plan-approved`.

## On an unplannable or blocked report

Unplannable means the ticket is too vague to plan, or big enough to be
several tickets. Blocked means its premise doesn't match the code: the
bug doesn't reproduce, or the feature already exists. They read
differently to a human. Unplannable needs more scoping, and blocked
needs someone to look at what the planner found. Keep them distinct
when you relay them.

Either way, the planner has already commented on the ticket with the
options it saw and the one it would pick. In the same step, label the
ticket `needs-attention` and post a `factory: escalation` comment (see
the `orchestrate` skill's "Escalation comments") with stage `planning`,
result `unplannable` or `blocked`, and the planner's `OPTIONS` verbatim.
That comment is what turns "unplannable" into a question somebody can
answer in one line, and the planner that wrote it is gone. Leave the
ticket's status where it is: it never left `Todo` or `Backlog`.

A human answers it by editing the ticket and clearing `needs-attention`,
never with a `factory: decision`, since none of that marker's `Next`
invocations replans.

Tell whoever is waiting on this: the human if you were invoked
standalone, or your caller if a skill or dispatcher invoked you. Don't
let one ticket's trouble stop the others you were given.

## Report, per ticket

Relay the planner's report upward, unchanged, adding one line of your
own, RESEARCH: whether you set the brief's RESEARCH line, read from the
labels at spawn. A caller that compares the ticket's `research` label
against the kind of plan it got compares against this line, since the
label may have changed both before and after the spawn.

    TICKET: <id>
    RESULT: planned | unplannable | blocked | skipped
    RESEARCH: yes | no
    SUMMARY: <the 2-4 sentence summary from the plan>
    FILES: <paths the change will touch>
    STOPLIST: <the entries those paths hit, or none>
    OPTIONS: <for unplannable or blocked: 2-3 options with consequences,
             and its pick>
    NOTES: <the risk or open question, why unplannable/blocked, or why
           skipped>

A skipped ticket's report carries only TICKET, RESULT and NOTES.

`RESULT: planned` means a plan is written. It says nothing about
whether anyone has approved it. This skill never implements, researches
or approves anything. See `implement-ticket`, `research-ticket` and, for
the full pipeline, `orchestrate`.
