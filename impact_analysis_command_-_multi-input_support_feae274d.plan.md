---
name: Impact Analysis Command - Multi-Input Support
overview: "Create a Cursor command to identify impact areas of changes with support for multiple input types: file-based, commit-based, working directory, MR/PR-based, and branch-based (with automatic base branch detection when not specified). The command will analyze impact areas including affected modules, features, dependencies, test files, and UI components."
todos:
  - id: update-command-file-structure
    content: Update impact-analysis.md with input type detection section and multiple workflow paths
    status: completed
  - id: implement-input-type-detection
    content: Implement input type detection logic (file-based, commit-based, working directory, MR/PR-based, branch-based)
    status: completed
    dependencies:
      - update-command-file-structure
  - id: implement-file-based-analysis
    content: "Implement file-based analysis workflow: extract files, find dependencies, identify modules"
    status: completed
    dependencies:
      - implement-input-type-detection
  - id: implement-commit-based-analysis
    content: "Implement commit-based analysis workflow: parse commits, get changed files, analyze impact"
    status: completed
    dependencies:
      - implement-input-type-detection
  - id: implement-working-directory-analysis
    content: "Implement working directory analysis workflow: detect uncommitted changes, categorize, analyze"
    status: completed
    dependencies:
      - implement-input-type-detection
  - id: implement-mr-pr-based-analysis
    content: "Implement MR/PR-based analysis workflow: extract MR number, get branch info, analyze changes"
    status: completed
    dependencies:
      - implement-input-type-detection
  - id: enhance-branch-based-analysis
    content: "Enhance branch-based analysis: detect user-specified target branch, fallback to automatic detection"
    status: completed
    dependencies:
      - implement-input-type-detection
  - id: create-unified-impact-analysis
    content: Create unified impact analysis workflow that works with any set of changed files (shared logic)
    status: completed
    dependencies:
      - implement-file-based-analysis
      - implement-commit-based-analysis
      - implement-working-directory-analysis
      - implement-mr-pr-based-analysis
      - enhance-branch-based-analysis
  - id: update-output-format
    content: Update output format to support all input types with appropriate naming and reporting
    status: completed
    dependencies:
      - create-unified-impact-analysis
  - id: add-examples-for-all-input-types
    content: Add comprehensive examples for each input type in the command documentation
    status: completed
    dependencies:
      - update-output-format
---

# Impact Analysis Command Plan - Multi

-Input Support

## Overview

Create a Cursor command `.cursor/commands/impact-analysis.md` that analyzes code changes and identifies impact areas for testing. The command supports multiple input types and automatically detects the appropriate base branch when analyzing branches.

## Key Features

1. **Multiple Input Type Support**:

- **File-based**: Analyze specific files provided by user
- **Commit-based**: Analyze specific commits or commit range
- **Working directory**: Analyze uncommitted changes
- **MR/PR-based**: Analyze changes from a merge request
- **Branch-based**: Analyze changes between branches (with automatic base branch detection)

2. **Automatic Base Branch Detection** (for branch-based analysis):

- If user specifies target branch in prompt → use it
- Otherwise: Use same logic as `create-mr.sh`:
    - For `bug/` branches: First look for `bugs_{number}` branch, fallback to `release_{number}`
    - For `task/`/`story/` branches: Use latest `release_{number}` branch
    - Find latest branch using version sorting (`sort -V`)
    - Fallback to `main`/`master`/`develop` if no release branches found

3. **Comprehensive Impact Analysis**:

- Affected modules (domain/UI modules)
- Affected features
- Dependencies and related files
- Test files that might need updating
- UI components and layouts affected

4. **Output Format**: Both markdown report (saved to `docs/impact-analysis/`) and terminal output

## Input Type Detection

### Detection Logic

The command should detect input type based on user prompt:

1. **File-based**: User mentions specific file paths or "analyze file(s)"

- Examples: "analyze HomeViewModel.kt", "impact of detailui/src/...", "files: file1.kt file2.kt"

2. **Commit-based**: User mentions commit hashes or commit range

- Examples: "analyze commit abc123", "impact of commits abc123..def456", "commit range HEAD~5..HEAD"

