---
allowed-tools: Bash, Read, Task, Glob, Grep
---

# Review

Review current branch changes for bugs, issues, and merge readiness.

## Usage
```
/pm:review
```

## Preflight Checklist

Before proceeding, complete these validation steps.
Do not bother the user with preflight checks progress. Just do them and move on.

1. **Check current branch:**
   ```bash
   git branch --show-current
   ```
   Note the current branch name.

2. **Check for changes against main:**
   ```bash
   git diff main...HEAD --stat
   ```
   If empty: "ℹ️ No changes to review. Current branch is up to date with main."
   Stop execution if no changes.

3. **Check for uncommitted changes:**
   ```bash
   git status --porcelain
   ```
   If not empty: "⚠️ You have uncommitted changes. These will NOT be included in the review. Commit them first if you want them reviewed."

## Instructions

### 1. Detect Context

Gather branch information:

```bash
# Current branch
branch=$(git branch --show-current)

# Changed files
git diff main...HEAD --name-only

# Diff summary
git diff main...HEAD --stat

# Commit history on this branch
git log main...HEAD --oneline
```

Display:
```
🔍 Reviewing branch: {branch}

Commits: {count} since main
Files changed: {file_count}
  {diff_stat_summary}
```

### 2. Launch Code Analyzer

Get the full diff and list of changed files, then launch the code-analyzer agent:

```yaml
Task:
  description: "Code review: {branch}"
  subagent_type: "general-purpose"
  prompt: |
    You are the code-analyzer agent. Read the agent definition at ccpm/agents/code-analyzer.md and follow its instructions.

    Review the following changes on branch: {branch}

    Changed files:
    {list_of_changed_files}

    Diff summary:
    {diff_stat}

    Commit history:
    {commit_log}

    Analyze all changed files for:
    1. Bugs and logic errors
    2. Security vulnerabilities
    3. Performance issues
    4. Code style inconsistencies
    5. Missing error handling
    6. Test coverage gaps

    Read each changed file and review the changes carefully.
    Produce your analysis in the standard code-analyzer output format.
```

### 3. Display Results

Present the code-analyzer's findings:

```
📊 Code Review Results: {branch}
══════════════════════════════════

{code_analyzer_output}

──────────────────────────────────
```

### 4. Gate Decision

Based on the analysis results, provide a merge readiness assessment:

**If no critical or high issues found:**
```
✅ Ready to merge

No critical issues found. Branch appears safe to merge.

Suggested next steps:
  • Push branch: git push -u origin {branch}
  • Create PR: gh pr create
  • Merge to main: git checkout main && git merge --no-ff {branch}
```

**If critical or high issues found:**
```
⚠️ Issues found — fix before merging

{count} issue(s) should be addressed before merging:
  {list_of_critical_and_high_issues}

After fixing:
  • Run /pm:review again to verify
  • Then proceed with merge
```

## Error Handling

If any step fails, report clearly:
- "❌ {What failed}: {How to fix}"
- If code-analyzer agent fails, fall back to showing the raw diff summary

## Important Notes

- This command is read-only — it does not modify any files or branches
- The review covers all commits on the current branch since it diverged from main
- For best results, commit all changes before running this command
