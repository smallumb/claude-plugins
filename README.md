# claude-plugins

Claude Code plugins, published as the `smallumb` marketplace.

| Plugin | Skill | What it does |
|---|---|---|
| `review-loop` | `/review-loop:review` | Review → PR comments → fix → re-review loop around the built-in `/code-review`. From round 2 it posts only defects introduced since the previous round, so rounds converge. If another session owns the PR branch, it reviews and comments only, watches the PR, and re-reviews after that session pushes. |

## Install

```
/plugin marketplace add smallumb/claude-plugins
/plugin install review-loop@smallumb
```

To have a project offer it to everyone working in it, add to the project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "smallumb": { "source": { "source": "github", "repo": "smallumb/claude-plugins" } }
  },
  "enabledPlugins": { "review-loop@smallumb": true }
}
```

### Cloud sessions (claude.ai/code)

Cloud sessions do **not** install plugins listed in a repository's `enabledPlugins`. Use a SessionStart hook that runs the commands below. This works only because this repository is public: a cloud session can reach just the repositories it was started with, so a private marketplace repo would be refused.

```bash
claude plugin marketplace add smallumb/claude-plugins
claude plugin install review-loop@smallumb --scope project
```

The watch-the-PR part of `review-loop` needs the cloud-only `claude-code-remote` tools, so cloud sessions are where it matters most.

## Configure a project

Add a `## Review harness` section to the project's `CLAUDE.md`. See [`plugins/review-loop/skills/review/config.md`](plugins/review-loop/skills/review/config.md) for the keys and an example. Without it, the skill infers verification commands and uses defaults.

## Updating

`version` in `plugins/review-loop/.claude-plugin/plugin.json` is set explicitly. Bump it with every change you want installed copies to pick up; then `claude plugin update review-loop@smallumb`.
