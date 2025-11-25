# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository provides three GitHub Actions for working with Quarto:

1. **setup** - Installs Quarto on GitHub Actions runners
2. **render** - Renders a Quarto project
3. **publish** - Publishes a Quarto project to various services

The actions are implemented as composite actions (using shell scripts) rather than JavaScript or Docker container actions.

## Architecture

### Action Structure

Each action lives in its own directory with:
- `action.yml` - Action definition with inputs/outputs
- `README.md` - User-facing documentation
- Supporting scripts (e.g., `install-quarto-windows.ps1` for setup action)

All actions use `using: 'composite'` with shell steps, making them OS-agnostic.

### Platform-Specific Installation (setup action)

The setup action handles three platforms differently:

- **Linux**: Downloads `.deb` file (AMD64 or ARM64), installs with `apt`
- **macOS**: Downloads `.pkg` file, installs with `installer`
- **Windows**: Uses Scoop package manager via `install-quarto-windows.ps1` (workaround because MSI installation doesn't work in GitHub Actions)

Version handling:
- `"release"` (default) - Fetches latest stable from `quarto.org/docs/download/_download.json`
- `"pre-release"` - Fetches latest dev from `quarto.org/docs/download/_prerelease.json`
- Specific version (e.g., `"1.3.450"`) - Downloads that exact release

### Quarto Binary Detection

The setup action determines the correct binary based on `$RUNNER_OS` and `$RUNNER_ARCH`, setting `BUNDLE_EXT` environment variable to one of:
- `linux-amd64.deb`
- `linux-arm64.deb`
- `macos.pkg`
- `win.msi`

### Render and Publish Actions

Both actions:
- Accept a `path` input for subdirectory rendering/publishing
- Set `QUARTO_PRINT_STACK: true` for better debugging
- Use bash shells exclusively (even on Windows, leveraging Git Bash)

The publish action:
- Configures git identity for gh-pages commits
- Handles multiple publishing targets (gh-pages, Netlify, Quarto Pub, RStudio Connect)
- Supports `--no-render` flag to skip rendering before publishing
- Has both `to` and `target` inputs (aliases) for backward compatibility

## Testing

Run tests with:
```bash
gh workflow run test.yaml
```

The test workflow (`.github/workflows/test.yaml`):
- Tests on multiple OS/architecture combinations (macOS, Windows, Ubuntu AMD64, Ubuntu ARM)
- Matrix tests both `release` and `pre-release` versions by default
- Supports manual dispatch to test specific versions or enable TinyTeX
- Uses `uses: ./setup` to test the action directly from the working tree

## Version Management

This repository follows GitHub's recommended release management:
- Major versions (`v2`, `v1`) are moving tags that point to latest minor/patch
- Specific versions (`v2.0.1`) are immutable
- Users should use `quarto-dev/quarto-actions/setup@v2` for automatic updates
- Breaking changes only come with major version bumps

## Windows-Specific Considerations

Windows installation uses Scoop because MSI installation doesn't work reliably in GitHub Actions. The `install-quarto-windows.ps1` script:
- Installs Scoop if not present
- Adds the `r-bucket` (maintained by Chris at github.com/cderv/r-bucket.git)
- Installs Quarto or specific versions through Scoop
- Adds Scoop shims to `$GITHUB_PATH`

## Common Issues

- **TinyTeX PATH**: After installing TinyTeX, the action adds its binary directory to `$GITHUB_PATH` differently per platform (Linux: `$HOME/bin`, macOS: finds via `~/Library/TinyTeX`, Windows: finds via `$APPDATA/TinyTeX`)
- **Chromium crashes**: Past issues with Chromium when rendering PDFs (see commented-out Chrome installation steps in render action)
- **Output directory detection**: Render action attempts to detect `_book` or `_site` output directories
