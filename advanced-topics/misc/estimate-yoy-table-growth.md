假设我们有一个多年前创建的10 TiB表，无论是否分区，如下所示：

```sql
create table t (
  id int8 primary key,   -- of course, not int4
  created_at timestamptz default now(),
  ...
);
```

我们需要快速了解表是如何逐年增长的，假设没有删除任何行（或只有可删除的数量）。因此，我们只需要计算每年的行数。

一个直接的方法是：

```sql
select
  date_trunc('year', created_at) as year,
  count(*)
from t
group by year
order by year;

```

但是，对于10 TiB表，我们需要等待数小时（如果不是数天的话）才能完成此分析。

这是一种快速但不精确的方法来获取每年的行数（假设表具有最新的统计信息；如果indoubt，请先运行ANALYZE）：

```sql
do $$
declare
  table_fqn text := 'public.t';
  year_start int := 2000;
  year_end int := extract(year from now())::int;
  year int;
  explain_json json;
begin
  for year in year_start..year_end loop
    execute format(
      $e$
        explain (format json) select *
        from %s
        where created_at
        between '%s-01-01' and '%s-12-31'
      $e$,
      table_fqn,
      year,
      year
    ) into explain_json;

    raise info 'Year: %, Estimated rows: %',
      year,
      explain_json->0->'Plan'->>'Plan Rows';
  end loop;
end $$;

```

