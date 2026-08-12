---
title: Environment variables
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 6
permalink: /drivers/arrowttc/environment-variables/
---

# Environment variables and options

ArrowTTC is configured almost entirely through ADBC options — a connection
string or structured `adbc.arrowttc.*` keys. A handful of environment variables
control diagnostics and low-level behaviour that has no place in a connection
string.

## Connection options

Set as structured options (`AdbcDatabaseSetOption`) under the `adbc.arrowttc.*`
namespace, or in a connection string / `uri`. Structured options always win
over the same field parsed from a connection string.

| Structured key | Connection-string keyword(s) | Meaning |
|---|---|---|
| `adbc.arrowttc.server` | `Server`, `Data Source`, `Host` | Host, or a full EZConnect string (`host:port/service`) |
| `adbc.arrowttc.port` | `Port` | Listener port (default 1521; TCPS is conventionally 2484) |
| `adbc.arrowttc.service_name` | `Service Name`, `Database` | Oracle service name |
| `adbc.arrowttc.sid` | `SID` | Oracle SID (mutually exclusive with a service name) |
| `adbc.arrowttc.username` | `User ID`, `UID` | Login user |
| `adbc.arrowttc.password` | `Password`, `PWD` | Login password |
| `adbc.arrowttc.app_name` | `Application Name` | Reported in `V$SESSION` |
| `adbc.arrowttc.login_timeout` | `Connection Timeout` | TCP connect timeout, seconds |
| `adbc.arrowttc.socket_timeout` | `Socket Timeout` | Per-I/O recv/send timeout, seconds (0 = none) |
| `adbc.arrowttc.batch_size` | `Buffer Size` | Rows per Arrow batch (default 100,000) |
| `adbc.arrowttc.statement_cache_size` | — | Server cursors held open and reused per connection (default 20; 0 disables) |
| `adbc.arrowttc.number_mapping` | `Number Mapping` | Unconstrained-`NUMBER` mapping: `auto` / `double` / `decimal:P,S` |
| `adbc.arrowttc.ssl_mode` | `SSL Mode`, `sslmode` | `disable` / `require` / `verify-ca` / `verify-full` |
| `adbc.arrowttc.ssl_root_cert` | `SSL Root Cert`, `sslrootcert` | PEM CA bundle for chain verification |
| `adbc.arrowttc.wallet_location` | `Wallet Location` | Directory holding `ewallet.pem` |
| `adbc.arrowttc.wallet_password` | `Wallet Password` | Passphrase for an encrypted wallet key |
| `adbc.arrowttc.proxy_user` | `Proxy User` | Target schema for proxy (`CONNECT THROUGH`) auth |
| `adbc.arrowttc.new_password` | `New Password` | Change the login password during connect |
| `adbc.arrowttc.encryption` | `Encryption` | Native Network Encryption stance: `accepted` (default) / `rejected` / `requested` / `required` (AES-256) |
| `adbc.arrowttc.data_integrity` | `Data Integrity` | Native checksum stance, same four values (SHA-256 or SHA-1) |
| `adbc.arrowttc.auth_method` | `Auth Method` | `password` (default), `kerberos` (KERBEROS5 external auth; Linux/MIT-krb5), or `token` (OCI IAM database token) |
| `adbc.arrowttc.token_location` | `Token Location` | Directory with the OCI db-token (`token`) and its PEM key (`oci_db_key.pem`); `token` auth. As written by `oci iam db-token get` |
| `adbc.arrowttc.krb5_cred_mode` | `Kerberos Cred Mode` | How the ticket is obtained: `ccache` (default) / `keytab` / `password` |
| `adbc.arrowttc.krb5_keytab` | `Kerberos Keytab` | Keytab file path (`keytab` mode) |
| `adbc.arrowttc.krb5_ccache` | `Kerberos Cache` | Per-connection credential cache (`ccache` mode; else `KRB5CCNAME`) |
| `adbc.arrowttc.krb5_spn` | `Kerberos SPN` | Target service principal; defaults to `oracle/<host>` |
| `adbc.arrowttc.krb5_realm` | `Kerberos Realm` | Default realm for the client principal |
| `adbc.arrowttc.krb5_principal` | `Kerberos Principal` | Client principal (`keytab` / `password` modes) |
| `adbc.arrowttc.krb5_password` | `Kerberos Password` | Client password (`password` mode; cleansed on release) |
| `adbc.arrowttc.connection_string` | `uri` | A whole connection string as one value |
| `arpeio.adbc.license` | — | Inline ARROW LICENCE token |
| `arpeio.adbc.license_file` | — | Path to a licence file |
| `arpeio.adbc.license.status` *(read-only)* | — | `AdbcDatabaseGetOption` reports `<state>;code=<ARROW_LIC_*>;tier=<tier>;expires=<epoch>` |

