---
title: What's new
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 9
permalink: /drivers/arrowtds/whats-new/
---

# What's new in ArrowTDS

Feature-oriented release highlights for ArrowTDS — the pure-native SQL Server ADBC
driver that streams Apache Arrow end-to-end. This is the reader-friendly view; the
full engineering detail (every fix, every internal change) lives in `CHANGELOG.md`.

Get it in one line — see the [install guide]({{ '/install/' | relative_url }}):

```bash
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh \
  | sh -s -- arrowtds --license ./arpeio_adbc.lic
```

---

## Hardened, reproducible release builds — August 2026 (v0.5.24)

A supply-chain and artifact-hygiene pass over the release build, with **no driver
behavior changes** — the wire protocol and the output bytes for `SELECT` and
Parquet export are byte-for-byte identical to the previous release. The shipped
`.so`/`.dll` no longer embed the build machine's source paths, the toolchain
identity, or the internal symbol table, and the release pipeline now fails closed
if any of that leaks back in. Local Release builds stay fully debuggable.

## Sturdier sessions and stronger secret hygiene — August 2026 (v0.5.23)

A security- and robustness-hardening pass over the wire-parsing, authentication,
and bulk-ingest paths:

- **Secrets are scrubbed from memory** — Entra bearer/access tokens and a
  `sqlserver://user:secret@host` URI password are now zeroed before their buffers
  are freed, closing gaps where they could linger in freed memory.
- **A multi-statement query no longer returns a half-read session to the pool** —
  `SELECT a; SELECT b` (or an `EXEC` yielding several result sets) used to leave
  trailing rows on the socket that the next query could read as its own reply. The
  reader now drains cleanly, ending the cross-query confusion and protocol desync.
- **Malformed wire metadata and bulk input are rejected instead of mis-parsed**,
  and an all-valid column now skips validity-bitmap work entirely (a read-path
  speed-up with identical output).

## SQL Server 2025 `VECTOR` type — August 2026 (v0.5.22)

Round-trip support for the SQL Server 2025 `VECTOR(n)` type. The driver negotiates
a native vector transport at login and reads `VECTOR(n)` columns as Arrow
`fixed_size_list<float32>[n]` — for both `SELECT` and Parquet export — and ingests
the same Arrow shape straight back into a `VECTOR(n)` target, with no server-side
cast to `varbinary` or JSON. Validated at conformance parity with SQL Server 2022.
See [Data types]({{ '/drivers/arrowtds/data-types/' | relative_url }}).

## Microsoft Entra ID authentication — August 2026

Connect to **Azure SQL Database** and **Microsoft Fabric** with Microsoft Entra ID
(Azure AD), no passwords. Four ways to authenticate, all first-class:

- **Bring your own token** — pass a bearer token you already hold (e.g. from
  `az account get-access-token`).
- **Service principal** — give the driver a tenant/client id and secret and it
  acquires the token for you.
- **Managed identity** — running on an Azure VM, App Service, or Container App? The
  driver gets a token from the host, no secret at all.
- **Default credential chain** — one setting (`ActiveDirectoryDefault`) that runs
  unchanged on your laptop, in CI, and in production, picking the first credential
  that works.

All four were verified against live Azure SQL. See
[Connection → Entra ID]({{ '/drivers/arrowtds/connection/' | relative_url }}#azure-sql-microsoft-fabric--entra-id).

## Azure SQL "Redirect" policy — August 2026

Azure SQL and Fabric can route you from the gateway straight to the database node
(the **Redirect** connection policy). ArrowTDS now follows that redirect
transparently and reconnects to the right node — previously only the **Proxy**
policy worked. (Redirect needs outbound TCP to ports 11000–11999.)

## A standalone driver you install by name — August 2026 (v0.5.19)

ArrowTDS is now an independently installable ADBC driver, not just an ingredient of
other tools:

- **One-line install** of a prebuilt, checksum-verified binary via
  `curl … | install.sh arrowtds`.
- **Load by name** — any ADBC client connects with `driver="arrowtds"`; no
  library paths to wire up.
- **Portable connection URIs** — `sqlserver://…`, `mssql://…`, or the branded
  `arrowtds://…`, matching the official SQL Server tooling, so strings copied from
  elsewhere just work.
- Standard transaction **isolation-level** and read-only connection options.

## Faster, more reliable connections — July 2026 (v0.5.18)

A field report of a machine name connecting ~3× slower than `localhost` led to a
reworked connect path: when a host resolves to several addresses they now share one
timeout budget, so a single dead address (a stray link-local `fe80::` entry on
Windows, say) can no longer stall the connection.

## Faster large extracts — July 2026 (v0.5.15)

A decode-path rework removed a scaling bottleneck under high parallelism, cutting
end-to-end extract time dramatically on big result sets — part of an overall
~4.6× speed-up on the reference SQL Server extract benchmark.

## Native geospatial, decimal, and LOB types — 2026

`geometry` and `geography` columns come back as interoperable **GeoArrow / WKB**,
converted client-side with Z/M coordinates and curved geometries preserved — ready
for GeoParquet, DuckDB, and GeoPandas with no server-side rewrite. `text`, `ntext`,
and `image` LOBs, `money`/`smallmoney`, `uniqueidentifier`, and `xml` all decode
natively. See [Data types]({{ '/drivers/arrowtds/data-types/' | relative_url }}).

## Integrated & Kerberos authentication — May 2026 (v0.4.5)

Trusted/integrated auth with no username or password: **Windows SSPI** (Negotiate →
Kerberos/NTLM) and **POSIX Kerberos** (GSSAPI/SPNEGO) with ccache, keytab, or
password credentials. See [Authentication]({{ '/drivers/arrowtds/authentication/' | relative_url }}).

## The native TDS engine — foundational

From the start, ArrowTDS speaks MS-TDS directly over TCP/TLS — no ODBC, no
FreeTDS — with OpenSSL the only runtime dependency beyond libc:

- **Native bulk ingest** — pipe an Arrow stream straight into `INSERT BULK`
  (~665k rows/sec on TPC-H `lineitem`).
- **Streaming Arrow fetch** — results arrive as Arrow `RecordBatch`es as the
  server produces them, with an auto-vectorised decode hot loop (~1.16M rows/sec on
  SF1 `lineitem`).
- **Full ADBC 1.1.0 surface** — prepared statements + bind, catalog browsing,
  statistics, `ExecuteSchema`, commit/rollback.

---

For the complete, dated, technical history see `CHANGELOG.md`. To
report an issue or request a licence, contact **sales@arpe.io** or visit
**https://www.arpe.io**.
