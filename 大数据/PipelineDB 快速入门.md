# PipelineDB 快速入门

## 背景

PipelineDB基于PostgreSQL数据库改造而来，是一款开源的流式计算数据库。它允许我们通过sql的方式，对数据流做操作，并把操作结果持续储存到表中。

官方介绍:

> PipelineDB is built to run SQL queries continuously on streaming data. The output of these continuous queries is stored in regular tables which can be queried like any other table or view. Thus continuous queries can be thought of as very high-throughput, incrementally updated materialized views. As with any data processing system, PipelineDB is built to shine under particular workloads, and simply not intended for others.

主要特性：允许只使用 SQL 进行实时数据处理而没有应用代码，兼容 PostgreSQL，无 ETL，高效可持续。

大致理解就是数据不停的增量的更新，不用跑批处理。方便之处是流的处理是使用SQL语句，很方便构建数据口径，方便与业务人员交流。

## 安装

参照: http://docs.pipelinedb.com/installation.html

下面以OSX环境介绍，从[下载地址]( https://www.pipelinedb.com/download)获得安装包，双机安装。

下一步初始化PipelineDB，需指定一个目录用于存放数据文件。

```
pipeline-init -D <data directory>
```

接下来，使用`pipeline-ctl`运行PipelineDB服务。命令:

```
pipeline-ctl -D <data directory> -l pipelinedb.log start
```

`-l`参数用于指定log日志路径。

停止服务的命令:

```
pipeline-ctl -D <data directory> stop
```

更多关于`pipeline-ctl`参考命令`pipeline-ctl --help`。

另一种连接`pipelineDB`的方式是使用[postgresql](https://www.postgresql.org/download/)，注: PipelineDB的默认端口号是5432。命令是:

```
psql -p 5432 -h localhost pipeline
```

## 快速入门

初始化数据目录，启动PipelineDB服务:

```
pipeline-init -D <data directory>
pipeline-ctl -D <data directory> -l <log directory>
```

连接到pipeline db:

```
psql -h localhost -p 5432 -d pipeline
```

创建STREAM:

```
CREATE STREAM wiki_stream (hour timestamp, project text, title text, view_count bigint, size bigint);
```

创建CONTINUOUS VIEW:

```
CREATE CONTINUOUS VIEW wiki_stats AS
SELECT hour, project,
        count(*) AS total_pages,
        sum(view_count) AS total_views,
        min(view_count) AS min_views,
        max(view_count) AS max_views,
        avg(view_count) AS avg_views,
        percentile_cont(0.99) WITHIN GROUP (ORDER BY view_count) AS p99_views,
        sum(size) AS total_bytes_served
FROM wiki_stream
GROUP BY hour, project;
```

从外部获取数据实时写入pipelinedb(数据量很大，随时可以停止数据流入):

```
curl -sL http://pipelinedb.com/data/wiki-pagecounts | gunzip | \
        psql -h localhost -p 5432 -d pipeline -c "
        COPY wiki_stream (hour, project, title, view_count, size) FROM STDIN"
```

查询结果:

```
SELECT * FROM wiki_stats ORDER BY total_views DESC;
```

## 基础操作

* 连接pipelinedb

```
psql -h localhost -p 5432 -d pipeline
```

* 命令帮助

```
# psql命令帮助
\?
# SQL命令帮助
\h
```

* 列出Database

```
\l 或 \l+
```

* 创建Schema

```
CREATE SCHEMA dw_bihell AUTHORIZATION username;
```

* 列出Schema

```
\dn 或 \dn+
```

* 切换Schema

```
SET search_path TO dw_bihell;
```

* 列出表、视图等

```
# 默认shema下的
\d 或 \d+
# 指定shema
\dp [PATTERN] 或 \z [PATTERN]
比如 \z dw_order.*
```

## 核心概念

### Streams

streams是Continuous Views的数据入口，向view推送数据，可以看做是常规数据表的一行数据，或者当做一个事件。

但与常规数据表的表行数据有着根本不同: 存在于stream中的事件在被所有的views消费以后就会‘消失’，无法被用户通过`select`语句查询到，即steam专门作为Continuous Views的数据输入源而存在。

* 创建语法:

> CREATE STREAM stream_name ( [
>         { column_name data_type [ COLLATE collation ] | LIKE parent_stream } [, ... ]
> ] )

```
-- 可以直接支持json数据
CREATE STREAM dw_bihell.rt_oreder_bihell (log json);

-- 或者直接接收文本(kafka发数据的时候根据分隔符分割行)
CREATE STREAM dw_bihell.rt_oreder_bihell (collect_date text,record_status integer,operate_type integer,update_mask integer,order_date text,bill_date text,order_id bigint,order_type)
```

* 增加字段

```
ALTER STREAM stream ADD COLUMN x integer;
```

* 删除

```
DROP STREAM
```

* 查询已创建STREAM

```
SELECT * FROM pipeline_streams() ORDER BY schema;
```

### Continuous Views

Continuous Views是PipelineDB的基础核心概念，类似view，从stream和table中获得输入数据，增量的、实时的更新数据。

* 创建语法

```
CREATE CONTINUOUS VIEW name AS query
```

其中 query是一个pg 的select 格式的语法，格式如下：

```
SELECT [ DISTINCT [ ON ( expression [, ...] ) ] ]
    expression [ [ AS ] output_name ] [, ...]
    [ FROM from_item [, ...] ]
    [ WHERE condition ]
    [ GROUP BY expression [, ...] ]
    [ WINDOW window_name AS ( window_definition ) [, ...] ]

where from_item can be one of:

    stream_name [ [ AS ] alias [ ( column_alias [, ...] ) ] ]
    table_name [ [ AS ] alias [ ( column_alias [, ...] ) ] ]
    from_item [ NATURAL ] join_type from_item [ ON join_condition ]
```

* 删除操作

```
DROP CONTINUOUS VIEW name
```

* 清数据操作

```
SELECT truncate_continuous_view('name');
```

* 查看数据操作

```
SELECT * FROM pipeline_views();
```

* 暂定/启动操作

```
SELECT activate('continuous_view_or_transform');
SELECT deactivate('continuous_view_or_transform');
```

> #important# 暂停操作会导致丢失数据，即使重新启动后，也不能读到暂停期间的流入数据

## Continuous Transforms

Continuous Transforms是传输通道，不存储数据，也不支持聚合数据操作。一般内用来做stream的通道，或者将数据流转/保存到一张实体表中。

* 创建语法

```
CREATE CONTINUOUS TRANSFORM name AS query [ THEN EXECUTE PROCEDURE function_name ( arguments ) ]
```

* 删除操作

```
DROP CONTINUOUS TRANSFORM name
```

* 查看transform:

```
SELECT * FROM pipeline_transforms();
```

* 内置的transform触发器

```
CREATE CONTINUOUS TRANSFORM t AS
  SELECT x::int, y::int FROM stream WHERE mod(x, 2) = 0
  THEN EXECUTE PROCEDURE pipeline_stream_insert('even_stream');
```

* 自定义触发器

```
CREATE TABLE t (user text, value int);

CREATE OR REPLACE FUNCTION insert_into_t()
  RETURNS trigger AS
  $$
  BEGIN
    INSERT INTO t (user, value) VALUES (NEW.user, NEW.value);
    RETURN NEW;
  END;
  $$
  LANGUAGE plpgsql;

CREATE CONTINUOUS TRANSFORM ct AS
  SELECT user::text, value::int FROM stream WHERE value > 100
  THEN EXECUTE PROCEDURE insert_into_t();
```

## 参考:

* http://docs.pipelinedb.com/introduction.html
* http://bihell.com/2018/07/16/pipelinedb-practice/
* https://ask.hellobi.com/blog/seng/6038

