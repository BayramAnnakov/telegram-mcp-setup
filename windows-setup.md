# Windows Setup Guide (Self-Contained)

Complete guide for setting up telegram-mcp on Windows 11. Follow this guide from start to finish — you do NOT need to read the main SKILL.md.

> **Status:** Experimental. macOS is the primary supported platform.

## What You Need to Know First

- **MCP** — the way Claude Code connects to external tools (like a plugin system)
- **Session string** — an authentication token for your Telegram account. MORE sensitive than a password — see Security section below
- **Docker** — a tool that runs apps in isolated containers (~1GB download, ~4GB disk)
- **API ID & Hash** — free credentials from Telegram to register your "app"
- **PowerShell** — Windows' command-line tool. Press Win+X > "Windows PowerShell" or "Terminal"

## Time Estimate

| Scenario | Time |
|----------|------|
| Already have Docker Desktop | 10-15 min |
| Need to install Docker | 30-45 min |
| Something goes wrong | Budget 1 hour |

## Pre-Flight Check

Open PowerShell (Win+X > Terminal) and run:
```powershell
Write-Host "=== Pre-flight Check ==="
Write-Host "Docker: $(try { docker --version } catch { 'NOT FOUND' })"
Write-Host "Python: $(try { python --version } catch { 'NOT FOUND' })"
Write-Host "Git: $(try { git --version } catch { 'NOT FOUND' })"
```

**Note:** On Windows, use `python` (not `python3`).

## Step 1: Install Docker Desktop

Docker is the recommended approach on Windows — it avoids Python environment issues.

