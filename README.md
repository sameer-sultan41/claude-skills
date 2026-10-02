# claude-skills

A growing collection of [Claude Code](https://claude.com/claude-code) skills, published as an installable plugin.

## Skills

### `babysit-pr`

Drive one or more open pull requests from "reviewer left comments" to "merged," looping fix → push → reply → notify → watch → merge for every new round of feedback, without needing a check-in prompt at each step.

Handles: filtering which reviewers actually need a ping, validating a reviewer's suggestion against the repo's own conventions before applying it (and rebutting it on GitHub when it doesn't hold up), distinguishing infra CI failures from real ones, and polling long-running CI without blocking the session.

Requires the GitHub CLI (`gh`), authenticated for your repo(s). A chat-notification step is optional — see the skill's `Requirements` section.

See [`skills/babysit-pr/SKILL.md`](skills/babysit-pr/SKILL.md) for full details.

### `review-my-requests`

Finds every open PR across your configured scope where you're a requested reviewer, reviews each one, and posts the result — `REQUEST_CHANGES` on any finding, a plain "ready to merge" comment on zero findings, and never an approval unless explicitly requested on a named PR afterward.

Handles: filtering out bot version-bump PRs, correctly skipping PRs already reviewed at the current commit (including the gotchas around unsubmitted draft reviews and description-only fixes that don't change the commit SHA), unconditional PR-size reporting every round, and a self-authored-PR mode for personal repos where there's no one else to request review from.

Requires the GitHub CLI (`gh`), authenticated for your repo(s), and some PR-review process to plug in for the actual code analysis — see the skill's `Requirements` section.

See [`skills/review-my-requests/SKILL.md`](skills/review-my-requests/SKILL.md) for full details.

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
