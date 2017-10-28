---
layout: post
title: "SQL Server 2016 phones home"
permalink: sql-server-2016-phones-home
date: 2017-09-24 22:41:43
comments: true
description: "SQL Server 2016 phones home"
keywords:
category: "MSSQL Server Security"

tags:

---
If you recently upgraded to SQL Server 2016, you may be in for a surprise when researching the new login **NT SERVICE\SQLTELEMETRY**. 
This login is part of a new user activity collection process that uploads directly to Microsoft. 
If this is not already concerning for you, chances are it will be for your auditors.

## “We are broadcasting what?...”

Ideally, your database environment does not connect to the outside world; but if that is not the case, the last thing you need is to explain to management that “*Surprise! We are now broadcasting database activity logs*”. 
Of course, the [official documentation]( https://support.microsoft.com/en-us/help/3153756/how-to-configure-sql-server-2016-to-send-feedback-to-microsoft) clarifies that the data collected is relatively harmless, but that means little when you are required to pre-approve all external communications in order to maintain compliance. 
Fortunately, Microsoft’s usage collection can be disabled to avoid these concerns all together, or for while you await approval.

## Take it slow

For any paid version of SQL Server, you can disable Microsoft’s collection of your activity by opening the “SQL Server Error and Usage Reporting” application and unchecking the options to send usage and error reports. 
If auditors or management are looking for confirmation that this process is disabled, I find it convenient to send them the output of the registry values that confirm just that:

{% highlight sql %}
DECLARE @RegOutput TABLE (
        [Name]          VARCHAR(100),
        [Value]         VARCHAR(400)
)

INSERT INTO @RegOutput
EXEC [master].[dbo].[xp_regread] @rootkey='HKEY_LOCAL_MACHINE',
@key='Software\Microsoft\Microsoft SQL Server\MSSQL13.<instance>\CPE',
@value_name='CustomerFeedback'

INSERT INTO @RegOutput
EXEC [master].[dbo].[xp_regread] @rootkey='HKEY_LOCAL_MACHINE',
@key='Software\Microsoft\Microsoft SQL Server\MSSQL13.<instance>\CPE',
@value_name='EnableErrorReporting'

SELECT * FROM @RegOutput
{% endhighlight %}

It feels like every few weeks we learn of a newly compromised company. 
Error logs can be a critical component in assisting those that are attempting to gain unauthorized access. 
When given the option, why make it any easier for them by potentially exposing data related to your company’s internal operations?


