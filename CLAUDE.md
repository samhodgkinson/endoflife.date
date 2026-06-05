# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

endoflife.date is a Jekyll-based static site that tracks end-of-life dates and release lifecycles for software products, OSes, frameworks, and devices. Each product is a single Markdown file with YAML frontmatter. The site also auto-generates a REST API (JSON files) and iCalendar files from that same frontmatter data.

## Commands

```sh
# Install Ruby dependencies
bundle install

# Run the development server at http://localhost:4000
bundle exec jekyll serve --host localhost --port 4000

# Build the site (also validates all product files)
bundle exec jekyll build --trace

# Build with URL validation (slow — checks every URL in every product file)
MUST_CHECK_URLS=true bundle exec jekyll build
```

There is no separate test runner. Validation runs as part of `jekyll build` via `_plugins/product-data-validator.rb`. A build failure means a product file has invalid data. The `--trace` flag gives full Ruby backtraces on errors.

**Docker alternative (no Ruby install needed):**
```sh
docker run --rm -v "$PWD:/srv/jekyll:Z" -p 4000:4000 jekyll/jekyll:4 jekyll serve --port 4000
```

## Architecture

The data pipeline flows like this:

1. `products/*.md` — YAML frontmatter is the source of truth for all release data.
2. Jekyll build triggers custom plugins in `_plugins/`:
   - `product-data-validator.rb` — validates mandatory fields, column config, URL format; fails the build on errors.
   - `product-data-enricher.rb` — adds derived metadata (icon slugs, identifier URLs, etc.).
   - `create-json-files.rb` — writes `api/<product>/<cycle>.json` for the public REST API.
   - `create-icalendar-files.rb` — writes `calendar/<product>.ics`.
   - `end-of-life-filters.rb` — provides Liquid filters used in templates (`liquify`, `parse_uri`, `drop_zero_patch`, etc.).
   - `generate-tag-pages.rb` — auto-generates `/tags/` index and per-tag pages.
   - `identifier-to-url.rb` — converts PURLs and repology identifiers to clickable URLs.
3. `_layouts/product.html` renders each product page using the Just the Docs theme.
4. `_includes/table.html` renders the release cycle table.

The `_data/release-data/` directory is a git submodule pointing to the [release-data](https://github.com/endoflife-date/release-data/) repository, which stores auto-fetched release version data. The deployment script (`_auto/deploy.sh`) updates this submodule and runs `_data/release-data/latest.py` before building — this is what keeps `latest` and `latestReleaseDate` fields current automatically.

## Product file schema

Product files live in `products/<name>.md` (lowercase, hyphen-separated). The YAML frontmatter drives everything — the HTML page, the API output, and the calendar files. `_config.yml` sets the defaults for most boolean columns.

**Mandatory fields:**
```yaml
title: Product Name
category: os|database|app|lang|framework|device|service|server-app
permalink: /product-slug
releases:
  - releaseCycle: "1.2"   # always quoted; newest first
    eol: 2026-01-01       # date or true/false
    latest: "1.2.3"       # always quoted
```

**Key optional fields:**
```yaml
tags: amazon linux-distribution          # space-separated, alphabetically ordered
iconSlug: simpleicons-slug               # from https://simpleicons.org/
releasePolicyLink: https://...
changelogTemplate: "https://example.com/__RELEASE_CYCLE__/__LATEST__"
releaseLabel: "Product __RELEASE_CYCLE__ (__CODENAME__)"
LTSLabel: "<abbr title='Long Term Support'>LTS</abbr>"

# Column visibility (true/false or a string to override the column header label)
eoasColumn: true           # End of Active Support (default: false)
eolColumn: true            # End of Life (default: true)
eoesColumn: true           # End of Extended Support (default: false)
releaseColumn: true        # Latest version (default: true)
releaseDateColumn: true    # Release date (default: false)
discontinuedColumn: true   # For devices (default: false)

# Auto-update from upstream sources
auto:
  methods:
    - git: https://github.com/org/repo.git
    - npm: package-name
    - docker_hub: owner/image
    - maven: groupId/artifactId
    - distrowatch: distro-id

# Package identifiers for SBOM tooling
identifiers:
  - repology: package-name
  - purl: pkg:npm/package-name
```

**Per-release fields:**
```yaml
releases:
  - releaseCycle: "1.2"
    releaseDate: 2024-01-01      # required if releaseDateColumn is true
    lts: true                    # or a date when LTS status begins
    eoas: 2025-01-01             # required if eoasColumn is true
    eol: 2026-01-01
    eoes: 2027-01-01
    discontinued: true
    latest: "1.2.3"
    latestReleaseDate: 2024-06-01
    link: https://...            # overrides changelogTemplate for this cycle
    codename: focal
```

Dates must be unquoted valid YAML dates (`YYYY-MM-DD`). Boolean fields accept `true`/`false`. Do not add a `v` prefix to `releaseCycle` or quote dates.

## Tags rules

- Must match `[a-z0-9\-]+`, alphabetically ordered, space-separated.
- Must use singular form (e.g. `web-server` not `web-servers`).
- Do not invent new tags; only use existing ones from `/tags/`.
- New tags require an issue and must be applied to all applicable products in one PR.

## Validation

The JSON Schema for product files is at `product-schema.json`. In VS Code, install the `redhat.vscode-yaml` extension and add:
```json
"files.associations": { "**/products/*.md": "yaml" },
"yaml.schemas": { "../product-schema.json": "products/*.md" }
```

## CI

- `.github/workflows/check-links.yml` — runs weekly with `MUST_CHECK_URLS=true` to validate all URLs.
- `.github/workflows/auto-merge-release-updates.yml` — triggered by Dependabot PRs; runs `latest.py` and auto-merges submodule bumps.
