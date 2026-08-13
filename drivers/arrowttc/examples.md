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

---

## Connect and query to Arrow

The same recipe in every ADBC client — the driver is loaded **by name** (`arrowttc`)
from the installed ADBC manifest, so only the language changes:

<div class="code-tabs" markdown="1">

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

```cpp
#include <cstdio>
#include <cstdlib>
#include <arrow-adbc/adbc.h>
#include <arrow-adbc/adbc_driver_manager.h>

#define CHECK(EXPR)                                                     \
  if (AdbcStatusCode s = (EXPR); s != ADBC_STATUS_OK) {                 \
    std::fprintf(stderr, "%s failed: %s\n", #EXPR,                      \
                 error.message ? error.message : "(unknown)");          \
    std::exit(1);                                                       \
  }

int main() {
  AdbcError error = {};
  AdbcDatabase database = {};
  CHECK(AdbcDatabaseNew(&database, &error));
  // Load the driver by name from the installed ADBC manifest.
  CHECK(AdbcDatabaseSetOption(&database, "driver", "arrowttc", &error));
  CHECK(AdbcDatabaseSetOption(
      &database, "uri",
      "oracle://scott:tiger@localhost:1521/orclpdb1", &error));
  CHECK(AdbcDatabaseInit(&database, &error));

  AdbcConnection connection = {};
  CHECK(AdbcConnectionNew(&connection, &error));
  CHECK(AdbcConnectionInit(&connection, &database, &error));

  AdbcStatement statement = {};
  CHECK(AdbcStatementNew(&connection, &statement, &error));
  CHECK(AdbcStatementSetSqlQuery(&statement, "SELECT * FROM lineitem WHERE ROWNUM <= 10", &error));

  ArrowArrayStream stream = {};            // Arrow C Data interface
  int64_t rows_affected = -1;
  CHECK(AdbcStatementExecuteQuery(&statement, &stream, &rows_affected, &error));
  // consume `stream` with nanoarrow / Arrow C++, then release:
  stream.release(&stream);
  AdbcStatementRelease(&statement, &error);
  AdbcConnectionRelease(&connection, &error);
  AdbcDatabaseRelease(&database, &error);
  return 0;
}
```

```csharp
using Apache.Arrow.Adbc;
using Apache.Arrow.Adbc.DriverManager;

var parameters = new Dictionary<string, string>
{
    ["uri"] = "oracle://scott:tiger@localhost:1521/orclpdb1",
};

// Load the driver by name from the installed ADBC manifest.
using AdbcDriver driver = AdbcDriverManager.FindLoadDriver(
    "arrowttc", loadOptions: AdbcLoadFlags.Default);
using AdbcDatabase database = driver.Open(parameters);
using AdbcConnection connection = database.Connect(null);
using AdbcStatement statement = connection.CreateStatement();

statement.SqlQuery = "SELECT * FROM lineitem WHERE ROWNUM <= 10";
QueryResult result = statement.ExecuteQuery();

using var stream = result.Stream!;       // IArrowArrayStream of RecordBatches
Console.WriteLine(stream.Schema);
```

```go
package main

import (
	"context"
	"fmt"

	"github.com/apache/arrow-adbc/go/adbc"
	"github.com/apache/arrow-adbc/go/adbc/drivermgr"
)

func main() {
	ctx := context.Background()

	var drv drivermgr.Driver
	// Load the driver by name from the installed ADBC manifest.
	db, err := drv.NewDatabase(map[string]string{
		"driver":          "arrowttc",
		adbc.OptionKeyURI: "oracle://scott:tiger@localhost:1521/orclpdb1",
	})
	if err != nil {
		panic(err)
	}
	defer db.Close()

	conn, err := db.Open(ctx)
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	stmt, err := conn.NewStatement()
	if err != nil {
		panic(err)
	}
	defer stmt.Close()

	if err := stmt.SetSqlQuery("SELECT * FROM lineitem WHERE ROWNUM <= 10"); err != nil {
		panic(err)
	}
	reader, _, err := stmt.ExecuteQuery(ctx) // (array.RecordReader, int64, error)
	if err != nil {
		panic(err)
	}
	defer reader.Release()

	fmt.Println(reader.Schema())
}
```

