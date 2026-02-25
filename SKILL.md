# Milo Watch Pro — Advanced OpenClaw Security & Health Monitor

Daily automated security, health, and performance monitoring with historical trends, automated fixes, and cost analysis.

## License

This is a paid skill. Requires an active Milo Watch Pro subscription.
Subscribe: https://buy.stripe.com/00w8wO5WdgTl1hg4CafIs06

After subscribing, you'll receive a license key via email. Set it in your environment:
```bash
export MILO_WATCH_KEY="mwp_your_license_key"
```

## When to Use

- Daily cron job for continuous monitoring
- After configuration changes to validate security
- Before deploying to production
- When troubleshooting performance or cost issues

## Setup

### Daily Monitoring (Recommended)
```bash
openclaw cron add --name milo-watch-pro \
  --schedule "0 8 * * *" \
  --task "Run the milo-watch-pro skill. Store results in memory/watch-history/. Alert me if any CRITICAL issues found."
```

## Scan Protocol

Run ALL checks in order. Store results in `memory/watch-history/YYYY-MM-DD.json` for trend tracking.

### Phase 1: Core Security (same as free tier)

#### 1.1 Gateway Security
```bash
cat ~/.openclaw/openclaw.json 2>/dev/null | python3 -c "
import json, sys
try:
    config = json.load(sys.stdin)
    host = config.get('gateway', {}).get('host', '127.0.0.1')
    port = config.get('gateway', {}).get('port', 3000)
    print(f'HOST={host}')
    print(f'PORT={port}')
    if host in ['0.0.0.0', '']:
        print('STATUS=CRITICAL')
        print('DETAIL=Gateway bound to 0.0.0.0 — exposed to all network interfaces')
    else:
        print('STATUS=OK')
        print(f'DETAIL=Gateway bound to {host}')
except Exception as e:
    print(f'STATUS=ERROR')
    print(f'DETAIL={e}')
"
```

#### 1.2 Authentication
```bash
cat ~/.openclaw/openclaw.json 2>/dev/null | python3 -c "
import json, sys
try:
    config = json.load(sys.stdin)
    api_key = config.get('gateway', {}).get('apiKey', '')
    allowed_origins = config.get('gateway', {}).get('allowedOrigins', [])
    print(f'API_KEY={\"set\" if api_key else \"missing\"}')
    print(f'ORIGINS={len(allowed_origins)}')
    if not api_key:
        print('STATUS=WARNING')
        print('DETAIL=No API key — anyone with network access can connect')
    else:
        print('STATUS=OK')
        print('DETAIL=API key configured')
except Exception as e:
    print(f'STATUS=ERROR')
    print(f'DETAIL={e}')
"
```

#### 1.3 Exec Security
```bash
cat ~/.openclaw/openclaw.json 2>/dev/null | python3 -c "
import json, sys
try:
    config = json.load(sys.stdin)
    mode = config.get('tools', {}).get('exec', {}).get('security', 'full')
    elevated = config.get('tools', {}).get('exec', {}).get('elevated', False)
    print(f'MODE={mode}')
    print(f'ELEVATED={elevated}')
    if mode == 'full':
        print('STATUS=WARNING')
        print('DETAIL=Exec security \"full\" — unrestricted shell access')
    elif mode == 'deny':
        print('STATUS=OK')
        print('DETAIL=Exec denied — most secure')
    else:
        print('STATUS=OK')
        print(f'DETAIL=Exec mode: {mode}')
    if elevated:
        print('ELEVATED_STATUS=CRITICAL')
        print('ELEVATED_DETAIL=Elevated exec enabled — agent can run commands as root')
except Exception as e:
    print(f'STATUS=ERROR')
    print(f'DETAIL={e}')
"
```

#### 1.4 Version Check
```bash
INSTALLED=$(openclaw --version 2>/dev/null || echo "unknown")
LATEST=$(npm view openclaw version 2>/dev/null || echo "unknown")
echo "INSTALLED=$INSTALLED"
echo "LATEST=$LATEST"
if [ "$INSTALLED" != "$LATEST" ] && [ "$INSTALLED" != "unknown" ] && [ "$LATEST" != "unknown" ]; then
    echo "STATUS=WARNING"
    echo "DETAIL=Running $INSTALLED, latest is $LATEST"
else
    echo "STATUS=OK"
    echo "DETAIL=Running latest ($INSTALLED)"
fi
```

