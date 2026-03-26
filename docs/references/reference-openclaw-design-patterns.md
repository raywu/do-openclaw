# OpenClaw Design Patterns & Architecture Reference

<!-- last-verified: 2026-03-26 | openclaw: 2026.3.8 -->

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

### Strict Schema Validation

OpenClaw validates `openclaw.json` against a strict schema. Invalid keys or types cause startup failures. When making config changes, always specify the exact JSON path and expected type. Test config changes by restarting the gateway and checking for schema errors.

### Cron Payload Cache Sync

Cron payloads are **static** — captured at registration time. Editing a SKILL.md does NOT update the corresponding cron job's payload. When editing a skill that has a cron trigger:

1. Edit the skill file
2. Check if the cron payload conflicts with the updated skill
3. If so: `openclaw cron edit <id> --message "<updated text>"` (agentTurn) or `--system-event "<updated text>"` (systemEvent)
4. Prefer minimal trigger prompts ("Run the [SKILL_NAME] skill") over inline instructions to minimize drift

---

## 13. Operational Learnings

Hard-won lessons from production debugging. Each entry caused a real incident or wasted significant debugging time.

### PATH Issues in Cron Scripts

Scripts invoked by OpenClaw cron jobs (`systemEvent`) run in a minimal shell environment. The `openclaw` binary (installed via npm) may not be on `$PATH`.

**Fix:** All cron-invoked scripts should:
1. Export the npm global bin directory to PATH: `export PATH="$HOME/.npm-global/bin:$PATH"`
2. Use an `$OPENCLAW` env var (defaulting to `openclaw`) for CLI invocation
3. Derive workspace paths dynamically from `$(dirname "$0")/..` rather than hardcoding

### Delivery Mode Leaks

The `delivery.mode: "announce"` setting on cron jobs sends the agent's final output to the configured channel. If the agent includes internal notes, debug info, or operator-only context in its output, that content leaks to the delivery channel.

**Rule:** Skills with `delivery.mode: "announce"` must structure their output so only the customer-facing message appears. Internal summaries go to the operator channel via explicit `message` tool calls, not as part of the agent's conversational output. Better yet, use `delivery.mode: "none"` on all crons (see Section 4 Rule 7).

### `sessions_send` Auto-Delivery Leak

`sessions_send` to `agent:main:main` for customer DM delivery is dangerous. The main session's auto-delivery context routes responses to whatever channel was most recently active. If a customer DM session is active, ALL main session output — including internal reasoning, cron summaries, and debug text — auto-delivers to that customer.

**Root cause:** Main session output routing is non-deterministic. Multiple fix attempts (session key format, stronger SOUL.md instructions) fail because the routing depends on gateway state, not agent reasoning.

**Final fix pattern:** Server-side DM delivery. The internal API server calls `openclaw message send` CLI directly (via process execution) to deliver DMs through the gateway, bypassing agent sessions entirely. Skills return structured action objects (`{type: "[CHANNEL]_dm", phone, message}`) instead of sending messages themselves.

**Rule:** Never use `sessions_send` for customer-facing message delivery. `sessions_send` is still valid for internal delegation (memory writes, data reads, sheet updates).

### Port Fixup for Multi-Environment Skills

When skills hardcode `localhost:[PORT]` for internal API calls, and DEV/PROD servers run on different ports (e.g., PROD on `:3000`, DEV on `:3001`), promotion scripts need port fixup.

**Pattern:** `promote-dev.sh` runs sed to replace `:3000` → `:3001` in skill files. `promote.sh` reverses the replacement for PROD.

**Gotcha:** The fixup sed must exclude the promotion scripts themselves — otherwise it rewrites the sed patterns inside `promote*.sh`, turning `s|3001|3000|` into `s|3001|3001|` (a no-op). Use `find -not -name 'promote*.sh'` in the fixup command.
