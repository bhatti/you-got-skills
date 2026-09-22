---
name: ygs-wizard
description: >
  Use when a procedure requires steps only a human can take: credential provisioning,
  CI secret setup, OAuth flows, third-party dashboard config, one-off migrations.
  Generates a guided interactive bash script the human runs — the agent authors it,
  the human executes it.
argument-hint: "[<procedure-description>]"
---

# Wizard

The wizard skill generates interactive bash scripts for procedures the agent cannot take autonomously — credentials, external dashboards, OAuth grants, manual migrations. The agent scopes and authors the script; the human runs it.

## When NOT to use

- The procedure can be done entirely with code (no human credentials, no dashboard steps) — just implement it directly
- The procedure is already documented in a runbook — link to it instead of regenerating
- Fewer than 2 stages — just write the steps as prose, no script needed

## Step 1: Scope

Read the repo to understand what credentials and services are involved:

```bash
grep -rl "process.env\|os.getenv\|ENV\[" src/ lib/ .env.example 2>/dev/null | head -10
cat .env.example 2>/dev/null || cat .env.template 2>/dev/null
cat docker-compose.yml 2>/dev/null | grep -E "environment|secrets" | head -20
ls .github/workflows/ 2>/dev/null | head -5
```

Build a complete list of required inputs: what's needed, from which service, in what format.

## Step 2: Map stages

For each required input, map:
- Service name
- Exact UI path or CLI command to get it
- Input format (URL, token, key, etc.)
- Where it lands (`.env` key name, GitHub secret name, config file path)

Show the stage list to the user in order. Confirm before scripting. Flag any stage you're unsure about.

## Step 3: Generate script

Generate from this template structure:

```bash
#!/usr/bin/env bash
# wizard: <procedure-name>
# Run with: bash <filename>
# DO NOT COMMIT if this script captures secrets interactively

set -euo pipefail

# === LIBRARY (do not edit this section) ===
STEPS_TOTAL=<N>
STEPS_DONE=0
step() { STEPS_DONE=$((STEPS_DONE+1)); echo ""; echo "[$STEPS_DONE/$STEPS_TOTAL] $1"; }
confirm() { read -r -p "$1 [y/N] " r; [[ "$r" =~ ^[Yy]$ ]] || { echo "Aborted."; exit 1; }; }
secret_input() { read -r -s -p "$1: " v; echo ""; echo "$v"; }
env_upsert() {
  local f="${ENV_FILE:-.env}" k="$1" v="$2"
  if grep -q "^$k=" "$f" 2>/dev/null; then sed -i.bak "s|^$k=.*|$k=$v|" "$f" && rm -f "$f.bak"
  else echo "$k=$v" >> "$f"; fi
}
ENV_FILE="${ENV_FILE:-.env}"
# === END LIBRARY ===

# === STAGES ===

# Stage 1: <service name>
stage_1() {
  step "<what this stage does>"
  echo "<exact UI path or CLI command>"
  echo "Navigate to: <URL or menu path>"
  local val
  val=$(secret_input "<Input label>")
  env_upsert "<ENV_KEY>" "$val"
  echo "✓ Saved to $ENV_FILE"
}

# ... one function per stage ...

main() {
  echo "=== <Procedure Name> Setup Wizard ==="
  echo "This wizard will configure: <list of services>"
  echo ""
  confirm "Ready to begin?"
  stage_1
  # stage_2 ...
  echo ""
  echo "=== Setup complete ==="
  echo "Run the application to verify: <verification command>"
}

main "$@"
```

Rules:
- Do not modify the LIBRARY section — it is fixed UX
- One function per stage below `# === STAGES ===`
- Confirmation gate (`confirm`) before any irreversible operation (pushing, deleting, deploying)
- Use `secret_input` for passwords and tokens (hidden entry)
- Use `env_upsert` for `.env` writes (idempotent)

## Step 4: Static verify

```bash
bash -n <script-path>   # syntax check
```

If `bash -n` exits non-zero: fix the syntax error and re-run. Do not write or deliver the script until `bash -n` exits 0.

Trace that every `secret_input` result is passed to `env_upsert` or a command — no captured values that land nowhere.

## Step 5: Write the script

```bash
# Reusable — commit to scripts/
scripts/setup-<name>.sh

# Ephemeral — add header and use scratch/
scratch/setup-<name>.sh   # header: "# EPHEMERAL — DO NOT COMMIT"
```

Make executable: `chmod +x <path>`.

## Completion

Report **DONE** with:
- Script path
- Stage list (N stages)
- Any stages where the exact UI path is uncertain (flag for human to verify before running)

Instruct the human: `bash <path>` (or `ENV_FILE=.env.local bash <path>` to target a non-default env file).
