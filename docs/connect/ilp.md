# InfluxDB Line Protocol

import Tabs from "@theme/Tabs"
import TabItem from "@theme/TabItem"

The InfluxDB Line Protocol (ILP) is a text protocol over TCP or UDP on port
9009.

It is designed to be a one-way protocol to insert data, focusing on simplicity
and performance.

This is how it compares with ways to insert data that we support:

| Protocol | Record Insertion Reporting | Data Insertion Performance |
| -------- | -------------------------- | -------------------------- |
| ILP | None | **Best** |
| CSV upload via HTTP REST | Configurable | Very Good |
| PostgreSQL insert statements | Transaction-level | Good |

No errors and confirmations get sent back, but with sufficient client-side
validation this isn't necessarily a concern.

This interface is the preferred ingestion method as it provides the following
benefits:

- high-throughput ingestion
- robust ingestion from multiple sources into tables with dedicated systems for
  reducing congestion
- configurable commit-lag for out-of-order data via
  [server configuration](/docs/reference/configuration/#influxdb-line-protocol-tcp)
  settings

For additional details on the message format, ports and authentication can be
found on the [InfluxDB line protocol](/docs/reference/api/ilp/overview/) page,
and a guide on the Telegraf agent for collecting and sending metrics to QuestDB
via this protocol can be found on the
[Telegraf guide](/docs/third-party-tools/telegraf/).

## ILP Client Libraries

We have client libraries for a growing number of languages:

* **C and C++**: [https://github.com/questdb/c-questdb-client](https://github.com/questdb/c-questdb-client)

* **Java**: [https://search.maven.org/artifact/org.questdb/questdb](https://search.maven.org/artifact/org.questdb/questdb)

* **C#**: [https://github.com/questdb/net-questdb-client](https://github.com/questdb/net-questdb-client)

* For other languages, we have examples and a [protocol reference](/docs/reference/api/ilp/overview/).

## Examples

These examples send a few rows of input. The use client libraries for C++, C#,
Java and C, and raw TCP socket connections for NodeJS, Go and Python.

<Tabs defaultValue="cpp" values={[
  { label: "C++", value: "cpp" },
  { label: "C#", value: "csharp" },
  { label: "Java", value: "java" },
  { label: "C", value: "c" },
  { label: "NodeJS", value: "nodejs" },
  { label: "Go", value: "go" },
  { label: "Python", value: "python" }
]}>


<TabItem value="cpp">

```cpp
// https://github.com/questdb/c-questdb-client

#include <questdb/ilp/line_sender.hpp>
#include <iostream>

using namespace questdb::ilp::literals;

int main()
{
    try
    {
        questdb::ilp::line_sender sender{"localhost", 9009};

        // We prepare all our table names and colum names in advance.
        // If we're inserting multiple rows, this allows us to avoid
        // re-validating the same strings over and over again.
        auto table_name = "trades"_name;
        auto name_name = "name"_name;
        auto value_name = "value"_name;

        sender
            .table(trades_name)
            .symbol(name_name, "test_ilp1"_utf8)
            .column(value_name, 12.4)
            .at_now();
        sender
            .table(trades_name)
            .symbol(name_name, "test_ilp2"_utf8)
            .column(value_name, 11.4)
            .at_now();

        sender.flush();

        return 0;
    }
    catch (const questdb::ilp::line_sender_error& err)
    {
        std::cerr
            << "Error running example: "
            << err.what()
            << std::endl;

        return 1;
    }
}

```

</TabItem>

<TabItem value="csharp">

```csharp
// https://github.com/questdb/net-questdb-client

using QuestDB;

namespace QuestDBDemo
{
    class Program
    {
        static void Main(string[] args)
        {
            var address = IPAddress.Loopback.ToString();
            using var sender = new LineTcpSender(address, 9009);

            sender
                .Table("trades")
                .Symbol("name", "test_ilp1")
                .Column("value", 12.4)
                .AtNow();
            sender
                .Table("trades")
                .Symbol("name", "test_ilp2")
                .Column("value", 11.4)
                .AtNow();
            sender.flush();
        }
    }
}

```

</TabItem>

<TabItem value="java">


```java
/*
    https://search.maven.org/artifact/org.questdb/questdb

    Maven:
        <dependency>
            <groupId>org.questdb</groupId>
            <artifactId>questdb</artifactId>
            <version>6.2.1</version>
        </dependency>

    Gradle:
        compile group: 'org.questdb', name: 'questdb', version: '6.2.1'
*/

import io.questdb.cutlass.line.LineTcpSender;
import io.questdb.network.Net;
import io.questdb.std.Os;

public class LineTCPSenderMain {
    public static void main(String[] args) {
        int host = Net.parseIPv4("127.0.0.1");
        int port = 9009;
        int bufferCapacity = 256 * 1024;

        try (LineTcpSender sender = new LineTcpSender(host, port, bufferCapacity)) {
            sender
                    .metric("trades")
                    .tag("name", "test_ilp1")
                    .field("value", 12.4)
                    .$(Os.currentTimeNanos());
            sender
                    .metric("trades")
                    .tag("name", "test_ilp2")
                    .field("value", 11.4)
                    .$(Os.currentTimeNanos());

            sender.flush();
        }
    }
}
```

</TabItem>

<TabItem value="c">

```c
// https://github.com/questdb/c-questdb-client

#include <questdb/ilp/line_sender.h>
#include <stdio.h>

int main()
{
    line_sender_error* err = NULL;
    line_sender* sender = NULL;

    sender = line_sender_connect(
        "0.0.0.0",    // bind to all outbound network interfaces
        "localhost",  // QuestDB host
        "9009",       // QuestDB port
        &err);
    if (!sender)
        goto on_error;

    // We prepare all our table names and colum names in advance.
    // If we're inserting multiple rows, this allows us to avoid
    // re-validating the same strings over and over again.
    line_sender_name table_name;
    if (!line_sender_name_init(&table_name, 6, "trades", &err))
      goto on_error;

    line_sender_name name_name;
    if (!line_sender_name_init(&name_name, 4, "name", &err))
        goto on_error;

    line_sender_name value_name;
    if (!line_sender_name_init(&value_name, 5, "value", &err))
        goto on_error;


    line_sender_utf8 test_ilp3_utf8;
    if (!line_sender_utf8_init(&test_ilp2_utf8, 9, "test_ilp2", &err))
        goto on_error;

    // Prepare the first row.
    if (!line_sender_table(sender, table_name, &err))
        goto on_error;

    line_sender_utf8 test_ilp1_utf8;
    if (!line_sender_utf8_init(&test_ilp1_utf8, 9, "test_ilp1", &err))
        goto on_error;

    if (!line_sender_symbol(sender, name_name, test_ilp1_utf8, &err))
        goto on_error;

    if (!line_sender_column_f64(sender, value_name, 12.4, &err))
        goto on_error;

    if (!line_sender_at_now(sender, &err))
        goto on_error;


    // Prepare second row.
    if (!line_sender_table(sender, table_name, &err))
        goto on_error;

    line_sender_utf8 test_ilp2_utf8;
    if (!line_sender_utf8_init(&test_ilp2_utf8, 9, "test_ilp2", &err))
        goto on_error;

    if (!line_sender_symbol(sender, name_name, test_ilp2_utf8, &err))
        goto on_error;

    if (!line_sender_column_f64(sender, value_name, 11.4, &err))
        goto on_error;

    if (!line_sender_at_now(sender, &err))
        goto on_error;


    // Send.
    if (!line_sender_flush(sender, &err))
        goto on_error;

    line_sender_close(sender);

    return true;

on_error: ;
    size_t err_len = 0;
    const char* err_msg = line_sender_error_msg(err, &err_len);
    fprintf(stderr, "Error running example: %.*s\n", (int)err_len, err_msg);
    line_sender_error_free(err);
    if (sender)
        line_sender_close(sender);
    return 0;
}
```

</TabItem>

<TabItem value="nodejs">


```javascript
// Raw socket connection with no validation and string quoting logic.
// Refer to protocol description:
// http://questdb.io/docs/reference/api/ilp/overview

"use strict"

const net = require("net")

const client = new net.Socket()

const HOST = "localhost"
const PORT = 9009

function run() {
  client.connect(PORT, HOST, () => {
    const rows = [
      `trades,name=test_ilp1 value=12.4 ${Date.now() * 1e6}`,
      `trades,name=test_ilp2 value=11.4 ${Date.now() * 1e6}`,
    ]

    function write(idx) {
      if (idx === rows.length) {
        client.destroy()
        return
      }

      client.write(rows[idx] + "\n", (err) => {
        if (err) {
          console.error(err)
          process.exit(1)
        }
        write(++idx)
      })
    }

    write(0)
  })

  client.on("error", (err) => {
    console.error(err)
    process.exit(1)
  })

  client.on("close", () => {
    console.log("Connection closed")
  })
}

run()
```

</TabItem>


<TabItem value="go">


```go
// Raw socket connection with no validation and string quoting logic.
// Refer to protocol description:
// http://questdb.io/docs/reference/api/ilp/overview

package main

import (
  "fmt"
  "io/ioutil"
  "net"
  "time"
)

func main() {
  host := "127.0.0.1:9009"
  tcpAddr, err := net.ResolveTCPAddr("tcp4", host)
  checkErr(err)
  rows := [2]string{
    fmt.Sprintf("trades,name=test_ilp1 value=12.4 %d", time.Now().UnixNano()),
    fmt.Sprintf("trades,name=test_ilp2 value=11.4 %d", time.Now().UnixNano()),
  }

  conn, err := net.DialTCP("tcp", nil, tcpAddr)
  checkErr(err)
  defer conn.Close()

  for _, s := range rows {
    _, err = conn.Write([]byte(fmt.Sprintf("%s\n", s)))
    checkErr(err)
  }

  result, err := ioutil.ReadAll(conn)
  checkErr(err)

  fmt.Println(string(result))
}

func checkErr(err error) {
  if err != nil {
    panic(err)
  }
}
```

</TabItem>

<TabItem value="python">

```python
# Raw socket connection with no validation and string quoting logic.
# Refer to protocol description:
# http://questdb.io/docs/reference/api/ilp/overview

import time
import socket
import sys

HOST = 'localhost'
PORT = 9009
# For UDP, change socket.SOCK_STREAM to socket.SOCK_DGRAM
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

def send_utf8(msg):
    sock.sendall(msg.encode())

try:
    sock.connect((HOST, PORT))
    # Single record insert
    send_utf8(f'trades,name=client_timestamp value=12.4 {time.time_ns()}\n')
    # Omitting the timestamp allows the server to assign one
    send_utf8('trades,name=server_timestamp value=12.4\n')
    # Streams of readings must be newline-delimited
    send_utf8('trades,name=ilp_stream_1 value=12.4\n' +
              'trades,name=ilp_stream_2 value=11.4\n')

except socket.error as e:
    sys.stderr.write(f'Got error: {e}')

sock.close()
```

</TabItem>

</Tabs>

## ILP Datatypes and Casts

### Strings vs Symbols
Strings may be recorded as either the `STRING` type or the `SYMBOL` type.

Inspecting a sample ILP we can see how a space `' '` separator splits
`SYMBOL` columns to the left from the rest of the columns.

```text
table_name,col1=symbol_val1,col2=symbol_val2 col3="string val",col4=10.5
                                            ┬ 
                                            ╰───────── separator
```

In this example, columns `col1` and `col2` are strings written to the database
as `SYMBOL`s, whilst `col3` is written out as a `STRING`.

`SYMBOL`s are strings with which are automatically
[interned](https://en.wikipedia.org/wiki/String_interning) by the database on a
per-column basis.
You should use this type if you expect the string to be re-used over and over,
such as is common with identifiers.

For one-off strings use `STRING` columns which aren't interned.

### Casts

QuestDB types are a superset of those supported by ILP.
This means that when sending data you should be aware of the performed
conversions.

See:
* [QuestDB Types in SQL](/docs/reference/sql/datatypes)
* [ILP types and cast conversion tables](/docs/reference/sql/datatypes)

## Constructing well-formed messages

Different library implementations will perform different degrees content
validation upfront before sending messages out. To avoid encoutering issues
follow these guidelines.

* **All strings must be UTF-8 encoded.**

* **Columns should only appear once per row.**
  If ignored, the first value used and following column values dropped.

* **Symbol columns must be written out before other columns.**

* **Table and column names can't have invalid characters.**
  These should not contain `?`, `.`,`,`, `'`, `"`, `\`,
  `/`, `:`, `(`, `)`, `+`, `-`, `*`, `%`, `~`,`' '` (space),
  `\0` (nul terminator),
  [ZERO WIDTH NO-BREAK SPACE](https://unicode-explorer.com/c/FEFF).

* **Write timestamp column via designated API**, or at the end of the message
  if you are using raw sockets. If you have multiple timestamp columns
  write additional ones as column values.

* **Don't change column type between rows.** The whole row will be dropped,
  unless we can cast.

* **Supply timestamps in order.** These need to be at least equal to previous
  ones in the same table, unless using the out of order feature.
  If the feature is turned off, out of order rows will be dropped.

*Check [server logs](/docs/concept/root-directory-structure#log-directory)
for errors.*

## Inserting NULL values

To insert a NULL value, skip the column (or symbol) for that row.

For example:

```text
table1 a=10.5 1647357688714369403
table1 b=1.25 1647357698714369403
```

Will insert as:

|a   |b   |timestamp                  |
|----|----|---------------------------|
|10.5|*NULL*|2022-03-15T15:21:28.714369Z|
|*NULL*|1.25|2022-03-15T15:21:38.714369Z|

## If you don't immediately see data

If you don't see your inserted data, this is usually down to one of two things:

* You prepared the messages, but forgot to call `.flush()` or similar in your
  client library, so no data was sent.

* The internal timers and buffers within QuestDB did not commit the data yet.
  For development (and development only), you may want to tweak configuration
  settings to commit data more frequently.
  ```ini title=server.conf
  cairo.max.uncommitted.rows=1
  line.tcp.maintenance.job.interval=100
  ```
  Refer to
  [ILP's commit strategy](/docs/reference/api/ilp/tcp-receiver/#commit-strategy)
  documentation for more on these configuration settings.

## Authentication

ILP can additionally provide authentication. This is an optional feature
which is documented [here](/docs/reference/api/ilp/authenticate).
