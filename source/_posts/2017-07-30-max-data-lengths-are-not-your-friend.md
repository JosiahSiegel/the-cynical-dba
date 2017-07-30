---
layout: post
title: "MAX data lengths are not your friend"
permalink: max-data-lengths-are-not-your-friend
date: 2017-07-30 15:23:31
comments: true
description: "MAX data lengths are not your friend"
keywords: "MSSQLServer"
category: MSSQLServer

tags:

---
If you want to maintain predictable query performance, shy away from data lengths that may force data to be pushed off-row.
Values that are under 8,000 bytes will remain on-row. 
Any rows that are greater than 8,000 bytes will be allocated as `LOB_DATA` and may incur a read/write penalty. 
This is due to an increase in the number of pages accessed.

To help enforce predictable performance, it is ideal to instead use `VARCHAR(n)`, removing the possibility of values being pushed to `LOB_DATA`.
Additionally, the **total row size** should not exceed 8,060 bytes if you do not want any values pushed off-row.
Keep this in mind when specifying column lengths.

The following scripts can be used to compare page allocation access:

# Non-LOB rows (`VARCHAR(8000)`)
{% highlight sql %}
SET STATISTICS IO OFF;
USE [TestDB]
GO

IF OBJECT_ID('[dbo].[LobTest]') IS NOT NULL
BEGIN
  DROP TABLE [LobTest]
END
GO
CREATE TABLE dbo.LobTest
  (
    ID   INT IDENTITY(1,1) PRIMARY KEY,
    VMAX VARCHAR(MAX)
  )  ON [PRIMARY]
GO

-- Insert non-LOB rows
SET NOCOUNT ON;
DECLARE @i INT = 1
WHILE (@i < 10000)
BEGIN
  INSERT INTO [LobTest] ([VMAX]) VALUES (REPLICATE('U', 8000))
  SET @i += 1;
END

-- Confirm page allocation
SELECT 
  alloc_unit_type_desc, 
  page_count 
FROM 
  sys.dm_db_index_physical_stats 
  (DB_ID(),OBJECT_ID(N'LobTest'), NULL, NULL, NULL)

-- Confirm page access count
SET STATISTICS IO ON;
SELECT * FROM [LobTest]
{% endhighlight %}

>### Non-LOB output
>
>| alloc_unit  | page_count | logical_reads | lob_logical_reads |
>|:------------|:----------:|:-------------:|------------------:|
>| IN_ROW_DATA | 9999       | **10037**     | 0                 |

# LOB rows (`VARCHAR(8001)`)
{% highlight sql %}
-- Update to LOB rows
SET STATISTICS IO OFF;
UPDATE [LobTest] SET [VMAX] = REPLICATE(CONVERT(VARCHAR(MAX),'O'), 8001)
GO

-- Reduce fragmentation
ALTER INDEX ALL ON LobTest REBUILD

-- Confirm page allocation
SELECT 
  alloc_unit_type_desc, 
  page_count 
FROM
  sys.dm_db_index_physical_stats 
  (DB_ID(),OBJECT_ID(N'LobTest'), NULL, NULL, NULL)

-- Confirm page access count
SET STATISTICS IO ON;
SELECT * FROM [LobTest]
{% endhighlight %}

>### LOB output
>
>| alloc_unit  | page_count | logical_reads | lob_logical_reads |
>|:------------|:----------:|:-------------:|------------------:|
>| IN_ROW_DATA | 52         | 54            | 0                 |
>| LOB_DATA    | 9999       | 0             | **29609**         |

# Conclusion

With non-LOB rows, the output resulted in 10,037 logical reads. 
The LOB rows resulted in 26,609 logical reads, as values were then stored in the `LOB_DATA` allocation unit.