3. **Working directory**: User mentions uncommitted changes or "working directory"

- Examples: "analyze uncommitted changes", "impact of current changes", "what's changed in working directory"

4. **MR/PR-based**: User mentions MR/PR number or merge request

- Examples: "analyze MR #123", "impact of merge request 456", "PR #789"

5. **Branch-based**: User mentions branch name or "branch" (default if no other type detected)

- Examples: "analyze branch feature/new-feature", "impact of bug/TTVTV-11594"
- If user specifies target branch: "analyze branch feature/x vs release_2542000" → use specified target
- If no target specified: Use automatic detection logic

## Implementation Steps

### Step 1: Update Command File Structure

Update `.cursor/commands/impact-analysis.md` with:

- Input type detection section
- Multiple workflow paths for each input type
- Unified impact analysis workflow (shared across all input types)
- Output formatting for all input types

### Step 2: Implement Input Type Detection

**Purpose**: Detect which input type the user wants to analyze**Detection Patterns**:

- File paths → File-based
- Commit hashes → Commit-based
- "uncommitted" / "working directory" → Working directory
- "MR" / "PR" / "#" → MR/PR-based
- Branch names → Branch-based
- "vs" / "compared to" → Branch-based with specified target

**Implementation**:

```markdown
## Input Type Detection

1. Check for file paths in prompt
2. Check for commit hashes (SHA format)
3. Check for MR/PR keywords
4. Check for working directory keywords
5. Check for branch names
6. Check for explicit target branch specification ("vs", "compared to")
7. Default to branch-based if on a branch
```



### Step 3: Implement File-Based Analysis

**Purpose**: Analyze impact of specific files**Actions**:

1. Extract file paths from user input
2. Verify files exist
3. Get file status (modified, added, deleted)
4. Find files that import/use these files (dependencies)
5. Identify modules containing these files
6. Run unified impact analysis

**Git Commands**:

```bash
# Get file status
git status --porcelain {file-path}

# Find files importing this file
grep -r "import.*{ClassName}" --include="*.kt" --include="*.java"

# Get file's module
# Extract from path: {module}/src/...
```



### Step 4: Implement Commit-Based Analysis

**Purpose**: Analyze impact of specific commits or commit range**Actions**:

1. Parse commit hash(es) or range from input
2. Verify commits exist
3. Get changed files from commit(s)
4. Run unified impact analysis

**Git Commands**:

```bash
# Single commit
git diff {commit-hash}^..{commit-hash} --name-only

# Commit range
git diff {start-commit}..{end-commit} --name-only

# Multiple commits
git diff {commit1}..{commit2} --name-only
```



### Step 5: Implement Working Directory Analysis

**Purpose**: Analyze impact of uncommitted changes**Actions**:

1. Check for uncommitted changes
2. Get changed files from working directory
3. Categorize changes (modified, added, deleted, untracked)
4. Run unified impact analysis

**Git Commands**:

```bash
# Get uncommitted changes
git diff --name-only
git diff --cached --name-only  # Staged changes
git status --porcelain  # All changes including untracked

# Get diff stats
git diff --stat
git diff --cached --stat
```



### Step 6: Implement MR/PR-Based Analysis

**Purpose**: Analyze impact of changes from a merge request**Actions**:

1. Extract MR/PR number from input
2. Get MR/PR information (source branch, target branch)
3. Get changed files from MR/PR diff
4. Run unified impact analysis

**Git Commands**:

```bash
# If using glab CLI
glab mr view {mr-number} --json

# Or fetch MR branch and compare
git fetch origin {source-branch}
git diff origin/{target-branch}...origin/{source-branch} --name-only
```



### Step 7: Implement Branch-Based Analysis (Enhanced)

**Purpose**: Analyze impact of changes between branches**Actions**:

1. Detect if user specified target branch in prompt

- Look for "vs", "compared to", "against" keywords
- Extract target branch name

2. If target branch specified → use it
3. If not specified → use automatic detection logic:

- Determine branch type from current branch
- Find appropriate base branch (bugs_{number} or release_{number})
- Use latest branch with version sorting

4. Get changed files between branches
5. Run unified impact analysis

