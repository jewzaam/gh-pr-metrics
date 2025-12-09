# Migration Guide: Old CLI → New CLI

## Overview

The `gh-pr-metrics` tool has been refactored from a monolithic single-command CLI to a clean subcommand architecture. This guide helps users transition from the old CLI to the new one.

**Breaking Change**: The old CLI syntax is no longer supported. All functionality is preserved but organized into three independent commands.

## Command Mapping

### Basic Fetch and CSV Generation

**Old (Monolithic)**:
```bash
gh-pr-metrics --owner ansible --repo awx --start 2024-01-01 --output metrics.csv
```

**New (Subcommands)**:
```bash
# Step 1: Fetch data
gh-pr-metrics fetch --owner ansible --repo awx --start 2024-01-01

# Step 2: Generate CSV
gh-pr-metrics csv --owner ansible --repo awx --output metrics.csv
```

### Initialize Repository Tracking

**Old**:
```bash
gh-pr-metrics --init --owner ansible --start 2024-01-01 --output "data/{owner}-{repo}.csv"
```

**New**:
```bash
gh-pr-metrics init --owner ansible --start 2024-01-01 --output "data/{owner}-{repo}.csv"
```

**Changes**:
- `--init` flag → `init` subcommand
- Functionality identical

### Initialize Single Repository

**Old**:
```bash
gh-pr-metrics --init --owner ansible --repo awx --start 2024-01-01 --output data/awx.csv
```

**New**:
```bash
gh-pr-metrics init --owner ansible --repo awx --start 2024-01-01 --output data/awx.csv
```

### Update Single Repository

**Old**:
```bash
gh-pr-metrics --owner ansible --repo awx --update
```

**New**:
```bash
# Step 1: Fetch new data
gh-pr-metrics fetch --owner ansible --repo awx --update

# Step 2: Regenerate CSV
gh-pr-metrics csv --owner ansible --repo awx
```

**Key Difference**: Two-step process separates fetch (API calls) from CSV generation (computation).

### Update All Tracked Repositories

**Old**:
```bash
gh-pr-metrics --update-all
```

**New**:
```bash
# Step 1: Fetch all tracked repos
gh-pr-metrics fetch --all

# Step 2: Regenerate all CSVs
gh-pr-metrics csv --all
```

**Changes**:
- `--update-all` flag → `--all` flag in both `fetch` and `csv`
- Two-step process for better control

### Update All with Wait for Rate Limit

**Old**:
```bash
gh-pr-metrics --update-all --wait
```

**New**:
```bash
# Fetch with wait
gh-pr-metrics fetch --all --wait

# Generate CSVs
gh-pr-metrics csv --all
```

### Single PR Update

**Old**:
```bash
gh-pr-metrics --pr 12345 --owner ansible --repo awx --output metrics.csv
```

**New**:
```bash
# Step 1: Fetch the PR
gh-pr-metrics fetch --pr 12345 --owner ansible --repo awx

# Step 2: Regenerate CSV
gh-pr-metrics csv --owner ansible --repo awx --output metrics.csv
```

### Auto-detect Repository from Git

**Old**:
```bash
# From within a git repository
gh-pr-metrics --start 2024-01-01 --output metrics.csv
```

**New**:
```bash
# Auto-detect still works in fetch command
gh-pr-metrics fetch --start 2024-01-01

# Need to specify owner/repo for CSV or rely on state
gh-pr-metrics csv --owner ansible --repo awx --output metrics.csv
```

**Note**: CSV command requires explicit owner/repo since it doesn't interact with git.

## New Workflows

### Regenerate CSV Without Re-fetching

**Old**: Not possible (would re-fetch data)

**New**:
```bash
# Just regenerate CSV from cached JSON
gh-pr-metrics csv --owner ansible --repo awx --output metrics.csv
```

**Use Cases**:
- Changed AI bot configuration
- Testing different output formats
- Fixed bug in metrics calculation
- No API rate limit impact

### Fetch Without Immediate CSV

**Old**: Not possible (always generated CSV)

