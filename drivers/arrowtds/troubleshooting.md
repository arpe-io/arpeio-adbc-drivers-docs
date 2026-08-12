---
title: Troubleshooting
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 8
permalink: /drivers/arrowtds/troubleshooting/
---

# ArrowTDS — Troubleshooting

Common failures and their fixes, grouped by where they show up: connecting,
authenticating, the licence gate, and querying/ingesting. For the option details
referenced here, see [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}).

- [Connection](#connection)
- [TLS / certificates](#tls--certificates)
- [Authentication](#authentication)
- [Licence](#licence)
- [Query & ingest](#query--ingest)
- [Diagnostics: timing & debug traces](#diagnostics-timing--debug-traces)

---

## Connection

**Connect hangs, then fails after ~30 s.** The TCP connect budget
(`login_timeout`, default 30 s) elapsed. When a host resolves to several addresses
they *share* that budget, so this usually means every address is unreachable — a
firewall, a wrong port, or the server not listening. Verify with
`nc -vz <host> 1433`. Lower `login_timeout` to fail faster while diagnosing.

**A machine name is much slower than `localhost` / an IP.** The name resolved to
several addresses and a leading one (often a link-local `fe80::` on Windows) is
dropping the SYN. The shared connect budget prevents a flat +30 s stall, but a
dead first address still costs time. Connect by IP, or fix DNS/`hosts` ordering.

**Named instance won't connect ("could not determine port for instance …").**
`Server=host\INSTANCE` needs the **SQL Server Browser** reachable on **UDP 1434**.
Either open UDP 1434 and start the Browser service, or pin the instance to a static
port in SQL Server Configuration Manager and connect with an explicit port
(`Server=host\INSTANCE,54312`), which skips the Browser lookup entirely. See
[`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}#named-instances).

**`(localdb)\...` fails.** LocalDB is reached over a named pipe, not TCP — this
driver cannot connect to it. Use a full SQL Server (Express is fine).

**Azure SQL "Redirect" connection resets.** The Redirect gateway policy sends the
client to a database node on ports **11000–11999**; the driver follows the routing
token automatically, but the reconnect needs **outbound TCP to 11000–11999**. Open
that range, or ask your admin to set the server's connection policy to **Proxy**.

**Azure SQL / Fabric: "Server name cannot be determined".** The gateway routes on
the LOGIN7 server name. This was a driver bug fixed in 0.5.20 — upgrade to
≥ 0.5.20.

---

## TLS / certificates

**"certificate verify failed" / hostname mismatch.** With `encrypt=true` the
server certificate and hostname are verified by default. For a self-signed dev
server, add `trust_server_cert=true`. **Do not** use `trust_server_cert=true` in
production — fix the certificate instead. For an IP-literal host, the cert must
carry a matching `iPAddress` SAN.

**Azure SQL rejects an unencrypted connection.** Azure requires TLS — set
`encrypt=true` (the default-safe path handles SNI + verification).

**Windows: driver loads but TLS fails / DLL not found.** OpenSSL is not part of
Windows. The matching `libssl-3-x64.dll` / `libcrypto-3-x64.dll` must sit next to
`arrowtds_adbc_driver.dll` (or the driver must be static-linked). The VC++ runtime
(`vcruntime140.dll`, `msvcp140.dll`) must also be present.

---

## Authentication

**Login failed for user '…'.** SQL login credentials are wrong, or the login is
disabled / lacks access to the database. Confirm with `sqlcmd`/SSMS first.

**Integrated auth: "Cannot generate SSPI context" / GSSAPI errors.** The Kerberos
SPN did not match. The driver derives `MSSQLSvc/host:<resolved-port>` — for a
named instance the port is the *resolved* one. Ensure the SPN is registered for
the service account, that you have a valid TGT (`klist`), and on Linux point at the
right credential cache/keytab with `krb5.ccache` / `krb5.keytab`. Override the SPN
with `krb5.spn` if needed. Details in [`AUTHENTICATION.md`]({{ '/drivers/arrowtds/authentication/' | relative_url }}).

**Entra ID: "Federated auth cannot be combined with a password."** Remove
`username`/`password` — federated (token) auth is exclusive and always uses full
TLS. Also ensure `encrypt=true`.

**Entra ID: token acquired but "login failed".** The identity (user, service
principal, or managed identity) must be a **database user**:
`CREATE USER [<name>] FROM EXTERNAL PROVIDER`. A valid token for a principal with
no database mapping still fails login.

**`auth_type=ActiveDirectory…` rejected.** Only `ActiveDirectoryDefault` (the
credential chain) is a valid `auth_type`. For the other Entra modes use the
dedicated options: `access_token`, `tenant_id`+`client_id`+`client_secret`, or
`managed_identity`. See [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}#authentication).

---

## Licence

**Connection fails with an `ARROW_LIC_*` error.** No valid Arpeio licence was
found. Supply one via `arpeio.adbc.license_file` (a `.lic` path),
`arpeio.adbc.license` (the blob inline), the `ARPEIO_ADBC_LICENCE[_FILE]`
environment variable, or an `arpeio_adbc.lic` file next to the driver library. The
installer's `--license` flag places the file for you. Full detail — including how
to read the current licence state with `arpeio.adbc.license.status` — is in
[`LICENSING.md`]({{ '/drivers/arrowtds/licensing/' | relative_url }}).

---

## Query & ingest

**Bulk ingest deadlocks or hangs.** The DB-API default `autocommit=False` turns on
`IMPLICIT_TRANSACTIONS`, which can deadlock the single-connection
`TRUNCATE`+`INSERT BULK` pattern. Open the connection with `autocommit=True` for
ingest. See [`EXAMPLES.md`]({{ '/drivers/arrowtds/examples/' | relative_url }}#bulk-ingest-an-arrow-table).

**`adbc_ingest(mode="create")` fails "already exists".** The target table exists;
the driver returns `ALREADY_EXISTS` (SQL Server error 2714). Use `append`,
`replace`, or `create_append`.

**Ingesting XML fails "Invalid column type from bcp client".** `INSERT BULK` does
not accept the XML wire type; the driver routes XML via `NVARCHAR(MAX)` + a cast
automatically — if you hit this on an older build, upgrade.

**A geospatial column reads as opaque bytes I can't parse.** That is
`geospatial=varbinary` (raw CLR-UDT bytes). Use the default `geoarrow` (or `wkb`)
mode for interoperable output. See [`DATA_TYPES.md`]({{ '/drivers/arrowtds/data-types/' | relative_url }}#geospatial-types).

**Cancelling a long query does nothing.** `Connection.Cancel` / `Statement.Cancel`
return `NOT_IMPLEMENTED` today — close the statement or connection to abort an
in-flight query.

---

## Diagnostics: timing & debug traces

Set these environment variables to see where time goes or to trace the wire. Full
list in [`ENV_VARS.md`]({{ '/drivers/arrowtds/environment-variables/' | relative_url }}).

| Variable | What it shows |
|---|---|
| `ARPEIO_ADBC_READ_TIMING=1` | Per-batch `recv_ms` / `decode_ms` / `writer_wait_ms` on the SELECT stream — tells you whether the wire, the decoder, or downstream backpressure is the bottleneck. |
| `ARROWTDS_TDS_READ_DEBUG=1` | Verbose per-token TDS read trace (very chatty; wire-protocol bugs only). |
| `ARPEIO_ADBC_QUIET=1` | Silences the ingest `[TDS TIMING]` summary and downgrade/row-count warnings. |
| `ARPEIO_ADBC_ALLOCATOR=system` | Falls back from mimalloc to libc `malloc` for decode buffers (if mimalloc misbehaves). |

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}) · [`AUTHENTICATION.md`]({{ '/drivers/arrowtds/authentication/' | relative_url }}) · [`LICENSING.md`]({{ '/drivers/arrowtds/licensing/' | relative_url }})
- `SQLSERVER_TUNING.md` — throughput tuning
- [`COMPATIBILITY.md`]({{ '/drivers/arrowtds/compatibility/' | relative_url }}) — is your target actually supported?
