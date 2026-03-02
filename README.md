# CMS Theme Security Scan

GitHub Action for WordPress and Drupal theme/module security scanning. Runs PHPCS, Psalm, Semgrep, and Pa11y against your code and posts results as PR annotations, comments, and downloadable reports.

## Quick Start

```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on:
  pull_request:
    paths: ['**.php', '**.module', '**.theme', '**.html.twig', '**.css']

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Samson-Labs/cms-scanner-action@v1
        with:
          theme-path: './wp-content/themes/my-theme'
```

That's it. The action pulls a pre-built Docker image — no source checkout, no pip install, no Docker builds. Scans start in ~5 seconds.

## What Happens on a PR

1. **SARIF upload** — Findings appear as inline annotations on the PR diff (Code scanning tab).
2. **PR comment** — A summary table with finding counts by severity.
3. **Artifacts** — HTML report and JSON results uploaded as workflow artifacts.
4. **Exit code** — If `fail-on` is set, the workflow fails when findings meet that threshold.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `theme-path` | *(required)* | Path to theme/module directory relative to repo root |
| `profile` | `quick` | Scan profile: `quick`, `standard`, `a11y` |
| `cms` | `auto` | CMS type: `auto`, `wordpress`, `drupal` |
| `fail-on` | `none` | Severity threshold for non-zero exit code |
| `comment-on-pr` | `true` | Post scan summary comment on the PR |
| `upload-sarif` | `true` | Upload SARIF to GitHub Code Scanning |
| `sarif-level` | *(empty)* | Min severity for SARIF annotations (defers to `.scanner.yml`) |
| `scanner-version` | `latest` | Docker image tag (e.g., `latest`, `1.0.0`, `edge`) |
| `github-token` | `${{ github.token }}` | Token for PR comments |

## Outputs

| Output | Description |
|--------|-------------|
| `total-findings` | Total number of active findings |
| `exit-code` | Scanner exit code (0 = pass, 2 = threshold exceeded) |
| `report-path` | Path to the HTML report artifact |
| `results-json` | Path to `scan_results.json` |
| `sarif-path` | Path to the SARIF output file |

## Scan Profiles

| Profile | Tools | Time |
|---------|-------|------|
| `quick` | PHPCS | ~5s |
| `standard` | PHPCS + Psalm + Semgrep | ~20s |
| `a11y` | Semgrep a11y rules + Pa11y (axe-core) | ~30s |

> The `full` profile (sandbox + DAST tools) requires Docker Compose and is available via the CLI only.

## CMS Support

The scanner auto-detects WordPress (`style.css`) vs Drupal (`.info.yml`) from the target directory.

| Feature | WordPress | Drupal |
|---------|-----------|--------|
| PHPCS standards | WPCS 3.0 | Drupal Coder |
| Semgrep rules | WordPress-specific | Drupal-specific |
| Accessibility | axe-core via Pa11y | axe-core via Pa11y |

Force a CMS type with `cms: drupal` or `cms: wordpress`.

## Project Configuration

Drop a `.scanner.yml` in your theme/module root:

```yaml
profile: standard
fail_on: high
sarif:
  level: medium
ignore_rules:
  - WordPress.Files.FileName.InvalidClassFileName
```

CLI flags (action inputs) always override `.scanner.yml`.

## Manual Trigger with `/scan`

Add `issue_comment` to your workflow triggers to enable on-demand scanning via PR comments:

```
/scan                     # Quick profile, auto-detect CMS
/scan standard            # Standard profile
/scan standard drupal     # Standard profile, force Drupal
/scan a11y                # Accessibility audit
```

See [examples/scan-on-pr.yml](examples/scan-on-pr.yml) for a complete workflow with `/scan` support.

## Branch Protection

To require scans to pass before merging:

1. Set `fail-on: high` in your workflow or `.scanner.yml`
2. In repo settings: **Branches > Branch protection > Require status checks** — add "Theme Security Scan"

## Examples

### Minimal (security only)

```yaml
- uses: Samson-Labs/cms-scanner-action@v1
  with:
    theme-path: './my-theme'
```

### Standard scan with CI gating

```yaml
- uses: Samson-Labs/cms-scanner-action@v1
  with:
    theme-path: './my-theme'
    profile: 'standard'
    fail-on: 'high'
```

### Drupal module

```yaml
- uses: Samson-Labs/cms-scanner-action@v1
  with:
    theme-path: './modules/my-module'
    cms: 'drupal'
    profile: 'standard'
```

### Accessibility audit

```yaml
- uses: Samson-Labs/cms-scanner-action@v1
  with:
    theme-path: './my-theme'
    profile: 'a11y'
```

### Monorepo matrix

```yaml
strategy:
  matrix:
    target:
      - './wp-content/themes/theme-a'
      - './wp-content/themes/theme-b'
steps:
  - uses: actions/checkout@v4
  - uses: Samson-Labs/cms-scanner-action@v1
    with:
      theme-path: ${{ matrix.target }}
```

### Pin to a specific version

```yaml
- uses: Samson-Labs/cms-scanner-action@v1
  with:
    theme-path: './my-theme'
    scanner-version: '1.0.0'
```

## License

MIT
