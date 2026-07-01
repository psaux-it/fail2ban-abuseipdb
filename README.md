# Fail2Ban AbuseIPDB Reporter (Enhanced)
 
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Shell](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnu-bash&logoColor=white)](#)
[![Fail2Ban](https://img.shields.io/badge/Fail2Ban-Action-blue)](#)
 
An enhanced, production-ready **Fail2Ban → AbuseIPDB** integration. It reports banned IPs to [AbuseIPDB](https://www.abuseipdb.com/) through a hardened Bash action script that runs **independently of Fail2Ban's internal state**, using its own SQLite database, a bounded concurrent-worker pool, per-IP locking, and cooldown tracking to stay within AbuseIPDB's rate limits.
 
Unlike the stock Fail2Ban `abuseipdb.conf` action, this script is built to survive restarts, bulk-ban bursts, and concurrent jail triggers without duplicate reports, race conditions, or silent API failures.
 
---
 
## ✨ Key Features
 
- **Isolated tracking database** — a dedicated SQLite database (WAL mode) tracks every banned/reported IP independently of Fail2Ban's own ban state, so reporting stays accurate even if Fail2Ban's internal ticket state drifts over time.
- **Dual API calls, zero quota waste** — uses `/v2/check` before `/v2/report` so IPs already listed on AbuseIPDB aren't re-reported unnecessarily. These endpoints have separate daily quotas, so checking doesn't cost report budget.
- **15-minute cooldown awareness** — tracks AbuseIPDB's per-IP report cooldown locally to avoid redundant calls and `HTTP 429` errors on repeat offenders.
- **Bounded worker pool (`MAX_WORKERS`)** — a `flock`-based semaphore caps concurrent API workers, preventing rate-limit bursts and SQLite write contention during mass-ban events. Bans are always queued, never dropped.
- **Per-IP dedup locking** — prevents the same IP from being processed twice in parallel by concurrent Fail2Ban triggers.
- **Safe restart handling** — choose between relying on Fail2Ban's `norestored` ticket replay, or fully bypassing Fail2Ban (`BYPASS_FAIL2BAN=1`) and trusting the local SQLite database as the single source of truth on service restart.
- **Self-healing initialization** — `actionstart` verifies DB integrity and schema completeness on every start, auto-migrates older schemas (e.g. adds `last_reported_at`), and blocks `actionban` if initialization fails.
- **Leak-safe custom comments** — enforces the use of a sanitized `tp_comment` per jail instead of Fail2Ban's raw `<matches>`, which can otherwise leak usernames, request paths, or partial credentials into a **public** AbuseIPDB report.
- **Detailed logging** — every check, report, skip, and error is logged with timestamps to a dedicated log file for auditing.
---
 
## 🧠 How It Works
 
```
Fail2Ban ban event
        │
        ▼
 actionban triggered ──► per-IP flock (dedup) ──► worker pool slot (MAX_WORKERS)
        │
        ▼
 check local SQLite DB ──► check AbuseIPDB (/v2/check)
        │
        ├── already reported & still listed ─► skip
        ├── in 15-min cooldown ─────────────► skip
        └── needs reporting ────────────────► insert/update DB ──► report (/v2/report)
                                                                        │
                                                            success ────┴──── failure → rollback DB entry
```
 
On `actionstart`, the script provisions the runtime directories, verifies dependencies (`curl`, `jq`, `sqlite3`), creates/migrates the SQLite schema, and writes a completion lock so redundant Fail2Ban restarts don't re-run initialization. If initialization ever fails, `actionban` is blocked outright rather than reporting against a broken environment.
 
---
 
## 📦 Requirements
 
| Dependency | Purpose |
|---|---|
| [Fail2Ban](https://github.com/fail2ban/fail2ban) | Intrusion prevention / ban orchestration |
| `bash` | Script runtime |
| `curl` | AbuseIPDB API requests |
| `jq` | JSON response parsing |
| `sqlite3` | Local persistent ban/report tracking |
| An [AbuseIPDB](https://www.abuseipdb.com/register) account & API key | Reporting endpoint |
 
---
 
## 🚀 Installation
 
1. **Copy the action script**
```bash
   sudo cp fail2ban-abuseipdb.sh /etc/fail2ban/action.d/fail2ban-abuseipdb.sh
   sudo chmod +x /etc/fail2ban/action.d/fail2ban-abuseipdb.sh
```
 
2. **Copy the action definition**
```bash
   sudo cp abuseipdb.conf /etc/fail2ban/action.d/abuseipdb.conf
   sudo cp abuseipdb.local /etc/fail2ban/action.d/abuseipdb.local
```
 
3. **Edit `abuseipdb.local`** — uncomment and configure:
```ini
   [Init]
   abuseipdb_apikey = YOUR_API_KEY
 
   SQLITE_DB = "/var/lib/fail2ban/abuseipdb/fail2ban_abuseipdb"
   LOG_FILE = "/var/log/abuseipdb/abuseipdb.log"
   BYPASS_FAIL2BAN = 0
   MAX_WORKERS = 10
 
   actionstart = nohup /etc/fail2ban/action.d/fail2ban-abuseipdb.sh \
       "--actionstart" "<SQLITE_DB>" "<LOG_FILE>" &
 
   actionban = /etc/fail2ban/action.d/fail2ban-abuseipdb.sh \
       "<abuseipdb_apikey>" "<matches>" "<ip>" "<abuseipdb_category>" "<bantime>" "<restored>" "<BYPASS_FAIL2BAN>" "<SQLITE_DB>" "<LOG_FILE>" "<MAX_WORKERS>"
```
 
4. **Wire it into a jail** in `jail.local` — always define a custom `tp_comment` to avoid leaking sensitive log data into a public report:
```ini
   [nginx-botsearch]
   enabled    = true
   logpath    = /var/log/nginx/*.log
   port       = http,https
   backend    = polling
   tp_comment = Fail2Ban - NGINX bad requests 400-401-403-404-444, high level vulnerability scanning
   maxretry   = 3
   findtime   = 1d
   bantime    = 7200
   action     = %(action_mwl)s
                %(action_abuseipdb)s[matches="%(tp_comment)s", abuseipdb_apikey="YOUR_API_KEY", abuseipdb_category="21,15", bantime="%(bantime)s"]
```
 
5. **Restart Fail2Ban**
```bash
   sudo systemctl restart fail2ban
```
 
6. **Verify** by tailing the log:
```bash
   tail -f /var/log/abuseipdb/abuseipdb.log
```
 
> ⚠️ **Ensure `norestored = 0`** (Fail2Ban's default). Restart/duplicate-report handling is deliberately controlled at the script level via `BYPASS_FAIL2BAN`, not through Fail2Ban's own `norestored` option.
 
---
 
## ⚙️ Configuration Reference
 
| Setting | Description | Default |
|---|---|---|
| `abuseipdb_apikey` | Your AbuseIPDB API key | — |
| `SQLITE_DB` | Path to the isolated tracking database | `/var/lib/fail2ban/abuseipdb/fail2ban_abuseipdb` |
| `LOG_FILE` | Path to the action log | `/var/log/abuseipdb/abuseipdb.log` |
| `BYPASS_FAIL2BAN` | `0` = trust Fail2Ban's restored-ticket state · `1` = fully isolate and decide reporting from the SQLite DB alone | `0` |
| `MAX_WORKERS` | Max concurrent API workers (each processes 2 calls: check + report) | `10` |
 
**Tuning `MAX_WORKERS`:** watch `abuseipdb.log` after a burst of bans —
- Frequent `"pool may be saturated"` warnings → raise `MAX_WORKERS`
- `HTTP 429` errors from AbuseIPDB → lower `MAX_WORKERS`
**Note on `BYPASS_FAIL2BAN=1`:** on every Fail2Ban restart, all restored IPs are replayed through the script. `MAX_WORKERS` bounds concurrent calls, but a very large restore count will still take proportionally longer to fully process.
 
### Script Arguments
 
| # | Argument | Source | Description |
|---|---|---|---|
| 1 | `APIKEY` | Fail2Ban jail | AbuseIPDB API key |
| 2 | `COMMENT` | Fail2Ban jail (`tp_comment`) | Sanitized report comment |
| 3 | `IP` | Fail2Ban jail | Offending IP address |
| 4 | `CATEGORIES` | Fail2Ban jail | AbuseIPDB abuse category codes |
| 5 | `BANTIME` | Fail2Ban jail | Ban duration |
| 6 | `RESTORED` | Fail2Ban `<restored>` | Whether this is a restored ticket |
| 7 | `BYPASS_FAIL2BAN` | `abuseipdb.local` | Restart isolation toggle |
| 8 | `SQLITE_DB` | `abuseipdb.local` | Tracking database path |
| 9 | `LOG_FILE` | `abuseipdb.local` | Log file path |
| 10 | `MAX_WORKERS` | `abuseipdb.local` | Worker pool size |
 
---
 
## 🔒 Security & Privacy Notice
 
AbuseIPDB reports are **public**. Fail2Ban's default `<matches>` filter value can include raw log lines — usernames, URLs, request paths, even partial credentials — verbatim. **Always** set a custom `tp_comment` per jail and reference it via `matches="%(tp_comment)s"` in the action line. Never pass Fail2Ban's raw `<matches>` directly to this script's `COMMENT` argument unless your logs have been manually verified to contain nothing sensitive.
 
---
 
## 🗃️ Database Schema
 
```sql
CREATE TABLE banned_ips (
    ip                TEXT PRIMARY KEY,
    bantime           INTEGER,
    last_reported_at  INTEGER DEFAULT 0
);
CREATE INDEX idx_ip ON banned_ips(ip);
```
 
The schema is created and migrated automatically on `actionstart`; existing installs are upgraded in place (e.g. `last_reported_at` is added via `ALTER TABLE` if missing).
 
---
 
## 🧪 Manual Testing
 
```bash
/etc/fail2ban/action.d/fail2ban-abuseipdb.sh \
  "your_api_key" "Failed SSH login attempts" "192.0.2.1" "18" "600"
```
 
---
 
## 🐛 Troubleshooting
 
| Symptom | Likely cause | Fix |
|---|---|---|
| `Initialization already failed! (actionstart)` on every ban | `actionstart` failed (missing dependency, permission error) | Check `abuseipdb.log` for the root cause; fix and restart Fail2Ban |
| Repeated `HTTP 429` | `MAX_WORKERS` too high, or reporting the same IP faster than the 15-min cooldown | Lower `MAX_WORKERS`; cooldown logic should self-correct |
| IP reported every restart | `BYPASS_FAIL2BAN=1` with a very large restore backlog | Expected behavior — tune `MAX_WORKERS`, or switch to `BYPASS_FAIL2BAN=0` |
| `sqlite3`/`jq`/`curl` not found | Missing dependency | Install via your package manager (e.g. `apt install sqlite3 jq curl`) |
 
---
 
## 🤝 Contributing
 
Issues and pull requests are welcome. Please include relevant `abuseipdb.log` excerpts when reporting bugs, and test against a non-production Fail2Ban instance before submitting changes to the locking/concurrency logic.
 
---
 
## 📄 License
 
Released under the [MIT License](LICENSE).
 
## 🙏 Acknowledgments
 
Original implementation and design by **Hasan ÇALIŞIR** ([@hsntgm](https://github.com/hsntgm)). This repository builds on that work — please retain attribution if you fork or redistribute.
