---
name: review-my-requests
description: Use when the user wants to review, right now in this session, every open PR where they've been requested as a reviewer — phrases like "review my requested PRs", "check PRs I need to review", "review pending review requests", or running /review-my-requests. Also covers "review my own open PRs too" / "review my self-assigned PRs" — self-authored PRs with no other reviewer requested, on repos (typically personal ones) where there's no one else to ask. Pairs well with a scheduled/cron version of the same prompt if you have one — this is the "do it now, here" on-demand equivalent. SKIP for reviewing one specific PR the user already named (use a dedicated PR-review skill directly, if you have one) — this skill is about finding and triaging the whole queue, not reviewing a single named PR.
---

# review-my-requests

Finds every open PR across your configured scope where you're a requested reviewer, reviews each one, and posts the result — `REQUEST_CHANGES` when there's a finding, a plain "ready to merge" `COMMENT` when there isn't, and never an `APPROVE` unless you explicitly ask for one on a named PR after seeing the summary. Designed to run synchronously in a terminal session on demand ("review my requests" → it just happens, now), not to wait for a scheduled tick.

## Requirements

- **GitHub CLI (`gh`)**, authenticated for every repo in scope.
- **A PR-review engine** — this skill drives the queue (find PRs, dedupe, decide what to post) but delegates the actual code review of each PR to whatever review skill/process you use (your own `CLAUDE.md`/`AGENTS.md` conventions, a dedicated review skill, or just doing the review inline). Reference it in step 5 below.

## Rules

- Find every open PR across your configured scope (see Setup) where you're a requested reviewer.
- **Skip Dependabot/Renovate PRs entirely by default** — don't review them, count them, or list them as pending work. Automated bump PRs dominate the queue on dormant repos (one measured scan: 16 of 22 pending requests were bot bumps aged 109–789 days, several already superseded by a newer bump in the same repo) and reviewing them burns the run without telling you anything actionable. Filter at the query (`-author:app/dependabot`), not after analysing them. If you explicitly ask for a bump review in a given run, honor that for that run only — a dedicated dependency-bump-review skill, if you have one, is the better tool for that anyway.
- Skip any PR you've already reviewed at its current head commit SHA — don't re-review something unchanged (see step 3 for the gotchas in doing this check correctly).
- Post a lightweight "🔍 Reviewing this PR now..." comment before starting analysis, so it's visible the moment it's picked up.
- Review using your full review process, including validating every candidate finding against the target repo's own `CLAUDE.md`/`AGENTS.md` and existing patterns before treating it as actionable — don't blindly apply a generic checklist item that doesn't fit this specific codebase.
- **If there is any finding at all** (including a single Minor or Suggestion): post `REQUEST_CHANGES` immediately, no confirmation needed — the findings themselves justify it.
- **If zero findings:** post a `COMMENT` review right away, no confirmation needed, with a short message stating the PR is clear and ready to merge (say what was actually checked, not just "LGTM"). **Never auto-`APPROVE`** — an actual GitHub approval only happens if you're explicitly asked to approve a specific PR after seeing the summary, never inferred from silence or from the zero-finding result itself.
- Append a stats line to the review body: elapsed time for that PR's review. Include a token figure only if there's a genuine way to know it in your environment — label it "approximate," never fabricate one.
- Read the posted review back to confirm it landed with the expected `state` before moving to the next PR — never report a review as posted without this check.

## Setup — confirm once per run, not every PR

At the start of a run, establish:

1. **Default scope.** If invoked from inside a repo directory, default to that single repo. Otherwise you need a short, deliberate list of repos to scan — not a ranking snapshot, a narrowing. If you don't already have one, ask, or build it from the repos the user actually gets review requests on (not the ones they author PRs in — those are frequently different sets, and an authorship-ranked list can return zero pending requests while an org-wide scan finds plenty, all outside that list). Only fall back to a full org-wide scan when the user explicitly says "across all repos"/"the whole org," or names a different org/repo outright.
2. **Working-hours gate, if wanted.** Some users want this skill to no-op outside working hours (useful mainly for the scheduled-routine variant, less so for an on-demand "do it now" run, but ask if unsure). If set, check the current time in the user's timezone first and stop cleanly if outside the window, rather than scanning and reviewing anything.
3. **Self-authored-PR scope, if used.** See "Self-authored PRs" below — only relevant for personal/solo repos, off by default.

