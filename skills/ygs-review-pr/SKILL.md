---
name: ygs-review-pr
description: Full PR review for GitHub and Bitbucket — auto-detects platform, fetches diff + description, runs correctness/security/API/SRE review, ranks findings by severity, posts structured report. Outputs JSON status line.
---

# PR Review

For shared review protocol (severity classification, finding format, verdict), read `~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`.

For tracker credential gates and platform detection, read `~/.claude/skills/you-got-skills/skills/shared/tracker.md`.

## Step 1: Detect platform

Determine which platform hosts this PR:

1. If a PR URL is given, parse the host:
   - `github.com` → GitHub
   - `bitbucket.org` → Bitbucket
   - Custom domain → check `.ygs/tracker.yml` for `bitbucket:` key

2. If no URL is given, auto-detect from env:
   ```bash
   if [ -n "$BITBUCKET_TOKEN" ] && [ -n "$BITBUCKET_USERNAME" ]; then
     PLATFORM=bitbucket
   else
     PLATFORM=github
   fi
   ```

3. Override: if `.ygs/tracker.yml` exists and has `tracker: bitbucket`, use Bitbucket.

Set `PLATFORM` to either `github` or `bitbucket` for all subsequent steps.

## Step 2: Credential gate

Follow the credential-verify gate from `shared/tracker.md` for the detected platform.

**GitHub:**
```bash
gh auth status >/dev/null 2>&1 \
  || { echo "[review] ERROR — not logged in to GitHub. Run: gh auth login"; exit 1; }
```

**Bitbucket:**
```bash
curl -sf "https://api.bitbucket.org/2.0/user" \
  -u "$BITBUCKET_USERNAME:$BITBUCKET_TOKEN" > /dev/null \
  || { echo "[review] ERROR — Bitbucket credentials invalid. Check BITBUCKET_USERNAME/BITBUCKET_TOKEN."; exit 1; }
```

Abort with a clear error if credentials are absent or invalid. Empty results from a bad token are indistinguishable from a clean PR.

## Step 3: Fetch PR diff and description

Extract the PR number and repository from the URL or from the current git context.

**GitHub:**
```bash
# Using PR URL or number
gh pr view "$PR_URL_OR_NUMBER" \
  --json title,body,headRefName,baseRefName,additions,deletions,changedFiles,author,labels \
  > /tmp/pr_meta.json

gh pr diff "$PR_URL_OR_NUMBER" > /tmp/pr.diff
```

If the PR URL points to a different repo than the CWD, pass `--repo owner/name`:
```bash
gh pr view "$NUMBER" --repo "$OWNER/$REPO" --json title,body,...
gh pr diff "$NUMBER" --repo "$OWNER/$REPO"
```

**Bitbucket:**
```bash
# Parse workspace, repo, PR ID from URL: bitbucket.org/{ws}/{repo}/pull-requests/{id}
WORKSPACE=...  REPO=...  PR_ID=...

curl -sf "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID" \
  -u "$BITBUCKET_USERNAME:$BITBUCKET_TOKEN" > /tmp/pr_meta.json

curl -sf "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/diff" \
  -u "$BITBUCKET_USERNAME:$BITBUCKET_TOKEN" > /tmp/pr.diff
```

Read the full contents of every changed file, not just diff hunks. Diff context is limited — bugs hide outside the changed lines.

## Step 4: Understand intent

Before reviewing, establish context:

1. Read the PR title and description from `pr_meta.json`
2. Note the base branch and head branch
3. If a linked issue/ticket is referenced in the description, note it — use it to distinguish intentional trade-off from mistake
4. Apply module-local review rules per the protocol in `shared/review-scaffold.md` (tree-walk per changed file, closest-wins precedence, root `.ygs/review-rules.md` as baseline)
5. Read the PR description fully: what problem is being solved? What approach was chosen?

## Step 5: Four-domain review

Run all four domains in parallel. Use the finding format from `shared/review-scaffold.md` (Severity: MUST/SHOULD/MAY, Confidence: HIGH/MEDIUM/LOW, file:line reference, issue, fix).

### Domain 1 — Correctness

- Logic errors, off-by-one, null/nil/undefined dereference
- Race conditions and TOCTOU (check-then-act without locks)
- Partial failure: what if the operation half-succeeds? Is state left consistent?
- Error swallowing: empty catch blocks, ignored return values, silent failures
- Enum / variant completeness: new variants traced through ALL consumers
- Boundary conditions: empty inputs, max limits, type coercion at system edges
- Data loss: destructive writes without confirmation, missing transactions
- Scalability: N+1 patterns, unbounded allocations, O(n²) in hot paths

### Domain 2 — Security

