---
layout: post
title: "Move MSSQL Server Database Files"
permalink: move-mssql-server-database-files
date: 2017-07-06 12:21:44
comments: true
description: "Move MSSQL Server Database Files"
keywords: ""
categories:

tags:

---

You can either read all about moving databases [here][msdn_move_db], or simply follow the 4 steps below:

### Step 1
 * Get current path of data &amp; log files

{% highlight sql %}
Use master;
GO

SELECT name, physical_name AS CurrentLocation, state_desc
FROM sys.master_files
WHERE database_id = DB_ID(N'DatabaseName')
GO
{% endhighlight %}

### Step 2
 * Take database offline
 * `WITH ROLLBACK IMMEDIATE` Tells SQL Server to cancel any pending transactions and to rollback immediately.  Without this termination clause, the SET OFFLINE will wait until all tasks are completed. 

{% highlight sql %}
ALTER DATABASE DatabaseName SET OFFLINE WITH ROLLBACK IMMEDIATE;
GO
{% endhighlight %}

### Step 3
 * Point data &amp; log files to **new** path

{% highlight sql %}
ALTER DATABASE DatabaseName
MODIFY FILE (NAME= [date_file], FILENAME = 'C:\path\to\point\to\data.mdf');
ALTER DATABASE DatabaseName
MODIFY FILE (NAME= [log_file], FILENAME = 'C:\path\to\point\to\log.ldf');
{% endhighlight %}

### Step 4
 * **After physical files have been moved**, put database back online

{% highlight sql %}
ALTER DATABASE DatabaseName SET ONLINE;
{% endhighlight %}

[msdn_move_db]:      https://msdn.microsoft.com/en-us/library/ms345483.aspx
