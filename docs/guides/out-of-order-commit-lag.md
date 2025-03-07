---
title: Configuring commit lag of out-of-order (O3) data
sidebar_label: Out-of-order commit lag
description:
  This document describes server configuration parameters for out-of-order
  commit-lag along with details when and why these settings should be applied
image: /img/guides/out-of-order-commit-lag/o3-data.jpeg
---

Server configuration may be applied when ingesting data over InfluxDB Line
Protocol (ILP) to allow user control on how the system processes and commits
late-arriving data for optimum throughput.

## Background

As of software version 6.0, QuestDB adds support for out-of-order (O3) data
ingestion. The skew and latency of out-of-order data are likely to be relatively
constant, so users may configure ingestion based on the characteristics of the
data.

Most real-time out-of-order data patterns are caused by the delivery mechanism
and hardware jitter, and therefore the timestamp distribution will be contained
by some boundary.

import Screenshot from "@theme/Screenshot"

<Screenshot
  alt="A diagram showing how data may arrive with random timings from clients due to network jitter or latency"
  height={334}
  src="/img/guides/out-of-order-commit-lag/o3-data.jpeg"
  title="Records with various network-induced delays"
  width={650}
/>

If any new timestamp value has a high probability to arrive within 10 seconds of
the previously received value, the boundary for this data is `10 seconds` and we
name this **commit lag** or just **lag**.

When the order of timestamp values follow this pattern, it will be recognized by
our out-of-order algorithm and prioritized using an optimized processing path. A
commit of this data re-orders uncommitted rows and then commits all rows up to
the boundary; the remaining rows stay in memory to be committed later.

## Out-of-order (O3) commit parameters

Commit parameters allow for specifying that commits of out-of-order data should
occur when:

- they are outside a window of time for which they are expected to be
  out-of-order or
- when the row count passes a certain threshold.

:::info

Commit parameters are user-configurable for ingestion using **InfluxDB line
protocol only**. This is the case as commits over Postgres wire protocol are
invoked client-side and commits via REST API occur either row-by-row or after a
CSV import is complete.

:::

The following server configuration parameters are user-configurable:

```bash
# the maximum number of uncommitted rows
cairo.max.uncommitted.rows=X
# the expected maximum time lag for out-of-order rows in milliseconds
cairo.commit.lag=X
```

The `cairo.max.uncommitted.rows` value defines row-based commit strategy. This
strategy means that the database will issue a commit when the number of
uncommitted rows reaches `cairo.max.uncommitted.rows`.

