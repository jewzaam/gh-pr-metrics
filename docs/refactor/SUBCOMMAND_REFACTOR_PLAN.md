---
name: Subcommand OOP Final
overview: Refactor monolithic CLI to OOP subcommand architecture. Three commands (init, fetch, csv), proper separation of concerns, no circular dependencies. Plan describes strategy, not implementation.
todos:
  - id: step0-preserve
    content: Save plan, copy old code, create docs
    status: in_progress
  - id: step1-managers
    content: Extract managers to src/managers/
    status: pending
  - id: step2-data
    content: Extract data classes to src/data/
    status: pending
  - id: step3-commands-base
    content: Create src/commands/ and BaseCommand
    status: pending
  - id: step4-init-cmd
    content: Implement InitCommand
    status: pending
  - id: step5-fetch-cmd
    content: Implement FetchCommand with helper
    status: pending
  - id: step6-csv-cmd
    content: Implement CsvCommand with helper
    status: pending
  - id: step7-main
    content: Create new main() and parser
    status: pending
  - id: step8-tests
    content: Update all tests for new structure
    status: pending
  - id: step9-quality
    content: Run format/lint/test-unit/coverage, validate
    status: pending
  - id: step10-cleanup
    content: Delete old code, finalize
    status: pending
---

# Subcommand Architecture - Object-Oriented Refactor

## Overview

Refactor monolithic `gh_pr_metrics.py` into clean OOP architecture with three independent commands and proper separation of concerns.

## Commands

```
gh-pr-metrics init   - Initialize repository tracking
gh-pr-metrics fetch  - Fetch PR data from GitHub → JSON
gh-pr-metrics csv    - Generate CSV from JSON cache
```

## State Management (Corrections Applied)

### State File Tracks

Per repository:

- **Repository URL**: Full URL including provider (e.g., `https://github.com/ansible/awx`)
  - Future-proof for non-GitHub providers
  - Already supported in current implementation
- **Last fetch timestamp**: When data was last successfully fetched
- **CSV file path**: Where CSV is written (optional, set by init or csv with --output)
- **Failed PRs list**: PR numbers that failed to fetch and need retry
  - PR added when fetch fails
  - PR removed when fetch succeeds
  - Carried across fetch operations until resolved

### Who Updates State

| Command | Updates Timestamp | Updates csv_file | Updates failed_prs | Creates Entry |

|---------|-------------------|------------------|--------------------|---------------|

| **init** | ✓ (sets initial) | ✓ (if --output provided) | ✓ (initializes empty) | ✓ Always |

| **fetch** | ✓ (on success) | ✗ Never | ✓ Always | ✓ If doesn't exist |

| **csv** | ✗ Never | ✓ (only if --output provided) | ✗ Never | ✗ Never |

**Fetch can create state entry**: If fetching a repo not previously initialized, fetch creates state entry automatically.

## Target File Structure

```
src/
  gh_pr_metrics.py              # Main CLI entry point, command dispatcher (~200 lines)
  gh_pr_metrics_old.py          # Preserved original (reference only, no coverage expected)
  github_api.py                 # Unchanged
  
  managers/
    __init__.py
    config_manager.py           # Config, ConditionalBotConfig, load_config
    state_manager.py            # StateManager
    quota_manager.py            # QuotaManager  
    csv_manager.py              # CSVManager
    logging_manager.py          # LoggingManager
  
  data/
    __init__.py
    pr_fetcher.py               # PRFetcher
    metrics_generator.py        # MetricsGenerator + helper functions
  
  commands/
    __init__.py
    base_command.py             # BaseCommand abstract class
    init_command.py             # InitCommand
    fetch_command.py            # FetchCommand
    csv_command.py              # CsvCommand

tests/
  unit/
    test_commands/              # NEW: Command tests
      test_init_command.py
      test_fetch_command.py
      test_csv_command.py
      test_base_command.py
    test_managers/              # Reorganized manager tests
    test_data/                  # Reorganized data tests
    [existing test files updated for new imports]
  
  integration/
    test_init_fetch_csv_pipeline.py  # NEW: Full workflow test
    [existing integration tests updated]

docs/
  refactor/
    SUBCOMMAND_REFACTOR_PLAN.md     # This plan
    ARCHITECTURE.md                 # Dependency diagram
    MIGRATION_GUIDE.md              # Old CLI → New CLI mapping
```

