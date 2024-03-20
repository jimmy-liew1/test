---
tags: [sql_server]
title: Handling Overgrown Transaction Log Files
created: '2024-02-28T00:59:58.471Z'
modified: '2024-02-28T13:49:37.334Z'
---

# Handling Overgrown Transaction Log Files

<p><a name="header1"></a></p>

## Navigation

- [Issue symptoms](#issue-symptoms)
- [Identifying the culprit](#identifying-the-culprit)
- [Fixing](#fixing)
    -   [Option 1](#option-1)
    -   [Option 2](#option-2)
    -   [Option 3](#option-3)
    -   [Option 4](#option-4)
- [Possible complications](#possible-complications)
- [Changelog](#changelog)

## Issue symptoms

- In most cases we'll either get the following alerts followed by the resulting incident:


```
SQL Server Alert System: 'Severity 017' occurred on \\HostName\SQ##*DB

DATE/TIME:      11/26/2020 2:07:37 AM

DESCRIPTION:    The transaction log for database 'DatabaseName' is full due to 'ACTIVE_TRANSACTION'.

COMMENT:        (None)

JOB RUN:        (None)

```

This means that the instance has most likely frozen all operations that require logged transaction (inserts/updates/deletes).

This will require immediate action in order to avoid having a P1 incident and you should go directly to [Identifying the culprit](#identifying-the-culprit) to single out the database that is causing this and then follow the steps outlined in [Option 3](#option-3)

- In other cases we might get an email/incident from TechOps and/or SolarWinds stating that the SQ##\*DB_TLog mount point is at 85%+ utilization. This means that the the issue isn't critical yet, but it has potential to fully lock up the instance in the next 10-60 minutes (depending on when the last TLog backup was done).

[_Back to top_](#header1)

## Identifying the culprit

- First, we'll need to find out which database(s) might have ended up with a transaction log larger than usual.

    To do this, just open a query editor window on the instance that just alerted and run the following block of code (run it all in one go)

```sql
CREATE TABLE #LogSpace
  (
     [DBName]          SYSNAME,
     [LogSizeMB]       FLOAT,
     [LogSpaceUsedPct] FLOAT,
     [Status]          INT
  );

INSERT INTO #LogSpace
EXEC ('DBCC SQLPERF(LOGSPACE);');

WITH [LogFiles] ([DBName], [LogLogicalName], [LogPhysicalLocation])
     AS (SELECT [db].[name]         AS [DBName],
                [f].[name]          AS [LogLogicalName],
                [f].[physical_name] AS [LogPhysicalLocation]
         FROM   [master].[sys].[databases] AS [db]
                INNER JOIN [master].[sys].[master_files] AS [f]
                        ON [db].database_id = [f].database_id
         WHERE
          [f].[type] = 1)
SELECT [ls].[DBName],
              [lf].[LogLogicalName],
              CAST(ROUND([ls].[LogSizeMB],
                         0) AS INT)            AS [LogSizeMB],
              CAST(ROUND([ls].[LogSpaceUsedPct],
                         2) AS DECIMAL(18, 2)) AS [LogSpaceUsedPct],
              CAST(ROUND([ls].[LogSizeMB] - ( [LogSizeMB] * [LogSpaceUsedPct] /
                                              100 ),
                         0) AS INT)            AS [LogSpaceUnusedMB],
              [lf].[LogPhysicalLocation]
FROM   #LogSpace AS [ls]
       INNER JOIN [LogFiles] AS [lf]
               ON [ls].[DBName] = [lf].[DBName]
ORDER  BY
  [LogSizeMB] DESC;

DROP TABLE #LogSpace;

```

- Since the query orders the result set descending based on LogSizeMB the first result will be the largest transaction log file on the instance, and the one that will need to be addressed.

[_Back to top_](#header1)

## Fixing

- The options to fix this are in ascending order of complexity, with the 1st option being for situations that do not require a complex workaround to avoid other issues caused by the TLog mount point being compeltely full.

### Option 1

This usually applies when we get an email/incident from TechOps or SolarWinds stating that there's less than 85% storage space free on the TLog mount point and is failry straightforward to address.

1. If the LogSpaceUsedPct value for the largest TLog on the instance is equal to or grater than 50% you'll first have to run a transaction log backup using the following command (replace the **DatabaseName** string with the name of the database identified in the [Identifying the culprit](#identifying-the-culprit) step), if the LogSpaceUsedPct is really small just run the command from step 3:

```sql
EXEC [DBMaint].[dbo].[USP_QUICKDATABASEBACKUP]
  @DatabaseName = 'DatabaseName',
  @BkpType = 'LOG'; --this will also push the backup to NetBackup

```

2. Run the command from the [Identifying the culprit](#identifying-the-culprit) step and if the value from the LogSpaceUsedPct column has gone down for that transaction log you can proceed to the next step, if not, you can skip to [Option 2](#option-2).
3. After the backup finishes, run the following command to shrink the transaction log file after doing the following replacements in the command:
    -   replace **DatabaseName** with the name of the database
    -   replace **LogicalNameOfTLog** with the logical name of the latest transaction log file ( can be found in the result set of the [Identifying the culprit](#identifying-the-culprit) command)
    -   replace **1000** with the current size of the transaction log divided by 8 (so if the current size of the transaction log is **40000MB** the value that you have to provide is **8000**)

```sql
USE [DatabaseName]
GO
DBCC SHRINKFILE (N'LogicalNameOfTLog', 1000);
GO

```

If not, proceed to [Option 2](#option-2)

[_Back to top_](#header1)

### Option 2

Now chances are that there might be one or more transactions locking up the transaction log so you'll have to kill all the sessions currently hitting that database.

1. To do this run the following command after replacing **DatabaseName** with the name of the database and **YourUserID** with your user name (without the domain name and \\ in front of it), you might have to run this command 2-3 times consecutively just to be sure:

```sql
USE MASTER
GO
DECLARE @Spid INT;
DECLARE @ExecSQL VARCHAR(255);
DECLARE KillCursor CURSOR LOCAL STATIC READ_ONLY FORWARD_ONLY FOR
  SELECT DISTINCT [session_id]
  FROM   [master].[sys].[dm_exec_sessions]
  WHERE
    [nt_user_name] NOT LIKE 'YourUserName%'                 --replace
    AND [session_id] <> @@SPID
    AND [session_id] > 50
    AND [database_id] = DB_ID('DatabaseName');              --replace
OPEN KillCursor
-- Grab the first SPID
FETCH NEXT FROM KillCursor INTO @Spid;
WHILE @@FETCH_STATUS = 0
  BEGIN
      SET @ExecSQL = 'KILL ' + CAST(@Spid AS VARCHAR(50));
      EXEC (@ExecSQL);
      -- Pull the next SPID and repeat
      FETCH NEXT FROM KillCursor INTO @Spid;
  END;
CLOSE KillCursor;
DEALLOCATE KillCursor;

```

2. Run a transaction log backup using the following command (replace the **DatabaseName** string with the name of the database identified in the [Identifying the culprit](#identifying-the-culprit) step), if the LogSpaceUsedPct is really small just run the command from step 3:

```sql
EXEC [DBMaint].[dbo].[USP_QUICKDATABASEBACKUP]
  @DatabaseName = 'DatabaseName',
  @BkpType = 'LOG'; --this will also push the backup to NetBackup

```

3. After the backup finishes, run the following command to shrink the transaction log file after doing the following replacements in the command:
    -   replace **DatabaseName** with the name of the database
    -   replace **LogicalNameOfTLog** with the logical name of the latest transaction log file ( can be found in the result set of the [Identifying the culprit](#identifying-the-culprit) command)
    -   replace **1000** with the current size of the transaction log divided by 8 (so if the current size of the transaction log is **40000MB** the value that you have to provide is **8000**)

```sql
USE [DatabaseName]
GO
DBCC SHRINKFILE (N'LogicalNameOfTLog', 1000);
GO

```

4. Run the command from the [Identifying the culprit](#identifying-the-culprit) step and if the transaction log size is relatively close to the one you've provided in the DBCC SHRINKFILE command, you should be all set and the incident can be closed, if not, you can head to [Option 3](#option-3).

### Option 3

At this point chances are that the TLog mount point doesn't have enough space to process a transaction log backup which means that you'll have to do some juggling with the transaction log files in order to free up some space on the mount point.

1. Run again the command from the [Identifying the culprit](#identifying-the-culprit) step and look for other large transaction logs ( either the second largest, the third largest, or the fourth largest should be good candidates).
2. Run the DBCC SHRINKFILE command, but adapt it to the findings from the above step.

```sql
USE [DatabaseName]
GO
DBCC SHRINKFILE (N'LogicalNameOfTLog', 1000);
GO

```

3. Now we should be good to address the actual cause of the issue, run the following command (replace **YourUserName** with your user name and **DatabaseName** with the database name that has the largest TLog file) to kill any leftover sessions on the database with the largest transaction log (run the command 2-3 times in a row):

```sql
USE MASTER
GO
DECLARE @Spid INT;
DECLARE @ExecSQL VARCHAR(255);
DECLARE KillCursor CURSOR LOCAL STATIC READ_ONLY FORWARD_ONLY FOR
  SELECT DISTINCT [session_id]
  FROM   [master].[sys].[dm_exec_sessions]
  WHERE
    [nt_user_name] NOT LIKE 'YourUserName%'                 --replace
    AND [session_id] <> @@SPID
    AND [session_id] > 50
    AND [database_id] = DB_ID('DatabaseName');              --replace
OPEN KillCursor
-- Grab the first SPID
FETCH NEXT FROM KillCursor INTO @Spid;
WHILE @@FETCH_STATUS = 0
  BEGIN
      SET @ExecSQL = 'KILL ' + CAST(@Spid AS VARCHAR(50));
      EXEC (@ExecSQL);
      -- Pull the next SPID and repeat
      FETCH NEXT FROM KillCursor INTO @Spid;
  END;
CLOSE KillCursor;
DEALLOCATE KillCursor;

```

4. Run a transaction log backup after replacing **DatabaseName** with the name of the database with the largest TLog file


```sql
EXEC [DBMaint].[dbo].[USP_QUICKDATABASEBACKUP]
  @DatabaseName = 'DatabaseName',
  @BkpType = 'LOG'; --this will also push the backup to NetBackup

```

5. After the backup finishes, run the following command to shrink the transaction log file after doing the following replacements in the command:
    -   replace **DatabaseName** with the name of the database
    -   replace **LogicalNameOfTLog** with the logical name of the latest transaction log file ( can be found in the result set of the [Identifying the culprit](#identifying-the-culprit) command)
    -   replace **1000** with the current size of the transaction log divided by 8 (so if the current size of the transaction log is **40000MB** the value that you have to provide is **8000**)

```sql
USE [DatabaseName]
GO
DBCC SHRINKFILE (N'LogicalNameOfTLog', 1000);
GO

```

6. Run the command from the [Identifying the culprit](#identifying-the-culprit) step and if the transaction log size is relatively close to the one you've provided in the DBCC SHRINKFILE command, you should be all set and the incident can be closed, if not, you can head to [Option 4](#option-4).

### Option 4

> Desperate times call for desperate measures

For this option you'll have to address the initially identified database (the one with the largest transaction log file)

1. Set the database to use the simple recovery model:

```sql
ALTER DATABASE [DatabaseName] SET RECOVERY SIMPLE;
GO

```

2. Kill any sessions that might still be on it (if you've made it here, you should already know what to repalce):

```sql
USE MASTER
GO
DECLARE @Spid INT;
DECLARE @ExecSQL VARCHAR(255);
DECLARE KillCursor CURSOR LOCAL STATIC READ_ONLY FORWARD_ONLY FOR
  SELECT DISTINCT [session_id]
  FROM   [master].[sys].[dm_exec_sessions]
  WHERE
    [nt_user_name] NOT LIKE 'YourUserName%'                 --replace
    AND [session_id] <> @@SPID
    AND [session_id] > 50
    AND [database_id] = DB_ID('DatabaseName');              --replace
OPEN KillCursor
-- Grab the first SPID
FETCH NEXT FROM KillCursor INTO @Spid;
WHILE @@FETCH_STATUS = 0
  BEGIN
      SET @ExecSQL = 'KILL ' + CAST(@Spid AS VARCHAR(50));
      EXEC (@ExecSQL);
      -- Pull the next SPID and repeat
      FETCH NEXT FROM KillCursor INTO @Spid;
  END;
CLOSE KillCursor;
DEALLOCATE KillCursor;

```

3. Shrink the transaction log file

```sql
USE [DatabaseName]
GO
DBCC SHRINKFILE (N'LogicalNameOfTLog', 1000);
GO

```

4. Switch back to full recovery model:

```sql
ALTER DATABASE [DatabaseName] SET RECOVERY FULL;
GO

```

5. Take a full backup:

```sql
EXEC [DBMaint].[dbo].[usp_QuickDatabaseBackup]
@DatabaseName = 'DatabaseName',
@BkpType = 'FULL',
@IsCopyOnly = 'No' --this will also push the backup to NetBackup

```

[_Back to top_](#header1)

## Possible complications

None aside from some potentially killed transactions and a gap in transaction log history for the amount of time when the database is in Simple recovery model (implying it wasn't using the Simple recovery model before).

[_Back to top_](#header1)

## Changelog

| Date | Version | Comment | Author |
|---|---|---|---|
| 02.Dec.2020 | 1.0 | Document created | vdrumea |

[_Back to top_](#header1)
