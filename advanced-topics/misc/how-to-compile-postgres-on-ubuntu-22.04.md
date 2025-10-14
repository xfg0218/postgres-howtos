这篇文章介绍了如何在Ubuntu 22.04上快速编译Postgres。

[官方文档](https://www.postgresql.org/docs/current/installation.html)。它非常详细，非常适合用作参考，但您不会找到特定操作系统版本（例如Ubuntu）的具体步骤列表。

这个howto足以从 `master` 分支构建Postgres并开始使用它。如果需要，扩展这组基本步骤。

还有几个注释：

- 当前（截至2023年，PG16）文档提到了一种构建Postgres的新方法- [Meson](https://mesonbuild.com/) -但它仍然被认为是实验性的，我们不会在这里使用它。
- 我们在这里使用的二进制文件和PGDATA的路径只是作为示例提供的（适用于用于测试的"快速n-脏"设置，例如，来自 `pgsql-hackers` 邮件列表的新补丁）。

1. 安装所需软件
2. 获取源代码
3. 配置
4. 编译安装
5. 创建一个集群并开始使用它

## 1）安装所需软件

```sql
sudo apt update

sudo apt install -y \
  build-essential libreadline-dev \
  zlib1g-dev flex bison libxml2-dev \
  libxslt-dev libssl-dev libxml2-utils \
  xsltproc ccache

```

## 2）获取源代码

```sql
git clone https://gitlab.com/postgres/postgres.git
cd ./postgres

```

## 3）配置

```sql
mkdir ~/pg16 # consider changing it!

./configure \
  --prefix=$(echo ~/pg16) \
  --with-ssl=openssl \
  --with-python \
  --enable-depend \
  --enable-cassert

```

## 4）编译安装

```sql
make -j$(grep -c processor /proc/cpuinfo)
make install

```

## 5）创建一个集群并开始使用它

```sql
~/pg16/bin/initdb \
  -D ~/pgdata \
  --data-checksums \
  --locale-provider=icu \
  --icu-locale=en

~/pg16/bin/pg_ctl \
  -D ~/pgdata \
  -l pg.log start

```

现在，检查：

```sql
$ ~/pg16/bin/psql postgres -c 'show server_version'
 server_version
----------------
 17devel
(1 row)

```

