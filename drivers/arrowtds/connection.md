---
title: Connection
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 1
permalink: /drivers/arrowtds/connection/
---

# ArrowTDS — Connection guide

Everything you can put in front of ArrowTDS to point it at a SQL Server, in one
place: the three ways to supply connection details, every option the driver
accepts, and the SQL Server-specific rules (named instances, Azure SQL / Fabric,
Entra ID).

For the wire-level detail of *how* each authentication method works (SSPI,
Kerberos/GSSAPI, FEDAUTH), see [`AUTHENTICATION.md`]({{ '/drivers/arrowtds/authentication/' | relative_url }}).
For read/ingest throughput knobs, see `SQLSERVER_TUNING.md`. For the licence, see
[`LICENSING.md`]({{ '/drivers/arrowtds/licensing/' | relative_url }}).

---

## Three ways to connect

Every driver in the Arpeio family accepts the same three connection forms. They
can be mixed; a **discrete option always wins** over the same field taken from a
`connection_string` or a `uri`, regardless of the order they are set.

| Form | Option key | Grammar | Best for |
|---|---|---|---|
| **Discrete options** | `adbc.arrowtds.<field>` | one option per field | programmatic clients, secrets kept out of a single string |
| **Connection string** | `adbc.arrowtds.connection_string` | ADO.NET `Key=Value;…` | pasting an existing SQL Server / SqlClient string |
| **Connection URI** | `uri` | `sqlserver://…` URL | portable tooling, copy-paste from `go-mssqldb`/ADBC tools |

<p class="code-lang-label">Python</p>

```python
import adbc_driver_manager.dbapi as dbapi

# Discrete options
conn = dbapi.connect(driver="arrowtds", db_kwargs={
    "adbc.arrowtds.server":            "localhost",
    "adbc.arrowtds.database":          "appdb",
    "adbc.arrowtds.username":          "dbuser",
    "adbc.arrowtds.password":          "<password>",
    "adbc.arrowtds.encrypt":           "true",
    "adbc.arrowtds.trust_server_cert": "true",
})

# Connection string
conn = dbapi.connect(driver="arrowtds", db_kwargs={
    "adbc.arrowtds.connection_string":
        "Server=localhost;Database=appdb;User ID=dbuser;Password=<password>;"
        "Encrypt=true;TrustServerCertificate=true",
})

# Connection URI
conn = dbapi.connect(driver="arrowtds", db_kwargs={
    "uri": "sqlserver://dbuser:<password>@localhost:1433/"
           "?database=appdb&encrypt=true&TrustServerCertificate=true",
})
```

**Precedence** is enforced by a per-field bitmask shared between the parsed
result and the database handle: a parsed field from `connection_string`/`uri` is
applied only if no discrete `adbc.arrowtds.*` option already claimed it. Secrets
held transiently on the stack during parsing are scrubbed on every return path.

---

## Option reference

