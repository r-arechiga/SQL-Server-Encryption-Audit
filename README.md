# SQL Server Encryption Audit– Centralized PowerShell Compliance Scanner 
---
There are several instances, whether during testing or troubleshooting, when a SQL Server may have ForceEncryption set to OFF. We’re all human, sometimes after an exhausting day, the technician responsible for the task may simply forget to turn encryption back on.

This repository contains a PowerShell automation script and SQL setup used to centrally audit Force Encryption settings across all SQL Server instances in an enterprise environment. The script retrieves all active SQL Servers from an inventory database, checks each server’s encryption registry value, and logs the results into a central compliance table.

This dataset can also be visualized in Power BI to provide an at-a-glance view of environment-wide encryption compliance.

---

## Powershell Script
```powershell

#Where the data will land and be stored in
$CentralServer = "ServerName"     
$CentralDatabase = "EncryptionDB"    

#Where the list of servers live. 
$OverwatchServer = "ServerName" 
$OverwatchDatabase = "Overwatch" 

# Get active SQL Server instance names from Overwatch
$ServerListQuery = @"
SELECT [Name]
FROM [Overwatch].[dbo].[Instance]
WHERE Type = 'SQL SERVER' AND [Status] = 'ACTIVE';
"@

$TargetServers = Invoke-Sqlcmd -ServerInstance $OverwatchServer -Database $OverwatchDatabase -Query $ServerListQuery -ErrorAction Stop | 
                 Select-Object -ExpandProperty Name

if (-not $TargetServers) {
    Write-Host "No active SQL Servers found in Overwatch.Instance." -ForegroundColor Red
    exit
}

Write-Host "Retrieved $($TargetServers.Count) active SQL Servers from Overwatch.Instance." -ForegroundColor Yellow

# Clear centralized table before new run
$TruncateQuery = "TRUNCATE TABLE dbo.SqlServerEncryptionAudit;"
Invoke-Sqlcmd -ServerInstance $CentralServer -Database $CentralDatabase -Query $TruncateQuery -ErrorAction Stop
Write-Host "Central table truncated successfully." -ForegroundColor Yellow

# T-SQL query to run on each target server
$CheckQuery = @"
DECLARE @EncryptionForced INT;
EXEC xp_instance_regread
    N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\Microsoft SQL Server\MSSQLSERVER\SuperSocketNetLib',
    N'ForceEncryption',
    @EncryptionForced OUTPUT;

SELECT 
    @@SERVERNAME AS ServerName,
    GETDATE() AS DateChecked,
    CASE WHEN @EncryptionForced = 1 THEN 'YES'
         ELSE 'NO' 
    END AS EncryptionStatus;
"@

foreach ($TargetServer in $TargetServers) {
    try {
        $result = Invoke-Sqlcmd -ServerInstance $TargetServer -Query $CheckQuery -ErrorAction Stop

        foreach ($row in $result) {
            # Insert result into central SQL table
            $insertQuery = "INSERT INTO dbo.SqlServerEncryptionAudit (ServerName, DateChecked, EncryptionStatus)
                            VALUES ('$($row.ServerName)', '$($row.DateChecked.ToString("yyyy-MM-dd HH:mm:ss"))', '$($row.EncryptionStatus)')"

            Invoke-Sqlcmd -ServerInstance $CentralServer -Database $CentralDatabase -Query $insertQuery -ErrorAction Stop
        }

        Write-Host "[$TargetServer] Successfully logged encryption status." -ForegroundColor Green
    }
    catch {
        Write-Host "[$TargetServer] Failed to retrieve or insert data: $_" -ForegroundColor Red
    }
}

```
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

## Database Table Schema

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

3. Check Encryption Status
- For each server, the script reads this registry key:

```pgsql
HKEY_LOCAL_MACHINE\
Software\Microsoft\Microsoft SQL Server\
MSSQLSERVER\SuperSocketNetLib\ForceEncryption
```

- if set to 1, encryption is enabled = YES
- if set to anything else, then = NO

4. Insert Results into the Central Table
- Each server's compliance result is logged for downstream reporting

## Power BI Integration (Optional)

This script powers the encryption check visual in the SQL Server Big Brother Dashboard. (https://github.com/r-arechiga/SQL-Server-Big-Brother-Dashboard)

<img width="1847" height="210" alt="image" src="https://github.com/user-attachments/assets/fcc25439-031c-4cad-8b58-b21942181cb0" />


<img width="1013" height="820" alt="image" src="https://github.com/user-attachments/assets/cf532aa9-d122-4e22-9791-3b3102e4e918" />

---


- Displays all SQL Servers and their encryption status
- Highlights non-compliant servers in red
- Provides filters by instnace and status
- Includes a donut chart showing overall compliance percentage

Great for governance, audits, and operational transparency. 


## Use Cases

- Security compliance / internal audit requirements
- Verifying encryption before enabling TLS/SSL enforcement
- Tracking misconfigurations across environments
- Power BI-powered compliance dashboards
- Automating routine health and security checks

Please feel free to use, adapt, or extend this script for your own enviornment :)



