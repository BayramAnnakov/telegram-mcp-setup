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

Since February 2023, Telegram has been silently suppressing verification codes for some accounts when authenticating new API sessions. The `send_code_request()` call returns success, but the code never arrives — not via in-app message, not via SMS. This is a known upstream issue documented in Telethon #3835, #4041, and #4050 with no reliable workaround.

**What does NOT help:**
- Waiting longer (the code will never arrive for affected accounts)
- Requesting new codes repeatedly (triggers rate limiting / FloodWait)
- Using a VPN or different network
- Reinstalling Telegram
- Using SMS fallback (Telegram suppresses that too)

**What to try first:**
- Check ALL devices with Telegram (phone, desktop, web) — codes go to active sessions first
- Wait 2 full minutes before concluding the code isn't coming
- If using a VoIP number, try a real phone number instead

**Solution:** Use **QR code login** (Option QR in the main skill). QR login uses a completely different authentication flow that bypasses code delivery entirely. The user scans a QR code in Telegram (Settings > Devices > Link Desktop Device) instead of entering a verification code.

### "Short name already taken"
The short name must be globally unique across ALL Telegram apps. Try:
- Your name + "ai": "ivan_ai"
- Your username + numbers: "myapp12345"
- Random combination: any 5-32 letter string

### "Too many tries" / Rate limiting
my.telegram.org rate-limits login attempts. Wait 15-30 minutes and try again. Do NOT repeatedly click "Send code."

## During Session Generation

### "externally-managed-environment" error (PEP 668)

Homebrew Python 3.12+ refuses bare `pip3 install` commands to protect the system Python environment:
```
error: externally-managed-environment
```

**Fix:** Use a virtual environment:
```bash
python3 -m venv /tmp/telegram-session-venv
/tmp/telegram-session-venv/bin/pip install telethon
/tmp/telegram-session-venv/bin/python /tmp/telegram-session-gen.py
```

Do NOT use `--break-system-packages` — it bypasses the protection and can cause system Python issues. The venv approach is cleaner and equally fast.

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

Use the QR login script from the main skill (Option QR), or regenerate with a venv in a **separate Terminal window**:
```bash
python3 -m venv /tmp/telegram-session-venv
/tmp/telegram-session-venv/bin/pip install telethon
```

Write a script to `/tmp/telegram-session-gen.py`:
```python
from telethon.sync import TelegramClient
from telethon.sessions import StringSession
api_id = int(input('API ID: '))
api_hash = input('API Hash: ')
with TelegramClient(StringSession(), api_id, api_hash) as client:
    print(client.session.save())
```

Then run:
```bash
/tmp/telegram-session-venv/bin/python /tmp/telegram-session-gen.py

# Update Keychain with the new session string
security add-generic-password -a "session_string" -s "telegram-mcp" -w 'NEW_SESSION_STRING' -U
```

**Windows:**
```powershell
docker run -it --rm bayramannakov/telegram-mcp:latest python session_string_generator.py
# Then update environment variable:
[System.Environment]::SetEnvironmentVariable('TELEGRAM_SESSION_STRING', 'NEW_STRING', 'User')
```

### "AuthKeyDuplicatedError" — Telegram permanently killed your session key

**Symptom:** telegram-mcp connects but tools fail; `mcp_errors.log` shows:
```
AuthKeyDuplicatedError: The authorization key (session file) was used under two
different IP addresses simultaneously, and can no longer be used.
```

**Cause:** the SAME session string was used by **two live connections at the same time** —
e.g. an interactive Claude window *plus* a scheduled/headless job, or two Claude windows
open at once. When Telegram sees one auth key on two connections it kills the key
**permanently** — retries don't help, you must re-issue it. (This is different from
`AUTH_KEY_UNREGISTERED`, which is an expired/terminated session.)

**Immediate recovery:** regenerate the session string via QR login (Option QR) and update
Keychain — same steps as `AUTH_KEY_UNREGISTERED` above.

**Permanent fixes — pick one:**
- **Separate session per consumer (simplest).** Give each scheduled/headless job its OWN
  session string: one extra QR login → store as a second Keychain entry
  (`session_string_scheduler`) → point only that job at it. Two different keys can never
  collide, even simultaneously, even across IPs. Tip: set `TELEGRAM_DEVICE_MODEL` when
  generating it so the extra session is labeled in Telegram > Settings > Devices.
- **Centralize (best if you run multiple windows).** Run telegram-mcp ONCE as a local
  HTTP service that every window + job shares — see "Running multiple windows or
  scheduled jobs" in the main skill. One process holds the single connection, so the key
  can't duplicate, and Telegram stays available in all windows at once.

