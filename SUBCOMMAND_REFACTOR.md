# Subcommand Refactor Design Document

**Status**: Planning  
**Created**: 2025-12-08  
**Author**: Assisted by Cursor (Claude Sonnet 4.5)

## Executive Summary

Refactor `gh-pr-metrics` from a monolithic CLI to a subcommand-based architecture separating data fetching, CSV generation, aggregation, and configuration management. This separation enables:
- **Incremental processing**: Fetch once, generate reports multiple times
- **Reduced API usage**: CSV regeneration uses local JSON cache
- **Flexible analysis**: Rollup subcommand for multi-repo aggregation
- **Better UX**: Interactive config wizard

## Current Architecture

### Single Entry Point
`gh-pr-metrics` currently performs fetch + CSV generation in one pass:
1. Query GitHub API for PRs (filtered by date range)
2. Fetch detailed data (comments, reviews, timeline) per PR
3. Calculate metrics
4. Write CSV incrementally (thread-safe, one PR at a time)
5. Write JSON cache (`data/raw/github.com/{owner}/{repo}/{pr_number}.json`)
6. Update state file (`~/.gh-pr-metrics-state.yaml`)

### Key Modes
- **Regular mode**: `--owner --repo --start --end --output`
- **Init mode**: `--init --owner [--repo] --start --output` (initialize tracking)
- **Update mode**: `--update --owner --repo` (fetch since last update)
- **Update-all mode**: `--update-all [--wait]` (process all tracked repos)
- **Single PR mode**: `--pr NUMBER --owner --repo --output` (surgical update)

### State Management
State file tracks per-repository:
```yaml
https://github.com/{owner}/{repo}:
  timestamp: "2025-12-08T12:00:00Z"  # Last successful fetch
  csv_file: "data/owner_repo.csv"    # Output path
```

### Critical Implementation Details

#### 1. **Quota Management**
- `QuotaManager` singleton tracks GitHub API rate limits
- `can_process_with_wait()` determines if chunk is safe to process
- Reserves buffer to avoid complete exhaustion
- Hardcoded: 50 calls reserve, 50 min buffer
- Chunking estimates: 5 calls per PR (base + comments + reviews + timeline + buffer)

#### 2. **Chunking Strategy**
- Fetch ALL PR IDs matching date range first (GitHub pagination)
- Process in chunks (default: 10 PRs per chunk)
- State updated after each chunk completion
- If quota exhausted mid-chunk, next run resumes from last completed chunk

#### 3. **Thread Safety**
- `CSVManager` uses `threading.Lock()` for read/write operations
- Multiple workers can process PRs concurrently
- CSV updates are serialized via lock
- JSON writes are inherently safe (one file per PR)

#### 4. **Merge vs Replace**
- **Merge mode** (`--update`): Reads existing CSV, merges new PRs, writes full file
- **Replace mode** (regular): Writes fresh CSV
- Merge identifies existing PRs by `pr_number` column

#### 5. **AI Bot Detection**
- Config-driven: `always` bots (name match) vs `conditional` (name + content patterns)
- Applied during metric calculation
- Metrics: `ai_review_comments`, `ai_pr_comments`, `is_ai_authored`

## Proposed Architecture

### Design Philosophy
1. **Separation of Concerns**: Fetch (network I/O) vs Process (computation) vs Aggregate (analysis)
2. **Idempotent Operations**: Re-running same command produces same output
3. **State as Single Source of Truth**: State file drives update logic
4. **JSON as Intermediate Format**: Complete PR data cached locally

### Command Structure
```
gh-pr-metrics <subcommand> [options]

Subcommands:
  fetch     Fetch PR data from GitHub API, store as JSON
  csv       Generate CSV metrics from local JSON files
  rollup    Aggregate metrics across multiple CSVs
  config    Interactive configuration wizard
```

---

## Subcommand: `fetch`

### Purpose
Fetch PR data from GitHub API and cache as JSON files. No CSV generation.

