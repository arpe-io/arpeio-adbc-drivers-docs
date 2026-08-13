---
title: Licensing
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 7
permalink: /drivers/arrowttc/licensing/
---

# ArrowTTC — Licensing

ArrowTTC is a proprietary Arpeio driver. The binary validates an **Arpeio
licence** at connection time, and on ArrowTTC the gate is **always enforced** —
there is no report-only build. This page is about **supplying your licence** —
where to put it and how to check it. To obtain a licence, contact
**sales@arpe.io** or see **https://www.arpe.io**.

The same licence mechanism and the shared `ARPEIO_ADBC_*` environment variables
are used by every driver in the family (ArrowTDS, ArrowFEBE, ArrowTTC, ArrowDRDA),
so one licence setup carries across them.

---

## Where the driver looks for a licence

At `AdbcDatabaseInit` the driver resolves a licence from the following sources, in
order — **the first present source wins**:

1. **`arpeio.adbc.license`** — ADBC database option, the licence blob inline.
2. **`arpeio.adbc.license_file`** — ADBC database option, path to a `.lic` file.
3. **`ARPEIO_ADBC_LICENCE`** — environment variable, the licence blob inline.
4. **`ARPEIO_ADBC_LICENCE_FILE`** — environment variable, path to a `.lic` file.
5. **`arpeio_adbc.lic`** — a file placed next to the driver library
   (`libarrowttc_adbc_driver.so` / `arrowttc_adbc_driver.dll`).

> Note the British spelling **`LICENCE`** in the environment-variable names.

If none of these yields a valid licence, the connection fails with an
`ARROW_LIC_*` error (see [Troubleshooting](#troubleshooting)). The token is an
ECDSA P-256 / SHA-256 signature validated against the driver family's production
key; a copy of the `.so`/`.dll` lifted out of a host bundle is inert on its own.

---

## Supplying it yourself

**As an ADBC option** (keeps it out of the environment):

<p class="code-lang-label">Python</p>

```python
import adbc_driver_manager.dbapi as dbapi

conn = dbapi.connect(driver="arrowttc", db_kwargs={
    "adbc.arrowttc.server":       "localhost",
    "adbc.arrowttc.service_name": "orclpdb1",
    "adbc.arrowttc.username":     "scott",
    "adbc.arrowttc.password":     "tiger",
    "arpeio.adbc.license_file":   "/etc/arpeio/arpeio_adbc.lic",
})
```

**As an environment variable** (one setting for every Arpeio driver in the
process):

```bash
export ARPEIO_ADBC_LICENCE_FILE=/etc/arpeio/arpeio_adbc.lic
# or the blob inline:
export ARPEIO_ADBC_LICENCE="$(cat /etc/arpeio/arpeio_adbc.lic)"
```

**A file next to the library** — drop `arpeio_adbc.lic` in the same directory as
the loaded `.so`/`.dll` (source #5), and no option or environment variable is
needed at runtime. Once the prebuilt `arrowttc` installer is published to
[`arpe-io/adbc-drivers`](https://github.com/arpe-io/adbc-drivers) its `--license`
flag will place the file for you, as it does for the other Arrow\* drivers (see
[Install]({{ '/install/' | relative_url }})).

**Embedding tools** (FastBCP, FastTransfer, …) forward their own licence into one
of these sources automatically — you normally do not configure the driver licence
separately when using them.

---

## Checking the licence state

Read the current state with the **read-only** `arpeio.adbc.license.status`
database option (via `GetOption`). It never fails and never opens a connection:

```
<state>;code=<ARROW_LIC_*>;tier=<tier>;expires=<epoch>
```

The value reports the state, the result code, the licence tier, and the expiry as
a Unix epoch.

---

## A note on test builds

Test and CI builds set `-DARROWTTC_LICENSE_TEST_KEY=ON`, which embeds a reserved
`arrow-test` key so the suite can self-mint a token offline with no network and
no production key. A
**shipped** driver trusts only the production key and never the test key — never
distribute a driver built with the test key.

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

- [`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}#licence) — the licence options in the full option reference
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowttc/troubleshooting/' | relative_url }}#licence) — the licence error in context
