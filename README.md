# mattwalters/skills

A Claude Code plugin marketplace. Projects install skills from here; nothing
here belongs to any one project.

The repo is public on purpose. Claude Code can clone it without a token, so
installing from it needs no credentials.

## Installing

Plugins from here are installed once per machine, at user scope:

```
claude plugin marketplace add mattwalters/skills
claude plugin install factory@mattwalters --scope user
```

A project declares nothing: no `enabledPlugins` and no
`extraKnownMarketplaces` in any of its `.claude/settings*.json`, and no
`--scope project` install. A project pin registers a separate install for
every checkout and worktree that opens the project, and those drift from the
user-scope version. Every machine that runs the factory, the cron host
included, runs the two commands above once instead.

To pick up later changes, run `claude plugin marketplace update mattwalters`.

### Opting in to `factory`

Installing `factory` isn't enough to run it in a project. The project also
needs a `## Orchestrate` section in its own `AGENTS.md`. If
that section is missing, `orchestrate` stops straight away instead of guessing.
The other factory skills read the same section.

The section declares:

- **Linear team key**: the team whose `Todo` queue the project draws from.
- **Check command**: always `./scripts/check.sh`, what an implementer or fixer
  must run, and pass, before pushing. The repo commits that script, and it
  runs whatever the repo's checks are; the section never names them. See
  "Repo configuration" in [orchestrate](plugins/factory/skills/orchestrate/SKILL.md).
- **Base branch**: what worktrees branch from and PRs merge into.
- **Worktrees**: the directory per-ticket worktrees go in. An absolute path
  outside the repo is recommended, so no ancestor `AGENTS.md`/`CLAUDE.md` can
  load into a run; it's shared across every clone of the repo on the machine.
  Namespace it per repo (this repo's own field uses
  `.../worktrees/mattwalters/skills/`) — two adopting repos pointed at
  the same shared directory would collide on worktree paths. A path
  inside the repo must be gitignored. A section that still declares a
  **Run manifest** field is fine; the pipeline keeps no manifest and the
  line is simply ignored.
- **Review invariants**: what a reviewer should be adversarial about in this
  project. Reviewers bring no invariants of their own.
- **Stop-list**: the paths and subjects whose merge always waits for a human,
  whatever mode a run is in. A project that declares none has none.
- **Write window**: the weekday hours, if any, during which the pipeline
  won't commit, push, or merge, so a public timestamp is never evidence of
  when someone worked. Required; `none` means no window.
- **Telemetry** (optional): the absolute path of a poster command that records
  each stage attempt's outcome and each review finding's severity. Absent
  means off, and a missing one never stops a run. A failing poster never
  stops or fails a run either. See "Telemetry" in
  [orchestrate](plugins/factory/skills/orchestrate/SKILL.md).

This repo's own [`AGENTS.md`](AGENTS.md) is a worked example.

For a brand-new project, `/factory:new-project` does all of this in one
sitting: it creates the repo, the Linear team, the `AGENTS.md` `## Orchestrate`
section, and refuses a repo or team that
already exists. The manual steps above remain the path for an existing repo.

### Upgrading from factory 0.4 or earlier

`factory` 0.5 renamed the `dispatch` skill to `orchestrate`, including the
`AGENTS.md` heading it reads. A project already on `factory` 0.4 or earlier
needs three steps to move to 0.5:

1. Rename the `AGENTS.md` heading `## Dispatch` to `## Orchestrate`. The
   fields under it are unchanged.
2. Rename any `.agents/skills/dispatch` symlink to
   `.agents/skills/orchestrate`.
3. Run `claude plugin marketplace update mattwalters`.

Invoke it afterwards as `/factory:orchestrate`. Until the heading is
renamed, `orchestrate` stops before doing anything and says so, rather than
running against a `## Dispatch` section it no longer reads.

## Plugins

| Plugin | What it is |
|---|---|
| `factory` | Software factory skills. `orchestrate` takes Linear tickets through `implement-ticket`, `adversarial-review` and `merge-queue` to a merged PR (a `research`-labelled ticket instead ends in a report written to the Linear ticket, with no PR); `decision-queue` turns whatever stopped into decisions to answer; `new-project` stands up a new repo, Linear team and the factory wiring in one interactive sitting; `/factory:preamble` loads the operating manual into a Claude Code session. Each repo that uses them declares its settings in an `AGENTS.md` `## Orchestrate` section. |

## Layout

```
.claude-plugin/marketplace.json   the catalog: one entry per plugin
plugins/<plugin>/
  .claude-plugin/plugin.json      name, description, version
  skills/<skill>/SKILL.md         one directory per skill
  commands/<command>.md           optional: slash commands (factory: preamble)
  OPERATING.md                    optional: read-aloud orientation (factory)
```

To get oriented on how the factory works, start with
[`plugins/factory/OPERATING.md`](plugins/factory/OPERATING.md), or run
`/factory:preamble` in a Claude Code session to load it.

This repo is meant to hold many plugins. Skills that aren't related go in
separate plugins, so each plugin can be versioned and enabled on its own. To
add a plugin:

1. Create `plugins/<name>/.claude-plugin/plugin.json` and
   `plugins/<name>/skills/`.
2. Add an entry to `plugins` in `.claude-plugin/marketplace.json` with
   `"source": "./plugins/<name>"`.
3. Run `bin/validate`, which CI also runs on every PR. It runs
   `claude plugin validate --strict` on the marketplace and on each plugin. It
   also checks that every plugin directory is listed, that every listed source
   exists, and that each `plugin.json` name matches its marketplace entry.

Bump a plugin's `version` in its `plugin.json` when you change what it does.

## Developing the factory with the factory

This repo opts in to `factory` like any other project, so changes to the
factory skills go through `orchestrate` too. That's safe because a run never
executes the files it is editing:

- An orchestrate run uses the **installed plugin**, the marketplace clone that
  Claude Code manages. It doesn't use the files checked out in the worktree it
  is changing. So a change is always planned, implemented and reviewed by the
  version from *before* that change, and the pipeline can't modify itself
  mid-run.
- A change takes effect only after it merges **and** someone runs
  `claude plugin marketplace update mattwalters`. That refresh is the gate.

The catch: a change that breaks a skill won't show up in the run that wrote
it. It shows up in the first run after the refresh. Treat that run as the real
test, and run it supervised.
