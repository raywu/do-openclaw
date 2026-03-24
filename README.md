# OpenClaw + DigitalOcean Setup Guide

Documentation for a **live deployed** OpenClaw instance (open-source, self-hosted AI agent framework) on DigitalOcean infrastructure. This repository contains documentation only — no application code, build system, or test suite.

## Documents

- **`docs/openclaw-setup-guide.md`** — Production deployment walkthrough for Ubuntu 24.04 on DigitalOcean
- **`docs/prompt-claude-code-openclaw-setup.md`** — Interactive Claude Code prompt for guided setup
- **`docs/prompt-multi-agent-openclaw-setup.md`** — Multi-agent orchestration prompt

### References

- **`docs/references/reference-openclaw-design-patterns.md`** — Architecture, session/sandbox model, skill patterns, cron patterns, memory system, messaging patterns
- **`docs/references/reference-openclaw-prompt-caching.md`** — Anthropic prompt caching configuration and cost optimization

## Key Concepts

- **OpenClaw Gateway** binds to `127.0.0.1:18789` — never expose publicly; access via SSH tunnel
- **Workspace files** (SOUL.md, IDENTITY.md, AGENTS.md, TOOLS.md, USER.md, HEARTBEAT.md, BOOT.md) define agent identity and behavior — loaded into system prompt each message
- **Dual enforcement model**: soft (LLM reasoning via workspace markdown) + hard (Gateway tool policies, sandbox, OS-level containment)
- **Skills** are `SKILL.md` files with YAML frontmatter in `~/.openclaw/workspace/skills/<name>/`
- **Memory** uses daily markdown files + SQLite hybrid search (vector + BM25)
- **DEV/PROD split**: `~/.openclaw-dev/` is the isolated DEV state dir (`--dev` flag); `~/.openclaw/` is PROD. DEV workspace is the git repo and Claude Code root. `promote.sh` syncs dev → prod with git-aware safety checks