### Phase 2: Pro — Performance & Cost Analysis

#### 2.1 Token Cost Estimation
```bash
python3 << 'PYEOF'
import os, json, glob

workspace = os.path.expanduser("~/.openclaw")
workspace_dirs = [workspace + "/workspace"] + glob.glob(workspace + "/workspace-*")

total_boot_tokens = 0
file_details = []

for ws in workspace_dirs:
    if not os.path.isdir(ws):
        continue
    ws_name = os.path.basename(ws)
    for f in sorted(glob.glob(ws + "/*.md")):
        size = os.path.getsize(f)
        tokens = size // 4  # rough estimate
        total_boot_tokens += tokens
        file_details.append({
            "workspace": ws_name,
            "file": os.path.basename(f),
            "bytes": size,
            "est_tokens": tokens
        })

# Estimate daily cost at different rates
# Claude Sonnet: ~$3/1M input tokens, ~$15/1M output tokens
# Assume 50 messages/day, each injecting boot context
messages_per_day = 50
daily_input_tokens = total_boot_tokens * messages_per_day
monthly_input_tokens = daily_input_tokens * 30

# Cost estimates (input tokens only from workspace injection)
sonnet_monthly = (monthly_input_tokens / 1_000_000) * 3
opus_monthly = (monthly_input_tokens / 1_000_000) * 15
flash_monthly = (monthly_input_tokens / 1_000_000) * 0.15

print(f"TOTAL_BOOT_TOKENS={total_boot_tokens}")
print(f"FILES_INJECTED={len(file_details)}")
print(f"EST_MONTHLY_COST_SONNET=${sonnet_monthly:.2f}")
print(f"EST_MONTHLY_COST_OPUS=${opus_monthly:.2f}")
print(f"EST_MONTHLY_COST_FLASH=${flash_monthly:.2f}")
print()

# Top 5 biggest files
file_details.sort(key=lambda x: x["est_tokens"], reverse=True)
print("TOP_FILES:")
for f in file_details[:5]:
    print(f"  {f['workspace']}/{f['file']}: ~{f['est_tokens']} tokens ({f['bytes']} bytes)")

if total_boot_tokens > 15000:
    print("STATUS=WARNING")
    print(f"DETAIL=Boot injection is {total_boot_tokens} tokens — consider trimming large files")
elif total_boot_tokens > 8000:
    print("STATUS=INFO")
    print(f"DETAIL=Boot injection is {total_boot_tokens} tokens — moderate, monitor growth")
else:
    print("STATUS=OK")
    print(f"DETAIL=Boot injection is {total_boot_tokens} tokens — efficient")
PYEOF
```

#### 2.2 Session Bloat Analysis
```bash
python3 << 'PYEOF'
import os, glob, time

sessions_dir = os.path.expanduser("~/.openclaw/sessions")
if not os.path.isdir(sessions_dir):
    print("STATUS=OK")
    print("DETAIL=No sessions directory found")
    exit()

total_size = 0
file_count = 0
large_files = []
old_files = []
now = time.time()
seven_days = 7 * 86400

for root, dirs, files in os.walk(sessions_dir):
    for f in files:
        path = os.path.join(root, f)
        try:
            size = os.path.getsize(path)
            mtime = os.path.getmtime(path)
            total_size += size
            file_count += 1
            if size > 10 * 1024 * 1024:  # >10MB
                large_files.append((path, size))
            if now - mtime > seven_days:
                old_files.append((path, size, mtime))
        except:
            pass

print(f"TOTAL_SIZE={total_size // 1024}KB")
print(f"FILE_COUNT={file_count}")
print(f"LARGE_FILES={len(large_files)}")
print(f"OLD_FILES_7D={len(old_files)}")

reclaimable = sum(s for _, s, _ in old_files)
print(f"RECLAIMABLE={reclaimable // 1024}KB")

if large_files:
    print("STATUS=WARNING")
    print(f"DETAIL={len(large_files)} files over 10MB — consider cleanup")
elif reclaimable > 100 * 1024 * 1024:  # >100MB reclaimable
    print("STATUS=INFO")
    print(f"DETAIL={reclaimable // (1024*1024)}MB reclaimable from old sessions")
else:
    print("STATUS=OK")
    print(f"DETAIL={file_count} session files, {total_size // 1024}KB total")
PYEOF
```

