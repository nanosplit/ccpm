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

### 2. Dual Review

Launch two independent reviews in parallel. These catch different classes of problems and together provide comprehensive coverage.

#### 2a. Spec Compliance Review

If the branch is associated with a task (detected from branch name pattern `task/*`), launch the spec-reviewer to check if the implementation matches the task's acceptance criteria:

```yaml
Task:
  description: "Spec review: {branch}"
  subagent_type: "general-purpose"
  prompt: |
    You are the spec-reviewer agent. Read the agent definition at .claude/agents/spec-reviewer.md and follow its instructions.

    Task file: {task_file_path}
    Task specification:
    {task_description}

    Acceptance criteria:
    {acceptance_criteria}

    Changed files:
    {list_of_changed_files}

    Commit history:
    {commit_log}

    Independently verify that the code changes satisfy every acceptance criterion.
    Read the actual code — do not trust commit messages as evidence of completion.
```

If no task file is associated with the branch, skip the spec review and note: "ℹ️ No task file found for this branch — skipping spec compliance review."

#### 2b. Code Quality Review

Launch the code-analyzer agent to review for bugs, security, and quality (runs in parallel with spec review):

```yaml
Task:
  description: "Quality review: {branch}"
  subagent_type: "general-purpose"
  prompt: |
    You are the code-analyzer agent. Read the agent definition at .claude/agents/code-analyzer.md and follow its instructions.

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

Present findings from both reviewers:

```
📊 Dual Review Results: {branch}
══════════════════════════════════

Spec Compliance:
  Verdict: {PASS/FAIL/CONDITIONAL PASS/Skipped}
  {criteria summary table or skip reason}

Code Quality:
  Risk Level: {Critical/High/Medium/Low}
  {code_analyzer_findings}

──────────────────────────────────
```

### 4. Gate Decision

Based on the combined results from both reviews:

**If both reviews pass (no critical or high issues):**
```
✅ Ready to merge

No critical issues found across both reviews.

Suggested next steps:
  • Push branch: git push -u origin {branch}
  • Create PR: gh pr create
  • Merge to main: git checkout main && git merge --no-ff {branch}
```

**If either reviewer found critical or high issues:**
```
⚠️ Issues found — fix before merging

{section for each reviewer that found issues}

Spec compliance issues:
  {list if any}

Code quality issues:
  {list if any}

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
