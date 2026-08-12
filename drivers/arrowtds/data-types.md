---
title: Data types
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 3
permalink: /drivers/arrowtds/data-types/
---

# ArrowTDS — Data types

How SQL Server types map to Apache Arrow on the fetch (read) path, and how Arrow
types map back to SQL Server on the ingest/bind (write) path.

The non-obvious cases — `TINYINT` is unsigned, `MONEY` is a fixed-scale decimal,
`DATETIMEOFFSET` collapses to a UTC instant — are called out below.

---

## Read path: SQL Server → Arrow

### Numeric

| SQL Server type | Arrow type | Notes |
|---|---|---|
| `TINYINT` | `uint8` | SQL Server `TINYINT` is **unsigned** (0–255) |
| `SMALLINT` | `int16` | |
| `INT` | `int32` | |
| `BIGINT` | `int64` | |
| `BIT` | `bool` | bit-packed on finish |
| `REAL` | `float32` | |
| `FLOAT` | `float64` | |
| `DECIMAL` / `NUMERIC(p,s)` | `decimal128(p,s)` | precision/scale preserved |
| `MONEY` | `decimal128(19,4)` | fixed scale 4 |
| `SMALLMONEY` | `decimal128(10,4)` | fixed scale 4 |

### Character & binary

| SQL Server type | Arrow type | Notes |
|---|---|---|
| `CHAR` / `VARCHAR` / `VARCHAR(MAX)` | `utf8` | single-byte code page (CP1252/125x/874/OEM) transcoded via the column collation; UTF-8 collations pass through; sent as BIGVARCHAR on ingest |
| `NCHAR` / `NVARCHAR` / `NVARCHAR(MAX)` | `utf8` | UCS-2 on the wire, decoded to UTF-8 |
| `TEXT` / `NTEXT` (legacy LOB) | `utf8` | `TEXT` honours its code page; `NTEXT` is UCS-2 |
| `BINARY` / `VARBINARY` / `VARBINARY(MAX)` | `binary` | |
| `IMAGE` (legacy LOB) | `binary` | raw bytes |
| `XML` | `utf8` | serialized as text |
| `UNIQUEIDENTIFIER` | `utf8` | 36-char canonical form; lowercase by default, casing via [`uuid_casing`]({{ '/drivers/arrowtds/connection/' | relative_url }}#type-rendering-read-path) |
| `VECTOR(n)` (SQL Server 2025) | `fixed_size_list<float32>[n]` | Native float32 embeddings, both read and [ingest](#write-path-arrow--sql-server-ingest--bind). Requires SQL Server 2025 + the negotiated native-vector transport (auto-on; see [`ARROWTDS_VECTOR`]({{ '/drivers/arrowtds/environment-variables/' | relative_url }})). Without negotiation the server returns the vector as `varchar(max)` JSON (→ `utf8`). `float16` vectors are JSON-only over TDS. |

### Temporal

| SQL Server type | Arrow type | Notes |
|---|---|---|
| `DATE` | `date32` | days since epoch |
| `TIME(n)` | `time32[s\|ms]` or `time64[us\|ns]` | unit follows the column scale |
| `SMALLDATETIME` | `timestamp[s]` | |
| `DATETIME` | `timestamp[ms]` | 1/300 s wire resolution |
| `DATETIME2(n)` | `timestamp[s\|ms\|us\|ns]` | unit follows the column scale |
| `DATETIMEOFFSET(n)` | `timestamp[s\|ms\|us\|ns, tz=UTC]` | offset applied → UTC instant |

**Temporal scale → unit** (for `TIME` / `DATETIME2` / `DATETIMEOFFSET`):

| Column scale | Arrow unit |
|---|---|
| 0 | seconds |
| 1–3 | milliseconds |
| 4–6 | microseconds |
| 7 | nanoseconds |

SQL Server's scale-7 ceiling is 100 ns, so **true Arrow-nanosecond precision is
not reachable** — this is the source of the `*_ns` validation xfails.

### CLR user-defined types

| SQL Server type | Arrow type | Notes |
|---|---|---|
| `hierarchyid` | `binary` | opaque server bytes |
| `geometry` / `geography` | `geoarrow.wkb` (default) | client-side converted to WKB; see [Geospatial types](#geospatial-types) |

### Not yet mapped

- `sql_variant` — the COLMETADATA descriptor is not yet parsed (export-side gap);
  tracked as a data-type xfail.
- `float16` — no SQL Server type maps to Arrow `float16` (bind xfail).
- Arrow nanosecond temporal precision — see the scale-7 note above.

---

## Geospatial types

{: .since }
> Since ArrowTDS v0.5.12.

`geometry` and `geography` are CLR UDTs whose wire bytes are Microsoft's
proprietary serialization ([MS-SSCLRT]) — **not** WKB — so passing them through
unchanged is useless to Arrow / GeoParquet / DuckDB / GeoPandas consumers. By
default the driver converts them to the canonical `geoarrow.wkb` extension type.
The [`adbc.arrowtds.geospatial`]({{ '/drivers/arrowtds/connection/' | relative_url }}#type-rendering-read-path) database
option controls the read path:

| Value | Output | Notes |
|---|---|---|
| `geoarrow` (default) | Arrow `binary` + `geoarrow.wkb` extension | WKB plus `ARROW:extension:name=geoarrow.wkb` and an `ARROW:extension:metadata` JSON object; read directly by GeoPandas / GeoParquet / DuckDB |
| `wkb` | Arrow `binary` | ISO WKB (x=lon, y=lat), no extension metadata |
| `varbinary` (opt-in) | Arrow `binary` | raw opaque UDT bytes; backward compatible, **not** usable outside SQL Server |

**How it works.** The conversion is **client-side**. Your query runs verbatim —
nothing is rewritten and there is no extra round-trip. The row decoder recognises
`geometry`/`geography` columns from the result's wire metadata and converts each
value's native serialization to ISO WKB as rows stream in. The output is
byte-identical to the server's own `STAsBinary()` / `AsBinaryZM()`. NULL
geometries are preserved, and `ORDER BY` — or any other query shape — keeps the
conversion.

- **Z/M coordinates are preserved** as ISO WKB Z/M/ZM type codes
  (+1000/+2000/+3000), matching `AsBinaryZM()`.
- **Curved geometries** (`CircularString` / `CompoundCurve` / `CurvePolygon`)
  convert to ISO WKB curve types 8/9/10; **`geography` `FullGlobe`** becomes SQL
  Server's nonstandard WKB type 126 — both exactly as `STAsBinary()` renders them.
  Consumer support for curve WKB varies.

In `geoarrow` mode the extension metadata carries `crs` as `EPSG:<srid>` (read
from the first non-NULL value's SRID header; omitted for SRID 4326 / OGC:CRS84 and
for undefined SRIDs, per the GeoParquet convention) and, for `geography`,
`edges: spherical` (`geometry` is planar, so `edges` is omitted). The reader
decodes rows until every spatial column has an observed SRID — usually just the
first row — then fixes the schema.

**Known limitations:**

- **Mixed SRIDs within one column** are not auto-detected; the first observed SRID
  is used for the whole column's `crs`.
- The schema is fixed before rows are delivered, so a spatial column that is NULL
  for the whole first batch omits its `crs`. Order the query so a non-NULL value
  appears early, or raise `adbc.arrowtds.batch_size`.
- **Schema-only execution** (`Statement.ExecuteSchema`) reports spatial columns as
  plain `binary` — it runs under `FMTONLY`, so no rows and no SRID come back.
  Bound-parameter SELECTs (`WHERE id = ?`) *are* converted normally.
- The **write/ingest path** (Arrow WKB → `geometry`/`geography`) is a planned
  follow-up; only read/extract is implemented today.
- A malformed or unknown-version spatial value fails the read with a clear error
  naming the column; `geospatial=varbinary` reads the raw UDT bytes as an escape
  hatch.

[MS-SSCLRT]: https://learn.microsoft.com/en-us/openspecs/sql_server_protocols/ms-ssclrt/

---

## Write path: Arrow → SQL Server (ingest & bind)

On `adbc_ingest` and prepared-statement bind, Arrow arrays are routed to SQL
Server types:

- Integers, floats, `bool`, `decimal`, `date`, `time`, `timestamp` (with and
  without timezone), `binary`, and the view types (`string_view`, `binary_view`)
  are bound through TDS RPC.
- `utf8` columns are sent as **BIGVARCHAR** for UTF-8 collations (wire-efficient),
  or NVARCHAR otherwise.
- `XML` targets are routed via `NVARCHAR(MAX)` + a server-side cast (the raw XML
  wire type is rejected by `INSERT BULK`).
- PLP (MAX-type) ingest uses the unknown-length form (`UNKNOWN_PLP_LEN`).
- `fixed_size_list<float32>[n]` ingests to a SQL Server 2025 **`VECTOR(n)`** column
  (native binary; `mode=create` emits `VECTOR(n)` DDL). Only the float32 child is
  supported; requires the negotiated native-vector transport.

Bind inputs with **no** SQL Server equivalent return `NOT_IMPLEMENTED`/are
xfailed: `float16`, `null`-typed and dictionary-encoded parameters, and Arrow
nanosecond temporal precision (SQL Server tops out at 100 ns).

---

## Testing the mapping

Two suites exercise the types end-to-end:

- **Driver-owned type tests** — a pytest matrix over SQL Server-specific
  columns (`hierarchyid`, `geography`, `rowversion`, `money`, `uniqueidentifier`,
  `xml`, `datetime`/`smalldatetime`, the `(MAX)` LOBs, and `decimal128` precision
  boundaries).
- **Upstream ADBC validation** — the conformance suite; see
  [`COMPATIBILITY.md`]({{ '/drivers/arrowtds/compatibility/' | relative_url }}) for the
  per-case rationale of the remaining xfails.

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}#type-rendering-read-path) — the `uuid_casing` and `geospatial` options
- [`COMPATIBILITY.md`]({{ '/drivers/arrowtds/compatibility/' | relative_url }}) — validated SQL Server versions
- [`EXAMPLES.md`]({{ '/drivers/arrowtds/examples/' | relative_url }}) — reading and ingesting typed data
