在Postgres中，存储方面的列顺序重要吗？

答案是yes。让我们考虑一个例子（像往常一样，我的建议是不要使用int4 PK，但这里是出于分享目的）：

```sql
create table t(
  id int4 primary key,
  created_at timestamptz,
  is_public boolean,
  modified_at timestamptz,
  verified boolean,
  published_at timestamptz,
  score int2
);

insert into t
select
  i,
  clock_timestamp(),
  true,
  clock_timestamp(),
  true,
  clock_timestamp(),
  0
from generate_series(1, 1000000) as i;

vacuum analyze t;

```

查看大小：

```sql
nik=# \dt+ t
                   List of relations
 Schema | Name | Type  |  Owner   | Size  | Description
--------+------+-------+----------+-------+-------------
 public | t    | table | postgres | 81 MB |
(1 row)

```

现在，让我们使用来自 [postgres_dba](https://github.com/NikolayS/postgres_dba) 的报告p1（假设它已安装）：

```sql
:dba

Type your choice and press <Enter>:
p1
 Table | Table Size | Comment |    Wasted *     |  Suggested Columns Reorder
-------+------------+---------+-----------------+-----------------------------
 t     | 81 MB      |         | ~23 MB (28.40%) | is_public, score, verified +
       |            |         |                 | id, created_at, modified_at+
       |            |         |                 | published_at
(1 row)

```

--报告声称我们只需更改列顺序即可节省~28%的磁盘空间。请注意，这是一个估计。

让我们检查优化的顺序：

```sql
drop table t;

create table t(
  is_public boolean,
  verified boolean,
  score int2,
  id int4 primary key,
  created_at timestamptz,
  modified_at timestamptz,
  published_at timestamptz
);

insert into t
select
  true,
  true,
  0::int2,
  i::int4,
  clock_timestamp(),
  clock_timestamp(),
  clock_timestamp()
from generate_series(1, 1000000) as i;

vacuum analyze t;

```

查看大小：

```sql
nik=# \dt+ t
                   List of relations
 Schema | Name | Type  |  Owner   | Size  | Description
--------+------+-------+----------+-------+-------------
 public | t    | table | postgres | 57 MB |
(1 row)

```

-我们节省了~30%，非常接近预期（57/81~=0.7037）。

postgres_dba的报告p1不再显示潜在的节约：

```sql
  Type your choice and press <Enter>:
  p1
   Table | Table Size | Comment | Wasted * | Suggested Columns Reorder
  -------+------------+---------+----------+---------------------------
   t     | 57 MB      |         |          |
  (1 row)

```

为什么这些更改减少了我们的表的大小

在Postgres中，存储系统可以向列值添加对齐填充。如果数据类型的自然alignmentrequirement大于值的大小，Postgres可以用零填充该值直到对齐边界。例如，当列的值小于8字节，后跟需要8字节对齐的值时，Postgres会填充第一个值以在8字节边界上对齐。这有助于确保内存中的值与特定硬件架构的CPU字边界一致，从而提高性能。

例如，一行（ `int4` ，`timestamptz` ）占用16个字节：

- `int4` 的4个字节
- 4个零对齐到8字节
- `timestamptz` 的8个字节

有些人更喜欢倒置的方法：首先我们从16字节和8字节的列开始，然后继续进行较小的列。无论如何，将具有 `VARLENA` 类型的列（`text`、`varchar`、`json`、`jsonb`、数组）放在最后是有意义的。

有关此主题的阅读文章：

- [StackOverflow answer by Erwin Brandstetter](https://stackoverflow.com/questions/2966524/calculating-and-saving-space-in-postgresql/7431468#7431468)
- [Ordering Table Columns in PostgreSQL (GitLab)](https://docs.gitlab.com/development/database/ordering_table_columns/)
- Docs: [Table Row Layout](https://www.postgresql.org/docs/current/storage-page-layout.html#STORAGE-TUPLE-LAYOUT)

