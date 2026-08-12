---
title: What's new
layout: default
parent: ArrowFEBE
grand_parent: Drivers
nav_order: 9
permalink: /drivers/arrowfebe/whats-new/
---

# What's new in ArrowFEBE

Feature-oriented release highlights for ArrowFEBE — the pure-native PostgreSQL
ADBC driver that speaks the Frontend/Backend (FEBE) protocol v3 directly and
streams Apache Arrow end-to-end (no libpq). This is the reader-friendly view; the
full engineering detail (every fix, every internal change) lives in
`CHANGELOG.md`.

Get it in one line — see the [install guide]({{ '/install/' | relative_url }}):

```bash
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh \
  | sh -s -- arrowfebe --license ./arpeio_adbc.lic
```

---

## Hardened release binaries — August 2026 (v0.3.8)

The published Linux and Windows binaries are now stripped of internal symbols,
toolchain strings, and build-machine paths, and every shipped Linux artifact is
gated in CI on being hardened — it must export only the two ADBC entrypoints and
carry no build-id or build-machine paths in its data. A supply-chain hardening
pass with no runtime cost.

## Refuse plaintext passwords & a safer cancel — August 2026 (v0.3.7)

A new opt-in `require_password_encryption` option (ADO.NET
`RequirePasswordEncryption`, URI `require_password_encryption`) makes the driver
refuse to send a cleartext or MD5 password when the transport is neither TLS- nor
GSS-encrypted, closing a MITM downgrade that coaxes the credential out of the
client. Off by default (libpq-compatible). This release also fixes a cross-thread
use-after-free in `AdbcConnectionCancel` / `AdbcStatementCancel` — cancel now
builds its request from a snapshot of the backend cancel key and never touches
the freeable session — and completes the ADBC 1.1.0 statistics surface with
`GetStatisticNames`.

## Portable connection URIs — August 2026 (v0.3.6)

Connect with a standard libpq-style PostgreSQL URI on the ADBC `uri` option:

```
postgresql://alice:secret@db.example:5432/tpch?sslmode=require
```

Schemes `postgresql://`, `postgres://`, and the branded `arrowfebe://` are
recognised; the path segment is the database name (as in libpq), values are
percent-decoded, and bracketed IPv6 hosts work. Query parameters (`user`,
`password`, `dbname`, `application_name`, `connect_timeout`, `sslmode`,
`channel_binding`, `sslrootcert`, `gssencmode`, `krbsrvname`) map onto the
existing options, and a discrete `adbc.arrowfebe.*` option still wins over a URI
value. So strings copied from psql or a `DATABASE_URL` just work. See
[Connection → URI]({{ '/drivers/arrowfebe/connection/#connection-uri' | relative_url }}).

## A standalone driver you install by name — August 2026 (v0.3.5)

ArrowFEBE is now an independently installable ADBC driver, not just an ingredient
of other tools:

- **One-line install** of a prebuilt, checksum-verified binary via
  `curl … | install.sh arrowfebe`.
- **Load by name** — any ADBC client connects with `driver="arrowfebe"` via the
  installed ADBC driver manifest; no library paths to wire up.
- **Faster ingest** — a reworked COPY-binary encoder made bulk ingest ~2.2×
  faster with client CPU roughly halved at every parallel degree.

## Shared family environment & always-on licence — August 2026 (v0.3.6)

The diagnostic env-var knobs moved to the shared `ARPEIO_ADBC_*` prefix
(`ARPEIO_ADBC_READ_TIMING`, `ARPEIO_ADBC_WRITE_TIMING`, `ARPEIO_ADBC_ALLOCATOR`),
so one variable configures every Arpeio driver and traces compare directly across
them. The licence gate is now always enforced in every build — a driver lifted
out of a host bundle is inert on its own. Exported symbols are restricted to the
ADBC entrypoints. See [`docs/ENV_VARS.md`]({{ '/drivers/arrowfebe/environment-variables/' | relative_url }}) and
[`docs/LICENSING.md`]({{ '/drivers/arrowfebe/licensing/' | relative_url }}).