```java
import java.util.HashMap;
import java.util.Map;

import org.apache.arrow.adbc.core.AdbcConnection;
import org.apache.arrow.adbc.core.AdbcDatabase;
import org.apache.arrow.adbc.core.AdbcDriver;
import org.apache.arrow.adbc.core.AdbcStatement;
import org.apache.arrow.adbc.driver.jni.JniDriver;
import org.apache.arrow.memory.BufferAllocator;
import org.apache.arrow.memory.RootAllocator;
import org.apache.arrow.vector.ipc.ArrowReader;

public class ConnectAndQuery {
  public static void main(String[] args) throws Exception {
    Map<String, Object> parameters = new HashMap<>();
    // Load the driver by name from the installed ADBC manifest.
    JniDriver.PARAM_DRIVER.set(parameters, "arrowttc");
    AdbcDriver.PARAM_URI.set(parameters,
        "oracle://scott:tiger@localhost:1521/orclpdb1");

    try (BufferAllocator allocator = new RootAllocator();
        AdbcDatabase db = new JniDriver(allocator).open(parameters);
        AdbcConnection connection = db.connect();
        AdbcStatement statement = connection.createStatement()) {
      statement.setSqlQuery("SELECT * FROM lineitem WHERE ROWNUM <= 10");
      try (AdbcStatement.QueryResult result = statement.executeQuery()) {
        ArrowReader reader = result.getReader();
        System.out.println(reader.getVectorSchemaRoot().getSchema());
      }
    }
  }
}
```

```r
library(adbcdrivermanager)
library(arrow)

# Load the driver by name from the installed ADBC manifest.
db <- adbc_database_init(
  adbc_driver("arrowttc"),
  uri = "oracle://scott:tiger@localhost:1521/orclpdb1"
)
con <- adbc_connection_init(db)

table <- read_adbc(con, "SELECT * FROM lineitem WHERE ROWNUM <= 10") |>
  as_arrow_table()
print(table$schema)
```

```rust
use adbc_core::options::{AdbcVersion, OptionDatabase, OptionValue};
use adbc_core::{Connection, Database, Driver, Statement, LOAD_FLAG_DEFAULT};
use adbc_driver_manager::ManagedDriver;
use arrow_array::RecordBatchReader;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Load the driver by name from the installed ADBC manifest.
    let mut driver = ManagedDriver::load_from_name(
        "arrowttc", None, AdbcVersion::default(), LOAD_FLAG_DEFAULT, None,
    )?;

    let mut database = driver.new_database_with_opts([(
        OptionDatabase::Uri,
        OptionValue::String(
            "oracle://scott:tiger@localhost:1521/orclpdb1".into(),
        ),
    )])?;
    let mut connection = database.new_connection()?;
    let mut statement = connection.new_statement()?;
    statement.set_sql_query("SELECT * FROM lineitem WHERE ROWNUM <= 10")?;

    let reader = statement.execute()?;   // impl RecordBatchReader
    println!("{}", reader.schema());
    Ok(())
}
```

</div>

## Connect with discrete options

Discrete options keep secrets out of a single string and override any `uri`. Set them
where the `uri` went — the rest of each program is identical to the recipe above:

<div class="code-tabs" markdown="1">

```python
conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "adbc.arrowttc.server":       "localhost",
    "adbc.arrowttc.port":         "1521",
    "adbc.arrowttc.service_name": "orclpdb1",
    "adbc.arrowttc.username":     "scott",
    "adbc.arrowttc.password":     "tiger",
})
```

```cpp
CHECK(AdbcDatabaseNew(&database, &error));
CHECK(AdbcDatabaseSetOption(&database, "driver", "arrowttc", &error));
CHECK(AdbcDatabaseSetOption(&database, "adbc.arrowttc.server", "localhost", &error));
CHECK(AdbcDatabaseSetOption(&database, "adbc.arrowttc.port", "1521", &error));
CHECK(AdbcDatabaseSetOption(&database, "adbc.arrowttc.service_name", "orclpdb1", &error));
CHECK(AdbcDatabaseSetOption(&database, "adbc.arrowttc.username", "scott", &error));
CHECK(AdbcDatabaseSetOption(&database, "adbc.arrowttc.password", "tiger", &error));
CHECK(AdbcDatabaseInit(&database, &error));
```

```csharp
var parameters = new Dictionary<string, string>
{
    ["adbc.arrowttc.server"]       = "localhost",
    ["adbc.arrowttc.port"]         = "1521",
    ["adbc.arrowttc.service_name"] = "orclpdb1",
    ["adbc.arrowttc.username"]     = "scott",
    ["adbc.arrowttc.password"]     = "tiger",
};
using AdbcDriver driver = AdbcDriverManager.FindLoadDriver("arrowttc", loadOptions: AdbcLoadFlags.Default);
using AdbcDatabase database = driver.Open(parameters);
```

