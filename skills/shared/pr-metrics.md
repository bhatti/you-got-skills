# PR Metrics — one definition for every PR report

Used by every skill that reports on pull requests (`ygs-pr-risk`, `ygs-pr-audit`, `ygs-merge-queue`, `ygs-review-pr`, `ygs-review-deep`, `ygs-pr-queue`, `ygs-standup`). The numbers come from the pipeline (`scripts.common.pr_signals`, rendered by `scripts.common.pr_metrics`); never recompute or hand-write them when the pipeline provided them. Works the same for GitHub and Bitbucket/Jira PRs.

## The metrics

| Metric | Definition |
|--------|------------|
| Comments (human · bot) | Review comments on the PR, inline and top-level, review summaries included, deleted ones excluded. Split by who wrote them |
| Tasks (human · bot) | Bitbucket: PR tasks by creator. GitHub: review threads by who started them. "Open" = unresolved. Both trackers: bot-created tasks counted separately |
| Major/Critical Issues | CRITICAL + HIGH, from every source and shown by source: AI review findings (`findings.json`), bot comments and human comments that carry a severity marker (`[critical]`, `**Major**`, `Severity: High`, 🔴/🟠, `P0`/`P1`) |
| Reviewers | Union of assigned reviewers, review authors and commenters, minus the PR author. **Bots that only commented count as reviewers**; humans and bots are counted separately; approvals are shown as `approved/humans` |
| Commits | Commits in the PR excluding merge commits (2+ parents), so merging the base branch in does not look like churn |
| Size and risk | LOC, files, blast radius, risk score/tier, categories, age (unchanged) |

## Who is a bot

1. The tracker's account type is authoritative (GitHub `User`/`Bot`, Bitbucket `personal`/`app`): a `User` account is a person even if its name ends in `bot` or its comment quotes a bot's heading.
2. Otherwise a login that **ends** in `bot` (`dependabot`, `deploy-bot`, `x[bot]`) or is an exact known review-bot login (`coderabbitai`, `claude`, `copilot`, `sonarcloud`...) is a bot. A name that merely *contains* "bot", "claude" or "copilot" (`claudette`, `robotics-lead`) is a person.
3. A review-bot body marker (`## Code Review`, `ygs-review-pr`) counts only when the account type is unknown.

CI bots are bots too and are not code reviewers; do not credit them as review coverage.

## Rendering rules

1. **One metrics table per report.** Single-PR reports (pr-risk, review) use the 5-column table `Metric | Value | Benchmark | Signal | Description`. Multi-PR reports use one per-PR table plus the one aggregate Metrics Dashboard. Never add a second metrics table, and never restate the same numbers in another table.
2. **Unavailable is not zero.** `—` means the signal was not collected; `n/a` means the fetch failed. Say so in the narrative; do not describe either as "no comments" or "no tasks".
3. Counts above the fetch cap are shown with a trailing `+`.
4. Slack output follows `output-format.md` (bullets): keep the same numbers, drop the table.

## Reading the signals

- No human comment and no human reviewer on a bot-authored PR is the strongest sign of an unreviewed implementation.
- Many bot comments with few human comments: check whether the bot findings were acted on before concluding the review was thorough.
- Open tasks at merge time, major issues with no follow-up, or many commits on a small PR point to rework: cite the PR numbers.
- Judge implementation quality from these together with the review findings; no single number is a verdict.
