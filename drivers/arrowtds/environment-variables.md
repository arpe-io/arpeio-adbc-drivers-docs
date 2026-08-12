---
title: Environment variables
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 6
permalink: /drivers/arrowtds/environment-variables/
---

# Environment variables

Diagnostic / opt-in knobs the driver reads at runtime. Every flag here
defaults to "off" and is meant for development, performance triage, or
ad-hoc protocol debugging — production callers should leave them unset.

**Naming convention.** Knobs that are meaningful for every Arpeio ADBC driver
(ArrowTDS/SQL Server, ArrowFEBE/PostgreSQL, ArrowTTC/Oracle, …) use the shared
**`ARPEIO_ADBC_*`** prefix, so one variable configures the whole family in a
process. Knobs specific to one driver/protocol use that driver's
**`ARROW<CODE>_*`** prefix (here, `ARROWTDS_*`). Each variable lives in exactly
one namespace — there is no per-driver override of a shared variable. The
mandatory licence lookup also uses the shared prefix (`ARPEIO_ADBC_LICENCE[_FILE]`,
see [Licensing]({{ '/drivers/arrowtds/licensing/' | relative_url }})).

A truthy value is anything non-empty whose first character is not `'0'`,
unless the table notes a stricter test (e.g. `ARROWTDS_USE_TDS_RPC`
specifically matches `1` / `true` / `TRUE` / `yes`).

## Read path (SELECT)

| Variable | Effect |
|---|---|
| `ARPEIO_ADBC_READ_TIMING` | Per-batch wall-time breakdown on the SELECT stream. Emits one `[TDS READ]` line per batch (`recv_ms`, `decode_ms`, `writer_wait_ms`, `recv_kb`, `cpu_user_ms`, `cpu_krn_ms`) plus a `[TDS READ SUMMARY]` at stream release that adds `busy_ms` (wall time the reader thread was runnable, i.e. not parked in `writer_wait`) and `alloc=` (the column-buffer allocator backend, see `ARPEIO_ADBC_ALLOCATOR`). Use this to decide whether the bottleneck is the wire (`recv_ms`), the row decoder (`decode_ms`), or downstream backpressure (`writer_wait_ms`) — and, when a stage is slow, whether the thread was *blocked* (`busy_ms` ≫ `cpu_user_ms + cpu_krn_ms`: waiting on locks/pages/preemption) or genuinely *out of CPU* (the two are close). |
| `ARROWTDS_TDS_READ_DEBUG` | Verbose per-token trace in the native TDS read path: ENVCHANGE token type/length, RPC stream framing, parser state transitions. Very chatty — only useful when triaging a wire-protocol bug. |
| `ARROWTDS_VECTOR` | **Kill-switch** for SQL Server 2025 native vector transport. Default *on*: LOGIN7 offers `FEATUREEXT_VECTORSUPPORT`, and a 2025 server then returns `VECTOR(n)` columns as Arrow `fixed_size_list<float32>[n]`. Set to a false-y value (`0`, `off`, `no`, `false`) to stop offering the feature, in which case the server falls back to exposing vectors as `varchar(max)` JSON (→ `utf8`). Older servers ignore the unknown feature regardless. Read at login. |

## Write path (BULK INSERT / `adbc_ingest`)

| Variable | Effect |
|---|---|
| `ARPEIO_ADBC_QUIET` | **Suppresses** the bulk path's stderr output: the `[TDS TIMING]` summary printed after every ingest (`begin / encode+send / wait_done / total / rows / server_rows / sub_batches`) **and** the encryption-downgrade and client/server row-count-mismatch warnings. Default is *visible*; any non-empty value not starting with `0` mutes them. Used by the validation suite to keep test output clean, and by shared-logging environments that must not leak server/topology details. |

## Memory / allocator

| Variable | Effect |
|---|---|
| `ARPEIO_ADBC_ALLOCATOR` | Selects the allocator backing the row decoder's Arrow column buffers: `mimalloc` (default when the driver is built with `ARROWTDS_WITH_MIMALLOC`, which release builds are) or `system` (libc `malloc` — the escape hatch if the mimalloc-backed buffers misbehave). Read **once**, at the first column-buffer allocation; the choice is stable for the process lifetime and is reported as `alloc=` in the `[TDS READ SUMMARY]` line. `system` always works; `mimalloc` is silently ignored when not compiled in. |

## Statement execution

| Variable | Effect |
|---|---|
| `ARROWTDS_USE_TDS_RPC` | Route no-rowset DML / DDL through the `sp_executesql` RPC token instead of the default `SQLBATCH` token. Phase 7a opt-in — used during the RPC migration to A/B the two paths. Accepts `1`, `true`, `TRUE`, `yes`. |

## Licence

Unlike the diagnostic flags above, these configure the mandatory licence check
the driver runs at connection time (see [Licensing]({{ '/drivers/arrowtds/licensing/' | relative_url }}) for the
full resolution order and the equivalent `arpeio.adbc.license*` database
options). They are read only if those options are not set.

| Variable | Effect |
|---|---|
| `ARPEIO_ADBC_LICENCE` | The licence blob, inline. Used when neither the `arpeio.adbc.license` nor `arpeio.adbc.license_file` option is set. |
| `ARPEIO_ADBC_LICENCE_FILE` | Path to a `.lic` file. Checked after `ARPEIO_ADBC_LICENCE`; if neither this nor the options are set, the driver falls back to an `arpeio_adbc.lic` file next to the driver binary. |

## Build-time overrides

Read once during build, not at runtime.

| Variable | Effect |
|---|---|
| `ARROWTDS_NATIVE_ARCH` | `ON` (default) compiles the ADBC driver with `-march=native`. Override `OFF` for portable-binary distributors. |

C# loader and parquet-export shim env vars (`ARROWTDS_DRIVER_PATH`,
`ARROWTDS_SHIM_PATH`, `ARROWTDS_VCPKG_DLLS`, `ARROWTDS_EXPORT_TIMING`,
`ARROWTDS_USE_WRITE_TABLE`, `ARROWTDS_PARQUET_VERSION`, …) moved with the
parquet_export tool to [adbc-file-export](https://github.com/aetperf/adbc-file-export).
