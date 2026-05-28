# IntentPHP Guard Action

A composite GitHub Action that runs [IntentPHP Guard](https://github.com/drnasin/intentphp) security scans and produces inline GitHub annotations on pull requests.

- Guard package: [intentphp/guard](https://packagist.org/packages/intentphp/guard)
- Source & docs: [github.com/drnasin/intentphp](https://github.com/drnasin/intentphp)

## Compatibility

| Action ref | Works with Guard |
|---|---|
| `@v2` / `@v2.0.0` | v2.0+ (recommended) |
| `@v1` / `@v1.1.0` | v1.1.x |
| `@v1.0.0` | v1.0.x |

The action just runs `php artisan guard:scan` with composed flags; all CLI flags used here (`--format`, `--severity`, `--baseline`, `--strict`, `--changed`, `--base`) are stable across the entire `1.x`/`2.x` line, so the same action ref works for any Guard release within that range. Pin to a major (`@v2`) to get bug-fix updates automatically.

## Intent Spec Support (Guard v1.1+)

The optional intent spec (`intent/intent.yaml`) is supported automatically. When present, intent-aware checks (`intent-auth`, `intent-mass-assignment`, drift) run alongside the built-in checks. When absent, Guard behaves exactly as before — no extra configuration needed.

## Upgrading to Guard v2.0

Guard v2.0 changes the fingerprint identity scheme for `route-authorization`, `mass-assignment`, and `dangerous-query-input` findings (methods are normalized, the dangerous-query identity moved off the raw snippet, paths are boundary-anchored). Existing baseline entries will no longer match — and because this action defaults to `baseline=true strict=true`, the next CI run will **exit 2** until you re-baseline.

After bumping `intentphp/guard` to `^2.0` in your `composer.json`:

```bash
composer update intentphp/guard
php artisan guard:baseline
git add storage/guard/baseline.json
git commit -m "Re-baseline for Guard v2.0"
```

Guard v2.0 also introduces new finding types (AST-based multi-line / interpolation / variable-indirection detection, plus a `scan/parse-error` MEDIUM finding for unparseable files). Real new HIGH findings will surface in the first run after upgrade; the re-baseline above suppresses them, but reviewing them is recommended.

## Prerequisites

Your Laravel project must have `intentphp/guard` installed:

```bash
composer require intentphp/guard --dev
```

In CI, make sure dependencies are installed (e.g. `composer install`) before running the action.

## Quick Start (Pull Request)

```yaml
name: Guard

on:
  pull_request:
    branches: [main]

jobs:
  guard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # required for incremental scanning

      - uses: shivammathur/setup-php@v2
        with:
          php-version: "8.2"

      - run: composer install --no-interaction --prefer-dist

      - uses: drnasin/intentphp-guard-action@v1
        with:
          changed: "true"
          base: "origin/main"
```

This will scan only files changed in the PR and annotate new security findings directly on the diff.

> **Note:** By default the action runs with `baseline=true` and `strict=true`, so it expects `storage/guard/baseline.json` to be committed. If you're trying it for the first time, either create a baseline or set `strict: "false"`.

## Full Scan (Push)

```yaml
name: Guard (full)

on:
  push:
    branches: [main]

jobs:
  guard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: shivammathur/setup-php@v2
        with:
          php-version: "8.2"

      - run: composer install --no-interaction --prefer-dist

      - uses: drnasin/intentphp-guard-action@v1
        with:
          changed: "false"
```

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `working-directory` | `.` | Path to the Laravel application root |
| `severity` | `high` | Filter by severity (`high` or `all`). See note below. |
| `baseline` | `true` | Suppress findings that match the saved baseline |
| `strict` | `true` | Exit 2 if baseline file is missing (requires `baseline=true`) |
| `changed` | `false` | Scan only files changed vs base branch |
| `base` | `origin/main` | Git base ref for incremental scanning |
| `format` | `github` | Output format (`github`, `json`, or `md`) |
| `extra-args` | `""` | Additional arguments appended to `guard:scan`. Whitespace-split; quoting is not supported. |

> **Note:** Guard v1.1 introduces additional MEDIUM-severity findings (for example, intent-declared public routes without auth middleware). To include those in CI failures, set `severity: "all"`.

## Baseline Workflow

Guard works best when you commit a baseline of existing findings and only fail CI on **new** issues.

**1. Create a baseline locally:**

```bash
php artisan guard:baseline
git add storage/guard/baseline.json
git commit -m "Add Guard baseline"
git push
```

**2. CI will now only report new findings:**

The action runs with `--baseline --strict` by default, which means:
- Known findings from the baseline are suppressed
- Only new security issues cause the build to fail
- If the baseline file is missing, the build fails (strict mode)

To disable strict mode, set `strict: "false"`.

## Incremental Scanning

When `changed: "true"` is set, Guard only scans files that differ from the base branch. This makes PR scans fast and focused.

**Why `fetch-depth: 0` matters:** GitHub Actions checks out only the latest commit by default (`fetch-depth: 1`). Incremental scanning needs the full history to compute the diff against the base branch. Always set `fetch-depth: 0` when using `changed: "true"`.

## Extra Arguments

Use `extra-args` to pass additional flags to `guard:scan`:

```yaml
- uses: drnasin/intentphp-guard-action@v1
  with:
    extra-args: "--ai --no-cache"
```

Arguments are split on whitespace and appended to the command array. Quoting inside the string is **not** supported — each whitespace-separated token becomes a separate argument. Only use simple flags and values:

```yaml
# Good — each token is a separate argument
extra-args: "--ai --no-cache"

# Good — flag with value (no spaces in the value)
extra-args: "--changed-since=v1.0.0"

# Bad — quotes are passed literally, not interpreted
extra-args: "--changed-since='v 1.0.0'"
```

## Runner Compatibility

This action uses a bash composite step and is tested on `ubuntu-latest`. It should work on any GitHub-hosted Linux runner. macOS runners (`macos-latest`) also work but are slower and more expensive. Windows runners are not supported/tested.

**Recommendation:** Use `runs-on: ubuntu-latest` for all Guard workflows.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `artisan file not found` | Wrong working directory | Set `working-directory` to your Laravel app root |
| `guard:scan not found` | Package not installed | Run `composer require intentphp/guard --dev` |
| Exit code 1 | HIGH severity findings detected | Review the annotations and fix the issues |
| Exit code 2 | Baseline file missing (strict mode) | Run `php artisan guard:baseline` and commit the file, or set `strict: "false"` |
| No annotations on PR | Wrong format | Ensure `format` is `github` (the default) |
| Incremental scan misses files | Shallow clone | Set `fetch-depth: 0` on `actions/checkout` |
| `origin/main not found` | Different default branch name | Set `base` to your branch (e.g. `origin/master`) |

## License

MIT
