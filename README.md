# sql-backup-intelligence

This project is a Python-based approach to monitoring SQL Server backups using real backup history data. The idea is simple: instead of only checking whether a backup succeeded or failed, we try to understand how healthy the backup process actually is over time.

It looks at things like backup duration, size changes, compression behavior, throughput, and SQL Server wait stats. Based on this, it tries to detect unusual patterns and highlight backups that don’t behave like the rest.

**What this helps with**

In real environments, backups usually don’t fail suddenly. They degrade slowly. You might see:
- backups taking longer than usual
- backup size suddenly increasing
- slower IO throughput
- pressure on WRITELOG or BACKUPIO
- log chain issues
- inconsistent performance across databases

This script tries to surface those issues early.

**What the script does**

It connects to SQL Server and reads backup history from a monitoring table. After that, it builds a few derived metrics like:
- IO throughput (backup size / duration)
- compression ratio
- growth rate compared to previous backups
- rolling averages for duration and throughput
- normalized wait stats

Then it runs an Isolation Forest model per database to flag unusual behavior.

On top of that, it applies a few simple rules for:
- SLA violations (long running backups)
- suspected performance issues
- backup growth spikes
- WRITELOG / BACKUPIO pressure
- broken log backup chains

**Output**

The final output is a filtered list of backups that look abnormal. For each one, it shows:
- database name
- backup type
- time of backup
- predicted risk level
- SLA status
- a simple root cause hint
- the metric that looked most unusual

**Example reasons might be:**
- IO throughput drop
- backup size growth
- write log pressure
- storage bottleneck signals

**Why Isolation Forest**

Instead of setting fixed thresholds for everything, Isolation Forest helps identify patterns that don't match normal behavior for that specific database.

Each database is treated separately, because "normal" for one system may be completely different for another.

**Requirements**
Python 3.x
SQL Server access
ODBC Driver 17
pandas, numpy, scikit-learn, pyodbc

**Install dependencies:**

pip install pandas numpy scikit-learn pyodbc

**How to use**

- Update your SQL Server connection details in the script and point it to your backup history table.
- Run the script and it will print a list of backups that look suspicious along with a simple explanation of why they were flagged.

**What this is not**

- This is not a replacement for backup jobs or SQL Server native backup validation.
- It's more of an extra layer on top of existing monitoring to help spot trends early and reduce surprise failures.

**Future ideas**

- Some things that can be added later:
- restore validation checks
- dashboard (Power BI or Grafana)
- alerting via email or Teams
- cross-database comparison
- better forecasting for backup growth
- integration with Query Store or disk IO metrics