#### 2.3 Memory System Health
```bash
python3 << 'PYEOF'
import os, glob

workspace = os.path.expanduser("~/.openclaw")
workspace_dirs = [workspace + "/workspace"] + glob.glob(workspace + "/workspace-*")

issues = []

for ws in workspace_dirs:
    if not os.path.isdir(ws):
        continue
    ws_name = os.path.basename(ws)
    
    # Check MEMORY.md size
    memory_md = os.path.join(ws, "MEMORY.md")
    if os.path.exists(memory_md):
        size = os.path.getsize(memory_md)
        lines = sum(1 for _ in open(memory_md))
        if lines > 90:
            issues.append(f"{ws_name}/MEMORY.md: {lines} lines (target: 50, max: 90)")
    else:
        issues.append(f"{ws_name}: No MEMORY.md found")
    
    # Check session-state.md exists
    ss = os.path.join(ws, "session-state.md")
    if not os.path.exists(ss):
        issues.append(f"{ws_name}: No session-state.md — compaction recovery at risk")
    
    # Check daily notes
    memory_dir = os.path.join(ws, "memory")
    if os.path.isdir(memory_dir):
        daily_notes = glob.glob(memory_dir + "/202?-??-??.md")
        if daily_notes:
            for note in daily_notes:
                size = os.path.getsize(note)
                if size > 5120:  # >5KB cap
                    issues.append(f"{ws_name}/{os.path.basename(note)}: {size//1024}KB (cap: 5KB)")

if issues:
    print("STATUS=WARNING")
    print("ISSUES:")
    for i in issues:
        print(f"  - {i}")
else:
    print("STATUS=OK")
    print("DETAIL=Memory system healthy across all workspaces")
PYEOF
```

### Phase 3: Pro — Network & External Exposure

#### 3.1 Listening Ports
```bash
echo "=== Listening Ports ==="
ss -tlnp 2>/dev/null | grep -E "LISTEN" | while read line; do
    port=$(echo "$line" | awk '{print $4}' | rev | cut -d: -f1 | rev)
    addr=$(echo "$line" | awk '{print $4}' | rev | cut -d: -f2- | rev)
    proc=$(echo "$line" | grep -oP 'users:\(\("\K[^"]+')
    echo "PORT=$port ADDR=$addr PROC=$proc"
    if echo "$addr" | grep -qE "^(0\.0\.0\.0|\*|::)$"; then
        echo "  STATUS=WARNING: Port $port open to all interfaces ($proc)"
    fi
done
```

#### 3.2 SSH Security
```bash
python3 << 'PYEOF'
import subprocess

checks = {
    "PasswordAuthentication": ("no", "Password auth should be disabled"),
    "PermitRootLogin": ("no", "Root login should be disabled"),
    "PubkeyAuthentication": ("yes", "Key-based auth should be enabled"),
}

try:
    result = subprocess.run(["sshd", "-T"], capture_output=True, text=True, timeout=5)
    config = {}
    for line in result.stdout.splitlines():
        parts = line.strip().split(None, 1)
        if len(parts) == 2:
            config[parts[0].lower()] = parts[1]
    
    for key, (expected, msg) in checks.items():
        actual = config.get(key.lower(), "unknown")
        if actual == expected:
            print(f"SSH_{key.upper()}=OK ({actual})")
        else:
            print(f"SSH_{key.upper()}=WARNING: {msg} (current: {actual})")
except Exception as e:
    print(f"SSH_CHECK=SKIPPED ({e})")
PYEOF
```

### Phase 4: Pro — Historical Trend Storage

After running all checks, store the results:

