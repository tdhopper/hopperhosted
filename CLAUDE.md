# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a MkDocs documentation site for HopperHosted services (https://hopperhosted.com). The site provides a dashboard with links to various self-hosted services (Pi-hole, Synology, Jellyfin) and operational runbooks.

## Development Commands

### Build and Serve
```bash
# Install dependencies using uv
uv sync

# Serve the documentation site locally with live reload
uv run mkdocs serve

# Build the static site (outputs to site/ directory)
uv run mkdocs build

# Build with strict mode (fail on warnings)
uv run mkdocs build --strict
```

### Preview
The local development server runs at `http://127.0.0.1:8000/` with automatic reloading when files change.

## Architecture

### Configuration
- **mkdocs.yml**: Main configuration file defining:
  - Site metadata (name, URL)
  - Material theme with slate color scheme
  - Navigation structure (Dashboard, Apps, Ops sections)
  - Enabled plugins: search, minify
  - Markdown extensions for admonitions, code blocks, tabs

### Content Structure
- **docs/**: Contains all markdown documentation files
  - `index.md`: Dashboard with links to hosted services
  - `apps.md`: Apps documentation (referenced in nav but not yet created)
  - `ops/runbooks.md`: Operational runbooks (referenced in nav but not yet created)

### Dependencies
Managed via `pyproject.toml` with uv:
- `mkdocs-material`: Material Design theme
- `mkdocs-minify-plugin`: HTML/CSS/JS minification

### Output
- **site/**: Generated static site (git-ignored, created by `mkdocs build`)

## Adding New Pages

1. Create markdown file in `docs/` directory
2. Add entry to `nav` section in `mkdocs.yml`
3. Use Material theme features: admonitions, code blocks with copy button, tabs, details/summary
