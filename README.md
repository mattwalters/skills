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
    "studio@mattwalters": true
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

## Plugins

| Plugin | What it is |
|---|---|
| `studio` | The ticket pipeline: `dispatch` runs `implement-ticket`, `adversarial-review` and `merge-queue` from Linear to a merged PR. The skills are being ported in (SKL-2). |

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
