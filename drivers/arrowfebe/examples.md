---
title: Examples
layout: default
parent: ArrowFEBE
grand_parent: Drivers
nav_order: 5
permalink: /drivers/arrowfebe/examples/
---

# ArrowFEBE — Examples & recipes

Copy-paste snippets for the common tasks. All assume the driver is installed and
loadable by name (`driver="arrowfebe"`) — see the [install guide]({{ '/install/' | relative_url }}). For the
full option reference behind each `db_kwargs`, see [`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/' | relative_url }}).

- [Connect and query to Arrow](#connect-and-query-to-arrow)
- [Connect with discrete options](#connect-with-discrete-options)
- [Straight to pandas / Polars / DuckDB](#straight-to-pandas--polars--duckdb)
- [Prepared statements & parameter binding](#prepared-statements--parameter-binding)
- [Bulk ingest an Arrow table](#bulk-ingest-an-arrow-table)
- [Export a query to Parquet](#export-a-query-to-parquet)
- [TLS with certificate verification](#tls-with-certificate-verification)
- [Integrated / Kerberos auth](#integrated--kerberos-auth)
- [C# (Apache.Arrow.Adbc)](#c-apachearrowadbc)

---

## Connect and query to Arrow

```python
import adbc_driver_manager.dbapi as dbapi

with dbapi.connect(
    driver="arrowfebe",
    db_kwargs={"uri": "postgresql://alice:<password>@localhost:5432/tpch?sslmode=require"},
    autocommit=True,
) as conn, conn.cursor() as cur:
    cur.execute("SELECT * FROM lineitem LIMIT 10")
    table = cur.fetch_arrow_table()     # pyarrow.Table, COPY-binary fast path
    print(table.schema)
```

## Connect with discrete options

Discrete options keep secrets out of a single string and override any `uri`:

```python
conn = dbapi.connect(driver="arrowfebe", db_kwargs={
    "adbc.arrowfebe.server":   "localhost",
    "adbc.arrowfebe.port":     "5432",
    "adbc.arrowfebe.database": "tpch",
    "adbc.arrowfebe.username": "alice",
    "adbc.arrowfebe.password": "<password>",
    "adbc.arrowfebe.sslmode":  "require",
}, autocommit=True)
```

Or the DB-API shortcut from the Python package:

```python
import arrowfebe_adbc
conn = arrowfebe_adbc.connect("localhost", "tpch", "alice", "<password>")
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
with conn.cursor() as cur:
    cur.execute("SELECT * FROM orders WHERE o_orderkey = $1", parameters=[42])
    row = cur.fetch_arrow_table()

    # Array binding via executemany (bind an Arrow batch of params)
    cur.executemany(
        "INSERT INTO t(a, b) VALUES ($1, $2)",
        seq_of_parameters=[(1, "x"), (2, "y"), (3, "z")],
    )
    conn.commit()
```

PostgreSQL uses `$1`, `$2`, … placeholders in the extended-query protocol.

## Bulk ingest an Arrow table

Native COPY FROM STDIN (binary) — pipe an Arrow table straight in. Use
`autocommit=True` for the single-connection `TRUNCATE`+ingest pattern:

```python
import pyarrow as pa

conn = dbapi.connect(driver="arrowfebe", db_kwargs={...}, autocommit=True)

table = pa.table({
    "id":   pa.array([1, 2, 3], pa.int32()),
    "name": pa.array(["a", "b", "c"], pa.string()),
})

with conn.cursor() as cur:
    cur.adbc_ingest("my_table", table, mode="create")   # create | append | replace | create_append
```

Ingest into a specific catalog/schema or a temp table via the statement options
`adbc.ingest.target_catalog`, `adbc.ingest.target_db_schema`,
`adbc.ingest.temporary` (see [`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/#statement--ingest-options' | relative_url }})).
The ingest path introspects the target column OIDs, so an Arrow `utf8` column
lands correctly in a `jsonb` / `inet` / `uuid` column — see
[`DATA_TYPES.md`]({{ '/drivers/arrowfebe/data-types/#write-path-arrow--postgresql-ingest--bind' | relative_url }}).

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

## TLS with certificate verification

Verify the server certificate chain and hostname, and pin a custom root CA:

```python
conn = dbapi.connect(driver="arrowfebe", db_kwargs={
    "adbc.arrowfebe.server":        "db.example.com",
    "adbc.arrowfebe.database":      "mydb",
    "adbc.arrowfebe.username":      "alice",
    "adbc.arrowfebe.password":      "<password>",
    "adbc.arrowfebe.sslmode":       "verify-full",
    "adbc.arrowfebe.ssl_root_cert": "/etc/ssl/certs/pg-root.pem",
    # bind SCRAM to the TLS channel, refusing a PLUS-stripping downgrade:
    "adbc.arrowfebe.channel_binding": "require",
}, autocommit=True)
```

See [`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/#tls--sslmode' | relative_url }}) for the `sslmode` matrix.

## Integrated / Kerberos auth

No username/password — POSIX Kerberos or Windows SSPI:

```python
conn = dbapi.connect(driver="arrowfebe", db_kwargs={
    "adbc.arrowfebe.server":    "db.example.com",
    "adbc.arrowfebe.database":  "mydb",
    "adbc.arrowfebe.auth_type": "integrated",
    # optional explicit Kerberos credentials on Linux/macOS:
    # "adbc.arrowfebe.krb5.ccache": "/tmp/krb5cc_1000",
    # "adbc.arrowfebe.krb5.keytab": "/etc/postgresql/pg.keytab",
    # encrypt without TLS (POSIX/GSSAPI builds):
    # "adbc.arrowfebe.gssencmode": "require",
})
```

See [`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/' | relative_url }}) for SPN and credential details.

## C# (Apache.Arrow.Adbc)

```csharp
using Apache.Arrow.Adbc;
using Apache.Arrow.Adbc.C;

AdbcDriver driver = CAdbcDriverImporter.Load(
    "libarrowfebe_adbc_driver.so", "AdbcDriverInit");

AdbcDatabase database = driver.Open(new Dictionary<string, string>
{
    ["uri"] = "postgresql://alice:<password>@localhost:5432/tpch?sslmode=require",
});
AdbcConnection connection = database.Connect(new Dictionary<string, string>());

AdbcStatement statement = connection.CreateStatement();
statement.SqlQuery = "SELECT * FROM lineitem LIMIT 10";
QueryResult result = statement.ExecuteQuery();

using var stream = result.Stream;                  // IArrowArrayStream of RecordBatches
var batch = await stream.ReadNextRecordBatchAsync();
```

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/' | relative_url }}) — every connection option
- [`DATA_TYPES.md`]({{ '/drivers/arrowfebe/data-types/' | relative_url }}) — the Arrow types you get back
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowfebe/troubleshooting/' | relative_url }}) — when a snippet does not connect
