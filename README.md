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

## Skill Pipeline

Recommended flow for a feature from idea to ship:

```
intent unclear?  →  /ygs-interview
                          ↓
      design unclear?  →  /ygs-brainstorm
                          ↓ (architectural path)
           /ygs-refine-prd  →  /ygs-refine-trd
                          ↓ (all paths)
     /ygs-estimate   /ygs-wbs   /ygs-spike
                          ↓
   /ygs-worktree  →  /ygs-implement  →  /ygs-qa
                          ↓
  /ygs-code-review / /ygs-review-pr / /ygs-receive-review
                          ↓
                     /ygs-ship
```

Cross-cutting (use any time): `/ygs-investigate`, `/ygs-triage`, `/ygs-observe`, `/ygs-parallel`

## Skills

### Requirements & Design

| Skill | Purpose |
|-------|---------|
| `/ygs-refine-prd` | Refine product requirements: structured questioning, code cross-reference, inline glossary |
| `/ygs-review-prd` | Critique a PRD for completeness, clarity, and feasibility |
| `/ygs-refine-trd` | Refine technical design: challenges architecture against code, inline glossary/ADRs |
| `/ygs-review-trd` | Critique a TRD for soundness, testability, and operational readiness |
| `/ygs-refine-architecture` | Evolve system architecture: deep-module principles, Design It Twice, inline glossary/ADRs |
| `/ygs-review-architecture` | Critique architecture for depth, scalability, and proportionality |

### Team Intelligence

Ambient awareness skills — gather signals across tracker and Slack, synthesize for the team.
Requires `.ygs/tracker.yml` (copy from `skills/shared/tracker-config-example.yml`).
Supports GitHub (`gh`) and JIRA/Bitbucket (`acli`) interchangeably. Slack optional.

| Skill | Purpose |
|-------|---------|
| `/ygs-standup` | Daily standup brief: per-person status, silence signals, dependency risks, discussion questions |
| `/ygs-risk-scan` | Ranked sprint risk report: stale work, review bottlenecks, dependency chains, capacity gaps |
| `/ygs-sprint-plan` | Sprint planning: propose scope from backlog, flag blockers and dependency conflicts before they enter sprint |

**Setup:**
```bash
cp ~/.claude/skills/you-got-skills/skills/shared/tracker-config-example.yml .ygs/tracker.yml
# edit .ygs/tracker.yml with your project details

# Required env vars (add to shell profile or secrets manager):
export JIRA_BASE_URL=https://yourorg.atlassian.net
export JIRA_EMAIL=you@company.com
export JIRA_API_TOKEN=ATATT3x...          # from id.atlassian.com/manage/api-tokens
export SLACK_BOT_TOKEN=xoxb-...           # optional, see skills/shared/slack.md
```

### Discovery

| Skill | Purpose |
|-------|---------|
| `/ygs-interview` | One-question-at-a-time intent extraction for underspecified asks — closes the want-vs-stated-want gap before any spec or code |
| `/ygs-brainstorm` | Design gate before writing code — classifies work (spike/bounded/architectural), explores approach, requires approval before any implementation |

### Planning & Execution

| Skill | Purpose |
|-------|---------|
| `/ygs-estimate` | Complexity-based estimation: t-shirt sizing, story points, capacity planning with KTLO buffers |
| `/ygs-wbs` | Work Breakdown Structure: vertical-slice tasks with HITL/AFK classification and dependency waves |
| `/ygs-spike` | Time-boxed spike to validate a hypothesis — feasibility, performance, or integration proof |
| `/ygs-implement` | Implement a task with scope guardrails, checkpoints, and deviation tracking |
| `/ygs-git` | Atomic commits, save-point pattern, semantic versioning, changelog hygiene |
| `/ygs-worktree` | Git worktree management — isolate feature work in a linked worktree before implementing |
| `/ygs-sync` | Bidirectional sync: keep design docs accurate as implementation evolves |
| `/ygs-ship` | Ship workflow: test, version bump, changelog, create PR |
| `/ygs-parallel` | Dispatch independent parallel subagents for 2+ tasks with no shared state or sequential dependencies |

