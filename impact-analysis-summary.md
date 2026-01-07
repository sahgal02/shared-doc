# Impact Analysis Command - Summary

## Input Types Handled

### 1. GitLab MR URLs (Highest Priority)
- Full URLs: `https://gitlab.*/-/merge_requests/{number}`
- Extracts MR number and fetches source/target branches via `glab` CLI or GitLab API

### 2. Explicit Branch Pairs (High Priority)
- Patterns: `source: {branch} target: {branch}`, `from {branch} to {branch}`, `{branch1} vs {branch2}`, `compare {branch1} and {branch2}`, `between {branch1} and {branch2}`

### 3. File-based
- Specific file paths (`.kt`, `.java`, `.xml`)
- Keywords: "analyze file", "impact of file", "files:"

### 4. Commit-based
- Commit hashes (SHA format)
- Commit ranges: `abc123..def456`, `HEAD~5..HEAD`

### 5. Working Directory
- Uncommitted changes: "uncommitted", "working directory", "current changes", "staged", "unstaged"

### 6. MR/PR-based (Without URL)
- Patterns: "MR #", "PR #", "merge request", "pull request", "#{number}"

### 7. Branch-based (Default)
- Branch names or "analyze branch"
- With target: "vs", "compared to", "against"
- Auto-detects base branch (bugs_{number} for bug branches, release_{number} for task/story branches)

---

## Analysis Performed

### Changed Files Identification
- Lists changed files using `git diff`
- Categorizes: source files, test files, layouts, resources, build/config files

### Module Impact Analysis
- Extracts affected modules from file paths
- Categorizes: UI modules (`*ui`), Domain modules (`*domain`), Shared modules, Player modules, DVB modules
- Determines impact scope: single module, multiple modules, cross-cutting, build system

### Feature Mapping
- Maps modules to features (Home, Detail, Login, Player, Search, EPG, etc.)
- Identifies cross-cutting features (shared modules, build system, analytics)

### Dependencies Analysis
- Finds files importing/using changed files
- Checks for interface/API changes (method signatures, visibility)
- Identifies breaking changes and migration requirements

### Test Files Analysis
- Finds corresponding test files for changed source files
- Checks test file existence and update status
- Identifies missing test files

### UI Components Analysis
- Identifies changed layouts and maps to Activities/Fragments
- Finds changed UI components (custom views, adapters, ViewHolders)
- Identifies changed resources (strings, drawables, colors, dimensions)

---

## Output Format

### Terminal Output (Concise Summary)
- Input type and metadata (branches, MR info)
- Summary stats (files changed, lines added/removed, modules affected)
- Affected modules with categories
- Affected features list
- Dependencies (files → used by)
- Test files status
- UI components affected
- Testing recommendations
- Report file path

### Markdown Report (Detailed)
- **Location**: `docs/analysis_report/`
- **Naming**: `{source-branch}-{target-branch(optional)}-{date}.md`
- **Date Format**: `YYYY-MM-DD`
- **Sections**:
  - Summary with counts
  - Changed files (categorized)
  - Affected modules (by category with descriptions)
  - Affected features (with testing focus)
  - Dependencies analysis (interfaces/APIs, import dependencies)
  - Test files analysis (existing, missing, infrastructure)
  - UI components analysis (layouts, components, resources)
  - Testing recommendations (high/medium/low priority, regression)
  - Testing checklist
  - Notes and observations

### File Naming Convention
- Branch-based: `{branch-name}-{date}.md`
- Branch pairs: `{source-branch}-{target-branch}-{date}.md`
- MR URLs: `{source-branch}-{target-branch}-{date}.md`
- File-based: `files-{sanitized-file-names}-{date}.md`
- Commit-based: `commits-{hash-range}-{date}.md`
- Working directory: `working-directory-{date}.md`
- Branch names sanitized: `/` → `-`

---

## Detection Priority Order

1. GitLab MR URLs
2. Explicit branch pair patterns
3. File paths
4. Commit hashes (SHA format)
5. MR/PR keywords and numbers
6. Working directory keywords
7. Branch names with explicit target ("vs", "compared to", "against")
8. Branch names
9. Default to branch-based if on a branch

---

## Use Cases

- Before creating MR: Understand what needs to be tested
- Code review preparation: Identify all affected areas
- Testing planning: Know which modules and features to focus on
- Regression testing: Understand scope of changes
- Analyzing specific files: Understand impact of changing specific files
- Analyzing commits: Understand impact of specific commits or commit ranges
- Analyzing uncommitted changes: Understand impact before committing
- Analyzing MRs: Understand impact of merge requests via URLs or branch pairs
- Comparing branches: Compare any two branches directly

---

## Related Files

- Command Definition: `.cursor/commands/impact-analysis.md`
- Report Directory: `docs/analysis_report/`
- Example Report: `docs/analysis_report/task-TTVTS-1730_jenkin_smoke_test_pipeline-release_2542000-2026-01-07.md`