```go
db, err := drv.NewDatabase(map[string]string{
	"driver":                      "arrowttc",
	"adbc.arrowttc.server":        "localhost",
	"adbc.arrowttc.port":          "1521",
	"adbc.arrowttc.service_name":  "orclpdb1",
	"adbc.arrowttc.username":      "scott",
	"adbc.arrowttc.password":      "tiger",
})
```

```java
Map<String, Object> parameters = new HashMap<>();
JniDriver.PARAM_DRIVER.set(parameters, "arrowttc");
parameters.put("adbc.arrowttc.server", "localhost");
parameters.put("adbc.arrowttc.port", "1521");
parameters.put("adbc.arrowttc.service_name", "orclpdb1");
parameters.put("adbc.arrowttc.username", "scott");
parameters.put("adbc.arrowttc.password", "tiger");
AdbcDatabase db = new JniDriver(allocator).open(parameters);
```

```r
db <- adbc_database_init(
  adbc_driver("arrowttc"),
  adbc.arrowttc.server       = "localhost",
  adbc.arrowttc.port         = "1521",
  adbc.arrowttc.service_name = "orclpdb1",
  adbc.arrowttc.username     = "scott",
  adbc.arrowttc.password     = "tiger"
)
```

```rust
let mut database = driver.new_database_with_opts([
    (OptionDatabase::Other("adbc.arrowttc.server".into()),       OptionValue::String("localhost".into())),
    (OptionDatabase::Other("adbc.arrowttc.port".into()),         OptionValue::String("1521".into())),
    (OptionDatabase::Other("adbc.arrowttc.service_name".into()), OptionValue::String("orclpdb1".into())),
    (OptionDatabase::Other("adbc.arrowttc.username".into()),     OptionValue::String("scott".into())),
    (OptionDatabase::Other("adbc.arrowttc.password".into()),     OptionValue::String("tiger".into())),
])?;
```

</div>

Or the DB-API shortcut from the Python package:

<p class="code-lang-label">Python</p>

```python
import arrowttc_adbc
conn = arrowttc_adbc.connect(server="localhost", service_name="orclpdb1",
                             username="scott", password="tiger")
```

## Straight to pandas / Polars / DuckDB

Because results are already Arrow, there is no row-by-row marshalling. These handoffs
are Python-ecosystem specific:

<p class="code-lang-label">Python</p>

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

Oracle uses positional `:1`, `:2` bind placeholders. In the compiled clients you bind an
Arrow batch of parameters to the statement before executing:

<div class="code-tabs" markdown="1">

```python
with conn.cursor() as cur:
    cur.execute("SELECT * FROM orders WHERE o_orderkey = :1", parameters=[42])
    row = cur.fetch_arrow_table()
```

```cpp
// One-row, one-column (int64) parameter batch, built with nanoarrow.
ArrowSchema param_schema = {};
ArrowSchemaInitFromType(&param_schema, NANOARROW_TYPE_STRUCT);
ArrowSchemaAllocateChildren(&param_schema, 1);
ArrowSchemaInitFromType(param_schema.children[0], NANOARROW_TYPE_INT64);
ArrowSchemaSetName(param_schema.children[0], "0");

ArrowArray param_array = {};
ArrowArrayInitFromSchema(&param_array, &param_schema, nullptr);
ArrowArrayStartAppending(&param_array);
ArrowArrayAppendInt(param_array.children[0], 42);
ArrowArrayFinishElement(&param_array);
ArrowArrayFinishBuildingDefault(&param_array, nullptr);

CHECK(AdbcStatementSetSqlQuery(&statement, "SELECT * FROM orders WHERE o_orderkey = :1", &error));
CHECK(AdbcStatementBind(&statement, &param_array, &param_schema, &error));
CHECK(AdbcStatementExecuteQuery(&statement, &stream, &rows_affected, &error));
```

```csharp
using Apache.Arrow;

var schema = new Schema(new[] { new Field("0", Int64Type.Default, nullable: false) }, null);
var paramCol = new Int64Array.Builder().Append(42).Build();
var parameters = new RecordBatch(schema, new IArrowArray[] { paramCol }, 1);

using AdbcStatement statement = connection.CreateStatement();
statement.SqlQuery = "SELECT * FROM orders WHERE o_orderkey = :1";
statement.Bind(parameters, schema);
QueryResult result = statement.ExecuteQuery();
using var stream = result.Stream!;
```

