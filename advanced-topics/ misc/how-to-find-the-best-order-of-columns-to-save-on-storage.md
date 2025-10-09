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

