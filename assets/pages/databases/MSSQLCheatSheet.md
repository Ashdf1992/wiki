# Microsoft SQL Server Cheat Sheet

<br>

## SQL DB compatibility_levels
```SQL
SELECT name, compatibility_level
FROM sys.databases;
```
> NOTE:Website for Compatibility Levels here:
https://sqlperformance.com/2019/01/sql-performance/compatibility-levels-and-cardinality-estimation-primer
https://sqlserverbuilds.blogspot.com/2014/01/sql-server-internal-database-versions.html
{.is-info}


<br>

## SQL Server patch level
```SQL
Select @@version;
```
> You can find the latest patch levels using the below 2 URLS
https://sqlserverbuilds.blogspot.com/#sql2012x
https://support.microsoft.com/en-gb/help/321185/how-to-determine-the-version-edition-and-update-level-of-sql-server-an
{.is-info}

<br>

## Expensive queries
```SQL
SELECT TOP 10 SUBSTRING(qt.TEXT, (qs.statement_start_offset/2)+1,
((CASE qs.statement_end_offset
WHEN -1 THEN DATALENGTH(qt.TEXT)
ELSE qs.statement_end_offset
END - qs.statement_start_offset)/2)+1),
qs.execution_count,
qs.total_logical_reads, qs.last_logical_reads,
qs.total_logical_writes, qs.last_logical_writes,
qs.total_worker_time,
qs.last_worker_time,
qs.total_elapsed_time/1000000 total_elapsed_time_in_S,
qs.last_elapsed_time/1000000 last_elapsed_time_in_S,
qs.last_execution_time,
qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_logical_reads DESC -- logical reads
-- ORDER BY qs.total_logical_writes DESC -- logical writes
-- ORDER BY qs.total_worker_time DESC -- CPU time
```
> You can change the order by commenting out and uncommenting the 'ORDER BY' lines at the bottom of the query.
{.is-info}


<br>

## High CPU usage Active Expensive Queries
```SQL
SELECT TOP 10 s.session_id,
           r.status,
           r.cpu_time,
           r.logical_reads,
           r.reads,
           r.writes,
           r.total_elapsed_time / (1000 * 60) 'Elaps M',
           SUBSTRING(st.TEXT, (r.statement_start_offset / 2) + 1,
           ((CASE r.statement_end_offset
                WHEN -1 THEN DATALENGTH(st.TEXT)
                ELSE r.statement_end_offset
            END - r.statement_start_offset) / 2) + 1) AS statement_text,
           COALESCE(QUOTENAME(DB_NAME(st.dbid)) + N'.' + QUOTENAME(OBJECT_SCHEMA_NAME(st.objectid, st.dbid)) 
           + N'.' + QUOTENAME(OBJECT_NAME(st.objectid, st.dbid)), '') AS command_text,
           r.command,
           s.login_name,
           s.host_name,
           s.program_name,
           s.last_request_end_time,
           s.login_time,
           r.open_transaction_count
FROM sys.dm_exec_sessions AS s
JOIN sys.dm_exec_requests AS r ON r.session_id = s.session_id CROSS APPLY sys.Dm_exec_sql_text(r.sql_handle) AS st
WHERE r.session_id != @@SPID
ORDER BY r.cpu_time DESC
```

<br>

## SQL DB index fragmentation
```SQL
SELECT dbschemas.[name] as 'Schema', 
    dbtables.[name] as 'Table', 
    dbindexes.[name] as 'Index',
    indexstats.alloc_unit_type_desc,
    indexstats.avg_fragmentation_in_percent,
    indexstats.page_count
FROM sys.dm_db_index_physical_stats (DB_ID(), NULL, NULL, NULL, NULL) AS indexstats
INNER JOIN sys.tables dbtables on dbtables.[object_id] = indexstats.[object_id]
INNER JOIN sys.schemas dbschemas on dbtables.[schema_id] = dbschemas.[schema_id]
INNER JOIN sys.indexes AS dbindexes ON dbindexes.[object_id] = indexstats.[object_id]
    AND indexstats.index_id = dbindexes.index_id
WHERE indexstats.database_id = DB_ID()
ORDER BY indexstats.avg_fragmentation_in_percent desc
```
> NOTE:
If index fragmentation is above 30% = Rebuild DB Indexes
If below 30% = Reindexing 
Both Rebuilding and Reindexing are really heavy on CPU tho so best done out of hours.
{.is-info}

<br>

## CPU Ring Buffers
```SQL
/* CPU ring buffers */
declare @ts_now bigint
select @ts_now = ms_ticks from
sys.dm_os_sys_info
select record_id, dateadd (ms, (y.[timestamp] -@ts_now), GETDATE()) as EventTime,
SQLProcessUtilization,
SystemIdle,
100 - SystemIdle - SQLProcessUtilization as OtherProcessUtilization
from (
select
record.value('(./Record/@id)[1]', 'int') as record_id,
record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int')
as SystemIdle,
record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]',
'int') as SQLProcessUtilization,
timestamp
from (
select timestamp, convert(xml, record) as record
from sys.dm_os_ring_buffers
where ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
and record like '%<SystemHealth>%') as x
) as y
order by record_id desc
```
<br>


