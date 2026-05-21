# you-got-skills

Production-grade SDLC skills for Claude Code. Full lifecycle from requirements through deployment — built for teams that ship real software.

These skills encode battle-tested engineering practices into repeatable AI workflows: structured refinement sessions, deep-module architecture principles, execution discipline with scope guardrails, and multi-dimensional reviews covering code, security, SRE, API, and UI concerns.

## Why

AI agents accelerate coding but also accelerate entropy. Without discipline, you get misaligned implementations, shallow architectures, and bugs that slip through. These skills fix that by embedding engineering fundamentals into every phase of the SDLC:

1. **Misalignment** — Refine skills question relentlessly until shared understanding. No building until both sides agree.
2. **Shallow design** — Architecture skills use deep-module principles (small interface, rich implementation) to prevent pass-through layers.
3. **Scope creep** — Implementation skill has guardrails: pause at 3+ unplanned files, checkpoints every 5 files.
4. **Missed bugs** — Reviews are multi-dimensional (code + security + SRE + API + UI) with MUST/SHOULD/MAY severity.
5. **No root cause** — Investigation skill enforces feedback loops and ranked hypotheses before any fix.

## Install

```bash
git clone https://github.com/bhatti/you-got-skills.git && cd you-got-skills && ./setup
```

Or install directly into `~/.claude/skills/`:

```bash
curl -fsSL https://raw.githubusercontent.com/bhatti/you-got-skills/main/setup | bash
```

### Update

Pull latest changes and refresh symlinks:

```bash
~/.claude/skills/you-got-skills/setup update
```

### Uninstall

```bash
~/.claude/skills/you-got-skills/setup uninstall
```

The setup script symlinks each skill into `~/.claude/skills/` where Claude Code discovers them automatically.

## Skills

### Requirements & Design

| Skill | Purpose |
|-------|---------|
| `/ygs-refine-prd` | Refine product requirements through structured questioning |
| `/ygs-review-prd` | Critique a PRD for completeness, clarity, and feasibility |
| `/ygs-refine-trd` | Refine technical design — challenges architecture against codebase |
| `/ygs-review-trd` | Critique a TRD for soundness, testability, and operational readiness |
| `/ygs-refine-architecture` | Evolve system architecture using deep-module principles |
| `/ygs-review-architecture` | Critique architecture for depth, scalability, and proportionality |

### Planning & Execution

| Skill | Purpose |
|-------|---------|
| `/ygs-estimate` | Complexity-based estimation: t-shirt sizing, story points, capacity planning with KTLO buffers |
| `/ygs-wbs` | Work Breakdown Structure: hierarchically decompose into INVEST-compliant vertical-slice tasks |
| `/ygs-spike` | Time-boxed spike to validate a hypothesis — feasibility, performance, or integration proof |
| `/ygs-implement` | Implement a task with scope guardrails, checkpoints, and deviation tracking |
| `/ygs-sync` | Bidirectional sync: keep design docs accurate as implementation evolves |
| `/ygs-ship` | Ship workflow: test, version bump, changelog, create PR |

### Reviews

| Skill | Purpose |
|-------|---------|
| `/ygs-code-review` | Two-pass code review (critical/informational) with testing discipline |
| `/ygs-security-review` | Security audit + red-team adversarial analysis |
| `/ygs-sre-review` | Operational review: failure modes, observability, capacity, rollback |
| `/ygs-ui-review` | UI/UX review: accessibility, consistency, responsiveness |
| `/ygs-api-review` | API review: breaking changes, conventions, backwards compatibility |

### Testing & Quality

| Skill | Purpose |
|-------|---------|
| `/ygs-qa` | QA testing with health scoring and issue taxonomy |
| `/ygs-uat` | User acceptance testing from customer perspective |

### Operations & Learning

| Skill | Purpose |
|-------|---------|
| `/ygs-investigate` | Disciplined debugging: feedback loop, hypotheses, root-cause enforcement |
| `/ygs-retro` | Retrospective on recent work: keep/start/stop recommendations |

## Typical Workflow

