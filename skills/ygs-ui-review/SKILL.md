---
name: ygs-ui-review
description: Review UI/UX changes for accessibility, consistency, and responsiveness. Use for frontend changes.
---

# UI Review

For shared review protocol (diff, severity, output format), read `~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`.

## Step 1: Get scope

Follow the diff protocol from `shared/review-scaffold.md`. Additionally filter for UI files:

```bash
git diff main...HEAD --name-only 2>/dev/null | grep -E '\.(tsx?|jsx?|css|scss|html|vue|svelte)$'
```

Read the changed UI files.

## Step 2: Review

Check for:

1. **Accessibility** — ARIA labels, keyboard navigation, color contrast, screen reader support, semantic HTML
2. **Consistency** — Matches existing design system/patterns? Spacing, typography, colors consistent?
3. **Responsiveness** — Works across viewport sizes? No overflow or broken layouts?
4. **Loading states** — Handles loading, empty, and error states gracefully?
5. **Interaction** — Clear affordances? Feedback on actions? Disabled states?
6. **Copy** — Clear, concise, user-friendly language? No jargon?
7. **Performance** — Large images, unnecessary re-renders, bundle size impact?
8. **i18n** — Hardcoded strings that should be translated?

## Step 3: Report

For each issue:
- Component/file reference
- Severity (blocker/major/minor/nit)
- What's wrong and suggested fix

Report **DONE** or **DONE_WITH_CONCERNS**.
