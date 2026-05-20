---
name: ygs-api-review
description: Review API changes for conventions, backwards compatibility, breaking changes, and documentation. Use for backend API changes.
---

# API Review

For detailed checklists, read:
- `references/anti-patterns.md` — 50 common API mistakes
- `references/design-patterns.md` — Positive patterns to recommend

## Step 1: Get scope

```bash
git diff main...HEAD --name-only 2>/dev/null | grep -E '(route|controller|handler|api|endpoint|schema|proto|graphql|openapi|swagger)'
```

Read changed API files and any specs (OpenAPI, proto, GraphQL schemas).

## Step 2: Breaking change detection

Check for changes that break existing clients:

- **Removed fields** from responses (even if "unused")
- **Type changes** (string → number, nullable → required)
- **New required parameters** on existing endpoints
- **Changed HTTP methods or status codes**
- **Renamed or removed endpoints**
- **Changed auth requirements** (public → authenticated)
- **Changed error response format**
- **Removed enum values** from responses

If breaking changes found: Is there a version bump? Deprecation period? Migration path?

## Step 3: Convention review

1. **Naming** — RESTful? Consistent with existing endpoints? Plural nouns for collections?
2. **HTTP methods** — GET (read), POST (create), PUT/PATCH (update), DELETE (remove)?
3. **Status codes** — 200/201/204 for success? 400/401/403/404/422 for errors? Never 200 for errors?
4. **Request validation** — Input validated? Types enforced? Size limits? Required fields checked?
5. **Response format** — Consistent envelope? Pagination for lists? Cursor vs offset?
6. **Error responses** — Structured (code + message + details)? No internal stack traces leaked?
7. **Idempotency** — Safe to retry mutations? Idempotency keys for create operations?
8. **Rate limiting** — Applied to new endpoints? Consistent with siblings?

## Step 4: Documentation & compatibility

- Are API docs updated (OpenAPI/Swagger/proto comments)?
- Do example requests in docs still work?
- Can older mobile/legacy clients still function?
- Is there a sunset timeline for deprecated endpoints?

## Step 5: Report

For each issue:
- **Severity:** Breaking (blocks deploy) | Major | Minor | Nit
- **Endpoint/file** reference
- **Issue** and suggested fix

Summary verdict:
- **Approve** — API changes are safe and well-designed
- **Request changes** — Breaking changes or convention violations need fixing
- **Comment** — Suggestions only

Report **DONE** or **DONE_WITH_CONCERNS**.
