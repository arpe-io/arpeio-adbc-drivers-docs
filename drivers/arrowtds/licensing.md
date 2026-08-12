---
title: Licensing
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 7
permalink: /drivers/arrowtds/licensing/
---

# ArrowTDS — Licensing

ArrowTDS is a proprietary Arpeio driver. The binary is free to download, and it
validates an **Arpeio licence** at connection time. This page is about **supplying
your licence** — where to put it and how to check it. To obtain a licence, contact
**sales@arpe.io** or see **https://www.arpe.io**.

The same licence mechanism and the shared `ARPEIO_ADBC_*` environment variables
are used by every driver in the family (ArrowTDS, ArrowFEBE, ArrowTTC, ArrowDRDA),
so one licence setup carries across them.

---

## Where the driver looks for a licence

At `AdbcDatabaseInit` (re-checked at `AdbcConnectionInit`) the driver resolves a
licence from the following sources, in order — **the first present source wins**:

1. **`arpeio.adbc.license`** — ADBC database option, the licence blob inline.
2. **`arpeio.adbc.license_file`** — ADBC database option, path to a `.lic` file.
3. **`ARPEIO_ADBC_LICENCE`** — environment variable, the licence blob inline.
4. **`ARPEIO_ADBC_LICENCE_FILE`** — environment variable, path to a `.lic` file.
5. **`arpeio_adbc.lic`** — a file placed next to the driver library
   (`libarrowtds_adbc_driver.so` / `arrowtds_adbc_driver.dll`).

> Note the British spelling **`LICENCE`** in the environment-variable names.

If none of these yields a valid licence, the connection fails with an
`ARROW_LIC_*` error (see [Troubleshooting](#troubleshooting)).

---

## The easy path: the installer places it for you

The Arpeio installer copies your licence next to the driver as `arpeio_adbc.lic`
(source #5), so no option or environment variable is needed at runtime:

```bash
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh \
  | sh -s -- arrowtds --license ./arpeio_adbc.lic
```

`--license-content "<blob>"` works too (handy in CI — but note it is visible in the
process list). See the [install guide]({{ '/install/' | relative_url }}) for details.

---

## Supplying it yourself

**As an ADBC option** (keeps it out of the environment):

```python
import adbc_driver_manager.dbapi as dbapi

conn = dbapi.connect(driver="arrowtds", db_kwargs={
    "adbc.arrowtds.server":     "localhost",
    "adbc.arrowtds.database":   "tpch",
    "adbc.arrowtds.username":   "sa",
    "adbc.arrowtds.password":   "<password>",
    "arpeio.adbc.license_file": "/etc/arpeio/arpeio_adbc.lic",
})
```

**As an environment variable** (one setting for every Arpeio driver in the
process):

```bash
export ARPEIO_ADBC_LICENCE_FILE=/etc/arpeio/arpeio_adbc.lic
# or the blob inline:
export ARPEIO_ADBC_LICENCE="$(cat /etc/arpeio/arpeio_adbc.lic)"
```

**Embedding tools** (FastBCP, FastTransfer, …) forward their own licence into one
of these sources automatically — you normally do not configure the driver licence
separately when using them.

---

## Checking the licence state

Read the current state with the **read-only** `arpeio.adbc.license.status` database
option (via `GetOption`):

```python
db = conn.adbc_connection.adbc_get_option  # low-level handle varies by binding
status = conn.adbc_connection._native_option("arpeio.adbc.license.status")
# -> "<state>;code=<ARROW_LIC_*>;tier=<tier>;expires=<epoch>"
```

The value reports the state, the result code, the licence tier, and the expiry as
a Unix epoch.

---

## Troubleshooting

**Connection fails with `ARROW_LIC_*`.** No valid licence was found by any of the
five sources above, or the licence has expired / does not cover this driver.

- Confirm the file exists and is readable at the path you configured.
- If you rely on the file-next-to-the-library fallback, confirm `arpeio_adbc.lic`
  really sits in the same directory as the loaded `.so`/`.dll` (the *loaded* one —
  a stale copy elsewhere on the path won't count).
- Check expiry and tier via `arpeio.adbc.license.status`.
- Contact **sales@arpe.io** for a renewal or a licence covering additional drivers.

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}#licence) — the licence options in the full option reference
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowtds/troubleshooting/' | relative_url }}#licence) — the licence error in context
- The installer README in [`arpe-io/adbc-drivers`](https://github.com/arpe-io/adbc-drivers)
