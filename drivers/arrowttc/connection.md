---
title: Connection
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 1
permalink: /drivers/arrowttc/connection/
---

# ArrowTTC — Connection guide

Everything you can put in front of ArrowTTC to point it at an Oracle database, in
one place: the three ways to supply connection details, every option the driver
accepts, and the Oracle-specific rules (EZConnect, SID vs service name, TCPS
wallets, Oracle Cloud Autonomous Database).

For the wire-level detail of *how* each authentication method works (O5LOGON,
Kerberos, OCI IAM token, OAuth2/Entra ID, proxy, change-password, TLS), see
[`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}). For read throughput and batch sizing,
see the README `## Performance` / `### Batch Size` sections. For the licence, see
[`LICENSING.md`]({{ '/drivers/arrowttc/licensing/' | relative_url }}).

---

## Three ways to connect

Every driver in the Arpeio family accepts the same three connection forms. They
can be mixed; a **discrete option always wins** over the same field taken from a
`connection_string` or a `uri`, regardless of the order they are set.

| Form | Option key | Grammar | Best for |
|---|---|---|---|
| **Discrete options** | `adbc.arrowttc.<field>` | one option per field | programmatic clients, secrets kept out of a single string |
| **Connection string** | `adbc.arrowttc.connection_string` | ADO.NET `Key=Value;…` (+ Oracle EZConnect in `Server`) | pasting an existing string |
| **Connection URI** | `uri` | `oracle://…` URL | portable tooling, copy-paste from other Oracle ADBC tools |

```python
import adbc_driver_manager.dbapi as dbapi

# Discrete options
conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "adbc.arrowttc.server":       "localhost",
    "adbc.arrowttc.port":         "1521",
    "adbc.arrowttc.service_name": "orclpdb1",
    "adbc.arrowttc.username":     "scott",
    "adbc.arrowttc.password":     "tiger",
})

# Connection string (ADO.NET keywords, or Oracle EZConnect in Server)
conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "adbc.arrowttc.connection_string":
        "Server=localhost:1521/orclpdb1;User ID=scott;Password=tiger",
})

# Connection URI (Oracle URL grammar)
conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "uri": "oracle://scott:tiger@localhost:1521/orclpdb1?ssl_mode=verify-full",
})
```

**Precedence** is enforced by a per-field bitmask shared between the parsed
result and the database handle: a parsed field from `connection_string`/`uri` is
applied only if no discrete `adbc.arrowttc.*` option already claimed it. Secrets
held transiently on the stack during parsing are scrubbed (`OPENSSL_cleanse`) on
every return path.

---

## Option reference

All options are **string-typed** and live under the `adbc.arrowttc.*` namespace.
Set them as ADBC database options before the connection is opened. Statement and
ingest options are set on the statement — see
[Statement & ingest options](#statement--ingest-options).

### Target — where to connect

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowttc.server` | — | Host name or IP. Also accepts an Oracle EZConnect descriptor (`host:port/service`) — see [EZConnect](#ezconnect). |
| `adbc.arrowttc.port` | 1521 | TCP port, `1..65535`. TCPS listeners conventionally use **2484** — set it explicitly. |
| `adbc.arrowttc.service_name` | — | Oracle **service name** / PDB. The usual way to name a database. |
| `adbc.arrowttc.sid` | — | Oracle **SID**. Selects a *different* `CONNECT_DATA` form on the wire — mutually exclusive with `service_name` (giving both is rejected). |
| `adbc.arrowttc.app_name` | driver default | Program name reported to the server (`V$SESSION.PROGRAM`). |

### Authentication

Full method-by-method detail is in [`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}).

