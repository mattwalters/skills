---
name: new-project
description: Stand up a new GitHub repo, a Linear team and the factory wiring (an `AGENTS.md` `## Orchestrate` section plus a `.claude/settings.json` declaring the marketplace) in one interactive sitting, through an interview where enter accepts each conservative default, ending with a first PR left open for the human to merge. Refuses if the repo or the Linear team already exists. Interactive only, at a terminal with a human answering — never for cron or unattended runs — and it does not add the project to cron. Use when asked to start or set up a new project or repo for the factory. Do not use to adopt an existing repo.
---

# New project

Starting a project by hand means the same setup every time: a GitHub
repo, a Linear team, a working copy, a `.claude/settings.json`, and an
`AGENTS.md` with a `## Orchestrate` section the other factory skills
read. Miss the last and `orchestrate` refuses to run. This skill does
all of it in one sitting. It is not a pipeline stage: nothing in
`orchestrate` calls it.

## Where it runs

**Interactive Claude Code at a terminal only. Never cron, never
unattended.** It uses the human's own `gh` CLI and the session's Linear
MCP, and both can create things. Those are different credentials, on a
different surface, from the read-only `ops-github` path voice mode
reads through. Nobody should wire this skill into a cron job or into
the read-only Worker.

If no human is there to answer (an unattended run, a cron job, a
subagent with no one to ask), say so and stop before any write.

## Interview

Ask one question at a time. State a default with each; enter accepts
it. Form an opinion and propose it rather than leaving a placeholder
for someone to fill in later.

1. **Owner** — the personal login, or an org from `gh api user/orgs`.
   Default: personal.
2. **Repo name.**
3. **Visibility** — default **private**.
4. **Linear team name and key** — suggest the key from the name.
5. **Base branch** — default `main`.
6. **Check command** — ask what it is. From the stated language and
   stack, propose one; never write a placeholder.
7. **Human checkout path** — default `~/src/<owner>/<repo>`, mirroring
   GitHub's owner/name layout. Confirm rather than assume.
8. **Cron-driven?** — default no. If yes, also create a clone dedicated
   to cron runs at `~/ops/src/<owner>/<repo>`. A cron run must never use
   the human's working checkout, because a run's tool allowlist permits
   `git worktree remove` against anything `git worktree list` reports.
9. **Worktrees directory** — default `$HOME/ops/worktrees/<owner>/<repo>/`,
   namespaced per repo so two adopting repos never collide. It sits under
   the one root the ops cron job can write to,
   `${OPS_WORKTREES:-$HOME/ops/worktrees}`.
10. **Write window** — default `none`. The human may instead declare one
    in the format of orchestrate's "The write window" section.
11. **Review invariants and stop-list** — propose a first pass for the
    human to edit, not a blank. They are the most valuable and the most
    project-specific part of `AGENTS.md`; they will be refined later, as
    an ordinary ticket in the new team.
12. **Seed ticket** — default yes (see "Create, in order").

**The stop-list default is deliberately broad:** auth and authz,
secrets and credential handling, CI config (`.github/workflows/`),
deploy scripts and infra, schema and migrations if any, and the
`## Orchestrate` section of `AGENTS.md` itself. Say the asymmetry
plainly: a too-cautious default costs a question, a too-permissive one
costs an unsupervised merge into something load-bearing.

Expand `~` and `$HOME` to absolute paths before any file operation or
prompt placeholder; Read, Edit and Write don't expand them. Write the
Worktrees field into `AGENTS.md` in its `$HOME/...` form.

## Refuse if anything exists

Before creating anything, check all of these:

- `gh repo view <owner>/<repo>` must fail.
- The Linear team name and the key must both be absent from
  `list_teams`.
- Each target clone directory must not exist.

If any check fails, say exactly what exists and stop. Do not adopt it.

## The write window

After the interview and before the first GitHub write (repo creation,
the seed push, the PR), evaluate the declared write window using
orchestrate's "The write window" procedure, without copying it here. With
`none` it passes trivially. If the window is closed, stop before
creating anything remote, say when it next opens, and print the drafted
`AGENTS.md` and `.claude/settings.json` so nothing typed is lost. There
is no override.

## The Linear team

The Linear MCP cannot create a team. After the absence check, tell the
human to create the team in Linear with the agreed name and key, and
wait. Then verify with `get_team` or `list_teams` that the key matches
and the team has no issues. On a mismatch, say what differs and stop.
Workspace labels (`needs-attention`, `approved-to-merge`) and the stock
statuses come with the team; there is nothing more to create.

## Create, in order

1. Check the write window (above).
2. `gh repo create` with no template and no auto-init.
3. Clone to the human path, and to the cron path if chosen: two
   independent `gh repo clone`s, not worktrees of one another.
4. Seed commit: a one-line `README.md` only, pushed straight to the base
   branch. A brand-new repo has no base branch to open a PR against.
5. Branch `new-project/wiring`. Write:
   - `.claude/settings.json` holding only the marketplace declaration,
     and **no** `enabledPlugins`:

     ```json
     {
       "extraKnownMarketplaces": {
         "mattwalters": {
           "source": {
             "source": "github",
             "repo": "mattwalters/skills"
           }
         }
       }
     }
     ```

   - `AGENTS.md`: a short project stub, then a `## Orchestrate` section
     carrying every field orchestrate's "Repo configuration" lists,
     under exactly those names: **Linear team key**, **Check command**,
     **Base branch**, **Worktrees**, **Review invariants**,
     **Stop-list**, **Write window**. The heading is `## Orchestrate`.
     Write no mode field: modes are chosen per invocation.
6. Commit, push, `gh pr create`. **Leave the PR open for the human to
   merge.** Never merge it. That runs the normal motion once as a smoke
   test.
7. If the human accepted it, create the seed ticket in the new team in
   `Backlog`, not `Todo`, titled along the lines of "Refine AGENTS.md
   review invariants and stop-list". Orchestrate never picks a `Backlog`
   ticket, so no unattended run edits the new project's own stop-list;
   the human promotes it when ready.

If a step fails partway, report what was created and what wasn't, and
do not try to undo it. Deleting a repo is not this skill's to do.

## Not done

This skill does not install or enable the plugin at project scope, and
installs nothing: it is running from the `factory` plugin, so factory is
already installed on this machine, at user scope. It does not add the
project to cron; that stays a deliberate, human-only step. It does not
merge the PR. It does not create the `.agents/skills/` symlinks Codex
and Antigravity use; that is a manual step.

## Closing summary

End with: the repo URL, the PR URL, the team key, the clone paths, and
the seed ticket if any. Then the next steps:

- Merge the PR.
- On any other machine that will run this project, the cron host
  included, install factory at user scope: `claude plugin marketplace add
  mattwalters/skills`, then `claude plugin install factory@mattwalters
  --scope user`.
- Refine the invariants and stop-list.
- Run the first orchestrate supervised.
