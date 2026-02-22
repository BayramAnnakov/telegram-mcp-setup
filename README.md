# Telegram MCP Setup Wizard

A Claude Code skill that walks you through connecting Telegram to Claude Code via [telegram-mcp](https://github.com/chigwell/telegram-mcp). Read, search, and draft messages in your Telegram chats — all from Claude.

## What It Does

Interactive setup wizard that handles the entire process:

1. **API credentials** — Guides you through my.telegram.org (optionally using Claude in Chrome browser automation)
2. **Session string** — Four generation paths: QR code login (recommended), Python + venv, Docker, or uv
3. **Secure storage** — Stores credentials in macOS Keychain (or env vars on Windows)
4. **MCP registration** — Creates a launcher script and registers the server in Claude Code
5. **Verification** — Tests the connection with `list_chats`

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

After setup, install the companion [telegram-assistant](https://github.com/BayramAnnakov/telegram-mcp-setup) skill for workflows like daily digests, chat search, and writing style analysis.

## Requirements

- **macOS** (primary) or **Windows** (experimental — see [windows-setup.md](windows-setup.md))
- **One of:** Python 3.10+, Docker, or uv — for session string generation
- **Optional:** [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/clkhfmahgnpfcoifnhidlljmjkmaonlk) extension for browser-guided credential setup

## How Data Flows

- **telegram-mcp runs locally** on your machine and fetches messages from Telegram servers.
- When Claude processes those messages (summarize, search, draft replies), **the message content is sent to Anthropic's API** — same as pasting messages into Claude manually.
- **The session token stays local** — it never leaves your machine.

## Reference Guides

- [macos-setup.md](macos-setup.md) — macOS-specific details (Keychain, Docker, Homebrew)
- [windows-setup.md](windows-setup.md) — Complete self-contained guide for Windows
- [troubleshooting.md](troubleshooting.md) — Error scenarios organized by setup phase

## Ran Into Issues?

Setup not working? We want to hear about it. Please [open an issue](https://github.com/BayramAnnakov/telegram-mcp-setup/issues/new) with:

- **What step failed** (pre-flight, credentials, session generation, registration, or verification)
- **Error message** (copy-paste the exact output)
- **Your environment** (OS, Python/Docker version, Claude Code version)

Your diagnosis reports help us fix bugs and improve the wizard for everyone.

## License

MIT