### Reviews

| Skill | Purpose |
|-------|---------|
| `/ygs-review-pr` | Full PR review: fetch diff, four-domain analysis (correctness/security/API/SRE), ranked findings by severity, verdict (APPROVE/REQUEST_CHANGES/COMMENT) |
| `/ygs-review-deep` | Deep PR review: seven-domain analysis (adds performance, testing quality, architecture to the standard four), verdict + severity ranking |
| `/ygs-code-review` | Two-pass code review (critical/informational) with testing discipline |
| `/ygs-security-review` | Security audit + red-team adversarial analysis |
| `/ygs-sre-review` | Operational review: failure modes, observability, capacity, rollback, deploy gate |
| `/ygs-ui-review` | UI/UX review: accessibility, consistency, responsiveness |
| `/ygs-api-review` | API review: breaking changes, conventions, backwards compatibility |
| `/ygs-receive-review` | Receive and evaluate code review feedback — evaluate technically before implementing, push back with reasoning when warranted |

### Testing & Quality

| Skill | Purpose |
|-------|---------|
| `/ygs-qa` | QA testing with health scoring and issue taxonomy |
| `/ygs-uat` | User acceptance testing from customer perspective |

### Operations & Learning

| Skill | Purpose |
|-------|---------|
| `/ygs-observe` | Production instrumentation: structured logging, RED metrics, OpenTelemetry tracing, symptom-based alerting |
| `/ygs-investigate` | Disciplined debugging: feedback loop, hypotheses, root-cause enforcement, backward tracing, architectural handoff |
| `/ygs-write-skill` | Create or improve YGS skills — SDO-optimized descriptions, DRY shared modules, skill verification |
| `/ygs-triage` | Issue triage state machine: classify, reproduce, write agent briefs, track out-of-scope rejections |
| `/ygs-deprecate` | Deprecation and migration: Expand/Contract schema migrations, Strangler pattern, zombie code removal |
| `/ygs-learn` | Capture and surface operational learnings across sessions |
| `/ygs-retro` | Retrospective on recent work: keep/start/stop recommendations, git commit-type ratios, hotspot files |
| `/ygs-handoff` | Compress session into a handoff doc for the next session |
| `/ygs-changelog` | Generate changelog from git history and task files |

### Codebase Intelligence

Long-horizon analysis skills that look across many commits to detect patterns that per-PR review misses.

| Skill | Purpose |
|-------|---------|
| `/ygs-codebase-audit` | Post-merge codebase archaeology: analyze last N commits (default 1000) for hotspots (files changed >15% of commits), temporal coupling (hidden cross-module dependencies), duplicate abstractions, architecture drift, security archaeology, test coverage gaps, commit health (fix:commit ratio, large commits), and knowledge silos (bus-factor risk). Reuses ygs-review-deep (architecture) and ygs-security-review (security) protocols. |

**Arguments:** `[<repo-url>] [--commits 1000] [--focus all|architecture|security|tests|duplicates|health]`

**Usage:**
```bash
/ygs-codebase-audit                                          # analyze codebase in CWD
/ygs-codebase-audit https://github.com/org/repo             # audit a specific repo
/ygs-codebase-audit --commits 500 --focus architecture      # focused audit
```

Via Slack (with Formicary integration): `@bot audit` or `@bot codebase audit`

## Shared Modules

Reusable protocols in `skills/shared/` referenced by individual skills. Not invoked directly — skills pull them in as needed.

