# macOS Setup Details

Additional details for macOS users. The main SKILL.md covers the full workflow — this file provides deeper platform-specific guidance.

## Opening Terminal

If you've never used Terminal before:
1. Press **Cmd+Space** (opens Spotlight search)
2. Type **Terminal**
3. Press **Enter**
4. A window with a text prompt appears — this is where you type commands

## Installing Prerequisites

### Option 1: Python Only (Lightest)

Most Macs come with Python 3 pre-installed. Check:
```bash
python3 --version
```

If not found:
```bash
# Via Homebrew (if installed)
brew install python3

# Or download from python.org — no Homebrew needed
# https://www.python.org/downloads/
```

### Option 2: Docker Desktop

Download size: ~1.1GB. Installed size: ~4GB. Requires ~2GB free for containers.

```bash
# Via Homebrew
brew install --cask docker

# Or download directly from:
# https://docker.com/products/docker-desktop
```

After installation:
1. Open Docker Desktop from Applications
2. First launch takes 2-5 minutes (longer than subsequent starts)
3. Wait for whale icon in menu bar to stop animating
4. Verify: `docker --version`

**Apple Silicon (M1/M2/M3):** Docker Desktop runs natively. No Rosetta needed for this use case.

### Don't have Homebrew?

Install: `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`

Or skip Homebrew entirely — download Python or Docker directly from their websites.

### Don't have git?

Needed only for Option C (uv + git clone). Install via:
```bash
xcode-select --install
```
This installs Apple's Command Line Tools which includes git. Takes 5-10 min.

## Keychain Details

macOS Keychain is an encrypted password store protected by your login password.

### How the `security` command works

```bash
# Store a value (-U updates existing entries)
security add-generic-password -a "ACCOUNT" -s "SERVICE" -w "VALUE" -U
#   -a = account name (identifier within the service)
#   -s = service name (groups related entries)
#   -w = the value to store (password/secret)
#   -U = update if exists, create if not

# Retrieve a value
security find-generic-password -a "ACCOUNT" -s "SERVICE" -w

# Delete an entry
security delete-generic-password -a "ACCOUNT" -s "SERVICE"
```

### Keychain access prompts

macOS may prompt for your password when accessing Keychain from Terminal. This is normal. Click "Allow" or "Always Allow."

If using Touch ID, Keychain may use it for additional security.

### Verify all credentials are stored

```bash
echo "API ID: $(security find-generic-password -a "api_id" -s "telegram-mcp" -w 2>/dev/null && echo 'OK' || echo 'MISSING')"
echo "API Hash: $(security find-generic-password -a "api_hash" -s "telegram-mcp" -w 2>/dev/null && echo 'OK' || echo 'MISSING')"
echo "Session: $(security find-generic-password -a "session_string" -s "telegram-mcp" -w 2>/dev/null | wc -c | xargs) chars stored"
```

## Session String with Special Characters

Session strings are base64-like and may contain `+`, `/`, `=`. Use **single quotes** when storing to prevent shell interpretation:

```bash
# Correct — single quotes prevent shell expansion
security add-generic-password -a "session_string" -s "telegram-mcp" -w '1BVtsOH8Bu+x/abc==' -U

# Wrong — double quotes may break on + and other chars
security add-generic-password -a "session_string" -s "telegram-mcp" -w "1BVtsOH8Bu+x/abc==" -U
```

## Docker Runtime Dependency

**Docker Desktop must be running every time** you use Telegram tools in Claude Code.

If Docker Desktop is closed:
- Telegram MCP tools will silently fail or return errors
- Restart Docker Desktop and wait for it to start
- No need to re-register the MCP server — just restart Docker

To set Docker Desktop to start on login:
Docker Desktop > Settings > General > "Start Docker Desktop when you log in"

## Launcher Script Details

The launcher script at `~/.local/bin/telegram-mcp-docker`:
1. Reads credentials from Keychain using the `security` command
2. Passes them as environment variables to the Docker container
3. Runs the telegram-mcp server with `docker run --rm -i` (the `-i` flag is **required** — it keeps stdin open for MCP's stdin/stdout JSON-RPC protocol; without it, Docker closes stdin and the MCP server can't communicate with Claude Code)

The script is created automatically by the skill. If you need to recreate it manually, see the main SKILL.md Step 4.

## Corporate/Managed Macs

If your Mac is managed by an employer (MDM profile):
- Docker Desktop may be blocked by IT policy
- Keychain may require additional approval
- Terminal usage may be restricted

In these cases, use the Python/pip option (Option A) which has fewer restrictions, and store credentials in a secured .env file instead of Keychain.

## Uninstall

Complete removal:
```bash
# 1. Terminate session in Telegram app
# (Settings > Privacy > Active Sessions > Terminate)

# 2. Remove from Claude Code
claude mcp remove telegram-mcp

# 3. Delete credentials
security delete-generic-password -a "api_id" -s "telegram-mcp" 2>/dev/null
security delete-generic-password -a "api_hash" -s "telegram-mcp" 2>/dev/null
security delete-generic-password -a "session_string" -s "telegram-mcp" 2>/dev/null

# 4. Delete launcher script
rm -f ~/.local/bin/telegram-mcp-docker

# 5. Remove Docker image (optional)
docker rmi bayramannakov/telegram-mcp:latest 2>/dev/null

# 6. Remove cloned repo if used
rm -rf ~/telegram-mcp
```
