import pyodbc 
import pandas as pd
import numpy as np 
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import LabelEncoder
from sklearn.linear_model import LinearRegression

pd.set_option('display.max_columns', None)

# Prevent line wrapping
pd.set_option('display.expand_frame_repr', False)

# Very large display width
pd.set_option('display.width', 10000)

# Show full column content
pd.set_option('display.max_colwidth', None)

# Show all rows if needed
pd.set_option('display.max_rows', None)

conn = pyodbc.connect(
    "DRIVER={ODBC Driver 17 for SQL Server};"
    "SERVER=ServerName;"
    "DATABASE=DatabaseName;"
    "UID=UserName;"
    "PWD=Password;"
)

query = """ SELECT
    BackupID,
    DatabaseName,
    BackupStartDate,
    BackupFinishDate,
    BackupType,
    DurationSeconds,
    BackupSizeMB,
    CompressedBackupSizeMB,
    ChecksumStatus,
    LastLSN,
    FirstLSN
FROM dbo.BackupMonitoringHistory
WHERE BackupStartDate >= DATEADD(DAY, -30, GETDATE())
ORDER BY DatabaseName, BackupStartDate"""
df = pd.read_sql_query(query, conn)

# Duration

df["DurationSeconds"] = df["DurationSeconds"].replace(0, np.nan)
df = df.fillna(0)

# IO_Throughput
df["IO_Throughput_MBPS"] = df["BackupSizeMB"] / df["DurationSeconds"]

# Compression Ratio
df["Compression_Ratio"] = df["BackupSizeMB"] / df["CompressedBackupSizeMB"].replace(0, 1)

df = df.sort_values(["DatabaseName", "BackupStartDate", "BackupType"])

# GrowthRate
df["PreviousBackupSizeMB"] = (
    df.groupby(["DatabaseName", "BackupType"])["BackupSizeMB"].shift(1)
)
df["GrowthRate"] = (
    (df["BackupSizeMB"] - df["PreviousBackupSizeMB"])
    / df["PreviousBackupSizeMB"].replace(0, np.nan)
)

df["GrowthRate"] = df["GrowthRate"].replace([np.inf, -np.inf], np.nan)
df["GrowthRate"] = df["GrowthRate"].fillna(0)
df["GrowthRate_Capped"] = df["GrowthRate"].clip(-5, 5)

# RollingAvgDuration
df["RollingAvgDuration"] = (
    df.groupby("DatabaseName")["DurationSeconds"]
    .transform(lambda x: x.rolling(7, min_periods=1).mean())
)

# RollingAvgThroughput
df["RollingAvgThroughput"] = (
    df.groupby("DatabaseName")["IO_Throughput_MBPS"]
    .transform(lambda x: x.rolling(7, min_periods=1).mean())
)


# Derived I/O Indicators

# Throughput deviation from rolling average
df["ThroughputDeviation"] = (
    (df["IO_Throughput_MBPS"] - df["RollingAvgThroughput"])
    / df["RollingAvgThroughput"].replace(0, 1)
)

# Duration deviation from rolling average
df["DurationDeviation"] = (
    (df["DurationSeconds"] - df["RollingAvgDuration"])
    / df["RollingAvgDuration"].replace(0, 1)
)

# Compression efficiency change
df["RollingAvgCompression"] = (
    df.groupby("DatabaseName")["Compression_Ratio"]
    .transform(lambda x: x.rolling(7, min_periods=1).mean())
)
df["CompressionDeviation"] = (
    (df["Compression_Ratio"] - df["RollingAvgCompression"])
    / df["RollingAvgCompression"].replace(0, 1)
)

# Size-adjusted duration (seconds per GB)
df["SecondsPerGB"] = (
    df["DurationSeconds"]
    / (df["BackupSizeMB"] / 1024).replace(0, 1)
)
df["RollingAvgSecondsPerGB"] = (
    df.groupby("DatabaseName")["SecondsPerGB"]
    .transform(lambda x: x.rolling(7, min_periods=1).mean())
)
df["SecondsPerGB_Deviation"] = (
    (df["SecondsPerGB"] - df["RollingAvgSecondsPerGB"])
    / df["RollingAvgSecondsPerGB"].replace(0, 1)
)

# I/O Risk flag
df["IO_Risk"] = np.where(
    (df["ThroughputDeviation"] < -0.3) &
    (df["DurationDeviation"] > 0.3) &
    (df["GrowthRate_Capped"].abs() < 0.5),
    "HIGH",
    "NORMAL"
)

# MODEL AND ANALYSIS

