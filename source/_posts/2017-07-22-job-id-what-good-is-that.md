---
layout: post
title: "Job ID? What good is that?"
permalink: job-id-what-good-is-that
date: 2017-07-22 16:57:58
comments: true
description: "Job ID? What good is that?"
keywords: "MSSQL Server"
category: MSSQL Server
---
You will inevitably find yourself in many situations where you need to figure out what’s hogging server resources.

After querying against dynamic management view **[sys.dm_exec_sessions]**, you notice there's a query running against your heaviest table with a **[program_name]** containing `Job 0xFA732G27919DA5689E3DD97061E55A53` or some other combination that is equally unintelligible.

The number vomit above is the ID of the job that is running that troublesome query.
Luckily, it’s possible to decipher that mess into the only thing that is helpful to you, the job name:

{% highlight sql %}
SELECT name
FROM msdb.dbo.sysjobs
WHERE job_id = CAST(0xFA732G27919DA5689E3DD97061E55A53 AS UNIQUEIDENTIFIER)
{% endhighlight %}

Alternatively, you can use the [Quick Analysis Query](/quick-mssql-server-analysis) to include the job name right away.

