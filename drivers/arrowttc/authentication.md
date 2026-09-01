---
title: Authentication
layout: default
parent: ArrowTTC
grand_parent: Drivers
nav_order: 2
permalink: /drivers/arrowttc/authentication/
---

# Authentication

This is the user-facing guide to connecting.

## Password (O5LOGON)

The default. Supply `User ID` and `Password`; ArrowTTC performs Oracle's
O5LOGON challenge-response. The 12c (PBKDF2-HMAC-SHA512) and 11g (SHA-1)
verifiers are both supported and selected automatically from the server's
challenge.

```
Server=dbhost:1521/appdb;User ID=dbuser;Password=<password>
```

The password is **never sent in clear**, even without TLS: it is AES-encrypted
under a key derived from the server's challenge, and the driver verifies the
server's response in return (mutual auth), so a man-in-the-middle that accepts
any password is detected. This protects the *credential* — it does **not**
encrypt SQL text or result rows. For that, use TLS.

A wrong username or password fails fast with ORA-01017. Oracle does not
distinguish "no such user" from "wrong password", and neither does the error.

## Proxy authentication (connect through)

Authenticate as one account but run in another user's schema. The authenticating
account must have been granted access:

```sql
ALTER USER target GRANT CONNECT THROUGH proxyacct;
```

```
Server=dbhost:1521/appdb;User ID=proxyacct;Password=...;Proxy User=target
```

Inside the session, `SYS_CONTEXT('USERENV','SESSION_USER')` is `TARGET` and
`PROXY_USER` is `PROXYACCT`. An ungranted target, or a wrong password for the
authenticating account, is refused with ORA-01017.

## Change password at connect

Change the login password as part of the connect — the way in past an expired
one. The old password authenticates the request; the new one takes effect on
success.

```
Server=dbhost:1521/appdb;User ID=dbuser;Password=old;New Password=new
```

After a successful connect the new password authenticates and the old one no
longer does.

## TLS (TCPS)

Encrypts the whole connection. `SSL Mode` selects the level:

| Mode | Encrypts | Authenticates the server |
|---|:-:|:-:|
| `disable` (default) | no | — |
| `require` | yes | no |
| `verify-ca` | yes | chain only |
| `verify-full` | yes | chain **and** hostname |

Use `verify-full` for anything real. Trust comes from `SSL Root Cert` (a PEM CA
bundle) or a `Wallet Location` directory holding `ewallet.pem` (client key +
certificate + CAs in one PEM, the python-oracledb thin layout); `Wallet
Password` unlocks an encrypted wallet key. TCPS listeners conventionally use
port 2484 — set `Port` explicitly.

```
Server=dbhost:2484/appdb;User ID=dbuser;Password=<password>;
sslmode=verify-full;wallet_location=/etc/oracle/wallet
```

## Kerberos (KERBEROS5)

{: .since }
> Since ArrowTTC v0.1.4.

External authentication with a Kerberos ticket — no Oracle password. Set
`Auth Method=kerberos` (option `adbc.arrowttc.auth_method`). The identity is
proven inside Oracle's ANO negotiation (AP-REQ / AP-REP mutual auth + credential
forwarding); the O5LOGON
password phase is skipped. **Linux / MIT-krb5 only** — the driver must be built
with `ARROWTTC_ENABLE_GSSAPI` (default `AUTO`: on when MIT krb5 is present); a
build without it reports Kerberos unavailable at connect and stays "OpenSSL
only". Kerberos is the only auth method scoped to Linux: the driver also
targets Windows x64, where the `windows-build` CI job builds it under MSVC
and runs the hermetic unit suite. A secret-gated step can additionally run a
live TLS connect to a cloud database using whichever auth its configured
secrets supply (password, or a bearer token). Windows support is a build
target the CI job checks on every push, not (yet) a platform with the same
depth of field verification as Linux.

`Kerberos Cred Mode` (`krb5_cred_mode`) chooses how the ticket is obtained:

- **`ccache`** (default) — the ambient `kinit` ticket (`KRB5CCNAME`, or a
  per-connection cache via `Kerberos Cache` / `krb5_ccache`). The ticket must be
  forwardable (`kinit -f`).
- **`keytab`** — a keytab file (`Kerberos Keytab` / `krb5_keytab`) plus
  `Kerberos Principal`; for service accounts and CI.
- **`password`** — `Kerberos Principal` + `Kerberos Password`, obtained through
  libkrb5 into a private in-memory cache (never written to disk).

`Kerberos SPN` (`krb5_spn`) overrides the target service principal, which
otherwise defaults to `oracle/<host>`; `Kerberos Realm` supplies the default
realm when the principal carries none.

```
Server=dbhost:1521/appdb;Auth Method=kerberos;Kerberos Cred Mode=ccache;
Kerberos SPN=oracle/dbhost.example.com
```

No username or password is needed — the DB user is mapped from the Kerberos
principal (`CREATE USER … IDENTIFIED EXTERNALLY AS '<principal>'`). Kerberos
composes with native network encryption — add `Encryption=required` (and/or
`Data Integrity=required`) to authenticate with a ticket over an AES-256 /
SHA-256 channel.

