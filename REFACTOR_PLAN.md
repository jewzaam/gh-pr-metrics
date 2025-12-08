# Step-by-Step Refactor Plan: Extract Fetch and CSV Classes

**Goal**: Separate fetch and CSV concerns by creating classes, WITHOUT changing existing behavior.

**Approach**: Baby steps. Extract logic into classes first, keep same call sites, then introduce subcommands later.

---

## Phase 1: Extract Fetch Functionality into Class

### Current State Analysis

**Where fetch happens now:**
- `fetch_and_process_pr()` (line ~1180-1220): Fetches single PR with all details
  - Calls `build_pr_json_data()` to get comments, reviews, timeline
  - Writes JSON file
  - Passes data to `process_pr()` for metrics
- `process_repository()` (line ~1255-1463): Main processing loop
  - Calls `github_client.fetch_all_prs()` to get PR list
  - Chunks PRs
  - Calls `fetch_and_process_pr()` for each PR in parallel

**What needs to be extracted:**
- PR list fetching
- PR detail fetching (comments, reviews, timeline)
- JSON file writing
- Chunking logic
- Parallel execution

### Step 1.1: Create `PRFetcher` Class

**File**: `src/gh_pr_metrics.py` (add before `process_repository`)

**Class structure:**
```python
class PRFetcher:
    """Handles fetching PR data from GitHub API and caching as JSON."""
    
    def __init__(self, github_client, quota_manager, config, logger):
        self.github_client = github_client
        self.quota_manager = quota_manager
        self.config = config
        self.logger = logger
    
    def fetch_pr_list(self, owner, repo, start_date, end_date):
        """Fetch list of PR IDs in date range."""
        # Move logic from process_repository line ~1312
        pass
    
    def fetch_pr_details(self, owner, repo, pr_number):
        """Fetch complete PR details including comments, reviews, timeline."""
        # Move logic from build_pr_json_data()
        pass
    
    def write_pr_json(self, owner, repo, pr_data):
        """Write PR data to JSON cache file."""
        # Move logic from write_pr_json()
        pass
    
    def fetch_and_cache_pr(self, owner, repo, pr_number):
        """Fetch PR details and cache to JSON. Returns pr_json_data."""
        # Move logic from fetch_and_process_pr() (fetch part only)
        pass
```

**Changes needed:**
1. Add class definition around line 1250 (before `process_repository`)
2. Move `build_pr_json_data()` logic into `fetch_pr_details()`
3. Move `write_pr_json()` logic into `write_pr_json()`
4. Keep existing standalone functions as thin wrappers (for now)

**Validation:**
- Run existing tests - they should all pass
- No behavior changes
- Same files written, same API calls

### Step 1.2: Update `fetch_and_process_pr()` to use `PRFetcher`

**Current** (line ~1180):
```python
def fetch_and_process_pr(owner, repo, pr_number, output_file, csv_manager, config):
    pr_json_data = build_pr_json_data(owner, repo, pr_number)
    write_pr_json(owner, repo, pr_json_data, config)
    metrics = process_pr(pr_json_data, owner, repo, config)
    csv_manager.write_csv(output_file, [metrics])
```

**After**:
```python
def fetch_and_process_pr(owner, repo, pr_number, output_file, csv_manager, config, pr_fetcher):
    pr_json_data = pr_fetcher.fetch_and_cache_pr(owner, repo, pr_number)
    metrics = process_pr(pr_json_data, owner, repo, config)
    csv_manager.write_csv(output_file, [metrics])
```

**Changes:**
1. Add `pr_fetcher` parameter to `fetch_and_process_pr()`
2. Replace `build_pr_json_data() + write_pr_json()` with `pr_fetcher.fetch_and_cache_pr()`
3. Update all call sites to pass `pr_fetcher` instance

**Validation:**
- Run tests
- Verify JSON files still written correctly
- Verify CSV files unchanged

### Step 1.3: Update `process_repository()` to use `PRFetcher`

**Current** (line ~1312):
```python
def process_repository(...):
    all_prs = github_client.fetch_all_prs(owner, repo, start_date, end_date)
    # ... chunking ...
    # ... parallel processing with fetch_and_process_pr ...
```