### Usage
```bash
# Fetch specific repository
gh-pr-metrics fetch --owner ansible --repo awx --start 2024-01-01

# Initialize and fetch all repos for an owner
gh-pr-metrics fetch --init --owner ansible --start 2024-01-01

# Update specific repository (fetch PRs since last fetch)
gh-pr-metrics fetch --update --owner ansible --repo awx

# Update all tracked repositories
gh-pr-metrics fetch --update-all [--wait]

# Fetch single PR (surgical update)
gh-pr-metrics fetch --pr 12345 --owner ansible --repo awx
```

### Arguments
- `--owner OWNER`: Repository owner (required unless `--update-all`)
- `--repo REPO`: Repository name (optional for `--init`, required otherwise)
- `--start DATE`: Start date for PR query (required for new fetches)
- `--end DATE`: End date for PR query (default: now)
- `--init`: Initialize mode - discovers repos, validates access, stores state
- `--update`: Update mode - fetch since last successful fetch timestamp
- `--update-all`: Update all tracked repositories
- `--wait`: Wait for rate limit reset between repos (used with `--update-all`)
- `--pr NUMBER`: Fetch single PR only (surgical update)
- `--workers N`: Parallel workers for PR fetching (default: 4)
- `--debug`: Enable debug logging

### Behavior

#### **Regular Fetch** (`--owner --repo --start`)
1. Query GitHub API: `GET /repos/{owner}/{repo}/pulls?state=all&since={start}`
2. Filter PRs by `updated_at` within `[start, end]`
3. Chunk PRs (default: 10 per chunk)
4. For each chunk:
   - Check quota via `QuotaManager.can_process_with_wait()`
   - Parallel fetch PR details: comments, reviews, timeline
   - Write JSON files: `data/raw/github.com/{owner}/{repo}/{pr_number}.json`
   - Update state file with chunk completion timestamp
5. State updated to `end_date` if all chunks completed successfully

#### **Init Mode** (`--init --owner --start`)
1. If `--repo` provided: initialize single repo
2. If only `--owner`: query `GET /orgs/{owner}/repos` or `GET /users/{owner}/repos`
3. For each repository:
   - Check if already tracked (skip if in state file)
   - Store initial state entry with provided `--start` as timestamp
4. Does NOT fetch PR data immediately - just tracks repos
5. Follow up with `fetch --update-all` to actually fetch data

#### **Update Mode** (`--update --owner --repo`)
1. Read state file, get last fetch timestamp for repo
2. Fetch PRs updated since that timestamp
3. Overwrite JSON files for updated PRs
4. Update state timestamp to current time

#### **Update-All Mode** (`--update-all`)
1. Read state file, get all tracked repos
2. For each repo, perform update fetch
3. If `--wait` provided, wait for rate limit reset between repos
4. Continue on individual repo errors (log and move to next)

#### **Single PR Mode** (`--pr NUMBER`)
1. Fetch only the specified PR
2. Overwrite JSON file for that PR
3. Do NOT update state timestamp (surgical operation)

### Output
- **JSON files**: `data/raw/github.com/{owner}/{repo}/{pr_number}.json`
- **State file**: `~/.gh-pr-metrics-state.yaml`
- **Logging**: Progress, quota status, errors to log file and stderr

### State File Changes
```yaml
https://github.com/{owner}/{repo}:
  timestamp: "2025-12-08T12:00:00Z"  # Last successful fetch
  # csv_file removed - managed by 'csv' subcommand
```

### Error Handling
- **Quota exhaustion**: Stop mid-chunk, update state to last completed chunk, exit code 1
- **API errors**: Log error, exit code 1 for single repo, continue for `--update-all`
- **Network errors**: Retry logic in `GitHubClient`, fail after retries

### Critical Implementation Notes
1. **Remove CSV generation**: No `CSVManager` usage, no `process_pr()` call for metrics
2. **Keep chunking**: Essential for quota management and resumability
3. **Keep state management**: Track fetch timestamps for update mode
4. **Keep quota logic**: `QuotaManager` calculations remain unchanged
5. **JSON structure**: Must include ALL data needed for CSV generation (comments, reviews, timeline)

---

## Subcommand: `csv`

### Purpose
Generate CSV metrics from local JSON files. No API calls.

