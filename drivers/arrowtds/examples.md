---
title: Examples
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 5
permalink: /drivers/arrowtds/examples/
---

# ArrowTDS — Examples & recipes

Copy-paste snippets for the common tasks. All assume the driver is installed and
loadable by name (`driver="arrowtds"`) — see [Install]({{ '/install/' | relative_url }}). For the full
option reference behind each `db_kwargs`, see [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}).

- [Connect and query to Arrow](#connect-and-query-to-arrow)
- [Connect with discrete options](#connect-with-discrete-options)
- [Straight to pandas / Polars / DuckDB](#straight-to-pandas--polars--duckdb)
- [Prepared statements & parameter binding](#prepared-statements--parameter-binding)
- [Bulk ingest an Arrow table](#bulk-ingest-an-arrow-table)
- [Export a query to Parquet](#export-a-query-to-parquet)
- [Integrated / Kerberos auth](#integrated--kerberos-auth)
- [Azure SQL with Entra ID](#azure-sql-with-entra-id)
- [C# (Apache.Arrow.Adbc)](#c-apachearrowadbc)

---

## Connect and query to Arrow

```python
import adbc_driver_manager.dbapi as dbapi

with dbapi.connect(
    driver="arrowtds",
    db_kwargs={"uri": "sqlserver://sa:<password>@localhost:1433/"
                      "?database=tpch&encrypt=true&TrustServerCertificate=true"},
) as conn, conn.cursor() as cur:
    cur.execute("SELECT TOP 10 * FROM lineitem")
    table = cur.fetch_arrow_table()     # pyarrow.Table
    print(table.schema)
```

## Connect with discrete options

Discrete options keep secrets out of a single string and override any `uri`:

```python
conn = dbapi.connect(driver="arrowtds", db_kwargs={
    "adbc.arrowtds.server":            "localhost",
    "adbc.arrowtds.database":          "tpch",
    "adbc.arrowtds.username":          "sa",
    "adbc.arrowtds.password":          "<password>",
    "adbc.arrowtds.encrypt":           "true",
    "adbc.arrowtds.trust_server_cert": "true",
})
```

Or the DB-API shortcut from the Python package:

```python
import arrowtds_adbc
conn = arrowtds_adbc.connect("localhost", "tpch", "sa", "<password>")
```

## Straight to pandas / Polars / DuckDB

Because results are already Arrow, there is no row-by-row marshalling:

```python
with conn.cursor() as cur:
    cur.execute("SELECT * FROM orders WHERE o_orderdate >= '1996-01-01'")

    df   = cur.fetch_arrow_table().to_pandas()   # pandas
    # or, without materialising twice:
    import polars as pl
    pf   = pl.from_arrow(cur.fetch_arrow_table()) # Polars
```

```python
import duckdb
tbl = conn.cursor().execute("SELECT * FROM lineitem").fetch_arrow_table()
duckdb.sql("SELECT l_returnflag, count(*) FROM tbl GROUP BY 1").show()
```

## Prepared statements & parameter binding

```python
import pyarrow as pa

with conn.cursor() as cur:
    cur.execute("SELECT * FROM orders WHERE o_orderkey = ?", parameters=[42])
    row = cur.fetch_arrow_table()

    # Array binding via the low-level ADBC statement (bind an Arrow batch of params)
    cur.executemany(
        "INSERT INTO t(a, b) VALUES (?, ?)",
        seq_of_parameters=[(1, "x"), (2, "y"), (3, "z")],
    )
    conn.commit()
```

## Bulk ingest an Arrow table

Native TDS `INSERT BULK` — pipe an Arrow table straight in. Use
`autocommit=True` for the single-connection `TRUNCATE`+ingest pattern (the default
`autocommit=False` enables `IMPLICIT_TRANSACTIONS` and can deadlock the bulk path):

```python
import pyarrow as pa

conn = dbapi.connect(driver="arrowtds", db_kwargs={...}, autocommit=True)

table = pa.table({
    "id":   pa.array([1, 2, 3], pa.int32()),
    "name": pa.array(["a", "b", "c"], pa.string()),
})

with conn.cursor() as cur:
    cur.adbc_ingest("my_table", table, mode="create")   # create | append | replace | create_append
```

Ingest into a specific catalog/schema or a temp table via the statement options
`adbc.ingest.target_catalog`, `adbc.ingest.target_db_schema`,
`adbc.ingest.temporary` (see [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}#statement--ingest-options)).

## Export a query to Parquet

```python
import pyarrow.parquet as pq

with conn.cursor() as cur:
    cur.execute("SELECT * FROM lineitem")
    reader = cur.fetch_record_batch()            # streaming RecordBatchReader
    with pq.ParquetWriter("lineitem.parquet", reader.schema) as w:
        for batch in reader:
            w.write_batch(batch)
```

Streaming keeps memory flat regardless of result size; the driver auto-flushes a
batch early if a wide column would cross Arrow's 2 GiB offset limit.

## Integrated / Kerberos auth

No username/password — Windows SSPI or POSIX Kerberos:

```python
conn = dbapi.connect(driver="arrowtds", db_kwargs={
    "adbc.arrowtds.server":   "sql.corp.example.com",
    "adbc.arrowtds.database": "mydb",
    "adbc.arrowtds.trusted":  "true",
    # optional explicit Kerberos credentials on Linux/macOS:
    # "adbc.arrowtds.krb5.ccache": "/tmp/krb5cc_1000",
    # "adbc.arrowtds.krb5.keytab": "/etc/krb5.keytab",
})
```

See [`AUTHENTICATION.md`]({{ '/drivers/arrowtds/authentication/' | relative_url }}) for SPN and credential details.

## Azure SQL with Entra ID

Access-token passthrough (see [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}#azure-sql-microsoft-fabric--entra-id)
for service-principal / managed-identity / default-chain variants):

```python
import subprocess
token = subprocess.check_output(
    ["az", "account", "get-access-token",
     "--resource", "https://database.windows.net/",
     "--query", "accessToken", "-o", "tsv"]).decode().strip()

conn = dbapi.connect(driver="arrowtds", db_kwargs={
    "adbc.arrowtds.server":       "myserver.database.windows.net",
    "adbc.arrowtds.database":     "mydb",
    "adbc.arrowtds.encrypt":      "true",
    "adbc.arrowtds.access_token": token,
})
```

## C# (Apache.Arrow.Adbc)

```csharp
using Apache.Arrow.Adbc;
using Apache.Arrow.Adbc.C;

AdbcDriver driver = CAdbcDriverImporter.Load(
    "libarrowtds_adbc_driver.so", "AdbcDriverInit");

AdbcDatabase database = driver.Open(new Dictionary<string, string>
{
    ["uri"] = "sqlserver://sa:<password>@localhost:1433/?database=tpch&encrypt=true",
});
AdbcConnection connection = database.Connect(new Dictionary<string, string>());

AdbcStatement statement = connection.CreateStatement();
statement.SqlQuery = "SELECT TOP 10 * FROM lineitem";
QueryResult result = statement.ExecuteQuery();

using var stream = result.Stream;                  // IArrowArrayStream of RecordBatches
var batch = await stream.ReadNextRecordBatchAsync();
```

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}) — every connection option
- [`DATA_TYPES.md`]({{ '/drivers/arrowtds/data-types/' | relative_url }}) — the Arrow types you get back
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowtds/troubleshooting/' | relative_url }}) — when a snippet does not connect
