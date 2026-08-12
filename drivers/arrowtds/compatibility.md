---
title: Compatibility
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 4
permalink: /drivers/arrowtds/compatibility/
---

# ArrowTDS — Compatibility

Which SQL Server versions, editions, and cloud services ArrowTDS is known to work
with. Three support states are used across the Arpeio driver family:

| State | Meaning |
|---|---|
| ✅ **Validated** | Exercised directly — live testing and/or the CI test matrix. |
| 🟢 **Supported** | Expected to work and covered by the same protocol surface, but not run continuously. Report issues and they will be treated as bugs. |
| 🟡 **Not yet validated** | Should be compatible on protocol grounds, but unverified. No promise yet. |

ArrowTDS speaks **MS-TDS 7.4** natively over TCP (+ optional TLS). Any SQL Server
that negotiates TDS 7.4 is a candidate; the mapping details are in
[`DATA_TYPES.md`]({{ '/drivers/arrowtds/data-types/' | relative_url }}).

---

## SQL Server (on-premises / IaaS)

| Version | State | Notes |
|---|---|---|
| SQL Server 2025 | ✅ Validated | Validated live against `2025-latest` (17.0.4065.4, RTM-CU7; compat level 170). ADBC conformance and the ArrowTDS type matrix run at parity with 2022. Adds native **`VECTOR(n)`** → Arrow `fixed_size_list<float32>[n]` (see [`DATA_TYPES.md`]({{ '/drivers/arrowtds/data-types/' | relative_url }})). |
| SQL Server 2022 | ✅ Validated | Primary development & CI target (Linux container). |
| SQL Server 2019 | 🟢 Supported | Same TDS 7.4 surface. |
| SQL Server 2017 | 🟢 Supported | Same TDS 7.4 surface; Linux and Windows. |
| SQL Server 2016 | 🟢 Supported | First version with the full 7.4 feature set used here. |
| SQL Server 2014 / 2012 | 🟡 Not yet validated | TDS 7.3/7.4 — expected to work for core types; untested. |
| SQL Server ≤ 2008 R2 | 🟡 Not yet validated | Older TDS; no testing, not a target. |

**Editions** — Enterprise, Standard, Developer, Web, and Express all speak the
same protocol; the driver does not depend on an edition-specific feature.
Express named instances are reached via the SQL Server Browser (see
[`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}#named-instances)). **LocalDB is not supported** —
it is reached over a named pipe, not TCP.

---

## Azure & Microsoft Fabric

| Service | State | Auth | Notes |
|---|---|---|---|
| Azure SQL Database | ✅ Validated | SQL, Entra ID | Both **Proxy** and **Redirect** gateway policies verified live (Redirect needs outbound TCP 11000–11999). |
| Microsoft Fabric Warehouse | ✅ Validated | SQL | `*.datawarehouse.fabric.microsoft.com`; Entra token passthrough also applies. |
| Azure SQL Managed Instance | 🟡 Not yet validated | SQL, Entra ID | Standard SQL Server surface; expected to work, untested. |
| Azure Synapse (dedicated SQL pool) | 🟡 Not yet validated | SQL, Entra ID | TDS-compatible; some T-SQL/type restrictions apply server-side. |

All Azure endpoints require `encrypt=true` (TLS 1.2+); this is the default-safe
path (SNI + certificate + hostname verification).

---

## Authentication methods

| Method | Platform | State |
|---|---|---|
| SQL login (username/password) | all | ✅ Validated |
| Integrated / trusted — Windows SSPI (Negotiate → Kerberos/NTLM) | Windows client | ✅ Validated |
| Integrated / trusted — POSIX Kerberos (GSSAPI/SPNEGO, ccache/keytab/password) | Linux/macOS client | ✅ Validated |
| Entra ID — access-token passthrough | Azure SQL / Fabric | ✅ Validated |
| Entra ID — service principal (client credentials) | Azure SQL / Fabric | ✅ Validated |
| Entra ID — managed identity | Azure host | 🟢 Supported (mock + unit tested; runs only on an Azure host) |
| Entra ID — `ActiveDirectoryDefault` chain | Azure SQL / Fabric | ✅ Validated |

See [`AUTHENTICATION.md`]({{ '/drivers/arrowtds/authentication/' | relative_url }}) and
[`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}#authentication) for setup.

---

## Client platforms

| Platform | State | Notes |
|---|---|---|
| Linux x64 | ✅ Validated | Primary build & test target. |
| Windows x64 | ✅ Validated | SSPI integrated auth; VC++ runtime + OpenSSL DLLs required alongside the driver. |
| macOS (arm64) | 🟡 Not yet validated | Build support is staged; no released binary yet. |

---

## ADBC conformance

Run against the upstream [`adbc-drivers/validation`](https://github.com/adbc-drivers/validation)
conformance suite: **241 passed / 12 xfailed / 0 xpassed / 0 skipped** (upstream
pinned). The xfails are SQL-Server-structural, not driver defects:

- Arrow nanosecond temporal precision (`time_ns`, `timestamp_ns`, `timestamptz_ns`)
  — SQL Server's scale-7 ceiling is 100 ns.
- `float16` bind — no SQL Server type maps to Arrow `float16`.
- A few unsupported bind inputs (`null`-typed and dictionary-encoded parameters,
  `create_append` schema mismatch).

The conformance suite and
the ArrowTDS type matrix were also re-run live against SQL Server 2025
(17.0.4065.4) with results identical to 2022 — same pass set, same xfails.

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowtds/connection/' | relative_url }}) — connecting to each of the above
- [`DATA_TYPES.md`]({{ '/drivers/arrowtds/data-types/' | relative_url }}) — the type mapping the conformance suite exercises
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowtds/troubleshooting/' | relative_url }}) — when a supported target does not connect
