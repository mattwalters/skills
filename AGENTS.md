# AGENTS.md

This repo is a Claude Code plugin marketplace. It is almost entirely
prose: each skill is a `SKILL.md` plus the prompts it hands to
subagents, and the only code is `bin/validate`. See `README.md` for the
layout, for installing a plugin (once per machine, at user scope — never
pinned in a project), and for how a project opts in to `factory`.

The `factory` plugin is developed with the `factory` plugin. The
`README.md` section "Developing the factory with the factory" explains
why that is safe and what it means for testing a change.

## Orchestrate

The `factory` pipeline — `orchestrate`, `implement-ticket`,
`adversarial-review`, `merge-queue`, `decision-queue` — reads this
section for its repo-specific configuration.

- **Linear team key**: `SKL` (ticket ids are `SKL-<n>`).
- **Check command**: `bin/validate`. It needs `claude` and `jq` on
  PATH. CI runs the same script on every PR.
- **Base branch**: `main`.
- **Worktrees**: `$HOME/ops/worktrees/mattwalters/skills/`, one
  detached worktree per ticket, named for the ticket — outside the
  repo so no `AGENTS.md`/`CLAUDE.md` above the checkout loads into a
  ticket's run.
- **Write window**: closed Monday–Friday 09:00–17:00
  America/Los_Angeles; open otherwise, including all weekend.

This pipeline keeps no run manifest; per-ticket state lives in the
orchestrator's own context for the life of a run, and whatever must
outlive it is written to Linear or the PR instead (see `orchestrate`'s
"Escalation comments"). A `## Orchestrate` section that still declares a
**Run manifest** field is not in error — nothing reads it.

Expand `$HOME` to an absolute path before writing the worktrees value
into a prompt or using it in a file operation. A shell expands `$HOME`
on its own (which is why `git worktree add` needs no change here), but
a subagent's Read/Edit/Write calls and a prompt placeholder do not.

The worktrees path is outside the repo, so there's nothing to
gitignore. It sits under `$HOME/ops/worktrees` because that is the only
directory the unattended orchestrate job in `mattwalters/ops` can write
to (`${OPS_WORKTREES:-$HOME/ops/worktrees}`); a worktrees path anywhere
else gets every file write in that job's runs denied. The field is
used exactly as written, never derived from `OPS_WORKTREES`; if that
job's root is ever set elsewhere, a human edits this field to match.
Per-run state stays on the machine that ran it. The
directory is shared by every checkout of this repo on the machine —
every clone, and every worktree of a clone — so run any `factory` skill
against this repo — whether as part of orchestrate or standalone — from
one checkout at a time (a worktree already at `<runs-dir>/<TICKET>`
belongs to whichever checkout made it).

### Review invariants

The code here is instructions to agents, so a bug usually looks like
an instruction that reads fine and does the wrong thing. Treat a diff
that breaks one of these as a major finding, not a nit.

- **A skill's frontmatter `description` matches what the skill does.**
  The description is what decides whether a skill triggers. A change
  in behaviour that leaves the description behind is a finding. So is
  a description that promises something the body doesn't do.
- **Cross-references stay true.** The skills name each other, each
  other's `RESULT:` values and report lines, section headings ("see
  'The stop-list'"), prompt files, and `AGENTS.md` fields. A change to
  any of those that leaves a reference elsewhere pointing at something
  gone or renamed is a finding. Check the other skills as well as the
  one that changed.
- **Instructions to subagents stay unambiguous.** A step a capable
  model could reasonably read two ways is a finding. So is an
  undefined term, or a rule that contradicts another rule without
  saying which one wins.
- **No stop condition or escalation path is silently dropped.** Every
  way a stage can stop (`blocked`, `failed`, `capped`, a stop-list hit,
  a held gate) must still end somewhere a human will see it. A diff
  that removes one, or makes one unreachable, without the ticket asking
  for it is a finding.
- **Nothing project-specific in a skill.** Team keys, check commands,
  branch names, paths and invariants come from the host repo's
  `## Orchestrate` section. A skill that hardcodes one is a finding.
  Examples are fine when they're clearly illustrations.
- **Plugin version bumped when behaviour changes.** A change to what a
  plugin does bumps `version` in its `plugin.json`. A wording-only fix
  doesn't need to.

### Stop-list

A change touching any of these waits for a human to merge it, whatever
mode the run is in. These rules decide when an orchestrate run stops for a
human. A change that loosens them is exactly the change that should
not approve itself.

- The gate and mode semantics in `orchestrate`: the three gates, what
  holds or releases each one, the modes and the invocation table, the
  default when an invocation is ambiguous, and the rule that only the
  human releases a gate.
- The stop-list rules: `orchestrate`'s "The stop-list" section, the
  stop-list checks in `adversarial-review` and `merge-queue`, and this
  stop-list itself.
- The escalation rules that override a released merge gate: a capped
  review never merges, and a blocked or failed ticket is never cleared
  under a released gate. That covers anywhere they are stated or
  enforced, in `orchestrate`, `adversarial-review` or `merge-queue`.
- Merge eligibility in `merge-queue`: what counts as approval to merge.