```bash
python3 << 'PYEOF'
import json, os, datetime

# Collect all results from the checks above into a structured object
# The agent should populate this from the actual check outputs
results = {
    "date": datetime.date.today().isoformat(),
    "timestamp": datetime.datetime.now().isoformat(),
    "checks": {
        "gateway": {"status": "PLACEHOLDER", "detail": ""},
        "auth": {"status": "PLACEHOLDER", "detail": ""},
        "exec": {"status": "PLACEHOLDER", "detail": ""},
        "version": {"status": "PLACEHOLDER", "detail": ""},
        "token_cost": {"status": "PLACEHOLDER", "boot_tokens": 0},
        "sessions": {"status": "PLACEHOLDER", "total_size_kb": 0},
        "memory": {"status": "PLACEHOLDER", "detail": ""},
        "ssh": {"status": "PLACEHOLDER", "detail": ""},
    },
    "score": 0,
    "total_checks": 8
}

# Save to history directory
history_dir = os.path.expanduser("~/.openclaw/memory/watch-history")
os.makedirs(history_dir, exist_ok=True)

filepath = os.path.join(history_dir, f"{results['date']}.json")
with open(filepath, 'w') as f:
    json.dump(results, f, indent=2)

print(f"Saved to {filepath}")

# Load last 7 days for trend comparison
import glob
history_files = sorted(glob.glob(history_dir + "/202?-??-??.json"))[-7:]
if len(history_files) > 1:
    print(f"\n=== 7-Day Trend ({len(history_files)} data points) ===")
    for hf in history_files:
        with open(hf) as f:
            h = json.load(f)
        print(f"  {h['date']}: Score {h['score']}/{h['total_checks']}")
else:
    print("\nFirst scan — trends will appear after 2+ days of data")
PYEOF
```

**IMPORTANT:** Replace the PLACEHOLDER values with actual check results before saving. The agent should parse the output of each check and populate the correct status/detail values.

## Report Format (Pro)

```
🔒 Milo Watch Pro — Security & Health Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🛡️ Gateway Security: [STATUS]
   [detail]

🔑 Authentication: [STATUS]
   [detail]

⚡ Exec Security: [STATUS]
   [detail]

📦 Version: [STATUS]
   [detail]

💰 Token Cost: [STATUS]
   Boot injection: ~X tokens
   Est. monthly (Sonnet): $X.XX
   Top file: [filename] (~X tokens)

💾 Sessions: [STATUS]
   [detail]
   Reclaimable: X KB from old sessions

🧠 Memory Health: [STATUS]
   [detail]

🌐 Network: [STATUS]
   [ports and exposure detail]

🔐 SSH: [STATUS]
   [detail]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Score: X/8 checks passed

📈 7-Day Trend:
   [date]: X/8  [date]: X/8  [date]: X/8
   [trend arrow and summary]

[Recommendations if any issues found]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Powered by Milo Watch Pro — getmilo.dev
Manage subscription: https://billing.stripe.com/p/login/xxx
```

## Automated Fix Suggestions

When issues are detected, provide copy-paste fix commands:

| Issue | Fix Command |
|-------|-------------|
| Gateway on 0.0.0.0 | `openclaw config set gateway.host 127.0.0.1` |
| No API key | `openclaw config set gateway.apiKey $(openssl rand -hex 32)` |
| Exec full mode | `openclaw config set tools.exec.security allowlist` |
| Outdated version | `npm update -g openclaw` |
| Session bloat >500MB | `find ~/.openclaw/sessions -mtime +7 -delete` |
| MEMORY.md >90 lines | Summarize and archive old entries |
| Password SSH enabled | `sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && sudo systemctl reload sshd` |

## Free vs Pro Comparison

| Feature | Free | Pro ($9/mo) |
|---------|------|-------------|
| Gateway security check | ✅ | ✅ |
| Auth check | ✅ | ✅ |
| Exec security check | ✅ | ✅ |
| Version check | ✅ | ✅ |
| Disk & session check | ✅ | ✅ |
| Workspace overhead | ✅ | ✅ |
| Token cost analysis | ❌ | ✅ |
| Session bloat analysis | ❌ | ✅ |
| Memory system health | ❌ | ✅ |
| Network exposure scan | ❌ | ✅ |
| SSH security audit | ❌ | ✅ |
| Historical trends | ❌ | ✅ |
| Automated fix commands | ❌ | ✅ |
| Priority support | ❌ | ✅ |

## Subscribe

**$9/month** — Cancel anytime.

→ https://buy.stripe.com/00w8wO5WdgTl1hg4CafIs06

## Support

- GitHub Issues: https://github.com/getmilodev/milo-watch/issues
- Email: milo@getmilo.dev

---

Built by [Milo](https://getmilo.dev) — autonomous AI security for OpenClaw.
