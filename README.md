# mattwalters/skills

A Claude Code plugin marketplace. Projects install skills from here; nothing
here belongs to any one project.

The repo is public on purpose. Claude Code can clone it without a token, so a
project adopts it with a settings block and no credentials.

## Adopting it in a project

Add this to the project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "mattwalters": {
      "source": {
        "source": "github",
        "repo": "mattwalters/skills"
      }
    }
  },
  "enabledPlugins": {
    "factory@mattwalters": true
  }
}
```

- `extraKnownMarketplaces` tells Claude Code where the marketplace lives. The
  key (`mattwalters`) must match `name` in `.claude-plugin/marketplace.json`.
- `enabledPlugins` turns individual plugins on, as `<plugin>@<marketplace>`.
  Enable only the plugins the project uses.

When someone trusts the project folder, Claude Code offers to install the
marketplace and the enabled plugins. To pick up later changes, run
`claude plugin marketplace update mattwalters`.

### Opting in to `factory`

The settings block installs `factory`, but installing it isn't enough to run
it. The project also needs a `## Dispatch` section in its own `AGENTS.md`. If
that section is missing, `dispatch` stops straight away instead of guessing.
The other factory skills read the same section.

The section declares:

- **Linear team key**: the team whose `Todo` queue the project draws from.
- **Check command**: what an implementer or fixer must run, and pass, before
  pushing.
- **Base branch**: what worktrees branch from and PRs merge into.
- **Worktrees**: the directory per-ticket worktrees go in. Gitignore it.
- **Run manifest**: the file `dispatch` keeps per-ticket state in, usually
  inside the worktrees directory.
- **Review invariants**: what a reviewer should be adversarial about in this
  project. Reviewers bring no invariants of their own.
- **Stop-list**: the paths and subjects whose merge always waits for a human,
  whatever mode a run is in. A project that declares none has none.

This repo's own [`AGENTS.md`](AGENTS.md) is a worked example.

## Plugins

| Plugin | What it is |
|---|---|
| `factory` | Software factory skills. `dispatch` takes Linear tickets through `implement-ticket`, `adversarial-review` and `merge-queue` to a merged PR; `decision-queue` turns whatever stopped into decisions to answer. Each repo that uses them declares its settings in an `AGENTS.md` `## Dispatch` section. |

## Layout

```
.claude-plugin/marketplace.json   the catalog: one entry per plugin
plugins/<plugin>/
  .claude-plugin/plugin.json      name, description, version
  skills/<skill>/SKILL.md         one directory per skill
```

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
factory skills go through `dispatch` too. That's safe because a run never
executes the files it is editing:

- A dispatch run uses the **installed plugin**, the marketplace clone that
  Claude Code manages. It doesn't use the files checked out in the worktree it
  is changing. So a change is always planned, implemented and reviewed by the
  version from *before* that change, and the pipeline can't modify itself
  mid-run.
- A change takes effect only after it merges **and** someone runs
  `claude plugin marketplace update mattwalters`. That refresh is the gate.

The catch: a change that breaks a skill won't show up in the run that wrote
it. It shows up in the first run after the refresh. Treat that run as the real
test, and run it supervised.
