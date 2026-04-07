# Claude Code Rollback and Tracking Data Cleanup Guide

> Quick reference for switching Claude Code versions on macOS and cleaning local data.

## Scope

Use this guide when a version has been rolled back, when pinning a version, when cleaning tracking data, or when keeping configuration while doing a light cleanup.

---

## 1. Rollback and Version Switching

### 1. Background

Claude Code was rolled back from `v2.1.88` to `v2.1.87` because the npm-side `latest` tag was reverted.

### 2. Why It Rolled Back

#### npm Registry Status

| dist-tag | Version |
|----------|------|
| `stable` | `2.1.81` |
| `latest` | `2.1.87` |
| `next` | `2.1.89` |

- `v2.1.88` was removed from npm
- the `latest` tag moved from `2.1.88` back to `2.1.87`

#### Timeline

| Time | Event |
|------|------|
| Mar 28 19:25 | Auto-update downloaded `v2.1.87` |
| Mar 30 18:31 | Auto-update downloaded `v2.1.88` |
| Mar 30 ~ Mar 31 | Anthropic removed `v2.1.88` from npm and moved the `latest` tag back to `v2.1.87` |
| Mar 31 05:59 | The updater checked the registry, found `latest` was `2.1.87`, and switched the symlink back |

### 3. Auto-Update Mechanism

#### Core Architecture

```bash
~/.local/bin/claude  →  symlink  →  ~/.local/share/claude/versions/{version}
```

- Package name: `@anthropic-ai/claude-code`
- Binary storage: `~/.local/share/claude/versions/`
- Entry symlink: `~/.local/bin/claude`

#### AutoUpdater Workflow

- Check the npm registry `latest` tag on startup
- Download and update the `symlink` when the version differs
- So a server-side tag rollback is effectively a rollback

#### Key Findings

- The binary is a Bun-compiled Mach-O arm64 executable
- It contains identifiers such as `auto_updater_disabled`, `AutoUpdater`, and `autoUpdaterStatus`
- Startup telemetry reports the `auto_updater_disabled` state
- Concurrent updates are protected by a mutex, with messages like `Another instance is currently performing an update`

### 4. The Correct Way to Disable Auto-Updates

Reverse-engineering confirmed that the auto-update disable logic works like this:

```javascript
// Simplified logic
function getAutoUpdaterDisabledReason() {
  if (process.env.DISABLE_AUTOUPDATER) return { type: "env" };
  if (config.autoUpdates === false) return { type: "config" };
  return null;
}
```

#### Gotcha

`autoUpdaterDisabled: true` is the wrong key, so setting it has no effect. The updater can still race at startup and switch the symlink back to the version pointed to by `latest`.

#### Method A: Environment Variable

```bash
# Persist in ~/.zshrc
echo 'export DISABLE_AUTOUPDATER=1' >> ~/.zshrc
source ~/.zshrc
```

#### Method B: settings.json

Edit `~/.claude/settings.json`; the correct key is `autoUpdates`:

```json
"autoUpdates": false
```

> Note: if you are using a native installation and also set `autoUpdatesProtectedForNative: true`, then `autoUpdates: false` may be overridden. In that case, the environment-variable method is the only reliable option.

### 5. Solution: Switch Back to a Specific Version

Pin to `v2.1.88`:

#### Step 1: Disable Auto-Updates

```bash
# Enable first
export DISABLE_AUTOUPDATER=1
```

#### Step 2: Switch the Symlink

```bash
ln -sf ~/.local/share/claude/versions/2.1.88 ~/.local/bin/claude
```

#### Step 3: Verify

```bash
claude --version
# Expected: 2.1.88
```

#### Step 4: Use It

```bash
claude --dangerously-skip-permissions
```

### 6. Re-Enable Auto-Updates

```bash
# 1. Remove DISABLE_AUTOUPDATER=1 from ~/.zshrc
# 2. Remove "autoUpdates": false if used
# 3. Restart terminal
```

### 7. Other Ways to Switch Versions

```bash
# Switch to next channel (v2.1.89)
claude update --channel next

# Switch to stable channel (v2.1.81)
claude update --channel stable

# List locally installed versions
ls ~/.local/share/claude/versions/

# Manually switch to any local version
ln -sf ~/.local/share/claude/versions/<version> ~/.local/bin/claude
```

### 8. Notes