### Usage
```bash
# Generate CSV for specific repository
gh-pr-metrics csv --owner ansible --repo awx --output data/ansible_awx.csv

# Generate CSV for all fetched repos
gh-pr-metrics csv --all

# Regenerate CSV with different AI bot config (no re-fetch needed)
gh-pr-metrics csv --owner ansible --repo awx --output data/ansible_awx.csv

# Update existing CSV (merge mode)
gh-pr-metrics csv --update --owner ansible --repo awx
```

### Arguments
- `--owner OWNER`: Repository owner (required unless `--all`)
- `--repo REPO`: Repository name (required unless `--all`)
- `--output PATH`: Output CSV file path (supports patterns: `data/{owner}-{repo}.csv`)
- `--all`: Process all repositories with fetched JSON data
- `--update`: Merge mode - read existing CSV, add/update PRs, write back
- `--debug`: Enable debug logging

### Behavior

#### **Regular CSV Generation** (`--owner --repo --output`)
1. Scan JSON directory: `data/raw/github.com/{owner}/{repo}/*.json`
2. Read all JSON files for the repository
3. For each PR JSON:
   - Apply AI bot detection (config-driven)
   - Calculate metrics (time to merge, review counts, comment counts, etc.)
   - Build CSV row
4. Write CSV file atomically (write to temp, move to final)
5. Do NOT update state file (state tracks fetch, not CSV generation)

#### **Update Mode** (`--update --owner --repo`)
1. Read state file to get stored `csv_file` path
2. Read existing CSV file
3. Load all JSON files for repository
4. Merge: Update existing PRs, add new PRs
5. Write updated CSV atomically
6. Update state file's `csv_file` path

#### **All Mode** (`--all`)
1. Scan `data/raw/github.com/` for all `{owner}/{repo}` directories
2. For each discovered repository:
   - Generate output path from config `output_pattern`
   - Generate CSV as in regular mode
   - Store `csv_file` in state file

### Output
- **CSV files**: Specified by `--output` or config `output_pattern`
- **State file**: Updated with `csv_file` path for `--update` or `--all` modes
- **Logging**: Progress, PR counts, errors to log file and stderr

### CSV Format (Unchanged)
Columns: `pr_number`, `title`, `author`, `created_at`, `merged_at`, `closed_at`, `state`, `review_comments`, `pr_comments`, `commits`, `additions`, `deletions`, `changed_files`, `is_ai_authored`, `ai_review_comments`, `ai_pr_comments`, `time_to_merge_hours`, etc.

### State File Changes
```yaml
https://github.com/{owner}/{repo}:
  timestamp: "2025-12-08T12:00:00Z"  # From 'fetch' subcommand
  csv_file: "data/ansible_awx.csv"   # From 'csv' subcommand
```

### Error Handling
- **Missing JSON files**: Error if no JSON files found for specified repo
- **Malformed JSON**: Log warning, skip PR, continue processing
- **File I/O errors**: Exit code 1, clear error message

### Critical Implementation Notes
1. **No API calls**: Pure local processing, no `GitHubClient` usage
2. **AI bot detection**: Must read config, apply same logic as current implementation
3. **Metrics calculation**: Reuse existing `process_pr()` logic but feed from JSON not API
4. **Thread safety not needed**: Single-threaded processing (no API bottleneck)
5. **JSON schema**: Must match what `fetch` subcommand writes

---

## Subcommand: `rollup`

### Purpose
Aggregate metrics across multiple CSV files for multi-repo analysis.

### Usage
```bash
# Rollup by repository
gh-pr-metrics rollup --input "data/*.csv" --output rollup_by_repo.csv --by repo

# Rollup by owner
gh-pr-metrics rollup --input "data/*.csv" --output rollup_by_owner.csv --by owner

# Rollup by month
gh-pr-metrics rollup --input "data/*.csv" --output rollup_by_month.csv --by month
```

### Arguments
- `--input PATTERN`: Glob pattern for input CSV files (e.g., `data/*.csv`)
- `--output PATH`: Output rollup CSV file
- `--by DIMENSION`: Aggregation dimension: `repo`, `owner`, `month`, `quarter`, `year`
- `--debug`: Enable debug logging

### Behavior

#### **Rollup by Repository**
1. Read all matching CSV files
2. For each repository (inferred from CSV filename or columns):
   - Count total PRs
   - Calculate average time to merge
   - Count merged vs closed vs open
   - Sum review comments, PR comments
   - Calculate AI bot participation rate
