# OpenClaw Design Patterns & Architecture Reference

<!-- last-verified: 2026-04-19 | openclaw: 2026.4.14 -->

> **Purpose:** Persistent reference for understanding OpenClaw's architecture,
> design patterns, and operational conventions. Derived from production
> deployment experience. This is platform-level knowledge, not specific to any
> particular agent or business domain.

---

## 1. Architecture Overview

OpenClaw is a local-first personal AI agent framework. Its core components:

```
Messaging Channels (WhatsApp, Telegram, Slack, Discord, Signal, etc.)
         |
    Gateway (ws://127.0.0.1:18789)
         |
    +--- Agent sessions (main + isolated/sandbox)
    +--- Cron scheduler
    +--- Webhook ingestion
    +--- Memory index (SQLite + vector)
```

**Key architectural decisions:**

- **Gateway as single control plane.** One daemon process manages sessions, channels, tools, cron, and events. No microservices, no external databases.
- **Workspace as knowledge anchor.** The `~/.openclaw/workspace/` directory is the persistent source of truth for agent identity, instructions, skills, and memory. Files in the workspace are injected into agent context at session start.
- **File-based everything.** Memory, skills, config overrides, and audit logs are Markdown files. No proprietary formats. Version-controllable with git.
- **Local-first execution.** The Gateway runs on the host machine. Tools execute locally. LLM calls go outbound but all data stays local.

### Context Injection Order

On every new session, OpenClaw loads workspace files into agent context in this order:

1. `SOUL.md` — identity, security boundaries, hard rules
2. `AGENTS.md` — tool policies, confirmation gates, cron schedule
3. `TOOLS.md` — prose instructions for available tools
4. `USER.md` — operator context
5. `MEMORY.md` — curated long-term facts
6. `memory/YYYY-MM-DD.md` — today + yesterday daily logs
7. Skills matching the current trigger/context

---

## 2. Skill Design Patterns

### SKILL.md Format

Every skill is a directory under `skills/` containing a `SKILL.md` file:

```
skills/
  daily-greeting/
    SKILL.md
  weekly-report/
    SKILL.md
  ...
```

#### Frontmatter (YAML)

```yaml
---
name: skill-name            # Required. Lowercase, hyphenated.
description: One-line summary of what the skill does.  # Required.
metadata:
  openclaw:
    emoji: "\U0001F4E6"     # Optional. Display emoji.
    requires:
      bins: [git]           # Optional. CLI binaries that must exist.
      env: [VAR_NAME]       # Optional. Required env vars.
    # Other optional fields:
    # always: true           — Skill always active (not trigger-dependent)
    # skillKey: custom-key   — Override invocation key
    # os: ["linux"]          — OS restrictions
    # homepage: "https://..."
---
```

#### Body Structure (Runbook Pattern)

A well-structured SKILL.md follows this pattern:

```markdown
# Skill Title

## When to Use
How this skill is triggered (CRON schedule, message pattern, manual).

## Configuration
External resource references or config keys needed.

## Workflow
### Step 0: Load Config        <-- Most skills start here
### Step 1: [First action]
### Step 2: [Second action]
...
### Step N: Operator Summary   <-- Most skills end here

## Edge Cases
Bullet list of known edge cases and how to handle each.

## Rules (or ## NEVER)
Hard constraints that must not be violated.

## Output
One-line summary of artifacts produced.
```

### Skill Design Rules

1. **Step 0: Load Config.** Skills that depend on external configuration should begin by reading config. If any required value is missing, the skill aborts and alerts the operator. This pattern ensures skills never run with stale or incomplete configuration.

2. **Operator summary as final step.** Every skill that mutates data should send a structured summary to the operator (via Telegram or other trusted channel) as its last step. This is the primary observability mechanism.

3. **SYSTEM_LOG.md for audit trail.** Every skill logs its actions (successes and failures) to `SYSTEM_LOG.md` with a timestamped header format: `## YYYY-MM-DD HH:MM UTC — [Action Name]`.

4. **Idempotency guards.** Skills that process batches must check for "already processed" markers to prevent double-processing on re-runs.

5. **Fail-safe on API errors.** If an external API call fails, the skill stops processing, reports what succeeded and what remains, and alerts the operator. It does NOT retry blindly.

6. **NEVER clauses.** Each skill should end with explicit prohibitions — non-negotiable constraints the agent must follow regardless of context.

### Skill Precedence (Load Order)

When skill names conflict:
1. `<workspace>/skills/` (highest — per-agent workspace skills)
2. `~/.openclaw/skills/` (shared across agents)
3. Bundled skills (lowest)

### Skill Lifecycle Checklist

When adding or editing a workspace skill, verify these:

1. **Format validation** — frontmatter YAML, runbook body structure
2. **Config hygiene** — no hardcoded credentials, tokens, or secrets
3. **Skill loading** — `openclaw skills list` confirms the skill loads without errors
4. **Cron sync** — if the skill has a cron trigger, verify the cron job prompt matches the skill name and schedule
5. **Sandbox sync** — verify any tools the skill uses are in `tools.sandbox.tools.allow`
6. **Tests** — run tests to catch regressions
7. **Git review** — `git diff` to review all changes before committing

### Verbatim Template Embedding

Customer-facing messages sent by cron jobs on smaller models (haiku-class, mini-class) must have their templates embedded **verbatim** in the cron prompt — not just referenced in SKILL.md. Without the literal template text in the prompt, smaller models will freestyle the message content: rewriting phrasing, omitting fields, or adding unsolicited content.

**Pattern:** Include the full message template in the cron prompt with the instruction: `"Use this EXACT template — substitute all {variables} with actual values from the data you read."` The template should contain every line, every field label, and every placeholder exactly as it should appear in the final message.

This applies to both group messages (announcements, blasts) and individual DMs (reminders, confirmations). On-demand skills triggered via frontier models (Sonnet-class) are less prone to this but should still include a `"Send this message VERBATIM"` reinforcement before the template block in SKILL.md.

### `exec curl -sf` Literal Pattern

Sandbox sessions cannot make HTTP calls via prose description. Writing "POST to the endpoint at `[URL]`" in a SKILL.md will cause smaller models (codex-mini class, haiku) to describe the call or reason about it rather than execute it. The SKILL.md must include the literal `exec curl` command.

**Pattern:**
```
Step 1: [Action name]
Run: exec curl -sf -X POST http://localhost:[PORT]/api/internal/[ENDPOINT] \
  -H "Content-Type: application/json" \
  -d '{"key": "[VALUE]"}'
```

The `-s` (silent) and `-f` (fail on HTTP error) flags are non-optional:
- `-s` prevents curl progress output from polluting the agent's context
- `-f` ensures non-2xx responses surface as tool errors rather than silent HTML error pages

---

## 3. Session & Sandbox Model

### Session Types

| Type | Context | Tools | Use Case |
|------|---------|-------|----------|
| **Main session** | Runs on host | Full tool access | Operator commands, system ops, cross-channel coordination |
| **Sandbox session** | Docker container | Restricted tools, workspace-only file access | Customer-facing channels, isolated cron jobs |

### Lane Queue (Concurrency Control)

OpenClaw uses a **"default serial, explicit parallel"** model:

- Each session has its own "lane" (serial queue)
- Messages within a session are processed one at a time
- Cross-session communication uses `sessions_send` (fire-and-forget or with timeout)
- This prevents state corruption from concurrent mutations to the same resource

### Delegation Pattern (Sandbox <-> Main)

When a sandbox session needs host-level capabilities:

1. Sandbox calls `sessions_send` to the main session with structured data
2. Main session processes the request (e.g., reads files, updates external systems)
3. Sandbox polls for the result (typically by re-reading the data source)

**Config prerequisites for `sessions_send` from isolated cron (v2026.3.8+):**

OpenClaw v2026.3.8 added "owner-auth hardening" that put `sessions_send` on the `SUBAGENT_TOOL_DENY_ALWAYS` list and defaulted sandbox session visibility to `"spawned"` (tree-only). Two `openclaw.json` entries are required to restore delegation from isolated cron sessions:

1. `tools.subagents.tools.alsoAllow: ["sessions_send"]` — overrides the deny list
2. `agents.defaults.sandbox.sessionToolsVisibility: "all"` — lets sandboxed sessions see `agent-main-main`

If either is missing, isolated cron jobs silently fail to delegate.

### Session Naming Convention

Main session key: `agent:main:main`

Channel-specific DM sessions (`agent:main:telegram:direct:{id}`, `agent:main:whatsapp:direct:{phone}`) are **sandboxed** and must NOT be used as delegation targets for cross-channel operations.

### Cross-Channel Delegation Rule

Sandboxed sessions can reply within their own channel (text output auto-delivers), but CANNOT send to a different channel unless the `message` tool is explicitly added to `tools.sandbox.tools.allow`.

**Default posture:** Do NOT add `message` to sandbox tools unless you have the three-layer DM delivery protection (below) in place. Adding `message` grants cross-channel messaging to ALL sandboxed sessions (customer DMs, isolated crons, groups). Without the delivery protection layers, this creates a large blast radius.

**When to add `message` to sandbox:** If your agent needs to send DMs from sandbox sessions (e.g., customer reminders from isolated cron jobs), add `message` to `tools.sandbox.tools.allow` AND implement all three protection layers below. After changing the allowlist, run `openclaw sandbox recreate --all`.

### Three-Layer DM Delivery Protection

Preventing unintended DM delivery requires three independent defense layers. Any single layer has gaps — all three are needed:

1. **`message` tool excluded from sandbox allowlist.** Sandbox sessions cannot initiate cross-channel messages. They can only reply within their own channel via text output auto-delivery.
2. **`delivery.mode: "none"` on all cron jobs.** Without this, cron output auto-delivers to the main session's last active channel. If the operator's last interaction was in a customer DM, cron output leaks to that customer.
3. **Hourly auto-prune of temporary DM sessions.** An hourly checkpoint script removes stale `[channel]:direct` sessions created by sandbox activity. This prevents session accumulation from contaminating the main session's routing table.

**Why all three?** Layer 1 prevents sandbox-initiated sends but doesn't stop auto-delivery from cron. Layer 2 stops cron auto-delivery but doesn't prevent `sessions_send` leaks. Layer 3 cleans up session artifacts that layers 1-2 miss.

### Session Type to Tool Access Matrix

| Session Type | Sandboxed | Tool Access | Notes |
|---|---|---|---|
| Main (`agent:main:main`) | No | Full | Operator commands, system ops |
| Channel DM (`agent:main:[channel]:direct:[id]`) | Yes | Restricted | Customer interactions |
| Isolated cron | Yes | Restricted | Scheduled jobs — commonly mistaken as unsandboxed |
| Group (`agent:main:[channel]:group:[id]`) | Yes | Restricted | Mention-triggered responses |

**Key misconception:** Isolated cron sessions ARE sandboxed. They run in Docker containers with the same tool restrictions as customer DM sessions. This is why cron jobs that need host-level access (file writes, system scripts) must target the main session via `systemEvent`.

### Sandbox Tool Allowlist Enforcement

