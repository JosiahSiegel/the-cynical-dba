---
layout: post
title: "Query MSSQL Server error log"
permalink: query-mssql-server-error-log
date: 2017-07-05 19:54:20
comments: true
description: "Complex filtering of MSSQL Server error log through T-SQL."
keywords: "MSSQL Server"
category: MSSQL Server Log

tags:

---

While it's possible to read the SQL Server error log with T-SQL via the `sp_readerrorlog` stored procedure, it does not allow for joins or advanced filtering if more complex analysis is needed.
Inserting the output of the error log into a table variable resolves this, as the output can now be manipulated like any other table:

{% highlight sql %}
DECLARE @temp_error_log TABLE
(
 date datetime,
 process varchar(50) null,
 text varchar(MAX) null
)

INSERT INTO @temp_error_log
EXEC sp_readerrorlog 0, 1, '<log filter>'
--Select from error log output
SELECT * FROM @temp_error_log 
ORDER BY date DESC
{% endhighlight %}

