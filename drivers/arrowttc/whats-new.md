---
title: What's new
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 9
permalink: /drivers/arrowttc/whats-new/
---

# What's new in ArrowTTC

Feature-oriented release highlights for ArrowTTC — the pure-native Oracle ADBC
driver that speaks TNS (Oracle Net) + TTC directly over TCP and streams Apache
Arrow end-to-end, with no OCI, no Instant Client, and no ODBC. This is the
reader-friendly view; the full engineering detail (every fix, every internal
change) lives in `CHANGELOG.md`.

---

## Validated on Oracle 12c / 18c / 19c / 21c — August 2026 (v0.2.9)

ArrowTTC is now validated live against Oracle **12c, 18c, 19c and 21c**. The full
type-matrix round-trip — every NUMBER mapping branch, `BINARY_FLOAT`/`BINARY_DOUBLE`,
`DATE`, all `TIMESTAMP` variants including TZ/LTZ, `RAW`, both `INTERVAL` kinds, and
`VARCHAR2`/`CHAR`/`NVARCHAR2`/`NCHAR` with multibyte and astral text — passes against a
live server of each version, on the same driver binary with no version-specific code
paths. See [Compatibility]({{ '/drivers/arrowttc/compatibility/' | relative_url }}).
Oracle **23ai / 26ai** (which report TTC field version 24/25 and gate native
`BOOLEAN`/`JSON`/`VECTOR` columns behind a field-version-23_4 auth renegotiation) are a
tracked follow-up. Validation and documentation only — the binaries are identical to
v0.2.8.

## Hardened release binaries — August 2026 (v0.2.8)

The `linux-x64` `.so` and `win-x64` `.dll` published on each release tag are now
built and verified as shipping artifacts rather than developer builds. Build-machine
paths and the compiler identity are kept out of the binary, and a release-only
strip reduces each library to just its two intended ADBC exports — enforced by a
CI gate that fails the build if anything else rides along. Distribution hygiene
only: no change to the driver's code, wire behaviour, or API.

## Faster result decoding — August 2026 (v0.2.7)

Three driver-internal optimizations on the `SELECT` decode path cut roughly
**13% of decode CPU** on a wide scan: link-time optimization inlining the hot
decode chain, width-specialised appends for Oracle's fixed types, and an in-place
Native Network Encryption receive path that drops a full-payload allocation and
copy per DATA packet. Purely performance — the Arrow output is verified
byte-for-byte unchanged.

## Login-path security fixes — August 2026 (v0.2.6)

Two server-triggerable memory-corruption bugs in the authentication path are
closed: an over-long `AUTH_SESSKEY` that overflowed the session-key field, and a
use-after-free in the Kerberos AP-REP handler — both found in code review and
covered by ASan-verified regression tests. A separate use-after-free race between
a statement cancel on another thread and connection teardown is fixed by keeping
the session allocation stable for the connection's lifetime. Security and
correctness fixes; no behaviour change on a healthy connection.

## Prebuilt binaries, NuGet & `GetStatisticNames` — August 2026 (v0.2.5)

`GetStatisticNames` is now implemented, completing the ADBC 1.1.0 statistics
surface (it returns the conformant empty set — ArrowTTC emits only the standard
keys). A release workflow now builds `linux-x64` and `win-x64` on every version
tag and attaches the binaries, the dual-RID `ArrowTtc.AdbcDriver` NuGet package,
and `SHA256SUMS` to the GitHub release, cross-published to the public installer
repo for anonymous install — the runtime licence gate keeps a downloaded driver
inert without a valid token.

## Statement cache & 21c/23ai native encryption — August 2026 (v0.2.4)

A bounded per-connection statement cache reuses open Oracle cursors across
repeated SQL, closing the server-side cursor leak that could otherwise exhaust
`OPEN_CURSORS` and fail a long-lived connection with `ORA-01000`. The new
`adbc.arrowttc.statement_cache_size` option (default 20, `0` disables) matches
python-oracledb's `stmtcachesize`. In the same release, Native Network Encryption
now interoperates with **Oracle 21c and 23ai** — a corrected ANO service-version
marker fixes the `ORA-12599` refusal those servers returned before.

## A standalone driver you load by name — August 2026 (v0.2.3)

ArrowTTC is becoming an independently installable ADBC driver, not just an
ingredient of other tools:

- **Load by name** — `cmake --install` writes an ADBC **driver manifest**
  (`arrowttc.toml`), so any ADBC client connects with `driver="arrowttc"`; no
  library path to wire up. The manifest version is parsed from the header, so it
  can't drift.
- **Portable connection URIs** — a real Oracle URL parser behind the standard
  `uri` option: `oracle://dbuser:<password>@dbhost:1521/appdb?ssl_mode=verify-full`
  (or the branded `arrowttc://`). The path segment is the service name; `?sid=`
  selects the SID form.
- Matching the ArrowTDS / ArrowFEBE convention across the driver family.

## Windows x64 build target — August 2026 (v0.2.3)

ArrowTTC now builds and runs on **Windows x64 under MSVC**. The socket and
cross-thread cancel layers moved into a small compatibility seam (POSIX `eventfd`
vs a Windows loopback socket-pair), keeping the `poll`-based reader identical. A
CI job compiles under MSVC and runs the full hermetic test suite on every push,
plus an optional live TLS connect to Autonomous Database. Password/O5LOGON, TLS,
OCI IAM token, and OAuth2/Entra all work on Windows; Kerberos stays Linux-only.