**Do NOT** put telegram-mcp behind a generic stdio→HTTP gateway (e.g. supergateway/
mcp-proxy) in stateless mode — it spawns a fresh connection per request and makes this
worse. Use the image's own HTTP transport (`MCP_TRANSPORT=http`).

### MCP server not responding

**First check:** Verify your launcher script uses `docker run --rm -i`, not `docker run --rm`. The `-i` flag keeps stdin open, which is required for MCP's stdin/stdout JSON-RPC protocol. Without it, Docker closes stdin immediately and the MCP server can't communicate with Claude Code.

```bash
# Check the launcher script
cat ~/.local/bin/telegram-mcp-docker | grep "docker run"
# Should show: docker run --rm -i \

# If missing -i, fix it:
# Edit ~/.local/bin/telegram-mcp-docker and add -i after --rm

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

### "claude: command not found"

If `claude` CLI is not found, it may be installed via npx rather than globally. Options:

1. **Use npx directly:**
   ```bash
   npx --yes @anthropic-ai/claude-code mcp add telegram-mcp -s user -- ~/.local/bin/telegram-mcp-docker
   ```

2. **Install globally (recommended):**
   ```bash
   npm install -g @anthropic-ai/claude-code
   ```

After global install, `claude` will be available system-wide.

### `claude mcp add` fails or hangs inside Claude Code

Running `claude mcp add` from within an active Claude Code session (i.e., when Claude runs it via the Bash tool) will fail or hang because it tries to spawn a nested Claude process.

**Workaround 1 (preferred): Edit config directly**

Claude should edit `~/.claude.json` using the Read and Write tools:
1. Read `~/.claude.json`
2. Add the `telegram-mcp` entry to the `mcpServers` object
3. Write the file back

```json
{
  "mcpServers": {
    "telegram-mcp": {
      "command": "/Users/USERNAME/.local/bin/telegram-mcp-docker",
      "scope": "user"
    }
  }
}
```

**Workaround 2: Separate terminal**

Tell the user to run `claude mcp add` in a **separate Terminal window** (not the one running Claude Code):
```bash
claude mcp add telegram-mcp -s user -- ~/.local/bin/telegram-mcp-docker
```

**Workaround 3: Full binary path**

If Claude Code was installed via npm globally, use the full path:
```bash
/usr/local/bin/claude mcp add telegram-mcp -s user -- ~/.local/bin/telegram-mcp-docker
```

### MCP returns errors or garbled responses

**Root cause:** The Docker image's `main.py` has `print()` calls to stdout before `mcp.run_stdio_async()` starts. MCP uses stdout exclusively for JSON-RPC communication — any non-JSON output on stdout corrupts the protocol and causes Claude Code to receive parse errors or garbled data.

**Diagnostic:** Run the launcher script and check what appears on stdout:
```bash
~/.local/bin/telegram-mcp-docker > /tmp/tg-stdout.log 2>/tmp/tg-stderr.log &
sleep 3 && kill %1
cat /tmp/tg-stdout.log
```
If stdout contains any text (like "Starting server..." or similar), the print-redirect fix is needed.

**Fix:** Update the launcher script (`~/.local/bin/telegram-mcp-docker`) to use the `python -c` wrapper that monkey-patches `builtins.print` to redirect to stderr. The `docker run` line should be:
```bash
docker run --rm -i \
    -e TELEGRAM_API_ID \
    -e TELEGRAM_API_HASH \
    -e TELEGRAM_SESSION_STRING \
    bayramannakov/telegram-mcp:latest \
    python -c "
import builtins, sys
_original_print = builtins.print
def _stderr_print(*args, **kwargs):
    kwargs.setdefault('file', sys.stderr)
    _original_print(*args, **kwargs)
builtins.print = _stderr_print
from main import main
main()
"
```

**Why monkey-patch instead of shell redirection?** MCP needs stdout open and clean for JSON-RPC. We can't redirect stdout because MCP uses it. The monkey-patch only intercepts the specific `print()` calls in `main.py` that pollute stdout before `mcp.run_stdio_async()` takes over.

Also verify the `-i` flag is present — without it, Docker closes stdin and MCP can't communicate.

### Multiple Docker images appearing

This is normal when the Docker image is updated. Claude Code's auto-troubleshooting may also pull or build extra images when the MCP server fails to connect.

Clean up old images:
```bash
docker image prune
```

This removes dangling (unused) images. Your current `bayramannakov/telegram-mcp:latest` image will not be affected.

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
