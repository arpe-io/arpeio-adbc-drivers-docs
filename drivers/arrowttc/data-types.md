---
title: Data types
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 3
permalink: /drivers/arrowttc/data-types/
---

# ArrowTTC — Data types

How Oracle types map to Apache Arrow on the fetch (read) path, and how Arrow
types map back to Oracle on the ingest/bind (write) path.

The non-obvious cases — unconstrained `NUMBER` has no exact Arrow type, `DATE`
carries a time component (so it is a timestamp, not `date32`), `TIMESTAMP WITH
TIME ZONE` collapses to a UTC instant — are called out below.

---

## Read path: Oracle → Arrow

### Numeric

| Oracle type | Arrow type | Notes |
|---|---|---|
| `NUMBER(p,0)`, p ≤ 18 | `int64` | Max 999,999,999,999,999,999 < 2⁶³; better for every consumer than a zero-scale decimal |
| `NUMBER(p,0)`, p 19–38 | `decimal128(p,0)` | Can exceed int64 |
| `NUMBER(p,s)`, s > 0 | `decimal128(p,s)` | Exact; Oracle's max precision is 38, so `decimal256` is never needed |
| `NUMBER` (unconstrained) | `utf8` (default) / `float64` / `decimal128(P,S)` | Governed by [`number_mapping`](#number-mapping) |
| `BINARY_INTEGER` | as `NUMBER` above | Same policy path |
| `BINARY_FLOAT` | `float32` | |
| `BINARY_DOUBLE` | `float64` | |

### Character & binary

| Oracle type | Arrow type | Notes |
|---|---|---|
| `VARCHAR2` / `CHAR` / `LONG` | `utf8` | AL32UTF8 passes through unchanged |
| `NVARCHAR2` / `NCHAR` | `utf8` | National charset (AL16UTF16) transcoded UTF-16 → UTF-8; distinguished from `VARCHAR2` only by the character-set form (`csfrm`) |
| `RAW` / `LONG RAW` | `binary` | |
| `ROWID` / `UROWID` | `utf8` | rendered as text |
| `CLOB` | `utf8` | fetched via a LOB locator round-trip after the row |
| `NCLOB` | `utf8` | national CLOB, UTF-16 → UTF-8 |
| `BLOB` | `binary` | LOB locator round-trip |

### Temporal

| Oracle type | Arrow type | Notes |
|---|---|---|
| `DATE` | `timestamp[s]` | **Not `date32`** — Oracle `DATE` always carries a time component |
| `TIMESTAMP(n)` | `timestamp[s\|ms\|us\|ns]` | unit follows the column's fractional-seconds precision |
| `TIMESTAMP(n) WITH TIME ZONE` | `timestamp[…, tz=UTC]` | offset applied → UTC instant |
| `TIMESTAMP(n) WITH LOCAL TIME ZONE` | `timestamp[…, tz=UTC]` | wire value is UTC (`DBTIMEZONE` assumed UTC) |
| `INTERVAL YEAR TO MONTH` | `interval (month_day_nano)` | |
| `INTERVAL DAY TO SECOND` | `interval (month_day_nano)` | |

**Fractional-seconds precision → unit** (for `TIMESTAMP` and its TZ variants):

| Column precision | Arrow unit |
|---|---|
| 0 | seconds |
| 1–3 | milliseconds |
| 4–6 | microseconds |
| 7–9 | nanoseconds |

Oracle allows up to 9 fractional digits, so the precision is rounded **up** to
the next representable Arrow unit and no precision is lost.

### Not yet mapped

- **`BFILE`** — a pointer to a server-filesystem file through a `DIRECTORY`
  object; a different permission model, not a different encoding, so it is
  surfaced as unsupported rather than half-decoded.
- **Region-id `TIMESTAMP WITH TIME ZONE`** (a named zone rather than a fixed
  offset) fails loudly rather than resolving the zone through DST rules;
  fixed-offset TSTZ and TSLTZ decode normally.
- **`JSON`, `BOOLEAN`** — Oracle 19c has no native JSON or boolean *column* type
  (JSON is stored in a LOB/VARCHAR2 with an `IS JSON` check; 23ai adds native
  types, which are covered under the 21c/23ai describe work — see
  [`COMPATIBILITY.md`]({{ '/drivers/arrowttc/compatibility/' | relative_url }})). The describe flags an `IS JSON` column
  but it decodes as its underlying text/LOB type.
- Unknown Oracle types are surfaced as opaque `binary` rather than guessed at.

---

## `Number Mapping`

Oracle's unconstrained `NUMBER` (no declared precision or scale — describe reports
scale −127) is Oracle's *default* numeric type and has no exact Arrow equivalent.
A fixed `decimal128(p,s)` cannot be chosen safely (it would truncate high-scale
values and overflow large ones), and an Arrow schema is fixed before the first
batch, so the scale cannot be inferred by looking ahead in the stream. The
[`adbc.arrowttc.number_mapping`]({{ '/drivers/arrowttc/connection/' | relative_url }}#type-rendering-read-path) database
option chooses:

| Value | Output | Notes |
|---|---|---|
| `auto` (default) | `utf8` | Lossless. The deliberate default — a lossy default would silently lose precision on Oracle's most common numeric column |
| `double` | `float64` | Fast, lossy — 15–17 significant digits against Oracle's 38 |
| `decimal:P,S` | `decimal128(P,S)` | For callers who know their data; `1 ≤ P ≤ 38`, `0 ≤ S ≤ P` |

Constrained columns are unaffected by this option — only unconstrained `NUMBER`
is routed through it. The lossless default is not theoretical: the type-matrix
fixture's unconstrained column round-trips **21 significant digits**, well beyond
`float64`.

---

## Write path: Arrow → Oracle (ingest & bind)

On `adbc_ingest` and prepared-statement bind, Arrow arrays drive positional
binds — one Arrow row is one execution. Ingest issues an array-bound `INSERT`
with a SQL-injection-safe identifier and literal escaper, in `create` / `append`
/ `replace` / `create_append` modes, with optional `target_db_schema` and a
`GLOBAL TEMPORARY` target (`adbc.ingest.temporary`). See
[`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}#statement--ingest-options) for the ingest options.

---

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}#type-rendering-read-path) — the `number_mapping` option
- [`COMPATIBILITY.md`]({{ '/drivers/arrowttc/compatibility/' | relative_url }}) — validated Oracle versions
- [`EXAMPLES.md`]({{ '/drivers/arrowttc/examples/' | relative_url }}) — reading and ingesting typed data