## Licence gate always enforced — August 2026 (v0.2.3)

The licence check at connect is now **unconditional** — the report-only build
mode is gone, matching ArrowTDS. Supply a token via `arpeio.adbc.license[_file]`,
the `ARPEIO_ADBC_LICENCE[_FILE]` environment variable, or an `arpeio_adbc.lic`
file next to the driver. See [`docs/LICENSING.md`]({{ '/drivers/arrowttc/licensing/' | relative_url }}).

## OAuth2 / Microsoft Entra ID login — August 2026 (v0.2.2)

Log in with a Microsoft Entra ID (Azure AD) access token — no password, no OCI
key. Set `access_token` inline (or `token_file`) and the driver sends the bearer
JWT; Oracle resolves the global user from the token's `upn` claim. Reverse-engineered
from python-oracledb's wire format and **verified live against Autonomous
Database** (an Entra-federated user reports `TOKEN_GLOBAL`). See
[Authentication → OAuth2 / Entra ID]({{ '/drivers/arrowttc/authentication/' | relative_url }}#oauth2--microsoft-entra-id-bearer-token).

## Oracle Cloud Autonomous Database & OCI IAM token — August 2026 (v0.2.0)

ArrowTTC connects to Oracle **Autonomous Database (ADB)** over mutual TLS with the
downloaded instance wallet, and authenticates with an **OCI IAM database token**
as well as a password. Point `wallet_location` at the unzipped wallet, use a
`_low`/`_tp`/… service on port 1522, and connect — password or token. Verified end
to end against a live ADB instance. See
[Connection → Oracle Cloud ADB]({{ '/drivers/arrowttc/connection/' | relative_url }}#oracle-cloud-autonomous-database-adb).

## Statement cancellation over TLS — August 2026 (v0.2.1)

Cancelling a long-running query now works on an encrypted (TCPS) connection, not
just plaintext — the last transport gap in the cancel path. A cancel from another
thread wakes the reader, which sends the in-band break and reads the server's
abandon reply; the connection transparently reconnects and stays usable.

## Kerberos authentication — August 2026 (v0.1.4 – v0.1.5)

External login with a **Kerberos ticket** (`KERBEROS5`) — no Oracle password on
the wire. Three credential modes (`ccache` / `keytab` / `password`), the identity
proven cryptographically inside Oracle's ANO negotiation, and it **composes with
Native Network Encryption** so you can authenticate with a ticket over an AES-256
channel. Linux / MIT krb5 only. See
[Authentication → Kerberos]({{ '/drivers/arrowttc/authentication/' | relative_url }}#kerberos-kerberos5).

## Native Network Encryption — July/August 2026 (v0.1.3, v0.1.7)

Encrypt the whole session without TCPS. After ACCEPT the driver runs Oracle's
Advanced Negotiation, agrees a Diffie-Hellman session key, and seals every DATA
payload with **AES-256** plus a **SHA-256 or SHA-1** integrity checksum — so a
server configured `SQLNET.ENCRYPTION_SERVER=REQUIRED` connects with no TCPS
needed. Only AES-256 and SHA-256/SHA-1 are offered; the legacy ciphers and MD5 are
never proposed. Set `encryption=required` (and/or `data_integrity=required`).

## The native TNS + TTC engine — foundational (v0.1.0)

From the start, ArrowTTC speaks Oracle Net + Two-Task Common directly over
TCP/TLS — no OCI, no Instant Client, no ODBC — with OpenSSL the only runtime
dependency beyond libc. Every protocol fact was verified by capture-diffing
against python-oracledb's thin mode:

- **Streaming Arrow fetch** — `SELECT` results decode straight into Arrow column
  buffers with no intermediate row objects; on a streaming SF1 `LINEITEM` extract
  over the server-required native-encryption channel, ~2.05× faster than
  python-oracledb's own Arrow path.
- **Configurable `NUMBER` mapping** — Oracle's unconstrained `NUMBER` maps to
  lossless `utf8` (default), `float64`, or `decimal128(P,S)`. See
  [Data types]({{ '/drivers/arrowttc/data-types/' | relative_url }}#number-mapping).
- **O5LOGON, proxy & change-password auth** — challenge-response login (12c
  PBKDF2-HMAC-SHA512 and 11g SHA-1 verifiers, chosen automatically), proxy
  `CONNECT THROUGH`, and change-password-at-connect, all with the password never
  in clear.
- **TCPS (TLS) & wallets** — `require` / `verify-ca` / `verify-full`, PEM CA
  bundles and python-oracledb-style `ewallet.pem` wallets, including mutual TLS.
- **Full ADBC 1.1.0 surface** — prepared statements + bound parameters, DML,
  transactions, bulk ingest (`create` / `append` / `replace` / `create_append`),
  LOB streaming (`CLOB` / `NCLOB` / `BLOB`), query cancellation, and
  catalog/metadata.

---

For the complete, dated, technical history see `CHANGELOG.md`. To
report an issue or request a licence, contact **sales@arpe.io** or visit
**https://www.arpe.io**.