3. Write rollup CSV with one row per repository

#### **Rollup by Owner**
Same as repository but grouped by `owner` (extracted from repo URL or filename pattern)

#### **Rollup by Time Period**
1. Parse `merged_at` or `created_at` timestamps
2. Group by month/quarter/year
3. Aggregate metrics per time bucket

### Output Format
Rollup CSV columns (example for `--by repo`):
```
owner,repo,total_prs,merged_prs,avg_time_to_merge_hours,total_review_comments,ai_participation_rate
```

### Error Handling
- **No matching files**: Error if glob matches zero files
- **Malformed CSV**: Log warning, skip file, continue
- **Missing columns**: Error if required columns missing from input CSVs

### Critical Implementation Notes
1. **Schema inference**: Input CSVs may have different column sets - handle gracefully
2. **Filename parsing**: May need to extract owner/repo from filename if not in CSV columns
3. **Date parsing**: Use `dateutil.parser` for flexible timestamp parsing
4. **Aggregation functions**: Mean, median, count, sum - use pandas or custom implementation

---

## Subcommand: `config`

### Purpose
Interactive wizard to create/update `.gh-pr-metrics.yaml` configuration file.

### Usage
```bash
# Interactive mode
gh-pr-metrics config

# Non-interactive (set specific values)
gh-pr-metrics config --set workers=16
gh-pr-metrics config --set output_pattern="data/{owner}-{repo}.csv"
```

### Arguments
- `--set KEY=VALUE`: Set config value non-interactively (can be repeated)
- `--show`: Display current configuration and exit
- `--debug`: Enable debug logging

### Behavior

#### **Interactive Mode** (default)
1. Check if `.gh-pr-metrics.yaml` exists
2. Load existing config if present, otherwise use defaults
3. For each configuration option, prompt user:
   - Display current value (if exists) or default
   - Show description and valid values
   - Accept input or Enter to keep current/default
   - Validate input
4. Write updated config to `.gh-pr-metrics.yaml`
5. Pretty-print YAML with comments

#### **Non-Interactive Mode** (`--set KEY=VALUE`)
1. Load existing config
2. Apply each `--set` value
3. Validate values
4. Write updated config
5. Exit

#### **Show Mode** (`--show`)
1. Load config
2. Pretty-print to stdout with descriptions
3. Exit

### Configuration Options

**Workers** (`workers`)
- Type: Integer
- Default: 4
- Validation: Must be >= 1
- Description: Number of parallel workers for PR fetching

**Output Pattern** (`output_pattern`)
- Type: String (pattern)
- Default: null (stdout)
- Placeholders: `{owner}`, `{repo}`
- Example: `data/{owner}-{repo}.csv`
- Description: Default output path pattern for CSV files

**Log File** (`log_file`)
- Type: String (filepath)
- Default: `gh-pr-metrics.log`
- Description: Log file location for persistent logging

**Raw Data Directory** (`raw_data_dir`)
- Type: String (dirpath)
- Default: `data/raw`
- Description: Base directory for JSON cache files

**AI Bots - Always** (`ai_bots.always`)
- Type: List of regex patterns
- Default: `["cursor\\[bot\\]", "Copilot"]`
- Description: Bot names always classified as AI

**AI Bots - Conditional** (`ai_bots.conditional`)
- Type: List of objects (name, content_patterns, match_any)
- Default: See example config
- Description: Bots classified as AI based on content patterns

### Interactive Prompt Example
```
GitHub PR Metrics Configuration Wizard
=====================================

Current configuration: .gh-pr-metrics.yaml (exists)

Workers (parallel PR fetching)
  Current: 16
  Default: 4
  Enter value [16]: 20
  ✓ Set to 20

Output Pattern (CSV file path template)
  Current: data/{owner}-{repo}.csv
  Default: null (stdout)
  Placeholders: {owner}, {repo}
  Enter value [data/{owner}-{repo}.csv]: 
  ✓ Keeping data/{owner}-{repo}.csv

... (continue for all options)

Configuration written to .gh-pr-metrics.yaml
```

### Error Handling
- **Invalid input**: Re-prompt with error message
- **Write failure**: Error if unable to write config file
- **Interrupt (Ctrl+C)**: Confirm before discarding changes

