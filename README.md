# Telegram MCP Setup Wizard

A Claude Code skill that walks you through connecting Telegram to Claude Code via [telegram-mcp](https://github.com/chigwell/telegram-mcp).

## What It Does

Interactive setup wizard that handles the entire process:
- Getting API credentials from my.telegram.org
- Generating a session string (Python, Docker, or uv)
- Storing credentials securely in macOS Keychain
- Registering the MCP server in Claude Code

## Install

The easiest way — ask Claude Code:

```
Install the telegram-mcp-setup skill from https://github.com/BayramAnnakov/telegram-mcp-setup
```

Or download manually:

```bash
mkdir -p ~/.claude/skills/telegram-mcp-setup
curl -o ~/.claude/skills/telegram-mcp-setup/SKILL.md \
  https://raw.githubusercontent.com/BayramAnnakov/telegram-mcp-setup/main/SKILL.md
```

## Usage

In Claude Code:

```
/telegram-mcp-setup
```

## Requirements

- macOS (primary), Windows (experimental)
- Python 3.10+ for session string generation
- Optional: Claude in Chrome extension for browser-guided setup

## License

MIT