## SPBlitz
### Run First (sp_Blitz.sql)
[sp_Blitz Download](https://github.com/Ashdf1992/wiki/blob/main/assets/attachments/sp_blitz.sql)

### Version 1 SPBlitz Server Health Check
> SQL Server healthcheck
```SQL
sp_blitz;
GO
```

### Version 2 SPBlitz with Server Info
> Adds additional rows at priority 250 with server configuration
```SQL
sp_blitz @CheckServerInfo = 1;
GO
```

### Version 3 SPBlitz Critical Issues Only
> Ignore everything over priority 50, "only critical issues"
```SQL
sp_blitz @IgnorePrioritiesAbove = 50;
GO
```

### Version 4 SPBlitz Disable Checking User Databases
> Disables checks inside of user databases
```SQL
sp_blitz @CheckUserDatabaseObjects = 0;
GO
```

### Version 5 SPBlitz Check User Databases (if Number of DBs is greater than 50
> Required for checking inside user databases if there's more than 50
```SQL
sp_blitz @BringThePain = 1;
GO
```

<br>

## SPBlitzCache
### Run First (sp_BlitzCache.sql)
[sp_BlitzCache Download](https://github.com/Ashdf1992/wiki/blob/main/assets/attachments/sp_blitzcache.sql)

### Version 1 SPBlitzCache Standard
> Looks at the plan cache and displays the 10 most expensive queries sorted by cumulative CPU run time
```SQL
sp_blitzcache;
GO
```

### Version 2 SPBlitzCache Exclude XML (easier working with excel)
> Doesn't return XML fields that make copying into Excel awkward
```SQL
sp_blitzcache @ExportToExcel = 1;
GO
```

### Version 3 SPBlitzCache Change Sort Order
> SPBlitzCache Sort Order can be one of the following, change the command to match (reads / CPU / executions / XPM / recent compliations / writes / memory grant)
```SQL
sp_blitzcache @SortOrder = Reads;
GO
```

<br>

## SPBlitzFirst
### Run First (sp_BlitzFirst.sql)
[sp_BlitzFirst Download](https://github.com/Ashdf1992/wiki/blob/main/assets/attachments/sp_blitzfirst.sql)

### Version 1 SPBlitzFirst Standard
> Takes a sample of data, waits 5 seconds, takes another sample and compares
```SQL
sp_blitzfirst;
GO
```

### Version 2 SPBlitzFirst Expert Mode
> Additional metrics, wait stats, perfmon counters, file stats
```SQL
sp_blitzfirst @ExpertMode = 1;
GO
```

### Version 3 SPBlitzFirst Change runtime duration
> Changes how long it runs for.
```SQL
sp_blitzfirst @ExpertMode = 1 , @Seconds = 60;
GO
```

### Version 4 SPBlitzFirst Since Start up
> Show wait stats for duration of time server has been up / since wait stats have been cleared
```SQL
sp_blitzfirst @SinceStartup = 1;
GO
```

<br>

## SPBlitzWhoIsActive
### Run First (sp_WhoIsActive.sql)
[sp_WhoIsActive Download](https://github.com/Ashdf1992/wiki/blob/main/assets/attachments/sp_whoisactive.sql)

### Version 1 SPBlitzWhoIsActive Standard
> Show current running queries
```SQL
sp_whoisactive;
```

### Version 2 SPBlitzWhoIsActive Average runtime
> Display average run time from plan cache for shown queries 
```SQL
sp_whoisactive @get_avg_time = 1;
```

### Version 3 SPBlitzWhoIsActive Get Locks
> Will show what queries are locking each other
```SQL
sp_whoisactive @get_locks = 1;
```

<br>

## Display Database Names
```SQL
select name from sys.sysdatabases 
order by name;
```

<br>

## Display Database Names and File Paths
```SQL
SELECT
    db.name AS DBName,
    type_desc AS FileType,
    Physical_Name AS Location
FROM
    sys.master_files mf
INNER JOIN 
    sys.databases db ON db.database_id = mf.database_id
```

<br>

## Find wait on a Transaction log.
```SQL
SELECT name, log_reuse_wait_desc
FROM sys.databases
```

<br>

## Check  SQL Database states
> Note: You will need to change 'db.state' to the state that you are looking for from the following list, in  this example, I am looking at Database with the State 'RECOVERING'
<br>
0 = ONLINE
<br>
1 = RESTORING
<br>
2 = RECOVERING
<br>
3 = RECOVERY_PENDING
<br>
4 = SUSPECT
<br>
5 = EMERGENCY
<br>
6 = OFFLINE
```SQL
SELECT *
FROM sys.databases db WHERE
db.state = 2
order by name
```

<br>

## Get a list of endpoints
```SQL
select * from sys.endpoints
```

<br>

## Set Mirroring Endpoint to Stopped
```SQL
ALTER ENDPOINT Mirroring STATE=STOPPED
```

<br>

## Set Mirroring Endpoint to Started
```SQL
ALTER ENDPOINT Mirroring STATE=Started;
```
