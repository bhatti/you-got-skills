# Project Conventions

When ygs skills run in a project, they create and use this directory structure:

```
<project-root>/
├── docs/
│   ├── prd/                    # Product Requirements Documents
│   │   └── YYYY-MM-DD-<slug>.md
│   ├── trd/                    # Technical Requirements/Design Documents
│   │   └── YYYY-MM-DD-<slug>.md
│   ├── adr/                    # Architecture Decision Records
│   │   └── NNN-<slug>.md      # Sequential numbering (001, 002, ...)
│   └── architecture/           # System architecture documents
│       └── <slug>.md
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

## Optional CLI tools

Skills use these when available (graceful fallback if missing):

- `git` — diff-based reviews, branch detection
- `gh` — GitHub PR integration (comments, status checks)
