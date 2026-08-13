---
title: Drivers
layout: default
nav_order: 3
has_children: true
permalink: /drivers/
---

# Drivers
{: .no_toc }

Three published drivers, one shared shape. Each speaks its database's wire
protocol directly and returns Apache Arrow end to end. Pick yours below; every
driver has the same set of pages — connection, authentication, data types,
compatibility, examples, troubleshooting, licensing, environment variables, and
what's new.
{: .fs-5 .fw-300 }

<div class="card-grid" markdown="0">
  <a class="card" href="{{ '/drivers/arrowtds/' | relative_url }}"><span class="card-db">Microsoft SQL Server</span><span class="card-title">ArrowTDS</span><span class="card-body">Native MS-TDS 7.4. Azure SQL &amp; Microsoft Fabric, SQL / integrated / Entra ID auth, geospatial &amp; VECTOR types.</span><span class="card-meta"><span class="badge badge-version">v0.5.25</span> <span class="badge badge-cloud">Azure SQL · Fabric</span></span></a>
  <a class="card" href="{{ '/drivers/arrowfebe/' | relative_url }}"><span class="card-db">PostgreSQL</span><span class="card-title">ArrowFEBE</span><span class="card-body">Native Frontend/Backend v3, no libpq. NUMERIC → decimal with no string detour, SCRAM / Kerberos / GSS auth.</span><span class="card-meta"><span class="badge badge-version">v0.3.8</span> <span class="badge badge-cloud">PG-compatible</span></span></a>
  <a class="card" href="{{ '/drivers/arrowttc/' | relative_url }}"><span class="card-db">Oracle</span><span class="card-title">ArrowTTC</span><span class="card-body">Native TNS + TTC, no OCI or Instant Client. Oracle Cloud Autonomous Database, OCI IAM &amp; Entra ID tokens, Kerberos.</span><span class="card-meta"><span class="badge badge-version">v0.2.8</span> <span class="badge badge-cloud">Oracle Cloud ADB</span></span></a>
</div>

## What's the same across the family

- **Load by name.** Every driver installs an ADBC manifest, so clients connect
  with `driver="arrowtds"` / `"arrowfebe"` / `"arrowttc"` — no library path.
- **Three ways to connect.** Discrete `adbc.arrow<code>.*` options, an ADO.NET
  connection string, or a URI. Structured options always win.
- **Shared diagnostics & licence** via the [`ARPEIO_ADBC_*`]({{ '/environment-variables/' | relative_url }})
  environment variables.
- **Arrow end to end** — results decode straight into Arrow column buffers, no
  intermediate row objects.

## What differs

Each database brings its own types, auth methods and cloud services — that's what
the per-driver pages cover. See each driver's **Compatibility** page for the exact
support matrix and any not-yet-implemented surface.