- Injection: SQL, command, LDAP, template injection
- SSRF / path traversal / open redirect
- Authentication bypass: missing auth checks, JWT not validated, session fixation
- Secrets in source: hardcoded tokens, passwords, API keys in diffs or comments
- Insecure deserialization or eval of untrusted input
- Missing rate limiting / DDOS surface on new endpoints
- Overly broad permissions or privilege escalation paths
- Ungated debug logging that leaks PII or secrets

### Domain 3 — API surface

- Breaking changes to public interfaces (removed/renamed fields, changed types, removed endpoints)
- Backward compatibility: existing callers would break without a migration path?
- Versioning: is a new version required? Is the old version deprecated correctly?
- Contract drift: does the implementation match the declared interface/schema/proto?
- Implicit coupling: callers that depend on undocumented behavior that is now changed

### Domain 4 — SRE concerns

- Missing timeouts on network calls, DB queries, or external dependencies
- No retry logic, or retry storms (exponential backoff missing)
- Observability gaps: missing metrics, traces, or logs for new code paths
- Cascading failure risk: if this service fails, what downstream services are affected?
- Missing circuit breakers or bulkheads on new dependencies
- Operational runbook gaps: new failure modes without documented recovery steps
- Graceful degradation: does the system degrade gracefully or fail hard?
- Deployment risk: schema migrations, flag rollouts, backward compatibility during rolling deploys

## Step 6: Approach-level assessment

Before listing individual findings, answer:

- Is this the right solution to the right problem, or a well-executed wrong approach?
- Does the change address the root cause, or paper over a symptom?
- Would a materially simpler design achieve the same goal?
- Does the implementation conflict with how the system actually works (check surrounding code)?

If the approach is fundamentally wrong, that is the first and most important finding — individual code issues are irrelevant if the direction is bad.

## Step 7: Verify all findings

Before reporting any finding, verify it against the actual source:

- Confirm the code actually does what you think it does — read surrounding context, not just the diff hunk
- Test your mental model against edge cases: would the code really fail this way?
- If uncertain, mark confidence as LOW and state what you could not verify
- Never report a finding you have not verified against the actual file

Remove any false positives. A wrong finding wastes reviewer time and erodes trust.

## Step 8: Merge and rank findings

Deduplicate findings that point to the same root cause (e.g., the same missing null check reported by both Correctness and Security is one finding, not two).

Rank by severity:

| Rank | Severity | Meaning |
|------|----------|---------|
| 1 | CRITICAL | Data loss, auth bypass, secret exposure, or system-down risk |
| 2 | HIGH | MUST-level: blocks merge, correctness or security failure |
| 3 | MEDIUM | SHOULD-level: strong recommendation with clear rationale |
| 4 | LOW | MAY-level: suggestion, take or leave |

Within each level, rank by confidence (HIGH before MEDIUM before LOW).

## Step 9: Write findings.json

Write the findings to `findings.json` in the working directory with this exact structure:

```json
{
  "pr_url": "<url>",
  "verdict": "APPROVE | REQUEST_CHANGES | COMMENT",
  "findings": [
    {
      "severity": "CRITICAL | HIGH | MEDIUM | LOW",
      "confidence": "HIGH | MEDIUM | LOW",
      "title": "<short one-line title>",
      "file": "<path/to/file or empty string>",
      "line": <line number or null>,
      "domain": "correctness | security | api | sre",
      "description": "<what is wrong and why it matters>",
      "fix": "<concrete suggested fix>"
    }
  ],
  "summary": "<one sentence overall assessment>"
}
```

Verdict mapping:
- Any CRITICAL or HIGH findings → `REQUEST_CHANGES`
- Only MEDIUM/LOW findings → `COMMENT`
- No findings, or only LOW-confidence suggestions → `APPROVE`

## Step 10: Report

Print the findings report in the format from `shared/review-scaffold.md`:

```
## PR Review: <title>

**Verdict:** APPROVE | REQUEST_CHANGES | COMMENT
**Platform:** GitHub | Bitbucket
**Findings:** <N> total (<critical> critical, <high> high, <medium> medium, <low> low)

---

### CRITICAL

[C1] <title> — <file>:<line>
Confidence: HIGH | MEDIUM | LOW
<description>
Fix: <suggested fix>

...

### HIGH

[H1] ...

### MEDIUM

[M1] ...

### LOW

[L1] ...

---

**Summary:** <one sentence>
```

Then output the JSON status on the **last line** (nothing after it):

```
{"status":"DONE","findings_count":<N>,"verdict":"<APPROVE|REQUEST_CHANGES|COMMENT>","summary":"<one sentence>"}
```

If an unrecoverable error occurs (bad credentials, PR not found):

```
{"status":"ERROR","reason":"<explanation>"}
```
