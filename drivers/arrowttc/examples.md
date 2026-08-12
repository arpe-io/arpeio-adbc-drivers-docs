---
title: Examples
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 5
permalink: /drivers/arrowttc/examples/
---

# ArrowTTC — Examples & recipes

Copy-paste snippets for the common tasks. All assume the driver is installed and
loadable by name (`driver="arrowttc"`) — see [Install]({{ '/install/' | relative_url }}). For the full
option reference behind each `db_kwargs`, see [`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}).

- [Connect and query to Arrow](#connect-and-query-to-arrow)
- [Connect with discrete options](#connect-with-discrete-options)
- [Straight to pandas / Polars / DuckDB](#straight-to-pandas--polars--duckdb)
- [Prepared statements & parameter binding](#prepared-statements--parameter-binding)
- [Bulk ingest an Arrow table](#bulk-ingest-an-arrow-table)
- [Export a query to Parquet](#export-a-query-to-parquet)
- [Native Network Encryption](#native-network-encryption)
- [TLS / Oracle Cloud ADB with a wallet](#tls--oracle-cloud-adb-with-a-wallet)
- [Kerberos auth](#kerberos-auth)
- [OCI IAM & OAuth2 / Entra ID token auth](#oci-iam--oauth2--entra-id-token-auth)
- [C# (Apache.Arrow.Adbc)](#c-apachearrowadbc)

---

## Connect and query to Arrow

```python
import adbc_driver_manager.dbapi as dbapi

with dbapi.connect(
    driver="arrowttc",
    db_kwargs={"uri": "oracle://scott:tiger@localhost:1521/orclpdb1"},
) as conn, conn.cursor() as cur:
    cur.execute("SELECT * FROM lineitem WHERE ROWNUM <= 10")
    table = cur.fetch_arrow_table()     # pyarrow.Table
    print(table.schema)
```

## Connect with discrete options

Discrete options keep secrets out of a single string and override any `uri`:

```python
conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "adbc.arrowttc.server":       "localhost",
    "adbc.arrowttc.port":         "1521",
    "adbc.arrowttc.service_name": "orclpdb1",
    "adbc.arrowttc.username":     "scott",
    "adbc.arrowttc.password":     "tiger",
})
```

Or the DB-API shortcut from the Python package:

```python
import arrowttc_adbc
conn = arrowttc_adbc.connect(server="localhost", service_name="orclpdb1",
                             username="scott", password="tiger")
```

## Straight to pandas / Polars / DuckDB

Because results are already Arrow, there is no row-by-row marshalling:

```python
with conn.cursor() as cur:
    cur.execute("SELECT * FROM orders WHERE o_orderdate >= DATE '1996-01-01'")

    df = cur.fetch_arrow_table().to_pandas()      # pandas
    # or, without materialising twice:
    import polars as pl
    pf = pl.from_arrow(cur.fetch_arrow_table())   # Polars
```

```python
import duckdb
tbl = conn.cursor().execute("SELECT * FROM lineitem").fetch_arrow_table()
duckdb.sql("SELECT l_returnflag, count(*) FROM tbl GROUP BY 1").show()
```

## Prepared statements & parameter binding

Oracle uses positional `:1`, `:2` bind placeholders:

```python
with conn.cursor() as cur:
    cur.execute("SELECT * FROM orders WHERE o_orderkey = :1", parameters=[42])
    row = cur.fetch_arrow_table()

    # Array binding — one Arrow row of parameters per execution
    cur.executemany(
        "INSERT INTO t(a, b) VALUES (:1, :2)",
        seq_of_parameters=[(1, "x"), (2, "y"), (3, "z")],
    )
    conn.commit()
```

## Bulk ingest an Arrow table

`adbc_ingest` pipes an Arrow table straight in via an array-bound `INSERT`:

```python
import pyarrow as pa

table = pa.table({
    "id":   pa.array([1, 2, 3], pa.int32()),
    "name": pa.array(["a", "b", "c"], pa.string()),
})

with conn.cursor() as cur:
    cur.adbc_ingest("my_table", table, mode="create")   # create | append | replace | create_append
conn.commit()
```

Ingest into a specific schema or a `GLOBAL TEMPORARY` table via the statement
options `adbc.ingest.target_db_schema` / `adbc.ingest.temporary` (see
[`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}#statement--ingest-options)).

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

Streaming keeps memory flat regardless of result size; batch size is a memory
knob (see the README `### Batch Size`).