Changes to `tools.sandbox.tools.allow` in `openclaw.json` require rebuilding sandbox containers:

```bash
openclaw sandbox recreate --all
```

**This is NOT a hot-reload operation.** Unlike workspace files (which hot-reload via file watcher), sandbox tool allowlists are baked into the container at creation time. Without `recreate`, existing containers continue running with the old allowlist. Tools not in the allowlist **silently fail** — the agent receives no error, the tool simply does nothing. This makes allowlist drift difficult to diagnose (the skill appears to succeed but produces no output).

### Context Compaction

When a session's context window fills up, OpenClaw automatically summarizes older messages. For cron-triggered skills, this rarely matters since each cron invocation starts a fresh session. For long-running DM conversations, the agent should rely on memory files and external data reads rather than conversation history.

### Spawn-and-Delegate Pattern

For on-demand skills that don't need main session capabilities, main can spawn an ephemeral isolated session on a cheaper model and delegate the work:

1. Main receives operator request
2. Main uses `sessions_spawn` with a cheaper model to create an ephemeral session
3. Main sends the request via `sessions_send` to the new session
4. The isolated session runs the skill, returns structured JSON
5. Main relays the result to the operator

**When to use:** On-demand skills with no vision, no cross-channel messaging, and no host-level tool requirements. The skill must support a "Delegation Mode" that returns structured data instead of prose.

**When NOT to use:** Skills requiring cross-channel sends or complex multi-resource mutations.

---

## 4. Cron Job Patterns

### Cron vs. Heartbeat

| Mechanism | Precision | Use Case |
|-----------|-----------|----------|
| **Cron** | Exact schedule (crontab syntax) | Hard deadlines, time-sensitive tasks |
| **Heartbeat** | Approximate (drifts) | Health checks, system monitoring, non-time-critical tasks |

**Rule: Never use `act:wait` or internal loops for delays >1 minute. Use `cron:add` with a one-shot schedule instead.**

### Cron Design Rules

1. **Self-contained execution.** Each cron job runs in its own session context. It cannot see other sessions' conversation history. It must read all state from files or external sources.

2. **No sub-agent spawning from cron.** Sub-agents lose context and can duplicate work. Keep cron jobs self-contained.

3. **Main vs. isolated targeting.** System maintenance tasks (cleanup, health checks) should target the main session via `systemEvent` so the primary agent has full tool access. Customer-facing or data-processing tasks can run in isolated sessions.

4. **Memory files for cross-session state.** Cron jobs in isolated sessions use `memory/` files to persist state between invocations. The MEMORY.md file should contain timezone and other facts the agent needs.

5. **Cron isolation.** Use `sessionTarget: "isolated"` for cron jobs to prevent cron output from leaking to customer channels via the main session's channel context.

6. **Version-controlled cron snapshots.** Periodically snapshot cron jobs to a `cron/jobs.json` file in the workspace and track in git. This provides an audit trail and makes it easy to detect drift.

7. **`delivery.mode: "none"` on all cron jobs.** Without this, cron output auto-delivers to the main session's last active channel. If the operator's last message was in a customer DM, cron output leaks to that customer. Set `delivery.mode: "none"` on every cron job and use explicit `message` tool calls (from main session) for intentional delivery.

8. **Shell-script-only jobs use `systemEvent`, not `agentTurn`.** If a cron job only runs a shell script with no LLM reasoning needed (backups, health checks, file initialization), use `sessionTarget: "main"` with `triggerType: "systemEvent"`. This runs the script directly on the host without spinning up an agent session — saving tokens and latency. Reserve `agentTurn` for jobs that require LLM reasoning.

### One-Shot Auto-Trigger Pattern

When a server-side event should immediately trigger an agent skill instead of waiting for the next cron poll, create a fire-and-forget one-shot cron job:

```
Server event → openclaw cron add --name {skill}-auto-{uuid}
  --at 1m --session isolated --no-deliver
  --delete-after-run --message "Run {skill}..."  --timeout 60000
→ Agent runs skill in sandbox ~1 min later → Job self-deletes after run
```

**Key requirements:**
- **UUID in job name** prevents name collisions from rapid-fire triggers
- **`--delete-after-run`** prevents job accumulation
- **Keep the fallback cron** as a safety net — the auto-trigger claims rows first via atomic `UPDATE...RETURNING`, leaving nothing for the fallback cron
- **Atomic claims** prevent duplicate actions when auto-trigger and fallback cron overlap. Use `UPDATE...RETURNING` (not SELECT-then-UPDATE) for any claim-before-act pattern

**When to use:** Server-side events (config changes, status updates, admin actions) that need near-real-time agent response. The server creates the one-shot job via the OpenClaw CLI or API.

**When NOT to use:** High-frequency events (>1/minute) — use a webhook-to-main-session pattern instead.

### Memory-Tag-Then-Batch Pattern

When there is no real-time API to query historical data, use this two-phase pattern:

1. **Tag phase (real-time):** During normal agent operation, silently log structured entries to the daily memory file using a bracketed tag (e.g., `[FEEDBACK]`, `[INCIDENT]`). No interjection or follow-up — passive observe and log.
2. **Batch phase (cron):** A daily cron job reads yesterday's memory file, extracts tagged entries, enriches them with data from other sources, and writes to a persistent store.

**Sandbox caveat:** Channel-specific DM sessions are sandboxed and may have no file-write capability. When a sandbox session identifies data to tag, it should delegate the memory write to `agent-main-main` via `sessions_send` (fire-and-forget).

### Model Routing

Cost optimization via strategic model tiering:

| Tier | Model Class | Use Case |
|------|-------------|----------|
| Customer-facing crons | Smallest viable (haiku-class) | Template fidelity, low cost, verbatim output |
| Light automation | Mini-class | Data extraction, formatting, tagging |
| Medium complexity | Mid-tier | Multi-step workflows, conditional logic |
| Complex / main session | Frontier | Reasoning, multi-resource mutations, operator interaction |

- Per-cron model override: `openclaw cron edit <id> --model <model>`
- Heartbeat model override: `heartbeat.model` in `openclaw.json`

**Provider caveat:** `cacheRetention` is Anthropic-only. OpenAI and other providers silently ignore the parameter — no error, no caching behavior applied. When using mixed providers across tiers, only Anthropic-model cron jobs benefit from prompt caching.

### Cron Health Audit

Use HEARTBEAT.md to perform periodic audits of all cron jobs:

1. Read the cron snapshot file (`cron/jobs.json`)
2. For each job, check: enabled/disabled state, `consecutiveErrors`, and `lastRunStatus`
3. Flag disabled jobs that should be active, and jobs with recent errors
4. Report anomalies to operator via trusted channel

This catches silent failures early — a cron job that errored or was accidentally disabled will be surfaced within the hour rather than discovered days later.

### Cron Schedule Auto-Sync

When cron schedules are driven by external configuration (e.g., a config sheet or config file that stores order deadlines, blast times, or reminder windows), schedule drift is inevitable — someone updates the config but forgets to update the cron registration.

**Pattern:** An hourly system cron (not OpenClaw cron) compares config-derived schedule expressions against live `openclaw cron list` output and auto-fixes discrepancies. The sync script should only modify schedule expressions — never payloads, models, or other job parameters.

```bash
# Example: sync-cron-from-config.sh (runs via system crontab, not OpenClaw)
# 1. Read config source (file, API, sheet) for expected schedules
# 2. Compare against `openclaw cron list --json`
# 3. For each mismatch: `openclaw cron edit <id> --schedule "<new_expression>"`
# 4. Log changes to SYSTEM_LOG.md
# DRY_RUN=1 for preview mode
```

---

## 5. Tool Policies & Security

### Tool Configuration Layers

Three layers of tool policy, from broadest to most specific:

1. **`openclaw.json`** — master config. Defines allowed/denied tools at the gateway level. Protects against workspace-level overrides.
2. **`AGENTS.md`** — workspace-level policies. Defines which tools are enabled, which require confirmation, and sandbox restrictions.
3. **`SOUL.md`** — hard security boundaries. Non-negotiable constraints that override everything at the reasoning level.

### Docker Sandbox Image

The sandbox uses `openclaw-sandbox:bookworm-slim`, built from a Dockerfile. The base `debian:bookworm-slim` image has **no `curl`** — skills that use `exec curl` from sandbox will fail silently without it.

**Build:** Create a `docker/sandbox.Dockerfile` that installs `curl` on top of the base image. Run your build script (idempotent, Docker layer cache makes it fast). After rebuilding the image, run `openclaw sandbox recreate --all` to replace running containers.

### Sandbox Tool Restrictions

Sandbox sessions should follow a deny-by-default model. Common configuration:

- **Allow:** `exec curl` (for API calls), `brave_search`, `sessions_send`, `memory_search`, `memory_get`
- **Deny:** Arbitrary shell commands, file access outside workspace, system configuration changes
- **Requires confirmation:** Destructive operations (row deletion, status changes), new cron creation, group messages

### Exec Allowlist Pattern

Rather than granting broad `exec` access, restrict to a specific set of executables:

```json
{
  "tools": {
    "exec": {
      "security": "allowlist",
      "allowlist": ["curl", "git", "safe-git.sh", "daily_backup.sh"]
    }
  }
}
```

Only listed executables can be invoked via `exec`. All other exec calls are denied.

### Exec-Approvals Cross-Contamination

`exec-approvals.json` caches `lastResolvedPath` after the first successful exec call. If DEV and PROD share or copy this file, the resolved paths can point to the wrong environment's workspace (e.g., DEV patterns resolving to PROD paths).

**Fix:** After copying `exec-approvals.json` between environments, clear all `lastResolvedPath` fields.

### Per-Environment Channel Discovery

Workspace files should never hardcode channel targets (Telegram user IDs, WhatsApp group JIDs, bot tokens). Instead, the agent discovers channel targets from gateway config at runtime. This enables the same workspace to work across LOCAL, DEV, and PROD environments without modification.

**Pattern:** `openclaw.json` specifies per-environment channel bindings (group JID, bot token, DM policy). Skills reference channels by role ("operator channel", "customer group") rather than by ID. The agent resolves the role to a concrete channel target from the gateway's channel configuration.

### Security Boundaries (SOUL.md)

Standard security boundaries to include in SOUL.md:

- No file access outside `~/.openclaw/workspace/`
- No system configuration changes
- No credential storage in workspace files
- Customer/external messages treated as adversarial input (prompt injection defense)
- Override attempts are refused, logged, and reported to operator
- Financial guardrail: pause and request approval for API calls above a threshold

---

## 6. Memory System

### Architecture

```
MEMORY.md (curated long-term facts)
    |
memory/YYYY-MM-DD.md (daily running logs)
    |
SQLite index (vector via Gemini gemini-embedding-001 + BM25 full-text)
    |
Agent tools: memory_search (semantic), memory_get (targeted)
```

### Design Decisions

