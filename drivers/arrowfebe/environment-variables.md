---
title: Environment variables
layout: default
parent: ArrowFEBE
grand_parent: Drivers
nav_order: 6
permalink: /drivers/arrowfebe/environment-variables/
---

# Environment variables

ArrowFEBE has a deliberately small runtime environment surface — almost all
behaviour is set through ADBC options (see [CONNECTION.md]({{ '/drivers/arrowfebe/connection/' | relative_url }})
and [AUTHENTICATION.md]({{ '/drivers/arrowfebe/authentication/' | relative_url }})). This page lists the env vars the
driver actually reads, and the compile-time defines that tune buffering.

## Runtime environment variables

| Variable | Used by | Effect |
|---|---|---|
| `ARPEIO_ADBC_LICENCE` | `arrowfebe_license.c` | Inline licence token, used when no token is supplied via the `arpeio.adbc.license` option |
| `ARPEIO_ADBC_LICENCE_FILE` | `arrowfebe_license.c` | Path to a licence-token file, used when neither the `arpeio.adbc.license_file` option nor an inline token is set |
| `ARPEIO_ADBC_READ_TIMING` | `febe_reader.c` | Truthy (non-empty, first char not `0`) enables per-batch read-path profiling — see below |
| `ARPEIO_ADBC_WRITE_TIMING` | `arrowfebe_bulk.c` | Truthy enables per-batch ingest (COPY) profiling — see below |
| `ARPEIO_ADBC_ALLOCATOR` | `arrowfebe_alloc.c` | `mimalloc` (default in builds with `ARROWFEBE_WITH_MIMALLOC`, which release builds are) or `system` (libc `malloc`) for the Arrow column buffers — see below |

These are the only `getenv` lookups in the driver. Everything else is an ADBC
option, and the licence option forms (`arpeio.adbc.license` /
`arpeio.adbc.license_file`) take precedence over the environment.

**Naming convention (shared across the Arpeio ADBC driver family).** Knobs that
are meaningful for *every* driver — the licence, the allocator, and the timing
traces — use the shared `ARPEIO_ADBC_*` prefix, so one variable configures the
whole family (ArrowTDS / ArrowFEBE / ArrowTTC) and traces from different drivers
compare directly. A knob that is specific to *this* driver or the PostgreSQL
protocol would use the `ARROWFEBE_*` prefix instead; the driver currently has
none. Each variable lives in exactly one namespace — there is no per-driver
override, and the old `ARROWFEBE_{READ_TIMING,WRITE_TIMING,ALLOCATOR}` names no
longer have any effect.

The three `ARPEIO_ADBC_*` timing/allocator vars above are diagnostics, not
configuration: they exist so a performance question can be answered on a shipped
binary, without a rebuild. ArrowFEBE's **write** timing is opt-in, where
ArrowTDS's `[TDS TIMING]` is opt-out — deliberate: this driver prints nothing
unless asked, and a library writing to stderr on every ingest would surprise
callers and break stderr-capturing tests.

### `ARPEIO_ADBC_READ_TIMING`

Emits one `[FEBE READ]` line per batch (`recv_ms`, `decode_ms`,
`writer_wait_ms`, `recv_kb`, `cpu_user_ms`, `cpu_krn_ms`) and a
`[FEBE READ SUMMARY]` at stream release adding `busy_ms` (wall time inside
`get_next`), `alloc=` (the live column-buffer backend) and `path=` (`copy` for
the COPY-binary fast path, `rows` for the DataRow fallback).

Use it to attribute an extract to the wire (`recv_ms`), the row decoder
(`decode_ms`), or downstream backpressure (`writer_wait_ms`). When a stage is
slow, the derived quantity

```
blocked = busy_ms - (cpu_user_ms + cpu_krn_ms)
```

says *why*: a thread waiting on a lock is descheduled and burns no CPU, so a
large `blocked` means it was **stuck**, not **out of CPU**. Cost when unset is
one `getenv` per stream; the thread-CPU clock is sampled twice per batch, never
per row.

### `ARPEIO_ADBC_WRITE_TIMING`

The ingest counterpart. Emits one `[FEBE WRITE]` line per Arrow batch and a
`[FEBE WRITE SUMMARY]` at the end of the COPY, splitting the wall clock so the
client's cost is separable from the server's:

| field | what |
|---|---|
| `begin_ms` | COPY command sent → `CopyInResponse`. The server preparing the target. |
| `encode_ms` | Arrow → PostgreSQL binary. **Ours.** Derived as `busy − send`, so it encloses the write buffer's allocations as well as the encoders. |
| `send_ms` | time in `send()` / `SSL_write`. |
| `wait_done_ms` | `CopyDone` → `ReadyForQuery`. The server committing. Past ~20 parallel connections this dominates. |

