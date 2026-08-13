---
title: ArrowTTC
layout: default
parent: Drivers
has_children: true
nav_order: 3
permalink: /drivers/arrowttc/
---

# ArrowTTC
{: .no_toc }

<p>
  <span class="badge badge-version">v0.2.8</span>
  <span class="badge badge-platform">Linux · Windows (x64)</span>
  <span class="badge badge-cloud">Oracle Cloud Autonomous Database</span>
</p>

A native ADBC driver for **Oracle**, written in C, returning Apache Arrow
`RecordBatch`es end to end. It speaks **TNS (Oracle Net) + TTC (Two-Task Common)**
directly over TCP — **no OCI, no Instant Client, no ODBC.** OpenSSL is the only
runtime dependency beyond libc. Targets Oracle 19c and **Oracle Cloud Autonomous
Database (ADB)**.
{: .fs-5 .fw-300 }

{: .note }
> **Reading these docs.** The badge above is the version this documentation
> describes. Options and features added recently are flagged inline with a
> _Since vX.Y.Z_ note; the [What's new]({{ '/drivers/arrowttc/whats-new/' | relative_url }})
> page is the release-by-release view. See [Versioning]({{ '/versioning/' | relative_url }}).

## Highlights

- **Native TNS + TTC** over TCP/TLS — no OCI, no Instant Client, no ODBC.
- **Oracle Cloud ready** — connects to **Autonomous Database (ADB)** over mutual
  TLS with the downloaded wallet, and authenticates with an **OCI IAM database
  token** or a password.
- **Modern auth** — O5LOGON password, proxy (CONNECT THROUGH), Kerberos, OCI IAM
  token, and **OAuth2 / Microsoft Entra ID** bearer tokens.
- **Native Network Encryption** — AES-256 session encryption + SHA-256/SHA-1
  integrity without TCPS, interoperating with Oracle 19c through 23ai.
- **Configurable `NUMBER` mapping** — lossless `utf8`, `float64`, or
  `decimal128(P,S)`.

## Connect in seconds

Load the driver by name (`arrowttc`) from any ADBC client:

<div class="code-tabs" markdown="1">

```python
import adbc_driver_manager.dbapi as dbapi
with dbapi.connect(driver="arrowttc",
                   db_kwargs={"uri": "oracle://scott:tiger@dbhost:1521/orclpdb1?ssl_mode=verify-full"}) as conn:
    cur = conn.cursor()
    cur.execute("SELECT * FROM hr.employees")
    table = cur.fetch_arrow_table()
```

```cpp
#include <arrow-adbc/adbc.h>
// (error checks elided for brevity)
AdbcError e = {}; AdbcDatabase db = {}; AdbcConnection conn = {}; AdbcStatement st = {};
AdbcDatabaseNew(&db, &e);
AdbcDatabaseSetOption(&db, "driver", "arrowttc", &e);
AdbcDatabaseSetOption(&db, "uri", "oracle://scott:tiger@dbhost:1521/orclpdb1?ssl_mode=verify-full", &e);
AdbcDatabaseInit(&db, &e);
AdbcConnectionNew(&conn, &e); AdbcConnectionInit(&conn, &db, &e);
AdbcStatementNew(&conn, &st, &e);
AdbcStatementSetSqlQuery(&st, "SELECT * FROM hr.employees", &e);
ArrowArrayStream stream = {}; int64_t n = -1;
AdbcStatementExecuteQuery(&st, &stream, &n, &e);   // consume `stream`
```

```csharp
using Apache.Arrow.Adbc;
using Apache.Arrow.Adbc.DriverManager;

using var driver = AdbcDriverManager.FindLoadDriver("arrowttc", loadOptions: AdbcLoadFlags.Default);
using var database = driver.Open(new Dictionary<string, string> {
    ["uri"] = "oracle://scott:tiger@dbhost:1521/orclpdb1?ssl_mode=verify-full" });
using var connection = database.Connect(null);
using var statement = connection.CreateStatement();
statement.SqlQuery = "SELECT * FROM hr.employees";
using var stream = statement.ExecuteQuery().Stream!;   // IArrowArrayStream
```

```go
var drv drivermgr.Driver
db, _ := drv.NewDatabase(map[string]string{
	"driver":          "arrowttc",
	adbc.OptionKeyURI: "oracle://scott:tiger@dbhost:1521/orclpdb1?ssl_mode=verify-full",
})
conn, _ := db.Open(ctx)
stmt, _ := conn.NewStatement()
_ = stmt.SetSqlQuery("SELECT * FROM hr.employees")
reader, _, _ := stmt.ExecuteQuery(ctx)   // array.RecordReader
defer reader.Release()
```

```java
Map<String, Object> params = new HashMap<>();
JniDriver.PARAM_DRIVER.set(params, "arrowttc");
AdbcDriver.PARAM_URI.set(params, "oracle://scott:tiger@dbhost:1521/orclpdb1?ssl_mode=verify-full");
try (BufferAllocator a = new RootAllocator();
     AdbcDatabase db = new JniDriver(a).open(params);
     AdbcConnection conn = db.connect();
     AdbcStatement st = conn.createStatement()) {
  st.setSqlQuery("SELECT * FROM hr.employees");
  try (AdbcStatement.QueryResult r = st.executeQuery()) {
    ArrowReader reader = r.getReader();
  }
}
```

```r
library(adbcdrivermanager)
db  <- adbc_database_init(adbc_driver("arrowttc"),
                          uri = "oracle://scott:tiger@dbhost:1521/orclpdb1?ssl_mode=verify-full")
con <- adbc_connection_init(db)
table <- read_adbc(con, "SELECT * FROM hr.employees") |> arrow::as_arrow_table()
```

```rust
use adbc_core::{Connection, Database, Driver, Statement, LOAD_FLAG_DEFAULT};
use adbc_core::options::{AdbcVersion, OptionDatabase, OptionValue};
use adbc_driver_manager::ManagedDriver;

let mut driver = ManagedDriver::load_from_name("arrowttc", None, AdbcVersion::default(), LOAD_FLAG_DEFAULT, None)?;
let mut db = driver.new_database_with_opts([(OptionDatabase::Uri,
    OptionValue::String("oracle://scott:tiger@dbhost:1521/orclpdb1?ssl_mode=verify-full".into()))])?;
let mut conn = db.new_connection()?;
let mut stmt = conn.new_statement()?;
stmt.set_sql_query("SELECT * FROM hr.employees")?;
let reader = stmt.execute()?;   // impl RecordBatchReader
```

</div>

New here? [Install ArrowTTC]({{ '/install/' | relative_url }}) first, then see the
[Connection guide]({{ '/drivers/arrowttc/connection/' | relative_url }}) (including
the [Oracle Cloud ADB]({{ '/drivers/arrowttc/connection/' | relative_url }}) section).

## All ArrowTTC pages

<div class="card-grid" markdown="0">
  <a class="card" href="{{ '/drivers/arrowttc/connection/' | relative_url }}"><span class="card-title">Connection</span><span class="card-body">Options, connection strings, Oracle URIs, EZConnect, Autonomous Database.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/authentication/' | relative_url }}"><span class="card-title">Authentication</span><span class="card-body">O5LOGON, proxy, Kerberos, OCI IAM token, OAuth2 / Entra ID, TLS.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/data-types/' | relative_url }}"><span class="card-title">Data types</span><span class="card-body">Oracle → Arrow mapping, the <code>NUMBER</code> mapping option, write path.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/compatibility/' | relative_url }}"><span class="card-title">Compatibility</span><span class="card-body">Supported Oracle versions, Oracle Cloud, auth methods, platforms.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/examples/' | relative_url }}"><span class="card-title">Examples</span><span class="card-body">Copy-paste recipes: query to Arrow/pandas/Polars/DuckDB, bind, ingest.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/environment-variables/' | relative_url }}"><span class="card-title">Environment variables</span><span class="card-body">Connection options and driver-specific <code>ARROWTTC_*</code> knobs.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/licensing/' | relative_url }}"><span class="card-title">Licensing</span><span class="card-body">Supplying and checking your Arpeio licence.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/troubleshooting/' | relative_url }}"><span class="card-title">Troubleshooting</span><span class="card-body">Connect, TLS/wallets, NNE, auth, licence and ingest failures.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/whats-new/' | relative_url }}"><span class="card-title">What's new</span><span class="card-body">Reader-facing release highlights.</span></a>
</div>
