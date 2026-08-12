---
title: Authentication
layout: default
parent: ArrowTDS
grand_parent: Drivers
nav_order: 2
permalink: /drivers/arrowtds/authentication/
---

# Authentication

ArrowTDS supports the same SQL Server connection modes as FastBCP:

| Mode | How to request it |
|------|-------------------|
| **SQL login** | `username` + `password` (structured options or `User ID=`/`Password=` in a connection string) |
| **Trusted / integrated** | `adbc.arrowtds.trusted=true` (or `=sspi`), or `Integrated Security=SSPI` / `Trusted_Connection=yes` in a connection string |
| **Full connection string** | `adbc.arrowtds.connection_string=...` — the escape hatch |

ArrowTDS *is* the driver (native TDS, no client library underneath), so the
"full connection string" is **parsed** by ArrowTDS, not passed through to
another driver. It can only honor modes ArrowTDS implements; anything else
fails fast (see below).

## Option / keyword reference

Structured ADBC options:

- `adbc.arrowtds.trusted` — `true`/`sspi`/`yes`/`1` → integrated; `false`/`no`/`0` → SQL.
- `adbc.arrowtds.auth_type` — `sql` | `integrated`/`sspi`/`trusted`. The Entra
  ID / certificate families are recognized and rejected with
  `NOT_IMPLEMENTED`.

POSIX Kerberos (GSSAPI) credential options — structured ADBC option and its
connection-string keyword(s). Only consulted under integrated auth; ignored by
SQL login and by the Windows SSPI backend (which always uses the logged-in
session). All are optional.

| Structured option | Connection-string keyword(s) | Selects |
|-------------------|------------------------------|---------|
| `adbc.arrowtds.krb5.keytab` | `Krb5 Keytab File` (`Krb5KeytabFile`, `krb5-keytabfile`) | **keytab** credential mode |
| `adbc.arrowtds.krb5.principal` | `Krb5 Principal` (`Krb5Principal`) | explicit client principal |
| `adbc.arrowtds.krb5.password` | `Krb5 Password` (`Krb5Password`) | **password** credential mode (with a principal) |
| `adbc.arrowtds.krb5.ccache` | `Krb5 Credential Cache` (`Krb5CredentialCache`, `krb5-credcachefile`) | per-connection credential-cache path |
| `adbc.arrowtds.krb5.spn` | `Service Principal Name` (`ServicePrincipalName`, `service_principal_name`) | SPN override (honored by SSPI too) |

`Authenticator=krb5`/`winsspi`/`sspi` is a pyodbc-style alias for integrated
auth; `Authenticator=sql` forces SQL login.

Connection-string auth-mode keywords (case-insensitive; aliases of the same
auth-mode field, so specifying two is a duplicate-key error):

- `Integrated Security` — `SSPI`/`true`/`yes`/`1` → integrated; `false`/`no`/`0` → SQL.
- `Trusted_Connection` — `yes`/`true`/`1` → integrated; `no`/`false`/`0` → SQL.
- `Authentication` — `SqlPassword` → SQL. Any `ActiveDirectory*` value is
  recognized and rejected with a clear `NOT_IMPLEMENTED` error (use
  `Integrated Security=SSPI` for Kerberos/NTLM).

**Mutual exclusion:** integrated/trusted auth cannot be combined with a SQL
`username`/`password`. Doing so fails with `INVALID_ARGUMENT` *before any socket
is opened*. The Kerberos `Krb5 Password` is a **separate** field used to acquire
a Kerberos ticket — it is not the SQL `Password` and does not trip this check.

## Credential modes (POSIX Kerberos)

The GSSAPI backend acquires the initiator credential one of three ways. The
mode is chosen by which options are set, in this precedence:

1. **keytab** — a `krb5.keytab` is set (`Krb5 Keytab File`). For service
   accounts / unattended CI. (MIT-only.)
2. **password** — a `Krb5 Principal` **and** `Krb5 Password` are both set. The
   backend acquires a TGT into a private in-memory credential cache via libkrb5
   (it does not touch the ambient cache). `Krb5 Password` without
   `Krb5 Principal` is an `INVALID_ARGUMENT` error. (MIT-only.)
3. **ccache** (default) — none of the above: the ambient credential cache (a
   prior `kinit`, or `KRB5CCNAME`; `Krb5 Credential Cache` overrides the path).

## Platform support

Trusted auth is implemented on **both** Windows and POSIX:

- **Windows** — SSPI, the "Negotiate" package → Kerberos with NTLM fallback,
  the same mechanism the Microsoft SQL Server ODBC driver and
  `Microsoft.Data.SqlClient` use. `Secur32` is linked automatically on `WIN32`.
- **Linux** — GSSAPI/SPNEGO via MIT Kerberos (`krb5-gssapi`/`krb5`). Built when
  the dev libraries are present at configure time (`ARROWTDS_ENABLE_GSSAPI`,
  default `AUTO`). All three credential modes (ccache / keytab / password) are
  available.
- **macOS** — GSSAPI/SPNEGO via the system GSS framework. The system framework
  lacks the MIT extensions, so only **ccache** mode (and no per-connection
  `krb5.conf` override) is supported there.

