# AI Backup Monitoring System

An intelligent SQL Server backup monitoring solution that uses machine learning anomaly detection and predictive analytics to identify performance issues, SLA violations, and potential backup failures.

## Features

- **Anomaly Detection**: Uses Isolation Forest ML model to identify unusual backup patterns
- **Performance Analysis**: Monitors I/O throughput, duration, compression efficiency, and growth rates
- **SLA Monitoring**: Tracks backup duration against configurable SLA thresholds
- **Failure Prediction**: Predicts potential backup failures based on size growth and I/O metrics
- **Root Cause Analysis**: Provides intelligent hints about the underlying causes of anomalies
- **LSN Chain Validation**: Detects broken transaction log backup chains
- **Alert Prioritization**: Automatically assigns severity levels (CRITICAL, HIGH, MEDIUM, LOW, INFO)
- **Multi-Database Support**: Monitors multiple databases simultaneously with per-database analysis

## What It Monitors

### Core Metrics
- **Backup Duration**: Time taken for backup operations (in seconds)
- **Backup Size**: Uncompressed and compressed backup sizes (in MB)
- **I/O Throughput**: Backup speed measured in MB/s
- **Compression Ratio**: Effectiveness of backup compression
- **Database Growth Rate**: Change in backup size between runs

### Derived Indicators
- **Rolling Averages**: 7-day moving averages for duration and throughput
- **Deviation Scores**: Percentage deviation from rolling baselines
- **Seconds Per GB**: Normalized performance metric (duration per GB of data)
- **Compression Efficiency Changes**: Deviations in compression behavior

## Alert Levels

| Priority | Condition | Action |
|----------|-----------|--------|
| **CRITICAL** | Broken LSN chain in transaction logs | Investigate immediately - recovery chain is broken |
| **HIGH** | SLA violation (duration exceeds threshold) or predicted failure risk | Address performance or scaling issues |
| **MEDIUM** | I/O bottleneck detected | Monitor storage performance |
| **LOW** | ML-detected anomalies without other risk factors | Review for potential issues |
| **INFO** | Normal operation | No action needed |

## Requirements

- Python 3.7+
- SQL Server (2016 or later)
- ODBC Driver 17 for SQL Server

### Python Dependencies

```bash
pyodbc>=4.0.0
pandas>=1.0.0
numpy>=1.18.0
scikit-learn>=0.22.0
```

## Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/ai-backup-monitoring.git
   cd ai-backup-monitoring
   ```

2. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

3. **Configure database connection**
   Edit the connection string in the script:
   ```python
   conn = pyodbc.connect(
       "DRIVER={ODBC Driver 17 for SQL Server};"
       "SERVER=your_server_name;"
       "DATABASE=your_database_name;"
       "UID=your_username;"
       "PWD=your_password;"
   )
   ```

4. **Ensure backup history table exists**
   The script expects a table named `dbo.BackupMonitoringHistory` with the following columns:
   - BackupID
   - DatabaseName
   - BackupStartDate
   - BackupFinishDate
   - BackupType
   - DurationSeconds
   - BackupSizeMB
   - CompressedBackupSizeMB
   - ChecksumStatus
   - LastLSN
   - FirstLSN

## Usage

### Basic Execution
```bash
python backup_monitoring.py
```

### Output
The script prints a formatted table of detected anomalies with the following columns:

- **BackupID**: Unique identifier for the backup
- **DatabaseName**: Name of the database
- **BackupType**: Type of backup (FULL, DIFF, LOG, etc.)
- **BackupStartDate**: When the backup started
- **AlertPriority**: Severity level of the alert
- **IO_Throughput_MBPS**: Backup speed in MB/s
- **ThroughputDeviation**: Percentage deviation from baseline
- **SecondsPerGB**: Performance metric (seconds per GB)
- **IO_Risk**: I/O bottleneck status
- **PredictedFailureRisk**: Risk assessment for failure
- **SLA_Risk**: SLA violation status
- **RootCauseHint**: Suggested reason for the anomaly
- **AnomalyReason**: Feature with highest deviation

### Example Output
```
========== AI BACKUP MONITORING RESULTS ==========

  BackupID DatabaseName BackupType    BackupStartDate AlertPriority  ... RootCauseHint