**Git Commands**:

```bash
# User-specified target branch
git diff origin/{target-branch}...HEAD --name-only

# Automatic detection (same as create-mr.sh)
# Find bugs_{number} or release_{number} branches
# Compare with detected base branch
git diff origin/{detected-base-branch}...HEAD --name-only
```



### Step 8: Unified Impact Analysis Workflow

**Purpose**: Shared analysis logic for all input types**Steps** (same as before, but works with any set of changed files):

1. Categorize changed files by type
2. Identify affected modules
3. Map modules to features
4. Identify dependencies
5. Identify test files
6. Identify UI components
7. Generate testing recommendations

### Step 9: Update Output Format

**Purpose**: Adapt output format for different input types**Terminal Output**:

- Show input type detected
- Show input source (files, commits, branch, etc.)
- Show impact analysis results
- Show testing recommendations

**Markdown Report**:

- Save with appropriate naming based on input type:
- File-based: `docs/impact-analysis/files_{file-names}_{timestamp}.md`
- Commit-based: `docs/impact-analysis/commits_{hash-range}_{timestamp}.md`
- Working directory: `docs/impact-analysis/working-directory_{timestamp}.md`
- MR/PR-based: `docs/impact-analysis/mr-{number}_{timestamp}.md`
- Branch-based: `docs/impact-analysis/{branch-name}_{timestamp}.md`

## Command Workflow

### Workflow for All Input Types

1. **Detect Input Type**: Parse user prompt to identify input type
2. **Extract Input Details**: Get files/commits/branch/MR from prompt
3. **Get Changed Files**: Use appropriate git command based on input type
4. **Run Unified Impact Analysis**: Analyze changed files (shared logic)
5. **Generate Report**: Create terminal output and markdown report
6. **Display Results**: Show summary in terminal, save detailed report

### Branch-Based Specific Workflow

1. **Check for Specified Target Branch**: Look for "vs", "compared to" in prompt
2. **If Specified**: Use user-provided target branch
3. **If Not Specified**: Run automatic detection:

- Get current branch
- Determine branch type (bug/task/story)
- Find appropriate base branch (bugs_{number} or release_{number})
- Use latest branch with version sorting

4. **Get Changed Files**: `git diff origin/{target-branch}...HEAD`
5. **Run Unified Impact Analysis**

## Example Usage Scenarios

### File-Based

```javascript
User: "Analyze impact of HomeViewModel.kt and DetailRepository.kt"
→ Detect: File-based
→ Extract: homeui/src/.../HomeViewModel.kt, detaildomain/src/.../DetailRepository.kt
→ Analyze: Find dependencies, modules, features
```



### Commit-Based

```javascript
User: "What's the impact of commit abc123def?"
→ Detect: Commit-based
→ Extract: abc123def
→ Analyze: git diff abc123def^..abc123def
```



### Working Directory

```javascript
User: "Analyze uncommitted changes"
→ Detect: Working directory
→ Analyze: git diff + git diff --cached
```



### MR/PR-Based

```javascript
User: "Analyze impact of MR #123"
→ Detect: MR-based
→ Extract: MR #123
→ Get: Source branch, target branch
→ Analyze: git diff origin/{target}...origin/{source}
```



### Branch-Based (Specified Target)

```javascript
User: "Analyze branch feature/new-feature vs release_2542000"
→ Detect: Branch-based with specified target
→ Extract: feature/new-feature (source), release_2542000 (target)
→ Analyze: git diff origin/release_2542000...origin/feature/new-feature
```



### Branch-Based (Auto-Detection)

```javascript
User: "Analyze impact of current branch"
→ Detect: Branch-based, no target specified
→ Current branch: bug/TTVTV-11594
→ Auto-detect: bugs_2542000 (latest bugs branch)
→ Analyze: git diff origin/bugs_2542000...HEAD
```



## Related Files

- `.cursor/commands/raise-merge-request.md` - Base branch detection logic
- `.cursor/commands/bug-debugging.md` - Multi-input type detection patterns
- `.cursor/commands/code-review.md` - Code review patterns
- `.cursor/rules/project/project-structure.mdc` - Module organization