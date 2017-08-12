---
layout: post
title: "Give that overgrown table a gentle purging"
permalink: give-that-overgrown-table-a-gentle-purging
date: 2017-07-27 20:33:24
comments: true
description: "Give that overgrown table a gentle purging"
keywords: "MSSQL Server"
category: MSSQL Server

tags:
- clean
- archive
---
You may spend much of your time making sure table data is readily available, but disk space is a finite resource that can cost you your job if not properly monitored.
Businesses usually have a required data retention period with anything beyond that being fair game, but what if all your data happens to reside in a production environment that cannot withstand heavy maintenance?

Assuming the table you want to clean up contains a date column, the below stored procedure will allow you to delete only a **single day’s data per transaction**.
Create the log table, specify the number of days you want to retain, an *optional* where clause filter, and you're good to go.

Don't trust it? Set the `@test_only` parameter to `'Y'` to force the stored procedure to ONLY print commands that it would otherwise run.
Errors will be sent to you via email if database mail is configured and an email address is specified.

**Example Usage**
{% highlight sql %}
exec incrementally_purge_rows
@percent_trans_log_limit = 75
,@days_to_retain = 958
,@transaction_limit = 6
,@source_database = 'SOURCE_DB'
,@source_schema = 'dbo'
,@source_table = 'source_table'
,@date_column_name = 'entry_date'
,@purge_filter = 'Type = ''AA'''
,@log_to_table = 'Y'
,@test_only = 'N'
,@email_error_to = 'me@email.com'
{% endhighlight %}

**Create Script**
{% highlight sql %}
/*
==============================================
Create "purge_history" table and run stored procedure
==============================================
*/
USE [master]
IF OBJECT_ID('[dbo].[purge_history]') IS NULL
BEGIN
CREATE TABLE [dbo].[purge_history](
    [source_database] VARCHAR(50) NULL,
    [source_schema]   VARCHAR(50) NULL,
    [source_table]    VARCHAR(50) NULL,
    [days_retained]   INT NULL,
    [retention_date]  DATETIME NULL,
    [purged_rows]     INT NULL,
    [purge_date]      DATETIME NULL,
    [purge_statement] VARCHAR(4000) NULL
)
END
GO

IF OBJECT_ID('[dbo].[incrementally_purge_rows]') IS NOT NULL DROP PROCEDURE [dbo].[incrementally_purge_rows]
GO

CREATE PROCEDURE [dbo].[incrementally_purge_rows]
    @percent_trans_log_limit INT = 100
    ,@days_to_retain         INT
    ,@transaction_limit      INT = 1000
    ,@source_database        VARCHAR(50)
    ,@source_schema          VARCHAR(50)
    ,@source_table           VARCHAR(50)
    ,@date_column_name       VARCHAR(50)
    ,@purge_filter           VARCHAR(200)
    ,@log_to_table           CHAR(1) = 'Y'
    ,@test_only              CHAR(1) = 'N'
    ,@email_error_to         VARCHAR(50) = ''

AS
SET NOCOUNT ON;

/*
INCREMENTALLY PURGE ROWS FROM ONE TABLE TO ANOTHER BASED UPON RETENTION PERIOD

Requirements:
====================
Source table must contain a date column to determine if the row is outside the retention period.

Required Parameters:
====================
@days_to_retain        Retention period that date column will be checked against.
@source_database       Name of database for table that rows will be purged from.
@source_schema         Name of schema for table that rows will be purged from.
@source_table          Name of table that rows will be purged from.
@date_column_name      Name of column that contains row entry date.
@purge_filter          Declared in 'WHERE' clause to filter purge statement.
                       Set to 'False' to disable.

Optional Parameters:
====================
@percent_trans_log_limit    If used transaction log space is above limit, do not begin another purge.
                            DEFAULT = 100. Set to 100 to disable limit.
@transaction_limit          Max purge iterations for procedure. Default is 1000.
@log_to_table               Log results to included purge_history table. Set to 'N' to disable.
@test_only                  Will only print results of test run if set to 'Y'.
@email_error_to             Email address to receive error alerts

Example:
====================
exec incrementally_purge_rows
@percent_trans_log_limit = 75
,@days_to_retain = 958
,@transaction_limit = 6
,@source_database = 'SOURCE_DB'
,@source_schema = 'dbo'
,@source_table = 'source_table'
,@date_column_name = 'entry_date'
,@purge_filter = 'Type = ''AA'''
,@log_to_table = 'Y'
,@test_only = 'N'
,@email_error_to = 'me@email.com'

*/

BEGIN

DECLARE 
    @days_retained           INT
    ,@percent_trans_log_used INT
    ,@retention_date         DATE = NULL
    ,@retention_date_stmt    NVARCHAR(4000) = NULL
    ,@purge_stmt             NVARCHAR(4000) = NULL
    ,@and_purge_filter       VARCHAR(200) = ''
    ,@where_purge_filter     VARCHAR(200) = ''
    ,@transaction_counter    INT = 0
    ,@purge_count            INT
    ,@log_to_table_stmt      NVARCHAR(4000)
    ,@email_last_stmt        NVARCHAR(4000)

IF LOWER(@purge_filter) = 'false'
BEGIN
  SET @purge_filter = '';
END
ELSE
BEGIN
  SET @and_purge_filter = N' AND ' + @purge_filter;
  SET @where_purge_filter = N' WHERE ' + @purge_filter;
END

SELECT @percent_trans_log_used = MAX([Percent Log Used])
  FROM (SELECT *
          FROM sys.dm_os_performance_counters
         WHERE counter_name IN ('Percent Log Used')
           AND instance_name = @source_database) AS Src
