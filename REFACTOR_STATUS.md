# Subcommand Refactor - Implementation Status

**Date**: 2025-12-08
**Status**: In Progress - Complexity Assessment

## Progress

✅ Design document created (SUBCOMMAND_REFACTOR.md)
⏳ Implementation in progress
- Attempted full refactor of argument parser
- Encountered challenges with tightly coupled code

## Key Challenge Discovered

The current `process_repository` function (lines 1255-1463) tightly couples:
1. Fetching PR data from GitHub API
2. Processing metrics  
3. Writing CSV files
4. Writing JSON files (added recently)
5. State management

Splitting into `fetch` and `csv` subcommands requires:
- **fetch subcommand**: API → JSON + state (NO CSV)
- **csv subcommand**: JSON → CSV processing (NO API)

This means extracting and separating logic from `process_repository`, which is non-trivial.

## Implementation Strategy Options

### Option A: Big Bang Refactor (Current Attempt)
- Replace argument parser with subcommands
- Split `process_repository` into fetch and CSV functions
- Update all tests
- **Risk**: High - many moving parts, easy to break
- **Time**: Multiple hours, many edits

### Option B: Incremental with Feature Flag
1. Add new subcommand parser alongside old parser
2. Keep old behavior as default, new subcommands opt-in
3. Implement fetch subcommand (API → JSON only)
4. Implement csv subcommand (JSON → CSV)
5. Switch default behavior
6. Remove old code
- **Risk**: Medium - can validate each step
- **Time**: More edits but safer

### Option C: New File Approach
1. Create `gh_pr_metrics_v2.py` with clean structure
2. Copy/adapt logic piece by piece
3. Test thoroughly
4. Replace old file
5. Update entry point
- **Risk**: Low - old code untouched until ready
- **Time**: Similar to Option A but cleaner

## Recommendation

**Option C** - Create new file with clean structure, test independently, then replace.

This avoids:
- Corrupting working code during development
- Complex merge conflicts in 2000+ line file
- Difficult-to-debug intermediate states

## Next Steps

1. Pause current inline refactor
2. Create `src/gh_pr_metrics_refactored.py` skeleton
3. Implement subcommand handlers with clean separation
4. Add tests for new structure
5. Validate against existing test suite
6. Replace old file

## Files Modified So Far

- `src/gh_pr_metrics.py` - reverted (backup in `src/gh_pr_metrics_backup.py`)
- No other files modified yet

## Estimated Effort Remaining

- 50-100 more tool calls to complete refactor
- 2-4 hours of focused work
- High complexity due to code coupling

---

**Note**: This refactor is significantly more complex than anticipated due to tight coupling between fetch and process logic. The design document correctly identified the separation needed, but the implementation requires careful extraction of intertwined logic.

