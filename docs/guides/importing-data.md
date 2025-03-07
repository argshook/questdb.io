---
title: Importing data in bulk via CSV
sidebar_label: Bulk CSV imports
description:
  This document describes how to load CSV data and specify text loader
  configuration for timestamp and date parsing
---

The REST API provides an `/imp` endpoint exposed on port `9000` by default. This
endpoint streams tabular text data directly into a table, supporting CSV, TAB
and pipe (`|`) delimited inputs with optional headers. Data types and structures
are detected automatically, but additional configuration can be provided to
improve automatic detection.

## Specifying a schema during CSV import

A `schema` JSON object can be provided with POST requests to `/imp` while
creating tables via CSV import. This allows for more control over user-defined
patterns for timestamps, or for explicitly setting types during column-creation.
The the following example demonstrates basic usage, in this case, that the
`ticker_name` column should be parsed as `SYMBOL` type instead of `STRING`:

```bash
curl -F schema='[{"name":"ticker_name", "type": "SYMBOL"}]' \
-F data=@trades.csv 'http://localhost:9000/imp'
```

If a timestamp column (`ts`) in this CSV file has a custom or non-standard
timestamp format, this may be included with the call as follows:

```bash
curl -F schema='[
{"name":"ts", "type": "TIMESTAMP", "pattern": "yyyy-MM-dd - HH:mm:ss"},
{"name":"ticker_name", "type": "SYMBOL"}
]' -F data=@trades.csv 'http://localhost:9000/imp'
```

For **nanosecond-precision** timestamps such as
`2021-06-22T12:08:41.077338934Z`, a pattern can be provided in the following
way:

```bash
curl -F schema='[
{"name":"ts", "type": "TIMESTAMP", "pattern": "yyyy-MM-ddTHH:mm:ss.SSSUUUNNNZ"}
]' -F data=@my_file.csv 'http://localhost:9000/imp'
```

More information on the patterns for timestamps can be found on the
[date and time functions](/docs/reference/function/date-time/#date-and-timestamp-format)
page.

:::info

The `schema` object must precede the `data` object in calls to this REST
endpoint. For example:

```bash
# correct order
curl -F schema='{my_schema_obj}' -F data=@my_file.csv http://localhost:9000/imp
# incorrect order
curl -F data=@my_file.csv -F schema='{my_schema_obj}' http://localhost:9000/imp
```

:::

## Text loader configuration

QuestDB uses a `text_loader.json` configuration file which can be placed in the
server's `conf` directory. This file does not exist by default, but has the
following implicit settings:

```json title="conf/text_loader.json"
{
  "date": [
    {
      "format": "dd/MM/y"
    },
    {
      "format": "yyyy-MM-dd HH:mm:ss"
    },
    {
      "format": "yyyy-MM-ddTHH:mm:ss.SSSz",
      "locale": "en-US",
      "utf8": false
    },
    {
      "format": "MM/dd/y"
    }
  ],
  "timestamp": [
    {
      "format": "yyyy-MM-ddTHH:mm:ss.SSSUUUz",
      "utf8": false
    }
  ]
}
```

### Example

Given a CSV file which contains timestamps in the format
`yyyy-MM-dd - HH:mm:ss.SSSUUU`, the following text loader configuration will
provide the correct timestamp parsing:

```json title="conf/text_loader.json"
{
  "date": [
    {
      "format": "dd/MM/y"
    },
    {
      "format": "yyyy-MM-dd HH:mm:ss"
    },
    {
      "format": "yyyy-MM-ddTHH:mm:ss.SSSz",
      "locale": "en-US",
      "utf8": false
    },
    {
      "format": "MM/dd/y"
    }
  ],
  "timestamp": [
    {
      "format": "yyyy-MM-ddTHH:mm:ss.SSSUUUz",
      "utf8": false
    },
    {
      "format": "yyyy-MM-dd - HH:mm:ss.SSSUUU",
      "utf8": false
    }
  ]
}
```

The CSV data can then be loaded via POST request, for example, using cURL:

```curl
curl -F data=@weather.csv 'http://localhost:9000/imp'
```

For more information on the `/imp` entry point, refer to the
[REST API documentation](/docs/reference/api/rest/#imp---import-data).

## Large datasets with out-of-order data

Using the `commitLag` and `batch` size parameters during `INSERT AS SELECT`
statements is a convenient strategy to load and order large datasets from CSV in
bulk when they contain out-of-order data.

The batch size specifies how many records to attempt to bulk insert at one time
and the **lag** allows for specifying the expected lateness of out-of-order
timestamp values (in microseconds):

```questdb-sql
INSERT batch 100000 commitLag 180s INTO my_table
SELECT * FROM unordered_table
```

### Examples

Given a data set `weather-unordered.csv` which contains out-of-order records, an
ordered table may be created by setting `commitLag` and `maxUncommittedRows` via
REST API using the `/imp` endpoint. The following example imports a file which
contains out-of-order records:

```shell
curl -F data=@weather-unordered.csv \
'http://localhost:9000/imp?&timestamp=ts&partitionBy=DAY&commitLag=120000000&maxUncommittedRows=10000'
```

:::info

The `timestamp` and `partitionBy` parameters **must be provided** for commit lag
and max uncommitted rows to have any effect in the API call above.

:::

Alternatively, it's possible to create an ordered table via `INSERT AS SELECT`.
Given an existing table 'weather-unordered' which contains out-of-order records:

1. Create a new table with the schema of the existing table and apply a
   partitioning strategy. Records are not yet inserted due to the use of
   `WHERE 1 != 1`. The `timestamp` column may be cast as a `timestamp` if the
   import did not automatically detect the correct format:

   ```questdb-sql
   CREATE TABLE weather AS (
   SELECT
     cast(timestamp AS timestamp) timestamp,
     windDir,
     windSpeed,
     windGust,
     rain1H,
     rain6H,
     rain24H
   FROM 'weather-unordered' WHERE 1 != 1
   ) timestamp(timestamp) PARTITION BY DAY;
   ```

2. Insert the unordered records into the partitioned table and provide a
   `commitLag` and `batch` size:

   ```questdb-sql
   INSERT batch 100000 commitLag 180s INTO weather
   SELECT
     cast(timestamp AS timestamp) timestamp,
     windDir,
     windSpeed,
     windGust,
     rain1H,
     rain6H,
     rain24H
   FROM 'weather-unordered';
   ```

To confirm that the table is ordered, the `isOrdered()` function may be used:

```questdb-sql
select isOrdered(timestamp) from weather
```

| isOrdered |
| --------- |
| true      |

More information about the use of `isOrdered()` can be found on the
[boolean functions documentation](/docs/reference/function/boolean/).
