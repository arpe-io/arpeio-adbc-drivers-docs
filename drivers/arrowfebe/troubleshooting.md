---
title: Troubleshooting
layout: default
parent: ArrowFEBE
grand_parent: Drivers
nav_order: 8
permalink: /drivers/arrowfebe/troubleshooting/
---

# ArrowFEBE — Troubleshooting

Common failures and their fixes, grouped by where they show up: connecting,
TLS/certificates, authenticating, the licence gate, and querying/ingesting. For
the option details referenced here, see [`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/' | relative_url }}).

- [Connection](#connection)
- [TLS / certificates](#tls--certificates)
- [Authentication](#authentication)
- [Licence](#licence)
- [Query & ingest](#query--ingest)
- [Diagnostics: timing & debug traces](#diagnostics-timing--debug-traces)

---

## Connection

**Connect hangs, then fails after ~30 s.** The TCP connect budget
(`login_timeout`, default 30 s) elapsed — usually the host/port is unreachable: a
firewall, a wrong port, or the server not listening. Verify with
`nc -vz <host> 5432`. Lower `login_timeout` to fail faster while diagnosing.

**A query hangs indefinitely and never times out.** `socket_timeout` defaults to
`0` (no per-I/O timeout). Set `adbc.arrowfebe.socket_timeout` to a bound in
seconds to abort a stalled recv/send. `query_timeout` bounds a single query.

**"the database system is starting up" / "too many connections".** These come
straight from PostgreSQL. Wait for recovery to finish, or raise
`max_connections` / use a pooler. Not a driver issue.

**Wrong database or "database does not exist".** A connection sees exactly one
database (no cross-catalog). Set `adbc.arrowfebe.database` (or the URI path
segment) to the right one; `SET search_path` selects the schema within it.

---

## TLS / certificates

**"certificate verify failed" / hostname mismatch.** With
`sslmode=verify-ca`/`verify-full` the server certificate chain (and, for
`verify-full`, the hostname) is verified. Point `ssl_root_cert` at the PEM root
CA that signed the server certificate, or fix the certificate. Do **not** drop to
`sslmode=require` in production to hide a bad certificate — `require` encrypts but
verifies nothing.

**Connection silently went plaintext.** The default `sslmode=prefer` falls back
to plaintext when the server answers the `SSLRequest` with `N`, and that reply is
unauthenticated (read before TLS). Over an untrusted network use
`sslmode=verify-full` and/or `channel_binding=require`. See
[`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/#tls--sslmode' | relative_url }}).

**`channel_binding=require` fails.** Channel binding needs a TLS channel — it is
unavailable under `sslmode=disable` and when GSS transport encryption is active.
Enable TLS (`sslmode=require`+), or drop `channel_binding` to `prefer`.

**Windows: driver loads but TLS fails / DLL not found.** OpenSSL is not part of
Windows. The matching `libssl-3-x64.dll` / `libcrypto-3-x64.dll` must sit next to
`arrowfebe_adbc_driver.dll` (or the driver must be static-linked). The MSVC
runtime (`vcruntime140.dll`, `msvcp140.dll`) must also be present.

---

## Authentication

**"password authentication failed for user '…'".** SCRAM/MD5/cleartext
credentials are wrong, or `pg_hba.conf` does not permit this user/host/method.
Confirm with `psql` first, and check the `pg_hba.conf` line that matches your
client address.

**"no pg_hba.conf entry for host … , SSL off".** The server requires TLS
(`hostssl`) for your address. Set `sslmode=require` (or `verify-full`).

**Integrated auth: GSSAPI/Kerberos errors ("Cannot find KDC", "Server not found
in Kerberos database").** The SPN did not match or you have no valid ticket. The
driver derives `postgres/<host>@REALM` (service name overridable with
`krbsrvname`, full SPN with `krb5.spn`). Ensure you have a TGT (`klist`), point at
the right ccache/keytab with `krb5.ccache` / `krb5.keytab`, and that the SPN is
registered. Run the `febe_krb_diag` CLI (built when GSSAPI is enabled) to test
credential acquisition and SPN resolution in isolation. Details in
[`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/' | relative_url }}).

**Integrated auth: "server did not complete mutual authentication".** Mutual auth
is always enforced — the driver refuses `AuthenticationOk` from a server that
never finished the GSS exchange. This is a real security signal, not a bug.

**`gssencmode=require` rejected with an `sslmode` conflict.** GSS transport
encryption skips TLS, so it cannot be combined with
`sslmode=require`/`verify-ca`/`verify-full`. Use one or the other.

**`auth_type=ActiveDirectory…` / a certificate auth mode is rejected.** ArrowFEBE
supports only `sql` and `integrated`/`sspi`; it is the driver and cannot delegate
Entra ID / certificate auth to a client library.

---

## Licence

**Connection fails with an `ARROW_LIC_*` error.** No valid Arpeio licence was
found. The gate is **always enforced** — there is no off switch. Supply a licence
via `arpeio.adbc.license_file` (a `.lic` path), `arpeio.adbc.license` (the blob
inline), the `ARPEIO_ADBC_LICENCE[_FILE]` environment variable, or an
`arpeio_adbc.lic` file next to the driver library. The installer's `--license`
flag places the file for you. Full detail — including how to read the current
licence state with `arpeio.adbc.license.status` — is in
[`LICENSING.md`]({{ '/drivers/arrowfebe/licensing/' | relative_url }}).

---

## Query & ingest

**Bulk ingest hangs or the transaction never commits.** Open the connection with
`autocommit=True` for the single-connection `TRUNCATE`+`adbc_ingest` pattern; a
manual transaction must be committed explicitly.

**`adbc_ingest(mode="create")` fails "already exists".** The target table exists;
use `append`, `replace`, or `create_append`.

**Ingesting `utf8` into a `jsonb` / `inet` / `uuid` column fails, but only for a
temp table.** The temp-table same-session fast path keeps Arrow-only type
inference (no OID transforms), so the server rejects a `utf8` bound into a
non-text column. Ingest into a persistent table (the OID-introspecting path), or
cast in SQL afterwards. See
[`DATA_TYPES.md`]({{ '/drivers/arrowfebe/data-types/#write-path-arrow--postgresql-ingest--bind' | relative_url }}).

**A geospatial column reads as opaque bytes I can't parse.** That is
`geospatial=binary` (raw EWKB, no extension metadata). Use the default
`geoarrow.wkb` (or `wkb`) mode for interoperable output. See
[`DATA_TYPES.md`]({{ '/drivers/arrowfebe/data-types/#geospatial-types' | relative_url }}).

**Cancelling a long query.** `AdbcConnectionCancel` / `AdbcStatementCancel` are
implemented via an out-of-band PostgreSQL `CancelRequest` — cancellation works.
If it does not take effect, the server may already have finished producing rows.

---

## Diagnostics: timing & debug traces

Set these environment variables to see where time goes. Full list in
[`ENV_VARS.md`]({{ '/drivers/arrowfebe/environment-variables/' | relative_url }}).

| Variable | What it shows |
|---|---|
| `ARPEIO_ADBC_READ_TIMING=1` | Per-batch read-path profiling on the SELECT stream — tells you whether the wire or the decoder is the bottleneck. |
| `ARPEIO_ADBC_WRITE_TIMING=1` | Per-batch ingest (COPY) profiling. |
| `ARPEIO_ADBC_ALLOCATOR=system` | Falls back from mimalloc to libc `malloc` for the Arrow column buffers (if mimalloc misbehaves). |

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/' | relative_url }}) · [`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/' | relative_url }}) · [`LICENSING.md`]({{ '/drivers/arrowfebe/licensing/' | relative_url }})
- `POSTGRES_TUNING.md` — throughput tuning
- [`COMPATIBILITY.md`]({{ '/drivers/arrowfebe/compatibility/' | relative_url }}) — is your target actually supported?
