今天，我们将讨论元组及其在页面中的位置——这是相当入门级的材料，但在许多情况下很有用。

在许多情况下，了解表中行的物理布局可能很重要，尤其是在性能优化工作期间。

# 一些术语

- Page / buffer /  block  – 磁盘和 Postgres 缓冲池中的存储单元（加载到 RAM 原封不动），在大多数情况下为 8 KiB（检查它：`show block_size;` ），它保存表或索引的一部分。
- Tuple – 表中行的物理版本。
- Tuple header - 有关元组的元数据，包括事务 ID、可见性信息等。
- Transaction ID (same as XID, tid, txid) - Postgres 中交易的唯一标识符。
  - 它用于修改事务。只读具有“virtualxid”以避免“浪费”XID，因为截至 PG16 它们仍然是 32 位。正在进行切换到 [64 位](https://commitfest.postgresql.org/patch/3594/) 的工作。
  - 您可以为调用函数 `pg_current_xact_id()` 或在 PG12 及更早版本中为 `txid_current()` 的交易分配一个 XID。

元组标头有有趣的 "hidden" 或 "system" 列（文档）：

- `ctid` – 表示元组在表中的物理位置的隐藏（系统）列，它具有两个整数(`X`, `Y`)的形式，其中：
  - `X` 是从 0 开始的页码
  - `Y` 是页面内从 1 开始的元组的序列号
- `xmin`， `xmax` – 创建此行版本（元组）并删除它（使此元组 "dead"）的事务的 XID。

# ctid

如果我们需要了解某些元组的存储稀疏程度，我们可以将 `ctid` 包含在查询的 `SELECT` 子句中。例如，我们有下表和一个简单的查询：

```sql
nik=# \d t1
                      Table "public.t1"
 Column  |       Type       | Collation | Nullable | Default
---------+------------------+-----------+----------+---------
 id      | bigint           |           | not null |
 user_id | double precision |           |          |
Indexes:
    "t1_pkey" PRIMARY KEY, btree (id)
    "t1_user_id_idx" btree (user_id)

nik=# select * from t1 where user_id = 101469;
   id   | user_id
--------+---------
  28414 |  101469
 235702 |  101469
 478876 |  101469
 495042 |  101469
 555593 |  101469
 626491 |  101469
 635785 |  101469
 702725 |  101469
(8 rows)

```

要了解这些行的物理位置，只需将 `ctid` 包含在同一查询的 SELECT 子句中：

```sql
nik=# select ctid, * from t1 where user_id = 101469;
    ctid    |   id   | user_id
------------+--------+---------
 (153,109)  |  28414 |  101469
 (1274,12)  | 235702 |  101469
 (2588,96)  | 478876 |  101469
 (2675,167) | 495042 |  101469
 (3003,38)  | 555593 |  101469
 (3386,81)  | 626491 |  101469
 (3436,125) | 635785 |  101469
 (3798,95)  | 702725 |  101469
(8 rows)

```

– 每行存储在不同的页面（`153`、`1274` 等）中。这不是此查询性能的最佳情况 - 需要大量 IO。

现在，如果我们想查看第 1274 页中存在哪些元组，我们可以使用一个技巧，将 ctid 转换为 point（通过文本，因为无法直接转换）并提取两者中的第一个数字-页码。

```sql
nik=# select ctid, * from t1 where (ctid::text::point)[0] = 1274 order by ctid limit 15;
   ctid    |   id   | user_id
-----------+--------+---------
 (1274,1)  | 235691 |  225680
 (1274,2)  | 235692 |  617397
 (1274,3)  | 235693 |  233968
 (1274,4)  | 235694 |  714957
 (1274,5)  | 235695 |  742857
 (1274,6)  | 235696 |  837441
 (1274,7)  | 235697 |  745413
 (1274,8)  | 235698 |  752474
 (1274,9)  | 235699 |  102335
 (1274,10) | 235700 |  908715
 (1274,11) | 235701 |  909036
 (1274,12) | 235702 |  101469
 (1274,13) | 235703 |  599451
 (1274,14) | 235704 |  359470
 (1274,15) | 235705 |  196437
(15 rows)

nik=# select count(*) from t1 where (ctid::text::point)[0] = 1274;
 count
-------
   185
(1 row)

```

– 总的来说，有 185 行，我们的 `user_id=101469` 行也存在，位于第 12 位。

（但是请注意，对于较大的表来说，这将是一个非常慢的查询，因为它需要 Seq Scan，并且我们无法在 `ctid` 或其他系统列上创建索引。对于旨在查找特定 `ctid` （ `...where ctid = '(123, 456)'` ），不过，由于 Tid Scan，性能会很好，参见 https://pgmustard.com/docs/explain/tid-scan）。

# 执行计划中的 ctid 和 BUFFERS 指标

确认原始查询确实涉及许多缓冲区作（另请参阅第 1 天，我们谈到了 `BUFFERS` 的重要性）：

```sql
nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Index Scan using t1_user_id_idx on t1  (actual time=0.131..0.367 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Buffers: shared hit=11
 Planning Time: 0.159 ms
 Execution Time: 0.383 ms
(5 rows)

```

– 11 次缓冲区命中，它是 88 KiB。这是检索 8 行。以下是我们如何确定这 8 行的大小：

```sql
nik=# select sum(pg_column_size(t1.*)) from t1 where user_id = 101469;
 sum
-----
 317
(1 row)

```

因此，Postgres 执行器必须处理 88 KiB 才能返回 317 字节——这远非最佳。由于我们这里有一个 `Index Scan`，其中一些缓冲区命中与索引相关，有些-从堆（表）中获取数据。

# 如何改进？

**选项 0** .除了了解正在发生的事情之外，什么都不做。也许，您不需要进行重大改进，因为下面讨论的选项都不是完美的。避免过度优化。但是要了解稀疏元组的位置，并准备好仔细检查它。在某些情况下，目标元组存储过于稀疏这一事实可能是查询性能的一个重要因素，从而导致超时。在这种情况下，请考虑以下策略。

**选项 1**.保持表和索引处于良好状态：

- 表膨胀控制: 定期分析膨胀，通过调整良好的自动真空来防止膨胀，并通过 `pg_repack` 定期消除。
- 索引维护: 膨胀控制 + 定期重新索引，因为即使自动真空调整得很好，索引健康也会随着时间的推移而下降（PG14 中提高了 btree 健康降级率，但这些优化并不能消除在重负载系统中定期重新索引的需要）。
- 分区: 分区的好处之一是提高了数据局部性。

**选项 2**.使用仅索引扫描而不是索引扫描。这可以通过使用多列索引或覆盖索引来实现，以包含查询所需的所有列。对于我们的示例：

```sql
nik=# create index on t1(user_id) include (id);
CREATE INDEX

nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Index Only Scan using t1_user_id_id_idx on t1 (actual time=0.040..0.044 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Heap Fetches: 0
   Buffers: shared hit=4
 Planning Time: 0.179 ms
 Execution Time: 0.072 ms
(6 rows)

```

– 4 次缓冲命中而不是 11 次，好多了。

**选项 3**：根据索引/列值对表进行物理重组：这是表的物理重组。它有两个缺点：

- 您需要选择要用于它的索引 - 只有一个索引。因此，它仅对工作负载上的特定查询子集有帮助，而对其他查询可能毫无用处
- 行的 UPDATE 将移动元组，从而降低 `CLUSTER` 的好处，因此可能需要重复它。

有两种方法可以重新组织表格

- SQL 命令 `CLUSTER` （ [文档](https://www.postgresql.org/docs/current/sql-cluster.html) ,不是在线作，不建议用于无法承受维护窗口的实时系统
- [pg_repack](https://github.com/reorg/pg_repack) 有选项 --order-by=<..>，它允许实现类似于 CLUSTER 的效果，但以在线方式，无需停机。

对于我们的示例：

```sql
nik=# cluster t1 using t1_user_id_idx;
CLUSTER

And now the query:

nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Index Scan using t1_user_id_idx on t1 (actual time=0.035..0.040 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Buffers: shared hit=4
 Planning Time: 0.141 ms
 Execution Time: 0.070 ms
(5 rows)

```

– 也只有 4 个缓冲区命中 – 与具有覆盖索引和 `Index-Only Scan` 的方法相同。不过，这里我们有一个 `Index Scan` 。

这就是今天的内容——我们讨论了 ctid。在未来的某个时候，出于实际原因，我们将继续使用 xmin 和 xmax，并对表/索引页面进行深入检查。

[订阅](https://x.com/samokhvalov)、分享、点赞！

不要忘记，我在这个 GitLab 存储库中镜像了这些技巧：https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/
