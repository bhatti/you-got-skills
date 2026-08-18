# Docs Discovery

Canonical commands for locating PRD, TRD, ADR, and architecture documents.
All skills that need design docs reference this file — do not inline these commands.

## Locate most recent documents

```bash
# PRD
ls -t docs/prd/*.md 2>/dev/null | head -5

# TRD
ls -t docs/trd/*.md 2>/dev/null | head -5

# ADRs
ls -t docs/adr/*.md 2>/dev/null | head -10

# Architecture doc
ls docs/architecture.md docs/arch/*.md 2>/dev/null | head -3

# CONTEXT.md (domain glossary)
cat CONTEXT.md 2>/dev/null
```

## Fallback — when no docs/ hierarchy

```bash
find . -name "*.prd.md" -o -name "*-prd.md" -o -name "PRD.md" 2>/dev/null | head -5
find . -name "*.trd.md" -o -name "*-trd.md" -o -name "TRD.md" 2>/dev/null | head -5
```

## Usage note

If no PRD/TRD is found and one is required, ask the user to point you to it or describe the requirements directly.