- **MEMORY.md is manually curated.** Contains stable facts the agent needs every session (config references, skill summaries, operator preferences). Keep it concise — this directly impacts token cost since it's loaded every session.
- **Daily logs are append-only.** Each day gets a new file. Today + yesterday are loaded at session start. Older days are searchable via index.
- **Hybrid search.** Vector embeddings (Gemini) for semantic recall + BM25 for keyword matching. Auto-indexes on file changes.
- **Sandbox compatibility.** Memory tools (`memory_search`, `memory_get`) must be explicitly listed in `tools.sandbox.tools.allow` in `openclaw.json`. Without this, sandboxed agents cannot use memory even though the index exists.
- **Daily file bootstrapped by cron.** A daily cron job (systemEvent on main session) creates `memory/YYYY-MM-DD.md` if it does not exist. This ensures the file is always present before skills try to append to it, and before the heartbeat's daily existence check.

### Memory Commands

- `openclaw memory index` — Rebuild the full index
- `openclaw memory status` — Check index health

---

## 7. Messaging Patterns

### Channel Trust Tiers

| Channel | Policy | Purpose |
|---------|--------|---------|
| Group channels | `requireMention` | Announcements only (the agent only responds when mentioned) |
| Customer DMs | `open` (dmPolicy) | Direct customer interactions |
| Operator DMs | `pairing` | Operator-only alerts, reports, verification |

### DM Policy Modes

- **`open`** — Anyone can DM the agent and get a response
- **`pairing`** — Unknown senders receive a pairing code; must be approved before full access

### Message Design Rules

1. **Group messages require confirmation.** Any message to a group channel should require operator approval (confirmation gate in AGENTS.md).
2. **DMs are fire-and-forget.** Customer DMs are sent without confirmation but logged to SYSTEM_LOG.md.
3. **Operator always gets a summary.** Every skill that sends customer messages also sends a rollup to the operator via a trusted channel.
4. **Never mix channels in sandbox.** A skill that runs in sandbox must not send messages to channels outside its own. It delegates to main session if cross-channel notification is needed.

### Pre/Post-Write Guardrails

For skills that mutate external data (updating rows, writing records), use pre/post-write verification to catch silent corruption:

- **Before write:** Re-read the target row/record and verify identity fields (e.g., record ID, timestamp + name) match the intended target. Row insertions, concurrent edits, or stale caches can cause off-by-one writes.
- **After write:** Re-read the written cells/record and confirm the values persisted correctly. API partial failures, rate limits, or schema mismatches can cause silent drops.

This adds two extra reads per write but catches data corruption that would otherwise go undetected until a customer reports it.

### Date Computation Verification

When skills compute future dates (pickup dates, deadlines, reminder windows), use two-layer validation:

1. **Static tests (build time):** Verify the skill references the correct config keys for date parameters and includes a verification step in its workflow.
2. **Runtime Verify step:** After computing the date, the skill checks the result against the expected day-of-week or range. If the computed date is wrong (LLM arithmetic errors are common), the Verify step adjusts forward to the next valid date.

This catches errors that static tests cannot — the LLM may correctly reference the right config key but still miscalculate the date.

---

## 8. Environment Architecture

### Three-Environment Model