| Module | Purpose |
|--------|---------|
| `shared/completion-signals.md` | Canonical DONE/DONE_WITH_CONCERNS/BLOCKED signals |
| `shared/condition-based-waiting.md` | Replace sleep/timeouts with condition polling — polyglot patterns |
| `shared/definition-of-done.md` | Project-wide quality bar: Correctness / Quality / Integration / Ship-readiness |
| `shared/dep-audit.md` | Polyglot dependency audit commands |
| `shared/docs-discovery.md` | Canonical PRD/TRD/ADR discovery commands |
| `shared/ears-patterns.md` | EARS requirement patterns reference (six temporal forms) |
| `shared/functional-design.md` | Functional design principles and anti-patterns checklist |
| `shared/init.md` | Step 1 bootstrap for team-intelligence skills: config load + credential verify |
| `shared/output-format.md` | Slack-compatible bullet-row format enforced by standup/risk-scan |
| `shared/ownership-principles.md` | Senior engineer mindset: trust nothing, verify everything, judgment over obedience |
| `shared/refine-scaffold.md` | Common one-question-at-a-time protocol for all refine skills |
| `shared/review-scaffold.md` | Common protocol for all review skills: diff gather, severity tiers, verdict |
| `shared/risk-criteria.md` | HIGH/MEDIUM/LOW risk severity rules used by standup and risk-scan |
| `shared/slack.md` | Slack bot token setup and query patterns for team-intelligence signals |
| `shared/subagent-dispatch.md` | Patterns for crafting isolated, self-contained subagent prompts |
| `shared/test-runner.md` | Polyglot test runner: auto-detect and run Makefile/npm/cargo/pytest/go suites |
| `shared/testing-discipline.md` | Testing rules: no flaky tests, no sleeps, iron law, rationalization table |
| `shared/tracker.md` | DRY query patterns for GitHub (`gh`) and JIRA/Bitbucket (`acli`) |
| `shared/tracker-config-example.yml` | Starter config for GitHub/JIRA tracker integration |
| `shared/verification-gate.md` | Iron law: no completion claims without fresh verification evidence |

## Typical Workflow

```
/ygs-interview               → (optional) Extract confirmed intent before speccing
/ygs-brainstorm              → Design gate: classify work, explore approach, get approval
/ygs-refine-prd              → Question until requirements are precise
/ygs-review-prd              → Independent critique
/ygs-refine-trd              → Question until design is sound
/ygs-refine-architecture     → For larger changes: define system architecture
/ygs-review-trd              → Validate design
/ygs-estimate                → T-shirt sizing + story points + capacity planning
/ygs-wbs                     → Hierarchical work breakdown into vertical-slice tasks
/ygs-spike                   → Time-boxed experiment to validate risky unknowns
/ygs-worktree                → Isolate feature work in a linked worktree
/ygs-implement               → Build with discipline
/ygs-git                     → Commit with discipline (save-point pattern, atomic commits)
/ygs-parallel                → Dispatch independent tasks to parallel subagents
/ygs-triage                  → Classify issues, write agent briefs
/ygs-review-pr               → Full PR review (all four domains, ranked findings, verdict)
/ygs-receive-review          → Evaluate and respond to review feedback with technical reasoning
/ygs-code-review             → Two-pass review
/ygs-security-review         → Security + red-team
/ygs-sre-review              → Operational readiness
/ygs-qa                      → Test with health scoring
/ygs-uat                     → Customer perspective validation
/ygs-sync                    → Sync design docs with implementation reality
/ygs-ship                    → Test, version, PR (checks deploy freeze)
/ygs-observe                 → Instrument: logging, metrics, tracing, alerting
/ygs-deprecate               → Retire features and APIs safely
/ygs-learn                   → Capture atomic learnings as they happen
/ygs-retro                   → Learn and improve
/ygs-codebase-audit          → Post-merge archaeology (hotspots, drift, silos, test gaps)
```

## Project Conventions

When you use these skills in a project, they create:

