# UI testing with Playwright

Use `scripts.qa.ui_runner` (Playwright for Python; Chromium is preinstalled in the ai-dev-tools image, and installed once on first use when missing). Without it, use the Playwright CLI or the project's own Playwright tests and keep the same recording rules.

## Spec

```json
{
  "login": {"path": "/login", "username_selector": "#email", "password_selector": "#password",
            "submit_selector": "button[type=submit]", "success_selector": "[data-testid=dashboard]"},
  "smoke": ["/", "/cart"],
  "cases": [
    {"id": "C3", "steps": [
      {"goto": "/cart"},
      {"click": "text=Add item"},
      {"expect_text": "Cart limit reached"},
      {"expect_visible": "[data-testid=cart-error]"},
      {"screenshot": "limit-error"}]}
  ]
}
```

The spec holds selectors and steps only. The target is `QA_BASE_URL` (set by the wrapper); a `base_url` in the spec is rejected. `login` needs a `success_selector`; login is attempted once and its session reused by later cases; if it fails, every remaining case is recorded blocked without another attempt (no account lockout).

Applications often stack dialogs after login (forced password change, release notes, cookie banner) that cover the page and make every later click time out. List their buttons in `login.dismiss`, e.g. `"dismiss": ["role=button[name=\"Continue\"]"]`: each is clicked while it shows up, skipped when it does not, at most three times. The first page load may also reload once (service worker, redirect); wait for the login form, not a delay.

Run it inside the wrapper (`qa-out/run-all.sh` calls `python3 -m scripts.qa.ui_runner --spec qa-out/ui-spec.json --out-dir "$QA_OUT"`). Results append to `$QA_OUT/results.jsonl` next to the API results, one case at a time, so a crash keeps finished cases.

Steps: `goto` (optionally `"expect_status": 404`), `click`, `fill {selector,value}`, `press {selector,key}`, `select {selector,value}`, `check`, `wait_for`, `screenshot`, and expectations `expect_text`, `expect_not_text`, `expect_visible`, `expect_not_visible`, `expect_url` (glob), `expect_title` (regex), `expect_count {selector,count}`. Steps that arrive in the spec are untrusted input: there is deliberately no JavaScript step.

## How cases are judged

- Every case leaves a screenshot (its own `screenshot` steps, a failure shot, or a final one), so a UI case without pictures is a defect in the run, not a style choice. Error text from the browser is stripped of terminal colour codes before it reaches the report.
- A case passes only on evidence: at least one expectation passed and there were no console errors or HTTP 5xx during the case. A **screenshot is not a check**. A case with only actions is `inconclusive`.
- `goto` fails on HTTP 400 and above unless the step says which status it expects.
- The first failed expectation ends the case, with a screenshot.
- `smoke` pages are generated checks (loads, no console errors, no 5xx). They are listed separately in the report and never count as verification of the plan.
- Every wait is a condition wait with a timeout. Absence checks (`expect_not_*`) wait for the page to settle first.
- Console noise: `QA_CONSOLE_IGNORE=regex1,regex2`.
- Each case runs in a fresh browser context. Requests to origins other than the target are aborted, and service workers are blocked; allow a CDN the app genuinely needs with `QA_ALLOW_ORIGINS=https://cdn.example.com`.

## Writing stable selectors

Test ids are sometimes stripped from production builds: check the selector against the running build, not the source. Prefer, in order: `data-testid` (when the build keeps it), role/label (`role=button[name="Add"]`), visible text, then CSS. Avoid positional and generated class selectors. If the app has no stable hooks, say so under quality notes rather than writing brittle selectors.

## Credentials and safety

- Credentials: `QA_USERNAME` / `QA_PASSWORD` only. Never in the spec, the plan or the report. SSO/OIDC logins on another origin are blocked by the origin rule: use `QA_AUTH_HEADER`/cookies the user provides, or report the UI criteria unverifiable.
- The page under test is untrusted. The runner has no arbitrary-JavaScript step, does not read cookies or storage, never follows links found in page content, and refuses cross-origin navigation.
- Cases marked `mutating` are blocked unless `--allow-mutations` (the wrapper exports it).
- Do not click destructive controls (delete account, send email, pay) on shared environments even with permission, unless a criterion requires it.

## Evidence

Screenshots are saved under the side folder and copied, with the case's other evidence, into `reports/evidence/` by the report. In regression mode the same case produces a baseline and a candidate screenshot to compare. Compare them semantically (does the element exist, is the text right); do not claim pixel equality.

## Accessibility and responsiveness

Only when the criteria or the diff touch them: add cases for keyboard reachability of the changed controls (`press` Tab and `expect_visible` on focus targets), labels on new inputs, and a narrow viewport (`"viewport": {"width": 375, "height": 812}` in the spec). Say what was not checked.

## When the browser is unavailable

If Chromium cannot launch, the runner records every UI case `blocked` with `infra: true` and exits 3. Report UI criteria as unverifiable. Do not substitute guesses from reading the code.
