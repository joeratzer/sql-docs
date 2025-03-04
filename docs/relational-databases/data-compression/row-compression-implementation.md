---
title: "Row compression implementation"
description: Learn how the SQL Server Database Engine implements row compression to help you plan the storage space that you need for your data.
author: WilliamDAssafMSFT
ms.author: wiassaf
ms.reviewer: randolphwest
ms.date: 08/21/2023
ms.service: sql
ms.subservice: performance
ms.topic: conceptual
helpviewer_keywords:
  - "compression [SQL Server], row"
  - "row compression [Database Engine]"
monikerRange: "=azuresqldb-current || >=sql-server-2016 || >=sql-server-linux-2017 || =azuresqldb-mi-current"
---
# Row compression implementation

[!INCLUDE [SQL Server Azure SQL Database Azure SQL Managed Instance](../../includes/applies-to-version/sql-asdb-asdbmi.md)]

This article summarizes how [!INCLUDE [ssDE](../../includes/ssde-md.md)] implements row compression. This summary provides basic information to help you plan the storage space that you need for your data.

Enabling compression only changes the physical storage format of the data that is associated with a data type but not its syntax or semantics. Application changes aren't required when one or more tables are enabled for compression. The new record storage format has the following main changes:

- It reduces the metadata overhead that is associated with the record. This metadata is information about columns, their lengths and offsets. In some cases, the metadata overhead might be larger than the old storage format.

- It uses variable-length storage format for numeric types (for example **integer**, **decimal**, and **float**) and the types that are based on numeric (for example **datetime** and **money**).

- It stores fixed character strings by using variable-length format by not storing the blank characters.

> [!NOTE]  
> `NULL` and `0` values across all data types are optimized and take no bytes.

## How row compression affects storage

The following table describes how row compression affects the existing types in [!INCLUDE [ssNoVersion](../../includes/ssnoversion-md.md)] and [!INCLUDE [ssazure-sqldb](../../includes/ssazure-sqldb.md)]. The table doesn't include the savings that can be achieved by using page compression.

