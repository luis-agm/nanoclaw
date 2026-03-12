---
name: uninstall
description: Uninstall NanoClaw from this host. Stops and removes the background service, cleans up config files, and optionally removes build artifacts, credentials, and the container image. Use when user wants to remove NanoClaw, clean up after a failed setup, or start fresh.
---

# NanoClaw Uninstall

Remove NanoClaw from the host cleanly and safely. Always confirm with the user before deleting anything. Never delete without explicit approval.

**Principle:** Ask once, act completely. Group related removals into clear choices. Show exactly what will be deleted before doing it. Prefer targeted removals over blanket `rm -rf`.

**UX Note:** Use `AskUserQuestion` for all decisions.

## 1. Detect current state

Run these checks in parallel to understand what's installed:

```bash
# Service state
systemctl --user is-active nanoclaw 2>/dev/null || launchctl list 2>/dev/null | grep nanoclaw || echo "no_service"

# Service unit files
ls ~/.config/systemd/user/nanoclaw.service 2>/dev/null || \
  ls ~/Library/LaunchAgents/com.nanoclaw.plist 2>/dev/null || \
  ls /etc/systemd/system/nanoclaw.service 2>/dev/null || echo "no_unit"

# Config dir
ls ~/.config/nanoclaw/ 2>/dev/null || echo "no_config"

# Docker image
docker images nanoclaw-agent:latest --format "{{.Repository}}:{{.Tag}}" 2>/dev/null || echo "no_image"

# Apple Container image
container images 2>/dev/null | grep nanoclaw-agent || echo "no_ac_image"

# Store dir (WhatsApp auth + DB)
ls store/auth/ 2>/dev/null | wc -l || echo "0"
```

Summarise findings to the user briefly before asking questions.

## 2. Confirm scope

AskUserQuestion (multiSelect): What would you like to remove?

- **Service** — Stop and unregister the background service (recommended)
- **Config** — Delete `~/.config/nanoclaw/` (mount allowlist)
- **Container image** — Remove the `nanoclaw-agent:latest` Docker/Apple Container image
- **Build artifacts** — Delete `node_modules/`, `dist/`, `logs/`
- **Credentials & data** — Delete `.env`, `store/` (WhatsApp auth, message DB). **Irreversible.**

Always recommend "Service" and "Config" as the safe baseline. Flag "Credentials & data" as irreversible.

## 3. Execute removals in order

### 3a. Service removal

**Linux (systemd, non-root):**
```bash
systemctl --user stop nanoclaw 2>/dev/null || true
systemctl --user disable nanoclaw 2>/dev/null || true
rm -f ~/.config/systemd/user/nanoclaw.service
systemctl --user daemon-reload
```

**Linux (systemd, root):**
```bash
systemctl stop nanoclaw 2>/dev/null || true
systemctl disable nanoclaw 2>/dev/null || true
rm -f /etc/systemd/system/nanoclaw.service
systemctl daemon-reload
```

**macOS (launchd):**
```bash
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist 2>/dev/null || true
rm -f ~/Library/LaunchAgents/com.nanoclaw.plist
```

**WSL / nohup fallback:**
```bash
if [ -f nanoclaw.pid ]; then
  kill "$(cat nanoclaw.pid)" 2>/dev/null || true
  rm -f nanoclaw.pid
fi
rm -f start-nanoclaw.sh
```

After removal, verify the service is gone:
- Linux: `systemctl --user list-unit-files | grep nanoclaw` should return nothing
- macOS: `launchctl list | grep nanoclaw` should return nothing

### 3b. Config removal

```bash
rm -rf ~/.config/nanoclaw/
```

### 3c. Container image removal

Docker:
```bash
docker rmi nanoclaw-agent:latest 2>/dev/null || true
```

Apple Container:
```bash
container rmi nanoclaw-agent:latest 2>/dev/null || true
```

### 3d. Build artifacts

```bash
rm -rf node_modules/ dist/ logs/
```

### 3e. Credentials & data (only if explicitly selected)

Before running, print a final warning: "This will permanently delete your WhatsApp session credentials and all message history. This cannot be undone."

AskUserQuestion: Are you sure you want to delete `.env` and `store/`? This is irreversible.
- Yes, delete them
- No, skip this

If confirmed:
```bash
rm -f .env
rm -rf store/
```

## 4. Verify and report

After all selected steps complete, run a quick check:

```bash
# Service gone?
systemctl --user is-active nanoclaw 2>/dev/null; launchctl list 2>/dev/null | grep nanoclaw || echo "service_removed"

# Config gone?
ls ~/.config/nanoclaw/ 2>/dev/null || echo "config_removed"
```

Report what was removed and what was skipped. If the user kept `node_modules/` and `dist/`, remind them the project can be re-setup with `/setup`.

## Notes

- If the service unit file doesn't exist but the service shows as active, the process may be an orphan from a previous run. Kill it with `pkill -f "dist/index.js"`.
- If Docker is not running, skip the image removal step (Docker not needed to remove the service).
- Never delete the project directory itself — that's the user's repo.
