# Troubleshooting Guide

Common issues organized by when they occur.

## Before Setup

### "I don't know how to open Terminal"
- **macOS:** Cmd+Space, type "Terminal", Enter
- **Windows:** Win+X > "Windows PowerShell" or "Terminal"

### "I don't have Python / Docker / git"
Run the pre-flight check from the main SKILL.md to see what's available. The lightest path is Python + pip (Option A).

- **Install Python (macOS):** `brew install python3` or download from python.org
- **Install Python (Windows):** `winget install Python.Python.3.12` or download from python.org
- **Install Docker:** https://docker.com/products/docker-desktop (~1GB download, ~4GB disk)
- **Install git (macOS):** `xcode-select --install`
- **Install git (Windows):** `winget install Git.Git`

### "Not enough disk space for Docker"
Docker Desktop needs ~4GB. If tight on space:
- Use Option A (Python + pip) instead — needs only ~50MB
- Or clean up disk space: `docker system prune -a` if you have old Docker data

## During my.telegram.org Login

### CAPTCHA appears
Browser automation cannot solve CAPTCHAs. The user must:
1. Complete the CAPTCHA manually in the browser
2. Continue with login
3. Navigate to "API development tools" manually if needed

### Verification code not arriving
- Check ALL devices with Telegram (phone, desktop, web)
- Telegram sends to active sessions first, then SMS after a delay
- If no active sessions, Telegram uses SMS (may take 1-2 minutes)
- If using a VoIP number, SMS may not work — use a real phone number

### "Short name already taken"
The short name must be globally unique across ALL Telegram apps. Try:
- Your name + "ai": "ivan_ai"
- Your username + numbers: "myapp12345"
- Random combination: any 5-32 letter string

### "Too many tries" / Rate limiting
my.telegram.org rate-limits login attempts. Wait 15-30 minutes and try again. Do NOT repeatedly click "Send code."

## During Session Generation

### "PHONE_NUMBER_INVALID"
Use international format:
- Correct: `+79123456789` (Russia), `+1234567890` (US)
- Wrong: `89123456789` (missing country code), `(123) 456-7890`

### "PHONE_CODE_INVALID"
- Use the MOST RECENT code (Telegram may send multiple)
- Code expires in 5 minutes — request a new one if needed
- No spaces before or after the code

### "PASSWORD_HASH_INVALID" (2FA error)
- Check caps lock
- Try typing the password in a text editor first to verify
- If forgotten: Telegram > Settings > Privacy > Two-Step Verification > Forgot Password
- **If no recovery email was set:** Cannot proceed. Consider using a different account or disabling/re-enabling 2FA.

### "FloodWait" / Rate limiting during auth
Too many attempts. The error message includes wait time (e.g., "FloodWait 300" = wait 5 minutes). In extreme cases, may need to wait up to 24 hours. Do NOT retry until the wait period expires.

### Session generation hangs / no response
- Press Ctrl+C to cancel
- Check internet connection
- Try again after a minute
- If using VPN, try disabling it (or enabling it if Telegram is blocked in your region)

### Docker image pull is slow
The image needs to download once (~500MB). On slow connections:
- Wait patiently — Docker resumes interrupted downloads
- Consider Option A (pip) instead — only downloads ~5MB

### "Docker: command not found"
- **macOS:** Docker Desktop not running. `open -a Docker` and wait 1-2 min.
- **Windows:** Docker Desktop not in PATH. Try full path or restart PowerShell.

### Interactive terminal issue (-it flag)
`docker run -it` requires an interactive terminal. If running from Claude Code's Bash tool and it hangs:
1. Open a separate Terminal window
2. Run the docker command there manually
3. Return to Claude Code with the session string

## After Setup

### "Could not find the input entity"
Chat ID format issue:
- Users: numeric ID or username without @
- Channels: username (no @) or numeric ID
- Supergroups: prepend `-100` to numeric ID (e.g., `-1001234567890`)
- Try the exact name from `list_chats` output

### "Chat not found"
- Account doesn't have access to this chat
- Check spelling matches exactly
- For private channels, user must be a member

### Drafts not appearing in Telegram
- Open the specific chat in Telegram app
- Scroll to bottom
- Force close and reopen Telegram
- Drafts are per-chat — check the right chat

### "FloodWait" during normal usage
Too many API calls. Space out requests:
- Don't fetch all chats at once
- Wait the specified time before retrying
- Avoid rapid-fire API calls when processing multiple chats

### Session expired / "AUTH_KEY_UNREGISTERED"
Session string no longer valid. Regenerate:

**macOS:**
```bash
# Regenerate (in a separate Terminal window)
pip3 install telethon  # if not installed
python3 -c "
from telethon.sync import TelegramClient
from telethon.sessions import StringSession
api_id = int(input('API ID: '))
api_hash = input('API Hash: ')
with TelegramClient(StringSession(), api_id, api_hash) as client:
    print(client.session.save())
"

# Update Keychain
security add-generic-password -a "session_string" -s "telegram-mcp" -w 'NEW_SESSION_STRING' -U
```

**Windows:**
```powershell
docker run -it --rm bayramannakov/telegram-mcp:latest python setup_wizard.py
# Then update environment variable:
[System.Environment]::SetEnvironmentVariable('TELEGRAM_SESSION_STRING', 'NEW_STRING', 'User')
```

### MCP server not responding
```bash
# Remove and re-register
claude mcp remove telegram-mcp
claude mcp add telegram-mcp -s user -- ~/.local/bin/telegram-mcp-docker

# Verify
claude mcp list | grep telegram

# Restart Claude Code
```

### Telegram tools silently failing
Most common cause: **Docker Desktop is not running.** Check:
- macOS: Look for whale icon in menu bar
- Windows: Look for Docker icon in system tray
- Start Docker Desktop and wait 1-2 min
- No need to re-register — just restart Docker

### High memory usage
- Docker Desktop settings > Resources > limit memory to 2GB
- Stop unused containers: `docker stop $(docker ps -q)`
- Consider switching to Python/pip method for lower resource usage

## Edge Cases

### Multiple Telegram accounts
Each account needs its own session string. You can only have ONE active telegram-mcp at a time. To switch:
1. Generate session string for the other account
2. Update Keychain with new session string
3. Restart Claude Code

### Corporate VPN blocking Telegram
Some VPNs block Telegram API servers. Try:
- Disconnect VPN temporarily during setup
- If VPN is mandatory, ask IT to whitelist Telegram's API endpoints
- Use a different network for initial setup

### Changing phone number / enabling 2FA after setup
- Changing phone number: session string remains valid
- Enabling 2FA: session string remains valid (it's post-auth)
- Disabling 2FA: session string remains valid
- "Terminate All Other Sessions" in Telegram: session string is INVALIDATED — regenerate

### Session string contains special characters
If pasting the session string fails, try:
- Use single quotes: `security add-generic-password ... -w 'STRING_HERE' -U`
- Copy from a text file to avoid clipboard issues
- Ensure no trailing whitespace or newlines

## Getting Help

1. **Check this guide first** — most issues are covered above
2. **Re-run pre-flight check** — verify prerequisites are still working
3. **Check Docker status** — most post-setup issues are Docker not running
4. **telegram-mcp issues:** https://github.com/chigwell/telegram-mcp/issues
5. **Manual setup:** If all automation fails, the core steps are:
   - Get API ID + Hash from my.telegram.org
   - Generate session string with telethon
   - Store in Keychain / env vars
   - Register MCP server with `claude mcp add`