## Dependency Architecture

**Layers** (no circular dependencies):

```
Layer 4: Commands
  ├─ InitCommand
  ├─ FetchCommand  
  └─ CsvCommand

Layer 3: Data Processing
  ├─ PRFetcher (uses Layer 2: GitHubClient, QuotaManager, ConfigManager, StateManager)
  └─ MetricsGenerator (uses Layer 2: ConfigManager)

Layer 2: Managers
  ├─ ConfigManager (independent)
  ├─ StateManager (independent)
  ├─ QuotaManager (independent)
  ├─ CSVManager (independent)
  ├─ LoggingManager (uses QuotaManager)
  └─ GitHubClient (uses QuotaManager)

Layer 1: Base
  └─ BaseCommand (uses Layer 2 managers)

Rules:
- Higher layers can depend on lower layers
- Same layer classes cannot depend on each other (except LoggingManager → QuotaManager)
- No upward dependencies
```

**Verification**: Create mermaid diagram in `docs/refactor/ARCHITECTURE.md` to visualize and check for cycles.

## Implementation Steps (High-Level Strategy)

### Step 0: Preservation and Documentation

**Actions**:

1. Save this plan to `docs/refactor/SUBCOMMAND_REFACTOR_PLAN.md`
2. Copy `src/gh_pr_metrics.py` → `src/gh_pr_metrics_old.py` (reference only)
3. Create architecture diagram in `docs/refactor/ARCHITECTURE.md`
4. Update `CONTEXT.md` to describe current codebase state for LLMs
5. Create `docs/refactor/MIGRATION_GUIDE.md` with old→new command mappings

**Validation**:

- Files exist and are readable
- Documentation clear and accurate

---

### Step 1: Extract Managers to Separate Files

**Objective**: Break monolith into independent manager modules.

**Create `src/managers/` directory with**:

1. `config_manager.py`: Config, ConditionalBotConfig, load_config function
2. `state_manager.py`: StateManager class (including failed PRs tracking)
3. `quota_manager.py`: QuotaManager class
4. `csv_manager.py`: CSVManager class
5. `logging_manager.py`: LoggingManager class
6. `__init__.py`: Export all managers

**Update `gh_pr_metrics.py`**:

- Change imports to use `from managers import ...`
- Keep global instances intact for now
- No functional changes

**Validation**:

- `make test-unit` passes
- `make lint` clean
- `make format` applied
- Old code reference untouched

---

### Step 2: Extract Data Classes to Separate Files

**Objective**: Separate data processing from main file.

**Create `src/data/` directory with**:

1. `pr_fetcher.py`: PRFetcher class
2. `metrics_generator.py`: MetricsGenerator class + helper functions (count_comments_from_json, get_review_metrics_from_json, etc.)
3. `__init__.py`: Export classes

**Update imports in `gh_pr_metrics.py`**:

- Change to `from data import PRFetcher, MetricsGenerator`

**Validation**:

- `make test-unit` passes
- `make lint` clean

---

### Step 3: Create Commands Directory and BaseCommand

**Objective**: Establish command class hierarchy.

**Create `src/commands/` directory with**:

1. `base_command.py`: 

   - BaseCommand abstract class
   - setup() method for common initialization
   - Abstract add_arguments() for subclass argument definition
   - Abstract run() for command execution
   - needs_github_client() hook (default True, CSV overrides to False)

2. `__init__.py`: Export all commands

**BaseCommand responsibilities**:

- Config loading
- Logging setup
- Manager initialization
- GitHub client initialization (if needed)
- Common argument handling (--debug, --config)

**Validation**:

- Code compiles
- Abstract class structure correct
- No tests run yet (no concrete implementations)