| Option | Values | Meaning |
|---|---|---|
| `adbc.arrowttc.username` / `.password` | string | Oracle credentials (O5LOGON challenge-response — the password is never sent in clear, even without TLS). |
| `adbc.arrowttc.auth_method` | `password` (default) / `kerberos` / `token` / `oauth2` | Selects the login method. `token`/`oauth2` are auto-selected when their token options are set. |
| `adbc.arrowttc.proxy_user` | schema | Authenticate as `username`, open the session in this user's schema via Oracle `CONNECT THROUGH` proxy. |
| `adbc.arrowttc.new_password` | string | Change `username`'s password as part of login — the way in past an expired one. |
| `adbc.arrowttc.token_location` | dir | OCI IAM database token: directory holding the token and its PEM private key (as `oci iam db-token get` writes). Selects `auth_method=token`. |
| `adbc.arrowttc.access_token` | JWT | OAuth2 / Microsoft Entra ID bearer token, inline. Selects `auth_method=oauth2`. |
| `adbc.arrowttc.token_file` | path | Same OAuth2 bearer token, read from a file (inline `access_token` wins if both are set). |
| `adbc.arrowttc.krb5_cred_mode` | `ccache` / `keytab` / `password` | Kerberos credential source. |
| `adbc.arrowttc.krb5_keytab` | path | Kerberos keytab (service accounts / CI). |
| `adbc.arrowttc.krb5_ccache` | path | Kerberos credential cache (the ambient `kinit` ticket). |
| `adbc.arrowttc.krb5_principal` | name | Kerberos client principal. |
| `adbc.arrowttc.krb5_password` | string | Password to obtain a TGT (password-mode Kerberos). |
| `adbc.arrowttc.krb5_spn` | SPN | Service principal name of the database (e.g. `oracle/db.example.com`). |
| `adbc.arrowttc.krb5_realm` | realm | Kerberos realm override. |

### Encryption / TLS (TCPS)

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowttc.ssl_mode` | `disable` | `disable` (plain TCP) / `require` (encrypt, cert **not** verified) / `verify-ca` (chain verified) / `verify-full` (chain **and** hostname — use this one). |
| `adbc.arrowttc.ssl_root_cert` | — | PEM CA bundle for `verify-ca`/`verify-full` (else the system trust store). |
| `adbc.arrowttc.wallet_location` | — | Directory holding `ewallet.pem` (client key + cert + CAs), python-oracledb-thin layout. This is exactly an unzipped **Oracle Cloud ADB** wallet. |
| `adbc.arrowttc.wallet_password` | — | Unlocks an encrypted key inside the wallet. |

> `ssl_mode` is `disable` by default — **the connection is plaintext unless you
> say otherwise.** O5LOGON keeps the password off the wire, but SQL text and
> result data are in clear until you set `ssl_mode` or Native Network Encryption.

### Native Network Encryption (NNE)

{: .since }
> Since ArrowTTC v0.1.3.

Oracle's transport-layer encryption, negotiated after ACCEPT — no TCPS required.
See [`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}) for the wire detail.

| Option | Values | Meaning |
|---|---|---|
| `adbc.arrowttc.encryption` | `accepted` (default) / `rejected` / `requested` / `required` | Client stance for AES-256 payload encryption. `required` fails the connect if the server will not encrypt. |
| `adbc.arrowttc.data_integrity` | same four values | Client stance for the SHA-256 (or SHA-1) integrity checksum. |

Only AES-256 and SHA-256/SHA-1 are offered — the legacy ciphers (DES/3DES/RC4)
and MD5 are never proposed. `accepted` leaves an existing plaintext connection
byte-for-byte unchanged.

### Timeouts & performance

| Option | Default | Meaning |
|---|---|---|
| `adbc.arrowttc.login_timeout` | driver default | TCP connect budget in seconds (`0` = no limit). |
| `adbc.arrowttc.socket_timeout` | driver default | Per-socket read timeout in seconds (`0` = no limit). |
| `adbc.arrowttc.batch_size` | 100000 | Rows per streamed Arrow batch (also `Buffer Size` in a connection string). A **memory** knob, not a throughput knob — see the README `### Batch Size`. |
| `adbc.arrowttc.statement_cache_size` | 20 | Number of server cursors held open and reused per connection, keyed by SQL text. Reusing a cursor skips the hard parse and prevents the per-statement cursor leak that would otherwise exhaust `OPEN_CURSORS` on a long-lived connection. `0` disables reuse (cursors are still closed, just not reused). Matches oracledb's `stmtcachesize`. |

### Type rendering (read path)

| Option | Values | Meaning |
|---|---|---|
| `adbc.arrowttc.number_mapping` | `auto` (default) / `double` / `decimal:P,S` | How Oracle's *unconstrained* `NUMBER` maps to Arrow. `auto` → lossless `utf8`; `double` → `float64` (lossy); `decimal:P,S` → `decimal128(P,S)`. See [`DATA_TYPES.md`]({{ '/drivers/arrowttc/data-types/' | relative_url }}#number-mapping). |