## Native Network Encryption

Encrypt the whole session without TCPS — the way to connect a server configured
`SQLNET.ENCRYPTION_SERVER=REQUIRED`:

```python
conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "adbc.arrowttc.server":         "dbhost",
    "adbc.arrowttc.service_name":   "orclpdb1",
    "adbc.arrowttc.username":       "scott",
    "adbc.arrowttc.password":       "tiger",
    "adbc.arrowttc.encryption":     "required",   # AES-256 payloads
    "adbc.arrowttc.data_integrity": "required",   # SHA-256 checksum
})
```

## TLS / Oracle Cloud ADB with a wallet

Point `wallet_location` at an unzipped ADB wallet (mutual TLS):

```python
conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "adbc.arrowttc.server":          "adb.<region>.oraclecloud.com",
    "adbc.arrowttc.port":            "1522",
    "adbc.arrowttc.service_name":    "<svc>_low.adb.oraclecloud.com",
    "adbc.arrowttc.username":        "scott",
    "adbc.arrowttc.password":        "tiger",
    "adbc.arrowttc.ssl_mode":        "verify-full",
    "adbc.arrowttc.wallet_location": "/home/you/wallet",
    "adbc.arrowttc.wallet_password": "<wallet-pw>",
})
```

## Kerberos auth

No Oracle password — a Kerberos ticket (Linux / MIT krb5 only):

```python
conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "adbc.arrowttc.server":       "db.corp.example.com",
    "adbc.arrowttc.service_name": "orclpdb1",
    "adbc.arrowttc.auth_method":  "kerberos",
    "adbc.arrowttc.krb5_spn":     "oracle/db.corp.example.com",
    # "adbc.arrowttc.krb5_cred_mode": "ccache",  # or keytab / password
})
```

See [`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}#kerberos-kerberos5) for SPN and
credential details.

## OCI IAM & OAuth2 / Entra ID token auth

OCI IAM database token (`oci iam db-token get` writes the token directory); pair
with the ADB wallet for mutual TLS:

```python
db_kwargs = {
    "adbc.arrowttc.server":          "adb.<region>.oraclecloud.com",
    "adbc.arrowttc.port":            "1522",
    "adbc.arrowttc.service_name":    "<svc>_low.adb.oraclecloud.com",
    "adbc.arrowttc.auth_method":     "token",
    "adbc.arrowttc.token_location":  "/home/you/.oci/db-token",
    "adbc.arrowttc.ssl_mode":        "verify-full",
    "adbc.arrowttc.wallet_location": "/home/you/wallet",
    "adbc.arrowttc.wallet_password": "<wallet-pw>",
}
```

OAuth2 / Microsoft Entra ID bearer token — setting the token auto-selects the
method (no username; Oracle maps the token's `upn` claim to a global user):

```python
db_kwargs = {
    "adbc.arrowttc.server":       "adb.<region>.oraclecloud.com",
    "adbc.arrowttc.port":         "1522",
    "adbc.arrowttc.service_name": "<svc>_low.adb.oraclecloud.com",
    "adbc.arrowttc.access_token": "<entra-jwt>",   # or token_file=<path>
    # wallet options as above for ADB mutual TLS
}
```

See [`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}#oci-iam-token-oracle-cloud).

## C# (Apache.Arrow.Adbc)

```csharp
using Apache.Arrow.Adbc;
using Apache.Arrow.Adbc.C;

AdbcDriver driver = CAdbcDriverImporter.Load(
    "libarrowttc_adbc_driver.so", "AdbcDriverInit");

AdbcDatabase database = driver.Open(new Dictionary<string, string>
{
    ["uri"] = "oracle://scott:tiger@localhost:1521/orclpdb1",
});
AdbcConnection connection = database.Connect(new Dictionary<string, string>());

AdbcStatement statement = connection.CreateStatement();
statement.SqlQuery = "SELECT * FROM lineitem WHERE ROWNUM <= 10";
QueryResult result = statement.ExecuteQuery();

using var stream = result.Stream;                  // IArrowArrayStream of RecordBatches
var batch = await stream.ReadNextRecordBatchAsync();
```

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}) — every connection option
- [`DATA_TYPES.md`]({{ '/drivers/arrowttc/data-types/' | relative_url }}) — the Arrow types you get back
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowttc/troubleshooting/' | relative_url }}) — when a snippet does not connect