---

### Step 4: Implement InitCommand

**Objective**: Standalone repository initialization command.

**Create `src/commands/init_command.py`**:

**Command behavior**:

- Arguments: --owner (required), --repo (optional), --start (required), --output (optional)
- If --repo provided: Initialize single repo
- If --repo omitted: List all repos for owner via API, initialize each
- Creates state entry with:
  - Full repository URL
  - Initial timestamp (from --start)
  - CSV file path (from --output or config output_pattern)
  - Empty failed PRs list
- Skips already-initialized repos
- No data fetching or CSV generation

**Validation**:

- Test init with single repo
- Test init with all owner repos
- Test skip already-initialized
- Verify state file structure
- `make test-unit` passes

---

### Step 5: Implement FetchCommand

**Objective**: Fetch PR data to JSON cache, update state.

**Extract helper function**:

- `fetch_only_repository()`: Copy logic from current `process_repository()`, remove all CSV/metrics code
- Keep: PR fetching, JSON writing, chunking, quota management, parallel execution
- Remove: MetricsGenerator usage, CSVManager usage

**Create `src/commands/fetch_command.py`**:

**Command modes**:

1. **Regular**: `--owner --repo --start [--end]` - Fetch with date range
2. **Update single**: `--owner --repo --update` - Fetch since last timestamp (from state)
3. **Update all**: `--all` - Fetch all tracked repos since their last timestamps
4. **Single PR**: `--owner --repo --pr NUMBER` - Fetch one PR only

**State updates** (on success):

- Update timestamp to end_date
- Update failed_prs list (add failures, remove successes)
- Create state entry if doesn't exist (allows fetch without init)

**Validation**:

- Test all four modes
- Verify JSON files written correctly
- Verify NO CSV generated
- Verify state updates (timestamp, failed_prs)
- Test auto-creation of state entry
- `make test-unit` passes

---

### Step 6: Implement CsvCommand

**Objective**: Generate CSV from JSON cache.

**Extract helper function**:

- `generate_csv_from_json()`: Read all JSON for repo, calculate metrics, write CSV
- Always processes complete JSON cache (no "merge" concept)
- CSV regeneration is full regeneration from cache

**Create `src/commands/csv_command.py`**:

**Command modes**:

1. **Single repo with explicit output**: `--owner --repo --output FILE`
2. **Single repo with state path**: `--owner --repo` (uses csv_file from state)
3. **All repos**: `--all` (processes all repos with JSON cache)

**Output path resolution** (for single repo mode):

1. CLI --output argument (highest priority)
2. State file csv_file value
3. Config output_pattern
4. Error if none specified

**State updates**:

- Update csv_file ONLY if --output explicitly provided by user
- Never updates timestamp or failed_prs

**No GitHub client needed**:

- Override `needs_github_client()` to return False
- Pure local processing

**Validation**:

