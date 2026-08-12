---
title: Authentication
layout: default
parent: ArrowFEBE
grand_parent: Drivers
nav_order: 2
permalink: /drivers/arrowfebe/authentication/
---

# Authentication

ArrowFEBE supports PostgreSQL password-family authentication (SCRAM-SHA-256,
MD5, cleartext), integrated Kerberos/GSSAPI (POSIX) and SSPI (Windows), TLS,
and optional GSS transport encryption. This document is the reference for the
options and credential modes.

## Authentication methods

| Method | Platform | Notes |
|---|---|---|
| **SCRAM-SHA-256** | all | RFC 7677. Default for modern PostgreSQL (`scram-sha-256` in `pg_hba.conf`) |
| **SCRAM-SHA-256-PLUS** | all (TLS) | SCRAM with RFC 5929 `tls-server-end-point` channel binding |
| **MD5** | all | Legacy `md5` auth method |
| **password** | all | Cleartext (`password` auth method) — only safe under TLS or `gssencmode` |
| **GSSAPI / Kerberos** | Linux / macOS | FEBE auth codes 7/8; mutual auth always enforced |
| **SSPI (Negotiate)** | Windows | FEBE auth code 9 |

The driver selects the method from what the server requests in the
`Authentication*` message; you choose between **password-family** and
**integrated** auth via the options below.

> **Non-ASCII passwords:** SCRAM uses the password bytes as-is; SASLprep
> (RFC 4013) normalisation is not applied. Printable-ASCII passwords (the
> common case) are unaffected. A password containing non-ASCII code points that
> the server normalised with `pg_saslprep` when storing the verifier may fail
> to authenticate — use an ASCII password, or pre-normalise it to the same form
> the server stored.

## Option / keyword reference

Options take both a dotted database-level form (`adbc.arrowfebe.<key>`, passed
to `AdbcDatabaseSetOption`) and a short connection-level form (passed to
`AdbcConnectionSetOption`); the database form is copied to the connection
before the session opens. The connection string also accepts ADO.NET-style
keywords (e.g. `User ID`, `Password`, `Integrated Security`).

| Option | Purpose |
|---|---|
| `username` / `password` | SQL login credentials |
| `auth_type=integrated` (or `trusted=true`) | Enable Kerberos/GSSAPI (POSIX) or SSPI (Windows) |
| `sslmode` | `disable` / `prefer` (default) / `require` / `verify-ca` / `verify-full` |
| `ssl_root_cert` | PEM root-CA file for `verify-ca` / `verify-full` (default: system store) |
| `channel_binding` | `prefer` (default) / `require` / `disable`; `require` enforces SCRAM-SHA-256-PLUS |
| `gssencmode` | `disable` (default) / `prefer` / `require` — GSS transport encryption (POSIX/GSSAPI builds) |
| `krb5_ccache` | Explicit ccache file path (default: ambient ccache from `kinit`) |
| `krb5_keytab` | Keytab file path (service accounts) |
| `krb5_principal` + `krb5_password` | Programmatic `kinit` (acquire a TGT via libkrb5) |
| `krb5_spn` | Override the full service principal name |
| `krbsrvname` | Override the SPN service component (default `postgres`, mirroring libpq) |

> **Security — the default `sslmode=prefer` gives no MITM protection.** Like
> libpq, `prefer` falls back to plaintext when the server answers the
> `SSLRequest` with `N`, and that reply is unauthenticated. On an untrusted
> network use `sslmode=verify-full` and/or `channel_binding=require` (which
> binds SCRAM to the TLS channel and refuses a PLUS-stripping downgrade).
> `require` encrypts but does **not** verify the certificate.

## Credential modes (POSIX Kerberos)

Integrated auth on Linux/macOS acquires the client ticket one of three ways,
selected by which `krb5_*` option is set:

1. **Ambient ccache** (default) — the TGT from a prior `kinit`, or the file in
   `KRB5CCNAME` / `krb5_ccache`.
2. **Keytab** (`krb5_keytab`) — for service accounts that cannot run an
   interactive `kinit`.
3. **Principal + password** (`krb5_principal` + `krb5_password`) — the driver
   performs a programmatic `kinit` via libkrb5.

The service principal defaults to `postgres/<host>` (override the service
component with `krbsrvname`, or the whole SPN with `krb5_spn`). Mutual
authentication is always required: the driver refuses an `AuthenticationOk`
from a server that never completed the GSS exchange, and refuses
password-family requests while integrated auth is active.

## GSS transport encryption (`gssencmode`)

{: .since }
> Since ArrowFEBE v0.2.0.

On POSIX/GSSAPI builds, `gssencmode` negotiates GSSAPI-level encryption *before*
the `SSLRequest` and StartupMessage:

| Value | Behaviour |
|---|---|
| `disable` (default) | never negotiate GSS encryption (**deliberate divergence from libpq's `prefer`**) |
| `prefer` | try GSS encryption; fall back to plain or TLS on refusal |
| `require` | GSS encryption must succeed — TLS is skipped, and combining with `sslmode=require`/`verify-ca`/`verify-full` is rejected |

When GSS encryption is active there is no TLS channel, so SCRAM channel binding
is unavailable (plain SCRAM still runs inside the encrypted stream).

## Platform support

| Mode | Linux | macOS | Windows |
|---|---|---|---|
| SQL login (SCRAM/MD5/password) | ✓ | ✓ | ✓ |
| TLS | ✓ | ✓ | ✓ |
| Kerberos / GSSAPI | ✓ | ✓ (ccache only; system GSS lacks MIT extensions) | — |
| SSPI (Negotiate) | — | — | ✓ |
| `gssencmode` | ✓ | ✓ | — |

GSSAPI is a build option (`-DARROWFEBE_ENABLE_GSSAPI`, default `AUTO`); the
released Windows build uses SSPI, and macOS is built with GSSAPI disabled.

## Diagnostics

`febe_krb_diag` is a standalone CLI built alongside the driver when GSSAPI is
enabled. It exercises credential acquisition and
SPN resolution — generating the first GSSAPI token only, **without** opening a
PostgreSQL connection — and reports GSSAPI minor-status strings. Run it with
`--help`.

## Automated coverage

- **C unit tests**: the SCRAM-SHA-256 client against an RFC
  7677 known-answer vector; the auth-provider factory and
  GSSAPI/SSPI dispatch; GSS-encryption negotiation over a
  socketpair.
- **CI**: md5, cleartext, and SCRAM auth paths
  against the PostgreSQL service; TLS (`require` + `verify-full` with a custom
  root cert); and an integrated-auth job that stands up a rootless MIT KDC and
  exercises all three Kerberos credential modes plus GSS encryption. A separate
  Windows job validates SSPI/Negotiate over loopback.
