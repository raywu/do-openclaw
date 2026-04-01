# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a documentation repository for the **OpenClaw** open-source, self-hosted AI agent framework — specifically, deployment guides and reference material for running OpenClaw on DigitalOcean infrastructure. There is no application code, build system, or test suite.

### Document Structure (`docs/`)

**Core:**
- **`openclaw-setup-guide.md`** — Generic deployment walkthrough for Ubuntu 24.04 on DigitalOcean (vanilla OpenClaw, no business-specific content)
- **`prompt-claude-code-openclaw-setup.md`** — Generic Claude Code prompt for guided setup (mirrors the setup guide)

**Other:**
- **`prompt-multi-agent-openclaw-setup.md`** — Multi-agent orchestration prompt (pre-v3, not yet updated)
- **`references/reference-openclaw-design-patterns.md`** — Architecture, session model, skill patterns, cron patterns, memory system, messaging patterns, environment architecture, promotion workflow, anti-patterns, CLI-to-API migration (auto-migrate), configuration patterns, operational learnings, A2A architecture
- **`references/reference-openclaw-prompt-caching.md`** — Anthropic prompt caching configuration (`cacheRetention`)
- **`references/reference-openclaw-digitalocean-setup-evaluation.md`** — Architecture evaluation: seven-layer architecture with dual specialization/security analysis
- **`references/reference-openclaw-order-crm-tools-skills.md`** — Order/CRM skill reference
- **`references/reference-openclaw-shopify-gmail-research-report.md`** — Shopify/Gmail integration research
- **`references/reference-openclaw-skill-editing-report.md`** — Skill editing patterns
- **`references/reference-whatsapp-injection-defense-analysis.md`** — WhatsApp injection defense analysis

## Key Config Patterns

- **Env var interpolation**: `${VAR_NAME}` in `openclaw.json` and `exec-approvals.json`; auto-loads `~/.openclaw/.env`
- **Gateway**: `mode: "local"` (not `gateway.bind`); binds to `127.0.0.1:18789`
- **Google Sheets**: `gog sheets` CLI (never `gsheet`)
- **Exec policy**: allowlist mode in `exec-approvals.json` (`security: "allowlist"`, not `"deny"`)
- **CRON**: uses `--cron` and `--message` flags (not `--schedule`/`--command`)
- **CRON exec caveat**: isolated sessions hit approval gates; use `session=main` for CRON jobs needing exec
- **CRON version control**: `jobs.json` is snapshotted into `workspace/cron/` by hourly checkpoint and tracked in git
- **Models**: primary `anthropic/claude-sonnet-4-6`, fallback `google/gemini-2.5-pro`; roster includes `gemini-3-pro-preview` and `claude-sonnet-4-5`
- **Sandbox**: `non-main` mode, `openclaw-sandbox:bookworm-slim`, 512m memory, 128 PIDs, read-only root, workspace read-only in sandbox
- **Cron isolation**: `sessionTarget: "isolated"` prevents cron output leaking to customer channels; system tasks use `systemEvent` on main session
- **Cron delivery mode**: `delivery.mode: "none"` on ALL cron jobs — without this, output auto-delivers to main session's last active channel
- **Sandbox allowlist rebuild**: changes to `tools.sandbox.tools.allow` require `openclaw sandbox recreate --all` — NOT hot-reloaded
- **Sandbox Docker image**: base `debian:bookworm-slim` has no `curl`; must add to Dockerfile for `exec curl` skills
- **Exec-approvals cross-contamination**: `lastResolvedPath` in `exec-approvals.json` caches per-env; clear when copying between envs
- **Cron payload static**: editing a SKILL.md does NOT update the cron payload; must `openclaw cron edit` separately
- **Strict schema validation**: `openclaw.json` is validated against strict schema; invalid keys cause startup failure
- **`sessions_send` from isolated cron (v2026.3.8+)**: requires `tools.subagents.tools.alsoAllow: ["sessions_send"]` and `agents.defaults.sandbox.sessionToolsVisibility: "all"` — without both, isolated cron delegation silently fails
- **Prompt caching**: `cacheRetention: "long"` in `agents.defaults.models` for Anthropic models with >5 min interaction gaps; Anthropic-only feature
- **LOCAL env (optional)**: symlink workspace to git repo for immediate visibility during development — no promote step needed

## Key Concepts

- **Context injection order**: SOUL.md → AGENTS.md → TOOLS.md → USER.md → MEMORY.md → daily memory → matching skills
- **Workspace files** (SOUL.md, IDENTITY.md, AGENTS.md, TOOLS.md, USER.md, HEARTBEAT.md, BOOT.md, CLAUDE.md, MEMORY.md, SYSTEM_LOG.md) define agent identity, behavior, and state — loaded into system prompt each message
- **Dual enforcement model**: soft (LLM reasoning via workspace markdown) + hard (Gateway tool policies, sandbox, OS-level containment)
- **Skills** are `SKILL.md` files with YAML frontmatter in `~/.openclaw/workspace/skills/<name>/`
- **Memory** uses daily markdown files + SQLite hybrid search (vector + BM25)
  - Activated 2026-02-28. `MEMORY.md` (long-term) + `memory/YYYY-MM-DD.md` (daily logs)
  - Auto-indexed via Gemini `gemini-embedding-001` embeddings + BM25 full-text
  - Agent writes daily observations and updates MEMORY.md when durable facts change
  - Boot and heartbeat checks verify memory index is non-empty
  - `memory_search` and `memory_get` must be in `tools.sandbox.tools.allow` — auto-detection provisions the index but sandbox blocks tool use without explicit allow
- **Three-environment model**: LOCAL (developer workstation, symlinked workspace) → DEV (Droplet, idle staging) → PROD (Droplet, live). All use `~/.openclaw-dev/` for LOCAL/DEV, `~/.openclaw/` for PROD. `promote.sh` family handles deployment.
- **Promotion scripts**: `promote.sh` (LOCAL→PROD), `promote-dev.sh` (LOCAL→DEV), `promote-skill.sh` (single skill), `auto-promote.sh` (CI/CD), `rollback.sh` (emergency)

## Git

- Only `master` branch (no `main`)
- Deploy key isolation via SSH aliases (`github-openclaw`, `github-backup`) — no default key for bare `git@github.com`
- Agent exec restricted to `~/scripts/safe-git.sh` wrapper (blocks `remote` subcommand)

## When Editing

- Maintain the document's evaluation structure (Part 1: architecture context, Part 2: critique, Part 3: rewrite) where applicable
- Preserve severity ratings (Critical/Moderate/Minor) and issue numbering in evaluation docs
- Keep the security-first posture — every recommendation should consider both specialization and constraint
- Infrastructure references target Ubuntu 24.04 + DigitalOcean Premium AMD Droplets
- Generic platform docs use `[PLACEHOLDER]` format for customizable values
- Deployment-specific content (skills, CRON jobs, business logic) belongs in a separate companion doc, not in the generic platform docs
- Always use `gog sheets` — never `gsheet`
