---
title: ArrowFEBE
layout: default
parent: Drivers
has_children: true
nav_order: 2
permalink: /drivers/arrowfebe/
---

# ArrowFEBE
{: .no_toc }

<p>
  <span class="badge badge-version">v0.3.8</span>
  <span class="badge badge-platform">Linux · Windows (x64)</span>
  <span class="badge badge-cloud">PostgreSQL-compatible services</span>
</p>

A pure-native ADBC driver for **PostgreSQL** that speaks the **Frontend/Backend
(FEBE) wire protocol v3** directly over TCP/TLS and delivers Apache Arrow
`RecordBatch`es end to end. **OpenSSL is the only runtime dependency beyond libc —
it does not use libpq.**
{: .fs-5 .fw-300 }

{: .note }
> **Reading these docs.** The badge above is the version this documentation
> describes. Options and features added recently are flagged inline with a
> _Since vX.Y.Z_ note; the [What's new]({{ '/drivers/arrowfebe/whats-new/' | relative_url }})
> page is the release-by-release view. See [Versioning]({{ '/versioning/' | relative_url }}).

## Highlights

- **Native FEBE v3** over TCP/TLS — PostgreSQL 12–17, no libpq.
- **NUMERIC without a string detour** — Postgres `NUMERIC` maps straight to Arrow
  `decimal128` / `decimal256`, a headline speed and fidelity win.
- **Full auth surface** — SCRAM-SHA-256 (with channel binding), MD5, cleartext,
  GSSAPI/Kerberos, SSPI, plus `gssencmode` GSS transport encryption.
- **Arrow straight from the wire** — results decode into Arrow column buffers with
  no intermediate row objects.

## Connect in seconds

Load the driver by name (`arrowfebe`) from any ADBC client:

<div class="code-tabs" markdown="1">

```python
import adbc_driver_manager.dbapi as dbapi
with dbapi.connect(driver="arrowfebe",
                   db_kwargs={"uri": "postgresql://dbuser:<password>@host:5432/appdb?sslmode=require"}) as conn:
    cur = conn.cursor()
    cur.execute("SELECT * FROM public.orders")
    table = cur.fetch_arrow_table()
```

```cpp
#include <arrow-adbc/adbc.h>
// (error checks elided for brevity)
AdbcError e = {}; AdbcDatabase db = {}; AdbcConnection conn = {}; AdbcStatement st = {};
AdbcDatabaseNew(&db, &e);
AdbcDatabaseSetOption(&db, "driver", "arrowfebe", &e);
AdbcDatabaseSetOption(&db, "uri", "postgresql://dbuser:<password>@host:5432/appdb?sslmode=require", &e);
AdbcDatabaseInit(&db, &e);
AdbcConnectionNew(&conn, &e); AdbcConnectionInit(&conn, &db, &e);
AdbcStatementNew(&conn, &st, &e);
AdbcStatementSetSqlQuery(&st, "SELECT * FROM public.orders", &e);
ArrowArrayStream stream = {}; int64_t n = -1;
AdbcStatementExecuteQuery(&st, &stream, &n, &e);   // consume `stream`
```

```csharp
using Apache.Arrow.Adbc;
using Apache.Arrow.Adbc.DriverManager;

using var driver = AdbcDriverManager.FindLoadDriver("arrowfebe", loadOptions: AdbcLoadFlags.Default);
using var database = driver.Open(new Dictionary<string, string> {
    ["uri"] = "postgresql://dbuser:<password>@host:5432/appdb?sslmode=require" });
using var connection = database.Connect(null);
using var statement = connection.CreateStatement();
statement.SqlQuery = "SELECT * FROM public.orders";
using var stream = statement.ExecuteQuery().Stream!;   // IArrowArrayStream
```

```go
var drv drivermgr.Driver
db, _ := drv.NewDatabase(map[string]string{
	"driver":          "arrowfebe",
	adbc.OptionKeyURI: "postgresql://dbuser:<password>@host:5432/appdb?sslmode=require",
})
conn, _ := db.Open(ctx)
stmt, _ := conn.NewStatement()
_ = stmt.SetSqlQuery("SELECT * FROM public.orders")
reader, _, _ := stmt.ExecuteQuery(ctx)   // array.RecordReader
defer reader.Release()
```

```java
Map<String, Object> params = new HashMap<>();
JniDriver.PARAM_DRIVER.set(params, "arrowfebe");
AdbcDriver.PARAM_URI.set(params, "postgresql://dbuser:<password>@host:5432/appdb?sslmode=require");
try (BufferAllocator a = new RootAllocator();
     AdbcDatabase db = new JniDriver(a).open(params);
     AdbcConnection conn = db.connect();
     AdbcStatement st = conn.createStatement()) {
  st.setSqlQuery("SELECT * FROM public.orders");
  try (AdbcStatement.QueryResult r = st.executeQuery()) {
    ArrowReader reader = r.getReader();
  }
}
```

```r
library(adbcdrivermanager)
db  <- adbc_database_init(adbc_driver("arrowfebe"),
                          uri = "postgresql://dbuser:<password>@host:5432/appdb?sslmode=require")
con <- adbc_connection_init(db)
table <- read_adbc(con, "SELECT * FROM public.orders") |> arrow::as_arrow_table()
```

```rust
use adbc_core::{Connection, Database, Driver, Statement, LOAD_FLAG_DEFAULT};
use adbc_core::options::{AdbcVersion, OptionDatabase, OptionValue};
use adbc_driver_manager::ManagedDriver;

let mut driver = ManagedDriver::load_from_name("arrowfebe", None, AdbcVersion::default(), LOAD_FLAG_DEFAULT, None)?;
let mut db = driver.new_database_with_opts([(OptionDatabase::Uri,
    OptionValue::String("postgresql://dbuser:<password>@host:5432/appdb?sslmode=require".into()))])?;
let mut conn = db.new_connection()?;
let mut stmt = conn.new_statement()?;
stmt.set_sql_query("SELECT * FROM public.orders")?;
let reader = stmt.execute()?;   // impl RecordBatchReader
```

</div>

New here? [Install ArrowFEBE]({{ '/install/' | relative_url }}) first, then see the
[Connection guide]({{ '/drivers/arrowfebe/connection/' | relative_url }}).

## All ArrowFEBE pages

<div class="card-grid" markdown="0">
  <a class="card" href="{{ '/drivers/arrowfebe/connection/' | relative_url }}"><span class="card-title">Connection</span><span class="card-body">Options, connection strings, URIs, <code>sslmode</code>, channel binding, Kerberos.</span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/authentication/' | relative_url }}"><span class="card-title">Authentication</span><span class="card-body">SCRAM, MD5, cleartext, GSSAPI/Kerberos, SSPI, <code>gssencmode</code>.</span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/data-types/' | relative_url }}"><span class="card-title">Data types</span><span class="card-body">PostgreSQL ↔ Arrow, NUMERIC → decimal, interval, PostGIS EWKB.</span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/compatibility/' | relative_url }}"><span class="card-title">Compatibility</span><span class="card-body">Supported versions, compatible forks, platforms, conformance.</span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/examples/' | relative_url }}"><span class="card-title">Examples</span><span class="card-body">Copy-paste recipes: query to Arrow/pandas/Polars/DuckDB, bind, ingest.</span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/environment-variables/' | relative_url }}"><span class="card-title">Environment variables</span><span class="card-body">Driver-specific <code>ARROWFEBE_*</code> knobs and build-time overrides.</span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/licensing/' | relative_url }}"><span class="card-title">Licensing</span><span class="card-body">Supplying and checking your Arpeio licence.</span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/troubleshooting/' | relative_url }}"><span class="card-title">Troubleshooting</span><span class="card-body">Connect, TLS, auth, licence and ingest failures and fixes.</span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/whats-new/' | relative_url }}"><span class="card-title">What's new</span><span class="card-body">Reader-facing release highlights.</span></a>
</div>
