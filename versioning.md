---
title: Versioning
layout: default
nav_order: 4
permalink: /versioning/
---

# Versioning
{: .no_toc }

Each driver in the family is versioned and released **independently**. This page
explains the model, the current versions, and how this documentation tracks
releases.
{: .fs-5 .fw-300 }

1. TOC
{:toc}

---

## Current versions

| Driver | Database | Current version | What's new |
|---|---|---|---|
| [ArrowTDS]({{ '/drivers/arrowtds/' | relative_url }}) | Microsoft SQL Server | **0.5.25** | [Release highlights]({{ '/drivers/arrowtds/whats-new/' | relative_url }}) |
| [ArrowFEBE]({{ '/drivers/arrowfebe/' | relative_url }}) | PostgreSQL | **0.3.8** | [Release highlights]({{ '/drivers/arrowfebe/whats-new/' | relative_url }}) |
| [ArrowTTC]({{ '/drivers/arrowttc/' | relative_url }}) | Oracle | **0.2.8** | [Release highlights]({{ '/drivers/arrowttc/whats-new/' | relative_url }}) |

Run the installer's `--list` for the authoritative, always-current set:

```sh
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh | sh -s -- --list
```

## The version model

- **Semantic versioning.** Each driver tags releases `vMAJOR.MINOR.PATCH`
  (e.g. `v0.5.24`). Drivers version on their own cadence — ArrowTDS being at
  0.5.x while ArrowTTC is at 0.2.x is expected, not a mismatch.
- **One source of truth.** The authoritative version is a `#define` in the
  driver's C header (`ARROW<CODE>_DRIVER_VERSION` in
  `adbc_driver_arrow<code>.h`). The build system, the ADBC driver manifest, and
  the value reported through ADBC `GetInfo` all derive from that define, so they
  cannot drift apart.
- **Release tags map to download assets.** Binaries are published to this repo's
  Releases under a per-driver tag, `arrow<code>-v<X.Y.Z>` (e.g.
  `arrowttc-v0.2.8`), each with a `SHA256SUMS`. The installer maps a load name +
  version to the matching asset.
- **Pin a version** at install time with `--version X.Y.Z` (`-Version` on
  Windows); the default is `latest`.

## Handling version drift in these docs

This documentation site is a **content prototype**: it documents **one current
version per driver** rather than keeping a browsable archive of every past
release. Version-specific detail is conveyed three ways, consistently:

1. **A version badge** on each driver's landing page states the release these
   docs describe.
2. **Inline `Since vX.Y.Z` notes** flag options and features that were added or
   changed recently, so you can tell what your installed build supports. For
   example, ArrowTTC's OCI IAM token auth is marked *since v0.2.0* and its
   OAuth2 / Entra ID login *since v0.2.2*.
3. **A per-driver _What's New_ page** is the reader-facing version diff — what
   changed and when — so if you're on an older build you can see what upgrading
   gains you. The full engineering history lives in each driver's `CHANGELOG`.

{: .note }
> A full **per-version documentation archive** with a version switcher is planned
> for the production documentation site, not this prototype. Because the drivers
> version quickly and independently, snapshotting the whole site per release
> isn't a good fit here — the current-version + _What's New_ approach keeps the
> content honest and easy to follow.