**New**:
```bash
# Just fetch and cache data
gh-pr-metrics fetch --owner ansible --repo awx --start 2024-01-01

# Generate CSV later (or never)
gh-pr-metrics csv --owner ansible --repo awx --output metrics.csv
```

**Use Cases**:
- Batch data collection
- Deferred analysis
- Different CSV outputs from same data

## Argument Changes

### Global Arguments (All Commands)

| Old | New | Notes |
|-----|-----|-------|
| `--debug` | `--debug` | Unchanged |
| `--config FILE` | `--config FILE` | Unchanged |
| `--version` | `--version` | Unchanged |

### Init Command Arguments

| Old | New | Notes |
|-----|-----|-------|
| `--init --owner X` | `init --owner X` | Flag → subcommand |
| `--init --start DATE` | `init --start DATE` | No change |
| `--init --output PATH` | `init --output PATH` | No change |
| `--repo Y` | `--repo Y` | Optional (omit to init all repos for owner) |

### Fetch Command Arguments

| Old | New | Notes |
|-----|-----|-------|
| `--owner X --repo Y` | `fetch --owner X --repo Y` | Add subcommand |
| `--start DATE` | `--start DATE` | Unchanged |
| `--end DATE` | `--end DATE` | Unchanged |
| `--update` | `--update` | Unchanged |
| `--update-all` | `--all` | Renamed for consistency |
| `--wait` | `--wait` | Unchanged (only with `--all`) |
| `--pr NUMBER` | `--pr NUMBER` | Unchanged |
| `--workers N` | `--workers N` | Unchanged |

### CSV Command Arguments

| Old | New | Notes |
|-----|-----|-------|
| `--output FILE` | `csv --owner X --repo Y --output FILE` | Add subcommand + owner/repo |
| N/A | `csv --all` | **New**: Process all cached repos |

**Removed from CSV**:
- `--start`, `--end` (CSV uses all cached JSON)
- `--update` (CSV always regenerates from cache)
- `--workers` (CSV is computation, not parallel I/O)

## State File Changes

### Old Format
```yaml
https://github.com/ansible/awx:
  timestamp: "2024-12-09T12:00:00Z"
  csv_file: "data/ansible-awx.csv"
```

### New Format
```yaml
https://github.com/ansible/awx:
  timestamp: "2024-12-09T12:00:00Z"
  csv_file: "data/ansible-awx.csv"
  failed_prs: [123, 456]  # NEW: Tracks PRs that failed to fetch
```

**Changes**:
- Added `failed_prs` list to track fetch failures
- `init` command can set `csv_file`
- `fetch` command updates `timestamp` and `failed_prs` (never `csv_file`)
- `csv` command updates `csv_file` only if `--output` explicitly provided

**Compatibility**: Old state files work (failed_prs defaults to empty list).

## Behavior Changes

### CSV Output Path Resolution

**Old Order**:
1. CLI `--output` argument
2. Error if not specified

**New Order** (for `csv` command):
1. CLI `--output` argument
2. State file `csv_file` value
3. Config `output_pattern`
4. Error if none specified

**Impact**: Can run `csv --owner X --repo Y` without `--output` if path stored in state.

### Update Mode

**Old Behavior**:
- Single command fetches new data and merges with CSV
- CSV always updated

**New Behavior**:
- `fetch --update` fetches new data only
- `csv` command regenerates complete CSV from all cached JSON
- No "merge" concept - always complete regeneration

**Impact**: Simpler logic, complete CSV regeneration ensures consistency.

### Failed PR Tracking

**Old**: PR fetch failures logged but not tracked

**New**: 
- Failed PRs added to state file `failed_prs` list
- Retried automatically on next fetch
- Removed from list on successful fetch

**Impact**: Better reliability, automatic retry of failures.

## Common Migration Scenarios

### Daily Automation (Cron Job)

**Old**:
```bash
#!/bin/bash
# Daily metrics update
gh-pr-metrics --update-all
```

**New**:
```bash
#!/bin/bash
# Daily metrics update
gh-pr-metrics fetch --all
gh-pr-metrics csv --all
```

### Multi-Repository Setup

