source "https://rubygems.org"

# Local preview of the GitHub Pages site.
#
#   bundle install
#   bundle exec jekyll serve
#
# GitHub Pages itself builds this site from the `remote_theme` in _config.yml
# (jekyll-remote-theme is on the Pages allow-list), so no Actions workflow is
# needed. These gems just make `jekyll serve` work locally.

gem "jekyll", "~> 4.3"
gem "just-the-docs", "0.10.1"
gem "jekyll-seo-tag"

# Windows / JRuby helpers (harmless elsewhere)
gem "webrick", "~> 1.8"

# NOTE: GitHub Pages builds this site from the `remote_theme` in _config.yml and
# supplies `jekyll-remote-theme` itself, so it is intentionally NOT listed here
# (its native `openssl` dependency is awkward to build locally). For local
# preview, use the bundled `just-the-docs` gem theme via the local config
# override:
#
#   bundle exec jekyll serve --config _config.yml,_config_local.yml
