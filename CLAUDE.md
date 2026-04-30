# gh-pr-metrics

CLI tool that generates CSV reports of GitHub PR metrics (comments, reviews, approvals, timestamps) for repositories over a configurable time range.

## Build & Test

```bash
make check          # Run all quality checks (format, lint, typecheck, test, coverage)
make format         # Auto-format with black
make test-unit      # Run pytest only
make test-complexity # Check cyclomatic complexity with xenon
make help           # Show all targets
```

Default target is `check` (read-only, safe for automation).

## Project Layout

- `src/gh_pr_metrics.py` — main CLI (single-file, ~3000 lines)
- `src/gh_pr_extract_comments.py` — secondary CLI for extracting detailed comment data
- `src/github_api.py` — GitHub REST API client with pagination, rate limiting, quota tracking
- `src/utils.py` — shared utilities
- `tests/unit/` — pytest unit tests (292 tests, 80% coverage, 75% threshold)
- `tests/integration/` — integration workflow tests
- `tests/conftest.py` — autouse fixtures: blocks subprocess, HTTP (socket-level), isolates state file and raw data dir
- `make/env.mk` — venv/dependency targets
- `make/test.mk` — test-* targets (unit, coverage, format, lint, typecheck, complexity)
- `utility/create-single-csv.py` — merges per-repo CSVs into aggregate files

## Key Conventions

- **Line length**: 88 (black default). flake8 ignores E501 since black is the authority
- **Exit codes**: `EXIT_SUCCESS` (0), `EXIT_ERROR` (1), `EXIT_ERROR_REPO_ACCESS` (2)
- **State file**: `~/.gh-pr-metrics-state.yaml` — YAML, atomic writes with backup recovery
- **Config file**: `.gh-pr-metrics.yaml` — AI bot patterns, workers, output pattern, quota settings
- **Module-level globals**: `github_client` (Optional, set in main()), `quota_manager`, `state_manager`, `csv_manager`, `logger`
- **Test guards**: subprocess blocked, socket.create_connection blocked (compatible with requests_mock), state file isolated to tmp_path

## CLI Entry Points

Defined in `pyproject.toml [project.scripts]`:
- `gh-pr-metrics` → `gh_pr_metrics:main`
- `gh-pr-extract-comments` → `gh_pr_extract_comments:main`

## API & Data Flow

Uses GitHub REST API v3 with token from `GITHUB_TOKEN` env var. Pagination via Link headers. Date ranges >30 days auto-chunk into 30-day windows with 1-day overlap and PR deduplication. Quota-aware: estimates API calls needed before starting, preserves progress per-chunk in state file.

## Documentation

- [README.md](README.md) — usage, installation, examples
- [docs/requirements.md](docs/requirements.md) — functional and non-functional requirements
- [docs/unit-tests.md](docs/unit-tests.md) — test strategy
- [docs/flow.md](docs/flow.md) — data flow documentation
- [docs/instructions.md](docs/instructions.md) — development instructions