- `v2.1.88` was removed from npm by Anthropic and may contain known issues
- After disabling auto-updates, you will not receive security fixes, so remember to check versions manually from time to time
- Any locally left behind `v2.1.88` binary will not be cleaned up automatically

---

## 2. Tracking Data Cleanup and Privacy Reset

### 1. What Does Claude Code Track?

Claude Code does not collect hardware fingerprints, but it does store local identifiers, caches, and session traces.

| Tracking ID | Persistence | Storage Location | Notes |
|----------|--------|----------|------|
| `userID` | Permanent until manually deleted | `~/.claude.json` | Randomly generated 64-bit hexadecimal string, the cross-session tracking key |
| `anonymousId` | Permanent | `~/.claude.json` | Fallback tracking ID in the form `claudecode.v1.<uuid>` |
| `accountUuid` | Permanent, tied to the account | `~/.claude.json` → `oauthAccount` | Direct identity linkage after OAuth login |
| `emailAddress` | Permanent | `~/.claude.json` → `oauthAccount` | Login email |
| `rh` (repo hash) | Per repository | Every API request header | First 16 characters of the SHA256 of the git remote URL |
| Statsig Stable ID | Permanent | `~/.claude/statsig/` | Device identifier used by the feature-flag system |

#### Data Flow

- Anthropic 1P → `/api/event_logging/batch`, including full environment data and auth
- Datadog → `https://http-intake.logs.us5.datadoghq.com`, allowlisted events, already redacted
- OTLP → user-configured endpoint, disabled by default

### 2. Level 1: Reset Device Identifiers

Reset the device identifiers first.

```bash
# Check IDs
grep -E '"userID"|"anonymousId"|"firstStartTime"|"claudeCodeFirstTokenDate"' ~/.claude.json
```

```bash
# Remove IDs, keep config
python3 -c "
import json, os
p = os.path.expanduser('~/.claude.json')
with open(p, 'r') as f:
    d = json.load(f)
removed = []
for k in ['userID', 'anonymousId', 'firstStartTime', 'claudeCodeFirstTokenDate']:
    if k in d:
        removed.append(k)
        del d[k]
with open(p, 'w') as f:
    json.dump(d, f, indent=2)
print(f'Removed: {removed}')
print('Claude Code will generate a new userID on next launch')
"
```

**Effect:** A new `userID` will be generated on the next launch.

### 3. Level 2: Clear Telemetry and Analytics Data

```bash
# Unsent analytics events, including environment and session data
rm -rf ~/.claude/telemetry/

# Statsig/GrowthBook feature flag cache, including stable_id
rm -rf ~/.claude/statsig/

# Statistics cache
rm -f ~/.claude/stats-cache.json
```

**Effect:** Removes local telemetry and feature-flag caches.

### 4. Level 3: Clear Sessions and History

```bash
# Full command history, including every prompt you typed
rm -f ~/.claude/history.jsonl

# Session snapshots
rm -rf ~/.claude/sessions/

# Hash cache for large paste payloads
rm -rf ~/.claude/paste-cache/

# Shell environment snapshots
rm -rf ~/.claude/shell-snapshots/

# Session environment variables
rm -rf ~/.claude/session-env/

# File edit history
rm -rf ~/.claude/file-history/

# Debug logs
rm -rf ~/.claude/debug/
```

**Effect:** Clears local session traces.

### 5. Level 4: Remove OAuth Account Association

```bash
# Check linked account
python3 -c "
import json, os
with open(os.path.expanduser('~/.claude.json')) as f:
    d = json.load(f)
oa = d.get('oauthAccount', {})
print(f\"Account UUID: {oa.get('accountUuid', 'none')}\")
print(f\"Email: {oa.get('emailAddress', 'none')}\")
"
```

```bash
# Delete Keychain tokens
security delete-generic-password -s "claude-code" 2>/dev/null
security delete-generic-password -s "claude-code-credentials" 2>/dev/null

# Remove account caches
python3 -c "
import json, os
p = os.path.expanduser('~/.claude.json')
with open(p, 'r') as f:
    d = json.load(f)
removed = []
for k in ['oauthAccount', 's1mAccessCache', 'groveConfigCache',
          'passesEligibilityCache', 'clientDataCache',
          'cachedExtraUsageDisabledReason', 'githubRepoPaths']:
    if k in d:
        removed.append(k)
        del d[k]
with open(p, 'w') as f:
    json.dump(d, f, indent=2)
print(f'Removed: {removed}')
"
```

**Effect:** Breaks the local account linkage.

### 6. Level 5: Full Reset

