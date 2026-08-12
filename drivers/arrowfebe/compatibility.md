---
title: Compatibility
layout: default
parent: ArrowFEBE
grand_parent: Drivers
nav_order: 4
permalink: /drivers/arrowfebe/compatibility/
---

# ArrowFEBE — Compatibility

Which PostgreSQL versions, PostgreSQL-compatible services, and client platforms
ArrowFEBE is known to work with. Three support states are used across the Arpeio
driver family:

| State | Meaning |
|---|---|
| ✅ **Validated** | Exercised directly — live testing and/or the CI test matrix. |
| 🟢 **Supported** | Expected to work and covered by the same protocol surface, but not run continuously. Report issues and they will be treated as bugs. |
| 🟡 **Not yet validated** | Should be compatible on protocol grounds, but unverified. No promise yet. |

ArrowFEBE speaks the **PostgreSQL Frontend/Backend (FEBE) protocol v3** natively
over TCP (+ optional TLS). Any server that negotiates FEBE v3 is a candidate; the
mapping details are in [`DATA_TYPES.md`]({{ '/drivers/arrowfebe/data-types/' | relative_url }}).

---

## PostgreSQL (on-premises / IaaS)

| Version | State | Notes |
|---|---|---|
| PostgreSQL 17 | ✅ Validated | In the CI version matrix. |
| PostgreSQL 16 | ✅ Validated | In the CI version matrix. |
| PostgreSQL 15 | ✅ Validated | In the CI version matrix. |
| PostgreSQL 12 | ✅ Validated | In the CI version matrix; the stated floor. |
| PostgreSQL 14 / 13 | 🟢 Supported | Same FEBE v3 surface as the matrix versions. |
| PostgreSQL ≤ 11 | 🟡 Not yet validated | Older but still FEBE v3 — expected to work for core types; untested and out of support upstream. |

The stated floor is **PostgreSQL 12+**. The CI matrix runs **12 / 15 / 16 / 17**
end to end (md5 + cleartext + TLS `require`/`verify-full` auth paths, the full
validation suite, and the data-type matrix under ASan).

---

## PostgreSQL-compatible services & forks

These speak the PostgreSQL wire protocol but are **not** part of the test matrix.
They are expected to work for the core surface on protocol grounds only —
treat them as unverified until you have run your own workload.

| Service | State | Notes |
|---|---|---|
| Amazon Aurora PostgreSQL | 🟡 Not yet validated | PostgreSQL-compatible; untested. |
| Amazon Redshift | 🟡 Not yet validated | PostgreSQL-protocol-compatible, but a different type/feature surface; unverified. |
| Greenplum | 🟡 Not yet validated | PostgreSQL-derived MPP; unverified. |
| CockroachDB | 🟡 Not yet validated | PostgreSQL wire-compatible, distinct engine; unverified. |
| YugabyteDB | 🟡 Not yet validated | PostgreSQL wire-compatible; unverified. |

Managed **stock-PostgreSQL** services (Amazon RDS for PostgreSQL, Azure Database
for PostgreSQL, Google Cloud SQL for PostgreSQL, Supabase, Neon, …) run genuine
PostgreSQL and fall under the version rows above; require TLS with
`sslmode=verify-full` for those endpoints.

---

## Authentication methods

| Method | Platform | State |
|---|---|---|
| SCRAM-SHA-256 (password) | all | ✅ Validated |
| SCRAM-SHA-256-PLUS (channel binding over TLS) | all (TLS) | ✅ Validated |
| MD5 password | all | ✅ Validated |
| Cleartext password | all | ✅ Validated |
| Integrated — POSIX Kerberos (GSSAPI/SPNEGO, ccache/keytab/principal+password) | Linux/macOS client | ✅ Validated |
| Integrated — Windows SSPI (Negotiate → Kerberos/NTLM) | Windows client | 🟢 Supported |
| GSS transport encryption (`gssencmode`) | POSIX/GSSAPI builds | 🟢 Supported |

See [`AUTHENTICATION.md`]({{ '/drivers/arrowfebe/authentication/' | relative_url }}) and
[`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/#authentication' | relative_url }}) for setup.

---

## Client platforms

| Platform | State | Notes |
|---|---|---|
| Linux x64 | ✅ Validated | Primary build & test target. |
| Windows x64 | 🟢 Supported | Self-contained DLL (SSPI integrated auth); the MSVC runtime + OpenSSL DLLs must sit alongside the driver unless static-linked. |
| macOS | 🟡 Not yet validated | A macOS build runs in CI; no released binary yet, symbol-hardening deferred. |

---

## ADBC conformance

Run against the upstream `adbc-drivers-validation` conformance suite:
**297 passed / 5 skipped / 25 xfailed / 0 failed**. Every
xfail is a genuine PostgreSQL limitation, not a driver defect — most notably
Arrow nanosecond temporal precision (PostgreSQL stores microseconds) and a few
unsupported bind inputs.

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowfebe/connection/' | relative_url }}) — connecting to each of the above
- [`DATA_TYPES.md`]({{ '/drivers/arrowfebe/data-types/' | relative_url }}) — the type mapping the conformance suite exercises
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowfebe/troubleshooting/' | relative_url }}) — when a supported target does not connect