### Critical Implementation Notes
1. **Use `prompt_toolkit`** or similar for rich interactive prompts
2. **YAML formatting**: Preserve comments and structure when updating existing config
3. **Validation**: Each config option has validation function
4. **Defaults from code**: Read `DEFAULT_*` constants from main module

---

## Implementation Plan

### Phase 1: Create Subcommand Structure ✓
1. Refactor `parse_arguments()` to use `argparse.add_subparsers()`
2. Create stub handler functions: `cmd_fetch()`, `cmd_csv()`, `cmd_rollup()`, `cmd_config()`
3. Update `main()` to dispatch to appropriate handler
4. Update tests to use subcommand syntax

### Phase 2: Implement `fetch` Subcommand ✓
1. Extract fetch logic from current `process_repository()`
2. Remove CSV generation calls
3. Ensure JSON writing works correctly
4. Update state management (remove `csv_file` updates during fetch)
5. Test all fetch modes: regular, init, update, update-all, single PR

### Phase 3: Implement `csv` Subcommand ✓
1. Create JSON reader: `read_pr_json(owner, repo, pr_number) -> dict`
2. Create JSON scanner: `scan_pr_jsons(owner, repo) -> List[dict]`
3. Refactor `process_pr()` to accept JSON dict instead of making API calls
4. Implement merge mode for `--update`
5. Implement `--all` mode with directory scanning
6. Test CSV generation from cached JSON

### Phase 4: Implement `rollup` Subcommand
1. Create CSV reader with schema inference
2. Implement aggregation functions
3. Implement grouping by repo/owner/time
4. Test with multiple input CSVs

### Phase 5: Implement `config` Subcommand
1. Choose prompt library (`prompt_toolkit` recommended)
2. Implement interactive wizard
3. Implement non-interactive `--set` mode
4. Implement `--show` mode
5. Test config file creation and updates

### Phase 6: Documentation and Cleanup
1. Update README.md with new subcommand usage
2. Update help text and examples
3. Remove legacy code paths
4. Final test pass

---

## Testing Strategy

### Unit Tests
- Each subcommand handler tested independently
- Mock GitHub API for `fetch` tests
- Mock filesystem for `csv` and `rollup` tests
- Config validation tests

### Integration Tests
- End-to-end: `fetch` → `csv` → `rollup` pipeline
- State file consistency across commands
- Error propagation and handling

### Regression Tests
- Ensure CSV output format unchanged (column order, values)
- Ensure AI bot detection behavior unchanged
- Ensure quota management behavior unchanged

---

## Risk Assessment

### High Risk: Data Loss
- **Mitigation**: Atomic file writes (write to temp, move to final)
- **Mitigation**: State file backups before updates
- **Mitigation**: Extensive testing of merge mode

### High Risk: Breaking Changes
- **Mitigation**: No backward compatibility needed (single user)
- **Mitigation**: Document migration path clearly

### Medium Risk: JSON Schema Drift
- **Mitigation**: Define explicit JSON schema in code
- **Mitigation**: Version JSON files (add `schema_version` field)
- **Mitigation**: Validate JSON on read

### Medium Risk: State File Corruption
- **Mitigation**: YAML parsing error handling
- **Mitigation**: State file validation on load
- **Mitigation**: Ability to rebuild state from JSON cache

### Low Risk: Performance Regression
- **Mitigation**: Benchmark CSV generation speed (should be faster without API calls)
- **Mitigation**: Profile memory usage for `rollup` with many CSVs

---

## Migration Path (For Reference)

Even though backward compatibility is not required, this documents the conceptual mapping:

### Old Command → New Command
```bash
# Old: Fetch and generate CSV in one pass
gh-pr-metrics --owner ansible --repo awx --start 2024-01-01 --output data.csv

# New: Two-step process
gh-pr-metrics fetch --owner ansible --repo awx --start 2024-01-01
gh-pr-metrics csv --owner ansible --repo awx --output data.csv

# Old: Update existing CSV
gh-pr-metrics --owner ansible --repo awx --update

# New: Fetch new data, then update CSV
gh-pr-metrics fetch --update --owner ansible --repo awx
gh-pr-metrics csv --update --owner ansible --repo awx

# Old: Update all tracked repos
gh-pr-metrics --update-all

# New: Fetch all, then generate all CSVs
gh-pr-metrics fetch --update-all
gh-pr-metrics csv --all
```

