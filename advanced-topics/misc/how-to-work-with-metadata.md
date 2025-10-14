在 Postgres 中处理元数据（关于数据的数据）时，值得使用以下参考文档：

- [System  Catalogs](https://www.postgresql.org/docs/current/catalogs.html)
- [System  Views](https://www.postgresql.org/docs/current/views.html)
- [The Cumulative Statistics SystemThe](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [The Information  Schema](https://www.postgresql.org/docs/current/information-schema.html)

没有必要重复这里文档中的材料。相反，让我们专注于一些技巧和原则，这些技巧和原则可以让你的工作更有效率。我们将介绍以下主题：

- `::oid, ::regclass`
- `\? and ECHO_HIDDEN`
- Performance
- `INFORMATION_SCHEMA`
- `pg_stat_activity` is not a table

# `::oid, ::regclass`

在 Postgres 术语中，表、索引、视图、物化视图都称为"relations"。关于它们的元数据可以通过多种方式查看，但"central"位置是pg_class系统目录。换句话说，这是一个表，它存储有关所有表、索引等的信息。它有两个键：

- PK: oid - a number ( [OID object identifier](https://postgresql.org/docs/current/datatype-oid.html))
- UK：一对列(relname, relnamespace)关系名称和模式的OID。

要记住的技巧：OID 可以快速转换为关系名称，反之亦然，使用类型转换为 `oid` 和 `regclass` 数据类型。

名为 `t1` 的表的简单示例：

```sql
nik=# select 't1'::regclass;
 regclass
----------
 t1
(1 row)

nik=# select 't1'::regclass::oid;
  oid
-------
 74298
(1 row)

nik=# select 74298::regclass;
 regclass
----------
 t1
(1 row)

```

因此，没有必要从 `select oid from pg_class where relname = ...` – 只需记住 `::regclass` 和 `::oid` 。

# ` ? 和 ECHO_HIDDEN `

`psql` 的 `\?` 命令至关重要 , 这是查找所有命令描述的方法。例如：

```sql
\d[S+]                 list tables, views, and sequences
```

"describing"命令隐式生成一些 SQL , 显示它们会很有帮助。为此，我们首先需要打开 ECHO_HIDDEN ：

– 或者在启动 `psql` 时使用选项 -E。然后我们就可以开始显示了：

```sql
nik=# \d t1
/********* QUERY **********/
SELECT c.oid,
 n.nspname,
 c.relname
FROM pg_catalog.pg_class c
    LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relname OPERATOR(pg_catalog.~) '^(t1)$' COLLATE pg_catalog.default
 AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 2, 3;
/**************************/

[... + more queries to get info about "t1" ...]

```

检查这些查询可以帮助构建各种工具来处理元数据。

# Performance

在某些情况下，元数据查询可能很重、很慢。如果是这样，该怎么办：

1. 考虑缓存以减少元数据查询的频率和必要性。
2. 检查目录膨胀。例如，由于频繁的 DDL、使用临时表等，pg_class可能会变得臃肿。不幸的是，在这种情况下，需要 VACUUM FULL（pg_repack无法重新打包系统目录）。如果需要，不要忘记 Postgres 中零停机时间 DDL 的黄金法则--[使用低lock_timeout和重试](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)。

# INFORMATION_SCHEMA

系统目录和视图是查询表和索引元数据的"native"方式，但不是标准的。标准方式称为 INFORMATION_SCHEMA，Postgres 支持它遵循 SQL标准：[文档](https://www.postgresql.org/docs/current/information-schema.html) `INFORMATION_SCHEMA` 使用什么：

- 使用信息架构进行简单的跨数据库兼容元数据查询。
- 使用本机系统目录进行更复杂的特定于 Postgres 的查询，或者当您需要详细的内部信息时。

# pg_stat_activity 不是表

重要的是要记住，在查询元数据时，您可能会处理一些行为不正常的东西，即使它看起来是这样。

例如，当您从pg_stat_activity读取记录时，您不会处理表数据的一致快照：读取第一行和理论上最后一行是在不同的时间点生成的，您可能会看到未同时运行的查询。

这种现象也解释了为什么 `select now() - query_start from pg_stat_activity;` 可能会给你负值：`now()` 在事务开始时执行一次，并且无论你调用多少次，都不会在事务中改变它的值。

要获得精确的时间间隔，请改用 `clock_timestamp()select clock_timestamp() - query_start from pg_stat_activity; ` ）。

