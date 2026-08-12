# Arpeio ADBC drivers — documentation

Source for the Arpeio ADBC drivers documentation website (a **prototype**;
content is portable Markdown, ready to move to the production Next.js site).

Built with [Jekyll](https://jekyllrb.com/) + the
[Just the Docs](https://just-the-docs.com/) theme and served by **GitHub Pages**
from this repo's root via `remote_theme` — no Actions workflow required.

## What's here

- `index.md` — landing page and driver-selection table
- `install.md` — installation guide (mirrors the drivers repo README)
- `drivers/` — one section per driver (**ArrowTDS**, **ArrowFEBE**, **ArrowTTC**),
  each with connection, authentication, data types, compatibility, examples,
  environment variables, licensing, troubleshooting, and what's new
- `environment-variables.md` — shared `ARPEIO_ADBC_*` reference
- `versioning.md` — version model and current versions
- `DOCS_GAPS.md` — maintainer notes / follow-ups (not published)

## Local preview

```sh
bundle install
# Uses the just-the-docs gem theme locally (no remote-theme fetch):
bundle exec jekyll serve --config _config.yml,_config_local.yml
# → http://localhost:4000
```

> On some systems a bundled assembler shadows the toolchain and breaks native
> gem builds; prefix the commands with `PATH="/usr/bin:/bin:$PATH"` if
> `bundle install` fails compiling `json`/`ffi`.

## Publishing

GitHub Pages: **Settings → Pages → Deploy from a branch → Branch: `main`,
Folder: `/ (root)`**. The site is served at
<https://arpe-io.github.io/arpeio-adbc-drivers-docs/>.

The driver **binaries** and installers live in
[`arpe-io/adbc-drivers`](https://github.com/arpe-io/adbc-drivers). Docs are
MIT-licensed; the driver binaries are proprietary and licence-gated at runtime —
contact <sales@arpe.io>.
