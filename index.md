---
title: Home
layout: default
nav_order: 1
permalink: /
---

<div class="hero" markdown="0">
  <p class="hero-eyebrow">Arpeio ADBC drivers</p>
  <h1>Apache Arrow, straight from your database.</h1>
  <p class="hero-lede">
    A family of pure-native, high-performance
    <a href="https://arrow.apache.org/adbc/">ADBC</a> drivers for SQL Server,
    PostgreSQL and Oracle. Each one speaks the database's wire protocol directly
    and returns Apache Arrow <code>RecordBatch</code>es end to end — no ODBC, no
    vendor client libraries. Install with a single command, then load it by name
    from any ADBC client.
  </p>
</div>

[Install a driver]({{ '/install/' | relative_url }}){: .btn .btn-primary .mr-2 }
[Browse the drivers]({{ '/drivers/' | relative_url }}){: .btn }

## Which driver do I need?

| Your database | Driver | Load name | Also covers | Status |
|---|---|---|---|---|
| Microsoft SQL Server | [**ArrowTDS**]({{ '/drivers/arrowtds/' | relative_url }}) | `arrowtds` | Azure SQL, Microsoft Fabric | ✅ Published |
| PostgreSQL | [**ArrowFEBE**]({{ '/drivers/arrowfebe/' | relative_url }}) | `arrowfebe` | PostgreSQL-compatible services | ✅ Published |
| Oracle | [**ArrowTTC**]({{ '/drivers/arrowttc/' | relative_url }}) | `arrowttc` | Oracle Cloud Autonomous Database | ✅ Published |
| IBM Db2 | ArrowDRDA | `arrowdrda` | — | 🚧 Coming soon |

The driver **binaries** are published as public GitHub Releases and are free to
download. They are **licence-gated**: a driver needs a valid Arpeio licence at
runtime — there is no trial build. Contact
[sales@arpe.io](mailto:sales@arpe.io) for a licence.

## The drivers

<div class="card-grid" markdown="0">
  <a class="card" href="{{ '/drivers/arrowtds/' | relative_url }}"><span class="card-db">Microsoft SQL Server</span><span class="card-title">ArrowTDS</span><span class="card-body">Native MS-TDS 7.4 over TCP/TLS. Azure SQL, Microsoft Fabric, SQL &amp; Entra ID auth, geospatial &amp; VECTOR types.</span><span class="card-meta"><span class="badge badge-version">v0.5.25</span> <span class="badge badge-platform">Linux · Windows</span> <span class="badge badge-cloud">Azure SQL · Fabric</span></span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/' | relative_url }}"><span class="card-db">PostgreSQL</span><span class="card-title">ArrowFEBE</span><span class="card-body">Native Frontend/Backend v3 over TCP/TLS — no libpq. NUMERIC → decimal with no string detour, SCRAM &amp; Kerberos auth.</span><span class="card-meta"><span class="badge badge-version">v0.3.8</span> <span class="badge badge-platform">Linux · Windows</span> <span class="badge badge-cloud">PG-compatible</span></span></a>
  <a class="card" href="{{ '/drivers/arrowttc/' | relative_url }}"><span class="card-db">Oracle</span><span class="card-title">ArrowTTC</span><span class="card-body">Native TNS + TTC over TCP/TLS — no OCI, no Instant Client. Oracle Cloud Autonomous Database, OCI IAM &amp; Entra ID tokens.</span><span class="card-meta"><span class="badge badge-version">v0.2.8</span> <span class="badge badge-platform">Linux · Windows</span> <span class="badge badge-cloud">Oracle Cloud ADB</span></span></a>
</div>

## Install in one line

**Linux / macOS:**

```sh
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh \
  | sh -s -- arrowtds --license /path/to/your.lic
```

**Windows (PowerShell):**

```powershell
& ([scriptblock]::Create((irm https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.ps1))) `
  arrowtds -License C:\path\to\your.lic
```

Then load it by name from any ADBC client:

<div class="code-tabs" markdown="1">

```python
import adbc_driver_manager.dbapi as dbapi
with dbapi.connect(driver="arrowtds",
                   db_kwargs={"uri": "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true"}) as conn:
    cur = conn.cursor()
    cur.execute("SELECT * FROM dbo.orders")
    table = cur.fetch_arrow_table()   # ready as Arrow
```

```cpp
#include <arrow-adbc/adbc.h>
// (error checks elided for brevity)
AdbcError e = {}; AdbcDatabase db = {}; AdbcConnection conn = {}; AdbcStatement st = {};
AdbcDatabaseNew(&db, &e);
AdbcDatabaseSetOption(&db, "driver", "arrowtds", &e);
AdbcDatabaseSetOption(&db, "uri", "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true", &e);
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
    ["uri"] = "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true" });
using var connection = database.Connect(null);
using var statement = connection.CreateStatement();
statement.SqlQuery = "SELECT * FROM dbo.orders";
using var stream = statement.ExecuteQuery().Stream!;   // IArrowArrayStream
```

```go
var drv drivermgr.Driver
db, _ := drv.NewDatabase(map[string]string{
	"driver":          "arrowtds",
	adbc.OptionKeyURI: "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true",
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
AdbcDriver.PARAM_URI.set(params, "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true");
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
                          uri = "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true")
con <- adbc_connection_init(db)
table <- read_adbc(con, "SELECT * FROM dbo.orders") |> arrow::as_arrow_table()
```

```rust
use adbc_core::{Connection, Database, Driver, Statement, LOAD_FLAG_DEFAULT};
use adbc_core::options::{AdbcVersion, OptionDatabase, OptionValue};
use adbc_driver_manager::ManagedDriver;

let mut driver = ManagedDriver::load_from_name("arrowtds", None, AdbcVersion::default(), LOAD_FLAG_DEFAULT, None)?;
let mut db = driver.new_database_with_opts([(OptionDatabase::Uri,
    OptionValue::String("sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true".into()))])?;
let mut conn = db.new_connection()?;
let mut stmt = conn.new_statement()?;
stmt.set_sql_query("SELECT * FROM dbo.orders")?;
let reader = stmt.execute()?;   // impl RecordBatchReader
```

</div>

Swap `arrowtds` for `arrowfebe` or `arrowttc`. Full options, licence handling and
troubleshooting are on the [Installation]({{ '/install/' | relative_url }}) page.

{: .note }
> **Platforms:** the drivers ship for **Linux (x64)** and **Windows (x64)**. macOS
> binaries are staged but not yet published.

## Find your way around

<div class="card-grid" markdown="0">
  <a class="card" href="{{ '/install/' | relative_url }}"><span class="card-title">Installation</span><span class="card-body">The one-line installer, options, licence handling, download-only, and troubleshooting.</span></a>
  <a class="card" href="{{ '/environment-variables/' | relative_url }}"><span class="card-title">Environment variables</span><span class="card-body">The shared <code>ARPEIO_ADBC_*</code> knobs that mean the same thing in every driver.</span></a>
  <a class="card" href="{{ '/versioning/' | relative_url }}"><span class="card-title">Versioning</span><span class="card-body">How the drivers are versioned, current versions, and how docs track releases.</span></a>
</div>