Because the default scope is a deliberate narrowing, a clean result from it does not mean the queue is empty — when it comes back with nothing, say so explicitly and offer an org-wide cross-check rather than flatly reporting "nothing to review."

## Self-authored PRs with no one else to review them — only when asked

`review-requested:@me` cannot surface these: GitHub refuses to let a PR's own author be added as a requested reviewer on it at all. Filtering on `assignee:@me` instead is **not** a usable proxy: many repos auto-assign the author as assignee by default regardless of review intent, so `author:@me assignee:@me` mostly returns ordinary PRs that already have a real human reviewer requested (or already reviewed) — noise, not a review queue.

The actual signal is **self-authored + zero requested reviewers on a repo with nobody else to ask** — in practice this means personal/solo repos, not team repos with real teammates (those already route through the normal `review-requested:@me` flow when someone else's input is wanted).

Only run this scope when the user asks for it by name ("review my own PRs too", "check my self-assigned PRs") — never fold it into the default scan, since most repos it would touch have nothing to do with the review queue.

```bash
gh pr list --repo OWNER/REPO --author "@me" --state open --json number,title,url,headRefOid
```

Every result here is in scope — there is no `review-requested` filter to further narrow it, since none of these PRs could ever carry one.

**GitHub blocks `REQUEST_CHANGES` and `APPROVE` when the reviewer is also the PR author (`422`).** Every review posted under this scope — findings or not — falls back to `event: COMMENT`, stating the verdict in prose ("would be REQUEST_CHANGES" / "would be APPROVE, but self-review blocks it"). This applies even to the zero-finding "ready to merge" comment from the Rules above — it was always going to be a `COMMENT` here regardless, since `APPROVE` was never reachable in the first place.

## Steps

### 1. Find pending review requests

Exclude bot bumps at the query, not after:

```bash
gh pr list --repo "OWNER/REPO" --search "review-requested:@me -author:app/dependabot" --state open --json number,title,url,repository
```

Repeat per repo in scope and merge results. For an org-wide scan, use the search API directly rather than a CLI search-shorthand command — a leading `-` in a negated query term (`-author:app/dependabot`) can be misparsed as the tool's own flag by some CLI search subcommands, and the safe escape form for at least one such tool has been observed to exit 0 and print nothing, which reads exactly like an empty queue:

```bash
gh api -X GET search/issues \
  -f q="org:ORG is:pr is:open review-requested:@me -author:app/dependabot" \
  -f per_page=100 \
  --jq '.items[] | {number, title, url: .html_url, repo: (.repository_url | split("/") | .[-1])}'
```

### 2. Filter out already-reviewed-at-this-SHA PRs

For each result:

```bash
gh pr view <number> --repo <owner>/<repo> --json headRefOid --jq .headRefOid
ME=$(gh api user --jq .login)
gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq ".[] | select(.user.login == \"$ME\" and .state != \"PENDING\") | .commit_id"
```

If the current head SHA is already in that list, skip — nothing changed since the last review. **Exception:** if that review's state is `CHANGES_REQUESTED`, read its body first (see "Description-only findings" below) before skipping. Only skip outright when that review was `COMMENT`/`APPROVE`, or was `CHANGES_REQUESTED` with a code-level finding still open.

**A `PENDING` (draft, never-submitted) review must not count as "already reviewed."** It's invisible to everyone but its author, so filtering it out of the check above is required — otherwise a manually-started-but-unsubmitted review permanently blocks this skill from ever reviewing that PR, even though nobody but you has actually seen anything. If a `PENDING` review from your account exists at the current head SHA, submit it as-is first — `gh api -X POST repos/<owner>/<repo>/pulls/<number>/reviews/<review_id>/events -f event="COMMENT"`, preserving its comments unchanged — so it becomes visible, then proceed with the full review below as normal.

**Description-only findings survive a head-SHA match, and the plain skip check above misses this.** A finding about the PR *description* (stale, doesn't match the shipped diff, missing a required section) can be fixed by editing the description text alone — no new commit, so `headRefOid` never changes, and the naive "same commit → skip" check would leave that PR stuck on an outdated `CHANGES_REQUESTED` forever even after the author fixes it. Before skipping a `CHANGES_REQUESTED` head-SHA match, read that review's own body:

- If **every** row in its findings table is marked resolved except one or more **description-only** findings (the finding is about the PR description text itself, not any file in the diff), re-check just the current PR description against the current diff yourself (no full code re-review needed, since nothing code-side changed). If the description now matches what's actually shipped, post a short follow-up: table showing that finding now resolved, verdict "PR is clear and ready to merge," `event="COMMENT"`. If it's still stale, leave it alone — you already told them once; wait for either a new commit or the user asking directly.
- If **any** remaining open finding is code-level (touches a file in the diff), skip as normal — nothing to re-check without a new commit.

### 3. Mark it as being reviewed

```bash
gh api repos/<owner>/<repo>/issues/<number>/comments -f body="🔍 Reviewing this PR now..."
```

### 4. Review

Run your full review process against this PR — metadata/diff gathering, stack detection, org/repo conventions, convention checks, code analysis across the usual dimensions (correctness, security, architecture, documentation, backward-compatibility, edge cases), dedupe against existing comments/reviews so you never re-raise something already flagged, validate each candidate finding against the target repo's own `CLAUDE.md`/`AGENTS.md`.

**Size check runs every round, unconditionally — never waved away as "just merge noise."** Pull the current totals (`gh pr view --json additions,deletions,changedFiles`) on every single pass, first round or twentieth. A genuinely huge diff (well past the point a diff viewer can render it usefully) is itself a finding regardless of whether the bulk is the ticket's own commits or accumulated merge noise — "it's just merge noise" is not an exemption. Report the actual current numbers every time, and restate it if a prior round already flagged it and it's still unaddressed.

### 5. Post `REQUEST_CHANGES` findings, or a ready-to-merge `COMMENT`

For any PR with at least one finding, post it now:

**Write the review body to a temp file first — never pass it inline via `-f body="..."`.** Findings are full of backtick-wrapped `` `file:line` `` citations, and the shell interprets backticks inside a double-quoted `-f` value as command substitution, which breaks the call. Use `-F body=@<path>` instead, which reads the field raw from the file with no shell interpretation:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews -F body=@/tmp/pr-review-body.md -f event="REQUEST_CHANGES"
gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq '.[-1] | {state, html_url, commit_id}'
```

Confirm `state` is `CHANGES_REQUESTED` and `commit_id` equals the head SHA from step 2, retrying once if it didn't land, before reporting success.

For a PR with **zero findings**, post it as `COMMENT` right away instead — no confirmation needed:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews -F body=@/tmp/pr-review-body.md -f event="COMMENT"
gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq '.[-1] | {state, html_url, commit_id}'
```

The drafted body for a zero-finding PR should say plainly that the PR is clear and ready to merge — state what was actually checked, not just "LGTM". Confirm `state` is `COMMENTED` and `commit_id` matches. **Never post `APPROVE` here or anywhere in this flow** — an actual GitHub approval only happens later, and only if explicitly requested for that specific PR after seeing the summary (step 6).

### 6. Summarize

Report as a markdown table, one row per PR: title/number as a clickable markdown link, verdict, elapsed time. **Every row must also carry the bare PR URL as plain, visible, copyable text** — a markdown link alone hides the target and can't be pasted into Slack/Jira/a browser, so the clickable link accompanies the plain URL, never replaces it.

- **PRs posted as `REQUEST_CHANGES`**: link straight to `<url>#pullrequestreview-<review_id>` (plus the bare URL), done.
- **PRs posted as `COMMENT`** (zero findings): same link format, note that it's a ready-to-merge comment, not an approval.
- **PRs skipped as unchanged**: note it, done — they already carry whatever review they had.

If the user later says "approve it" for a specific PR (typically one that got the ready-to-merge comment), post an actual GitHub approval for that PR only:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews -f event="APPROVE" -f body="Reviewed — no issues found."
```

Read back and confirm `state` is `APPROVED` before reporting success. Only act on the specific PR(s) named — never batch-approve multiple PRs off one reply unless clearly asked to.

## Related

- Whatever PR-review skill/process you use for the actual code analysis in step 4 — this skill is the queue-finder and poster around it.
- A PR-author-side skill (fixing feedback on your own PR until merged) for the other direction of this workflow.
- A scheduled/cron equivalent of this same prompt, if you want it running unattended on a timer — this skill is the "do it now, here" on-demand version of that.
