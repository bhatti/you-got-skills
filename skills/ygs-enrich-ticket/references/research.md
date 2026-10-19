# Bounded research for ACs and plans

Ground every AC and plan in the repository; keep the cost bounded.

1. **Keywords:** 3-5 meaningful terms from the title and description (feature area, component, entity). Skip noise words (fix, add, update).
2. **Search in parallel** (the three are independent):
   - Design docs: `~/.claude/skills/you-got-skills/skills/shared/docs-discovery.md` paths first, then a case-insensitive keyword search of the docs directories. Docs rarely carry the ticket id in the file name. A linked page you cannot read is listed as "unread, review manually"; do not guess its content.
   - Source: keyword search limited to source files, excluding `node_modules`, `vendor`, `dist`, `build`, `coverage`, `.git`, generated and test directories. Never an unscoped recursive grep over the whole tree.
   - Tests: the same keywords, matching test file names and test directories.
3. **Read at most 3 files** in total, the most specific hits, the first 80 lines of each unless a narrower section is obviously the one. Skip generated output, vendored code and migrations.
4. **History:** `git log --oneline -n 5 -- <file>` for the files you name; prior changes show hidden constraints and why this was not done before. Use blame only for a specific line you are about to rely on.
5. **Report what you found:** a "Research" section listing each doc and file read and one sentence on how it shaped an AC or the plan. If no design doc exists, say so.

Variants and process modes come from the repository's own docs when the project defines them (`.ygs/readiness.yml`); otherwise do not ask about them.
