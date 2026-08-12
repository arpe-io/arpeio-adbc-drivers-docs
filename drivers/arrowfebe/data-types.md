---
title: Data types
layout: default
parent: ArrowFEBE
grand_parent: Drivers
nav_order: 3
permalink: /drivers/arrowfebe/data-types/
---

# ArrowFEBE — Data types

How PostgreSQL types map to Apache Arrow on the fetch (read) path, and how Arrow
types map back to PostgreSQL on the ingest/bind (write) path.

Both directions use the **COPY-binary** fast path — read via COPY TO / the binary
result format, write via COPY FROM STDIN (binary) — so there is no text round-trip
for the common types. The headline case is `NUMERIC`, decoded straight from
PostgreSQL's base-10000 binary layout into a `decimal128`/`decimal256` with no
string detour.

---

## Read path: PostgreSQL → Arrow

### Numeric

| PostgreSQL type | Arrow type | Notes |
|---|---|---|
| `bool` | `bool` | |
| `int2` | `int16` | |
| `int4` | `int32` | |
| `int8` | `int64` | |
| `float4` | `float32` | |
| `float8` | `float64` | |
| `numeric(p,s)` | `decimal128(p,s)` (p ≤ 38) · `decimal256` (39–76) · `utf8` (> 76 / unconstrained) | native base-10000 binary decode, no text round-trip |

An unconstrained `numeric` (no declared precision) has no fixed Arrow decimal
width, so it renders as `utf8` (canonical PostgreSQL text).

### Character & binary

| PostgreSQL type | Arrow type | Notes |
|---|---|---|
| `text` / `varchar` / `bpchar` (`char(n)`) | `utf8` | |
| `uuid` | `utf8` | 36-char canonical form; lowercase by default, casing via [`uuid_casing`]({{ '/drivers/arrowfebe/connection/#type-rendering-read-path' | relative_url }}) |
| `json` / `jsonb` | `utf8` | canonical PostgreSQL text |
| `inet` / `cidr` / `macaddr` | `utf8` | canonical PostgreSQL text |
| `bit` / `varbit` | `utf8` | canonical PostgreSQL text |
| `xml` | `utf8` | text export |
| `bytea` | `binary` | raw bytes |
| unknown / exotic OIDs (arrays, ranges, enums, …) | `binary` | raw wire bytes |

### Temporal

| PostgreSQL type | Arrow type | Notes |
|---|---|---|
| `date` | `date32` | days since epoch |
| `time(p)` | `time32[s\|ms]` or `time64[us]` | unit follows the column precision |
| `timetz` | `utf8` | text export (Arrow has no time-with-offset type) |
| `timestamp(p)` | `timestamp[s\|ms\|us]` | unit follows the column precision |
| `timestamptz(p)` | `timestamp[s\|ms\|us, tz=UTC]` | stored UTC instant |
| `interval` | `month_day_nano` | Arrow's month/day/nanosecond interval |

PostgreSQL's microsecond storage resolution means **true Arrow-nanosecond
precision is not reachable** — this is the source of the `*_ns` validation
xfails.

### Geospatial (PostGIS)

| PostgreSQL type | Arrow type | Notes |
|---|---|---|
| `geometry` / `geography` | `binary` + `geoarrow.wkb` extension (default) | EWKB passes straight through; see [Geospatial types](#geospatial-types) |

---

## Geospatial types

{: .since }
> Since ArrowFEBE v0.2.4.

PostGIS `geometry` / `geography` columns are read over the COPY-binary path as
**EWKB** — which is already a valid [GeoArrow](https://geoarrow.org/)
`geoarrow.wkb` encoding, so the bytes pass straight through as Arrow `binary`
with **no server-side `ST_AsBinary` round-trip**. The
[`adbc.arrowfebe.geospatial`]({{ '/drivers/arrowfebe/connection/#type-rendering-read-path' | relative_url }}) database
option controls the Arrow field metadata:

| Value | Output | Notes |
|---|---|---|
| `geoarrow.wkb` (default) | Arrow `binary` + `geoarrow.wkb` extension | EWKB plus `ARROW:extension:name=geoarrow.wkb` and an `ARROW:extension:metadata` JSON object; read directly by GeoPandas / GeoParquet / DuckDB |
| `wkb` | Arrow `binary` | EWKB bytes, no extension metadata |
| `binary` | Arrow `binary` | opaque binary, no extension metadata |

**How it works.** The geometry/geography type OIDs are resolved per session from
`pg_type`; when PostGIS is not installed this is a clean no-op. In `geoarrow.wkb`
mode the extension metadata carries `edges: spherical` for `geography` (planar
`geometry` omits it) and `crs: EPSG:<srid>` taken from the column's PostGIS
typmod. CRS is emitted only for a typmod-constrained column with an SRID other
than 0/4326 (mirroring the GeoParquet convention); a plain `geometry` column with
mixed/unconstrained SRID omits it. The SRID embedded in each EWKB value is
preserved regardless of the metadata.

---

## Write path: Arrow → PostgreSQL (ingest & bind)

On `adbc_ingest` (COPY FROM STDIN, binary) and prepared-statement bind (extended
protocol), Arrow arrays are routed to PostgreSQL types.

Bulk ingest **introspects the target table's column OIDs**
(`pg_catalog.pg_attribute`) before writing, so Arrow columns land in the right
receive format even when the source Arrow type is a generic `utf8`:

- Integers, floats, `bool`, `decimal`, `date`, `time`, `timestamp` (with and
  without timezone), and `binary` map directly.
- Arrow `utf8` lands correctly in `json`, `jsonb`, `inet`, `cidr`, `macaddr`,
  `bit`/`varbit`, and `uuid` columns (the encoder transforms the text into each
  type's binary receive format).
- Arrow `month_day_nano` lands in `interval` columns.

Both the array-bind and stream-bind ingest paths introspect (since v0.2.1 the
stream-bind path streams its input one batch at a time *and* applies the OID map).
The **temp-table same-session fast path** is the one exception: it keeps
Arrow-only type inference (no OID transforms), so a `utf8` bound into a
`jsonb`/`inet`/… column through it is rejected by the server.

Arrow inputs with **no** PostgreSQL equivalent, and Arrow nanosecond temporal
precision (PostgreSQL tops out at microseconds), are `NOT_IMPLEMENTED`/xfailed.

---

## Testing the mapping

Two suites exercise the types end-to-end:

- **Type matrix** — a driver-owned pytest matrix over PostgreSQL-specific
  columns (`numeric` precision boundaries, `interval`, `json`/`jsonb`, `inet` /
  `cidr` / `macaddr`, `bit`/`varbit`, `uuid`, `timetz`, `xml`, PostGIS
  `geometry`/`geography`), including write→read round-trips and export-only
  cases (arrays, ranges, enum, `money`, `point`, `tsvector`, `pg_lsn`, …).
- **Upstream ADBC validation** — the conformance suite; see
  [`COMPATIBILITY.md`]({{ '/drivers/arrowfebe/compatibility/' | relative_url }}) for the
  per-case rationale of the remaining xfails.

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/#type-rendering-read-path' | relative_url }}) — the `uuid_casing` and `geospatial` options
- [`COMPATIBILITY.md`]({{ '/drivers/arrowfebe/compatibility/' | relative_url }}) — validated PostgreSQL versions
- [`EXAMPLES.md`]({{ '/drivers/arrowfebe/examples/' | relative_url }}) — reading and ingesting typed data