- Test with --output (updates state)
- Test without --output (uses state path, doesn't update state)
- Test --all mode
- Verify CSV content matches old implementation
- `make test-unit` passes

---

### Step 7: Create New Main Entry Point

**Objective**: Wire commands to CLI parser.

**Update `src/gh_pr_metrics.py`**:

**New parse_arguments()**:

- Global arguments: --debug, --config, --version
- Required subcommand: init, fetch, or csv
- Each subcommand calls Command.add_arguments() for its specific args
- Returns (command_name, args)

**New main()**:

- Parse arguments
- Map command name to Command class
- Instantiate command
- Call command.setup(args)
- Call command.run(args)
- Return exit code

**Delete from main file**:

- Old parse_arguments() implementation
- Old main() implementation (~400 lines)
- Keep only: imports, helper functions (parse_timestamp, expand_output_pattern, etc.)

**Validation**:

- `gh-pr-metrics --help` shows three subcommands
- Each subcommand help works
- `make test-unit` passes

---

### Step 8: Update All Tests

**Objective**: Comprehensive test coverage for new architecture.

**Update existing tests**:

- Change imports to new module structure
- Update argument construction for subcommands
- Ensure all 257 tests updated and passing

**Add new tests**:

- `tests/unit/test_commands/` (4 files: base, init, fetch, csv)
- `tests/integration/test_init_fetch_csv_pipeline.py`
- Manager tests if needed (should mostly work with updates)

**Test scenarios to cover**:

- Each command in isolation
- State management rules (who updates what)
- Failed PRs tracking
- Output path resolution precedence
- All command modes
- Error cases

**Validation**:

- All existing tests pass with updates
- New tests add coverage
- `make test-unit` passes
- `make coverage` shows >80%

---

### Step 9: Quality Gates

**Objective**: Production-ready validation.

**Run make targets** (in order):

1. `make format` - Auto-format all code
2. `make lint` - Zero linting errors
3. `make test-unit` - All unit tests pass
4. `make coverage` - Coverage >80% (note: old code excluded from coverage)

**Manual validation checklist**:

- [ ] Full workflow: init → fetch → csv
- [ ] Update workflow: fetch --all → csv --all  
- [ ] Failed PR tracking works
- [ ] State file format correct (full URLs, failed_prs list)
- [ ] CSV output byte-identical to old implementation (compare)
- [ ] Each command independently functional
- [ ] No circular dependencies (check architecture diagram)

**Documentation validation**:

- [ ] CONTEXT.md describes current codebase state
- [ ] README.md has new CLI examples
- [ ] ARCHITECTURE.md has dependency diagram
- [ ] MIGRATION_GUIDE.md complete

**Validation**:

- All checks pass
- Everything documented
- Code ready for review

---

### Step 10: Cleanup

**Objective**: Remove old code, finalize structure.

**Files to delete**:

- `src/gh_pr_metrics_old.py` (after confirming new code works)
- Any remaining dead code or unused imports in main file

**Final review**:

- Each source file focused and < 500 lines
- Clear module separation
- No global state issues
- Documentation complete

**Validation**:

- `make test-unit` still passes
- `make lint` clean
- Code structure matches plan

---

## Success Criteria

- [ ] Three independent commands (init, fetch, csv)
- [ ] Command classes in `src/commands/`
- [ ] Managers in `src/managers/`
- [ ] Data processors in `src/data/`
- [ ] No circular dependencies (verified by diagram)
- [ ] State management rules correctly implemented
- [ ] Failed PRs tracking working
- [ ] All 257+ tests passing
- [ ] make format, lint, test-unit, coverage all pass
- [ ] CSV output identical to old implementation
- [ ] Old code preserved during refactor (reference only)
- [ ] Documentation complete and accurate
- [ ] Plan saved to filesystem

## Risk Mitigation

**At each step**:

1. Run `make test-unit` - must pass
2. Run `make lint` - must be clean
3. Commit with descriptive message
4. Can rollback any individual step

**Testing strategy**:

- Use `make test-unit` not `make test` (no GitHub token for integration tests)
- Focus on unit tests with mocks
- Manual validation of end-to-end workflows

**Rollback plan**:

- Each step is a separate commit
- Can revert to any previous working state
- Old code preserved as `gh_pr_metrics_old.py` until Step 10

## Known Limitations

- `gh_pr_metrics_old.py` will show in codebase but have 0% coverage (expected, reference only)
- Integration tests require GitHub token (not run during refactor)
- Breaking changes to CLI (acceptable per requirements)

## Estimated Effort

- Step 0: ~10 calls, 15 min
- Step 1: ~40 calls, 60 min  
- Step 2: ~25 calls, 40 min
- Step 3: ~20 calls, 30 min
- Step 4: ~35 calls, 50 min
- Step 5: ~50 calls, 75 min
- Step 6: ~40 calls, 60 min
- Step 7: ~25 calls, 40 min
- Step 8: ~70 calls, 105 min
- Step 9: ~35 calls, 50 min
- Step 10: ~10 calls, 15 min

**Total**: ~360 tool calls, ~9 hours

Realistic timeline accounting for testing, validation, and potential issues: 10-12 hours.