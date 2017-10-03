# Mssql

* change memory threshold on resource pools
* kill spi for long-running tasks.
* cpus configurations

## Scripts
* WAITFOR DELAY '00:02'
* kill spid #spid number

### show actual queries

SELECT      r.start_time [Start Time],session_ID [SPID],
            DB_NAME(database_id) [Database],
            SUBSTRING(t.text,(r.statement_start_offset/2)+1,
            CASE WHEN statement_end_offset=-1 OR statement_end_offset=0
            THEN (DATALENGTH(t.Text)-r.statement_start_offset/2)+1
            ELSE (r.statement_end_offset-r.statement_start_offset)/2+1
            END) [Executing SQL],
            Status,command,wait_type,wait_time,wait_resource,
            last_wait_type
FROM        sys.dm_exec_requests r
OUTER APPLY sys.dm_exec_sql_text(sql_handle) t
WHERE       session_id != @@SPID -- don't show this query
AND         session_id > 50 -- don't show system queries
ORDER BY    r.start_time

### Kill freeze processes from database.

declare @max_count int, @count int, @sqlstring varchar(100)
declare @spid_table table (spid int NOT NULL)

INSERT @spid_table
select spid
from master.dbo.sysprocesses
where spid in (select blocked from master.dbo.sysprocesses where blocked <> 0) and blocked = 0

select @max_count = MAX(spid) FROM @spid_table
select top 1 @count = spid from @spid_table

while @count <= @max_count
begin
select @sqlstring = 'kill ' + CONVERT(varchar(4), @count)
exec(@sqlstring)
print @sqlstring

IF @count = @max_count
begin
break
end
ELSE
BEGIN
select top 1 @count = spid FROM @spid_table where spid > @count
end
end

### Freezeinfo related queries 



USE Master
GO
EXEC sp_who2


USE Master
GO
SELECT * 
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
GO


SELECT      r.start_time [Start Time],session_ID [SPID],
            DB_NAME(database_id) [Database],
            SUBSTRING(t.text,(r.statement_start_offset/2)+1,
            CASE WHEN statement_end_offset=-1 OR statement_end_offset=0
            THEN (DATALENGTH(t.Text)-r.statement_start_offset/2)+1
            ELSE (r.statement_end_offset-r.statement_start_offset)/2+1
            END) [Executing SQL],
            Status,command,wait_type,wait_time,wait_resource,
            last_wait_type
FROM        sys.dm_exec_requests r
OUTER APPLY sys.dm_exec_sql_text(sql_handle) t
WHERE       session_id != @@SPID -- don't show this query
AND         session_id > 50 -- don't show system queries
ORDER BY    r.start_time

### clear tempdb

use tempdb
GO

DBCC FREEPROCCACHE -- clean cache
DBCC DROPCLEANBUFFERS -- clean buffers
DBCC FREESYSTEMCACHE ('ALL') -- clean system cache
DBCC FREESESSIONCACHE -- clean session cache
DBCC SHRINKDATABASE(tempdb, 10); -- shrink tempdb
dbcc shrinkfile ('tempdev') -- shrink db file
dbcc shrinkfile ('templog') -- shrink log file
GO
	
-- report the new file sizes
SELECT name, size
FROM sys.master_files
WHERE database_id = DB_ID(N'tempdb');
GO

### check db
DBCC CHECKDB

### Most used tables 
--SQL Script begin
IF OBJECT_ID('tempdb..#Temp') IS NOT NULL
DROP TABLE #Temp
GO

CREATE TABLE #Temp
(TableName NVARCHAR(255), UserSeeks DEC, UserScans DEC, UserUpdates DEC)
INSERT INTO #Temp
EXEC sp_MSForEachDB 'USE [?]; IF DB_ID(''?'') > 4
BEGIN
SELECT DB_NAME() + ''.'' + object_name(b.object_id), a.user_seeks, a.user_scans, a.user_updates 
FROM sys.dm_db_index_usage_stats a
RIGHT OUTER JOIN [?].sys.indexes b on a.object_id = b.object_id and a.database_id = DB_ID()
WHERE b.object_id > 100 
END'

SELECT TableName as 'Table Name', sum(UserSeeks + UserScans + UserUpdates) as 'Total Accesses',
sum(UserUpdates) as 'Total Writes', 
CONVERT(DEC(25,2),(sum(UserUpdates)/sum(UserSeeks + UserScans + UserUpdates)*100)) as '% Accesses are Writes',
sum(UserSeeks + UserScans) as 'Total Reads', 
CONVERT(DEC(25,2),(sum(UserSeeks + UserScans)/sum(UserSeeks + UserScans + UserUpdates)*100)) as '% Accesses are Reads',
SUM(UserSeeks) as 'Read Seeks', CONVERT(DEC(25,2),(SUM(UserSeeks)/sum(UserSeeks + UserScans)*100)) as '% Reads are Index Seeks', 
SUM(UserScans) as 'Read Scans', CONVERT(DEC(25,2),(SUM(UserScans)/sum(UserSeeks + UserScans)*100)) as '% Reads are Index Scans'
FROM #Temp
GROUP by TableName
HAVING sum(UserSeeks + UserScans) > 0
ORDER by sum(UserSeeks + UserScans + UserUpdates) DESC
DROP table #Temp
--SQL Script end

## location 
D:\MSSQL\100\Tools\Binn\VSShell\Common7\IDE\Ssms.exe