## Real query cancellation & a self-contained Windows DLL — June 2026 (v0.2.2)

`AdbcConnectionCancel` / `AdbcStatementCancel` now abort an in-flight query via an
out-of-band PostgreSQL `CancelRequest`. The Windows build ships as a
self-contained DLL, and `application_name` is reported in the startup packet
(`pg_stat_activity`).

## Integrated & Kerberos authentication — June 2026 (v0.2.0)

Trusted/integrated auth with no username or password: **POSIX Kerberos/GSSAPI**
(auth codes 7/8) on Linux/macOS and **Windows SSPI** (Negotiate:
Kerberos with NTLM fallback), plus **GSS transport encryption** (`gssencmode`) for
callers who want an encrypted stream without TLS. Mutual authentication is always
enforced. See [Authentication]({{ '/drivers/arrowfebe/authentication/' | relative_url }}).

## PostGIS geospatial — June 2026 (v0.2.4)

PostGIS `geometry` / `geography` columns read as interoperable **GeoArrow / WKB**:
their EWKB wire bytes are already a valid `geoarrow.wkb` encoding, so they pass
straight through as Arrow `binary` with no server-side `ST_AsBinary` round-trip.
The `geospatial` option (`geoarrow.wkb` / `wkb` / `binary`) controls the extension
metadata — `edges: spherical` for geography, `crs: EPSG:<srid>` from the column's
typmod. Clean no-op when PostGIS is absent. See
[Data types → Geospatial]({{ '/drivers/arrowfebe/data-types/#geospatial-types' | relative_url }}).

## SCRAM channel binding — June 2026 (v0.1.4)

SCRAM-SHA-256-**PLUS** channel binding (`channel_binding=require`) binds the auth
exchange to the server certificate over TLS, defeating a MITM that proxies the
SASL flow — even without full certificate verification. Configurable I/O timeouts
and in-memory secret scrubbing landed in the same review pass.

## Native NUMERIC, interval & richer types — June 2026 (v0.1.5)

`numeric` decodes straight from PostgreSQL's base-10000 binary layout into a
`decimal128`/`decimal256` with no string detour — the headline differentiator over
the upstream Apache PostgreSQL driver. Full write→read round-trips landed for
`interval` (⇄ Arrow `month_day_nano`), `json`/`jsonb`, `inet`/`cidr`/`macaddr`,
and `bit`/`varbit` (⇄ Arrow `utf8`), driven by pre-COPY `pg_attribute`
introspection of the target table. See [Data types]({{ '/drivers/arrowfebe/data-types/' | relative_url }}).

## The native FEBE engine — foundational (v0.1.0)

From the start, ArrowFEBE speaks the PostgreSQL Frontend/Backend protocol v3
directly over TCP/TLS — no libpq, no ODBC — with OpenSSL the only runtime
dependency beyond libc:

- **COPY-binary fast path** — results stream as Arrow `RecordBatch`es built
  directly from the binary wire format, measured locally at ~1.70× the upstream
  Apache PostgreSQL driver on a TPC-H SF10 `lineitem` extract.
- **Native bulk ingest** — pipe an Arrow stream straight into COPY FROM STDIN
  (binary), with `create` / `append` / `replace` / `create_append` modes and
  temp-table support.
- **Full ADBC 1.1.0 surface** — prepared statements + bind, catalog browsing
  (`GetObjects` with PK/FK/UNIQUE/CHECK), `GetStatistics` / `GetStatisticNames`,
  `GetTableSchema`,
  commit/rollback, and password-family auth (SCRAM-SHA-256 / MD5 / cleartext)
  with TLS via the `SSLRequest` handshake.

---

For the complete, dated, technical history see `CHANGELOG.md`. To
report an issue or request a licence, contact **sales@arpe.io** or visit
**https://www.arpe.io**.