## OCI IAM token (Oracle Cloud)

{: .since }
> Since ArrowTTC v0.2.0.

`Auth Method=token` logs in to Oracle Cloud with an **OCI IAM database token**
instead of a password. `token_location` is a directory holding the token
(`token`) and its PEM private key (`oci_db_key.pem`) — exactly what
`oci iam db-token get` writes to `~/.oci/db-token`. It is an ADBC option
(`adbc.arrowttc.token_location`), set via `AdbcDatabaseSetOption`, not a
connection-string keyword. No username or password is
set: the driver sends the token as `AUTH_TOKEN` and proves possession with
`AUTH_HEADER` (a `date` / `(request-target)` / `host` string naming the service)
and `AUTH_SIGNATURE` (RSA-SHA256 of that header, base64 — the OCI HTTP-signature
scheme). The database validates the signature against the public key inside the
token, so a wrong signature comes back as an ORA error.

```
Server=adb.<region>.oraclecloud.com:1522/<svc>_low.adb.oraclecloud.com;
Auth Method=token;sslmode=verify-full;Wallet Location=/home/you/wallet;Wallet Password=...
```

with `adbc.arrowttc.token_location=/home/you/.oci/db-token` set as an ADBC
option alongside it (the same way the OAuth2 token options below are set).

The database must have IAM authentication enabled
(`DBMS_CLOUD_ADMIN.ENABLE_EXTERNAL_AUTHENTICATION(type => 'OCI_IAM')`) and the
OCI identity mapped to a schema (`CREATE USER … IDENTIFIED GLOBALLY AS
'IAM_PRINCIPAL_NAME=<user>'`). ADB requires mutual TLS, so pair it with the
instance wallet. Tokens expire (~1 h); re-mint with `oci iam db-token get`.

## OAuth2 / Microsoft Entra ID bearer token

{: .since }
> Since ArrowTTC v0.2.2.

External authentication with a Microsoft Entra ID (Azure AD) access token —
no Oracle password, no OCI private key. Set `adbc.arrowttc.access_token` to
the bearer JWT inline, or `adbc.arrowttc.token_file` to a path holding just
the JWT; if both are set, the inline `access_token` wins. These are ADBC
options, not connection-string keywords — set them via `AdbcDatabaseSetOption`
(or the equivalent in the language binding), not inside the `Server=...;...`
string.

Setting either option **auto-selects** the method, so `Auth Method` does not
need to be set explicitly; `Auth Method=oauth2` (`adbc.arrowttc.auth_method`)
also works and is required if you want the driver to fail fast when neither
token option ends up set. An explicit `Auth Method` naming anything other than
`oauth2` alongside `access_token`/`token_file` is rejected as a configuration
conflict rather than silently overridden.

No username is configured: Oracle derives the global user from the token's
`upn` claim.

On the wire this is an external `AUTH_PHASE_TWO` with `auth_mode = LOGON`
(`0x1`) and a single `AUTH_TOKEN` key-value pair carrying the JWT — no
O5LOGON verifier round, and, unlike the [OCI IAM token](#oci-iam-token-oracle-cloud)
path above, **no proof of possession**: no `AUTH_HEADER`/`AUTH_SIGNATURE`
pair and the `IAM_TOKEN` bit is never set in `auth_mode`. A bearer token
carries no private key to sign with, so there is nothing to prove beyond
presenting the token itself.

```
Server=adb.<region>.oraclecloud.com:1522/<svc>_low.adb.oraclecloud.com;
sslmode=verify-full;Wallet Location=/home/you/wallet;Wallet Password=...
```

(with `adbc.arrowttc.access_token` or `adbc.arrowttc.token_file` set
separately as an ADBC option — see above).

### Autonomous Database setup lesson: the Application ID URI

On Autonomous Database, the Entra app registration's **Application ID URI
must be a domain-qualified `https://` URI** — e.g.
`https://<tenant>.onmicrosoft.com/<appId>` — **not** Azure's default
`api://<appId>`. The `application_id_uri` passed to
`DBMS_CLOUD_ADMIN.ENABLE_EXTERNAL_AUTHENTICATION` must match it exactly. A
non-`https` Application ID URI makes the ADB reject otherwise-valid tokens
with a generic `ORA-01017` — Oracle does not surface the real reason, so a
misconfigured URI is easy to mistake for a bad token or a blocked account.

Verified live: connecting with an Entra token maps to an Autonomous Database
global user with `AUTHENTICATION_METHOD = TOKEN_GLOBAL` — a user created
`IDENTIFIED GLOBALLY AS 'AZURE_USER=<upn>'`.

## Not supported

- **NTS/NTLM and Windows SSPI.** Windows integrated auth needs SSPI, which
  ArrowTTC does not implement on any platform. This does not affect Kerberos
  ticket auth on Linux (MIT-krb5, above) or any of the non-Kerberos methods on
  Windows (password, TLS, OCI token, OAuth2/Entra).
