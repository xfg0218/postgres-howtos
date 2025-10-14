在Postgres中，所有表都有隐藏的系统列；ctid是其中之一。阅读它，我们可以看到元组的物理分配（元组=行物理版本）、页码和里面的偏移量：

```sql
nik=# create table t0 as select 1 as id;
SELECT 1

nik=# select ctid, id from t0;
 ctid  | id
-------+----
 (0,1) |  1
(1 row)

```

👉 第0页，位置1。

单个PostgreSQL页面，默认为8 KiB，可以通过查看block_size进行检查：

```sql
nik=# show block_size;
 block_size
------------
 8192
(1 row)

```

一个页面可以容纳多少个元组？让我们看看：

```sql
nik=# create table t0 as select i
from generate_series(1, 1000) as i;
SELECT 1000

nik=# select count(*)
from t0
where (ctid::text::point)[0] = 0;
 count
-------
   226
(1 row)

nik=# select pg_column_size(i) from t0 limit 1;
 pg_column_size
----------------
              4
(1 row)

```

👉 如果我们使用4字节的数字，那么它是226个元组。这里我使用 `(ctid::text::point)[0]` 将 `ctid` 值转换为"point"以获取其第一个组件（页码）。

如果我们使用2字节数字，或者说1字节 `boolean` 值（是的，boolean需要1字节，而不是1位），数字是相同的：

```sql
nik=# drop table t0;
DROP TABLE

nik=# create table t0 as select true
from generate_series(1, 1000) as i;
SELECT 1000

nik=# select count(*)
  from t0
  where (ctid::text::point)[0] = 0;
 count
-------
   226
(1 row)

```

为什么又是226？问题是，值的大小在这里并不重要；它只需要小于或等于8字节。对于每一行，对齐填充都会添加零，因此每行总是有8个字节。数学：

```sql
floor((8192 - 24) / (4 + 24 + 8)) = 226
```

👉 我们在这里统计的内容：

1. 单个24字节的页眉（`PageHeaderData`）。
2. 每个元组的N个指针-每个元组4个字节（`ItemIdData`）。
3. N个元组标头-每个23个字节，填充到24个（`HeapTupleHeaderData`）。
4. N个元组值-如果<=8个字节，则将其填充为8个字节。

定义结构的源代码（用于 [PG16](https://github.com/postgres/postgres/blob/REL_16_STABLE/src/include/storage/bufpage.h) ）。

**我们可以容纳更多的元组吗？**

答案是肯定的。Postgres允许没有列的表（!）在这种情况下，数学是：

```sql
floor((8192 - 24) / (4 + 24)) = 291
```

让我们看看（注意 `SELECT` 子句中的空列列表）：

```sql
nik=# create table t0 as select
from generate_series(1, 1000) as i;
SELECT 1000

nik=# select count(*)
from t0
where (ctid::text::point)[0] = 0;
 count
-------
   291
(1 row)

```
