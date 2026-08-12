---
title: Troubleshooting
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 8
permalink: /drivers/arrowttc/troubleshooting/
---

# ArrowTTC — Troubleshooting

Common failures and their fixes, grouped by where they show up: connecting,
encryption/TLS, authenticating, the licence gate, and querying/ingesting. For the
option details referenced here, see [`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}).

- [Connection](#connection)
- [TLS / wallets](#tls--wallets)
- [Native Network Encryption](#native-network-encryption)
- [Authentication](#authentication)
- [Licence](#licence)
- [Query & ingest](#query--ingest)
- [Diagnostics: timing & debug traces](#diagnostics-timing--debug-traces)

---

## Connection

**Connect hangs, then fails.** Every candidate address was unreachable within
`login_timeout` — usually a firewall, a wrong port, or the listener not running.
Verify with `nc -vz <host> 1521`. Lower `login_timeout` to fail faster while
diagnosing.

**"specify either 'SID' or 'Service Name', not both".** A `sid` and a
`service_name` select different `CONNECT_DATA` forms on the wire, so they cannot
be combined. Pick one. In a `uri`, that also means a path segment (service name)
plus `?sid=` is rejected.

**EZConnect with an IPv6 address is rejected.** A bare IPv6 literal in a `Server`
descriptor has ambiguous colons. Give the host and `Port` as separate keywords,
or use the bracketed `[::1]` form in a `uri`.

**Connecting to ADB hangs / the listener drops the connection.** ADB's connect
descriptor is long and must spill into a following DATA packet; this was a driver
bug fixed in **v0.2.0** — upgrade. Also confirm you are using a wallet service
name (`<svc>_low.adb.oraclecloud.com`) on port **1522** with `ssl_mode=verify-full`.

**Queries fail against Oracle 21c / 23ai on describe.** 21c and 23ai change the
`DESCRIBE_INFO` (column-metadata) layout and it is not yet parsed. 19c is the
supported target — see [`COMPATIBILITY.md`]({{ '/drivers/arrowttc/compatibility/' | relative_url }}).

---

## TLS / wallets

**The connection is plaintext when I expected encryption.** `ssl_mode` defaults
to `disable`. Set `ssl_mode=require`/`verify-ca`/`verify-full`, or use
[Native Network Encryption](#native-network-encryption). O5LOGON keeps the
*password* off the wire either way, but SQL text and data are in clear until you
encrypt.

**"certificate verify failed" / hostname mismatch.** With `verify-ca`/`verify-full`
the certificate chain (and, for `verify-full`, the hostname) is checked against
`ssl_root_cert` or the system trust store. Point `ssl_root_cert` at the right CA
bundle, or use a wallet.

**The wallet won't load.** ArrowTTC reads **`ewallet.pem`** (client key + cert +
CAs in one PEM), the python-oracledb-thin layout — **not** Oracle's `cwallet.sso`.
Convert an SSO wallet with `orapki wallet pkcs12_to_pem -wallet <dir>`. An
encrypted key needs `wallet_password`.

**Windows: driver loads but TLS fails / DLL not found.** OpenSSL is not part of
Windows. The matching `libssl-3-x64.dll` / `libcrypto-3-x64.dll` must sit next to
`arrowttc_adbc_driver.dll` (or the driver must be static-linked), and the VC++
runtime (`vcruntime140.dll`, `msvcp140.dll`) must be present.

---

## Native Network Encryption

**`encryption=required` fails the connect.** The server declined to negotiate a
cipher ArrowTTC offers. ArrowTTC offers **only AES-256** with **SHA-256/SHA-1**
integrity; a server that requires DES/3DES/RC4 or MD5 will not connect. Adjust
`SQLNET.ENCRYPTION_TYPES_SERVER` / `CRYPTO_CHECKSUM_TYPES_SERVER` to include
AES256 / SHA256 (or SHA1).

**Data decodes as garbage / the stream desyncs mid-result over NNE.** This was
the SHA-1 integrity path, fixed in **v0.1.7**; and the out-of-band-break
requirement, removed in **v0.1.6** (the driver no longer needs `DISABLE_OOB=ON`).
Upgrade to ≥ v0.1.7.

---

## Authentication

**"ORA-01017: invalid username/password".** Credentials are wrong, or — for token
auth on ADB — the server-side setup is off. For OAuth2/Entra, the Application ID
URI must be a domain-qualified `https://` URI; the Azure default `api://<appId>`
makes ADB reject otherwise-valid tokens with a generic ORA-01017.

**Kerberos: "Kerberos unavailable" at connect.** The driver was built without MIT
krb5 (`ARROWTTC_ENABLE_GSSAPI=OFF`, or `AUTO` with no krb5 present). Kerberos is
**Linux / MIT krb5 only** — there is no Windows SSPI path. Rebuild with krb5
available.

**Kerberos: login fails / SPN mismatch.** Ensure a valid, **forwardable** TGT
(`kinit -f`; check with `klist`), that the database SPN matches `krb5_spn`
(e.g. `oracle/db.example.com`), and pick the right `krb5_cred_mode`
(`ccache`/`keytab`/`password`). Details in
[`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}#kerberos-kerberos5).

**OCI IAM / OAuth2 token: "login failed" though the token is valid.** The Oracle
identity must be mapped to a schema — `IDENTIFIED GLOBALLY AS 'IAM_PRINCIPAL_NAME=…'`
(OCI) or the Entra global-user mapping. Tokens also expire (~1 h); re-mint with
`oci iam db-token get` or refresh the Entra token. ADB requires the wallet
(mutual TLS) alongside the token.

**Proxy auth: "ORA-…: proxy not permitted".** The target must grant it:
`ALTER USER target GRANT CONNECT THROUGH proxyacct;`. Inside the session
`SESSION_USER` is the target and `PROXY_USER` the authenticating account.

---

## Licence

**Connection fails with an `ARROW_LIC_*` error.** The licence gate is **always
enforced** — a driver with no valid token refuses to connect. Supply one via
`arpeio.adbc.license_file` (a `.lic` path), `arpeio.adbc.license` (the blob
inline), the `ARPEIO_ADBC_LICENCE[_FILE]` environment variable, or an
`arpeio_adbc.lic` file next to the driver library. Full detail — including how to
read the current licence state with `arpeio.adbc.license.status` — is in
[`LICENSING.md`]({{ '/drivers/arrowttc/licensing/' | relative_url }}).

---

## Query & ingest

**`adbc_ingest(mode="create")` fails "already exists".** The target table exists.
Use `append`, `replace`, or `create_append`.

**Ingest into a different catalog is rejected.** An Oracle session reaches one
catalog only; `adbc.ingest.target_catalog` must be the connected service (or
unset). Use `target_db_schema` to target a schema.

**A `NUMBER` column comes back as strings.** That is the lossless default —
unconstrained `NUMBER` maps to `utf8`. For `float64` or a fixed decimal, set
`number_mapping=double` or `decimal:P,S`. See
[`DATA_TYPES.md`]({{ '/drivers/arrowttc/data-types/' | relative_url }}#number-mapping).

**A `TIMESTAMP WITH TIME ZONE` query fails loudly.** A region-id (named-zone)
TSTZ value is not resolved through DST rules; fixed-offset TSTZ/TSLTZ decode
normally. Select the value at a fixed offset, or `CAST` it, meanwhile.

**Cancelling a long query.** `Statement.Cancel` / `Connection.Cancel` send an
in-band TNS break from another thread and work over plaintext, Native Network
Encryption, and TCPS (TLS was closed in **v0.2.1**). The session is poisoned after
a cancel and transparently reconnects on the next statement.

---

## Diagnostics: timing & debug traces

Set these environment variables to see where time goes. Full list in
[`ENV_VARS.md`]({{ '/drivers/arrowttc/environment-variables/' | relative_url }}).

| Variable | What it shows |
|---|---|
| `ARPEIO_ADBC_READ_TIMING=1` | One `[TTC READ]` line per batch and a `[TTC READ SUMMARY]` at stream release, splitting an extract into wire (`recv`), decoder, and consumer — tells you whether the wire or the decoder is the bottleneck. |
| `ARPEIO_ADBC_ALLOCATOR=system` | Falls back from mimalloc to libc `malloc` for the Arrow column buffers (if mimalloc misbehaves, or for an A/B). |
| `ARROWTTC_NO_LOB_PREFETCH=1` | Disables the LOB length prefetch (an Oracle-specific escape hatch). |

## See also

- [`CONNECTION.md`]({{ '/drivers/arrowttc/connection/' | relative_url }}) · [`AUTHENTICATION.md`]({{ '/drivers/arrowttc/authentication/' | relative_url }}) · [`LICENSING.md`]({{ '/drivers/arrowttc/licensing/' | relative_url }})
- [`ENV_VARS.md`]({{ '/drivers/arrowttc/environment-variables/' | relative_url }}) — every option and environment variable
- [`COMPATIBILITY.md`]({{ '/drivers/arrowttc/compatibility/' | relative_url }}) — is your target actually supported?
