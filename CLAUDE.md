# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a documentation repository for the **OpenClaw** open-source, self-hosted AI agent framework — specifically, deployment guides and reference material for running OpenClaw on DigitalOcean infrastructure. There is no application code, build system, or test suite.

### Document Structure (`docs/`)

**Core:**
- **`openclaw-setup-guide.md`** — Generic deployment walkthrough for Ubuntu 24.04 on DigitalOcean (vanilla OpenClaw, no business-specific content)
- **`prompt-claude-code-openclaw-setup.md`** — Generic Claude Code prompt for guided setup (mirrors the setup guide)

**Other:**
- **`prompt-multi-agent-openclaw-setup.md`** — **DEPRECATED (2026-04-19).** Use `prompt-claude-code-openclaw-setup.md` instead. File retained for link stability; do not add new content.
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
- **Strict schema validation**: `openclaw.json` is validated against strict schema; invalid keys cause startup failure. Never hand-edit — use `openclaw config set`
- **Workspace path default**: defaults to `~/.openclaw/workspace/` (shared) unless `agents.defaults.workspace` explicitly set per profile
- **Cron delivery mode flag**: `--delivery-mode none` (not `--params`); `cron list --json` returns `{"jobs": [...]}` dict
- **Gateway .env baking**: `openclaw gateway install` bakes `.env` into systemd; after `.env` changes: `gateway install --force` + `daemon-reload`
- **Telegram IPv6**: OpenClaw 2026.3.24+ / Node 24 `autoSelectFamily=true` breaks on droplets without public IPv6; fix: `channels.telegram.network.autoSelectFamily false`
- **`sessions_send` from isolated cron (v2026.3.8+)**: requires `tools.subagents.tools.alsoAllow: ["sessions_send"]` and `agents.defaults.sandbox.sessionToolsVisibility: "all"` — without both, isolated cron delegation silently fails
- **Prompt caching**: `cacheRetention: "long"` in `agents.defaults.models` for Anthropic models with >5 min interaction gaps; Anthropic-only feature
- **Session maintenance**: `session.maintenance.mode: "enforce"` + `cron.sessionRetention: "4h"` to prevent sandbox container buildup
- **Thinking for tool calls**: `params.thinking: "high"` on model config required for tool calls from messaging channel turns (Telegram, WhatsApp)
- **Sandbox env var forwarding**: sandbox containers don't inherit host env; use `sandbox.docker.env` with SecretRef `${VAR_NAME}`; vars named KEY/SECRET/TOKEN/PASSWORD are blocked
- **`strictInlineEval`**: default `true` blocks `node -e`, `python -c` even when binary is allowlisted; use `exec curl` or `exec printenv` instead
- **`autoAllowSkills`**: set `true` in exec-approvals if skills declare `requires.bins`; without it, socket approval times out at 120s in cron/sandbox
- **Sandbox workspace snapshot**: only copies identity files + `skills/` dir; custom root workspace files unavailable in sandbox
- **Cron delivery channel syntax**: `--channel telegram --to <chatId>` (not combined `telegram:dm:<chatId>`)
- **`systemEvent` relay regression (v2026.4.1+)**: runtime event trust wraps payloads as untrusted; use system crontab for deterministic shell ops
- **Upgrade ceremony**: after `openclaw update`, must `gateway install --force` + `daemon-reload` per profile
- **LOCAL env (optional)**: symlink workspace to git repo for immediate visibility during development — no promote step needed
- **No shell expansion in exec**: `exec` uses `execFile`, not shell — `$VAR` passed as literal; use `exec printenv` or API endpoints via `exec curl`
- **Hardcoded URLs + sed fixup**: env vars, wrapper scripts, and LLM variable substitution all fail for URL parameterization in skills; hardcode + sed in promote scripts
- **One-shot auto-trigger**: `openclaw cron add --at 1m --delete-after-run` for event-driven skill execution; use atomic `UPDATE...RETURNING` for claim-before-act
- **Deploy rsync silent skip**: rsync can report success while SKILL.md files stay stale; always post-deploy grep for a known marker
- **`sessions_send` ACK verification**: grepping session JSONL for ACK strings gives false positives from template text; pair by `toolCallId` to distinguish real ACKs
- **Main-session cron `wakeMode`**: cron with `sessionTarget: "main"` must use `wakeMode: "next-heartbeat"` — `wakeMode: "now"` polls for 120s and serializes all infra crons behind in-flight operator DMs
- **Config cache freshness**: sync scripts compare `cachedAt` (fresh-fetch ms), not `lastUpdated` (DB-mod ms); use `cachedAt ?? lastUpdated` fallback for transition safety
- **Cron snapshot per-env**: each env owns its own `workspace/cron/jobs.json` via hourly checkpoint — add `--exclude='cron/'` to all cross-env rsync (promote.sh, deploy.sh)
- **Silent-skip → loud-fail**: reconciliation scripts (`sync-cron-from-config.sh` et al.) must `echo >&2` + `exit 2` on guard-skip, never `exit 0`. Invisible auto-fix loops are the #1 cause of undetected drift
- **Delivery-mode audit whitelist**: cron delivery audits must skip jobs with `delivery.mode: "none"` (they deliver via sandbox `message` tool, not cron-level channel) — else false-alarm storm masks real failures
- **`deploy.sh --ref <sha|branch>`**: allow DEV to test arbitrary git refs; PROD rejects non-main. Never `git clean -fd` after ref switch (wipes runtime `memory/` / `cron/` state)
- **Post-deploy session warmup**: every deploy script must `sessions delete agent:main:main` + send a warmup ping (both non-fatal) to refresh the cached handler after HEARTBEAT.md changes
- **Exec-approvals bare-name parity**: allowlist BOTH bare name (`curl`) and absolute path (`/usr/bin/curl`) — `exec` uses `execFile` without PATH resolution; also add `ls`, `cat`, `env`, `grep`, `printenv` for self-diagnosis
- **`fnm default` symlink in systemd**: point systemd `ExecStart` at `~/.local/share/fnm/aliases/default/bin/node` (not a pinned version); survives `fnm install --lts`
- **SYSTEM_LOG split**: `SYSTEM_LOG.jsonl` for self-check envelopes (machine-readable), `SYSTEM_LOG.md` for text ops writers (BOOT/BACKUP/INCIDENT); exclude both from rsync
- **Handler side-effects BEFORE ACK-return**: in HEARTBEAT handlers, Telegram alerts / ledger writes / incident escalations MUST run in a step numbered below the ACK-return step — post-ACK logic can be silently skipped
- **Post-deploy marker grep (mandatory)**: every deploy that edits workspace files bakes a unique marker and grep-verifies it post-rsync; rsync's success signal is unreliable
- **SSH ControlMaster**: UFW rate-limits SSH at 6 conn / 30s. Fleet-ops scripts enable `ControlMaster auto` in `~/.ssh/config` or batch commands into one `ssh` call
- **Canonical log-marker casing**: declare canonical casing for inter-script log markers in HEARTBEAT.md; assert writer + reader agree (CI check) — mismatched casing silently breaks heartbeat greps
- **Cross-profile cron validator**: standalone `validate-crons.sh` SSHs each profile, runs `cron list --json`, classifies state (stale / auth / infra / duration-creep), exits 1 on RED — catches silent 401 streaks
- **Agent hallucination check**: for operator DM sessions demanding side effects, grep active session JSONL for a `toolCall` block between operator msg and agent reply; absence = hallucination — reset session JSONL + mint new UUID

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