| Data type | Is storage affected? | Description |
| --- | --- | --- |
| **tinyint** | No | 1 byte is the minimum storage needed. |
| **smallint** | Yes | If the value fits in 1 byte, only 1 byte is used. |
| **int** | Yes | Uses only the bytes that are needed. For example, if a value can be stored in 1 byte, storage takes only 1 byte. |
| **bigint** | Yes | Uses only the bytes that are needed. For example, if a value can be stored in 1 byte, storage takes only 1 byte. |
| **decimal** | Yes | Uses only the bytes that are needed, regardless of the precision specified. For example, if a value can be stored in 3 bytes, storage takes only 3 bytes. The storage footprint is exactly the same as the **vardecimal** storage format. |
| **numeric** | Yes | Uses only the bytes that are needed, regardless of the precision specified. For example, if a value can be stored in 3 bytes, storage takes only 3 bytes. The storage footprint is exactly the same as the **vardecimal** storage format. |
| **bit** | Yes | The metadata overhead brings this to 4 bits. |
| **smallmoney** | Yes | Uses the integer data representation by using a 4-byte integer. Currency value is multiplied by 10,000 and the resulting integer value is stored by removing any digits after the decimal point. This type has a storage optimization similar to that for integer types. |
| **money** | Yes | Uses the integer data representation by using an 8-byte integer. Currency value is multiplied by 10,000 and the resulting integer value is stored by removing any digits after the decimal point. This type has a larger range than **smallmoney**. This type has a storage optimization similar to that for integer types. |
| **float** | Yes | Least significant bytes with zeros aren't stored. **float** compression is applicable mostly for nonfractional values in mantissa. |
| **real** | Yes | Least significant bytes with zeros aren't stored. **real** compression is applicable mostly for nonfractional values in mantissa. |
| **smalldatetime** | No | Uses the integer data representation by using two 2-byte integers. It is the number of days since `1900-01-01`. There is no row compression benefit to the date portion of **smalldatetime**.<br /><br />The time is the number of minutes since midnight. Time values that are slightly past 4AM start to use the second byte.<br /><br />If a **smalldatetime** is only used to represent a date (a common case), the time is `0.0`. Compression saves 2 bytes by storing the time in most significant byte format for row compression. |
| **datetime** | Yes | Uses the integer data representation by using two 4-byte integers. The integer value represents the number of days with base date of `1900-01-01`. The first 2 bytes can represent up to the year `2079`. Compression can always save 2 bytes here until that point. Each integer value represents 3.33 milliseconds. Compression exhausts the first 2 bytes in first five minutes and needs the fourth byte after 4PM. Therefore, compression can save only 1 byte after 4PM. When **datetime** is compressed like any other integer, compression saves 2 bytes in the date. |
| **date** | No | Uses the integer data representation by using 3 bytes. This represents the date from `0001-01-01`. For contemporary dates, row compression uses all 3 bytes. This achieves no savings. |
| **time** | No | Uses the integer data representation by using 3 - 6 bytes. There are various precisions that start with 0 to 9 that can take 3 - 6 bytes. Compressed space is used as follows:<br /><br />**Precision = 0. Bytes = 3**. Each integer value represents a second. Compression can represent time up to 6PM by using 2 bytes, potentially saving 1 byte.<br /><br />**Precision = 1. Bytes = 3**. Each integer value represents 1/10 seconds. Compression uses the third byte before 2AM. Results in little savings.<br /><br />**Precision = 2. Bytes = 3**. Similar to the previous case, it's unlikely to achieve savings.<br /><br />**Precision = 3. Bytes = 4**. Because the first 3 bytes are taken by 5AM, this option achieves little savings.<br /><br />**Precision = 4. Bytes = 4**. The first 3 bytes are taken in the first 27 seconds. No savings are expected.<br /><br />**Precision = 5, Bytes = 5**. The fifth byte will be used after 12-noon.<br /><br />**Precision = 6 and 7, Bytes = 5**. Achieves no savings.<br /><br />**Precision = 8, Bytes = 6**. The sixth byte will be used after 3AM.<br /><br />There is no change in storage for row compression. Overall, not much savings can be expected from compressing the **time** data type. |
| **datetime2** | Yes | Uses the integer data representation by using 6 - 9 bytes. The first 4 bytes represent the date. The bytes taken by the time depend on the precision of the time that is specified.<br /><br />The integer value represents the number of days since `0001-01-01` with an upper bound of 12/31/9999. To represent a date in the year 2005, compression takes 3 bytes.<br /><br />There are no savings on time because it allows for 2 - 4 bytes for various time precisions. Therefore, for one-second time precision, compression uses 2 bytes for time, which takes the second byte after 255 seconds. |
| **datetimeoffset** | Yes | Resembles **datetime2**, except that there are 2 bytes of time zone of the format (`HH:mm`).<br /><br />Like **datetime2**, compression can save 2 bytes.<br /><br />For time zone values, the `mm` value might be `0` for most cases. Therefore, compression can possibly save 1 byte.<br /><br />There are no changes in storage for row compression. |
| **char** | Yes | Trailing padding characters are removed. The [!INCLUDE [ssDE](../../includes/ssde-md.md)] inserts the same padding character regardless of the collation that is used. |
| **varchar** | No | No effect. |
| **text** | No | No effect. |
| **nchar** | Yes | Trailing padding characters are removed. The [!INCLUDE [ssDE](../../includes/ssde-md.md)] inserts the same padding character regardless of the collation that is used. |
| **nvarchar** | No | No effect. |
| **ntext** | No | No effect. |
| **binary** | Yes | Trailing zeros are removed. |
| **varbinary** | No | No effect. |
| **image** | No | No effect. |
| **cursor** | No | No effect. |
| **timestamp** / **rowversion** | Yes | Uses the integer data representation by using 8 bytes. There is a timestamp counter that is maintained for each database, and its value starts from 0. This can be compressed like any other integer value. |
| **sql_variant** | No | No effect. |
| **uniqueidentifier** | No | No effect. |
| **table** | No | No effect. |
| **xml** | No | No effect. |
| User-defined types | No | This is represented internally as **varbinary**. |
| FILESTREAM | No | This is represented internally as **varbinary**. |

## Next steps

- [Data compression](data-compression.md)
- [Page compression implementation](page-compression-implementation.md)
- [Unicode compression implementation](unicode-compression-implementation.md)