**Old**:
```bash
# Initialize multiple repos
gh-pr-metrics --init --owner ansible --start 2024-01-01 --output "data/{owner}-{repo}.csv"

# Daily updates
gh-pr-metrics --update-all
```

**New**:
```bash
# Initialize multiple repos (same)
gh-pr-metrics init --owner ansible --start 2024-01-01 --output "data/{owner}-{repo}.csv"

# Daily updates (two steps)
gh-pr-metrics fetch --all
gh-pr-metrics csv --all
```

### Single Repository Analysis

**Old**:
```bash
# One-time analysis
gh-pr-metrics --owner kubernetes --repo kubernetes --start 2024-01-01 --output k8s.csv
```

**New**:
```bash
# Fetch data
gh-pr-metrics fetch --owner kubernetes --repo kubernetes --start 2024-01-01

# Generate CSV
gh-pr-metrics csv --owner kubernetes --repo kubernetes --output k8s.csv
```

### Iterative Config Tuning

**Old**:
```bash
# Edit config
vim .gh-pr-metrics.yaml

# Re-fetch everything (slow, uses API calls)
gh-pr-metrics --owner ansible --repo awx --start 2024-01-01 --output metrics.csv
```

**New**:
```bash
# Edit config
vim .gh-pr-metrics.yaml

# Just regenerate CSV (fast, no API calls)
gh-pr-metrics csv --owner ansible --repo awx --output metrics.csv
```

## Advantages of New Architecture

### Separation of Concerns
- **Fetch**: API interaction, network I/O, rate limiting
- **CSV**: Computation, metrics, local processing
- Failures in one don't affect the other

### Efficiency
- Regenerate CSV without re-fetching data
- Change AI bot config and regenerate instantly
- Test metrics changes without API calls

### Flexibility
- Fetch data now, analyze later
- Multiple CSV outputs from same fetch
- Different time windows without duplicate fetches

### Testability
- Each command independently testable
- Clearer failure modes
- Better error messages

### Maintainability
- Smaller, focused modules
- Clear dependencies
- Easier to extend

## Troubleshooting

### "Command not found: fetch"

**Problem**: Using old CLI syntax

**Solution**: Add subcommand:
```bash
# Old
gh-pr-metrics --owner X --repo Y

# New
gh-pr-metrics fetch --owner X --repo Y
```

### "CSV file path required"

**Problem**: CSV command needs output path

**Solutions**:
1. Provide `--output`:
   ```bash
   gh-pr-metrics csv --owner X --repo Y --output file.csv
   ```

2. Or initialize first:
   ```bash
   gh-pr-metrics init --owner X --repo Y --start DATE --output file.csv
   gh-pr-metrics csv --owner X --repo Y  # Uses path from state
   ```

3. Or set config `output_pattern`:
   ```yaml
   # .gh-pr-metrics.yaml
   output_pattern: "data/{owner}-{repo}.csv"
   ```

### "No cached JSON found"

**Problem**: Running `csv` before `fetch`

**Solution**: Fetch data first:
```bash
gh-pr-metrics fetch --owner X --repo Y --start 2024-01-01
gh-pr-metrics csv --owner X --repo Y --output file.csv
```

### "Repository not tracked"

**Problem**: Using `fetch --update` or `fetch --all` without init

**Solution**: Either:
1. Initialize first:
   ```bash
   gh-pr-metrics init --owner X --repo Y --start DATE --output file.csv
   gh-pr-metrics fetch --update --owner X --repo Y
   ```

2. Or use regular fetch (auto-creates state entry):
   ```bash
   gh-pr-metrics fetch --owner X --repo Y --start 2024-01-01
   ```

## Getting Help

For command-specific help:
```bash
gh-pr-metrics --help
gh-pr-metrics init --help
gh-pr-metrics fetch --help
gh-pr-metrics csv --help
```

For issues or questions:
- Check `docs/refactor/ARCHITECTURE.md` for architecture details
- Review `docs/refactor/SUBCOMMAND_REFACTOR_PLAN.md` for implementation details
- Enable `--debug` for detailed logging