| Aspect | LOCAL | DEV | PROD |
|--------|-------|-----|------|
| Location | Developer workstation | Droplet | Droplet |
| State directory | `~/.openclaw-dev/` (symlinked) | `~/.openclaw-dev/` (rsync'd) | `~/.openclaw/` |
| Gateway port | 19001 (auto via `--dev`) | 19001 (auto via `--dev`) | 18789 |
| Gateway flag | `openclaw --dev` | `openclaw --dev` | `openclaw gateway start` |
| Workspace source | Git repo (symlink) | rsync from LOCAL | rsync from LOCAL or DEV |
| Channels | None or test | None or sandbox test channel | Live channels |
| Cron | None | Idle (cost control) | Live cron jobs |

- **LOCAL** is the developer's workstation. The workspace is the git repo, symlinked to `~/.openclaw-dev/workspace`. Edits are immediately visible to the DEV gateway — no promote step needed. Used for rapid iteration.
- **DEV** is an idle staging environment on the Droplet. Its workspace is rsync'd from LOCAL. Cron jobs are registered but idle by default (zero token cost). Used for integration testing with real channels in sandbox mode.
- **PROD** is the live deployment on the Droplet. Its workspace is a deployed artifact — never edited directly. Cron jobs are live.

### DEV Auto-Shutdown

To prevent token burn from forgotten DEV instances, use a system cron (not OpenClaw cron) as a watchdog:

```bash
# System crontab: check every 5 minutes, shut down DEV if idle >60 minutes
*/5 * * * * /path/to/scripts/dev-watchdog.sh --idle-threshold 60
```

The watchdog checks the DEV gateway's last activity timestamp. If no operator interaction has occurred within the threshold, it stops the DEV gateway. This is critical — DEV cron jobs left running silently consume tokens with no production value.

### Config Drift Prevention

Periodically diff DEV vs PROD configs to catch unintended divergence:

```bash
diff <(jq 'del(.channels, .cron)' ~/.openclaw-dev/openclaw.json) \
     <(jq 'del(.channels, .cron)' ~/.openclaw/openclaw.json)
```

Exclude environment-specific keys (channels, cron schedules) and focus on structural config: tool policies, sandbox settings, model routing, and security boundaries. Non-env-specific differences should be investigated and reconciled.

### Environment Behavior for DMs

- **PROD:** Messages sent to real recipients
- **DEV/LOCAL:** Messages redirected to operator's own phone/account with a `[DEV]` prefix, or restricted via `dmPolicy: "pairing"`
- **TEST:** No-op, returns `{ok: true}`

---

## 9. Promotion Workflow

### Promotion Script Family

| Script | Direction | Scope | Use Case |
|--------|-----------|-------|----------|
| `promote.sh` | LOCAL -> PROD | Full workspace | Production deployment |
| `promote-dev.sh` | LOCAL -> DEV | Full workspace | Staging deployment |
| `promote-skill.sh [SKILL_NAME]` | LOCAL -> PROD | Single skill directory | Hot-fix a specific skill |
| `auto-promote.sh` | Git -> DEV | Full workspace (on merge) | CI/CD auto-deploy to staging |
| `rollback.sh [COMMIT_SHA]` | Git history -> PROD | Full workspace | Emergency rollback |

### `promote.sh` Behavior

1. Refuse if LOCAL workspace has uncommitted changes
2. rsync LOCAL workspace to PROD, excluding `.git/`, `.claude/`, `tests/`, `scripts/`
3. Swap environment-specific IDs (e.g., DEV config sheet ID -> PROD config sheet ID)
4. Show diff of changes, require operator confirmation
5. No gateway restart needed — workspace files hot-reload via file watcher

### `auto-promote.sh` (CI/CD Pattern)

A system cron (every minute) on the Droplet runs:

1. `git fetch && git merge --ff-only origin/master`
2. If new commits: run tests
3. If tests pass: `promote-dev.sh` (or `promote.sh` for PROD auto-deploy)
4. Notify operator via Telegram with commit summary

### `rollback.sh` Behavior

1. `git checkout [COMMIT_SHA] -- workspace/` to restore workspace files
2. Run `auto-promote.sh` to deploy the rolled-back state
3. Notify operator via Telegram

### PROD-Owned Files

These files are never overwritten by promotion — they are owned by the PROD agent:

- `MEMORY.md` and `memory/` directory (runtime state)
- `SYSTEM_LOG.md` (audit trail)
- `cron/jobs.json` (registered cron snapshot)

---

## 10. Anti-Patterns

Production-discovered mistakes that cause silent failures or data leaks:

| Anti-Pattern | Why It Fails | Impact |
|---|---|---|
| `sessions_send` for customer DMs | A2A echo loop bug (#7804) — main auto-replies to itself | Infinite message loop between sessions |
| `openclaw message send` CLI for DMs | Creates `[channel]:direct` sessions on main | Contaminates main session's routing table; internal messages leak to customers |
| HTTP calls described in prose | Smaller models interpret prose as description, not action | Silent no-op — skill appears to succeed but sends nothing |
| `agentTurn` for shell-script-only jobs | Spins up full LLM agent session unnecessarily | Wasted tokens and latency |
| DEV cron jobs left running | Token consumption with no production value | Budget burn |
| Sandbox allowlist change without `recreate --all` | Old containers retain stale allowlist | Silent tool failures in sandbox |
| Cron without `delivery.mode: "none"` | Output auto-delivers to main's last active channel | Cron output leaks to customer DMs |
| Hardcoded channel targets in workspace | Workspace tied to single environment | Breaks LOCAL/DEV/PROD portability |
| Operator notes in `delivery.mode: "announce"` output | Internal context included in agent's final output | Operator notes leak to customer-facing channel |
| Cron scripts without explicit PATH export | `openclaw` binary (npm-installed) not on minimal shell PATH | Script silently fails or uses wrong binary |
| Editing a skill without updating its cron prompt | Cron payloads are static — captured at registration time | Cron runs with stale instructions, ignoring skill changes |
| Sharing `exec-approvals.json` across envs | `lastResolvedPath` caches point to wrong workspace | DEV exec calls resolve to PROD paths |
| Hand-editing `openclaw.json` | Breaks internal config metadata tracking | Gateway restart loop with "missing-meta-vs-last-good" error |
| Testing skills with `agent -m` only | Runs in main session, not sandbox | Misses exec-approvals, workspace ro, Docker, tool restrictions |
| `$(command)` in heredocs sent via SSH | Evaluates on local machine, not droplet | Empty .env values, gateway crash-loops |
| Tool calls in BOOT.md depending on external services | Boot-time tool failures poison the session — model stops calling tools for all subsequent requests | Agent enters "degraded mode" for entire session lifetime |
| Step-by-step dispatch templates in AGENTS.md | LLM pattern-matches the template structure and generates expected output text instead of executing tools | Agent produces formatted text describing what it would do instead of calling tools |
| DEV config diverging from PROD structure | Partial DEV config tests a different system — bugs pass DEV, fail PROD | `diff <(openclaw --profile prod config get .) <(openclaw --profile dev config get .)` to catch drift |
| `sessions_send` without verifying main session exists | Target session key doesn't exist in `sessions.json` | Silent failure — cron completes but delegation never happens |
| Mini-class models for cron jobs that read sandbox files | `gpt-4.1-mini` cannot use the file read tool in sandbox | Cron silently skips file-dependent steps; haiku-class works |
| Overwriting (not appending) memory files from isolated cron | Isolated cron uses `write` instead of `edit` (append) | Previous entries silently lost; only latest cron run survives |
| `rsync --delete` without excluding runtime data | `--delete` removes `memory/`, `SYSTEM_LOG.md` written at runtime | Destroys agent state, memory, and audit trail on next sync |
| Env vars in SKILL.md `exec` commands (`$VAR`) | `exec` uses `execFile` — no shell expansion; `$VAR` passed as literal string | API calls get literal `$VAR` as hostname/token; silent 4xx or connection refused |
| Wrapper scripts in exec-approvals with glob patterns | Glob patterns (`**/scripts/*`) break the allowlist parser | ALL exec commands fail, not just the wrapper — total exec lockout |
| LLM-substituted variables in `exec` commands | LLMs unreliably substitute TOOLS.md variables; sometimes pass literal `$VAR` to exec | Intermittent PROD failures — works in testing, fails under load or with different models |
| SELECT-then-UPDATE for claim-before-act | Race condition when auto-trigger and fallback cron overlap | Duplicate DMs/actions sent to customers; use atomic `UPDATE...RETURNING` instead |

---

## 11. CLI-to-API Migration Pattern

As an agent deployment matures, direct CLI access to external services (e.g., `gog` for Google Sheets, `gh` for GitHub) can be replaced with server-side internal API endpoints. This is not required but offers significant benefits.

### Pattern

1. Build a lightweight internal API server (e.g., Next.js, Express) on `localhost:[PORT]`
2. Migrate SKILL.md steps from CLI commands to `exec curl -sf` calls against internal endpoints
3. Server handles authentication, caching, rate limiting, and data access
4. Skills POST to `http://localhost:[PORT]/api/internal/[ENDPOINT]`

### Benefits

- **Eliminates OAuth token exposure in sandbox.** CLI tools require credentials in the environment; the API server holds credentials server-side.
- **Centralizes data access control.** One place to enforce read/write permissions, audit logging, and rate limits.
- **Enables local caching.** The server can cache frequently-read data (config, customer lists) in SQLite or memory, reducing external API calls.
- **Decouples agent from external API changes.** If the external service changes its API, only the server adapter needs updating — not every skill.

### Migration Strategy

Migrate incrementally: start with the most frequently called CLI command, add the internal endpoint, update skills one at a time. Keep the CLI tool in the exec allowlist as a fallback during migration.

### Auto-Migrate on Startup

The internal API server should run database migrations automatically on every start. Using an ORM migration runner (e.g., Drizzle `migrate()`, Prisma `$migrate.deploy()`, Knex `migrate.latest()`) at module initialization ensures:

- No separate migration CLI step in deployment
- Schema changes deploy with the code that uses them
- Promotion scripts only need to restart the server, not run migrations separately
- Safe for SQLite (single-writer, no concurrent migration risk)

**Pattern:** Call `migrate()` at the top level of the database module, before any route handlers initialize:

```typescript
// server/db/index.ts
import { drizzle } from "drizzle-orm/better-sqlite3";
import { migrate } from "drizzle-orm/better-sqlite3/migrator";

const db = drizzle(sqlite);
migrate(db, { migrationsFolder: path.join(__dirname, "migrations") });

export { db };
```

Every import of the database module triggers migration. New migration files are picked up automatically on the next server restart — no manual steps, no deployment scripts.

---

## 12. Configuration Patterns

### `openclaw.json` Key Sections

| Section | Purpose |
|---------|---------|
| `agents.defaults.model` | Primary model and fallbacks |
| `agents.defaults.heartbeat` | Heartbeat model, interval, target |
| `agents.defaults.models` | All available models with aliases and params |
| `channels.[channel]` | Per-channel config (group JID, DM policy, allowlist) |
| `gateway` | Bind address, port, auth mode |
| `agents.defaults.sandbox` | Sandbox config (mode, workspace access, session tools visibility) |
| `tools.sandbox.tools.allow` | Whitelist of tools available in sandbox sessions |
| `tools.subagents.tools.alsoAllow` | Override built-in subagent tool deny list |
| `agents.defaults.sandbox.sessionToolsVisibility` | Session visibility for sandboxed sessions (`"spawned"` or `"all"`) |
| `skills.load.extraDirs` | Additional skill directories to load |
| `cron` | Scheduled job definitions |

### Config Priority

Environment variables > `openclaw.json` > built-in defaults.

### Workspace Path Defaults

OpenClaw defaults to `~/.openclaw/workspace/` (shared) unless `agents.defaults.workspace` is explicitly set. Custom profiles (`--profile name`) create config at `~/.openclaw-$NAME/` but the workspace still defaults to the shared path. Always set `agents.defaults.workspace` explicitly in `openclaw.json` for each profile.

### Strict Schema Validation

OpenClaw validates `openclaw.json` against a strict schema. Invalid keys or types cause startup failures. Key gotchas:
- `models` is a **record** `{model: {options}}`, not an array
- `agents.list` items need `id`, not `name`
- Cron jobs are registered at runtime via `openclaw cron add`, not in the config file

Always use `openclaw config set` to modify config — never hand-edit `openclaw.json`. OpenClaw tracks config metadata internally; hand-editing breaks it, causing "Config observe anomaly: missing-meta-vs-last-good" errors and gateway restart loops.

### Cron CLI Syntax

- Set delivery mode: `openclaw cron edit <id> --delivery-mode none` (not `--params`)
- `openclaw cron list --json` returns `{"jobs": [...]}` (a dict with a `jobs` key), not a bare array. Parse accordingly: `jq '.jobs[]'`

### Cron Payload Cache Sync

Cron payloads are **static** — captured at registration time. Editing a SKILL.md does NOT update the corresponding cron job's payload. When editing a skill that has a cron trigger:

1. Edit the skill file
2. Check if the cron payload conflicts with the updated skill
3. If so: `openclaw cron edit <id> --message "<updated text>"` (agentTurn) or `--system-event "<updated text>"` (systemEvent)
4. Prefer minimal trigger prompts ("Run the [SKILL_NAME] skill") over inline instructions to minimize drift

---

## 13. Operational Learnings

Hard-won lessons from production debugging. Each entry caused a real incident or wasted significant debugging time.

### Sandbox & Exec Isolation

#### Exec-Approvals Paths Inside Docker

`exec-approvals.json` uses absolute paths (e.g., `/usr/bin/curl`). Inside the Docker sandbox, binaries may be at different paths than on the host. The allowlist silently denies if the path doesn't match inside the container.

**Fix:** Verify paths inside the sandbox: `docker run --rm openclaw-sandbox:bookworm-slim which curl`. If host and container paths differ, use the container path in `exec-approvals.json`. Also verify the exec-approvals socket path is accessible from inside the container.

#### `agent -m` Does NOT Test Sandbox

`openclaw agent -m "Run skill"` runs in the **main session** (on host, full access). It does NOT test sandbox behavior. It bypasses:
1. `exec-approvals` allowlist path resolution inside Docker
2. Workspace read-only enforcement
3. Tool restrictions (`sessions_spawn` denied in sandbox)
4. Docker image availability

To test sandbox behavior: trigger a cron job (`openclaw cron run <id>`) and check its session log, or use `openclaw sandbox explain` to verify config. `agent -m` is a reasoning test, not an integration test.

**Testing flow:** LOCAL `agent -m` (reasoning) → DEV `cron run` (sandbox integration) → PROD cron schedule (live).

#### `exec curl` — LLMs Describe Instead of Emitting

Sandbox sessions cannot make HTTP calls via high-level constructs (e.g., "POST to localhost:3000"). The LLM must emit `exec curl` explicitly. Smaller models (Codex, GPT-5.1) tend to describe the HTTP call in prose rather than writing the actual `exec curl` command.

**Fix:** All SKILL.md files should include the literal `exec curl` command in the workflow steps. The skill body serves as the authoritative instruction — the agent copies the command verbatim rather than improvising.

**Pattern:**
```markdown
## Workflow
1. Run: exec curl -sf -X POST http://localhost:[PORT]/api/internal/[ENDPOINT] -H "Content-Type: application/json"
```

**Why `-sf`:** `-s` (silent) suppresses progress bars; `-f` (fail) returns non-zero on HTTP errors so the agent sees the failure. Always use both.

#### Sandbox Env Var Forwarding

Sandbox containers do NOT inherit the host process environment. Environment variables must be explicitly forwarded via `agents.defaults.sandbox.docker.env` in `openclaw.json`. Use SecretRef syntax (`${VAR_NAME}`) to reference gateway env vars.

**Impact:** Without this config, `exec curl` with auth headers and any database-dependent logic will fail silently or return auth errors from sandbox/cron sessions.

**Fix:** Add all required env vars to `sandbox.docker.env`:
```json
"agents": {
  "defaults": {
    "sandbox": {
      "docker": {
        "env": {
          "DATABASE_URL": "${DATABASE_URL}",
          "APP_URL": "${APP_URL}"
        }
      }
    }
  }
}
```

#### Docker Sensitive Var Filter

Docker sandbox blocks environment variables with `KEY`, `SECRET`, `TOKEN`, or `PASSWORD` in their names — even when explicitly listed in `sandbox.docker.env`.

**Workaround:** Use the internal API endpoint pattern (Section 11). The sandbox calls REST APIs via `exec curl`, and the API server handles authenticated operations server-side with its own env vars. The sandbox only needs non-sensitive env vars (URLs, user IDs) and a single write credential (e.g., `AGENT_WRITE_CRED` — "CRED" does not trigger the filter).

**Rule:** Don't rename env vars to bypass the filter (e.g., `API_KEY` → `API_CRED1`). This makes config unreadable. Build API endpoints instead.

#### Shell Expansion in Sandbox

Sandbox `exec` does NOT expand shell variables. `exec echo $VAR` returns the literal string `$VAR`, not the value.

**Fix:** Use `exec printenv VAR_NAME` to read environment variables in sandbox. `printenv` reads directly from the process environment without shell expansion.

**Do NOT use:**
- `echo $VAR` (returns literal `$VAR`)
- `bash -c "echo $VAR"` (not in exec allowlist)
- `${VAR}` in curl args (causes "Bad hostname")

#### `strictInlineEval` Blocks Inline Code Execution

OpenClaw has `tools.exec.strictInlineEval` (default: `true`) which blocks interpreter eval forms (`node -e`, `python -c`, `bash -c`) even when the binary is allowlisted in exec-approvals. This is a security feature to prevent arbitrary code execution.

**Workaround:** Use the API endpoint pattern — the agent calls REST APIs via `exec curl`, and the server handles complex operations. For simple reads, use `exec printenv` for env vars or `exec cat` for files. If `node -e` is essential, set `strictInlineEval: false` (weakens security) or write to a temp script file and execute it.

#### Sandbox Workspace Snapshot Scope

Sandbox sessions only receive a snapshot of core identity files (`SOUL.md`, `AGENTS.md`, `TOOLS.md`, `USER.md`, `IDENTITY.md`, `HEARTBEAT.md`) and the `skills/` directory. Custom root-level workspace files are NOT available in sandbox sessions.

**Impact:** Files like custom config dumps, blacklists, or data files placed at the workspace root will not be accessible from cron/sandbox sessions.

**Fix:** Use the OpenClaw memory system (`memory_search`/`memory_get`) for cross-run state persistence from sandbox. Place data that sandbox sessions need inside `skills/` subdirectories or the `memory/` directory (indexed by the memory system).

#### Sandbox Docker Image Must Be Explicitly Built

`sandbox-setup.sh` is not run automatically during OpenClaw installation. Without the sandbox Docker image, ALL isolated and cron sessions fail silently — they cannot start a container, so the session produces no output and no error.

**Fix:** Verify the image exists after installation and after any Docker cleanup: `docker images | grep openclaw-sandbox`. If missing, rebuild: `sg docker -c "cd ~/.openclaw && bash scripts/sandbox-setup.sh"`. Add this check to deployment and upgrade scripts.

#### fnm Binary Path Resolution in `exec-approvals`

When Node.js is managed by `fnm` (Fast Node Manager), binaries live at `~/.local/share/fnm/node-versions/<version>/bin/node` instead of `/usr/bin/node`. The exec-approvals allowlist silently denies if the path doesn't match inside the container or on the host.

**Fix:** Include both standard paths (`/usr/bin/node`) and fnm-managed paths in exec-approvals. Better yet, prefer API endpoints via `exec curl` over `exec node` — this avoids path resolution issues entirely.

#### fnm Not in Default SSH PATH

SSH commands that need `npm` or `node` (e.g., deploy scripts, cron sync scripts) fail because `fnm` is not loaded in non-login shell sessions. The `.bashrc` that initializes fnm is not sourced for non-interactive SSH commands.

**Fix:** Wrap SSH commands with `bash -lc "<command>"` or prefix with `eval "$(~/.local/share/fnm/fnm env)"` to load fnm into the shell environment.

#### `autoAllowSkills` for Skill Bin Execution

Without `autoAllowSkills: true` in `exec-approvals.json`, binaries listed in a skill's frontmatter `requires.bins` trigger socket-based approval. Since there's no approval listener in cron/sandbox sessions, the request times out at 120s and denies.

**Fix:** Set `autoAllowSkills: true` in `exec-approvals.json` if skills declare required binaries. This auto-approves bins declared in skill frontmatter without socket confirmation.

**Tradeoff:** `false` is more restrictive (manual allowlist only); `true` trusts skill frontmatter declarations.

#### Exec-Approvals: Bare Names AND Absolute Paths

`exec` passes args to `execFile` without shell `PATH` resolution. The agent emits `curl ...` (bare name), but `exec-approvals.json` may list `/usr/bin/curl` (absolute path) — causing a silent allowlist miss despite the binary being "allowed". Agents then hallucinate success (see "LLM Hallucinated API Responses").

**Fix:** Allowlist BOTH forms for every binary the agent uses:
```json
{
  "bins": {
    "curl":          { "allow": true },
    "/usr/bin/curl": { "allow": true },
    "ls":            { "allow": true },
    "cat":           { "allow": true },
    "env":           { "allow": true },
    "grep":          { "allow": true },
    "printenv":      { "allow": true }
  }
}
```

Bare-name entries for `ls`/`cat`/`env`/`grep` enable agent self-diagnosis (listing sandbox config, reading env) without escalation. Docker sandbox isolation contains the residual risk.

**Verification:** Trigger from a sandbox session (cron run), not from `agent -m` — `-m` runs on host with full access and masks the allowlist gap.

### Session & Model Behavior

#### LLM Hallucinated API Responses

When `exec` is denied (allowlist miss), the agent can fabricate a plausible API response — including realistic HTTP status codes, UUIDs, and success messages. The agent reports "201 Created" with a UUID that looks real but doesn't exist in the data store.

**Fix:** Add explicit verification steps in SKILL.md: after any write operation, independently confirm the write succeeded by reading back from the data store. Do not trust the agent's reported HTTP status alone.

#### `memory_search` FTS Limitations

OpenClaw's `memory_search` full-text search misses on compound/multi-word queries. `memory_search("PENDING_IDEAS SignalMidway")` returns empty, but `memory_search("PENDING_IDEAS")` returns the chunk containing "SignalMidway".

**Fix:** Always use broad single-anchor queries and filter results client-side. Never append entity names or secondary terms to the search query.

#### `sessions_send` Partial Processing

Multi-section payloads sent via `sessions_send` may be partially processed. When a single `sessions_send` call includes instructions to write multiple data sections, the receiving session may act on one section and skip others.

**Fix:** Split `sessions_send` into separate calls — one per write intent. Each call should be one atomic operation. If combining is unavoidable, put the most important section first.

#### `thinking:high` for Tool Calls on Messaging Channels

Without extended thinking enabled (`params.thinking: "high"` or equivalent in model config), messaging channel turns (Telegram, WhatsApp) may produce text-only responses with no tool calls. The model generates text tokens first, and the single-inference turn ends before tools execute.

**Fix:** Set `params.thinking: "high"` on the model config for agents that must call tools from messaging channel turns. This forces the model to reason before generating, enabling multi-step tool chains in a single turn.

**Tradeoff:** Thinking adds reasoning overhead. Cron jobs with `thinking:high` may need increased timeouts (e.g., 900s instead of 300s).

#### Conversation History Pollution

After many failed tool attempts accumulate in a session (28+ in one observed case), the model learns a "generate text, don't call tools" pattern and stops calling tools entirely — even with explicit instructions.

**Fix:** Delete the polluted session and restart the gateway to create a fresh session. In one observed case, the threshold was ~28 failed attempts before the model stopped calling tools entirely. Prevention: fix tool failures early (don't let them accumulate), monitor tool failure counts, and configure session maintenance to auto-clear stale sessions.

#### Mini Models Cannot Read Sandbox Files

`gpt-4.1-mini` and similar mini-class models cannot use the file read tool in sandbox sessions. The tool call is available but the model fails to invoke it correctly. Haiku-class models (e.g., `claude-3.5-haiku`) can read sandbox files reliably.

**Fix:** Upgrade cron jobs that need to read config or data files in sandbox from mini-class to haiku-class. Reserve mini for template-based sends and simple formatting tasks that don't require file reads.

#### `thinking:high` Timeout Overhead

Extended thinking (`params.thinking: "high"`) adds 2-3x latency to cron job execution (e.g., a 480s job becomes 900s). Do not override with `--thinking off` on individual cron jobs — this fundamentally changes model behavior and may break tool-calling chains.

**Fix:** Increase cron job timeouts to accommodate thinking overhead rather than disabling thinking. Use `openclaw cron edit <id> --timeout 900` for jobs that run long with thinking enabled.

#### `sessions_send` Requires Main Session to Exist

Clearing `sessions.json` (e.g., during cleanup or redeployment) before running cron jobs that use `sessions_send` fails silently. The target session key (`agent:main:main`) must exist in the session store before any isolated cron can delegate to it.

**Fix:** After clearing sessions or deploying, restart the gateway and send a message to the main session (via Telegram or CLI) to ensure it is initialized before cron jobs fire. Add a session existence check to deployment scripts.

#### Agent Hallucinated "Done ✅" with No Tool Calls

Long-lived operator DM sessions can drift into a text-only rut where the agent replies "Done ✅" without actually invoking any tool. The downstream effect (DB write, DM send, log append) never happened — but the operator sees a convincing confirmation. Related to conversation-history pollution: the model has learned a "generate text, don't call tools" pattern from prior failed attempts in the same session.

**Detection:** For any operator message that requires a side effect, grep the active session JSONL (path from `sessions.json`) for a `toolCall` block between the operator message and the agent's reply. Absence of a tool call is the signature.

```bash
JSONL=$(jq -r '."agent:main:main".transcript' ~/.openclaw/sessions.json)
tail -200 "$JSONL" | jq -r 'select(.role=="assistant") | .content[]? | .type' \
  | grep -q tool_use || echo "HALLUCINATION: no tool_use in recent assistant turns"
```

**Recovery:** Rename the polluted JSONL to `<id>.jsonl.reset.<iso-ts>`, mint a new session UUID in `sessions.json`, set `systemSent: false`, restart the gateway.

**Prevention:** Configure `session.maintenance.mode: "enforce"` with aggressive `sessionRetention`; monitor tool-failure counts; don't let failed attempts accumulate unboundedly in a single session.

#### `sessions_send` From Cron: Single 60s Sync, No Retry

Retry-on-timeout patterns for cron-initiated `sessions_send` expose a sandbox-lifecycle race: the retry fires while the original sandbox is tearing down, and the envelope is silently dropped after a successful `status: "accepted"`.

**Rule:** Issue a single `sessions_send` with `timeoutSeconds=60`, fail-fast on timeout, emit `failure_class=infra` to the observer. Do NOT implement a fire-and-forget retry path.

**HEARTBEAT dedup caveat:** When multiple envelopes share a `run_id` but differ by `kind` (e.g., three distinct self-check kinds in one run), the dedup key MUST include `.kind` — otherwise peer envelopes collapse to `OK_DUPLICATE` and get dropped.

#### Post-Deploy Session Clear + Warmup

When a deploy modifies `HEARTBEAT.md` handlers (or any workspace file the main session caches), the running session continues applying the old logic until it is deleted. This manifests as CRITICAL alerts emitted with the pre-deploy format hours after the new handler landed.

**Pattern for every deploy script:**
```bash
# 1. Delete the main session
openclaw --profile "$PROFILE" sessions delete agent:main:main || true

# 2. Warmup ping to recreate — non-fatal
openclaw --profile "$PROFILE" agent --message "warmup after deploy $(date -u +%FT%TZ)" \
  --timeout 30 || echo "[warn] warmup ping failed, continuing"
```

Both steps must be non-fatal (session may not exist pre-first-deploy; warmup may race gateway restart). The goal is best-effort freshness, not a hard gate.

### Delivery & Messaging

#### Delivery Mode Leaks

The `delivery.mode: "announce"` setting on cron jobs sends the agent's final output to the configured channel. If the agent includes internal notes, debug info, or operator-only context in its output, that content leaks to the delivery channel.

**Rule:** Skills with `delivery.mode: "announce"` must structure their output so only the customer-facing message appears. Internal summaries go to the operator channel via explicit `message` tool calls, not as part of the agent's conversational output. Better yet, use `delivery.mode: "none"` on all crons (see Section 4 Rule 7).

#### `sessions_send` Auto-Delivery Leak

`sessions_send` to `agent:main:main` for customer DM delivery is dangerous. The main session's auto-delivery context routes responses to whatever channel was most recently active. If a customer DM session is active, ALL main session output — including internal reasoning, cron summaries, and debug text — auto-delivers to that customer.

**Root cause:** Main session output routing is non-deterministic. Multiple fix attempts (session key format, stronger SOUL.md instructions) fail because the routing depends on gateway state, not agent reasoning.

**Final fix pattern:** Server-side DM delivery. The internal API server calls `openclaw message send` CLI directly (via process execution) to deliver DMs through the gateway, bypassing agent sessions entirely. Skills return structured action objects (`{type: "[CHANNEL]_dm", phone, message}`) instead of sending messages themselves.

**Rule:** Never use `sessions_send` for customer-facing message delivery. `sessions_send` is still valid for internal delegation (memory writes, data reads, sheet updates).

#### Cron Delivery Channel Syntax

Use `--channel telegram --to <chatId>` when registering cron jobs, not a combined format like `--channel telegram:dm:<chatId>`. The combined format fails to resolve on delivery.

**Also:** The `--to` field provides Telegram session context even when delivery is disabled (`--delivery-mode none`). Use `--no-deliver` as an alternative to `--delivery-mode none`.

#### Telegram Message Length Limits

Telegram enforces a 4096 character limit per message. Skills that produce long output (reports, lists, summaries) must instruct the agent to split into multiple messages.

**Also:** The `message` tool target must be `telegram:<chatId>` (e.g., `telegram:5906288273`), not a symbolic name like "operator". Specify the exact tool call format in SKILL.md to prevent the agent from guessing.

#### Delivery-Mode Audit Whitelist

HEARTBEAT-style cron delivery audits that flag `lastDelivered: false` as a failure will false-alarm on every job intentionally set to `delivery.mode: "none"` (those deliver via the sandbox `message` tool, not the cron-level channel). Noise from these false alarms masks real delivery failures.

**Rule:** Delivery audits must skip jobs where `delivery.mode == "none"`. Only jobs with `delivery.mode` in (`announce`, `direct`) are eligible for the `lastDelivered` check.

```
6b. For each cron job:
    if delivery.mode == "none": skip (delivers via message tool from sandbox)
    else if lastDelivered == false and lastRunAt > threshold: flag
```

#### Handler Side-Effects BEFORE ACK-Return

In HEARTBEAT handler flows (`SELF_CHECK_LOG`, `EVOLUTION_LOG_APPEND`, and similar runtime envelopes), the agent may return the ACK envelope and terminate the turn before post-ACK steps execute. Any side effect placed AFTER the ACK-return step — Telegram alerts, ledger writes, incident escalation — can be silently skipped.

**Rule:** All side effects must run BEFORE the ACK-return step. In handler markdown, explicitly number steps and place alerts/writes in a step whose number is LOWER than the ACK-return step.

```markdown
## SELF_CHECK_LOG handler
Step 1: parse envelope
Step 2: write to SYSTEM_LOG.jsonl
Step 3: if failure_class in (regressed, source_degraded) AND confidence >= medium:
         send Telegram alert to operator  <-- side effect BEFORE ACK
Step 4: write daily dedup ledger entry    <-- side effect BEFORE ACK
Step 5: return ACK_OK envelope            <-- terminates turn
```

Verify by auditing each handler: no critical write should appear after the return-step.

### Cron & Scheduling

#### PATH Issues in Cron Scripts

Scripts invoked by OpenClaw cron jobs (`systemEvent`) run in a minimal shell environment. The `openclaw` binary (installed via npm) may not be on `$PATH`.

**Fix:** All cron-invoked scripts should:
1. Export the npm global bin directory to PATH: `export PATH="$HOME/.npm-global/bin:$PATH"`
2. Use an `$OPENCLAW` env var (defaulting to `openclaw`) for CLI invocation
3. Derive workspace paths dynamically from `$(dirname "$0")/..` rather than hardcoding

#### Session Maintenance for Container Buildup

Without session maintenance, sandbox containers from cron and subagent runs accumulate indefinitely. One deployment accumulated 106 orphaned containers on a DEV gateway.

**Fix:** Configure `session.maintenance.mode: "enforce"` and `cron.sessionRetention: "4h"` in `openclaw.json`:
```json
"session": {
  "maintenance": {
    "mode": "enforce"
  }
},
"cron": {
  "sessionRetention": "4h"
}
```

#### `dmScope` Conflict with Sandbox Non-Main

`session.dmScope: "per-channel-peer"` combined with `sandbox.mode: "non-main"` creates sandboxed DM sessions. DM sessions become non-main, get sandboxed in Docker, and lose access to host-level tools (exec, file writes, full env vars).

**Impact:** For single-operator setups where the operator needs full tool access from Telegram/WhatsApp DMs, this combination silently blocks tool execution.

**Fix:** Remove `dmScope` for single-operator bots so DMs route to the unsandboxed main session. Re-enable `per-channel-peer` only when multi-user support is needed.

#### Cron Timezone Drift

The `tz` field on cron jobs can be silently lost when editing jobs via `openclaw cron edit` or when sync scripts modify schedule expressions. A cron registered with `tz: "America/Los_Angeles"` may revert to UTC interpretation after an edit.

**Fix:** After any cron edit, verify timezone preservation: `openclaw cron list --json | jq '.jobs[] | {name, schedule, tz}'`. Include this check in cron sync scripts and weekly maintenance audits. Always re-specify `--tz` when editing cron schedules.

#### Config-Cache Transient Failures

Config-cache refresh operations can fail with `node ENOENT` errors referencing a temporary file. The failure is transient — retry always succeeds. Observed pattern: 4-5 transient failures in a 6-hour window, each succeeding on second attempt.

**Fix:** Build retry logic (2-3 attempts with 1s backoff) into any script that refreshes the config cache. Do not treat the first failure as fatal.

#### Main-Session `systemEvent` Crons Need `wakeMode: "next-heartbeat"`

Cron jobs with `sessionTarget: "main"` and `wakeMode: "now"` poll `runHeartbeatOnce` for up to 120s. When the main session is busy (operator DM turn in flight), every such poll returns `skipped: requests-in-flight` — serializing ALL infra crons behind the operator interaction. Isolated cron jobs are unaffected; this becomes a strong diagnostic signal.

**Rule:** For every cron targeting main session (config-cache refresh, hourly checkpoint, daily backup, memory init reminders), set `wakeMode: "next-heartbeat"` — it defers to the next scheduled heartbeat tick instead of polling.

```json
{
  "sessionTarget": "main",
  "wakeMode": "next-heartbeat"
}
```

**Heartbeat detection:** Flag a stuck main-session cron lane when `(now - lastRunAtMs) > 1.5x interval` AND (`consecutiveErrors > 0` OR `runningAtMs` stale >10 min). The alert should include an isolated-lane comparison ("isolated healthy; main stuck") for fast triage.

#### Exclude `cron/` from Rsync Between Environments

Each environment owns its own runtime cron snapshot via the hourly checkpoint (gateway → `workspace/cron/jobs.json`). If `promote.sh` rsyncs `cron/` from DEV to PROD, a stale DEV snapshot can silently overwrite PROD schedules — causing DOW drift, schedule regressions, or re-enabling disabled shadow jobs. No alert fires until the heartbeat detects stale config-cache hours later.

**Rule:** Add `--exclude='cron/'` to all cross-env rsync. `sync-cron-from-config.sh` (config-as-authority) reconciles schedules; git is NOT authoritative for runtime cron snapshots.

```bash
rsync -av --delete \
  --exclude='cron/' --exclude='memory/' --exclude='.openclaw/' \
  --exclude='SYSTEM_LOG.md' --exclude='SYSTEM_LOG.jsonl' \
  ~/.openclaw-dev/workspace/ ~/.openclaw/workspace/
```

#### Silent-Skip → Loud-Fail for Reconciliation Scripts

The most pernicious drift mode in auto-fix loops: a reconciliation script (e.g., `sync-cron-from-config.sh`, called hourly) hits a guard condition, logs nothing, and `exit 0`s. The auto-fix loop reports success every cycle while the underlying state rots for days.

**Rule:** Any reconciliation script that skips due to a guard condition must:
1. Write a clear message to stderr
2. Exit with a distinct non-zero code (e.g., `2` for "skipped, not an error but not a fix")
3. Leave an audit trail (append to a log file)

```bash
if (( now_ms - cached_at_ms > STALE_THRESHOLD_MS )); then
  echo "[sync-cron] skipping: cache age $(( (now_ms - cached_at_ms)/60000 ))min > threshold" >&2
  echo "$(date -u +%FT%TZ) skip cache-stale" >> ~/.openclaw/logs/sync-cron.log
  exit 2
fi
```

The caller (hourly checkpoint, CI, operator) treats exit 2 as "needs attention" rather than invisible success.

#### Config Cache Freshness: `cachedAt` vs `lastUpdated`

Config-cache JSON blobs typically carry two timestamps:

| Field | Semantics | Use for |
|-------|-----------|---------|
| `cachedAt` | When the cache was last refreshed from the source | Freshness guards in sync scripts |
| `lastUpdated` | When the underlying config was last modified | Change detection / invalidation |

A sync script that compares `Date.now()` against `lastUpdated` to decide "is the cache fresh?" will permanently block when the config is stable but the cache is live-refreshed — because `lastUpdated` does not advance, even though the cache is current.

**Rule:** Freshness checks read `cachedAt`. For transition safety across deployments that predate `cachedAt`, fall back: `effective_ms = cachedAt ?? lastUpdated`.

#### Cross-Profile Cron Health Validator

Multi-profile deployments (DEV + PROD, or multiple agents) need a single ad-hoc health check that catches silent degradation between deploys. Individual cron metadata can report `lastStatus: "success"` while 401 auth errors accumulate invisibly in session logs.

**Pattern — `validate-crons.sh`:**
1. For each profile, SSH to the droplet and run `openclaw --profile $P cron list --json`
2. Classify each job's state:
   - **STALE**: `lastRunAt` older than 2× cadence
   - **AUTH**: recent session log contains `401` / `403` / `unauthorized`
   - **INFRA**: consecutive timeouts or `ENOTFOUND`
   - **DURATION_CREEP**: avg runtime >1.5× 7-day baseline
3. Summarize per profile (counts by class)
4. Exit 1 on any **RED** state; non-fatal (exit 0) on **WARN**; normal exit on all-green

Run manually post-deploy and optionally from a `cron`-style monitor on a separate host. The key property: *one script* validates *all profiles*, so drift between profiles is visible at a glance.

### Memory

#### Memory Init: Append Not Overwrite

Isolated cron sessions writing to `memory/SYSTEM_LOG.md` or daily memory files can overwrite instead of appending. The model defaults to using the `write` tool (full file replacement) rather than `edit` (targeted append) when not explicitly instructed.

**Fix:** Include explicit "APPEND to the file — do not overwrite existing content" instructions in cron payloads that write to memory files. Use `edit` tool with append semantics, not `write`. Verify by checking file size after cron runs — a shrinking file indicates overwrite.

#### Memory Files Indexed, Workspace Root Files Not

Only files in the `memory/` directory are indexed for `memory_search`. Root-level workspace files (e.g., custom config dumps, data files, signal files) do not appear in search results even though they are readable from the workspace.

**Fix:** For persistent searchable state, write to the `memory/` subdirectory. For data that must remain at workspace root, reference it by explicit file path in skills rather than relying on `memory_search` to surface it.

#### SYSTEM_LOG: Split Telemetry from Ops Text

Deployments with both machine-readable self-check envelopes AND text-heavy ops writes (BOOT, HEALTH, BACKUP, INCIDENT) end up parsing SYSTEM_LOG.md as JSON and tripping over the text. Typical symptom: an analyzer reports a high "malformed" rate because legitimate text log lines are being (incorrectly) parsed as JSON.

The collision gets worse as writers scale: text writers out-produce the reader's age-out logic, and pseudo-JSON envelopes emitted by different skills drift in schema.

**Split pattern:**

| File | Format | Writers | Readers |
|------|--------|---------|---------|
| `memory/SYSTEM_LOG.jsonl` | Newline-delimited JSON | Self-check envelope handler ONLY | Harness/observer, analyzers |
| `memory/SYSTEM_LOG.md` | Free-form text | Cron skills, backup/incident/boot writers | Operator |

**Migration safety:** On first run after split, the reader may find the `.jsonl` file missing. Emit `source_missing: true` in the wrapper output rather than failing the check — the file will be created by the next self-check write.

**Rsync:** Exclude both files (`--exclude='SYSTEM_LOG.md' --exclude='SYSTEM_LOG.jsonl'`) to prevent env-to-env contamination.

### Deployment & Upgrades

#### Port Fixup for Multi-Environment Skills

When skills hardcode `localhost:[PORT]` for internal API calls, and DEV/PROD servers run on different ports (e.g., PROD on `:3000`, DEV on `:3001`), promotion scripts need port fixup.

**Pattern:** `promote-dev.sh` runs sed to replace `:3000` → `:3001` in skill files. `promote.sh` reverses the replacement for PROD.

**Gotcha:** The fixup sed must exclude the promotion scripts themselves — otherwise it rewrites the sed patterns inside `promote*.sh`, turning `s|3001|3000|` into `s|3001|3001|` (a no-op). Use `find -not -name 'promote*.sh'` in the fixup command.

#### Telegram IPv6 / autoSelectFamily Issue

OpenClaw 2026.3.24+ with Node.js 24 defaults `autoSelectFamily=true`. Droplets without public IPv6 (only link-local `fe80::`) cause the Telegram plugin to try IPv6 first, timing out with `ETIMEDOUT`/`ENETUNREACH`.

**Fix:** `openclaw config set channels.telegram.network.autoSelectFamily false`

Also available as env var: `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`. The `--dns-result-order=ipv4first` NODE_OPTIONS flag does NOT fix this — `autoSelectFamily` operates at the socket level, not DNS.

**Check:** `ip -6 addr show scope global` — if empty, set `autoSelectFamily=false`.

#### Gateway `.env` Baking

`openclaw gateway install` bakes `.env` values into the systemd service unit file at install time. Changing `.env` alone has no effect on the running gateway.

**After `.env` changes:**
1. Export the new vars in your shell
2. Run `openclaw gateway install --force`
3. `systemctl daemon-reload`
4. Restart the gateway

Without steps 2-3, the gateway continues using the old baked values. This also means API credential rotations in `.env` are silently ignored until the service is reinstalled.

#### Stale Sessions After Deploy

Workspace file changes (SOUL.md, AGENTS.md, TOOLS.md, skills) are not picked up by existing sessions. The gateway reuses sessions from the session store — it does not recreate them on restart.

**Fix:** After deploying updated workspace files:
1. Deploy the files
2. Restart the gateway
3. If major instruction changes: delete the relevant session entry from `sessions.json` and restart again

Without step 3, the agent continues using cached instructions from the previous session.

#### Upgrade Process (Post-Update Ceremony)

After `openclaw update`, the systemd service files still reference the old version. The gateway continues running the old binary until the service is reinstalled.

**Full upgrade sequence:**
1. `openclaw update`
2. `openclaw --profile $NAME gateway install --force` (for each profile)
3. `systemctl --user daemon-reload`
4. Restart each gateway service
5. Wait 15-20s for warmup (v2026.4.1+ needs this before responding to CLI queries)
6. Verify: `openclaw --profile $NAME approvals get --gateway --timeout 30000`

**Also:** Re-verify exec-approvals after every upgrade — profile isolation bugs may persist across versions.

#### `systemEvent` Relay Regression (v2026.4.1+)

OpenClaw v2026.4.1 introduced "Runtime Event Trust" security hardening (#62003) that reclassifies cron `systemEvent` payloads as untrusted. The gateway wraps them in a relay frame ("Please relay this reminder to the user in a helpful and friendly way"), causing the model to relay instead of execute.

**Impact:** Cron jobs using `systemEvent` for script execution stop working. SOUL.md directives and session resets do NOT override the relay wrapper — the delivery framing wins.

**Fix:** Use system crontab (`crontab -e`) for deterministic shell operations. Reserve OpenClaw cron for agent-reasoning tasks (`agentTurn` with isolated sessions). For agent-aware tasks from system crontab, use `openclaw agent --deliver`.

#### Rsync `--delete` Must Exclude Runtime Data

When using `rsync --delete` to sync workspace files from source to target (e.g., promotion scripts), the `--delete` flag removes files that exist on the target but not the source. Agent-written runtime data — `memory/`, `.openclaw/`, `SYSTEM_LOG.md` — will be destroyed.

**Fix:** Add explicit excludes to all sync commands: `rsync --delete --exclude='memory/' --exclude='.openclaw/' --exclude='SYSTEM_LOG.md' --exclude='cron/'`. Verify exclude list in all promotion and sync scripts. The `PROD-Owned Files` list (Section 9) defines which files must be excluded.

#### Hardcoded URLs in Skills: Three Failed Approaches

When DEV and PROD servers run on the same Droplet at different ports (e.g., `:3000` vs `:3001`), skills must target the correct port per environment. Three approaches to parameterize URLs all failed:

| Approach | Why It Failed |
|----------|---------------|
| Env var in SKILL.md (`$API_URL`) | `exec` uses `execFile` — no shell expansion. `$VAR` passed as literal string. |
| Wrapper script (`scripts/call-api.sh`) | `exec-approvals` glob patterns (`**/scripts/*`) broke the allowlist parser — ALL exec commands failed. |
| TOOLS.md prompt variable (`$SERVER_URL`) | LLM substitution unreliable. Both GPT-5.1 and GPT-5.3-codex sometimes pass the literal variable to exec instead of substituting. Caused a PROD incident within hours — immediately reverted. |

**Current reliable solution:** Hardcode `localhost:[PORT]` in SKILL.md files with the PROD port as the source-of-truth. Promotion scripts (`promote-dev.sh`) run sed to swap ports at deploy time. Defense-in-depth: internal API responses can include an `"environment"` field to catch mismatches during debugging.

**Verification rule:** Before deploying to any environment, verify no wrong-environment ports exist:
- DEV: `grep -r "localhost:[PROD_PORT]" ~/.openclaw-dev/workspace/skills/` — must return nothing
- PROD: `grep -r "localhost:[DEV_PORT]" ~/.openclaw/workspace/skills/` — must return nothing

#### Deploy Rsync Silent SKILL.md Skip

`rsync` in deploy scripts can report a successful transfer while SKILL.md files on the target retain stale content and old mtimes. Root cause is unclear (possibly rsync size/time comparison logic or mtime preservation), but this has caused multiple incidents where deployed skills ran with outdated instructions.

**Fix:** After every deploy that edits SKILL.md files, verify the change landed:
```bash
ssh $HOST 'grep -c "<known-new-marker>" ~/.openclaw-$PROFILE/workspace/skills/<skill>/SKILL.md'
```
If count is 0, directly scp the stale files and restart the gateway. Add this post-deploy verification to all deployment scripts.

#### `sessions_send` ACK Verification in Session Logs

Grepping session JSONL files for ACK outcomes (e.g., `OK_APPENDED`, `ERR_PARSE`) produces false positives because preloaded SKILL.md template text contains the same string literals. Template text appears in assistant `toolCall` message content, while real ACKs appear in separate `toolResult` blocks.

**Fix:** Pair `sessions_send` `toolCall` blocks with their corresponding `toolResult` by `toolCallId` to distinguish real runtime ACKs from template text. Fire-and-forget sends use `status: "accepted"` (no ACK body). Timeout sends appear in `toolCall` with `timeoutSeconds` but have no matching `toolResult`.

#### `deploy.sh --ref <sha|branch>` for DEV Feature Testing

Promotion flows often assume "DEV = tip of main", which prevents testing a feature branch before merge. Support an explicit `--ref` parameter on `deploy.sh` so DEV can validate arbitrary git refs without forcing an early merge. PROD must reject non-main refs.

**Shape:**
```bash
deploy.sh --env dev --ref feature/my-branch   # OK: git checkout --detach
deploy.sh --env prod --ref feature/my-branch  # ERROR: PROD refuses non-main
deploy.sh --env prod                          # OK: uses current main HEAD
```

**Caveat:** Do NOT `git clean -fd` after the ref switch — this wipes untracked but live runtime state (`memory/` daily logs, `cron/jobs.json` snapshots). Only clean what you intentionally gitignored.

#### Custom MCP Service Deployment

When shipping a custom MCP service alongside the gateway (e.g., a per-agent build executor), a naive deploy can "false-green": the service process comes up, but the token baked into the gateway's `.env` doesn't match the token configured in the MCP service, and every agent call returns 401.

**Pattern:** fingerprint the token at three locations, then fail-fast if any pair disagrees.

| Location | How to read |
|----------|-------------|
| Deploy-time secret | `sha256sum < .secrets/mcp-token` |
| Gateway systemd env | `systemctl --user show openclaw-gateway-$PROFILE -p Environment` |
| MCP service | `curl http://localhost:$MCP_PORT/healthz` → returns `{tokenFingerprint: "..."}` |

If any fingerprint differs, abort the deploy.

**Packaging:**
- Bundle the service with `esbuild` (CJS output), externalizing `node_modules/@yourorg/*` so template packages stay hot-reloadable
- Bake `version` and `GIT_COMMIT` at bundle time
- Install as a user-level systemd unit; don't cold-start on failure — do a preflight boot check first and fail the deploy if startup takes >N seconds
- Validate post-deploy by sending an agent message that triggers an `exec curl` MCP ping, and assert the response embeds the fingerprint

#### Multi-Profile Config Parity

When running DEV and PROD (or multiple agent profiles) on the same host, config drift between profiles is a dominant source of "works in DEV, fails in PROD" incidents. Teams instinctively maintain DEV and duplicate the values into a bootstrap/deploy script — then the two copies drift.

**Rules:**
1. Single source of truth for every config value. Store the canonical value in one well-known file (e.g., `deploy/config.yml` or `plan.md`), read it in both the bootstrap AND deploy scripts. Never copy values between scripts.
2. Validate both profiles end-to-end before marking a phase complete. DEV is a production test environment for PROD; if DEV is skipped, PROD will discover the regression.
3. Diff full config between profiles on every deploy:
   ```bash
   diff <(openclaw --profile dev config get .) <(openclaw --profile prod config get .)
   ```
   Expected diffs are narrow (ports, hostnames, channel IDs). Unexpected diffs must be investigated before the deploy proceeds.

#### Post-Deploy Marker Verification (Mandatory)

This extends "Deploy Rsync Silent SKILL.md Skip" above: the belt-and-suspenders rule is that EVERY deploy that edits workspace files (skills, HEARTBEAT, identity files) must post-deploy grep for a known marker. No exceptions — rsync's success signal is unreliable.

**Pattern:** bake a short, unique marker (e.g., a new section name or a version comment) into every deploy, and verify:
```bash
MARKER="# deploy-$(git rev-parse --short HEAD)"
grep -q "$MARKER" ~/.openclaw/workspace/skills/**/SKILL.md || {
  echo "[deploy] FAIL: marker not found; rsync silently skipped" >&2
  exit 1
}
```

Make this gate fatal, not advisory. One silent skip is enough to erode confidence in the deploy pipeline.

#### SSH ControlMaster for Fleet-Ops Scripts

`ufw` on Ubuntu defaults to rate-limiting SSH at 6 connections / 30s. Scripts that fan out many small SSH commands (validators, fingerprinters, multi-profile deploys) trip this limit and start failing opaquely with `Connection refused` partway through.

**Fix:** Enable `ControlMaster` in `~/.ssh/config` on the control host:
```
Host droplet-*
  ControlMaster auto
  ControlPath /tmp/ssh-%r@%h:%p
  ControlPersist 10m
```

One TCP/SSH handshake; all subsequent `ssh droplet-prod …` invocations multiplex over it. Alternative: batch remote commands into a single `ssh` call.

#### `fnm default` Symlink Over Hardcoded Node Path in systemd

Hardcoding `/home/user/.local/share/fnm/node-versions/v24.3.0/bin/node` into a systemd unit breaks the next `fnm install --lts` upgrade. Use the `fnm default` alias symlink instead:

```
ExecStart=/home/user/.local/share/fnm/aliases/default/bin/node /path/to/bin.js
```

This survives node version bumps. After `openclaw update` or any Node bump, re-run:
```bash
openclaw --profile $P gateway install --force
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway-$P
```

#### Canonical Casing for Log-Marker Greps

When HEARTBEAT (or any monitor) greps for a log-line marker emitted by another script, case matters. One typical incident: writer script emits `— Backup completed`; heartbeat greps for `— backup`; heartbeat reports "backup missing" for days.

**Rule:** Declare canonical casing for every inter-script marker in HEARTBEAT.md (or a shared `CONVENTIONS.md`). Writers and readers both reference that canonical form. Add a test assertion — grep the writer script for the writer-side string AND grep the reader for the reader-side string; fail CI if they diverge.

---

## 14. A2A Architecture Principle

<!-- status: reference | industry-verified: 2026-03-31 -->

When agent-to-agent integration becomes a priority, the server — not the LLM agent — should be the A2A endpoint.

### Core Principle

**External agents should talk to deterministic JSON APIs, not LLM conversations.** The OpenClaw agent's role in A2A is messaging delivery only (DMs, notifications). All business logic — placing orders, checking status, retrieving catalogs — is pure database operations that require no LLM reasoning. Routing external agents through an LLM orchestrator adds latency, cost, and stochastic failure risk to deterministic pipelines.

### Three Protocol Layers

Build in this order — each layer wraps the previous one:

| Layer | Protocol | Purpose | Audience |
|-------|----------|---------|----------|
| 1 | **OpenAPI REST** (`/api/v1/`) | Primary integration surface | Any HTTP client, LangChain, GPT agents, curl |
| 2 | **MCP Server** (stdio/SSE) | Claude ecosystem tool discovery | Claude Code, MCP-aware agents |
| 3 | **A2A Agent Card** (`/.well-known/agent.json`) | Agent discovery | Agent registries, search engines |

Layer 1 (REST) handles 90%+ of integrations. Layer 2 (MCP) wraps the same endpoints as Claude-native tools. Layer 3 (Agent Card) is metadata for discovery — no new logic.

### MCP vs A2A

These are complementary protocols, not competing standards:

- **A2A** (Agent-to-Agent Protocol, Linux Foundation): Agent discovers and collaborates with **other agents**. Peer-to-peer, opaque partners, task-oriented.
- **MCP** (Model Context Protocol, Anthropic): Agent discovers and uses **tools and resources**. Client-server, tool exposure.

An agent can use A2A to communicate with other agents while internally using MCP to access its own tools. Both emphasize structured data exchange (JSON), scoped authentication, and deterministic operations.

### Authentication

API key auth (`X-API-Key` header), per-agent, scoped, rate-limited. OAuth2 is overkill at small scale — API keys are simpler and sufficient for agent-to-agent use. Can be added later.

**Scopes:** `[resource]:read`, `[resource]:write`, `admin` (reserved).

### Agent Card for Discovery

Serve a JSON metadata file at `/.well-known/agent.json` describing the agent's capabilities, supported protocols, and authentication requirements. This follows the A2A spec and enables agent registries and search engines to discover the agent programmatically.

### Implementation Notes

- All A2A code goes in the internal API server — zero OpenClaw changes needed
- Reverse proxy (Caddy/nginx) must expose `/api/v1/*` and `/.well-known/` publicly
- Internal APIs (`/api/internal/*`) remain localhost-only

### Industry Alignment

This architecture aligns with:
- **Linux Foundation A2A Project** (launched April 2025) — task-scoped, deterministic, opaque agent collaboration
- **NIST AI Agent Standards Initiative** (launched February 2026) — enterprise-grade identity, scoped privileges, interoperability
- **Google A2A Protocol** — protocol-agnostic, Agent Card discovery, structured data exchange

All three initiatives converge on deterministic APIs for routine operations, with LLMs reserved for tasks requiring reasoning.

---

## 15. Advanced Design Patterns

### Cross-Session Write Pattern

When sandbox sessions need to persist data to workspace or memory files, they cannot write directly (`sandbox.workspaceAccess: "ro"`). Use a delegation chain:

1. Cron/sandbox session processes data and produces structured output
2. Calls `sessions_send` to `agent:main:main` with explicit write instructions
3. Main session receives the payload and writes to `memory/` or workspace files

This is a specific application of the Delegation Pattern (Section 3) for write operations. All prerequisites apply: `tools.subagents.tools.alsoAllow: ["sessions_send"]` and `agents.defaults.sandbox.sessionToolsVisibility: "all"` must be configured. Split each write intent into a separate `sessions_send` call to avoid partial processing (see Section 13, `sessions_send` Partial Processing).

### Quorum Gates for Shared Files

When multiple cron jobs or data sources contribute to a shared output file (e.g., a daily report, aggregated signals), use a quorum gate to prevent bad-day runs from poisoning downstream consumers.

**Pattern:** If fewer than N out of M sources pass a quality gate (data freshness, minimum row count, format validation), keep the previous file intact. Only overwrite when the quorum passes.

**Implementation:** The skill checks source quality before writing. If the quorum fails, it logs the failure to `memory/` and exits without modifying the shared file. Include a `run_id` (e.g., `skill-name-2026-04-10T09:00:00Z`) for idempotency — a single `sessions_send` call with `run_id` ensures all-or-nothing writes.

### Skillsync Verification (4 Axes)

After editing any skill that has a cron trigger, verify four axes before considering the change complete:

1. **Registry:** `openclaw skills list` shows the updated skill without errors
2. **Cron snapshot:** If the cron payload references skill content, update it: `openclaw cron edit <id> --message "<updated text>"`
3. **Cron-to-skill mapping:** The cron job's trigger prompt and the skill's frontmatter `name`/`description` agree on invocation conditions
4. **Sandbox tools:** `openclaw sandbox explain` lists all tools the updated skill needs in the allow list

Missing any axis leads to silent failures — the cron fires but the skill runs with stale instructions, missing tools, or a mismatched trigger. Automate this check as a post-edit verification script.

### Signals vs. Observations Data Pattern

For agents that collect and consume recurring data across cron cycles, separate storage into two tiers:

| Tier | Update Strategy | Retention | Consumer Pattern |
|------|----------------|-----------|-----------------|
| **Signals** | Deduplicated, compact, overwritten daily | Latest only (max ~30 entries) | Skills read signals — cheap, small context |
| **Observations** | Raw log, append-only, capped by size/age | Rolling window (e.g., 500 lines / 14 days) | Audit trail, debugging, reprocessing |

**Rule:** Consumer skills read signals, not observations. Reading 14 days of raw observation logs is expensive and wastes context; reading 30 signal lines is cheap.

**Implementation:** Signals go in `memory/signals/` (e.g., `memory/signals/market-summary.md`). Observations go in `memory/observations/` (e.g., `memory/observations/2026-04-10.md`). A daily cron processes observations into signals. Apply quorum gates (above) when overwriting signal files.

### Heartbeat Incident Packet Pattern

For agents running repeated skills, HEARTBEAT.md should detect failure patterns and assemble structured incident packets for the operator:

1. Skills log self-check results to `SYSTEM_LOG.md`: `[SELF-CHECK] YYYY-MM-DD HH:MM | <skill> | PASS|FAIL | <detail> | <run_id>`
2. Heartbeat scans SYSTEM_LOG.md for patterns: N or more failures of the same check within M days (e.g., 3+ failures in 7 days)
3. On pattern match, assemble an incident packet:
   ```
   [INCIDENT] <timestamp> | <skill> | <check_id> | <check_name>
   failures: <N> in last 7 days (of <runs_evaluated> total)
   last_good: <most recent PASS run_id>
   last_fail: <most recent FAIL run_id>
   context: <detail from most recent failure>
   ```
4. Send to operator via trusted channel (Telegram DM)

**Dedup:** Max 1 alert per check_id per UTC day to prevent alert fatigue.

**Grace period:** Suppress alerts for 3 hours after gateway restart to avoid false alarms during startup and session warm-up.

**Threshold tuning:** Start with 3 failures in 7 days. Tighten for critical skills (2 in 3 days), loosen for flaky external dependencies (5 in 7 days).

### Harness Self-Eval Escalation Pattern

Self-evaluating agents (skills that score their own outputs and emit a result envelope) surface a gap the basic "N failures → alert" pattern misses: the agent can self-identify regressions that never cross any single check's failure threshold. Three complementary rules close that gap.

**Rule 1 — SKIP-streak watchdog.** Alongside the failure-streak rule, count consecutive `SKIP` outcomes (the skill couldn't even run because upstream inputs were missing or malformed). Alert on `≥3 SKIPs` within 7 days for the same check. SKIP streaks indicate silent input-source degradation — the consuming skill looks healthy but produces nothing.

**Rule 2 — Source-coverage floor.** For skills that ingest multiple independent sources (e.g., compiling signals from N feeds), track a `source_coverage` field: `passed / total`. Fail the outer run if `passed < coverage_floor` (e.g., `<8/10`), independent of whether the LLM's decision was correct. Prevents a low-health input set from masking a bad decision with a plausible output.

**Rule 3 — Evolution-log mid-turn escalation.** When the self-eval envelope reports `outcome ∈ (regressed, source_degraded)` AND `confidence ∈ (medium, high)`, fire the escalation (Telegram DM, incident ledger write) BEFORE the handler returns ACK. Post-ACK logic can be skipped (see §13 "Handler Side-Effects BEFORE ACK-Return").

**Dedup ledger.** A single daily ledger, keyed by `(skill, failure_sig)`, prevents alert storms from the same regression signature. `failure_sig` is a short hash of the envelope's diagnostic fields (not the timestamp or run_id).

**Envelope shape (reference):**
```json
{
  "skill": "compile-signals",
  "run_id": "2026-04-19T07:00:00Z",
  "kind": "self-check",
  "outcome": "regressed",
  "confidence": "high",
  "source_coverage": { "passed": 6, "total": 10 },
  "failure_class": "source_degraded",
  "failure_sig": "sha1:7e2a…"
}
```

Handler emits `OK_APPENDED` AFTER it has routed the envelope to alerts, the ledger, and `SYSTEM_LOG.jsonl`. If the `source_coverage.passed < floor` check triggers, the handler emits `REGRESSION_ESCALATED` and includes the incident packet in the alert body.
