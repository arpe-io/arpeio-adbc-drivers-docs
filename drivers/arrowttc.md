---
title: ArrowTTC
layout: default
parent: Drivers
has_children: true
nav_order: 3
permalink: /drivers/arrowttc/
---

# ArrowTTC
{: .no_toc }

<p>
  <span class="badge badge-version">v0.2.8</span>
  <span class="badge badge-platform">Linux · Windows (x64)</span>
  <span class="badge badge-cloud">Oracle Cloud Autonomous Database</span>
</p>

A native ADBC driver for **Oracle**, written in C, returning Apache Arrow
`RecordBatch`es end to end. It speaks **TNS (Oracle Net) + TTC (Two-Task Common)**
directly over TCP — **no OCI, no Instant Client, no ODBC.** OpenSSL is the only
runtime dependency beyond libc. Targets Oracle 19c and **Oracle Cloud Autonomous
Database (ADB)**.
{: .fs-5 .fw-300 }

{: .note }
> **Reading these docs.** The badge above is the version this documentation
> describes. Options and features added recently are flagged inline with a
> _Since vX.Y.Z_ note; the [What's new]({{ '/drivers/arrowttc/whats-new/' | relative_url }})
> page is the release-by-release view. See [Versioning]({{ '/versioning/' | relative_url }}).

## Highlights

- **Native TNS + TTC** over TCP/TLS — no OCI, no Instant Client, no ODBC.
- **Oracle Cloud ready** — connects to **Autonomous Database (ADB)** over mutual
  TLS with the downloaded wallet, and authenticates with an **OCI IAM database
  token** or a password.
- **Modern auth** — O5LOGON password, proxy (CONNECT THROUGH), Kerberos, OCI IAM
  token, and **OAuth2 / Microsoft Entra ID** bearer tokens.
- **Native Network Encryption** — AES-256 session encryption + SHA-256/SHA-1
  integrity without TCPS, interoperating with Oracle 19c through 23ai.
- **Configurable `NUMBER` mapping** — lossless `utf8`, `float64`, or
  `decimal128(P,S)`.

## Connect in seconds

```python
import adbc_driver_manager.dbapi as dbapi
with dbapi.connect(driver="arrowttc",
                   db_kwargs={"uri": "oracle://scott:tiger@dbhost:1521/orclpdb1?ssl_mode=verify-full"}) as conn:
    cur = conn.cursor()
    cur.execute("SELECT * FROM hr.employees")
    table = cur.fetch_arrow_table()
```

New here? [Install ArrowTTC]({{ '/install/' | relative_url }}) first, then see the
[Connection guide]({{ '/drivers/arrowttc/connection/' | relative_url }}) (including
the [Oracle Cloud ADB]({{ '/drivers/arrowttc/connection/' | relative_url }}) section).

## All ArrowTTC pages

<div class="card-grid" markdown="0">
  <a class="card" href="{{ '/drivers/arrowttc/connection/' | relative_url }}"><span class="card-title">Connection</span><span class="card-body">Options, connection strings, Oracle URIs, EZConnect, Autonomous Database.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/authentication/' | relative_url }}"><span class="card-title">Authentication</span><span class="card-body">O5LOGON, proxy, Kerberos, OCI IAM token, OAuth2 / Entra ID, TLS.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/data-types/' | relative_url }}"><span class="card-title">Data types</span><span class="card-body">Oracle → Arrow mapping, the <code>NUMBER</code> mapping option, write path.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/compatibility/' | relative_url }}"><span class="card-title">Compatibility</span><span class="card-body">Supported Oracle versions, Oracle Cloud, auth methods, platforms.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/examples/' | relative_url }}"><span class="card-title">Examples</span><span class="card-body">Copy-paste recipes: query to Arrow/pandas/Polars/DuckDB, bind, ingest.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/environment-variables/' | relative_url }}"><span class="card-title">Environment variables</span><span class="card-body">Connection options and driver-specific <code>ARROWTTC_*</code> knobs.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/licensing/' | relative_url }}"><span class="card-title">Licensing</span><span class="card-body">Supplying and checking your Arpeio licence.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/troubleshooting/' | relative_url }}"><span class="card-title">Troubleshooting</span><span class="card-body">Connect, TLS/wallets, NNE, auth, licence and ingest failures.</span></a>
  <a class="card" href="{{ '/drivers/arrowttc/whats-new/' | relative_url }}"><span class="card-title">What's new</span><span class="card-body">Reader-facing release highlights.</span></a>
</div>