## Environment variables

Two namespaces, by design. **Generic** family-wide diagnostic knobs — the ones
that mean the same thing in every Arpeio ADBC driver (ArrowTDS/FEBE/TTC/DRDA) —
use the shared **`ARPEIO_ADBC_*`** prefix, so a trace or an A/B recipe carries
across drivers unchanged. **Driver- or protocol-specific** knobs keep the
**`ARROWTTC_*`** prefix (e.g. `ARROWTTC_NO_LOB_PREFETCH`, an Oracle-LOB escape
hatch with no analogue elsewhere). Each variable lives in exactly one namespace;
there is no aliasing or override between them.

| Variable | Values | Effect |
|---|---|---|
| `ARPEIO_ADBC_READ_TIMING` | `1` | Emit a `[TTC READ]` line per batch and a `[TTC READ SUMMARY]` at stream release on stderr, attributing an extract to the wire (`recv`), the decoder (`decode`), and the consumer. See below. |
| `ARPEIO_ADBC_ALLOCATOR` | `system`, `mimalloc` | Force the column-buffer allocator backend, read once on the first allocation. `system` always works; `mimalloc` is honoured only when the driver was built with `ARROWTTC_WITH_MIMALLOC` (the default). |
| `ARROWTTC_NO_LOB_PREFETCH` | any non-empty | Disable prefetching LOB lengths with the initial define. Prefetch shows no gain on loopback but saves a round trip on a real network; this is the escape hatch if a server mishandles it. |
| `ARPEIO_ADBC_LICENCE` | inline token | Licence token, if no option is set |
| `ARPEIO_ADBC_LICENCE_FILE` | path | Licence file, if no option is set |

A licence is also looked for in `arpeio_adbc.lic` next to the driver library.
Sources are tried in precedence order: inline option, file option, env inline,
env file, default file. The gate is **always enforced**: `DatabaseInit` refuses
without a valid token. The `arpeio.adbc.license.status` option and a grace
warning still report the licence state. Test/CI builds set
`ARROWTTC_LICENSE_TEST_KEY=ON` to self-mint a token offline; a shipped driver
trusts only the production key.

Note: `ADBC_ARROWTTC_*` (uppercase) is **not** read by the driver. Some of the
test and benchmark scripts read `ADBC_ARROWTTC_SERVER` etc. as a credential
source, but that is a convenience of those scripts, not driver behaviour.

### Reading the timing summary

```
[TTC READ SUMMARY] batches=61 rows=6001215 wall_ms=2090 busy_ms=2075 \
  recv_ms=72 decode_ms=2003 consumer_ms=2 cpu_ms=2057 blocked_ms=18 \
  MiB=596 MiB/s=285 alloc=mimalloc
```

- `recv_ms` — time inside `recv()`/`SSL_read()`. High means the wire (or the
  server) is the bottleneck.
- `decode_ms` — `busy_ms − recv_ms`: turning wire bytes into Arrow buffers.
- `consumer_ms` — wall time spent outside the driver, i.e. waiting for the
  caller to take each batch. High means the consumer is the bottleneck.
- `blocked_ms` — `busy_ms − cpu_ms`: descheduled time the driver was neither on
  a CPU nor in `recv`. Under parallel load this is the allocator convoy signal.
- `alloc` — the resolved allocator backend, `system` or `mimalloc`.