```bash
# 1. Back up config
mkdir -p ~/Desktop/claude-backup
cp ~/.claude/CLAUDE.md ~/Desktop/claude-backup/ 2>/dev/null
cp ~/.claude/settings.json ~/Desktop/claude-backup/ 2>/dev/null
cp -r ~/.claude/skills ~/Desktop/claude-backup/ 2>/dev/null
cp -r ~/.claude/hooks ~/Desktop/claude-backup/ 2>/dev/null
echo "Backed up to ~/Desktop/claude-backup/"

# 2. Delete Claude Code data
rm -rf ~/.claude/
rm -f ~/.claude.json

# 3. Clear Keychain tokens
security delete-generic-password -s "claude-code" 2>/dev/null
security delete-generic-password -s "claude-code-credentials" 2>/dev/null

echo "Reset complete. Claude Code will reinitialize on next launch."
```

**Effect:** Equivalent to a fresh install.

### 7. Prevent Future Tracking

#### Method 1: Disable Non-Essential Network Traffic

Add the following to `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
```

**Effect:** Disables analytics, feature-flag fetches, and quota prechecks.

#### Method 2: Use a Third-Party Cloud Backend

When using Bedrock or Vertex, analytics is skipped.

```bash
# Use AWS Bedrock
export CLAUDE_CODE_USE_BEDROCK=1

# Or use Google Vertex AI
export CLAUDE_CODE_USE_VERTEX=1
```

#### Method 3: Periodic Automatic Cleanup

You can create `~/claude-privacy-clean.sh`:

```bash
#!/bin/bash
# Clean tracking data

rm -rf ~/.claude/telemetry/
rm -rf ~/.claude/statsig/
rm -f ~/.claude/stats-cache.json
rm -f ~/.claude/history.jsonl
rm -rf ~/.claude/sessions/
rm -rf ~/.claude/paste-cache/
rm -rf ~/.claude/shell-snapshots/
rm -rf ~/.claude/session-env/
rm -rf ~/.claude/debug/

# Reset device ID
python3 -c "
import json, os
p = os.path.expanduser('~/.claude.json')
if os.path.exists(p):
    with open(p, 'r') as f:
        d = json.load(f)
    for k in ['userID', 'anonymousId', 'firstStartTime', 'claudeCodeFirstTokenDate']:
        d.pop(k, None)
    with open(p, 'w') as f:
        json.dump(d, f, indent=2)
"

echo "[$(date)] Claude Code tracking data cleaned"
```

```bash
# Make it executable
chmod +x ~/claude-privacy-clean.sh

# Optional: run it daily via crontab
# crontab -e
# 0 3 * * * ~/claude-privacy-clean.sh >> ~/claude-clean.log 2>&1
```

### 8. Comparison of Cleanup Levels

| Operation | Effect on Anthropic | Impact on You |
|------|---------------------|------------|
| Reset Device ID | Historical usage data can no longer be linked | A new ID is generated automatically |
| Clear telemetry | Local caches will not be resent | No visible impact |
| Clear history | — | Lose command history and `ctrl+r` search |
| Clear OAuth | Breaks account linkage | Need to sign in again |
| Full reset | Equivalent to a new user | Lose all custom configuration |
| Disable non-essential traffic | No more analytics data is received | Feature flags may stop updating |
| Use Bedrock/Vertex | Analytics code does not run | Cloud provider credentials are required |

### 9. Data File Reference

| File/Directory | Tracking Data Included |
|-----------|----------------|
| `~/.claude.json` | `userID`, `anonymousId`, `oauthAccount`, first-use time, GitHub repository mappings |
| `~/.claude/telemetry/` | Unsent 1P analytics events, including full `EnvContext` |
| `~/.claude/statsig/` | Statsig `stable_id`, feature-flag cache |
| `~/.claude/stats-cache.json` | Statistics cache |
| `~/.claude/history.jsonl` | Full command history, including every prompt you typed |
| `~/.claude/sessions/` | Session metadata |
| `~/.claude/paste-cache/` | Hash cache for pasted content |
| `~/.claude/file-history/` | Record of file edits made by Claude |
| `~/.claude/debug/` | Debug logs |
| `~/.claude/shell-snapshots/` | Shell environment snapshots |
| macOS Keychain → `claude-code` | OAuth access/refresh tokens |

---

## 3. Recommended Order

Disable auto-updates first, switch versions second, clean data third, and only then decide whether to re-enable updates.