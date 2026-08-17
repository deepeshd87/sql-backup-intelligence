# SQL Backup Intelligence

**Catch SQL Server backup degradation weeks before it becomes an incident.**

Most backup monitoring is binary — the backup either succeeded or it didn't. But most backup problems don't *start* with a failure. They start with drift: durations creeping up, throughput sliding down, compression ratios slipping. By the time a backup actually fails or a maintenance window is breached, the warning signs have usually been in the data for weeks.

`sql-backup-intelligence` persists your SQL Server backup history as structured telemetry and applies a **hybrid detection model** — an Isolation Forest (unsupervised ML) to surface patterns you'd never think to write rules for, layered with **operational business rules** to catch the things a model is blind to (SLA breaches, broken LSN chains). The model catches the unexpected; the rules catch the known-critical. Together they've proven more reliable than either alone.

> Companion article: [A Hybrid ML and Rule-Based Approach to SQL Backup Monitoring](https://hackernoon.com/a-hybrid-ml-and-rule-based-approach-to-sql-backup-monitoring)

---

## Why this exists

Traditional threshold alerting misses two common failure modes:

- **Slow drift.** A backup that grows from 20 minutes to 55 minutes over three months looks "fine" on any single day — until it breaches the maintenance window. No individual run trips a threshold, but the trend is unmistakable in the telemetry.
- **Silent recoverability loss.** A broken transaction-log LSN chain leaves duration, size, and throughput looking perfectly normal — while quietly making point-in-time recovery impossible.

An ML model alone won't catch either (a consistently slow backup looks "normal" to the model; a broken LSN chain has no signal in the performance features). Rules alone can't anticipate patterns nobody thought to encode. This project combines both.

---

## Features

- **ML anomaly detection** — per-database Isolation Forest models flag unusual backup patterns without predefined thresholds
- **Operational rules layer** — SLA-breach detection, predicted-failure heuristics, and LSN-chain integrity checks
- **Derived telemetry** — I/O throughput, compression ratio, growth rate, rolling baselines, and deviation scores
- **Root-cause hints** — maps detected anomalies to likely causes (I/O bottleneck, compression disabled, storage slowdown, etc.)
- **Alert prioritization** — assigns CRITICAL / HIGH / MEDIUM / LOW / INFO so recoverability issues always surface first
- **Per-database analysis** — models are trained per database to avoid conflating different workload profiles

---

## Alert levels

| Priority | Condition | Action |
|----------|-----------|--------|
| **CRITICAL** | Broken LSN chain in transaction logs | Investigate immediately — recovery chain is broken |
| **HIGH** | SLA violation or predicted failure risk | Address performance or scaling issues |
| **MEDIUM** | I/O bottleneck detected | Monitor storage performance |
| **LOW** | ML-detected anomaly with no other risk factor | Review for potential issues |
| **INFO** | Normal operation | No action needed |

---

## Requirements

- Python 3.7+
- SQL Server 2016 or later
- [ODBC Driver 17 for SQL Server](https://learn.microsoft.com/sql/connect/odbc/download-odbc-driver-for-sql-server)

Python dependencies (see `requirements.txt`):

```
pyodbc>=4.0.0
pandas>=1.0.0
numpy>=1.18.0
scikit-learn>=0.22.0
```

---

## Quick start

```bash
# 1. Clone
git clone https://github.com/deepeshd87/sql-backup-intelligence.git
cd sql-backup-intelligence

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure your connection (see below) — copy the example and edit
cp config.example.yaml config.yaml
#   then edit config.yaml with your server details

# 4. Create the telemetry table (one-time)
#    Run backup_telemetry.sql against your SQL Server instance

# 5. Collect backup history, then run detection
Run DataCollection.sql       # populates the telemetry table from msdb
python detect_anomalies.py        # runs the hybrid detection and prints alerts
```

> **First run note:** anomaly detection needs history to learn from. Let `DataCollection` run on a schedule for at least 7–14 days (ideally a few weeks) before the model output becomes meaningful. On day one you'll mostly see rule-based alerts, which is expected.

---

## Configuration

Credentials are **not** hardcoded. Copy `config.example.yaml` to `config.yaml` (which is git-ignored) and edit:

```yaml
# config.yaml
database:
  driver: "ODBC Driver 17 for SQL Server"
  server: "YOUR_SERVER_NAME"
  database: "YOUR_DATABASE_NAME"
  # Use trusted (Windows) auth OR username/password — not both
  trusted_connection: true
  # username: "YOUR_USERNAME"
  # password: "YOUR_PASSWORD"

detection:
  sla_seconds: 3600          # backup duration SLA (1 hour)
  contamination: 0.03        # expected anomaly rate (~3% of records)
  rolling_window_days: 7     # rolling-baseline window
  growth_rate_cap: 5         # cap extreme growth-rate values at ±5x
  lookback_days: 30          # how much history to analyze per run
```

**Tuning tips**
- **Too many alerts?** Lower `contamination` (e.g. `0.01`) and/or raise thresholds for your environment.
- **Missing subtle issues?** Raise `contamination` (e.g. `0.05`).
- **Memory pressure on large estates?** Reduce `lookback_days` and let the script process databases individually.

---

## How it works

1. **Collect** — `collect_telemetry.py` pulls recent backup history from `msdb.dbo.backupset` into a persistent telemetry table (`dbo.BackupMonitoringHistory`), so trends can be analyzed over time instead of queried on demand.
2. **Engineer features** — derives I/O throughput, compression ratio, growth rate, rolling baselines, and deviation scores per database.
3. **Detect (ML)** — trains a per-database Isolation Forest to flag statistical outliers.
4. **Detect (rules)** — layers SLA-breach, predicted-failure, and LSN-chain-integrity checks on top.
5. **Explain & prioritize** — attaches a root-cause hint and an alert priority to each finding, sorted so CRITICAL items surface first.

---

## Roadmap

- [ ] `pip install`-able package (PyPI release)
- [ ] Config-file support (YAML) — *in progress per this README*
- [ ] Email / webhook alerting for CRITICAL findings
- [ ] Dashboard for historical trend visualization
- [ ] Integrations (Grafana, Datadog, etc.)
- [ ] Per-database SLA overrides

---

## Contributing

Contributions and real-world feedback are welcome — especially reports of the tool catching (or missing) backup issues in your environment.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add improvement'`)
4. Push (`git push origin feature/improvement`)
5. Open a Pull Request

If you deploy this, please [open an issue](https://github.com/deepeshd87/sql-backup-intelligence/issues) describing where and how it worked — that feedback directly shapes the roadmap.

---

## License

MIT — see [LICENSE](LICENSE).

---

## Acknowledgments

- [scikit-learn](https://scikit-learn.org/) for the Isolation Forest implementation
- [pandas](https://pandas.pydata.org/) for data manipulation
- The SQL Server community for feedback and real-world use cases
- Isolation Forest: Liu, Ting & Zhou (2008), [*Isolation Forest*](https://cs.nju.edu.cn/zhouzh/zhouzh.files/publication/icdm08.pdf) (ICDM)
