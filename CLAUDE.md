# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code **skill** (not a code project) that provides an interactive setup wizard for connecting Telegram to Claude Code via [telegram-mcp](https://github.com/chigwell/telegram-mcp). There is no application code, build system, or tests — only documentation files that Claude Code interprets as skill instructions.

## Repository Structure

- `SKILL.md` — The skill definition file. This is the core artifact. It uses YAML frontmatter (name, triggers, allowed-tools, metadata) followed by the full wizard instructions that Claude Code follows when the user invokes `/telegram-mcp-setup`.
- `README.md` — Public-facing install instructions for the skill.
- `macos-setup.md` — Platform-specific reference for macOS (Keychain, Docker, Homebrew details).
- `windows-setup.md` — Self-contained Windows guide (experimental). Windows users are routed here entirely.
- `troubleshooting.md` — Error scenarios organized by setup phase.

## How the Skill Works

When installed to `~/.claude/skills/telegram-mcp-setup/SKILL.md`, invoking `/telegram-mcp-setup` triggers Claude Code to follow the wizard in SKILL.md:

1. **Pre-flight check** — Detects OS and available tools (Python, Docker, uv)
2. **API credentials** — Guides user through my.telegram.org (optionally using Claude in Chrome browser automation)
3. **Session string generation** — Three paths: Python+pip (lightest), Docker, or uv
4. **Credential storage** — macOS Keychain via `security` command; Windows uses env vars or config file
5. **MCP registration** — Creates a launcher script at `~/.local/bin/telegram-mcp-docker` and registers via `claude mcp add`
6. **Verification** — Tests the connection with `list_chats`

## Key Design Decisions

- **Session generation is always interactive** — requires a separate terminal window because Claude Code's Bash tool doesn't support `-it` (interactive) input
- **Credentials never leave the machine** — session string stored in Keychain/env vars, never sent to Anthropic; only message *content* is sent when Claude processes it
- **Docker image**: `bayramannakov/telegram-mcp:latest` — used both for session generation and as the MCP server runtime
- **Platform routing**: macOS is primary; Windows users get a fully self-contained guide in `windows-setup.md`

## Editing Guidelines

- `SKILL.md` frontmatter fields (`allowed-tools`, `metadata.version`, triggers in `description`) affect how Claude Code discovers and runs the skill
- The `references/` prefix in SKILL.md paths (e.g., `references/windows-setup.md`) is the expected location when installed as a skill — in this repo they're at root level
- Security language (session string risks, revocation checklist) is intentionally explicit — do not soften it
