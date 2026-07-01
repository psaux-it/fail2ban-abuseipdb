# Fail2Ban AbuseIPDB Reporter

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Shell](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnu-bash&logoColor=white)](#)
[![Fail2Ban](https://img.shields.io/badge/Fail2Ban-Action-blue)](#)
[![Upstream PR](https://img.shields.io/badge/Fail2Ban%20PR-%233948-orange?logo=github)](https://github.com/fail2ban/fail2ban/pull/3948)

An enhanced, production-ready **Fail2Ban → AbuseIPDB** integration. It reports banned IPs to [AbuseIPDB](https://www.abuseipdb.com/) through a hardened Bash action script that runs **independently of Fail2Ban's internal state**, using its own SQLite database, a bounded concurrent-worker pool, per-IP locking, and cooldown tracking to stay within AbuseIPDB's rate limits.

Unlike the stock Fail2Ban `abuseipdb.conf` action, this script is built to survive restarts, bulk-ban bursts, and concurrent jail triggers without duplicate reports, race conditions, or silent API failures — and instead of passing Fail2Ban's raw log matches straight into a **public** report, it lets you set a custom, per-jail comment (`tp_comment`), so you control exactly what gets published and avoid leaking sensitive info.

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

<img width="2100" height="1180" alt="Image" src="https://github.com/user-attachments/assets/f5eca6ad-c508-4423-80c3-cc1fe140754e" />

Every ban follows the same path: `actionban` fires → the IP acquires a **per-IP dedup lock** and a **worker-pool slot** (see Worker Pool Semaphore) → the script checks its local SQLite DB and AbuseIPDB's `/v2/check` → and based on that, either **skips** (already reported & still listed, or within the 15-minute cooldown) or **reports** the IP via `/v2/report`, updating the local DB on success or rolling back the DB row on failure.

On `actionstart`, the script provisions the runtime directories, verifies dependencies (`curl`, `jq`, `sqlite3`), creates/migrates the SQLite schema, and writes a completion lock so redundant Fail2Ban restarts don't re-run initialization. If initialization ever fails, `actionban` is blocked outright rather than reporting against a broken environment.

---

## 🔐 Worker Pool Semaphore (Concurrency Model)

<img width="900" height="542" alt="Image" src="https://github.com/user-attachments/assets/d7366cdb-eebd-4274-bb8f-639b5eaae09f" />

Every Fail2Ban ban event triggers `actionban` as its own **background subshell** (`& ... ) >> "${LOG_FILE}" 2>&1 &`), so under a ban burst, many IPs can hit the script at almost the same instant. Without a limiter, that means dozens of simultaneous `curl` calls to AbuseIPDB and dozens of concurrent SQLite writers.

The script solves this with a **file-lock-based counting semaphore**, sized by `MAX_WORKERS`:

1. **Numbered slot files.** `MAX_WORKERS` lock files are used as slots — `abuseipdb_worker_1.lock`, `abuseipdb_worker_2.lock`, … `abuseipdb_worker_N.lock` — one per allowed concurrent worker (`MAX_WORKERS = 3` in the diagram above).
2. **Non-blocking acquisition.** Each incoming IP's subshell loops over the slot numbers and tries `flock -n 8` (non-blocking) on each slot file in turn. The **first slot that isn't currently held by another process** is acquired — that's the "Slot 1 / Slot 2 / Slot 3" boxes in the diagram, each mapped to one active IP.
3. **Pool-full backpressure.** If all slots are held (IP 4 in the diagram, arriving after Slots 1–3 are already busy), the subshell doesn't fail or drop the ban — it `sleep 1`s and retries the whole scan until a slot frees up. This is logged as a `WARNING: ... pool may be saturated` if the wait exceeds 30 seconds, which is your signal to raise `MAX_WORKERS`.
4. **Work inside the slot.** Once a slot is acquired, the IP proceeds through the actual reporting pipeline shown at the bottom of the diagram: check the local SQLite DB → `/v2/check` (already listed?) → `/v2/report` (submit) → update the local DB with the new `last_reported_at` timestamp (or roll back the DB row if the report call failed).
5. **Automatic release.** The slot's `flock` is held only for the lifetime of that IP's subshell. As soon as the subshell exits — success or failure — the file descriptor closes, the lock is released automatically (no explicit "unlock" call needed), and the **next waiting IP in the retry loop immediately acquires that freed slot**.
This design caps AbuseIPDB API concurrency at exactly `MAX_WORKERS` regardless of how many IPs are banned simultaneously, while guaranteeing every ban is *eventually* processed — bans queue behind the semaphore, they are never silently dropped. It runs independently of the **per-IP dedup lock** (`abuseipdb_<ip>.lock`), which prevents the *same* IP from being double-processed by overlapping triggers; the worker-pool semaphore instead limits *total* concurrent throughput across *different* IPs.

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

2. **Add the override config**
   `abuseipdb.local` is an **override**, not a replacement — Fail2Ban's `.conf` + `.local` mechanism lets `abuseipdb.local` extend the base action without touching it. Fail2Ban `>= 0.10.0` already ships `action.d/abuseipdb.conf` out of the box, so you normally only need to add the `.local` file:
```bash
   sudo cp abuseipdb.local /etc/fail2ban/action.d/abuseipdb.local
```

   > Verify first with `ls /etc/fail2ban/action.d/abuseipdb.conf`. If it's missing (older Fail2Ban, or a minimal/custom install), fetch the upstream base config before proceeding:
   > ```bash
   > sudo curl -o /etc/fail2ban/action.d/abuseipdb.conf \
   >   https://raw.githubusercontent.com/fail2ban/fail2ban/master/config/action.d/abuseipdb.conf
   > ```
   > Also confirm your version supports it: `fail2ban-client -V` should report `>= 0.10.0`.

3. **Edit `abuseipdb.local`** — uncomment and configure. Note the section split: everything except the API key lives under `[Definition]`; `abuseipdb_apikey` is the only setting under `[Init]`:
```ini
   [Definition]
   norestored = 0

   SQLITE_DB = "/var/lib/fail2ban/abuseipdb/fail2ban_abuseipdb"
   LOG_FILE = "/var/log/abuseipdb/abuseipdb.log"
   BYPASS_FAIL2BAN = 0
   MAX_WORKERS = 10

   actionstart = nohup /etc/fail2ban/action.d/fail2ban-abuseipdb.sh \
       "--actionstart" "<SQLITE_DB>" "<LOG_FILE>" &

   actionban = /etc/fail2ban/action.d/fail2ban-abuseipdb.sh \
       "<abuseipdb_apikey>" "<matches>" "<ip>" "<abuseipdb_category>" "<bantime>" "<restored>" "<BYPASS_FAIL2BAN>" "<SQLITE_DB>" "<LOG_FILE>" "<MAX_WORKERS>"

   [Init]
   abuseipdb_apikey = YOUR_API_KEY
```

   > ⚠️ `norestored = 0` must not be changed — restart/duplicate handling is deliberately controlled at the script level via `BYPASS_FAIL2BAN`, not through Fail2Ban's own `norestored` option.

4. **Wire it into all jails** in `jail.local` — `tp_comment` lets you write whatever report comment you want per jail (not just a generic default), and doubles as your safeguard against leaking sensitive log data into a **public** AbuseIPDB report:
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
| 2 | `COMMENT` | Fail2Ban jail (`tp_comment`) | Custom, per-jail report comment |
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

### The Isolated Database's Exact Role

The local SQLite DB matters in one specific scenario: **the same IP gets banned again.**

It solves two problems:

1. **API quota protection (primary).** `/v2/check` and `/v2/report` share a daily quota. Without the DB, every repeat ban of an already-known IP would blindly hit `/v2/report` again. The DB acts as a cheap local gate before any API call is made:
   | IP status in local DB | Action taken |
   |---|---|
   | Never seen before | Report immediately — no `/v2/check` call needed |
   | Seen before | Ask AbuseIPDB via `/v2/check` first, then decide whether to re-report |

2. **Fail2Ban restart isolation (secondary).** With `BYPASS_FAIL2BAN=1`, the script ignores Fail2Ban's own `<restored>` flag on restart and decides independently — from its own DB — what's already been reported.

**What the DB is *not*:** it is not a live ban list, not a record of currently active Fail2Ban bans, not an expiry/TTL tracker (the `bantime` column is stored for reference only and nothing ever purges a row when a ban expires), and it's not a substitute for `fail2ban-client status`. It only ever answers one question: *"Have we already told AbuseIPDB about this IP, and is that report still relevant?"*

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