Apart from the row-based commit strategy, the ILP server also implements
interval-based and idle table timeout commit strategies. Refer to the
[ILP commit strategy](/docs/reference/api/ilp/tcp-receiver#commit-strategy) page
to learn more.

The `cairo.commit.lag` value is applied each time when a commit happens. As a
result, data older than the lag value will be committed and become visible.

## When to change out-of-order commit configuration

The defaults for the out-of-order algorithm are optimized for real-world usage
and should cover most patterns for timestamp arrival. The default configuration
is as follows:

```txt title="Defaults"
cairo.commit.lag=300000
cairo.max.uncommitted.rows=500000
```

Users should modify out-of-order parameters if there is a known or expected
pattern for:

1. The length of time by which most records are late
2. The frequency of incoming records and the expected throughput

For optimal ingestion performance, the number of commits of out-of-order data
should be minimized. For this reason, if throughput is low and timestamps are
expected to be consistently delayed up to 30 seconds, the following
configuration settings can be applied

```txt title="server.conf"
cairo.commit.lag=30000
cairo.max.uncommitted.rows=500
```

For high-throughput scenarios, lower commit timer and larger number of
uncommitted rows may be more appropriate. The following settings would assume a
throughput of ten thousand records per second with a likely maximum of 1 second
lateness for timestamp values:

```txt title="server.conf"
cairo.commit.lag=1000
cairo.max.uncommitted.rows=10000
```

## How to configure out-of-order ingestion

### Server-wide configuration

These settings may be applied via
[server configuration file](/docs/reference/configuration/):

```txt title="server.conf"
cairo.max.uncommitted.rows=500
cairo.commit.lag=10000
```

As with other server configuration parameters, these settings may be passed as
environment variables:

- `QDB_CAIRO_MAX_UNCOMMITTED_ROWS`
- `QDB_CAIRO_COMMIT_LAG`

To set this configuration for the current shell:

```bash title="Setting environment variables"
export QDB_CAIRO_MAX_UNCOMMITTED_ROWS=1000
export QDB_CAIRO_COMMIT_LAG=20000
questdb start
```

Passing the environment variables via Docker is done using the `-e` flag:

```bash
docker run -p 9000:9000 \
 -p 9009:9009 \
 -p 8812:8812 \
 -p 9003:9003 \
 -e QDB_CAIRO_MAX_UNCOMMITTED_ROWS=1000 \
 -e QDB_CAIRO_COMMIT_LAG=20000 \
 questdb/questdb
```

### Per-table commit lag and maximum uncommitted rows

It's possible to set out-of-order values per table when creating a new table as
part of the `PARTITION BY` clause. Configuration is passed using the `WITH`
keyword with the following two parameters:

- `maxUncommittedRows` - equivalent to `cairo.max.uncommitted.rows`
- `commitLag` - equivalent to `cairo.commit.lag`

```questdb-sql title="Setting out-of-order table parameters via SQL"
CREATE TABLE my_table (timestamp TIMESTAMP) timestamp(timestamp)
PARTITION BY DAY WITH maxUncommittedRows=250000, commitLag=240s;
```

Checking the values per-table may be done using the `tables()` function:

```questdb-sql title="List all tables"
select id, name, maxUncommittedRows, commitLag from tables();
```

| id  | name        | maxUncommittedRows | commitLag |
| --- | ----------- | ------------------ | --------- |
| 1   | my_table    | 250000             | 240000000 |
| 2   | device_data | 10000              | 30000000  |

The values can changed per each table with:

```questdb-sql title="Altering maximum number of out-of-order rows via SQL"
ALTER TABLE my_table SET PARAM maxUncommittedRows = 10000;
```

and

```questdb-sql title="Altering out-of-order commit lag via SQL"
ALTER TABLE my_table SET PARAM commitLag = 20s;
```

For more information on setting table parameters via SQL, see the
[SET PARAM](/docs/reference/sql/alter-table-set-param/) reference. Additional
details on checking table metadata is described in the
[meta functions](/docs/reference/function/meta/) documentation page.

### Out-of-order CSV import

It's also possible to set `commitLag` and `maxUncommittedRows` via REST API when
importing data via the `/imp` endpoint. The following example imports a file
which contains out-of-order records. The `timestamp` and `partitionBy`
parameters **must be provided** for commit lag and max uncommitted rows to have
any effect:

```shell
curl -F data=@weather.csv \
'http://localhost:9000/imp?&timestamp=ts&partitionBy=DAY&commitLag=120000000&maxUncommittedRows=10000'
```

### INSERT as SELECT with batch size and lag

The `INSERT` keyword may be
[passed parameters](/docs/reference/sql/insert/#parameters) for handling the
expected _lag_ of out-of-order records and a _batch_ size can be specified for
the number of rows to process and insert at once. The following query shows an
`INSERT AS SELECT` operation with lag and batch size applied:

```questdb-sql
INSERT batch 100000 commitLag 180s INTO trades
SELECT ts, instrument, quantity, price
FROM unordered_trades
```

:::info

Using the lag and batch size parameters during `INSERT AS SELECT` statements is
a convenient strategy to load and order large datasets from CSV in bulk. This
strategy along with an example workflow is described in the
[importing data guide](/docs/guides/importing-data/).

:::