### Licence

| Option | Meaning |
|---|---|
| `arpeio.adbc.license` | Licence blob, inline. |
| `arpeio.adbc.license_file` | Path to a `.lic` file. |
| `arpeio.adbc.license.status` | **Read-only** (`GetOption`); reports `<state>;code=<ARROW_LIC_*>;tier=<tier>;expires=<epoch>`. |

The driver also reads the shared `ARPEIO_ADBC_LICENCE[_FILE]` environment
variables and an `arpeio_adbc.lic` file next to the library. Full resolution
order in [`LICENSING.md`]({{ '/drivers/arrowttc/licensing/' | relative_url }}).

### Standard ADBC connection options

These use the ADBC-standard keys (no `arrowttc` namespace):

| Option | Values | Meaning |
|---|---|---|
| `adbc.connection.autocommit` | `true`/`false` | Autocommit mode. `Connection.Commit` / `Rollback` drive an explicit transaction otherwise. |

### Statement & ingest options

Set on the statement handle, not the database:

| Option | Default | Meaning |
|---|---|---|
| `adbc.ingest.target_table` | — | Target table for `adbc_ingest`. |
| `adbc.ingest.mode` | `create` | `create` / `append` / `replace` / `create_append`. |
| `adbc.ingest.target_db_schema` | — | Target schema for ingest. |
| `adbc.ingest.target_catalog` | — | Oracle reaches one catalog per session; a value that is not the connected service is rejected (`NOT_IMPLEMENTED`). |
| `adbc.ingest.temporary` | `false` | Ingest into a `GLOBAL TEMPORARY` table (`ON COMMIT PRESERVE ROWS`). |

---

## Connection string (ADO.NET form)

`adbc.arrowttc.connection_string` accepts the ADO.NET `Key=Value;…` grammar:
case-insensitive keywords, quoted values, doubled-quote escapes. The `Server`
keyword additionally accepts an Oracle **EZConnect** descriptor (see below).

| Keyword(s) | Option |
|---|---|
| `Server`, `Data Source`, `Host`, `Address`, `Addr` | `server` (+ EZConnect `host:port/service`) |
| `Port` | `port` |
| `Service Name`, `ServiceName`, `Database`, `Initial Catalog` | `service_name` |
| `SID` | `sid` |
| `User ID`, `UID`, `User`, `Username` | `username` |
| `Password`, `PWD` | `password` |
| `Application Name`, `App` | `app_name` |
| `Connection Timeout`, `Connect Timeout`, `Timeout` | `login_timeout` |
| `Socket Timeout` | `socket_timeout` |
| `Buffer Size` | `batch_size` |
| `Number Mapping` | `number_mapping` |
| `SSL Mode`, `sslmode` | `ssl_mode` |
| `SSL Root Cert`, `sslrootcert` | `ssl_root_cert` |
| `Wallet Location` | `wallet_location` |
| `Wallet Password` | `wallet_password` |
| `Proxy User` | `proxy_user` |
| `New Password` | `new_password` |
| `Encryption` | `encryption` |
| `Data Integrity` | `data_integrity` |
| `Auth Method` | `auth_method` |
| `Kerberos Cred Mode`, `Kerberos Keytab`, `Kerberos Cache`, `Kerberos SPN`, `Kerberos Realm`, `Kerberos Principal`, `Kerberos Password` | the matching `krb5_*` option |

```
Server=localhost:1521/orclpdb1;User ID=scott;Password=tiger;sslmode=verify-full
```

`Database` / `Initial Catalog` map to the **service name** — Oracle's unit of
connection is a service (or a PDB), and callers coming from the sibling drivers
reach for `Database` first. `SID` and `Service Name` are mutually exclusive.

---

## Connection URI

The standard ADBC `uri` option takes an Oracle-style URL, so a string copied from
other Oracle ADBC tooling works unchanged.

```
<scheme>://[user[:password]@]host[:port][/service_name][?key=value&…]
```

- **Schemes** (case-insensitive, equivalent): `oracle://` and the branded
  `arrowttc://`. The scheme only selects the URL grammar — it never picks the
  driver, so `oracle://` never collides with another Oracle ADBC driver
  installed alongside this one.
