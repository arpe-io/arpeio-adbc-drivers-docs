# Docs prototype — known gaps & follow-ups

Maintainer note for the `docs-site` branch. **Not published** (it lives outside
`docs/`, so GitHub Pages ignores it). Records what the first-version docs site
flagged rather than silently fixed, plus follow-ups for the production (Next.js)
site.

## How this site was built

- The site under `docs/` is a **Jekyll + Just the Docs** site, served natively by
  GitHub Pages (`remote_theme`, no Actions workflow). Content is portable
  Markdown, ready to move to the Next.js production site.
- The per-driver pages were **copied** from each driver's private source repo
  (`aetperf/ArrowTDS|ArrowFEBE|ArrowTTC`, under `docs/` + `WHATS_NEW.md`) as of
  the versions below, with front matter added and cross-links fixed.

| Driver | Version documented | Source |
|---|---|---|
| ArrowTDS | 0.5.24 | `aetperf/ArrowTDS` |
| ArrowFEBE | 0.3.8 | `aetperf/ArrowFEBE` |
| ArrowTTC | 0.2.8 | `aetperf/ArrowTTC` |

## Content decisions worth revisiting

- **`ENV_VARS.md` is only partly shared.** The `ARPEIO_ADBC_*` knobs are truly
  family-wide (surfaced on the shared `environment-variables.md` page); the bulk
  of each driver's env-vars doc is driver-specific (connection options,
  protocol knobs like `ARROWTTC_NO_LOB_PREFETCH`) and stays on the per-driver
  page. So there is no single fully-shared env-vars document.
- **What's New was refreshed** to the current version by pulling highlights from
  each driver's `CHANGELOG.md` (ArrowTTC lagged most — WHATS_NEW stopped at
  v0.2.3 while the driver is v0.2.8). Verify the added highlights read well.
- **Examples + Licensing** pages were included beyond the originally requested
  six docs, because they are user-facing and are the targets of most internal
  cross-links.

## Upstream inconsistencies (fix in the source repos, not here)

- **ArrowTDS version stamps are stale in the source repo:**
  `python/pyproject.toml` is `0.1.0` and `core/CMakeLists.txt` sets the
  shared-library `VERSION 0.1.0 / SOVERSION 0`, both out of sync with the driver
  define `0.5.24`. (ArrowFEBE sets `VERSION 0.3.8` correctly.)
- **Org naming drift** across repos: `arpe-io` (dist repo), `aetperf` (source
  repos), and placeholder `github.com/example/arrowtds` URLs in ArrowTDS's
  `pyproject.toml`. Install/download URLs on the site were normalized to
  `arpe-io/adbc-drivers`.
- **ArrowTDS docs asymmetry:** ArrowTDS ships `DRIVER_FAMILY_CONVENTION.md` and an
  excalidraw diagram that FEBE/TTC don't; FEBE ships a large `CODE_REVIEW.md`.
  None of these are published on the site.

## Capability differences to reconcile on a combined site

- **Query cancellation:** ArrowTDS returns `NOT_IMPLEMENTED`; ArrowFEBE implements
  real cancellation via out-of-band `CancelRequest`; ArrowTTC cancels over both
  plaintext and TLS. A combined "feature matrix" page would make this explicit.
- **ArrowTTC not-yet-done surface:** 21c/23ai `DESCRIBE_INFO` "not yet parsed";
  `GetStatistics`, `ExecuteSchema`, `GetParameterSchema` not implemented; several
  types "not yet mapped" (BFILE, region-id TSTZ, JSON/BOOLEAN).

## Platform status

- Binaries ship for **linux-x64** and **win-x64** only. macOS is staged but **not
  published** — the site advertises Linux | Windows and the Compatibility pages
  keep their honest "macOS — not yet published/validated" rows. Do not add macOS
  install instructions until binaries are released.

## Follow-ups for the production (Next.js) site

- **Docs sync.** These pages are copied from private repos; they will drift. Add a
  sync mechanism (a script that pulls the six+ docs per driver, or git submodules)
  so the site tracks the source docs automatically.
- **Per-version archive + version switcher.** This prototype documents one current
  version per driver. Production should support browsing docs for past releases
  (per-driver, since they version independently).
- **ArrowDRDA (IBM Db2)** is "coming soon"; the nav/structure leaves room to add it
  as a fourth driver.
- **Search + feature matrix.** Consider a cross-driver capability/type matrix and
  a unified example gallery.
