# HTTP REST API

import Tabs from "@theme/Tabs"
import TabItem from "@theme/TabItem"

QuestDB exposes a REST API for compatibility with a wide range of libraries and
tools. The REST API is accessible on port `9000` and has the following
entrypoints:

|Entrypoint|HTTP Method|Description|
|----------|-----------|-----------|
|[`/imp`](/docs/reference/api/rest#imp---import-data)|POST|Import CSV data|
|[`/exp?query=..`](/docs/reference/api/rest#exp---export-data)|GET|Export SQL Query as CSV|
|[`/exec?query=..`](/docs/reference/api/rest#exec---execute-queries)|GET|Run SQL Query returning JSON result set|

For details such as content type, query parameters et cetera, refer to the
[REST API](/docs/reference/api/rest) docs.

Here we'll get you started with a few examples in different programming languages.

## Uploading Tabular Data
Let's assume you want to upload the following data into "

<Tabs defaultValue="csv" values={[
  { label: "CSV", value: "csv" },
  { label: "Table", value: "table" },
]}>

<TabItem value="csv">

```csv
a,b,c
d,e,f
```

TODO EXAMPLE WITH FORMAT

</TabItem>

<TabItem value="table">

|a|b|c|
|-|-|-|
|d|e|f|

</TabItem>

</Tabs>

<Tabs defaultValue="cpp" values={[
  { label: "C++", value: "cpp" }
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

</Tabs>