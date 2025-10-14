今天我们将在Postgres中实现RAG （ [检索增强生成](https://en.wikipedia.org/wiki/Prompt_engineering#Retrieval-augmented_generation%5D) ）

1. 将完整的Postgres提交历史记录加载到Postgres表
2. 使用 `plpython3u`（在某些托管服务上为N/A，例如RDS），直接从Postgres开始调用OpenAI API。
> ⚠️ 警告：这种方法不能很好地扩展，因此不建议用于较大的生产集群。考虑这是为了好玩或仅用于小型项目/服务。
3. 对于每次提交，生成OpenAI嵌入并将它们存储为"vector"格式（ `pgvector` ）。
4. 使用语义搜索查找提交，由 `pgvector` 的HNSW索引加速。
5. 最后，通过OpenAI GPT4 "交谈提交历史"。


# 准备工作

首先，安装扩展：

```sql
create extension if not exists plpython3u;
create extension if not exists vector;

```

然后，我们需要一个 [OpenAI API](https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key) 密钥。让我们将其存储为为当前数据库定义的自定义GUC

> ⚠️ 警告：这不是最安全的存储方式；为了简单起见，我们使用这种方法：

```sql
do $$ begin
 execute(
   format(
     'alter database %I set openai.api_key = %L',
     current_database(),
     'sk-XXXXXXXXXXXX'
   )
 );
end $$;

```

# 从Git导入提交历史记录

创建一个表，我们将在其中存储Git提交：

```sql
psql -X \
  -c "create table commit_logs (
    id bigserial primary key,
    hash text not null,
    created_at timestamptz not null,
    authors text,
    message text,
    embedding vector(1536)
  )" \
  -c "create unique index on commit_logs(hash)"

```

现在，从GitHub或GitLab克隆Postgres存储库并将完整的提交历史记录获取到CSV文件（注意在提交消息中转义双引号）：

```sql
git clone https://gitlab.com/postgres/postgres.git
cd postgres

git log --all --pretty=format:'%H,%ad,%an,🐘%s %b🐘🐍' --date=iso \
 | tr '\n' ' ' \
 | sed 's/"/""/g' \
 | sed 's/🐘/"/g' \
 | sed 's/🐍/\n/g' \
 > commit_logs.csv

```

将提交历史从CSV加载到表中：

```sql
psql -Xc "copy commit_logs(hash, created_at, authors, message)
   from stdin
   with delimiter ',' csv" \
 < commit_logs.csv

psql -X \
 -c "update commit_logs set hash = trim(hash)" \
 -c "vacuum full analyze commit_logs"

```

截至2025年7月，这将产生约96k行，涵盖Postgres发展历史的10k天--超过27年第一次提交是在1996年7月！

# 创建和存储嵌入

这是一个函数，我们将使用 `plpython3u` 从OpenAI API获取每个提交条目的向量（ `u` 这里的意思是"不受信任"-允许此类函数与外部世界对话）：

```sql
create or replace function openai_get_embedding(
  content text,
  api_key text default current_setting('openai.api_key', true)
) returns vector(1536)
as $$
  import requests

  response = requests.post(
    'https://api.openai.com/v1/embeddings',
    headers={ 'Authorization': f'Bearer {api_key}' },
    json={
      'model': 'text-embedding-3-small',
      'input': content.replace("\n", " ")
   }
 )

  if response.status_code >= 400:
   raise Exception(f"Failed. Code: {response.status_code}")

  return response.json()['data'][0]['embedding']
$$ language plpython3u;

```

一旦创建，开始获取和存储向量。我们将小批量进行，以避免long-runningtransactions-不，在失败的情况下丢失大量数据，并且不阻塞并发会话（如果有）：

```sql
with scope as (
  select hash
  from commit_logs
  where embedding is null
  order by id
  limit 5
), upd as (
  update commit_logs
  set embedding = openai_get_embedding(
    content := format(
      'Date: %s. Hash: %s. Message: %s',
      created_at,
      hash,
      message
    )
  )
  where hash in (
    select hash
    from scope
 )
 returning *
)
select
  count(embedding) as cnt_vectorized,
  max(upd.id) as latest_upd_id,
  round(
    max(upd.id)::numeric * 100 /
      (select max(c.id) from commit_logs as c),
    2
  )::text || '%' as progress,
  1 / count(*) as err_when_done
from upd
\watch .1

```

这个过程可能需要大量的时间，也许超过一个小时，所以准备等待。此外，这里是我们开始为API使用向OpenAI付费的地方（尽管，嵌入创建非常便宜，您将在这里支付~~1美元，请参阅定价）。

一旦完成——或者更早，有部分结果——你就可以开始使用它了。

# 语义搜索

这是一个使用语义搜索查找最相关提交的简单示例：

这里的概念很简单：

1. 首先，我们调用OpenAI API来"vectorize"我们请求的文本。
2. 然后，我们使用 `pgvector` 的相似度搜索来找到K个最近的邻居。

我们将使用HNSW索引，它被认为是当今最好的方法之一（尽管最初在2016中描述）；由许多DBMS添加。在pgvector中，它是在版本0.5.0中添加的。请注意，这是ANN索引-"approximate nearest neighbors"，所以为了速度起见，它允许生成严格的结果，不像Postgres中的常规索引。

创建索引：

```sql
psql -Xc "create index on commit_logs
 using hnsw (embedding vector_cosine_ops)"

```

现在，在 `psql` 中，执行搜索：

```sql
select openai_get_embedding('psql features') as q_vector \gset

select
  created_at,
  format(
    'https://gitlab.com/postgres/postgres/-/commit/%s',
    left(hash, 8)
 ),
 left(message, 150),
 authors,
 1 - (embedding <-> :'q_vector') as similarity
from commit_logs
order by embedding <-> :'q_vector'
limit 5 \gx

```

如果创建了index，第二个查询应该非常快。您可以使用EXPLAIN (ANALYZE, BUFFERS)检查执行计划和详细信息。我们的数据集很小（<100k），所以搜索速度应该是~1ms，缓冲区命中/读取数~1000或更少。索引pgvector提供了一些调整选项，请查看其 [README](https://github.com/pgvector/pgvector/blob/master/README.md)。

以下是查询 `psql "\watch" limited count of loops` 的示例结果：

```sql
postgres-# limit 5 \gx
┌─[ RECORD 1 ]────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ created_at │ 2025-05-29 20:58:57-07                                                                                                                                 │
│ format     │ https://gitlab.com/postgres/postgres/-/commit/13cb16e7                                                                                                 │
│ left       │ Implement comprehensive improvements to pg_restore object tracking - Add command-line options for file paths and disable flag - Capture actual Postgre │
│ authors    │ Nikolay Samokhvalov                                                                                                                                    │
│ similarity │ -0.025786986265972756                                                                                                                                  │
├─[ RECORD 2 ]────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ created_at │ 2025-05-04 11:38:42-07                                                                                                                                 │
│ format     │ https://gitlab.com/postgres/postgres/-/commit/2448c7a9                                                                                                 │
│ left       │ doc: Put some psql documentation pieces back into alphabetical order                                                                                   │
│ authors    │ Peter Eisentraut                                                                                                                                       │
│ similarity │ -0.03374895125754507                                                                                                                                   │
├─[ RECORD 3 ]────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ created_at │ 2025-05-29 20:05:54-07                                                                                                                                 │
│ format     │ https://gitlab.com/postgres/postgres/-/commit/64512a7b                                                                                                 │
│ left       │ Improve test and add documentation for pg_restore object tracking - Simplify test to use single table instead of complex schema - Use unique index cor │
│ authors    │ Nikolay Samokhvalov                                                                                                                                    │
│ similarity │ -0.037387386782650855                                                                                                                                  │
├─[ RECORD 4 ]────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ created_at │ 2025-05-23 14:05:35-07                                                                                                                                 │
│ format     │ https://gitlab.com/postgres/postgres/-/commit/4050aedd                                                                                                 │
│ left       │ Implement complete WAL metrics functionality for table statistics This commit adds comprehensive WAL tracking to PostgreSQL table-level statistics, en │
│ authors    │ Nikolay Samokhvalov                                                                                                                                    │
│ similarity │ -0.050729119237905795                                                                                                                                  │
├─[ RECORD 5 ]────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ created_at │ 2025-05-20 14:31:00-07                                                                                                                                 │
│ format     │ https://gitlab.com/postgres/postgres/-/commit/a6060f1c                                                                                                 │
│ left       │ pg_dump: Fix array literals in fetchAttributeStats(). Presently, fetchAttributeStats() builds array literals by treating the elements as SQL identifie │
│ authors    │ Nathan Bossart                                                                                                                                         │
│ similarity │ -0.0509084177885033                                                                                                                                    │
└────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

```

# 直接从Postgres使用OpenAI API"与Git历史对话"

最后，再创建两个函数并提出有关Postgres提交历史的问题：

```sql
create or replace function openai_gpt_call(
  question text,
  data_to_embed text,
  model text default 'gpt-4.1-nano-2025-04-14',
  token_limit int default 4096,
  api_key text default current_setting('openai.api_key', true)
) returns text
as $$
  import requests, json

  prompt = """Be terse. Discuss only Postgres and it's commits.
For commits, mention timestamp and hash.
CONTEXT (git commits):
---
%s
---
QUESTION: %s
""" % (data_to_embed[:2000], question)

  ### W: this code lacks error handling
  response = requests.post(
    'https://api.openai.com/v1/chat/completions',
    headers={ 'Authorization': f'Bearer {api_key}' },
    json={
      'model': model,
      'messages': [
          {"role": "user", "content": prompt}
      ],
      'max_tokens': token_limit,
      'temperature': 0
    }
 )

  if response.status_code >= 400:
    raise Exception(f"Failed. Code: {response.status_code}. Response: {response.text}")

  return response.json()['choices'][0]['message']['content']
$$ language plpython3u;

create or replace function openai_chat(
  in question text,
  in model text default 'gpt-4',
  out answer text
) as $$
  with q as materialized (
    select openai_get_embedding(
      question
    ) as emb
  ), find_enries as (
    select
      format(
        e'Created: %s, hash: %s, message: %s, committer: %s\n',
        created_at,
        left(hash, 8),
        message,
        authors
      ) as entry
    from commit_logs
    where embedding <=> (select emb from q) < 0.8
    order by embedding <=> (select emb from q)
    limit 10 -- adjust if needed
  )
  select openai_gpt_call(
    question := openai_chat.question,
    data_to_embed := string_agg(entry, e'\n'),
    model := openai_chat.model
  )
  from find_enries;
$$ language sql;

```

现在，只需使用 `openai_chat(...)` 进行询问，例如：

```sql
nik=# \a
Output format is unaligned.

nik=# select openai_chat('tell me about fixes of CREATE INDEX CONCURRENTLY – when, who, etc.');
openai_chat
There are two notable fixes for CREATE INDEX CONCURRENTLY. The first one was made on 2016-02-16 18:43:03+00 by Tom Lane, with the commit hash a65313f2. The commit improved the documentation about CREATE INDEX CONCURRENTLY, clarifying the description of which transactions will block a CREATE INDEX CONCURRENTLY command from proceeding.

The second fix was made on 2012-11-29 02:25:27+00, with the commit hash 3c840464. This commit fixed assorted bugs in CREATE/DROP INDEX CONCURRENTLY. The issue was that the pg_index state for an index that's reached the final pre-drop stage was the same as the state for an index just created by CREATE INDEX CONCURRENTLY. This was fixed by adding an additional boolean column "indislive" to pg_index.
(1 row)

```

请注意，默认情况下，它使用模型 `text-embedding-3-small` 进行嵌入，使用 `gpt-4.1-nano-2025-04-14` 进行完成API调用。在您阅读本文时，可能有更新、更好的模型-请参阅 [定价](https://openai.com/pricing) 和 [模型](https://platform.openai.com/docs/models)。

# 最后的几个节点

1. 如上所述，这种方法从Postgres调用外部API不能很好地扩展。它有利于快速原型设计，但不应用于预期TPS大幅增长的项目中（否则，随着增长，CPU饱和风险、tx空闲峰值等。，可能会导致严重的性能问题甚至中断）。
2. 这种方法的另一个缺点是 `plpython3u` 在某些Postgres服务（例如RDS）中不可用。
3. 最后，当在SQL环境中工作时，很容易无意中循环API调用。这可能会导致过多的费用。为了避免这种情况，我们需要仔细检查执行计划。
4. 对于某些人来说，这样的代码更难调试。

尽管如此，上面描述的概念在某些情况下可能会很好地工作——只要记住这些细微差别并避免低效的移动。如果担忧太高，调用API的代码应该从 `plpython3u` 移到Postgres之外。 

