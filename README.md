# SQL Server Encryption Audit & Monitoring

There are serveral instances whether it be in testing or troubleshooting that a SQL Server may have ForcedEncryption set to off. We're all human, sometimes after an exhausting day the tech in charge of the intiative may forget to turn encryption back on. This repository contains a PowerShell automation script and SQL schema used to centrally audit **Force Encryption** settings across all SQL Server instances in an enterprise environment.  
The script retrieves all active SQL Servers from an inventory database, checks each server's encryption registry value, and logs the results into a central compliance table.  
This dataset can also be visualized in Power BI to provide an "at-a-glance" view of environment-wide encryption compliance.

---

## What This Script Does

- Automatically pulls the list of active SQL Servers from an inventory database (`Overwatch.Instance`)
  *You could tweak it to read from a csv or hardcoded list if that's preferred.*
- Checks whether **Force Encryption** is enabled on each SQL Server instance
- Logs each server’s:
  - Name  
  - Timestamp of the check  
  - Encryption status (`YES` / `NO`)
- Stores results in a centralized SQL Server table
- Supports visualization via Power BI dashboards

This provides a simple, reliable way to ensure encryption compliance across the entire SQL estate.

---

## 🗂 Database Table Schema

Create this table in your central monitoring database before running the script:

```sql
CREATE TABLE [dbo].[SqlServerEncryptionAudit](				
	[ServerName] [nvarchar](128) NULL,			
	[DateChecked] [datetime] NULL,			
	[EncryptionStatus] [varchar](3) NULL			
) ON [PRIMARY]
```

## How the Script Works

1. Retrieve Active SQL Servers
- The script queries the Overwatch database to retrieve all active SQL Server instances:
  ```sql
  SELECT [Name]
  FROM [Overwatch].[dbo].[Instance]
  WHERE Type = 'SQL SERVER'
  AND [Status] = 'ACTIVE';
  ```
2. Clear Previous Audit Data
- Before each run, the central audit table is truncated:

```sql
TRUNCATE TABLE dbo.SqlServerEncryptionAudit;
```






