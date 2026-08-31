---
title: Compatibility
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 4
permalink: /drivers/arrowttc/compatibility/
---

# ArrowTTC — Compatibility

Which Oracle versions, cloud services, auth methods, and client platforms
ArrowTTC is known to work with. Three support states are used across the Arpeio
driver family:

| State | Meaning |
|---|---|
| ✅ **Validated** | Exercised directly — live testing and/or the CI test matrix. |
| 🟢 **Supported** | Expected to work and covered by the same protocol surface, but not run continuously. Report issues and they will be treated as bugs. |
| 🟡 **Not yet validated** | Should be compatible on protocol grounds, but unverified. No promise yet. |

ArrowTTC speaks **TNS (Oracle Net) + TTC (Two-Task Common)** natively over TCP
(+ optional TCPS/TLS or Native Network Encryption). The primary target is Oracle
**19c** and Oracle Cloud **Autonomous Database**; the type mapping is in
[`DATA_TYPES.md`]({{ '/drivers/arrowttc/data-types/' | relative_url }}).

---

## Oracle Database (on-premises / IaaS)

| Version | State | Notes |
|---|---|---|
| Oracle 12c | ✅ Validated | Type-matrix round-trip verified live (Enterprise 12.2.0.1). The 12c O5LOGON PBKDF2 verifier and the 12c TTC field version are exercised end to end. |
| Oracle 18c | ✅ Validated | Type-matrix round-trip verified live (Oracle 18c XE). Same `DESCRIBE_INFO` surface as 19c. |
| Oracle 19c | ✅ Validated | Primary development & CI target. Password, TLS, NNE, Kerberos, OCI IAM token, OAuth2/Entra all verified live. |
| Oracle 21c | ✅ Validated | Type-matrix round-trip verified live (Oracle 21c XE). Reports TTC field version 16 and parses cleanly. |
| Oracle 23ai | 🟡 Not yet validated | Connects, but 23ai reports TTC field version 25 (23_4+): the server appends native VECTOR describe fields and can return native `BOOLEAN`/`JSON` columns only when the client advertises field version 23_4 — which also switches the auth handshake to a `PROTOCOL` renegotiation. Native-type decode + the fv-23_4 auth flow are a tracked follow-up. Best-effort until then. |
| Oracle 26ai (23.26) | 🟡 Not yet validated | Oracle's rebrand of 23ai — version `23.26.x`, same protocol family. Same follow-up as 23ai. |

**19c is the primary target.** 12c, 18c and 21c are now validated live (the
type-matrix round-trip passes on each); 23ai/26ai are a tracked follow-up
(native `BOOLEAN`/`JSON`/`VECTOR` decode plus the field-version-23_4 auth flow).

---

## Oracle Cloud

| Service | State | Auth | Notes |
|---|---|---|---|
| Autonomous Database (ADB) | ✅ Validated | password, OCI IAM token, OAuth2/Entra | Verified live over mutual TLS with the downloaded instance wallet (`ewallet.pem`). Use a `_low`/`_tp`/… service on port 1522 and `ssl_mode=verify-full`. |

ADB requires mutual TLS — pair every ADB connection with the wallet
(`wallet_location` + `wallet_password`). See
[`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}#oracle-cloud-autonomous-database-adb).

---

## Authentication methods

All verified live (on-prem 19c and/or Autonomous Database):

| Method | State | Notes |
|---|---|---|
| Password (O5LOGON, 12c PBKDF2-HMAC-SHA512 and 11g SHA-1 verifiers) | ✅ Validated | Password never crosses the wire in clear, even without TLS. |
| Proxy authentication (`CONNECT THROUGH`) | ✅ Validated | Rides the O5LOGON login, no extra round trip. |
| Change-password-at-connect | ✅ Validated | The way in past an expired password. |
| Kerberos (`KERBEROS5`) | ✅ Validated | ccache / keytab / password modes; composes with Native Network Encryption. **Linux / MIT krb5 only.** |
| OCI IAM database token | ✅ Validated | Oracle Cloud; token + RSA proof-of-possession signature. |
| OAuth2 / Microsoft Entra ID bearer token | ✅ Validated | Global user resolved from the token's `upn` claim. |
| TLS (TCPS), incl. downloaded ADB wallet | ✅ Validated | `require` / `verify-ca` / `verify-full`; mutual TLS with wallets. |
| Native Network Encryption (AES-256 + SHA-256/SHA-1) | ✅ Validated | No TCPS required; connects a `SQLNET.ENCRYPTION_SERVER=REQUIRED` server. |

Not implemented: `NTS`/NTLM and Windows SSPI. See
[`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}).

---

## Client platforms

| Platform | State | Notes |
|---|---|---|
| Linux x64 | ✅ Validated | Primary build & test target. MIT krb5 (optional) enables Kerberos. |
| Windows x64 | 🟢 Supported | Builds and runs under MSVC; the hermetic CTest suite runs on `windows-latest` every push, plus an optional live TLS connect to Autonomous Database. Password/O5LOGON, TLS, OCI IAM token, and OAuth2/Entra all work; **Kerberos stays Linux-only** (no SSPI). No prebuilt binary published yet. |
| macOS (arm64) | 🟡 Not yet validated | Staged; no released binary yet. |

---

## ADBC conformance

ArrowTTC implements the **ADBC 1.1.0** surface: query, prepared statements +
bound parameters, DML/DDL update, transactions (commit/rollback + autocommit),
bulk ingest, LOB streaming, query cancellation, and catalog/metadata (`GetInfo`,
`GetObjects`, `GetTableSchema`, `GetTableTypes`).

Correctness is guarded by two suites: an **offline hermetic C tier** (20 CTest
suites — error/SQLSTATE mapping, allocator, metadata builders, OAuth2 + OCI-IAM
auth builders, option mapping, session lifecycle, and the decoders — run under
gcc, clang, MSVC, and ASan/UBSan on every push), and the **live Oracle
type-matrix**. `GetStatistics`, `ExecuteSchema`, and
`GetParameterSchema` are not yet implemented.

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}) — connecting to each of the above
- [`DATA_TYPES.md`]({{ '/drivers/arrowttc/data-types/' | relative_url }}) — the type mapping the suites exercise
- [`TROUBLESHOOTING.md`]({{ '/drivers/arrowttc/troubleshooting/' | relative_url }}) — when a supported target does not connect
