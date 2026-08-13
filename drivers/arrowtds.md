---
title: ArrowTDS
layout: default
parent: Drivers
has_children: true
nav_order: 1
permalink: /drivers/arrowtds/
---

# ArrowTDS
{: .no_toc }

<p>
  <span class="badge badge-version">v0.5.25</span>
  <span class="badge badge-platform">Linux · Windows (x64)</span>
  <span class="badge badge-cloud">Azure SQL · Microsoft Fabric</span>
</p>

A pure-native ADBC driver for **Microsoft SQL Server** that speaks **MS-TDS 7.4**
directly over TCP/TLS and delivers Apache Arrow `RecordBatch`es end to end.
OpenSSL is the only runtime dependency beyond libc — no ODBC, no SQL Server client
libraries. It also covers **Azure SQL** and **Microsoft Fabric Warehouse**.
{: .fs-5 .fw-300 }

{: .note }
> **Reading these docs.** The badge above is the version this documentation
> describes. Options and features added recently are flagged inline with a
> _Since vX.Y.Z_ note; the [What's new]({{ '/drivers/arrowtds/whats-new/' | relative_url }})
> page is the release-by-release view. See [Versioning]({{ '/versioning/' | relative_url }}).

## Highlights

- **Native TDS 7.4** over TCP/TLS — SQL Server 2012–2025, Azure SQL, Fabric.
- **Every auth mode** — SQL logins, trusted/integrated (Windows SSPI; POSIX
  Kerberos via GSSAPI/SPNEGO), and Microsoft Entra ID.
- **Rich type mapping** — including SQL Server 2025 `VECTOR(n)` → Arrow
  `fixed_size_list`, and `geometry`/`geography` → GeoArrow WKB.
- **Arrow straight from the wire** — results decode into Arrow column buffers with
  no intermediate row objects.

## Connect in seconds

```python
import adbc_driver_manager.dbapi as dbapi
with dbapi.connect(driver="arrowtds",
                   db_kwargs={"uri": "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true"}) as conn:
    cur = conn.cursor()
    cur.execute("SELECT * FROM dbo.orders")
    table = cur.fetch_arrow_table()
```

New here? [Install ArrowTDS]({{ '/install/' | relative_url }}) first, then see the
[Connection guide]({{ '/drivers/arrowtds/connection/' | relative_url }}).

## All ArrowTDS pages

<div class="card-grid" markdown="0">
  <a class="card" href="{{ '/drivers/arrowtds/connection/' | relative_url }}"><span class="card-title">Connection</span><span class="card-body">Options, connection strings, URIs, named instances, Azure &amp; Entra.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/authentication/' | relative_url }}"><span class="card-title">Authentication</span><span class="card-body">SQL, trusted/integrated (SSPI, Kerberos), Entra ID.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/data-types/' | relative_url }}"><span class="card-title">Data types</span><span class="card-body">SQL Server ↔ Arrow mapping, temporal precision, geospatial, VECTOR.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/compatibility/' | relative_url }}"><span class="card-title">Compatibility</span><span class="card-body">Supported versions, editions, Azure/Fabric, platforms, conformance.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/examples/' | relative_url }}"><span class="card-title">Examples</span><span class="card-body">Copy-paste recipes: query to Arrow/pandas/Polars/DuckDB, bind, ingest.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/environment-variables/' | relative_url }}"><span class="card-title">Environment variables</span><span class="card-body">Driver-specific <code>ARROWTDS_*</code> knobs and build-time overrides.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/licensing/' | relative_url }}"><span class="card-title">Licensing</span><span class="card-body">Supplying and checking your Arpeio licence.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/troubleshooting/' | relative_url }}"><span class="card-title">Troubleshooting</span><span class="card-body">Connect, TLS, auth, licence and ingest failures and fixes.</span></a>
  <a class="card" href="{{ '/drivers/arrowtds/whats-new/' | relative_url }}"><span class="card-title">What's new</span><span class="card-body">Reader-facing release highlights.</span></a>
</div>
