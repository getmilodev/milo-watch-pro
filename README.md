# Milo Watch Pro

**Advanced security & health monitoring for OpenClaw — with historical trends, cost analysis, and automated fixes.**

[Free version](https://github.com/getmilodev/milo-watch) covers the basics. Pro goes deeper.

## What's in Pro?

Everything in the free tier, plus:

### Token Cost Analysis
Know exactly how much your workspace injection costs per month. Identifies the biggest files eating your budget and estimates costs across models (Sonnet, Opus, Flash).

### Session Bloat Detection
Finds oversized session files, calculates reclaimable space, flags files older than 7 days that are just burning disk.

### Memory System Health
Validates your memory architecture: checks MEMORY.md line count, session-state.md existence, daily note size caps. Catches memory bloat before it causes compaction issues.

### Network Exposure Scan
Audits listening ports, identifies services exposed to all interfaces, flags non-localhost bindings that shouldn't be public.

### SSH Security Audit
Checks password auth, root login, key-based auth settings. The basics that prevent 90% of SSH attacks.

### Historical Trends
Stores daily scan results in `memory/watch-history/`. After 2+ days, shows 7-day trend comparison so you can see if your security posture is improving or degrading.

### Automated Fix Commands
Every warning comes with a copy-paste fix command. No guessing, no docs hunting.

## Pricing

**$9/month** — cancel anytime.

[Subscribe →](https://buy.stripe.com/00w8wO5WdgTl1hg4CafIs06)

## Quick Start

1. Subscribe and get your license key
2. Clone the repo:
   ```bash
   git clone https://github.com/getmilodev/milo-watch-pro.git
   cp -r milo-watch-pro ~/.openclaw/skills/milo-watch-pro
   ```
3. Set your key:
   ```bash
   export MILO_WATCH_KEY="mwp_your_key"
   ```
4. Set up daily monitoring:
   ```bash
   openclaw cron add --name milo-watch-pro \
     --schedule "0 8 * * *" \
     --task "Run the milo-watch-pro skill. Store results and alert on CRITICAL issues."
   ```

## Free vs Pro

| Feature | Free | Pro |
|---------|------|-----|
| Core security checks (6) | ✅ | ✅ |
| Token cost analysis | ❌ | ✅ |
| Session bloat detection | ❌ | ✅ |
| Memory system health | ❌ | ✅ |
| Network exposure scan | ❌ | ✅ |
| SSH security audit | ❌ | ✅ |
| Historical trends | ❌ | ✅ |
| Automated fix commands | ❌ | ✅ |
| Priority support | ❌ | ✅ |

## Part of the Milo Security Suite

- [**milo-scan**](https://github.com/getmilodev/milo-scan) — One-command config audit (`npx milo-scan`)
- [**Milo Watch**](https://github.com/getmilodev/milo-watch) — Free daily monitoring
- [**Milo Shield**](https://github.com/getmilodev/milo-shield) — Auto-fix misconfigurations in real time
- [**Security Blog**](https://getmilo.dev/blog) — OpenClaw security guides and analysis
- [**All Services**](https://getmilo.dev/security) — Professional audits and done-for-you hardening

---

Built by [Milo](https://getmilo.dev) — AI security and automation for small businesses.