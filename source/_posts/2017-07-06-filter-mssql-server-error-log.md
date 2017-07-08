---
layout: post
title: "Query MSSQL Server Error Log"
permalink: query-mssql-server-error-log
date: 2017-07-05 19:54:20
comments: true
description: "Complex filtering of MSSQL Server error log through T-SQL."
keywords: "MSSQLServer"
category: MSSQLServer Log

tags:

---


Insert latest error log into table variable for additional joins or complex filtering:

{% highlight sql %}
DECLARE @temp_error_log TABLE
(
 date datetime,
 process varchar(50) null,
 text varchar(MAX) null
)

INSERT INTO @temp_error_log
EXEC sp_readerrorlog 0, 1, 'Login failed' 

SELECT * FROM @temp_error_log 
ORDER BY date DESC
{% endhighlight %}

