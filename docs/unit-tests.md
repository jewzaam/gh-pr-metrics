# Unit Tests Documentation

## Overview

The `gh-pr-metrics` project maintains comprehensive unit test coverage to ensure reliability and prevent regressions. All tests are located in `tests/unit/` and use pytest as the test framework.

**Current Status:**
- **Test Count**: 147 tests
- **Coverage**: 82% (minimum threshold: 75%)
- **Test Framework**: pytest with fixtures and mocks
- **CI Integration**: Tests run on every commit

## Test Organization

### Test Files

| File | Purpose | Test Count |
|------|---------|-----------|
| `test_argument_validation.py` | CLI argument combination rules | 10 |
| `test_argument_parsing.py` | Argument parsing logic | ~8 |
| `test_chunking.py` | PR chunking for quota management | ~6 |
| `test_csv_output.py` | CSV read/write operations | ~12 |
| `test_date_handling.py` | Timestamp parsing and validation | ~8 |
| `test_github_client.py` | GitHub API interactions | ~15 |
| `test_init_mode.py` | Repository initialization | ~10 |
| `test_main.py` | Main application flows | ~25 |
| `test_pr_processing.py` | PR metric extraction | ~20 |
| `test_progress_summary.py` | Progress reporting | 2 |
| `test_quota_manager.py` | API quota management | ~12 |
| `test_repo_detection.py` | Git repository auto-detection | ~6 |
| `test_state_management.py` | State file operations | ~10 |
| `test_utils.py` | Utility functions | ~3 |

## Test Fixtures (conftest.py)

### Isolation Fixtures

All tests run with automatic isolation to prevent cross-test pollution:

#### `isolate_state_file` (autouse)
- **Purpose**: Prevents tests from touching real `~/.gh-pr-metrics-state.yaml`
- **Behavior**: Each test gets isolated temp state file
- **Scope**: Function-level

#### `reset_quota_manager` (autouse)
- **Purpose**: Resets quota to fresh state (5000/5000) before each test
- **Behavior**: Prevents quota depletion from affecting subsequent tests
- **Scope**: Function-level

#### `disable_file_logging` (autouse)
- **Purpose**: Prevents tests from creating `gh-pr-metrics.log`
- **Behavior**: Patches `setup_logging()` to only log to stderr
- **Scope**: Function-level

### Why Autouse Fixtures?

These fixtures use `autouse=True` because:
1. **State isolation** is critical for test reliability
2. **Quota reset** prevents cascading failures
3. **No log pollution** keeps test environment clean

Without these fixtures, tests would:
- Overwrite each other's state files
- Fail due to depleted quota from previous tests
- Create large log files during test runs

## Test Categories

### Argument Validation Tests

**File**: `test_argument_validation.py`

Tests all CLI argument combination rules:
- `--wait` requires `--update` or `--update-all`
- `--update` restrictions (no `--start`, `--end`, `--output`)
- `--update-all` restrictions (no `--owner`, `--repo`, `--start`, `--end`, `--output`)
- `--init` requirements
- Date range validation (`start < end`)

### Functional Tests

**Files**: `test_main.py`, `test_pr_processing.py`

Tests core business logic:
- PR processing with concurrent workers
- Multi-chunk processing with quota management
- Update mode vs regular mode behavior
- Empty repository handling (no CSV created)
- Quota exhaustion mid-processing (returns error code 1)

### API Integration Tests

**File**: `test_github_client.py`

Tests GitHub API interactions:
- PR fetching with pagination
- Date range filtering (start/end dates)
- Descending fetch optimization
- API error handling

### State Management Tests

**File**: `test_state_management.py`

Tests persistence layer:
- State file CRUD operations
- Repository tracking
- Timestamp updates
- Malformed state handling

### CSV Tests

**File**: `test_csv_output.py`

Tests CSV operations:
- Header-only CSV files
- Merge mode (update existing)
- Stdout vs file output
- Field ordering and completeness

## Running Tests

### Run All Tests
```bash
make test
```

### Run Specific Test File
```bash
pytest tests/unit/test_argument_validation.py -v
```

### Run Specific Test
```bash
pytest tests/unit/test_main.py::TestProcessRepository::test_returns_error_when_quota_exhausted_mid_processing -v
```

### Run with Coverage
```bash
make coverage
```

This runs tests AND enforces 75% minimum coverage threshold.

### Debug Failed Test
```bash
pytest tests/unit/test_main.py::test_name -vv --log-cli-level=DEBUG
```