```go
import (
	"github.com/apache/arrow-go/v18/arrow"
	"github.com/apache/arrow-go/v18/arrow/array"
	"github.com/apache/arrow-go/v18/arrow/memory"
)

schema := arrow.NewSchema([]arrow.Field{
	{Name: "0", Type: arrow.PrimitiveTypes.Int64},
}, nil)

bldr := array.NewRecordBuilder(memory.DefaultAllocator, schema)
defer bldr.Release()
bldr.Field(0).(*array.Int64Builder).Append(42)
rec := bldr.NewRecord()
defer rec.Release()

_ = stmt.SetSqlQuery("SELECT * FROM orders WHERE o_orderkey = :1")
if err := stmt.Bind(ctx, rec); err != nil {
	panic(err)
}
reader, _, err := stmt.ExecuteQuery(ctx)
if err != nil {
	panic(err)
}
defer reader.Release()
```

```java
import org.apache.arrow.vector.BigIntVector;
import org.apache.arrow.vector.VectorSchemaRoot;
import org.apache.arrow.vector.types.pojo.ArrowType;
import org.apache.arrow.vector.types.pojo.Field;
import org.apache.arrow.vector.types.pojo.FieldType;
import org.apache.arrow.vector.types.pojo.Schema;
import java.util.List;

Field field = new Field("0", FieldType.notNullable(new ArrowType.Int(64, true)), null);
try (VectorSchemaRoot root = VectorSchemaRoot.create(new Schema(List.of(field)), allocator)) {
  BigIntVector col = (BigIntVector) root.getVector("0");
  col.allocateNew(1);
  col.set(0, 42);
  root.setRowCount(1);

  try (AdbcStatement statement = connection.createStatement()) {
    statement.setSqlQuery("SELECT * FROM orders WHERE o_orderkey = :1");
    statement.bind(root);
    try (AdbcStatement.QueryResult result = statement.executeQuery()) {
      ArrowReader reader = result.getReader();
      while (reader.loadNextBatch()) {
        System.out.println(reader.getVectorSchemaRoot().contentToTSVString());
      }
    }
  }
}
```

```r
result <- read_adbc(
  con,
  "SELECT * FROM orders WHERE o_orderkey = :1",
  bind = data.frame(o_orderkey = 42L)
) |>
  as_arrow_table()
print(result)
```

```rust
use std::sync::Arc;
use arrow_array::{Int64Array, RecordBatch, RecordBatchReader};
use arrow_schema::{DataType, Field, Schema};

let schema = Arc::new(Schema::new(vec![Field::new("0", DataType::Int64, false)]));
let batch = RecordBatch::try_new(schema, vec![Arc::new(Int64Array::from(vec![42_i64]))])?;

statement.set_sql_query("SELECT * FROM orders WHERE o_orderkey = :1")?;
statement.bind(batch)?;
let reader = statement.execute()?;
println!("{}", reader.schema());
```

</div>

Array binding — bind a whole Arrow batch of parameters in one call (Python's
`executemany`):

<p class="code-lang-label">Python</p>

```python
with conn.cursor() as cur:
    cur.executemany(
        "INSERT INTO t(a, b) VALUES (:1, :2)",
        seq_of_parameters=[(1, "x"), (2, "y"), (3, "z")],
    )
    conn.commit()
```

## Bulk ingest an Arrow table

`adbc_ingest` pipes an Arrow table straight in via an array-bound `INSERT`:

<div class="code-tabs" markdown="1">

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

```csharp
using Apache.Arrow;

var schema = new Schema(new[]
{
    new Field("id",   Int32Type.Default,  nullable: false),
    new Field("name", StringType.Default, nullable: false),
}, null);
var idCol   = new Int32Array.Builder().AppendRange(new[] { 1, 2, 3 }).Build();
var nameCol = new StringArray.Builder().Append("a").Append("b").Append("c").Build();
var batch   = new RecordBatch(schema, new IArrowArray[] { idCol, nameCol }, 3);

using AdbcStatement statement = connection.CreateStatement();
statement.SetOption("adbc.ingest.target_table", "my_table");
statement.SetOption("adbc.ingest.mode", "adbc.ingest.mode.create");
statement.Bind(batch, schema);
statement.ExecuteUpdate();
connection.Commit();
```

