# claude-skills

A growing collection of [Claude Code](https://claude.com/claude-code) skills, published as an installable plugin.

## Skills

### `babysit-pr`

Drive one or more open pull requests from "reviewer left comments" to "merged," looping fix → push → reply → notify → watch → merge for every new round of feedback, without needing a check-in prompt at each step.

Handles: filtering which reviewers actually need a ping, validating a reviewer's suggestion against the repo's own conventions before applying it (and rebutting it on GitHub when it doesn't hold up), distinguishing infra CI failures from real ones, and polling long-running CI without blocking the session.

Requires the GitHub CLI (`gh`), authenticated for your repo(s). A chat-notification step is optional — see the skill's `Requirements` section.

See [`skills/babysit-pr/SKILL.md`](skills/babysit-pr/SKILL.md) for full details.

## Installation

```
/plugin marketplace add sameer-sultan41/claude-skills
/plugin install claude-skills@sameer-claude-skills
```

Or via the CLI:

```
claude plugin marketplace add sameer-sultan41/claude-skills
claude plugin install claude-skills@sameer-claude-skills
```

## Contributing

More skills will be added to this same plugin over time — new skills just need a `skills/<name>/SKILL.md` at the repo root, no changes to the plugin/marketplace manifest required.

## License

MIT — see [LICENSE](LICENSE).