## Writing New Tests

### Test Naming Convention

- **Test files**: `test_<module>.py`
- **Test classes**: `Test<Feature>`
- **Test functions**: `test_<behavior>`

Example:
```python
class TestArgumentValidation:
    def test_wait_requires_update_mode(self):
        """Test that --wait requires --update or --update-all."""
        ...
```

### Test Structure

Follow AAA pattern:
1. **Arrange**: Set up test data and mocks
2. **Act**: Execute the code under test
3. **Assert**: Verify expected behavior

Example:
```python
def test_no_csv_created_when_no_prs(self, tmp_path, requests_mock):
    # Arrange
    output_file = tmp_path / "output.csv"
    requests_mock.get("https://api.github.com/repos/test/test/pulls", json=[])
    
    # Act
    exit_code, _, _ = process_repository(...)
    
    # Assert
    assert exit_code == 0
    assert not output_file.exists()
```

### Mocking Guidelines

#### Mock External Dependencies
Always mock:
- GitHub API calls (use `requests_mock` fixture)
- File system operations (use `tmp_path` fixture)
- Current time (when testing time-based calculations)

#### Don't Mock
Avoid mocking:
- Internal business logic
- Simple data transformations
- Pure functions

#### Example: Mocking GitHub API
```python
def test_fetch_prs(self, requests_mock):
    requests_mock.get(
        "https://api.github.com/repos/owner/repo/pulls",
        json=[{"number": 1, "title": "Test PR"}]
    )
    
    result = github_client.fetch_prs("owner", "repo")
    assert len(result) == 1
```

## Coverage Requirements

### Minimum Threshold
- **Current**: 82%
- **Enforced**: 75%
- **Command**: `make coverage`

### Coverage Report
```bash
make coverage-report  # Generate HTML report in htmlcov/
```

View detailed coverage:
```bash
open htmlcov/index.html  # macOS
xdg-open htmlcov/index.html  # Linux
```

### What's NOT Covered

Some code is intentionally not covered by unit tests:
- Error handling for catastrophic failures
- Argcomplete integration (optional dependency)
- Debug logging paths
- Edge cases requiring integration tests

## Continuous Integration

Tests run automatically on:
- Every commit (via pre-commit hook)
- Pull requests (via GitHub Actions)
- Before release builds

CI enforces:
- All tests pass
- Coverage ≥ 75%
- No linter errors
- Code formatting (black, isort)

## Common Test Patterns

### Testing Argument Validation
```python
def test_invalid_arg_combination(self):
    with mock.patch.object(sys, "argv", ["gh-pr-metrics", "--bad-combo"]):
        result = gh_pr_metrics.main()
        assert result == 1  # Error exit code
```

### Testing State Management
```python
def test_state_update(self, tmp_path):
    state_file = tmp_path / "state.yaml"
    with mock.patch.object(state_manager, "_state_file", state_file):
        state_manager.update_repo("owner", "repo", timestamp, csv_file)
        state = state_manager.load()
        assert "https://github.com/owner/repo" in state
```

### Testing Quota Exhaustion
```python
def test_quota_exhausted(self):
    with mock.patch.object(quota_manager, 'check_sufficient', return_value=(False, 0)):
        exit_code, _, _ = process_repository(...)
        assert exit_code == 1
```

## Troubleshooting

### Test Fails Locally But Passes in CI
- Check for state file pollution: ensure `conftest.py` fixtures are working
- Check for timezone issues: use `timezone.utc` explicitly
- Check for quota state: verify `reset_quota_manager` fixture runs

### Quota Exhaustion in Tests
If you see quota-related failures:
1. Verify `reset_quota_manager` fixture is running
2. Check that all GitHub API calls are mocked
3. Ensure `requests_mock` fixture is used when needed

### State File Conflicts
If tests interfere with each other:
1. Verify `isolate_state_file` fixture is running
2. Load state inside mock context (not after)
3. Use `tmp_path` fixture for test-specific files

## Known Gaps

### P2 - Medium Priority (Future Work)
- Descending fetch strict ordering across multiple pages
- Multi-chunk processing with truncation edge cases
- `days_in_review` for closed (non-merged) PRs
- Negative time values (clock skew edge cases)

### P3 - Low Priority
- File logging behavior verification
- Directory creation during init
- Progress summary calculation accuracy

Most critical functionality (P0/P1) is already tested.