```go
import (
	"github.com/apache/arrow-adbc/go/adbc"
	"github.com/apache/arrow-go/v18/arrow"
	"github.com/apache/arrow-go/v18/arrow/array"
	"github.com/apache/arrow-go/v18/arrow/memory"
)

schema := arrow.NewSchema([]arrow.Field{
	{Name: "id", Type: arrow.PrimitiveTypes.Int32},
	{Name: "name", Type: arrow.BinaryTypes.String},
}, nil)

bldr := array.NewRecordBuilder(memory.DefaultAllocator, schema)
defer bldr.Release()
bldr.Field(0).(*array.Int32Builder).AppendValues([]int32{1, 2, 3}, nil)
bldr.Field(1).(*array.StringBuilder).AppendValues([]string{"a", "b", "c"}, nil)
rec := bldr.NewRecord()
defer rec.Release()

_ = stmt.SetOption(adbc.OptionKeyIngestTargetTable, "my_table")
_ = stmt.SetOption(adbc.OptionKeyIngestMode, adbc.OptionValueIngestModeCreate)
if err := stmt.Bind(ctx, rec); err != nil {
	panic(err)
}
if _, err := stmt.ExecuteUpdate(ctx); err != nil {
	panic(err)
}
_ = conn.Commit(ctx)
```

```java
import org.apache.arrow.adbc.core.BulkIngestMode;
import org.apache.arrow.vector.IntVector;
import org.apache.arrow.vector.VarCharVector;
import org.apache.arrow.vector.VectorSchemaRoot;
import org.apache.arrow.vector.types.pojo.ArrowType;
import org.apache.arrow.vector.types.pojo.Field;
import org.apache.arrow.vector.types.pojo.FieldType;
import org.apache.arrow.vector.types.pojo.Schema;
import java.nio.charset.StandardCharsets;
import java.util.List;

Schema schema = new Schema(List.of(
    new Field("id",   FieldType.notNullable(new ArrowType.Int(32, true)), null),
    new Field("name", FieldType.notNullable(new ArrowType.Utf8()),        null)));

try (VectorSchemaRoot root = VectorSchemaRoot.create(schema, allocator)) {
  IntVector id = (IntVector) root.getVector("id");
  VarCharVector name = (VarCharVector) root.getVector("name");
  id.allocateNew(3);
  name.allocateNew(3);
  id.set(0, 1); id.set(1, 2); id.set(2, 3);
  name.setSafe(0, "a".getBytes(StandardCharsets.UTF_8));
  name.setSafe(1, "b".getBytes(StandardCharsets.UTF_8));
  name.setSafe(2, "c".getBytes(StandardCharsets.UTF_8));
  root.setRowCount(3);

  try (AdbcStatement statement =
           connection.bulkIngest("my_table", BulkIngestMode.CREATE)) {
    statement.bind(root);
    statement.executeUpdate();
  }
}
```

```r
tbl <- arrow::arrow_table(
  id   = arrow::Array$create(1:3, type = arrow::int32()),
  name = c("a", "b", "c")
)
tbl |> write_adbc(con, "my_table", mode = "create")
```

</div>

{: .note }
> **Rust & C++** have no one-call `adbc_ingest` helper: set the standard statement
> options `adbc.ingest.target_table` / `adbc.ingest.mode`, `Bind` the Arrow batch, then
> `execute_update` (Rust) / `AdbcStatementExecuteQuery` (C++).

Ingest into a specific schema or a `GLOBAL TEMPORARY` table via the statement
options `adbc.ingest.target_db_schema` / `adbc.ingest.temporary` (see
[`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}#statement--ingest-options)).

## Export a query to Parquet

<p class="code-lang-label">Python</p>

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

<p class="code-lang-label">Python</p>

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

<p class="code-lang-label">Python</p>

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

<p class="code-lang-label">Python</p>

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

<p class="code-lang-label">Python</p>

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

<p class="code-lang-label">Python</p>

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

## Loading by explicit path

Load-by-name (above) uses the ADBC manifest the installer registers. When the driver is
not installed system-wide, point the driver manager at the shared library instead — e.g.
in C# via `CAdbcDriverImporter.Load("libarrowttc_adbc_driver.so", "AdbcDriverInit")`, or by
setting `ARROWTTC`/`ADBC_DRIVER_PATH` for the name-based loaders. Everything after loading
is identical.

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}) — every connection option
- [`DATA_TYPES.md`]({{ '/drivers/arrowttc/data-types/' | relative_url }}) — the Arrow types you get back
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowttc/troubleshooting/' | relative_url }}) — when a snippet does not connect
