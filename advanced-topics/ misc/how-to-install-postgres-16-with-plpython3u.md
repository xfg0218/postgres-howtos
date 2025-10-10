PL/Python 是PostgreSQL的过程语言扩展，它允许您在Python中编写存储过程和触发器，Python是一种广泛使用的高级通用编程语言。

`plpython3u` 是 PL/Python 的 "不受信任" 版本。此变体允许 Python 函数执行文件 I/O 、网络通信和其他可能影响服务器行为或安全性的操作。

我们在 [如何直接从Postgres使用OpenAI API来实现语义搜索和GPT聊天](https://gitlab.com/postgres-ai/docs/-/tree/master) 中使用了 `plpython3u`，现在让我们讨论如何安装它。

我觉得我们将来会更多地使用它，在各种任务中。

此 howto 仅适用于自我管理的 Postgres 。

# macOS (Homebrew)

```sql
brew tap petere/postgresql
brew install petere/postgresql/postgresql@16

psql postgres \
  -c 'create extension plpython3u'

```

# Ubuntu 22.04 LTS or Debian 12

```sql
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'

curl -fsSL https://postgresql.org/media/keys/ACCC4CF8.asc \
  | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg

sudo apt update
sudo apt install -y \
  postgresql-16 \
  postgresql-contrib-16 \
  postgresql-plpython3-16

sudo -u postgres psql \
  -c 'create extension plpython3u'

```

# CentOS Stream 9

```sql
dnf module reset -y postgresql
dnf module enable -y postgresql:16

dnf install -y \
  postgresql-server \
  postgresql \
  postgresql-contrib \
  postgresql-plpython3

postgresql-setup --initdb

systemctl enable --now postgresql

sudo -u postgres psql \
  -c 'create extension plpython3u'

```

# Docker

```sql
echo "FROM postgres:16
RUN apt update
RUN apt install -y postgresql-plpython3-16" \
> postgres_plpython3u.Dockerfile

sudo docker build \
  -t postgres-plpython3u:16 \
  -f postgres_plpython3u.Dockerfile \
  .

sudo docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v $(echo ~)/pgdata:/var/lib/postgresql/data \
  postgres-plpython3u:16

sudo docker exec -it pg16 \
  psql -U postgres -c 'create extension plpython3u'

```


