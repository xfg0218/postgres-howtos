# 如何读取 LSN 值

LSN – 日志序列号，指向预写日志 （WAL） 中某个位置的指针。了解它以及如何使用它对于处理物理和逻辑复制、备份、恢复非常重要。Postgres 文档：

- [WAL Internals](https://www.postgresql.org/docs/current/wal-internals.html)
- [pg_lsn Type](https://www.postgresql.org/docs/current/datatype-pg-lsn.html)

LSN 是一个 8 字节（64 位）值（ [源代码](https://gitlab.com/postgres/postgres/blob/4f2994647ff1e1209829a0085ca0c8d237dbbbb4/src/include/access/xlogdefs.h#L17) ）。它可以以 `A/B` 的形式表示（更具体地说，`A/BBbbbbbb` 见下文），其中 `A` 和 `B` 都是 4 字节值。例如：

```sql
nik=# select pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 5D/257E19B0
(1 row)

```

- 这里 `5D` 是 LSN 的更高 4 字节（32 位）部分
- `257E19B0` 也可以分为两部分：
  - 25 – LSN 的较低 4 字节部分（更具体地说，仅该 4 字节部分的最高 1 字节）
  - 7E19B0 – WAL 中的偏移量（默认为 16 MiB，在某些情况下，它会发生变化,例如，RDS 将其更改为 64 MiB）

有趣的是，LSN 值可以比较，甚至可以相互减去一个,假设我们使用 pg_lsn 数据类型。结果将以字节为单位：

```sql
nik=# select pg_lsn '5D/257D6780' - '5D/251A8118';
 ?column?
----------
  6481512
(1 row)
 nik=# select pg_size_pretty(pg_lsn '5D/257D6780' - '5D/251A8118');
 pg_size_pretty
----------------
 6330 kB
(1 row)

```

这也意味着我们可以得到 LSN 的整数值，只需看看我们从"point 0 ",值 0/0 走了多远：

```sql
nik=# select pg_lsn '5D/257D6780' - '0/0';
   ?column?
--------------
 400060934016
(1 row)

```

# 如何读取WAL文件名

现在让我们看看 LSN 值如何对应于 WAL 文件名（位于 `$PGDATA/pg_wal` 中的文件）。我们可以使用函数 `pg_walfile_name()` 获取任何给定 LSN 的 WAL 文件名

```sql
nik=# select pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());
 pg_current_wal_lsn |     pg_walfile_name
--------------------+--------------------------
 5D/257E19B0        | 000000010000005D00000025
(1 row)

```

这里 `000000010000005D00000025` 是WAL文件名，它由三个4字节（32位）字组成：

1. `00000001` – 时间线 ID （TimeLineID），初始化 Postgres 集群时以 1 开头的连续"history number"。它“识别不同的数据库历史记录，以防止在恢复数据库安装的先前状态后出现混淆”（ [源代码](https://gitlab.com/postgres/postgres/blob/4f2994647ff1e1209829a0085ca0c8d237dbbbb4/src/include/access/xlogdefs.h#L50) ）。
2. `0000005D` – 序列号的更高 4 字节部分。
3. `00000025` – 可以看作是两部分：
 - `000000` - 6 个前导零.
 - `25` – 序列号下部的最高字节。

重要的是要记住：WAL 文件名中的第三个 4 字节字有 6 个前导零,通常，这在比较两个 WAL 文件名以了解其中预期的 LSN 时会导致混淆和错误。

有用的插图（来自 [这篇文章](https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames) ）：

```sql
LSN:                5D/      25   7E19B0
WAL: 00000001 0000005D 00000025

```

如果您需要使用 LSN 值或 WAL 文件名或两者，快速导航或比较它们的值以了解它们之间的距离，这将非常有用。理解一些有用的示例：

1. 服务器每天生成多少字节
2. 自创建复制槽以来经过的量
3. 两个备份之间的距离是多少
4. 需要重放多少 WAL 数据才能达到一致性点

**除了官方文档之外，值得一读的好博客文章：**

- [Postgres 9.4 feature highlight - LSN datatype](https://paquier.xyz/postgresql-2/postgres-9-4-feature-highlight-lsn-datatype/)
- [Postgres WAL Files and Sequence Numbers](https://www.crunchydata.com/blog/postgres-wal-files-and-sequuence-numbers)
- [WAL, LSN, and File Names](https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames)

感谢您的阅读！像往常一样：请与您的同事和与 #PostgreSQL 一起工作的朋友分享。