0     1234       MyDB     FULL    2024-01-15 23:00:00           HIGH  ... I/O bottleneck: throughput dropped
1     1235       MyDB      LOG    2024-01-16 00:15:00        CRITICAL  ... No obvious issue
```

## Configuration

### Tunable Parameters

**SLA Threshold** (line 131)
```python
SLA_SECONDS = 3600  # 1 hour
```

**Anomaly Detection Sensitivity** (line 139)
```python
contamination=0.03,  # Expect ~3% of backups to be anomalies
```

**Rolling Window** (line 82-93)
```python
x.rolling(7, min_periods=1).mean()  # 7-day rolling average
```

**Growth Rate Cap** (line 59)
```python
df["GrowthRate_Capped"] = df["GrowthRate"].clip(-5, 5)  # ±5x
```

### Anomaly Threshold Adjustments

For stricter monitoring, decrease `contamination`:
```python
contamination=0.01  # Only ~1% flagged as anomalies
```

For more lenient detection, increase `contamination`:
```python
contamination=0.05  # ~5% flagged as anomalies
```

## How It Works

1. **Data Collection**: Queries last 30 days of backup history from SQL Server
2. **Feature Engineering**: Calculates performance metrics and deviations
3. **Anomaly Detection**: Uses Isolation Forest to identify unusual patterns
4. **Risk Assessment**: Evaluates SLA compliance and failure risk
5. **Root Cause Analysis**: Suggests probable causes for detected issues
6. **Alert Prioritization**: Assigns severity based on multiple risk factors
7. **Results Aggregation**: Displays only problematic backups

## Root Cause Hints

The system provides intelligent hints based on detected patterns:

- **Compression disabled**: Compression ratio below 1
- **I/O bottleneck**: Throughput dropped while backup size stable
- **Storage slowdown**: Time per GB increasing significantly
- **Possible storage bottleneck**: Low throughput with long duration
- **Throughput degradation**: Significant drop from baseline
- **Compression behavior change**: Deviation in compression efficiency
- **Backup size increase**: Rapid database growth detected

## Requirements for Database Setup

If your backup history table doesn't exist, create it with:

```sql
CREATE TABLE dbo.BackupMonitoringHistory (
    BackupID INT PRIMARY KEY,
    DatabaseName NVARCHAR(128),
    BackupStartDate DATETIME2,
    BackupFinishDate DATETIME2,
    BackupType NVARCHAR(20),
    DurationSeconds INT,
    BackupSizeMB BIGINT,
    CompressedBackupSizeMB BIGINT,
    ChecksumStatus INT,
    LastLSN NUMERIC(25,0),
    FirstLSN NUMERIC(25,0)
);
```

Populate it by querying `msdb.dbo.backupset` and related tables.

## Scheduling

### Windows Task Scheduler
```batch
python C:\path\to\backup_monitoring.py >> C:\logs\backup_monitoring.log
```

### Linux/Mac (Cron)
```bash
0 1 * * * python /path/to/backup_monitoring.py >> /var/log/backup_monitoring.log
```

### Integration with Alerts
Pipe output to a logging system or email alerting service for critical alerts.

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add improvement'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

### Potential Enhancements
- Email alerting for critical backups
- Dashboard visualization with historical trends
- Database-specific SLA configuration
- Integration with monitoring platforms (Grafana, Datadog, etc.)
- Support for other SQL Server versions
- Configuration file support (JSON/YAML)

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## Troubleshooting

**Connection Error: "ODBC Driver 17 not found"**
- Install ODBC Driver 17 for SQL Server from Microsoft

**No anomalies detected**
- This is normal! It means backups are running smoothly
- Adjust `contamination` parameter to detect more subtle issues
- Check that backup history table is populated with recent data

**High false positive rate**
- Increase `contamination` value (e.g., from 0.03 to 0.05)
- Review and adjust threshold values for your environment

**Memory issues with large datasets**
- Reduce the time window in the SQL query (e.g., -14 instead of -30 days)
- Process databases individually

## Support

For issues, questions, or feature requests, please open an issue on GitHub.

## Acknowledgments

- Scikit-learn for machine learning capabilities
- Pandas for data manipulation
- SQL Server community for feedback and use cases

---