- The **path segment is the Oracle *service name***. Use `?sid=<SID>` for the SID
  `CONNECT_DATA` form instead; giving both a service name (path or
  `?service_name=`) and a `?sid=` is rejected.
- `userinfo`, host, and query values are **percent-decoded**; query values treat
  `+` as a space. Bracketed IPv6 hosts use `[::1]:1521`.
- For the ADO.NET `key=value;` form, use `connection_string`, not `uri` (a `uri`
  without a `scheme://` is rejected with a pointer to `connection_string`).

**Query parameters** (unknown or repeated parameters are rejected):

| Parameter(s) | Option |
|---|---|
| `user` | `username` |
| `password` | `password` |
| `service_name` | `service_name` |
| `sid` | `sid` |
| `ssl_mode` | `ssl_mode` |
| `ssl_root_cert` | `ssl_root_cert` |
| `wallet_location` | `wallet_location` |
| `wallet_password` | `wallet_password` |
| `encryption` | `encryption` |
| `data_integrity` | `data_integrity` |
| `connect_timeout` | `login_timeout` |
| `application_name`, `app_name` | `app_name` |
| `number_mapping` | `number_mapping` |
| `proxy_user` | `proxy_user` |

```
oracle://scott:tiger@dbhost:2484/orclpdb1?ssl_mode=verify-full&wallet_location=/etc/oracle/wallet
oracle://scott:tiger@dbhost:1521/?sid=ORCL
```

---

## EZConnect

Oracle lets a whole connect descriptor be written as `host:port/service`, and it
is what most users reach for. ArrowTTC accepts it wherever a `Server` is given
(discrete `server`, the `Server` connection-string keyword):

```
Server=dbhost                       # host only, service/SID given separately
Server=dbhost:1521/orclpdb1         # host:port/service
Server=//dbhost:1521/orclpdb1       # optional leading //
Server=dbhost,1521                  # ADO.NET host,port form (cross-driver muscle memory)
```

- The `/service` suffix sets the **service name** (`use_sid=false`).
- A bare IPv6 literal is **rejected** rather than mis-parsed — give the host and
  `Port` as separate keywords, or use the bracketed `[::1]` form in a `uri`.

---

## Oracle Cloud Autonomous Database (ADB)

{: .since }
> Since ArrowTTC v0.2.0.

A downloaded ADB instance wallet works as-is (mutual TLS). Point
`wallet_location` at the unzipped wallet directory (which contains
`ewallet.pem`), set `wallet_password`, use `ssl_mode=verify-full`, and connect to
one of the wallet's `_low` / `_tp` / … services on port **1522**:

```python
db_kwargs = {
    "adbc.arrowttc.server":          "adb.<region>.oraclecloud.com",
    "adbc.arrowttc.port":            "1522",
    "adbc.arrowttc.service_name":    "<svc>_low.adb.oraclecloud.com",
    "adbc.arrowttc.username":        "scott",
    "adbc.arrowttc.password":        "tiger",
    "adbc.arrowttc.ssl_mode":        "verify-full",
    "adbc.arrowttc.wallet_location": "/home/you/wallet",
    "adbc.arrowttc.wallet_password": "<wallet-pw>",
}
```

Oracle's `cwallet.sso` cannot be read — convert it with
`orapki wallet pkcs12_to_pem -wallet <dir>` to produce `ewallet.pem`. ADB also
accepts **OCI IAM token** and **OAuth2 / Entra ID** login instead of a password
(both still require the wallet for mutual TLS) — see
[`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}#oci-iam-token-oracle-cloud).

---

## See also

- [`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}) — O5LOGON / Kerberos / OCI IAM / OAuth2 / TLS wire detail
- [`DATA_TYPES.md`]({{ '/drivers/arrowttc/data-types/' | relative_url }}) — Oracle → Arrow type mapping
- [`COMPATIBILITY.md`]({{ '/drivers/arrowttc/compatibility/' | relative_url }}) — supported Oracle versions & cloud services
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowttc/troubleshooting/' | relative_url }}) — common connection failures
- [`EXAMPLES.md`]({{ '/drivers/arrowttc/examples/' | relative_url }}) — copy-paste recipes
- [`LICENSING.md`]({{ '/drivers/arrowttc/licensing/' | relative_url }}) — supplying your Arpeio licence
