# garmin-cli

[![CI](https://github.com/grunt3714-lgtm/garmin-cli/actions/workflows/ci.yml/badge.svg)](https://github.com/grunt3714-lgtm/garmin-cli/actions/workflows/ci.yml)
[![Release](https://github.com/grunt3714-lgtm/garmin-cli/actions/workflows/release.yml/badge.svg)](https://github.com/grunt3714-lgtm/garmin-cli/actions/workflows/release.yml)
[![Latest Release](https://img.shields.io/github/v/release/grunt3714-lgtm/garmin-cli?display_name=tag)](https://github.com/grunt3714-lgtm/garmin-cli/releases)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Fast Garmin Connect CLI for Linux, with working auth, sync, and health/training commands.

## Install

One-liner:

```bash
curl -fsSL https://github.com/grunt3714-lgtm/garmin-cli/releases/latest/download/garmin-linux-amd64.tar.gz | tar -xz && install -Dm755 garmin-linux-amd64 ~/.local/bin/garmin
```

Build from source:

```bash
cargo build --release && install -Dm755 target/release/garmin ~/.local/bin/garmin
```

## Quick examples

```bash
garmin auth login
garmin health summary
garmin health sleep --days 7
garmin health training-readiness --days 14
garmin health training-status --days 14
garmin sync run --backfill
```

## Authentication

```bash
# Login (opens browser for SSO)
garmin auth login

# Check status
garmin auth status

# Logout
garmin auth logout
```

## More examples

```bash
# Today
garmin health summary

# Recovery trends
garmin health sleep --days 7
garmin health hrv --date 2025-12-13
garmin health training-readiness --days 14

# Training state
garmin health training-status --days 14
garmin health race-predictions --date 2025-12-13
garmin health performance-summary --date 2025-12-13

# Other useful commands
garmin health steps --days 28
garmin health weight --from 2025-01-01 --to 2025-12-31
garmin health insights --days 28
```

## Activities

```bash
garmin activities list --limit 20
garmin activities get 21247810009
garmin activities download 21247810009 --type gpx --output activity.gpx
garmin activities upload activity.fit
```

## Sync

```bash
garmin sync run
garmin sync run --from 2025-01-01 --to 2025-12-31
garmin sync run --backfill
garmin sync run --health
garmin sync status
```

Synced data lives under:

```bash
~/.local/share/garmin/
```

DuckDB example:

```bash
export GARMIN_DATA="$HOME/.local/share/garmin"
duckdb -c "SELECT date, training_status, training_readiness, vo2max FROM '$GARMIN_DATA/performance_metrics/*.parquet' ORDER BY date DESC LIMIT 10"
```

## Profiles

```bash
garmin --profile work auth login
garmin --profile work health summary
GARMIN_PROFILE=work garmin health summary
```

## Need more?

Use built-in help:

```bash
garmin --help
garmin health --help
garmin sync run --help
```

## License

MIT