**After**:
```python
def process_repository(...):
    pr_fetcher = PRFetcher(github_client, quota_manager, config, logger)
    all_prs = pr_fetcher.fetch_pr_list(owner, repo, start_date, end_date)
    # ... chunking ...
    # ... parallel processing with fetch_and_process_pr(pr_fetcher=pr_fetcher) ...
```

**Changes:**
1. Create `PRFetcher` instance at start of `process_repository()`
2. Replace `github_client.fetch_all_prs()` with `pr_fetcher.fetch_pr_list()`
3. Pass `pr_fetcher` to `fetch_and_process_pr()` calls

**Validation:**
- Run tests
- Verify behavior unchanged

---

## Phase 2: Extract CSV/Metrics Functionality into Class

### Current State Analysis

**Where CSV/metrics happen now:**
- `process_pr()` (line ~1064-1137): Calculates metrics from PR JSON
  - AI bot detection
  - Time calculations
  - Comment/review counts
  - Returns dict of metrics
- `CSVManager` (line ~835-1052): Reads/writes CSV files
  - Has `read_csv()` and `write_csv()`
  - Thread-safe with lock
- `fetch_and_process_pr()`: Calls `process_pr()` then `csv_manager.write_csv()`

**What needs to be extracted:**
- Metrics calculation logic
- Reading PR JSON from cache
- Converting JSON → metrics
- CSV generation

### Step 2.1: Create `MetricsGenerator` Class

**File**: `src/gh_pr_metrics.py` (add after `PRFetcher`)

**Class structure:**
```python
class MetricsGenerator:
    """Handles generating CSV metrics from PR JSON data."""
    
    def __init__(self, config, logger):
        self.config = config
        self.logger = logger
    
    def read_pr_json(self, owner, repo, pr_number):
        """Read PR JSON from cache file."""
        # New function - reads from data/raw/github.com/{owner}/{repo}/{pr_number}.json
        pass
    
    def calculate_metrics(self, pr_json_data, owner, repo):
        """Calculate metrics from PR JSON. Returns metrics dict."""
        # Move logic from process_pr()
        pass
    
    def generate_csv_row(self, pr_json_data, owner, repo):
        """Read JSON, calculate metrics, return CSV row."""
        return self.calculate_metrics(pr_json_data, owner, repo)
```

**Changes needed:**
1. Add class definition
2. Move `process_pr()` logic into `calculate_metrics()`
3. Add `read_pr_json()` method to read from cache
4. Keep existing `process_pr()` as thin wrapper (for now)

**Validation:**
- Run tests
- CSV output unchanged

### Step 2.2: Update `fetch_and_process_pr()` to use `MetricsGenerator`

**Current**:
```python
def fetch_and_process_pr(...):
    pr_json_data = pr_fetcher.fetch_and_cache_pr(owner, repo, pr_number)
    metrics = process_pr(pr_json_data, owner, repo, config)
    csv_manager.write_csv(output_file, [metrics])
```

**After**:
```python
def fetch_and_process_pr(..., metrics_generator):
    pr_json_data = pr_fetcher.fetch_and_cache_pr(owner, repo, pr_number)
    metrics = metrics_generator.calculate_metrics(pr_json_data, owner, repo)
    csv_manager.write_csv(output_file, [metrics])
```

**Changes:**
1. Add `metrics_generator` parameter
2. Replace `process_pr()` with `metrics_generator.calculate_metrics()`
3. Update call sites to pass `metrics_generator` instance

**Validation:**
- Run tests
- CSV unchanged

### Step 2.3: Update `process_repository()` to use `MetricsGenerator`

**After**:
```python
def process_repository(...):
    pr_fetcher = PRFetcher(github_client, quota_manager, config, logger)
    metrics_generator = MetricsGenerator(config, logger)
    all_prs = pr_fetcher.fetch_pr_list(owner, repo, start_date, end_date)
    # ... pass both to fetch_and_process_pr ...
```

**Changes:**
1. Create `MetricsGenerator` instance
2. Pass to worker functions

