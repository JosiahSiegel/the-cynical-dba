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
Values that are under 8,000 bytes will remain in-row. 
Values that are greater than 8,000 bytes will be allocated as `LOB_DATA` and may incur a read/write penalty.

For example, it is ideal to use the `VARCHAR(n)` data type, as `VARCHAR(MAX)` permites 2,147,483,647 bytes (2 GB).
Additionally, the **total row size** should not exceed 8,060 bytes if you do not want any values pushed off-row.
Keep this in mind when specifying column data lengths.

The following scripts can be used to compare page allocation &amp; access performance:

# Non-LOB data (`VARCHAR(8000)`)
{% highlight sql %}
SET STATISTICS IO OFF;
USE [TestDB]
GO

CREATE TABLE dbo.LobTest
  (
    ID   INT IDENTITY(1,1) PRIMARY KEY,
    VMAX VARCHAR(MAX)
  )
GO

-- Insert non-LOB data
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

>| alloc_unit  | page_count | logical_reads |
>|:------------|:----------:|--------------:|
>| IN_ROW_DATA | 9999       | **10037**     |

# LOB data (`VARCHAR(8001)`)
{% highlight sql %}
-- Update to LOB data
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

>| alloc_unit  | page_count | logical_reads |
>|:------------|:----------:|--------------:|
>| IN_ROW_DATA | 52         | 54            |
>| LOB_DATA    | 9999       | **29609**     |

# Conclusion

With non-LOB data (<= 8,000 bytes), the output resulted in 10,037 logical reads. 
Once the values were updated to 8,001 bytes, they were pushed off-row onto a `LOB_DATA` allocation unit with just a 16 byte text pointer stored in-row. 
The LOB data resulted in significantly more logical reads (29,609) to access data that was just 1 byte larger.

