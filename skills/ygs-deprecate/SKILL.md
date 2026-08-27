---
name: ygs-deprecate
description: Deprecation and migration — decide when to remove code, run safe migrations (Expand/Contract), remove zombie code. Use when retiring a feature, API, or data schema.
argument-hint: "[what to deprecate: feature / API endpoint / table / dependency]"
---

# Deprecate

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — deprecation done wrong is worse than the code you're removing.

**Code is a liability, not an asset.** Every line has a maintenance cost. Deprecation is how you pay down that debt without creating new debt.

**Hyrum's Law:** With enough consumers, every observable behavior of a system will be depended upon by someone. Removing anything has a blast radius. Start by mapping it.

## When NOT to use

- No replacement exists and no completion timeline has been set — deprecation without a destination is abandonment, not lifecycle management
- External consumers exist with no agreed notice period — compulsory deprecation requires a timeline commitment before announcing
- Migration cost clearly exceeds the next 12 months of maintenance cost — document the analysis and defer; revisit next planning cycle

## Step 1: Deprecation decision

Answer these five questions before proceeding:

1. **Unique value?** Does this code provide value that nothing else provides, or is it a redundant path?
2. **Consumer count?** How many callers, users, or dependents exist? (grep the codebase; check API logs for endpoints)
3. **Replacement exists?** Is there a better alternative already, or does it need to be built first?
4. **Migration cost?** How much work to move all consumers to the replacement?
5. **Maintenance cost?** What is the ongoing cost of keeping this alive?

If (maintenance cost) > (migration cost): proceed. Otherwise: document why and defer.

## Step 2: Choose deprecation type

| Type | When to use | What it means |
|------|------------|---------------|
| **Advisory** | You own the code, consumers are internal | Mark deprecated, add warning, migrate at your own pace |
| **Compulsory** | External API consumers, SLA commitments | Announce + give notice period + provide migration tooling + support migration actively |

Default to advisory. Compulsory requires: deprecation notice with timeline, migration guide, tooling (compat shim or automated codemod), and active migration support.

## Step 3: Build the replacement first

**Never remove before the replacement exists and is proven.** The sequence is always:

1. Build and ship the replacement
2. Verify the replacement is correct and performs at scale
3. Then begin migration

Removing first is how you create incidents.

## Step 4: Migrate consumers

**If you own the infrastructure, you migrate the consumers.** Don't create migration work for others.

The Churn Rule: adding a new API to replace an old one doubles the maintenance surface until migration is complete. Move fast.

Migration patterns:
- **Strangler Fig:** New code handles new traffic; old code handles old traffic. Gradually shift traffic until old code has zero consumers, then remove.
- **Adapter:** Write a thin adapter that calls the new API in the old API's shape. Migrate consumers to the new API at their own pace; remove the adapter when all consumers are migrated.
- **Feature Flag Migration:** Gate new behavior behind a flag. Enable for new consumers; migrate old consumers; remove the flag and old code path.

## Step 5: Database schema migrations (Expand/Contract)

Schema changes require special discipline. **Never rename or remove in place.**

The Expand/Contract pattern:
1. **Expand:** Add the new column/table/index. Old code continues using the old shape. Deploy.
2. **Migrate:** Backfill data from old to new. Run in batches to avoid locking. Verify data integrity.
3. **Switch:** Deploy code that writes to both old and new, reads from new.
4. **Contract:** Remove the old column/table/index once all consumers are migrated. Deploy.

**Rules:**
- Test the rollback path: can old code read the database after step 1? Can it still read it after step 3?
- Batch backfills — never update millions of rows in a single transaction
- Do not rename a column in one step; add the new column, migrate, remove the old column
- `NOT NULL` columns: add as nullable first, backfill, then add the constraint in a separate migration

## Step 6: Zombie code identification and removal

Zombie code: code that is technically alive but never runs in any real execution path.

Signs:
- Functions with zero callers (search the codebase exhaustively before concluding)
- Feature flags permanently false for > 6 months
- Database tables with zero rows and no write path in production
- Dead configuration branches (`if env == "legacy_prod"` where legacy_prod no longer exists)

Required response: **delete it**. Zombie code accumulates silently and confuses every future reader. If uncertain about a caller: add a log + metric at the entry point, run in production for 2 weeks, observe. If never triggered: remove.

## Step 7: Completion

Verify:
- [ ] All consumers migrated or notified (with timeline for compulsory)
- [ ] No callers remain for removed code (grep confirms)
- [ ] Database migrations are backwards-compatible during rollout window
- [ ] Rollback path tested: if new code is rolled back, old data shape still works
- [ ] Tests for removed paths removed too (dead test code is also a liability)

Report **DONE** with:
- What was deprecated
- Migration pattern used
- Consumer count before/after
- Rollback safety verified

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "We might need it later" | You might. But you can read it in git history if you do. The maintenance cost of keeping it is paid today; the hypothetical value is paid never. |
| "Let consumers migrate at their own pace" | If you own the infrastructure, you own the migration. "Their own pace" means it never happens. |
| "The schema migration is risky, let's do it all at once" | Doing it all at once is why schema migrations are risky. Expand/Contract breaks it into safe steps. |
| "The column is unused, I'll just leave it" | Unused columns confuse readers, slow queries, and get populated by mistake. Remove them. |
| "I'll add a deprecation notice and move on" | Notices without migration support means the deprecated code lives forever. |