**Validation:**
- Run full test suite
- Manual test: `gh-pr-metrics --owner ansible --repo awx --start 2024-01-01 --output test.csv`
- Compare output to previous run

---

## Phase 3: Checkpoint and Review

**At this point:**
- ✅ Fetch logic encapsulated in `PRFetcher` class
- ✅ Metrics/CSV logic encapsulated in `MetricsGenerator` class
- ✅ Existing CLI still works exactly the same
- ✅ All tests pass
- ✅ Clear boundaries between concerns

**Deliverables for review:**
1. Updated `src/gh_pr_metrics.py` with two new classes
2. All existing functionality preserved
3. Test results showing no regressions
4. Summary of what each class does

**Review questions to address:**
- Are class boundaries clear?
- Is any logic misplaced?
- Are there missing pieces?
- Is the approach sound for next steps?

---

## Phase 4: Add Subcommands (Future - After Checkpoint)

**Only proceed after Phase 3 approval**

### Step 4.1: Add Subcommand Parser (preserve existing behavior)

Add subcommands but keep monolithic mode as default:
```python
def parse_arguments():
    parser = argparse.ArgumentParser(...)
    subparsers = parser.add_subparsers(dest='subcommand')
    
    # Legacy mode (no subcommand) - default behavior
    # ... existing arguments ...
    
    # New: fetch subcommand
    fetch_parser = subparsers.add_parser('fetch', ...)
    # ... fetch arguments ...
    
    # New: csv subcommand  
    csv_parser = subparsers.add_parser('csv', ...)
    # ... csv arguments ...
```

### Step 4.2: Create Subcommand Handlers

```python
def cmd_fetch(args, config):
    """Fetch PR data and cache as JSON (no CSV)."""
    pr_fetcher = PRFetcher(...)
    # Use only pr_fetcher, no metrics_generator
    # Write JSON only
    
def cmd_csv(args, config):
    """Generate CSV from cached JSON (no fetch)."""
    metrics_generator = MetricsGenerator(...)
    # Read JSON from cache
    # Generate CSV
```

### Step 4.3: Update `main()` to Dispatch

```python
def main():
    args = parse_arguments()
    config = load_config(args.config)
    
    if args.subcommand == 'fetch':
        return cmd_fetch(args, config)
    elif args.subcommand == 'csv':
        return cmd_csv(args, config)
    else:
        # Legacy mode - existing behavior
        return legacy_main(args, config)
```

---

## Risk Assessment

### Phase 1-2 Risks: LOW
- Pure refactor, no behavior change
- Classes are just reorganizing existing code
- Easy to validate with tests
- Easy to revert if needed

### Phase 4 Risks: MEDIUM
- New CLI surface area
- Must maintain backward compatibility (or explicitly break it)
- Need new tests for subcommands
- State file implications (fetch updates timestamp, csv reads it)

---

## Questions for User

Before I start Phase 1:

1. **Class naming**: `PRFetcher` and `MetricsGenerator` - do these names make sense? Alternative: `DataFetcher`, `CSVProcessor`?

2. **File organization**: Keep everything in `gh_pr_metrics.py` or split classes into separate files?

3. **JSON reading**: Should `MetricsGenerator.read_pr_json()` handle missing files gracefully, or error loudly?

4. **Validation approach**: After each step, should I run just unit tests, or also do manual CLI validation?

5. **Incremental commits**: Do you want to review after each step (1.1, 1.2, 1.3...) or after each phase (1, 2, 3)?

---

## What I'm NOT doing (Learning from mistakes)

❌ Changing argument parser yet
❌ Removing any existing functions
❌ Changing test files yet  
❌ Modifying behavior in any way
❌ Assuming I know best
❌ Making multiple changes at once without validation

## What I AM doing

✅ Creating classes to encapsulate concerns
✅ Moving logic into classes method-by-method
✅ Keeping existing functions as wrappers initially
✅ Running tests after each change
✅ Asking for guidance when uncertain
✅ Taking baby steps
✅ Validating at each checkpoint

---

**Ready to proceed with Phase 1, Step 1.1?** 

Or would you like me to clarify/revise any part of this plan first?