PIVOT(MAX(cntr_value) FOR counter_name IN ([Percent Log Used])) AS pvt
WHERE [Percent Log Used] IS NOT NULL

SET @retention_date_stmt = N'SELECT @retention_date = min(' + @date_column_name + ') from ' 
  + @source_database + '.' + @source_schema + '.' + @source_table + @where_purge_filter
SET @email_last_stmt = @retention_date_stmt

EXEC sp_executesql @retention_date_stmt
    ,N'@retention_date DATE OUTPUT'
    ,@retention_date = @retention_date OUTPUT;

SET @days_retained = DATEDIFF(DAY, @retention_date, getdate())
PRINT 'Old retention date: ' + CAST(@retention_date AS VARCHAR(20))
PRINT 'Days retained: ' + CAST(@days_retained AS VARCHAR(10)) + '/' + CAST(@days_to_retain AS VARCHAR(10))
    + ', Percent log used: ' + CAST(@percent_trans_log_used AS VARCHAR(3))

WHILE
((@percent_trans_log_used <= @percent_trans_log_limit) OR @percent_trans_log_limit = 100)
AND
(@days_retained > @days_to_retain)
AND
(@transaction_counter < @transaction_limit)
BEGIN
  PRINT ''
  BEGIN TRANSACTION
  BEGIN TRY

    SET @purge_stmt = N'
    DELETE ' + @source_database + '.' + @source_schema + '.' + @source_table + '
    WHERE CONVERT(DATE, ' + @date_column_name + ') <= ''' + CAST(@retention_date AS VARCHAR(20)) + '''' 
	+ @and_purge_filter
    SET @email_last_stmt = @purge_stmt

    IF @test_only = 'N'
    BEGIN
      EXEC sp_executesql @purge_stmt;
    END
    ELSE
    BEGIN
      PRINT '*test only*'
      PRINT 'Statement to purge rows:'
      PRINT @purge_stmt
    END

    SET @purge_count = @@ROWCOUNT

  /*
  ==============================================
  Purge from source complete.
  ==============================================
  */

  SET @retention_date_stmt = N'SELECT @retention_date = min(' + @date_column_name + ') from ' 
  + @source_database + '.' + @source_schema + '.' + @source_table + @where_purge_filter
  SET @email_last_stmt = @retention_date_stmt

  EXEC sp_executesql @retention_date_stmt
      ,N'@retention_date DATE OUTPUT'
      ,@retention_date = @retention_date OUTPUT;

  SET @days_retained = DATEDIFF(DAY, @retention_date, getdate())
  
  --Replace single quotes before log insert
  SET @purge_stmt =  REPLACE(@purge_stmt, '''', '''''') 
  SET @log_to_table_stmt = N'
  INSERT INTO master.dbo.purge_history (
    [source_database],
    [source_schema],
    [source_table],
    [days_retained],
    [retention_date],
    [purged_rows],
    [purge_date],
    [purge_statement])
  VALUES (
    ' + CHAR(39) + @source_database + CHAR(39) + ',
    ' + CHAR(39) + @source_schema + CHAR(39) + ',
    ' + CHAR(39) + @source_table + CHAR(39) + ',
    CAST(' + CHAR(39) + CAST(@days_retained AS NVARCHAR(23)) + CHAR(39) + ' AS INT),
    CAST(' + CHAR(39) + CAST(@retention_date AS NVARCHAR(23)) + CHAR(39) + ' AS DATE),
    CAST(' + CHAR(39) + CAST(@purge_count AS NVARCHAR(23)) + CHAR(39) + ' AS INT),
    CAST(' + CHAR(39) + CONVERT(NVARCHAR(23), getdate(), 121) + CHAR(39) + ' AS DATETIME),
    ' + CHAR(39) + @purge_stmt + CHAR(39) + ')
  '
  SET @email_last_stmt = @log_to_table_stmt

  IF @log_to_table = 'Y'
  BEGIN
    EXECUTE dbo.sp_executesql @log_to_table_stmt
  END
  ELSE
  BEGIN
    PRINT 'Statement to log to purge_history:'
    PRINT @log_to_table_stmt
  END

  /*
  ==============================================
  Log activity to purge_history table complete.
  ==============================================
  */
       
  COMMIT
  PRINT 'Commit successful'
   
  END TRY
       
  BEGIN CATCH
    PRINT 'Begin rollback'
    ROLLBACK
    PRINT 'Rollback successful'
    SET @purge_count = 0;

    EXEC msdb.dbo.sp_send_dbmail
         @recipients = @email_error_to,
         @subject = 'Purge Failure',
         @body = @email_last_stmt
  END CATCH

  PRINT 'New retention date: ' + CAST(@retention_date AS VARCHAR(20))

  SELECT @percent_trans_log_used = MAX([Percent Log Used])
  FROM (SELECT *
          FROM sys.dm_os_performance_counters
         WHERE counter_name IN ('Percent Log Used')
           AND instance_name = @source_database) AS Src
  PIVOT(MAX(cntr_value) FOR counter_name IN ([Percent Log Used])) AS pvt
  WHERE [Percent Log Used] IS NOT NULL

  SET @transaction_counter = @transaction_counter + 1

  PRINT 'Days retained: ' + CAST(@days_retained AS VARCHAR(10)) + '/' + CAST(@days_to_retain AS VARCHAR(10)) 
  + ', Percent log used: ' + CAST(@percent_trans_log_used AS VARCHAR(3))

  PRINT 'Transaction counter: ' + CAST(@transaction_counter AS VARCHAR(10))
END

END

GO
{% endhighlight %}

