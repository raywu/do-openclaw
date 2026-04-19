# do-openclaw

A production-hardened deployment guide for running [**OpenClaw**](https://docs.openclaw.ai) — an open-source, self-hosted AI agent framework — on DigitalOcean. Aimed at operators who want a single droplet, clean config, and an honest list of the gotchas you only discover in production.

This is a **documentation-only** repo: no application code, no build system, no tests. The artifacts are:

- a step-by-step manual setup guide you can follow yourself
- a copy-pasteable prompt that hands the setup to Claude Code running on the droplet
- reference docs for architecture, configuration, and ~60 operational learnings harvested from live deployments

## Why it exists

OpenClaw ships with excellent platform docs and a marketplace 1-Click image. What's missing is the "day 2" layer: how to harden the host, wire up DEV/PROD cleanly on one droplet, deploy custom skills without silently breaking them, and catch the dozen failure modes that only show up under real load (config-cache drift, main-session cron serialization, sandbox-lifecycle races, hallucinated tool calls, and so on).

`do-openclaw` is where we record that layer and keep it generic — nothing here is business-specific. Patterns are generalized from two production deployments and refreshed on a rolling cadence.

## How to use it

You have two paths. Both target **Ubuntu 24.04 on a DigitalOcean Premium AMD droplet** with Claude Code already installed.

### Path A — Follow the guide yourself

```text
https://github.com/raywu/do-openclaw/blob/master/docs/openclaw-setup-guide.md
```

Work through Phase 1 (provision + harden) and Phase 2 (install + configure OpenClaw) in order. The guide uses `[PLACEHOLDER]` tokens for anything you need to customize (sheet IDs, channel IDs, dates). Expect 60–90 minutes for a fresh droplet.

### Path B — Let Claude Code drive it

SSH into your droplet as the non-root user, start a tmux session, launch `claude`, and paste this single message:

```text
Read the do-openclaw deployment scaffold at https://github.com/raywu/do-openclaw and use it to set up OpenClaw on this droplet.

Start with docs/prompt-claude-code-openclaw-setup.md — follow the task blocks in order, stop at every human gate (marked 🛑), and list all [PLACEHOLDER] tokens you need me to fill in. Do not improvise file contents — every workspace file has exact content that matters for security.
```

Claude Code reads the prompt doc, walks through the 15 task blocks with you, creates workspace files verbatim, and pauses at each gate so you can verify before it moves on.

### What happens after setup

1. OpenClaw Gateway is running at `127.0.0.1:18789` behind the firewall (access via `ssh -L 18789:localhost:18789 …`).
2. Your agent has the standard workspace files (`SOUL.md`, `IDENTITY.md`, `AGENTS.md`, `TOOLS.md`, `USER.md`, `HEARTBEAT.md`, `BOOT.md`, `MEMORY.md`) filled in with your identity and security posture.
3. Two environments share the droplet: **DEV** at `~/.openclaw-dev/` (git repo + `--dev` flag) and **PROD** at `~/.openclaw/`. The `promote.sh` family syncs DEV → PROD.
4. Customize from there: add skills in `workspace/skills/<name>/SKILL.md`, register cron jobs with `openclaw cron add`, and wire up channels (Telegram, WhatsApp) once you're ready for real traffic.

Then: start harvesting your own learnings. Every production deployment finds failure modes that aren't in the reference yet.

## What's in the repo

| Path | What it is |
|------|------------|
| [`docs/openclaw-setup-guide.md`](docs/openclaw-setup-guide.md) | Manual walkthrough for Ubuntu 24.04 on DigitalOcean — Phase 1 (provision/harden) + Phase 2 (install) + Phase 3 (workspace files) |
| [`docs/prompt-claude-code-openclaw-setup.md`](docs/prompt-claude-code-openclaw-setup.md) | Single prompt for Claude Code — 15 task blocks, human gates, exact-content workspace templates |
| [`docs/references/reference-openclaw-design-patterns.md`](docs/references/reference-openclaw-design-patterns.md) | Architecture, session/sandbox model, skill patterns, cron patterns, memory system, messaging, environment architecture, promotion workflow, **operational learnings** (§13), A2A principle, advanced patterns |
| [`docs/references/reference-openclaw-prompt-caching.md`](docs/references/reference-openclaw-prompt-caching.md) | `cacheRetention` configuration for Anthropic prompt caching — the single biggest cost lever on chatty agents |
| [`docs/references/`](docs/references/) *(other)* | DigitalOcean setup evaluation, order/CRM skill reference, Shopify/Gmail research, skill editing notes, WhatsApp injection defense, X API cost projection |
| [`CLAUDE.md`](CLAUDE.md) | Dense reference of every config pattern and gotcha — the file Claude Code loads first when working in this repo |

## Key concepts

- **OpenClaw Gateway** binds to `127.0.0.1:18789` — never expose publicly; access via SSH tunnel.
- **Workspace files** (`SOUL.md`, `IDENTITY.md`, `AGENTS.md`, `TOOLS.md`, `USER.md`, `HEARTBEAT.md`, `BOOT.md`) define agent identity and behavior — loaded into the system prompt on every message.
- **Dual enforcement**: soft (LLM reasoning via workspace markdown) + hard (Gateway tool policies, sandbox, OS-level containment). Both layers matter; neither is sufficient alone.
- **Skills** are `SKILL.md` files with YAML frontmatter in `~/.openclaw/workspace/skills/<name>/`. Skill bodies load verbatim; store rules and workflows there, not in memory.
- **Memory** uses daily markdown files (`memory/YYYY-MM-DD.md`) + SQLite hybrid search (vector + BM25). `MEMORY.md` holds durable facts; daily files hold observations.
- **SYSTEM_LOG split**: `memory/SYSTEM_LOG.jsonl` for machine-readable self-check envelopes; `memory/SYSTEM_LOG.md` for human-readable ops text (BOOT, BACKUP, INCIDENT). Readers pick the file by purpose.
- **DEV/PROD split**: `~/.openclaw-dev/` (the `--dev` flag, the git repo) → `~/.openclaw/` (PROD). `promote.sh` syncs DEV → PROD with git-aware safety checks and runtime-state exclusions.
- **Cross-profile cron validator** (`validate-crons.sh` pattern): one script SSHs every profile, classifies each job's state (stale / auth / infra / duration-creep), and exits RED on any failing profile. Catches silent 401 streaks that per-job metadata reports as `lastStatus: success`.
- **Custom MCP service deployment**: esbuild bundle + user systemd unit + token fingerprinted at three locations (deploy secret, gateway `.env`, MCP `/healthz`). Mismatch fails the deploy instead of producing a "false-green" 401 loop.

For the dense config-pattern reference (~80 entries including the 2026-04-19 harvest), see [`CLAUDE.md`](CLAUDE.md).

## Operational learnings

Every production OpenClaw deployment eventually hits the same set of failure modes. Rather than learn them from scratch, start here:

[`reference-openclaw-design-patterns.md` §13 "Operational Learnings"](docs/references/reference-openclaw-design-patterns.md#13-operational-learnings)

Organized by area — Sandbox & Exec Isolation, Session & Model Behavior, Delivery & Messaging, Cron & Scheduling, Memory, Deployment & Upgrades — each entry has a diagnostic signature, a fix, and (usually) the config snippet you need. §15 covers the larger architectural patterns: cross-session writes, quorum gates, signals vs. observations, heartbeat incident packets, harness self-eval escalation.

## Canonical upstream

[docs.openclaw.ai](https://docs.openclaw.ai)

## License

MIT — see [`LICENSE`](LICENSE).