---

## Open Questions

1. **JSON Schema Versioning**: Should we version the JSON schema and handle migrations?
   - **Recommendation**: Add `schema_version: 1` to each JSON file, handle in reader
   
2. **CSV Subcommand State**: Should `csv` subcommand update state file with `csv_file` path?
   - **Recommendation**: Yes, for `--update` and `--all` modes to enable future `csv --update` calls
   
3. **Rollup Subcommand State**: Should rollup operations be tracked in state?
   - **Recommendation**: No, rollup is stateless analysis
   
4. **Config Subcommand**: Should it validate AI bot regex patterns?
   - **Recommendation**: Yes, compile regex on input and catch errors
   
5. **Directory Structure**: Should raw data directory be configurable per-provider?
   - **Recommendation**: Keep simple for now: `{raw_data_dir}/github.com/{owner}/{repo}/`

---

## Success Criteria

- [ ] All existing functionality works via new subcommands
- [ ] CSV output format unchanged (validates against existing CSVs)
- [ ] State file format handles new structure
- [ ] Test coverage maintained at 80%+
- [ ] No increase in API quota usage
- [ ] Documentation complete and accurate
- [ ] User (single) successfully uses new CLI

---

## Rollback Plan

If implementation fails:
1. This document provides complete specification
2. Current code is in git history
3. Can restart from clean slate with this design doc
4. Consider breaking into smaller PRs (one subcommand at a time)

---

## Appendix A: JSON Schema Example

```json
{
  "schema_version": 1,
  "pr": {
    "number": 12345,
    "title": "Fix critical bug",
    "user": {"login": "contributor"},
    "created_at": "2024-01-01T00:00:00Z",
    "merged_at": "2024-01-02T00:00:00Z",
    "closed_at": "2024-01-02T00:00:00Z",
    "state": "closed",
    "merged": true,
    "commits": 5,
    "additions": 100,
    "deletions": 50,
    "changed_files": 3
  },
  "comments": [
    {
      "user": {"login": "reviewer"},
      "body": "LGTM",
      "created_at": "2024-01-01T12:00:00Z"
    }
  ],
  "reviews": [
    {
      "user": {"login": "reviewer"},
      "state": "APPROVED",
      "body": "Looks good",
      "submitted_at": "2024-01-01T12:00:00Z"
    }
  ],
  "timeline": [
    {
      "event": "reviewed",
      "user": {"login": "reviewer"},
      "created_at": "2024-01-01T12:00:00Z"
    }
  ]
}
```

---

## Appendix B: State File Schema Evolution

### Current Schema
```yaml
https://github.com/{owner}/{repo}:
  timestamp: "2025-12-08T12:00:00Z"
  csv_file: "data/ansible_awx.csv"
```

### Proposed Schema (No Changes Needed)
Same as current. `timestamp` managed by `fetch`, `csv_file` managed by `csv`.

---

## Appendix C: Code Organization

### File Structure (Proposed)
```
src/
  gh_pr_metrics.py          # Main entry, argument parsing, dispatch
  cmd_fetch.py              # Fetch subcommand implementation
  cmd_csv.py                # CSV subcommand implementation
  cmd_rollup.py             # Rollup subcommand implementation
  cmd_config.py             # Config subcommand implementation
  github_api.py             # Unchanged
  csv_manager.py            # Refactored for JSON input
  state_manager.py          # Unchanged
  quota_manager.py          # Unchanged
  logging_manager.py        # Unchanged
```

### Alternative: Keep Monolithic File
Given project size (2000 lines), could keep single file with clear section markers:
```python
# ============================================================================
# SUBCOMMAND: fetch
# ============================================================================

def cmd_fetch(args, config):
    ...

# ============================================================================
# SUBCOMMAND: csv
# ============================================================================

def cmd_csv(args, config):
    ...
```

**Recommendation**: Start with monolithic, split if any single subcommand exceeds 500 lines.

---

## Document Changelog

- **2025-12-08**: Initial version - comprehensive design before implementation