```
your-project/
├── CONTEXT.md            # Domain glossary (created lazily by refine skills)
├── .out-of-scope/        # Rejected feature knowledge base (concept-slug.md)
├── docs/
│   ├── intent/           # Confirmed intent docs (ygs-interview output)
│   ├── prd/              # Product requirements (YYYY-MM-DD-slug.md)
│   ├── trd/              # Technical designs (YYYY-MM-DD-slug.md)
│   ├── adr/              # Architecture decision records (NNN-slug.md)
│   ├── architecture/     # System architecture / design docs
│   ├── spikes/           # Spike findings
│   ├── learnings/        # Operational learnings (YYYY-MM-DD-slug.md)
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
- **EARS notation** — Behavioral requirements use one of six temporal patterns (Ubiquitous, Event-driven `When`, State-driven `While`, Optional feature `Where`, Unwanted behaviour `If/Then`, Complex `While`+`When`). Every MUST requirement needs an `If/Then` failure mode. See [EARS reference](skills/shared/ears-patterns.md).
- **Scope guardrails** — Pause and ask when scope exceeds plan. No silent creep.
- **Deviation tracking** — "Design said X, did Y because Z" is captured explicitly.
- **Feedback loops first** — Debugging starts with a fast, deterministic pass/fail signal.
- **Design It Twice** — When an interface feels forced, explore 3+ radically different designs before committing.
- **HITL/AFK classification** — Every task is classified by whether an agent can complete it autonomously.
- **Inline glossary maintenance** — Domain terms resolved during refinement update CONTEXT.md immediately.
- **Durable over precise** — Agent briefs use behavioral contracts, not file paths that go stale.

## Design

- **Convention over configuration** — No config files, just directory conventions
- **Each skill is independent** — Use any subset, no required ordering
- **DRY shared scaffolding** — Common protocols (review, refine, test-run) in `skills/shared/`, referenced by individual skills
- **Compact** — Skills are 40-100 lines of markdown, scannable in 30 seconds
- **No external deps** — Just Claude Code + optionally `git` and `gh`
- **Strongly-typed language focus** — Interfaces + DI, clean seams, type safety
- **Extensible** — Add new skills by creating `skills/<name>/SKILL.md`
- **Organized by concern** — Skills use `references/` subdirectories for detailed checklists, keeping SKILL.md focused and scannable

## Optional CLI Tools

Skills use these when available (graceful fallback if missing):
- `git` — diff-based reviews, branch detection
- `gh` — GitHub CLI for PR workflows

## Related Projects

**[Superpowers](https://github.com/jessevictors/superpowers)** — Complementary skills with deep execution-time discipline: subagent-driven development, strict TDD, systematic debugging, and worktree isolation. Pairs well with YGS's planning and team-intelligence coverage.

## Credits

Inspired by and borrowing best practices from:
- [gstack](https://github.com/garrytan/gstack) — Comprehensive AI engineering workflow system
- [mattpocock/skills](https://github.com/mattpocock/skills) — Engineering skills with deep-module principles and structured refinement
- [OpenSpec](https://github.com/fission-ai/openspec) — Specification-driven development with RFC 2119 requirements and Given/When/Then scenarios
- [OpenSPDD](https://github.com/gszhangwei/open-spdd) — REASONS Canvas structured prompts and bidirectional design-code synchronization
- [BMAD-METHOD](https://github.com/bmadcode/BMAD-METHOD) — Scale-adaptive rigor, skill chaining, and ceremony-proportional-to-risk
- [Kiro](https://kiro.dev/docs/specs/) — dependency waves for parallel task execution
- [EARS (Easy Approach to Requirements Syntax)](https://alistairmavin.com/ears/) — six temporal patterns (Ubiquitous, Event-driven, State-driven, Optional feature, Unwanted behaviour, Complex) for unambiguous, testable requirements
- [Shahzad Bhatti's engineering blog](https://shahbhat.medium.com) — API design patterns, microservice security, fault tolerance, transaction boundaries, production readiness, and post-mortem practices

## License

MIT
