---
title: Connection
layout: default
parent: ArrowFEBE
grand_parent: Drivers
nav_order: 1
permalink: /drivers/arrowfebe/connection/
---

# ArrowFEBE — Connection guide

Everything you can put in front of ArrowFEBE to point it at a PostgreSQL server,
in one place: the three ways to supply connection details, every option the
driver accepts, and the PostgreSQL-specific rules (TLS / `sslmode`, SCRAM channel
binding, integrated Kerberos/GSSAPI, GSS transport encryption).

For the wire-level detail of *how* each authentication method works (SCRAM-SHA-256
+ channel binding, Kerberos/GSSAPI, SSPI, `gssencmode`), see
[`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/' | relative_url }}). For read/ingest throughput knobs, see
`POSTGRES_TUNING.md`. For the licence, see
[`LICENSING.md`]({{ '/drivers/arrowfebe/licensing/' | relative_url }}).

---

## Three ways to connect

Every driver in the Arpeio family accepts the same three connection forms. They
can be mixed; a **discrete option always wins** over the same field taken from a
`connection_string` or a `uri`, regardless of the order they are set.

| Form | Option key | Grammar | Best for |
|---|---|---|---|
| **Discrete options** | `adbc.arrowfebe.<field>` | one option per field | programmatic clients, secrets kept out of a single string |
| **Connection string** | `adbc.arrowfebe.connection_string` | ADO.NET `Key=Value;…` | pasting an existing SQL Server / SqlClient-style string |
| **Connection URI** | `uri` | `postgresql://…` URL | portable tooling, copy-paste from libpq / psql / ADBC tools |

<p class="code-lang-label">Python</p>

```python
import adbc_driver_manager.dbapi as dbapi

# Discrete options
conn = dbapi.connect(driver="arrowfebe", db_kwargs={
    "adbc.arrowfebe.server":   "localhost",
    "adbc.arrowfebe.database": "appdb",
    "adbc.arrowfebe.username": "dbuser",
    "adbc.arrowfebe.password": "<password>",
    "adbc.arrowfebe.sslmode":  "require",
}, autocommit=True)

# Connection string (ADO.NET grammar)
conn = dbapi.connect(driver="arrowfebe", db_kwargs={
    "adbc.arrowfebe.connection_string":
        "Server=localhost;Database=appdb;User ID=dbuser;Password=<password>",
}, autocommit=True)

# Connection URI (libpq grammar)
conn = dbapi.connect(driver="arrowfebe", db_kwargs={
    "uri": "postgresql://dbuser:<password>@localhost:5432/appdb?sslmode=require",
}, autocommit=True)
```

**Precedence** is enforced by a per-field bitmask shared between the parsed
result and the database handle: a parsed field from `connection_string`/`uri` is
applied only if no discrete `adbc.arrowfebe.*` option already claimed it. Secrets
held transiently on the stack during parsing are scrubbed on every return path.

---

## Option reference

All options are **string-typed** and live under the `adbc.arrowfebe.*` namespace
(a few also accept a short bare alias, noted below). Set them as ADBC database
options before the connection is opened. Statement/ingest options are set on the
statement — see [Statement & ingest options](#statement--ingest-options).

### Target — where to connect

| Option | Alias | Default | Meaning |
|---|---|---|---|
| `adbc.arrowfebe.server` | `hostname` | — | Host name or address. IPv6 literals use the bracketed form `[::1]`; a `tcp:` prefix and a `,port` / `:port` suffix in a `connection_string` value are accepted. |
| `adbc.arrowfebe.port` | `port` | 5432 | TCP port, `1..65535`. |
| `adbc.arrowfebe.database` | `database` | server default (the role's default DB) | Initial database. A connection sees exactly one database (no cross-catalog). |
| `adbc.arrowfebe.username` | `username` | OS user (server-dependent) | Login role. |
| `adbc.arrowfebe.password` | `password` | — | Password for the password-family auth methods. |
| `adbc.arrowfebe.application_name` | `application_name` | driver default | Program name reported in the startup packet (`application_name` GUC, `pg_stat_activity.application_name`). |

### Authentication

Full method-by-method detail is in [`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/' | relative_url }}); the
TLS / channel-binding options are under [TLS & encryption options](#tls--encryption-options) and the
integrated path under
[Integrated / Kerberos authentication](#integrated--kerberos-authentication).

| Option | Values | Meaning |
|---|---|---|
| `adbc.arrowfebe.username` / `.password` | string | Password-family credentials (SCRAM-SHA-256 / MD5 / cleartext, negotiated by the server). Aliases `username` / `password`. |
| `adbc.arrowfebe.trusted` | `true`/`sspi`/`false` | Integrated auth: POSIX Kerberos/GSSAPI or Windows SSPI, no username/password. Alias `trusted`. |
| `adbc.arrowfebe.auth_type` | `sql`/`SqlPassword`, `integrated`/`sspi`/`trusted` | Selects the auth mode explicitly. The Entra ID / certificate families are recognised and rejected (ArrowFEBE is the driver — it cannot delegate to a client library). |
| `adbc.arrowfebe.krb5.keytab` | path | Kerberos keytab for integrated auth (service accounts that cannot `kinit`). |
| `adbc.arrowfebe.krb5.ccache` | path | Kerberos credential cache (default: the ambient ccache from `kinit`). |
| `adbc.arrowfebe.krb5.principal` | name | Kerberos client principal (e.g. `dbuser@REALM`), with `krb5.password` for a programmatic kinit. |
| `adbc.arrowfebe.krb5.password` | string | Password to obtain a TGT for `krb5.principal`. |
| `adbc.arrowfebe.krb5.spn` | SPN | Override the full derived service principal name (e.g. `postgres/db.example.com@REALM`). |
| `adbc.arrowfebe.krbsrvname` | name | Override the service-name component of the SPN (default `postgres`, mirroring libpq). |
| `adbc.arrowfebe.gssencmode` | `disable` (default) / `prefer` / `require` | GSS transport encryption, negotiated before TLS and the startup packet. `require` skips TLS entirely. POSIX/GSSAPI builds only. See [`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/#gss-transport-encryption-gssencmode' | relative_url }}). |

### TLS & encryption options

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowfebe.sslmode` | `prefer` | `disable` / `prefer` / `require` / `verify-ca` / `verify-full`, via the FEBE `SSLRequest` handshake. `require` encrypts but does **not** verify the certificate; only `verify-ca`/`verify-full` do (and `verify-full` also checks the hostname). |
| `adbc.arrowfebe.ssl_root_cert` | system CA store | PEM root-CA file used by `verify-ca` / `verify-full`. |
| `adbc.arrowfebe.channel_binding` | `prefer` | `prefer` / `require` / `disable`. `require` enforces SCRAM-SHA-256-**PLUS** channel binding over TLS and refuses a PLUS-stripping downgrade. |
| `adbc.arrowfebe.encrypt` | — | Compatibility bridge for the ADO.NET TLS vocabulary. An explicit `sslmode` always wins; only when none is given does `encrypt=true` map to `require` (or `verify-full` when `trust_server_cert=false`). Prefer `sslmode` directly. |
| `adbc.arrowfebe.trust_server_cert` | — | Part of the same bridge (see `encrypt`). Prefer `sslmode` directly. |

> **⚠️ Security — the default `sslmode=prefer` gives no MITM protection.** Like
> libpq, `prefer` falls back to plaintext when the server answers the
> `SSLRequest` with `N`, and that reply is read **before** TLS so it is
> unauthenticated. Over an untrusted network use `sslmode=verify-full` and/or
> `channel_binding=require`. See [TLS & `sslmode`](#tls--sslmode) below.

### Timeouts & performance

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowfebe.login_timeout` | 30 | TCP connect budget in seconds (`0` = no limit). |
| `adbc.arrowfebe.socket_timeout` | 0 | Per-I/O recv/send timeout in seconds (`0` = none). Set it to bound a long-running or stalled query. |
| `adbc.arrowfebe.query_timeout` | 0 | Per-query timeout in seconds (`0` = no limit). |
| `adbc.arrowfebe.buffer_size` | 100000 | Rows per streamed Arrow batch (per connection). Capped at 10,000,000; large values are safe (the driver auto-flushes early if a wide `utf8`/`binary` column would cross Arrow's 2 GiB offset limit — see `POSTGRES_TUNING.md`). |

More tuning guidance in `POSTGRES_TUNING.md`.

### Type rendering (read path)

| Option | Values | Meaning |
|---|---|---|
| `adbc.arrowfebe.uuid_casing` | `lower` (default) / `upper` | Hex-digit case of `uuid` values rendered to Arrow `utf8`. `lower` matches RFC 4122 and PostgreSQL's text output. |
| `adbc.arrowfebe.geospatial` | `geoarrow.wkb` (default) / `wkb` / `binary` | How PostGIS `geometry`/`geography` columns are read. See [`DATA_TYPES.md`]({{ '/drivers/arrowfebe/data-types/#geospatial-types' | relative_url }}). |

### Licence

| Option | Meaning |
|---|---|
| `arpeio.adbc.license` | Licence blob, inline. |
| `arpeio.adbc.license_file` | Path to a `.lic` file. |
| `arpeio.adbc.license.status` | **Read-only** (`GetOption`); reports `<state>;code=<ARROW_LIC_*>;tier=<tier>;expires=<epoch>`. |

The driver also reads the shared `ARPEIO_ADBC_LICENCE[_FILE]` environment
variables and an `arpeio_adbc.lic` file next to the library. Full resolution
order in [`LICENSING.md`]({{ '/drivers/arrowfebe/licensing/' | relative_url }}).

### Standard ADBC connection options

These use the ADBC-standard keys (no `arrowfebe` namespace):

| Option | Values | Meaning |
|---|---|---|
| `adbc.connection.autocommit` | `true`/`false` | Autocommit mode. Set `true` for the single-connection `TRUNCATE`/`adbc_ingest` pattern. |

> Transaction **isolation level** and session **read-only** are not yet exposed
> as ADBC options on ArrowFEBE; set them from SQL (`SET TRANSACTION …`,
> `SET default_transaction_read_only`) on the connection if needed.

### Statement & ingest options

Set on the statement handle, not the database:

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowfebe.batch_size` | inherits `buffer_size` | Rows per Arrow batch for this statement. |
| `adbc.arrowfebe.max_batch_size` | 1000000 | Upper bound for `batch_size`. |
| `adbc.arrowfebe.memory_budget_mb` | 256 | Decode memory budget, `16..8192` MB (option surface). |
| `adbc.ingest.mode` | `create` | `create` / `append` / `replace` / `create_append`. |
| `adbc.ingest.target_catalog` | — | Target database for ingest. |
| `adbc.ingest.target_db_schema` | — | Target schema for ingest. |
| `adbc.ingest.temporary` | `false` | Ingest into a `TEMP` table (same-session fast path). |

### Accepted no-ops (back-compat)

Recognised and ignored so strings copied from other Arpeio drivers still load:
`adbc.arrowfebe.prefetch` (the double-buffered prefetch path was removed —
streaming already overlaps recv with decode) and the legacy bulk knobs
`arrowfebe.bulk_batch_size`, `arrowfebe.bulk_tablock`,
`arrowfebe.bulk_keep_identity`, `arrowfebe.bulk_check_constraints`,
`arrowfebe.bulk_fire_triggers`, `arrowfebe.bulk_keep_nulls` (SQL-Server-era
ingest hints with no PostgreSQL equivalent).

---

## Connection string (ADO.NET form)

`adbc.arrowfebe.connection_string` accepts the ADO.NET `Key=Value;…` grammar
(inherited from ArrowTDS for cross-family compatibility): case-insensitive
keywords, quoted values, doubled-quote escapes. Keywords map onto the options
above.

| Keyword(s) | Option |
|---|---|
| `Server`, `Data Source`, `Address`, `Addr`, `Network Address` | `server` |
| `Database`, `Initial Catalog` | `database` |
| `User ID`, `UID`, `User` | `username` |
| `Password`, `PWD` | `password` |
| `Application Name`, `App` | `application_name` |
| `Encrypt` | `encrypt` |
| `TrustServerCertificate` | `trust_server_cert` |
| `Connection Timeout`, `Connect Timeout`, `Timeout` | `login_timeout` |
| `Integrated Security` (`true`/`sspi`/`yes`), `Trusted_Connection` | `trusted` |
| `Authentication`, `Authenticator` | auth mode (`SqlPassword` / integrated) |
| `Krb5 Keytab File`, `Krb5 Credential Cache`, `Krb5 Principal`, `Krb5 Password`, `Service Principal Name` | the `krb5.*` family |
| `GssEncMode` | `gssencmode` |
| `KrbSrvName` | `krbsrvname` |

```
Server=localhost;Database=appdb;User ID=dbuser;Password=<password>;Integrated Security=false
```

---

## Connection URI

{: .since }
> Since ArrowFEBE v0.3.6.

The standard ADBC `uri` option takes a **libpq-style PostgreSQL URI**, so a
string copied from psql, a `DATABASE_URL`, or other PostgreSQL ADBC tooling works
unchanged.

```
<scheme>://[user[:password]@]host[:port][/dbname][?key=value&…]
```

- **Schemes** (case-insensitive, equivalent): `postgresql://`, `postgres://`, and
  the branded `arrowfebe://`. The scheme only selects the URL grammar — it never
  picks the driver, so `postgresql://`/`postgres://` never collide with another
  PostgreSQL ADBC driver installed alongside this one.
- The **path segment is the database name** (as in libpq) — not an instance name.
- `host`, `userinfo`, and query values are **percent-decoded**; a `+` in a query
  value decodes to a space. IPv6 hosts use the bracketed form:
  `postgresql://[::1]:5432/appdb`.

**Query parameters** (unknown or repeated parameters are rejected):

| Parameter | Option |
|---|---|
| `user` | `username` |
| `password` | `password` |
| `dbname` | `database` (alternative spelling for callers who put it in the query) |
| `application_name` | `application_name` |
| `connect_timeout` | `login_timeout` |
| `sslmode` | `sslmode` |
| `channel_binding` | `channel_binding` |
| `sslrootcert` | `ssl_root_cert` |
| `gssencmode` | `gssencmode` |
| `krbsrvname` | `krbsrvname` |

For the ADO.NET `Key=Value;` form use `connection_string` instead of `uri`.

---

## TLS & `sslmode`

ArrowFEBE negotiates TLS with the PostgreSQL `SSLRequest` handshake. The
`sslmode` values mirror libpq:

| `sslmode` | Encrypts | Verifies certificate | Verifies hostname |
|---|---|---|---|
| `disable` | no | — | — |
| `prefer` (default) | if the server offers it | no | no |
| `require` | yes | no | no |
| `verify-ca` | yes | yes (chain to `ssl_root_cert` / system store) | no |
| `verify-full` | yes | yes | yes |

- **`prefer` gives no MITM protection.** The server's response to `SSLRequest`
  (`S` = TLS, `N` = plaintext) is read *before* TLS, so an on-path attacker can
  strip it and observe the cleartext / MD5 / SCRAM-without-binding exchange. Over
  an untrusted network use `verify-full`.
- **SCRAM channel binding** (`channel_binding=require`) binds the SCRAM exchange
  to the server certificate (SCRAM-SHA-256-PLUS), defeating a MITM that proxies
  the SASL flow even without `verify-full`. It needs TLS, so it is unavailable
  under `sslmode=disable` or active GSS encryption.
- `verify-ca` / `verify-full` load the CA chain from `ssl_root_cert`, defaulting
  to the system CA store.

The `encrypt` / `trust_server_cert` options are a compatibility bridge for
callers who think in the ADO.NET vocabulary and are mapped to `sslmode` only when
no explicit `sslmode` is given. Prefer setting `sslmode` directly.

---

## Integrated / Kerberos authentication

{: .since }
> Since ArrowFEBE v0.2.0.

Set `adbc.arrowfebe.auth_type=integrated` (or `adbc.arrowfebe.trusted=true`) to
authenticate with no username/password: **POSIX Kerberos/GSSAPI** (FEBE auth
codes 7/8) on Linux/macOS, or **Windows SSPI** (code 9, Negotiate:
Kerberos with NTLM fallback). Mutual authentication is always required and
enforced — the driver refuses `AuthenticationOk` from a server that never
completed the GSS exchange, and refuses password-family requests while this mode
is active.

<p class="code-lang-label">Python</p>

```python
# 1. Obtain a ticket:  kinit dbuser@REALM
# 2. Connect — the ambient ccache is used automatically:
conn = dbapi.connect(driver="arrowfebe", db_kwargs={
    "adbc.arrowfebe.server":    "db.example.com",
    "adbc.arrowfebe.database":  "appdb",
    "adbc.arrowfebe.auth_type": "integrated",
    # optional explicit credentials:
    # "adbc.arrowfebe.krb5.ccache": "/tmp/krb5cc_1000",
    # "adbc.arrowfebe.krb5.keytab": "/etc/postgresql/pg.keytab",
})
```

**GSS transport encryption** (`adbc.arrowfebe.gssencmode`): `disable` (default —
a deliberate divergence from libpq's `prefer`), `prefer`, or `require`. `require`
encrypts the whole stream over GSSAPI and **skips TLS entirely**, so it cannot be
combined with `sslmode=require`/`verify-ca`/`verify-full`, and SCRAM channel
binding is unavailable (plain SCRAM still works inside the encrypted stream).
POSIX/GSSAPI builds only.

The full method-by-method reference, the SPN derivation, and the `febe_krb_diag`
diagnostic CLI are in [`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/' | relative_url }}).

---

## See also

- [`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/' | relative_url }}) — SCRAM / channel binding / Kerberos / SSPI / `gssencmode` wire detail
- [`DATA_TYPES.md`]({{ '/drivers/arrowfebe/data-types/' | relative_url }}) — PostgreSQL → Arrow type mapping
- [`COMPATIBILITY.md`]({{ '/drivers/arrowfebe/compatibility/' | relative_url }}) — supported PostgreSQL versions & forks
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowfebe/troubleshooting/' | relative_url }}) — common connection failures
- [`EXAMPLES.md`]({{ '/drivers/arrowfebe/examples/' | relative_url }}) — copy-paste recipes
- [`LICENSING.md`]({{ '/drivers/arrowfebe/licensing/' | relative_url }}) — supplying your Arpeio licence
