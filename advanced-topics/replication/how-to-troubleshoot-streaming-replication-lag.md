Postgres 中的流复制允许从主服务器到备用服务器的连续数据复制，以确保高可用性并平衡只读工作负载。但是，可能会出现复制滞后，从而导致数据同步延迟。本指南提供了故障排除和缓解复制延迟的步骤。

# 识别滞后

要开始调查，我们需要了解实际存在滞后的位置，在复制的哪个阶段：

- 通过网络将WAL流发送到副本 by `walsender`
- 通过 `walreciever` 从网络接收副本上的 WAL 流。
- 在副本上的磁盘上写入 WAL by `walreciever`
- 应用（重放）WAL作为恢复过程

因此，流复制延迟可分为三种类型：

- **Write Lag:** 事务在主服务器上提交到备用服务器上的 WAL 之间的延迟。
- **Flush Lag:** 事务写入备用数据库上的 WAL 和刷新到磁盘之间的延迟。
- **Apply (Replay) Lag:** 将事务刷新到磁盘与将其应用于备用数据库之间的延迟。

# 分析查询

要识别延迟，请使用以下 SQL 查询：

```sql
select
  pid,
  client_addr,
  application_name,
  state,
  coalesce(pg_current_wal_lsn() - sent_lsn, 0) AS sent_lag_bytes,
  coalesce(sent_lsn - write_lsn, 0) AS write_lag_bytes,
  coalesce(write_lsn - flush_lsn, 0) AS flush_lag_bytes,
  coalesce(flush_lsn - replay_lsn, 0) AS replay_lag_bytes,
  coalesce(pg_current_wal_lsn() - replay_lsn, 0) AS total_lag_bytes
from pg_stat_replication;

```

# 例子:

我们将得到这样的结果：

```sql
postgres=# select
  pid,
  client_addr,
  application_name,
  state,
  coalesce(pg_current_wal_lsn() - sent_lsn, 0) AS sent_lag_bytes,
  coalesce(sent_lsn - write_lsn, 0) AS write_lag_bytes,
  coalesce(write_lsn - flush_lsn, 0) AS flush_lag_bytes,
  coalesce(flush_lsn - replay_lsn, 0) AS replay_lag_bytes,
  coalesce(pg_current_wal_lsn() - replay_lsn, 0) AS total_lag_bytes
from pg_stat_replication;

   pid   |  client_addr   | application_name | state     | sent_lag_bytes | write_lag_bytes | flush_lag_bytes | replay_lag_bytes | total_lag_bytes
---------+----------------+------------------+-----------+----------------+-----------------+-----------------+------------------+-----------------
 3602908 | 10.122.224.101 | backupmachine1   | streaming |              0 |       728949184 |               0 |                0 |               0
 2490863 | 10.122.224.102 | backupmachine1   | streaming |              0 |       519580176 |               0 |                0 |               0
 2814582 | 10.122.224.103 | replica1         | streaming |              0 |          743384 |               0 |          1087208 |         1830592
 3596177 | 10.122.224.104 | replica2         | streaming |              0 |         2426856 |               0 |          4271952 |         6698808
  319473 | 10.122.224.105 | replica3         | streaming |              0 |          125080 |          162040 |          4186920 |         4474040

```

# 如何读取结果

这些 `_lsn` 的含义:

- `sent_lsn`：已经通过网络发送了多少 WAL（lsn 头寸）
- `write_lsn`：已向作系统发送了多少 WAL（lsn 位置）（刷新前）
- `flush_lsn`：有多少 WAL（lsn 位置）被刷新到磁盘（写入磁盘上）
- `replay_lsn`：应用了多少 WAL（lsn 位置）（查询可见）

因此，滞后是 `pg_current_wal_lsn` 和 `replay_lsn` 之间的差距（ `total_lag_bytes` ，最好将其添加到监控中，但出于故障排除目的，我们将需要所有 4 个 )

- `sent_lag_bytes` 滞后意味着我们在发送数据时遇到问题，即 `CPU` 饱和的 `WALsender` 或主端的网络套接字过载。
- `write_lag_bytes` 滞后意味着我们在接收数据时遇到问题，即 `CPU` 饱和的 `WALreceiver` 或副本端的网络套接字过载。
- `flush_lag_bytes` 滞后意味着我们在副本端的磁盘上写入数据时遇到问题，即 `CPU` 饱和或 `WALreceiver` 的 `IO` 争用。
- 滞后 `replay_lag_bytes` 意味着我们在副本上应用 `WAL` 时遇到问题，通常是 `CPU` 饱和或 `postgres` 进程的 `IO` 争用。

一旦我们查明了问题，我们就需要在作系统级别对进程进行故障排除，以找到瓶颈。

# 可能的瓶颈

# 其他资源

- [流复制](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION)（Postgres 官方文档）
- [pg_stat_replication视图](https://www.postgresql.org/docs/current/monitoring-stats.html#PG-STAT-REPLICATION-VIEW)（Postgres 官方文档）
- [Replication configuration parameters ](https://www.postgresql.org/docs/current/runtime-config-replication.html)