features = [
    "DurationSeconds",
    "BackupSizeMB",
    "CompressedBackupSizeMB",
    "IO_Throughput_MBPS",
    "Compression_Ratio",
    "GrowthRate",
    "ThroughputDeviation",
    "DurationDeviation",
    "SecondsPerGB"
]

results = []

for db_name in df["DatabaseName"].unique():

    df_db = df[df["DatabaseName"] == db_name].copy()

    df_db["AnomalyReason"] = "NONE"
    df_db["AnomalyDeviation"] = 0.0

    # Machine Learning model
    model = IsolationForest(
        contamination=0.03,
        random_state=42
    )

    df_db["Anomaly"] = model.fit_predict(df_db[features])

    SLA_SECONDS = 3600
    df_db["SLA_Risk"] = np.where(
        df_db["DurationSeconds"] > SLA_SECONDS,
        "HIGH",
        "NORMAL"
    )

    df_db["PredictedFailureRisk"] = np.where(
        (df_db["GrowthRate_Capped"] > 3) |
        ((df_db["IO_Throughput_MBPS"] < 20) & (df_db["DurationSeconds"] > 5)),
        "HIGH",
        "LOW"
    )

    def root_cause(row):
        if row["Compression_Ratio"] < 1:
            return "Compression may be disabled"

        if row["IO_Risk"] == "HIGH":
            return "I/O bottleneck: throughput dropped while size is stable"

        if row["SecondsPerGB_Deviation"] > 0.5:
            return "Storage slowdown: time per GB increasing"

        if (row["IO_Throughput_MBPS"] < 20) and (row["DurationSeconds"] > 5):
            return "Possible storage bottleneck"

        if row["ThroughputDeviation"] < -0.4:
            return "Significant throughput degradation"

        if row["CompressionDeviation"] > 0.3:
            return "Compression behavior change detected"

        if row["GrowthRate"] > 3:
            return "Backup size increase detected"

        return "No obvious issue"

    df_db["RootCauseHint"] = df_db.apply(root_cause, axis=1)

    # LSN Chain Check
    df_logs = df_db[df_db["BackupType"] == "LOG"].copy()
    df_logs = df_logs.sort_values(["DatabaseName", "BackupStartDate"])
    df_logs["PrevLastLSN"] = (
        df_logs.groupby("DatabaseName")["LastLSN"].shift(1)
    )
    df_logs["BrokenLSNChain"] = np.where(
        df_logs["PrevLastLSN"].isna(),
        0,
        np.where(
            np.isclose(df_logs["FirstLSN"], df_logs["PrevLastLSN"]),
            0,
            1
        )
    )
    df_db.loc[df_logs.index, "BrokenLSNChain"] = df_logs["BrokenLSNChain"]
    df_db["BrokenLSNChain"] = df_db["BrokenLSNChain"].fillna(0).astype(int)

    # Anomaly reason
    deviation_scores = abs(df_db[features] - df_db[features].median())
    df_db["AnomalyReason"] = deviation_scores.idxmax(axis=1)


    # ALERT PRIORITY

    def alert_priority(row):
        if row["BrokenLSNChain"] == 1:
            return "CRITICAL"
        if row["SLA_Risk"] == "HIGH":
            return "HIGH"
        if row["PredictedFailureRisk"] == "HIGH":
            return "HIGH"
        if row["IO_Risk"] == "HIGH":
            return "MEDIUM"
        if row["Anomaly"] == -1:
            return "LOW"
        return "INFO"

    df_db["AlertPriority"] = df_db.apply(alert_priority, axis=1)

    # Final selection
    anomalies = df_db[
        (df_db["Anomaly"] == -1) |
        (df_db["SLA_Risk"] == "HIGH") |
        (df_db["PredictedFailureRisk"] == "HIGH") |
        (df_db["BrokenLSNChain"] == 1) |
        (df_db["IO_Risk"] == "HIGH")
    ]

    results.append(anomalies)

if results:

    final_df = pd.concat(results)

    print("\n========== AI BACKUP MONITORING RESULTS ==========\n")

    print(
        final_df[
            [
                "BackupID",
                "DatabaseName",
                "BackupType",
                "BackupStartDate",
                "AlertPriority",
                "IO_Throughput_MBPS",
                "ThroughputDeviation",
                "SecondsPerGB",
                "IO_Risk",
                "PredictedFailureRisk",
                "SLA_Risk",
                "RootCauseHint",
                "AnomalyReason",
            ]
        ]
    )

else:

    print("No significant backup anomalies detected.")
