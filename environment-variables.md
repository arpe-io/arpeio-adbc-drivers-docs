---
title: Environment variables
layout: default
nav_order: 5
permalink: /environment-variables/
---

# Environment variables
{: .no_toc }

The Arpeio ADBC drivers use **two namespaces** of environment variables. This
page documents the **shared, family-wide** ones — the knobs that mean the same
thing in *every* driver. The **driver-specific** variables live on each driver's
own environment-variables page.
{: .fs-5 .fw-300 }

1. TOC
{:toc}

---

## Two namespaces

| Prefix | Scope | Documented |
|---|---|---|
| `ARPEIO_ADBC_*` | **Shared** — identical meaning in ArrowTDS, ArrowFEBE, ArrowTTC (and ArrowDRDA) | On this page |
| `ARROW<CODE>_*` | **Driver / protocol specific** — e.g. `ARROWTTC_NO_LOB_PREFETCH` | Each driver's env-vars page |

Every variable lives in exactly one namespace; there is no aliasing between them.
When a driver adds a generic knob, it uses the shared prefix from day one, so a
trace or an A/B recipe carries across drivers unchanged.

Per-driver pages:
[ArrowTDS]({{ '/drivers/arrowtds/environment-variables/' | relative_url }}) ·
[ArrowFEBE]({{ '/drivers/arrowfebe/environment-variables/' | relative_url }}) ·
[ArrowTTC]({{ '/drivers/arrowttc/environment-variables/' | relative_url }})

## Shared diagnostics (`ARPEIO_ADBC_*`)

These control diagnostics and low-level behaviour that has no place in a
connection string. They are read from the environment, once, at the point they
apply.

| Variable | Values | Effect |
|---|---|---|
| `ARPEIO_ADBC_READ_TIMING` | `1` | Emit a per-batch read-timing line and a summary at stream release on stderr, attributing an extract to the wire (`recv`), the decoder (`decode`), and the consumer. |
| `ARPEIO_ADBC_WRITE_TIMING` | `1` | Per-batch **ingest** profiling on the write path (drivers that support ingest timing). |
| `ARPEIO_ADBC_ALLOCATOR` | `system`, `mimalloc` | Force the column-buffer allocator backend, read once on the first allocation. `system` always works; `mimalloc` is honoured only when the driver was built with mimalloc (the default). Used to A/B allocators without a rebuild. |
| `ARPEIO_ADBC_QUIET` | `1` | Silence non-error diagnostic output. |

### Reading the timing summary

With `ARPEIO_ADBC_READ_TIMING=1`, each driver prints a summary line at stream
release (the tag names the driver, e.g. `[TTC READ SUMMARY]`):

```
[READ SUMMARY] batches=61 rows=6001215 wall_ms=2090 busy_ms=2075 \
  recv_ms=72 decode_ms=2003 consumer_ms=2 cpu_ms=2057 blocked_ms=18 \
  MiB=596 MiB/s=285 alloc=mimalloc
```

- `recv_ms` — time inside `recv()` / `SSL_read()`. High means the wire (or the
  server) is the bottleneck.
- `decode_ms` — `busy_ms − recv_ms`: turning wire bytes into Arrow buffers.
- `consumer_ms` — wall time spent outside the driver, i.e. waiting for the caller
  to take each batch. High means the consumer is the bottleneck.
- `blocked_ms` — `busy_ms − cpu_ms`: descheduled time the driver was neither on a
  CPU nor in `recv`. Under parallel load this is the allocator-contention signal.
- `alloc` — the resolved allocator backend, `system` or `mimalloc`.

## Licence (`ARPEIO_ADBC_*`)

The licence is resolved the same way in every driver. These environment variables
are two of the sources checked, in precedence order:

| Variable | Values | Effect |
|---|---|---|
| `ARPEIO_ADBC_LICENCE` | inline token | The ARROW LICENCE token itself, if no option is set. |
| `ARPEIO_ADBC_LICENCE_FILE` | path | Path to a licence file, if no option is set. |

Full resolution order (first match wins): the `arpeio.adbc.license` /
`arpeio.adbc.license_file` ADBC option → `ARPEIO_ADBC_LICENCE` /
`ARPEIO_ADBC_LICENCE_FILE` → an `arpeio_adbc.lic` file next to the driver library.
The gate is **always enforced** — a driver refuses to initialise without a valid
token. See any driver's Licensing page (e.g.
[ArrowTTC → Licensing]({{ '/drivers/arrowttc/licensing/' | relative_url }})) for
the details.

{: .note }
> **British spelling.** The licence variables are `LICENCE`, not `LICENSE`
> (`ARPEIO_ADBC_LICENCE_FILE`).