```
/ygs-refine-prd              → Question until requirements are precise
/ygs-review-prd              → Independent critique
/ygs-refine-trd              → Question until design is sound
/ygs-refine-architecture     → For larger changes: define system architecture
/ygs-review-trd              → Validate design
/ygs-estimate                → T-shirt sizing + story points + capacity planning
/ygs-wbs                     → Hierarchical work breakdown into vertical-slice tasks
/ygs-spike                   → Time-boxed experiment to validate risky unknowns
/ygs-implement               → Build with discipline
/ygs-code-review             → Two-pass review
/ygs-security-review         → Security + red-team
/ygs-sre-review              → Operational readiness
/ygs-qa                      → Test with health scoring
/ygs-uat                     → Customer perspective validation
/ygs-sync                    → Sync design docs with implementation reality
/ygs-ship                    → Test, version, PR
/ygs-retro                   → Learn and improve
```

## Project Conventions

When you use these skills in a project, they create:

```
your-project/
├── docs/
│   ├── prd/              # Product requirements (YYYY-MM-DD-slug.md)
│   ├── trd/              # Technical designs (YYYY-MM-DD-slug.md)
│   ├── adr/              # Architecture decision records (NNN-slug.md)
│   ├── architecture/     # System architecture / design docs
│   ├── spikes/           # Spike findings
│   └── threat-models/    # Security threat models
└── tasks/
    ├── backlog/          # Not started (task-NNN.md)
    ├── in-progress/      # Being worked on
    └── done/             # Completed
```

**Multi-product hierarchy:** For organizations with multiple products/projects, docs support nesting: `docs/prd/{product}/{project}/YYYY-MM-DD-slug.md`. Skills auto-detect which convention is in use.

Tasks use **directory-as-status**: moving a file changes its status. All git-tracked, no database.

## Engineering Principles

These principles are embedded throughout the skills:

- **Deep modules over shallow** — Small interfaces with rich implementation. Pass-throughs are a smell.
- **Testability as design signal** — If you can't test without mocking internal code, the design is wrong.
- **Stubs at boundaries only** — Real implementations for owned code; stubs only at 3rd-party/OS edges.
- **Vertical slices** — Tasks cut through all layers end-to-end, not horizontal layer-by-layer.
- **Proportionality** — Complexity must be justified with data, not hypothetical future needs.
- **MUST/SHOULD/MAY severity** — Reviews classify findings by impact and reversibility.
- **Scope guardrails** — Pause and ask when scope exceeds plan. No silent creep.
- **Deviation tracking** — "Design said X, did Y because Z" is captured explicitly.
- **Feedback loops first** — Debugging starts with a fast, deterministic pass/fail signal.

## Design

- **Convention over configuration** — No config files, just directory conventions
- **Each skill is independent** — Use any subset, no required ordering
- **Compact** — Skills are 40-100 lines of markdown, scannable in 30 seconds
- **No external deps** — Just Claude Code + optionally `git` and `gh`
- **Strongly-typed language focus** — Interfaces + DI, clean seams, type safety
- **Extensible** — Add new skills by creating `skills/<name>/SKILL.md`
- **Organized by concern** — Skills use `references/` subdirectories for detailed checklists, keeping SKILL.md focused and scannable

## Optional CLI Tools

Skills use these when available (graceful fallback if missing):
- `git` — diff-based reviews, branch detection
- `gh` — GitHub CLI for PR workflows

## Credits

Inspired by and borrowing best practices from:
- [gstack](https://github.com/garrytan/gstack) — Comprehensive AI engineering workflow system
- [mattpocock/skills](https://github.com/mattpocock/skills) — Engineering skills with deep-module principles and structured refinement
- [OpenSpec](https://github.com/fission-ai/openspec) — Specification-driven development with RFC 2119 requirements and Given/When/Then scenarios
- [OpenSPDD](https://github.com/gszhangwei/open-spdd) — REASONS Canvas structured prompts and bidirectional design-code synchronization
- [BMAD-METHOD](https://github.com/bmadcode/BMAD-METHOD) — Scale-adaptive rigor, skill chaining, and ceremony-proportional-to-risk
- [Kiro](https://kiro.dev/docs/specs/) — EARS notation for testable requirements, dependency waves for parallel task execution
- [Shahzad Bhatti's engineering blog](https://shahbhat.medium.com) — API design patterns, microservice security, fault tolerance, transaction boundaries, production readiness, and post-mortem practices

## License

MIT