Same `blocked = busy_ms − (cpu_user_ms + cpu_krn_ms)` diagnostic as the read path.
Sampled twice per batch, never per row; one `getenv` per ingest when unset.

### `ARPEIO_ADBC_ALLOCATOR`

Selects the allocator backing the row decoder's Arrow column buffers. Read
**once**, at the first column-buffer allocation; the choice is then stable for
the process lifetime and is reported as `alloc=` in `[FEBE READ SUMMARY]`.
`system` always works; `mimalloc` is silently ignored when not compiled in.

**Read path only.** The ingest path's COPY buffer is on libc regardless of this
setting — measured in v0.3.4: it does not convoy, so there is nothing here for
the seam to make cheaper. That is why
`[FEBE WRITE SUMMARY]` carries no `alloc=` field — it would be reporting a
backend with no bearing on the line it appears on.

This is the escape hatch for the allocation seam, and the switch that lets one
binary be A/B'd at high parallel degree.

> Kerberos honours the standard MIT env vars (`KRB5CCNAME`, `KRB5_KTNAME`, …)
> through libkrb5/GSSAPI itself — ArrowFEBE does not read them directly; the
> `krb5_ccache` / `krb5_keytab` options override them when set. See
> [AUTHENTICATION.md]({{ '/drivers/arrowfebe/authentication/#credential-modes-posix-kerberos' | relative_url }}).

## Build-time overrides

These are C preprocessor defines (`-D…` at compile time), not env vars. Defaults
are in the source; override them only with a deliberate measurement in hand.

| Define | Default | Effect |
|---|---|---|
| `FEBE_COPY_CHUNK_BYTES` | 4 MiB | `CopyData` flush threshold for ingest (`adbc_ingest`, `arrowfebe_bulk.c`) |
| `FEBE_COPY_MAX_BYTES` | 2 GiB | Cap on COPY-out reassembly, guarding against a malicious/buggy server forcing unbounded client memory (`febe_reader.c`) |
| `FEBE_RBUF_KIB` | 256 | Size (KiB) of the userspace receive buffer in `febe_conn.c` (`-DFEBE_RBUF_KIB=<n>`); the gain plateaus at the default |
| `FEBE_DEBUG` | unset | When defined, compiles in verbose diagnostic `stderr` output on the statement/ingest paths |

## CMake options

Build configuration is set via CMake cache variables, not the environment. The common ones:
`ARROWFEBE_BUILD_TESTS`, `ARROWFEBE_ASAN`, `ARROWFEBE_FUZZ`,
`ARROWFEBE_ENABLE_GSSAPI`, `ARROWFEBE_NATIVE_ARCH`, `ARROWFEBE_LICENSE_TEST_KEY`,
`ARROWFEBE_WITH_MIMALLOC`.

### `ARROWFEBE_WITH_MIMALLOC` (default `ON`)

Links mimalloc and routes the Arrow column buffers through it. Configuring it
together with `ARROWFEBE_ASAN`/`ARROWFEBE_TSAN` is a **fatal error**: mimalloc
would bypass the sanitizer's malloc interceptor on exactly the buffers under
test, silently blinding it. Sanitizer builds must pass
`-DARROWFEBE_WITH_MIMALLOC=OFF`.

This is the driver's only build-time network dependency. CMake first tries
`find_package(mimalloc 3 CONFIG)` — a system, Homebrew, vcpkg or
`conda install -c conda-forge mimalloc` install satisfies it with no download —
and only falls back to fetching `microsoft/mimalloc v3.3.2`. To build offline
without a system copy, either point at a local clone:

```
cmake -S core -B build -DFETCHCONTENT_SOURCE_DIR_MIMALLOC=$HOME/src/mimalloc
```

or drop the dependency entirely with `-DARROWFEBE_WITH_MIMALLOC=OFF` (the seam
degrades to libc `malloc`; everything still builds and passes).

## Test-harness variables

These are read by the Python test suites, not the driver:

| Variable | Effect |
|---|---|
| `ARROWFEBE_DRIVER_PATH` | Explicit path to the built `.so`/`.dll` for the tests to load |
| `ADBC_ARROWFEBE_SERVER` / `_PORT` / `_DATABASE` / `_USERNAME` / `_PASSWORD` | Override the test target (CI) |
| `ADBC_ARROWFEBE_SSLMODE` / `_SSL_ROOT_CERT` | TLS knobs for the CI TLS job |
