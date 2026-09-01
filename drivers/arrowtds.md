---
title: ArrowTDS
layout: default
parent: Drivers
has_children: true
nav_order: 1
permalink: /drivers/arrowtds/
---

# ArrowTDS
{: .no_toc }

<p>
  <span class="badge badge-version">v0.5.25</span>
  <span class="badge badge-platform">Linux · Windows (x64)</span>
  <span class="badge badge-cloud">Azure SQL · Microsoft Fabric</span>
</p>

A pure-native ADBC driver for **Microsoft SQL Server** that speaks **MS-TDS 7.4**
directly over TCP/TLS and delivers Apache Arrow `RecordBatch`es end to end.
OpenSSL is the only runtime dependency beyond libc — no ODBC, no SQL Server client
libraries. It also covers **Azure SQL** and **Microsoft Fabric Warehouse**.
{: .fs-5 .fw-300 }

{: .note }
> **Reading these docs.** The badge above is the version this documentation
> describes. Options and features added recently are flagged inline with a
> _Since vX.Y.Z_ note; the [What's new]({{ '/drivers/arrowtds/whats-new/' | relative_url }})
> page is the release-by-release view. See [Versioning]({{ '/versioning/' | relative_url }}).

## Highlights

- **Native TDS 7.4** over TCP/TLS — SQL Server 2012–2025, Azure SQL, Fabric.
- **Every auth mode** — SQL logins, trusted/integrated (Windows SSPI; POSIX
  Kerberos via GSSAPI/SPNEGO), and Microsoft Entra ID.
- **Rich type mapping** — including SQL Server 2025 `VECTOR(n)` → Arrow
  `fixed_size_list`, and `geometry`/`geography` → GeoArrow WKB.
- **Arrow straight from the wire** — results decode into Arrow column buffers with
  no intermediate row objects.

## Connect in seconds

Load the driver by name (`arrowtds`) from any ADBC client:

<div class="code-tabs" markdown="1">

```python
import adbc_driver_manager.dbapi as dbapi
with dbapi.connect(driver="arrowtds",
                   db_kwargs={"uri": "sqlserver://dbuser:<password>@host:1433/?database=appdb&encrypt=true"}) as conn:
    cur = conn.cursor()
    cur.execute("SELECT * FROM dbo.orders")
    table = cur.fetch_arrow_table()
```

```cpp
#include <arrow-adbc/adbc.h>
// (error checks elided for brevity)
AdbcError e = {}; AdbcDatabase db = {}; AdbcConnection conn = {}; AdbcStatement st = {};
AdbcDatabaseNew(&db, &e);
AdbcDatabaseSetOption(&db, "driver", "arrowtds", &e);
AdbcDatabaseSetOption(&db, "uri", "sqlserver://dbuser:<password>@host:1433/?database=appdb&encrypt=true", &e);
AdbcDatabaseInit(&db, &e);
AdbcConnectionNew(&conn, &e); AdbcConnectionInit(&conn, &db, &e);
AdbcStatementNew(&conn, &st, &e);
AdbcStatementSetSqlQuery(&st, "SELECT * FROM dbo.orders", &e);
ArrowArrayStream stream = {}; int64_t n = -1;
AdbcStatementExecuteQuery(&st, &stream, &n, &e);   // consume `stream`
```

```csharp
using Apache.Arrow.Adbc;
using Apache.Arrow.Adbc.DriverManager;

using var driver = AdbcDriverManager.FindLoadDriver("arrowtds", loadOptions: AdbcLoadFlags.Default);
using var database = driver.Open(new Dictionary<string, string> {
    ["uri"] = "sqlserver://dbuser:<password>@host:1433/?database=appdb&encrypt=true" });
using var connection = database.Connect(null);
using var statement = connection.CreateStatement();
statement.SqlQuery = "SELECT * FROM dbo.orders";
using var stream = statement.ExecuteQuery().Stream!;   // IArrowArrayStream
```

```go
var drv drivermgr.Driver
db, _ := drv.NewDatabase(map[string]string{
	"driver":          "arrowtds",
	adbc.OptionKeyURI: "sqlserver://dbuser:<password>@host:1433/?database=appdb&encrypt=true",
})
conn, _ := db.Open(ctx)
stmt, _ := conn.NewStatement()
_ = stmt.SetSqlQuery("SELECT * FROM dbo.orders")
reader, _, _ := stmt.ExecuteQuery(ctx)   // array.RecordReader
defer reader.Release()
```

```java
Map<String, Object> params = new HashMap<>();
JniDriver.PARAM_DRIVER.set(params, "arrowtds");
AdbcDriver.PARAM_URI.set(params, "sqlserver://dbuser:<password>@host:1433/?database=appdb&encrypt=true");
try (BufferAllocator a = new RootAllocator();
     AdbcDatabase db = new JniDriver(a).open(params);
     AdbcConnection conn = db.connect();
     AdbcStatement st = conn.createStatement()) {
  st.setSqlQuery("SELECT * FROM dbo.orders");
  try (AdbcStatement.QueryResult r = st.executeQuery()) {
    ArrowReader reader = r.getReader();
  }
}
```

```r
library(adbcdrivermanager)
db  <- adbc_database_init(adbc_driver("arrowtds"),
                          uri = "sqlserver://dbuser:<password>@host:1433/?database=appdb&encrypt=true")
con <- adbc_connection_init(db)
table <- read_adbc(con, "SELECT * FROM dbo.orders") |> arrow::as_arrow_table()
```

```rust
use adbc_core::{Connection, Database, Driver, Statement, LOAD_FLAG_DEFAULT};
use adbc_core::options::{AdbcVersion, OptionDatabase, OptionValue};
use adbc_driver_manager::ManagedDriver;

let mut driver = ManagedDriver::load_from_name("arrowtds", None, AdbcVersion::default(), LOAD_FLAG_DEFAULT, None)?;
let mut db = driver.new_database_with_opts([(OptionDatabase::Uri,
    OptionValue::String("sqlserver://dbuser:<password>@host:1433/?database=appdb&encrypt=true".into()))])?;
let mut conn = db.new_connection()?;
let mut stmt = conn.new_statement()?;
stmt.set_sql_query("SELECT * FROM dbo.orders")?;
let reader = stmt.execute()?;   // impl RecordBatchReader
```

</div>

New here? [Install ArrowTDS]({{ '/install/' | relative_url }}) first, then see the
[Connection guide]({{ '/drivers/arrowtds/connection/' | relative_url }}).

## All ArrowTDS pages

<div class="card-grid" markdown="0">
  <a class="card" href="{{ '/drivers/arrowtds/connection/' | relative_url }}"><span class="card-title">Connection</span><span class="card-body">Options, connection strings, URIs, named instances, Azure &amp; Entra.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/authentication/' | relative_url }}"><span class="card-title">Authentication</span><span class="card-body">SQL, trusted/integrated (SSPI, Kerberos), Entra ID.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/data-types/' | relative_url }}"><span class="card-title">Data types</span><span class="card-body">SQL Server ↔ Arrow mapping, temporal precision, geospatial, VECTOR.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/compatibility/' | relative_url }}"><span class="card-title">Compatibility</span><span class="card-body">Supported versions, editions, Azure/Fabric, platforms, conformance.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/examples/' | relative_url }}"><span class="card-title">Examples</span><span class="card-body">Copy-paste recipes: query to Arrow/pandas/Polars/DuckDB, bind, ingest.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/environment-variables/' | relative_url }}"><span class="card-title">Environment variables</span><span class="card-body">Driver-specific <code>ARROWTDS_*</code> knobs and build-time overrides.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/licensing/' | relative_url }}"><span class="card-title">Licensing</span><span class="card-body">Supplying and checking your Arpeio licence.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/troubleshooting/' | relative_url }}"><span class="card-title">Troubleshooting</span><span class="card-body">Connect, TLS, auth, licence and ingest failures and fixes.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/whats-new/' | relative_url }}"><span class="card-title">What's new</span><span class="card-body">Reader-facing release highlights.</span></a>
</div>