All options are **string-typed** and live under the `adbc.arrowtds.*` namespace
(a few also accept a short bare alias, noted below). Set them as ADBC database
options before the connection is opened. Statement/ingest options are set on the
statement — see [Statement & ingest options](#statement--ingest-options).

### Target — where to connect

| Option | Alias | Default | Meaning |
|---|---|---|---|
| `adbc.arrowtds.server` | `hostname` | — | Host, `host,port` / `host:port` / `[::1]:port`, `.`/`(local)` (→ `localhost`), or a **named instance** `host\INSTANCE`. See [Named instances](#named-instances). |
| `adbc.arrowtds.port` | `port` | 1433 | TCP port, `1..65535`. Ignored for a named instance unless given explicitly. |
| `adbc.arrowtds.database` | `database` | server default | Initial database. |
| `adbc.arrowtds.app_name` | `app_name` | driver default | Program name reported in LOGIN7 (`sys.dm_exec_sessions.program_name`). |

### Authentication

Full method-by-method detail is in [`AUTHENTICATION.md`]({{ '/drivers/arrowtds/authentication/' | relative_url }}); the
cloud/Entra options are described under
[Azure SQL, Microsoft Fabric & Entra ID](#azure-sql-microsoft-fabric--entra-id).

| Option | Values | Meaning |
|---|---|---|
| `adbc.arrowtds.username` / `.password` | string | SQL-authentication credentials (aliases `username` / `password`). |
| `adbc.arrowtds.trusted` | `true`/`sspi`/`false` | Integrated auth: Windows SSPI or POSIX Kerberos, no username/password. Alias `trusted`. |
| `adbc.arrowtds.auth_type` | `sql`/`SqlPassword`, `integrated`/`sspi`/`trusted`, `ActiveDirectoryDefault`/`default` | Selects the auth mode explicitly. Other `ActiveDirectory*` values are rejected with a pointer to the supported options. |
| `adbc.arrowtds.krb5.keytab` | path | Kerberos keytab for integrated auth. |
| `adbc.arrowtds.krb5.ccache` | path | Kerberos credential cache. |
| `adbc.arrowtds.krb5.principal` | name | Kerberos client principal. |
| `adbc.arrowtds.krb5.password` | string | Password to obtain a TGT (password-based Kerberos). |
| `adbc.arrowtds.krb5.spn` | SPN | Override the derived service principal name (`MSSQLSvc/host:port`). |

### Encryption / TLS

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowtds.encrypt` | login-only | `true`/`false` (also `1`/`0`, `yes`/`no`). `true` encrypts the whole session (TLS 1.2+, SNI, cert + hostname verification). Azure SQL requires `true`. |
| `adbc.arrowtds.trust_server_cert` | `false` | `true` skips server-certificate/hostname verification (self-signed dev servers). Leave `false` in production. |
| `adbc.arrowtds.allow_cleartext_login` | `false` | Permit the credentials-in-cleartext login exchange when encryption is off. |

With `encrypt=true` and `trust_server_cert=false` the server certificate and
hostname are verified by default; an IP-literal host is checked against the
certificate's `iPAddress` SAN.

### Cloud / Entra ID (Azure SQL & Microsoft Fabric)

| Option | Meaning |
|---|---|
| `adbc.arrowtds.access_token` | Caller-supplied Entra ID bearer token (passthrough); carried in LOGIN7 as a FEDAUTH feature extension. |
| `adbc.arrowtds.tenant_id` / `.client_id` / `.client_secret` | Service principal; the driver acquires a token via OAuth2 client credentials at connect. |
| `adbc.arrowtds.managed_identity` | `true`/`system` (system-assigned) or a user-assigned identity's client id; the driver acquires the token from the Azure host, no secret. |

See [Azure SQL, Microsoft Fabric & Entra ID](#azure-sql-microsoft-fabric--entra-id).

### Timeouts & performance

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowtds.login_timeout` | 30 | TCP connect budget in seconds (`0` = no limit). When a host resolves to several addresses they **share** this budget, so one dead address cannot stall the connect. |
| `adbc.arrowtds.query_timeout` | 0 | Per-query timeout in seconds (`0` = no limit). |
| `adbc.arrowtds.buffer_size` | 100000 | Rows per streamed Arrow batch (per connection). Capped at 10,000,000. The driver auto-flushes early if a wide `utf8`/`binary` column would cross Arrow's 2 GiB offset limit, so large values are safe. |
| `adbc.arrowtds.packet_size` | 0 (→ 8192) | TDS network packet size. `0` = the driver default (8192); otherwise `512..32767` (MS-TDS spec maximum). |

More tuning guidance in `SQLSERVER_TUNING.md`.

### Type rendering (read path)

| Option | Values | Meaning |
|---|---|---|
| `adbc.arrowtds.uuid_casing` | `lower` (default) / `upper` | Hex case of `uniqueidentifier`/GUID values rendered to Arrow `utf8`. `lower` matches RFC 4122 and the `mssql` connection type. |
| `adbc.arrowtds.geospatial` | `geoarrow` (default) / `wkb` / `varbinary` | How `geometry`/`geography` (CLR UDT) columns are read. See [`DATA_TYPES.md`]({{ '/drivers/arrowtds/data-types/' | relative_url }}#geospatial-types). |

### Ingest SRID (write path)

> Since ArrowTDS v0.5.25.

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowtds.ingest.srid` | unset (`-1`) | SRID stamped into every `geometry`/`geography` value written on the Arrow WKB → UDT ingest path. Must be an integer `>= -1`. When unset, the driver uses the column's GeoArrow `crs` metadata if present, otherwise the kind default (`geometry` 0, `geography` 4326). See [`DATA_TYPES.md`]({{ '/drivers/arrowtds/data-types/' | relative_url }}#write-path-arrow--sql-server-ingest--bind). |

### Licence

| Option | Meaning |
|---|---|
| `arpeio.adbc.license` | Licence blob, inline. |
| `arpeio.adbc.license_file` | Path to a `.lic` file. |
| `arpeio.adbc.license.status` | **Read-only** (`GetOption`); reports `<state>;code=<ARROW_LIC_*>;tier=<tier>;expires=<epoch>`. |

The driver also reads the shared `ARPEIO_ADBC_LICENCE[_FILE]` environment
variables and an `arpeio_adbc.lic` file next to the library. Full resolution
order in [`LICENSING.md`]({{ '/drivers/arrowtds/licensing/' | relative_url }}).

### Standard ADBC connection options

These use the ADBC-standard keys (no `arrowtds` namespace):

| Option | Values | Meaning |
|---|---|---|
| `adbc.connection.autocommit` | `true`/`false` | Autocommit mode. Set `true` for the single-connection `TRUNCATE`+`INSERT BULK` ingest pattern. |
| `adbc.connection.transaction.isolation_level` | `read_uncommitted`/`read_committed`/`repeatable_read`/`snapshot`/`serializable`/`default` | Maps to `SET TRANSACTION ISOLATION LEVEL`; survives an internal reconnect. `linearizable` → `NOT_IMPLEMENTED` (no SQL Server equivalent). |
| `adbc.connection.readonly` | `false` | Accepted as a no-op; `true` → `NOT_IMPLEMENTED` (SQL Server has no session read-only mode). |

### Statement & ingest options

Set on the statement handle, not the database:

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowtds.batch_size` | 100000 | Rows per Arrow batch for this statement (`1..max_batch_size`). |
| `adbc.arrowtds.max_batch_size` | — | Upper bound for `batch_size`. |
| `adbc.arrowtds.memory_budget_mb` | — | Decode memory budget, `16..8192` MB. |
| `adbc.ingest.mode` | `create` | `create` / `append` / `replace` / `create_append`. |
| `adbc.ingest.target_catalog` | — | Target database for ingest. |
| `adbc.ingest.target_db_schema` | — | Target schema for ingest. |
| `adbc.ingest.temporary` | `false` | Ingest into a temp table. |

### Accepted no-ops (back-compat)

Recognised and ignored so strings copied from ODBC-era tooling still load:
`adbc.arrowtds.mars`, `adbc.arrowtds.application_intent`,
`adbc.arrowtds.application_intent_readonly`, `adbc.arrowtds.compression`,
`adbc.arrowtds.enable_compression`, `adbc.arrowtds.prefetch`, and the removed BCP
knobs (`arrowtds.use_bcp`, `arrowtds.bcp_*`, which log a deprecation warning). The
bare `mars` / `application_intent` / `compression` aliases are accepted too.

---

## Connection string (ADO.NET form)

`adbc.arrowtds.connection_string` accepts the ADO.NET `Key=Value;…` grammar:
case-insensitive keywords, quoted values, doubled-quote escapes. Keywords map
onto the options above.

| Keyword(s) | Option |
|---|---|
| `Server`, `Data Source`, `Address`, `Addr`, `Network Address` | `server` |
| `Database`, `Initial Catalog` | `database` |
| `User ID`, `UID`, `User` | `username` |
| `Password`, `PWD` | `password` |
| `Application Name`, `App` | `app_name` |
| `Encrypt` | `encrypt` |
| `TrustServerCertificate` | `trust_server_cert` |
| `Connection Timeout`, `Connect Timeout`, `Timeout` | `login_timeout` |
| `Packet Size` | `packet_size` |
| `Integrated Security` (`true`/`sspi`/`yes`) | `trusted` |
| `Authentication` | auth mode (`SqlPassword`, `ActiveDirectory*` …) |

```
Server=localhost\SQL2022;Database=appdb;User ID=dbuser;Password=<password>;Encrypt=true;Packet Size=8192
```

---

## Connection URI

{: .since }
> Since ArrowTDS v0.5.19.

The standard ADBC `uri` option takes the portable SQL Server URL used by the
official drivers (Microsoft `go-mssqldb`, the ADBC Foundry driver), so a string
copied from other SQL Server ADBC tooling works unchanged.

```
<scheme>://[user[:password]]@host[:port][/INSTANCE][?key=value&…]
```

- **Schemes** (case-insensitive, equivalent): `sqlserver://`, `mssql://`, and the
  branded `arrowtds://`. The scheme only selects the URL grammar — it never picks
  the driver, so `sqlserver://`/`mssql://` never collide with another SQL Server
  ADBC driver installed alongside this one.
- The **path segment is the SQL Server *instance* name** (folded into
  `host\INSTANCE`), **not** the database — the database is the `database` query
  parameter. This matches `go-mssqldb`; it is the one surprising part of the
  grammar.
- `userinfo` and query values are **percent-decoded**; query values treat `+` as
  a space (`connection+timeout` ≡ `connection timeout`). IPv6 hosts use the
  bracketed form: `sqlserver://[::1]:1433/?database=appdb`.

**Query parameters** (unknown or repeated parameters are rejected):

| Parameter(s) | Option |
|---|---|
| `database`, `initial catalog` | `database` |
| `user id`, `user`, `uid` | `username` |
| `password`, `pwd` | `password` |
| `encrypt` | `encrypt` |
| `TrustServerCertificate` | `trust_server_cert` |
| `connection timeout`, `connect timeout`, `dial timeout` | `login_timeout` |
| `packet size` | `packet_size` |
| `app name`, `application name` | `app_name` |
| `ApplicationIntent` | accepted, no-op |
| `hostNameInCertificate`, `fedauth=ActiveDirectory*` | recognised, **not yet implemented** |

For Entra ID today, pass a bearer token via the `adbc.arrowtds.access_token`
option rather than a `fedauth` URL parameter — see
[Azure SQL, Microsoft Fabric & Entra ID](#azure-sql-microsoft-fabric--entra-id).

---

## Named instances

{: .since }
> Since ArrowTDS v0.5.14.

A SQL Server named instance normally listens on a dynamic TCP port, not 1433.
Given `Server=host\INSTANCE`, the driver asks the **SQL Server Browser** on **UDP
port 1434** which port the instance uses, then connects there — the same
handshake SqlClient performs. Both the query path and the `INSERT BULK` path use
it.

```
Server=localhost\DATAQ1          # port discovered via the Browser
Server=.\SQLEXPRESS              # "." and "(local)" mean this machine
Server=localhost\DATAQ1,54312    # explicit port: no Browser lookup at all
```

- **UDP 1434 must be reachable** and the Browser service must be running. If it
  does not answer, the connection fails after ~3 s with a message naming the
  instance and the workaround. The driver never falls back to 1433 on its own.
- **An explicit port bypasses the lookup** (including an explicit `,1433`) — the
  escape hatch when the Browser is firewalled off.
- `(localdb)\...` is rejected: LocalDB is reached over a named pipe, not TCP.

For integrated auth the Kerberos SPN is derived *after* resolution, as
`MSSQLSvc/host:<resolved-port>`. See [`AUTHENTICATION.md`]({{ '/drivers/arrowtds/authentication/' | relative_url }}).

---

## Azure SQL, Microsoft Fabric & Entra ID

{: .since }
> Entra ID authentication since ArrowTDS v0.5.20.

SQL-authentication connections to `*.database.windows.net` and
`*.datawarehouse.fabric.microsoft.com` work out of the box — TLS 1.2+, SNI, and
hostname verification are on the default path (use `encrypt=true`). Both the
**Proxy** and **Redirect** gateway policies are supported: on Redirect the driver
follows the gateway's `ENVCHANGE ROUTING` token and reconnects to the database
node transparently (needs outbound TCP to ports **11000–11999**).

Entra ID (Azure AD) authentication has four modes, all feeding the same LOGIN7
FEDAUTH path over verified TLS. Federated auth always uses full TLS and cannot be
combined with a username/password.

### 1. Access-token passthrough

Supply a bearer token you acquired out-of-band:

<p class="code-lang-label">Python</p>

```python
import subprocess, adbc_driver_manager.dbapi as dbapi

token = subprocess.check_output(
    ["az", "account", "get-access-token",
     "--resource", "https://database.windows.net/",
     "--query", "accessToken", "-o", "tsv"],
).decode().strip()

conn = dbapi.connect(driver="arrowtds", db_kwargs={
    "adbc.arrowtds.server":       "myserver.database.windows.net",
    "adbc.arrowtds.database":     "appdb",
    "adbc.arrowtds.encrypt":      "true",
    "adbc.arrowtds.access_token": token,
})
```

### 2. Service principal (client credentials)

The driver acquires the token itself; omit `access_token`:

<p class="code-lang-label">Python</p>

```python
db_kwargs={
    "adbc.arrowtds.server":        "myserver.database.windows.net",
    "adbc.arrowtds.database":      "appdb",
    "adbc.arrowtds.encrypt":       "true",
    "adbc.arrowtds.tenant_id":     "<tenant-guid-or-domain>",
    "adbc.arrowtds.client_id":     "<app-client-id>",
    "adbc.arrowtds.client_secret": "<secret>",
}
```

The service principal must be a database user
(`CREATE USER [<app>] FROM EXTERNAL PROVIDER`). The token is acquired once per
connection and refreshes on reconnect.

### 3. Managed identity

On an Azure host (VM/VMSS via IMDS, or App Service / Functions / Container Apps),
no secret is needed:

<p class="code-lang-label">Python</p>

```python
db_kwargs={
    "adbc.arrowtds.server":           "myserver.database.windows.net",
    "adbc.arrowtds.database":         "appdb",
    "adbc.arrowtds.encrypt":          "true",
    "adbc.arrowtds.managed_identity": "true",   # or a user-assigned client id
}
```

### 4. Default credential chain

`auth_type=ActiveDirectoryDefault` tries, in order: **environment service
principal** (`AZURE_TENANT_ID` + `AZURE_CLIENT_ID` + `AZURE_CLIENT_SECRET`) →
**managed identity** (short probe) → **Azure CLI** (`az account get-access-token`).
Handy for code that runs unchanged on a dev machine, in CI, and on an Azure host:

<p class="code-lang-label">Python</p>

```python
db_kwargs={
    "adbc.arrowtds.server":    "myserver.database.windows.net",
    "adbc.arrowtds.database":  "appdb",
    "adbc.arrowtds.encrypt":   "true",
    "adbc.arrowtds.auth_type": "ActiveDirectoryDefault",
}
```

This is a practical subset of Azure's `DefaultAzureCredential`;
workload-identity-file, Visual Studio, Azure PowerShell, and `azd` sources are
not covered.

---

## See also

- [`AUTHENTICATION.md`]({{ '/drivers/arrowtds/authentication/' | relative_url }}) — SSPI / Kerberos / FEDAUTH wire detail
- [`DATA_TYPES.md`]({{ '/drivers/arrowtds/data-types/' | relative_url }}) — SQL Server → Arrow type mapping
- [`COMPATIBILITY.md`]({{ '/drivers/arrowtds/compatibility/' | relative_url }}) — supported SQL Server versions & editions
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowtds/troubleshooting/' | relative_url }}) — common connection failures
- [`EXAMPLES.md`]({{ '/drivers/arrowtds/examples/' | relative_url }}) — copy-paste recipes
- [`LICENSING.md`]({{ '/drivers/arrowtds/licensing/' | relative_url }}) — supplying your Arpeio licence