The SSPI token exchange is implemented inside the TDS LOGIN7 handshake, using a
provider abstraction and a shared SSPI continuation loop across both login paths.
Both the initial token (in LOGIN7) and continuation tokens (TDS `0x11`) fragment
across packets via a shared framer, so a large Kerberos PAC no longer overflows a
single packet; the response reader reassembles a server SSPI (`0xED`) token that
spans packets.

If a build has neither SSPI (non-Windows) nor GSSAPI (krb5 not found), requesting
integrated auth fails fast with a clear message naming both backends, and SQL
login still works.

## Diagnostics

`tds_krb_diag` (built when GSSAPI is enabled) exercises the GSSAPI provider —
SPN derivation, credential acquisition, the first SPNEGO leg — **without**
opening a TDS socket, to isolate Kerberos/SPN problems from the protocol:

```
tds_krb_diag --server HOST [--port N] [--spn SPN] \
             [--mode ccache|keytab|password] [--keytab FILE] \
             [--ccache NAME] [--principal NAME] [--password PW]
```

## Automated coverage

Linux CI (no Active Directory required):

| Suite | Covers |
|-------|--------|
| `arrowtds_conn_string_tests` | auth keyword parsing incl. `Krb5 Password` + redaction, value validation, Entra-ID rejection, alias-duplicate, precedence merge |
| `arrowtds_login7_tests` | LOGIN7 byte layout (SQL unchanged; INT_SECURITY bit, empty user/pwd, ibSSPI/cbSSPI, cbSSPILong sentinel) **and** multi-packet fragmentation of an oversized token (NORMAL/EOM flags, PacketID, lossless reassembly) |
| `arrowtds_auth_tests` | provider factory platform dispatch, mock 1-leg/2-leg |
| `arrowtds_auth_gssapi_tests` | GSSAPI backend (built when krb5 present): ccache/keytab/password construction, SPN override, password-requires-principal, clean failure without a KDC |
| `arrowtds_auth_flow_tests` | SSPI continuation state machine: NTLM 2-leg, Kerberos 1-leg, server-error leg, provider-step error, **`0xED` token split across packets reassembled** |
| `arrowtds_token_tests` | `0xED` SSPI token parse + bounds check + leak-free free (ASan) |

End-to-end CI:

- **SQL login** against the local Docker SQL Server (SA/SQL auth): logs in and
  queries; mutual-exclusion and Entra-ID rejection verified.
- **Linux Kerberos** (`integrated-auth-kerberos` job): a dockerized MIT KDC +
  SQL Server configured for Kerberos. Builds with GSSAPI, `kinit`s, connects
  with `Integrated Security=SSPI` and asserts `SELECT SUSER_SNAME()` returns the
  Kerberos principal; also exercises keytab and password modes.
- **Windows SSPI** (`integrated-auth-windows` job): SQL Server on a Windows
  runner with Windows Authentication; connects over loopback with
  `Integrated Security=SSPI` (no user/pwd) and asserts
  `SELECT auth_scheme FROM sys.dm_exec_connections WHERE session_id=@@SPID` is
  **`NTLM`** (not `SQL`) — direct proof of trusted auth — plus `SUSER_SNAME()` is
  the Windows account. Loopback resolves Negotiate to **NTLM** (the 2-leg
  challenge/response path); Kerberos (1-leg, SPN handling) is covered by the
  Linux KDC job and the manual domain test below.

## Manual verification on a domain-joined Windows host (Kerberos)

CI's Windows job exercises NTLM over loopback; full **Kerberos** on Windows is
verified manually on a domain-joined host:

1. Domain-joined Windows; SQL Server configured for Windows Authentication;
   the logged-in Windows user has a SQL login + database access.
2. Connect with `Server=<FQDN>;Database=<db>;Integrated Security=SSPI;`
   (no `User ID`/`Password`). `SELECT SUSER_SNAME()` must return the **Windows
   domain account**; `SELECT auth_scheme ...` should report `KERBEROS`.
3. Repeat with `Trusted_Connection=yes` and the structured option
   `adbc.arrowtds.trusted=true`.
4. Negative: integrated + `User ID`/`Password` → `INVALID_ARGUMENT`
   ("cannot be combined") before any socket open;
   `Authentication=ActiveDirectoryIntegrated` → `NOT_IMPLEMENTED`.
5. SPN/FQDN: connect once by short hostname and once by FQDN. The SPN is
   `MSSQLSvc/<host>:<port>` built from the host **verbatim** (no rewrite); a
   short name/alias/IP often falls back to NTLM (still works for a domain user)
   while the FQDN yields Kerberos. The raw SSPI error is surfaced for diagnosing
   `MSSQLSvc/...` mismatches.
6. Large token: connect as a user in many AD groups (large Kerberos PAC). The
   token now fragments across TDS packets via the shared framer, so the login
   succeeds rather than failing on a single-packet limit.
7. Named instance: connect by `host\INSTANCE` (see
   [Named instances]({{ '/drivers/arrowtds/connection/' | relative_url }}#named-instances)). The instance's TCP port is
   resolved from the SQL Server Browser *before* the SPN is derived, so the SPN
   is `MSSQLSvc/<host>:<resolved-port>` — the dynamic port SQL Server registers
   at startup, not 1433. The instance name never appears in the SPN and never
   reaches TLS SNI / certificate verification, both of which see the bare host.
   `krb5_spn` still overrides the derived value.
