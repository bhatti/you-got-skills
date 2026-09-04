# Project Conventions

When ygs skills run in a project, they create and use this directory structure:

```
<project-root>/
├── docs/
│   ├── prd/                    # Product Requirements Documents
│   │   └── YYYY-MM-DD-<slug>.md                    # Simple (single product)
│   │   └── <product>/<project>/YYYY-MM-DD-<slug>.md # Multi-product hierarchy
│   ├── trd/                    # Technical Requirements/Design Documents
│   │   └── YYYY-MM-DD-<slug>.md
│   │   └── <product>/<project>/YYYY-MM-DD-<slug>.md
│   ├── adr/                    # Architecture Decision Records
│   │   └── NNN-<slug>.md      # Sequential numbering (001, 002, ...)
│   ├── architecture/           # System architecture / design documents
│   │   └── <slug>.md
│   ├── spikes/                 # Spike findings
│   │   └── YYYY-MM-DD-<slug>.md
│   └── threat-models/          # Security threat models
│       └── YYYY-MM-DD-<slug>.md
└── tasks/
    ├── backlog/                # Not yet started
    │   └── task-NNN.md
    ├── in-progress/            # Currently being worked on
    │   └── task-NNN.md
    └── done/                   # Completed
        └── task-NNN.md
```

## Rules

- **Status = directory.** Moving a task file between directories changes its status.
- **All files are markdown.** Git-tracked, no database.
- **Tasks reference source docs** via a `source:` field in frontmatter.
- **ADRs are numbered sequentially.** Never reuse a number.
- **PRD/TRD slugs should match** when they describe the same feature.
- **Task IDs are auto-incremented** by scanning existing files across all status directories.

## Document hierarchy

**Simple projects (single product):** Use flat structure — `docs/prd/YYYY-MM-DD-slug.md`

**Multi-product/project organizations:** Use hierarchy — `docs/prd/{product}/{project}/YYYY-MM-DD-slug.md`

Examples:
```
docs/prd/platform/auth/2025-06-01-oauth-migration.md
docs/prd/platform/billing/2025-06-15-usage-based-pricing.md
docs/trd/mobile/ios/2025-06-01-push-notifications.md
```

Skills auto-detect which convention is in use by checking existing directory structure. If no docs exist yet, ask the user which convention to follow.

## Optional CLI tools

Skills use these when available (graceful fallback if missing):

- `git` — diff-based reviews, branch detection
- `gh` — GitHub PR integration (comments, status checks)

## Skill Discovery Optimization (SDO)

The `description` field in skill frontmatter is what agents read to decide whether to invoke the skill. Write it as **triggering conditions** — not a workflow summary.

**Bad (workflow summary):**
> "Manages code review workflow — runs multi-pass review, deduplicates findings, posts structured report"

**Good (triggering conditions):**
> "Full PR review for GitHub and Bitbucket — use when asked to review a PR, before merging, or after a significant diff"

**Test:** Can an agent read the description and know exactly when to invoke this skill without reading the full SKILL.md? If not, rewrite it.

Descriptions that summarize workflow cause agents to treat the description as a substitute for the full skill content — they follow the summary instead of the steps. Triggering-condition descriptions cause agents to invoke the skill and follow the steps.

## Adding New Skills

1. Create `skills/ygs-<name>/SKILL.md`
2. Use the SDO principle for the `description` field
3. Extract shared content to `skills/shared/` if 2+ skills reference it
4. Reference shared files with the absolute installed path: `~/.claude/skills/you-got-skills/skills/shared/<file>.md`
5. Run `./setup install` to register the symlink
6. Update `README.md` skill table and Shared Modules table

For detailed guidance: `/ygs-write-skill`
