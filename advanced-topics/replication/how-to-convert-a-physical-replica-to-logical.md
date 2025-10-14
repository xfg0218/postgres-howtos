在某些情况下，将现有的常规异步物理副本转换为逻辑副本，或者先创建一个新的物理副本，然后将其转换为逻辑副本可能会有所帮助。

这种方法：

- 一方面，无需执行初始数据加载步骤，在大型、重负载数据库的情况下，该步骤可能很脆弱且压力很大，但是。
- 在另一个平台上，以这种方式创建的逻辑副本具有源 Postgres 实例所拥有的所有内容。

因此，当您需要将源中的所有数据呈现在您正在创建的逻辑副本中时，此方法更适合，如果您使用非常大、负载较重的集群，它非常有用。

以下步骤非常简单。在这种情况下，我们使用物理副本，通过流复制primary_conninfo和复制槽，例如在 Patroni 的控制下 立即从主副本复制数据，而不是涉及级联复制，尽管也可以实现。

**步骤 1：有一个物理副本进行转换**

选择要转换的物理副本，或使用 `pg_basebackup` 、从备份中恢复或从云快照创建物理副本。

确保在转换时，普通用户不会使用此副本。

**第 2 步：确保满足要求**

首先，确保设置已准备好进行逻辑复制，如逻辑复制配置中所述。

主要设置：

- `wal_level = 'logical'`
- `max_replication_slots > 0`
- `max_wal_senders > max_replication_slots`

在物理副本上，我们将转换：

- `max_replication_slots > 0`
- `max_logical_replication_workers > 0`
- `max_worker_processes >= max_logical_replication_workers + 1`

此外：

- 复制滞后低;
- 每个表都有一个 PK 或具有 `REPLICA IDENTITY FULL`;
- `restore_command` 未在我们将使用的副本上设置（如果是，请暂时将其值设置为空字符串）;
- 暂时，将主服务器上的 `wal_keep_size`（PG13+;在PG12或更早版本中，wal_keep_segments）增加到对应于几个小时的WAL生成的值。

**步骤 3：停止物理副本**

关闭物理副本，并在下一步中将其保持关闭。这是必需的，以便与我们将要在主服务器上创建的逻辑槽相比，保证其位置在过去。

**步骤 4：创建发布、逻辑槽并记住其 LSN**

在初级：

- 发出手动 `CHECKPOINT`;
- 创建 `publication`
- 创建一个逻辑槽并记住其 LSN 位置;

例：

```sql
checkpoint;

create publication my_pub for all tables;

select lsn
from pg_create_logical_replication_slot(
  'my_slot',
  'pgoutput'
);

```

记住最后一个命令的 lsn 值很重要——我们将进一步使用它。

**第五步：让物理副本迎头赶上**

重新配置物理副本：

- `recovery_target_lsn` – 将其设置为我们从上一步获得的 LSN 值。
- `restore_command`、`recovery_target_timeline`、`recovery_target_xid`、`recovery_target_time`、`recovery_target_name` 未设置或为空

现在，启动物理副本。监控其滞后以及副本如何赶上我们需要的 LSN 然后自动提升。这可能需要一些时间。完成后，检查它：

```sql
select pg_is_in_recovery();
```

- 必须返回 `f`，这意味着此节点现在是具有位置的主节点，对应于源节点上复制槽的位置。

**步骤 6：创建订阅并启动逻辑复制**

现在，对于新创建的 "clone"，创建逻辑订阅，`copy_data = false` 和 `create_slot = false`：

```sql
create subscription 'my_sub'
connection 'host=.. port=.. user=.. dbname=..'
publication my_pub
with (
  copy_data = false,
  create_slot=false,
  slot_name = 'my_slot'
);

```

确保复制现在处于活动状态 - 在源主数据库上检查它：

```sql
select * from pg_replication_slots;
```

- 对于我们的插槽，active必须是 `t`。

# 完成

- 等到逻辑复制滞后完全赶上（偶尔出现急性峰值是可以的）。
- 将 `wal_keep_size` （ `wal_keep_segments` ） 返回到主服务器上的原始值。

附加说明

在这里，我们在此配方中使用了单个发布和逻辑槽。可以使用多个插槽，稍微调整程序。但是，如果您选择这样做，请记住使用多个插槽/出版物的潜在复杂性，首先是这些：

- 不保证逻辑副本上的引用完整性（偶尔发生临时 FK 冲突），
- 更脆弱的发布创建过程（为 `FOR ALL TABLES` 发布不需要表级锁;但是当我们使用多个发布并为某些表创建发布时，表级锁是必需的——但是，这只是 `ShareUpdateExclusiveLock`，根据 [PostgreSQL 源代码上的这条评论](https://github.com/postgres/postgres/blob/1b6da28e0668eb977dcab6987d192ddedf32b752/src/backend/commands/publicationcmds.c#L1550) ）。

无论如何：

- 确保您准备好处理您的版本（例如，对于 [PG16](https://www.postgresql.org/docs/16/logical-replication-restrictions.html) ）的逻辑复制限制;
- 如果您考虑使用此方法执行重大升级，请避免在已转换的节点上运行 `pg_upgrade` - 它可能不安全（参见：[pg_upgrade和逻辑复制](https://postgresql.org/message-id/flat/20230217075433.u5mjly4d5cr4hcfe%40jrouhaud) ）。