1. Download from https://docker.com/products/docker-desktop
2. Run installer. When prompted:
   - **Enable WSL 2** — check this box (it's required)
   - If WSL 2 isn't installed, Windows will prompt you to install it. Follow the prompts.
3. **Restart your computer** if prompted
4. Launch Docker Desktop from Start Menu
5. Wait 1-2 minutes for Docker to start (icon in system tray stops animating)
6. Verify in PowerShell: `docker --version`

**If Docker won't start:** Ensure virtualization is enabled in BIOS. Search "Turn Windows features on or off" > enable "Virtual Machine Platform" and "Windows Subsystem for Linux". Restart.

**Disk space:** Docker needs ~4GB minimum. Check with: `Get-PSDrive C | Select-Object Free`

## Step 2: Get Telegram API Credentials (~5 min)

This is the same on all platforms:

1. Open https://my.telegram.org in your browser
2. Enter your phone number (with country code: +7... for Russia, +1... for US)
3. Enter the verification code from Telegram
4. Click **"API development tools"**
5. If you don't have an app yet, create one:
   - App title: "Claude Code Integration"
   - Short name: something unique, letters only, 5-32 chars (e.g., "yourname_claude")
   - Platform: Desktop
6. Note your **api_id** (number) and **api_hash** (32-character string)

**Tip:** The short name must be globally unique. If "claudecode" is taken, try adding your initials or numbers.

## Step 3: Generate Session String (~3-5 min)

**IMPORTANT:** If you have 2FA enabled, have your password ready before starting.

Open a **new PowerShell window** (this is interactive — you'll type answers):

```powershell
docker run -it --rm bayramannakov/telegram-mcp:latest python setup_wizard.py
```

You'll be prompted for:
1. API ID (from Step 2)
2. API Hash (from Step 2)
3. Phone number (+7... format)
4. Verification code (check Telegram on all your devices)
5. 2FA password (if enabled)

**Result:** A long string of ~300-400 characters. Copy it carefully. This is your session string.

**Verification:** The string should be 300-400 characters of letters, numbers, +, /, =. If it's shorter, something went wrong.

## Step 4: Store Credentials

### Option A: Environment Variables (Simplest)

```powershell
# Set as user environment variables (persist across restarts)
[System.Environment]::SetEnvironmentVariable('TELEGRAM_API_ID', 'YOUR_API_ID', 'User')
[System.Environment]::SetEnvironmentVariable('TELEGRAM_API_HASH', 'YOUR_API_HASH', 'User')
[System.Environment]::SetEnvironmentVariable('TELEGRAM_SESSION_STRING', 'YOUR_SESSION_STRING', 'User')
```

**Close and reopen PowerShell** for variables to take effect.

Verify:
```powershell
echo $env:TELEGRAM_API_ID
```

### Option B: Secure Config File

```powershell
# Create directory
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\telegram-mcp"

# Create config file
@"
TELEGRAM_API_ID=YOUR_API_ID
TELEGRAM_API_HASH=YOUR_API_HASH
TELEGRAM_SESSION_STRING=YOUR_SESSION_STRING
"@ | Out-File -FilePath "$env:USERPROFILE\.config\telegram-mcp\.env" -Encoding UTF8

# Restrict access to current user only
$acl = Get-Acl "$env:USERPROFILE\.config\telegram-mcp\.env"
$acl.SetAccessRuleProtection($true, $false)
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    $env:USERNAME, "FullControl", "Allow"
)
$acl.SetAccessRule($rule)
Set-Acl "$env:USERPROFILE\.config\telegram-mcp\.env" $acl
```

## Step 5: Register MCP Server

### If using environment variables (Option A):

```powershell
claude mcp add telegram-mcp -s user -- docker run --rm -i -e TELEGRAM_API_ID -e TELEGRAM_API_HASH -e TELEGRAM_SESSION_STRING bayramannakov/telegram-mcp:latest
```

### If using config file (Option B):

Create a launcher script at `$env:USERPROFILE\.local\bin\telegram-mcp.bat`:

```batch
@echo off
for /f "usebackq tokens=1,2 delims==" %%a in ("%USERPROFILE%\.config\telegram-mcp\.env") do (
    set %%a=%%b
)
docker run --rm -i -e TELEGRAM_API_ID -e TELEGRAM_API_HASH -e TELEGRAM_SESSION_STRING bayramannakov/telegram-mcp:latest
```

Then register:
```powershell
claude mcp add telegram-mcp -s user -- "$env:USERPROFILE\.local\bin\telegram-mcp.bat"
```

Verify:
```powershell
claude mcp list
```

**Important:** Docker Desktop must be running every time you use Telegram in Claude Code.

## Step 6: Test Connection

1. **Close and reopen Claude Code** (restart required for MCP to load)
2. Ask Claude: "Show me my Telegram chats"
3. You should see a list of your chats

## Security: What You Must Know

### Session string is MORE sensitive than a password

If compromised, an attacker can:
- Read ALL your messages (every chat, group, channel)
- Send messages as you
- Download all files and photos
- Access your contacts
- Do all this **silently** with no notification to you

A password needs 2FA. A session string **bypasses 2FA entirely**.

### Data flow

- Your session string stays on your computer (local)
- When Claude summarizes or searches your messages, the **message content is sent to Anthropic's API** for processing
- This is the same as copy-pasting messages into Claude manually

### Best practices

1. Never share your session string
2. Use draft mode — always save_draft, never auto-send
3. Check active sessions: Telegram > Settings > Privacy > Active Sessions
4. Consider testing with a secondary account first

## Complete Revocation

To fully disconnect:

1. **Terminate session in Telegram:** Settings > Privacy > Active Sessions > Terminate All Other Sessions
2. **Remove environment variables:**
   ```powershell
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_API_ID', $null, 'User')
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_API_HASH', $null, 'User')
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_SESSION_STRING', $null, 'User')
   ```
3. **Remove MCP registration:** `claude mcp remove telegram-mcp`
4. **Delete config file (if used):** `Remove-Item -Recurse "$env:USERPROFILE\.config\telegram-mcp"`
5. **Delete Docker image:** `docker rmi bayramannakov/telegram-mcp:latest`

## Troubleshooting (Windows-Specific)

**"docker: command not found"**
Docker Desktop not in PATH. Try full path: `& "C:\Program Files\Docker\Docker\resources\bin\docker.exe" --version`

**Docker won't start**
- Enable virtualization: Search "Turn Windows features on or off" > enable "Virtual Machine Platform" + "Windows Subsystem for Linux"
- Restart computer
- Run `wsl --install` if WSL not present

**PowerShell execution policy error**
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**"python: command not found"**
Install via: `winget install Python.Python.3.12` or download from python.org. Note: on Windows use `python` not `python3`.

**Session expired**
Regenerate in PowerShell:
```powershell
docker run -it --rm bayramannakov/telegram-mcp:latest python setup_wizard.py
```
Then update your environment variable or config file with the new session string.

**MCP server not appearing**
```powershell
claude mcp remove telegram-mcp
claude mcp add telegram-mcp -s user -- docker run --rm -i -e TELEGRAM_API_ID -e TELEGRAM_API_HASH -e TELEGRAM_SESSION_STRING bayramannakov/telegram-mcp:latest
claude mcp list
```
Then restart Claude Code.
