---
allowed-tools: Bash, Read, Write, LS, Task, Glob, Grep
---

# Autopilot

Unattended execution of all tasks in an epic. Researches, implements, tests, and advances through each task automatically — no confirmation prompts. Creates a PR when done.

## Usage
```
/pm:autopilot <epic-name>
```

## Required Rules

**IMPORTANT:** Before executing this command, read and follow:
- `.claude/rules/datetime.md` - For getting real current date/time
- `.claude/rules/branch-operations.md` - For git branch operations
- `.claude/rules/frontmatter-operations.md` - For frontmatter updates
- `.claude/rules/verification-before-completion.md` - For verification gate before marking tasks closed
- `.claude/rules/rationalization-prevention.md` - For catching shortcut rationalizations (especially the Unattended Mode section)
- `.claude/rules/testing-discipline.md` - For choosing the right testing tier per task

## Preflight Checklist

Before proceeding, complete these validation steps.
Do not bother the user with preflight checks progress. Just do them and move on.

1. **Validate epic name was provided:**
   - If not: "❌ Epic name required. Usage: /pm:autopilot <epic-name>"

2. **Verify epic exists:**
   - Check if `.claude/epics/$ARGUMENTS/epic.md` exists
   - If not: "❌ Epic not found: $ARGUMENTS. Create it first with /pm:start-feature $ARGUMENTS"

3. **Check for uncommitted changes:**
   ```bash
   git status --porcelain
   ```
   If not empty: "❌ You have uncommitted changes. Commit or stash them before running autopilot."

4. **Load all tasks:**
   - Read all numbered `.md` files in `.claude/epics/$ARGUMENTS/`
   - Parse frontmatter for status, depends_on, name
   - Build dependency graph
   - Identify tasks that are `open` or `in-progress` (skip `closed`/`completed`)
   - If no remaining tasks: "✅ All tasks in $ARGUMENTS are already complete."

## Instructions

**CRITICAL: This is unattended mode. Do NOT ask the user for confirmation at any point. Do NOT present plans for approval. Research, implement, test, and advance automatically.**

**EQUALLY CRITICAL: Unattended does not mean undisciplined.** No human is watching, which makes it more important — not less — to follow every step rigorously. Do not skip research, do not skip testing, do not skip verification. See `.claude/rules/rationalization-prevention.md` — Unattended Mode Shortcuts. Every rationalization for skipping a step is amplified when no one is watching.

### 1. Setup Branch

Find the right branch to work on. Check for existing branches in this order:

```bash
# 1. Check for existing feature branch
# 2. Check for existing task branches with work from this epic
# 3. Fall back to creating from main

branch_name="feature/$ARGUMENTS"

if git branch -a | grep -q "$branch_name"; then
  # Feature branch exists — use it
  git checkout "$branch_name"
  git pull origin "$branch_name" 2>/dev/null || true
  echo "✅ Using existing branch: $branch_name"
else
  # No feature branch. Check if there are task branches with work from this epic.
  # Look for task/* branches that might contain prior work.
  existing_task_branch=$(git branch -a | grep -E "task/.*$ARGUMENTS" | head -1 | xargs 2>/dev/null || true)

  if [ -n "$existing_task_branch" ]; then
    # Found a task branch with prior work — create feature branch from it
    clean_branch=$(echo "$existing_task_branch" | sed 's/remotes\/origin\///' | xargs)
    git checkout "$clean_branch"
    git pull origin "$clean_branch" 2>/dev/null || true
    git checkout -b "$branch_name"
    echo "✅ Created branch: $branch_name (from $clean_branch with prior work)"
  else
    # No prior work — create from main
    git checkout main
    git pull origin main
    git checkout -b "$branch_name"
    echo "✅ Created branch: $branch_name (from main)"
  fi
fi
```

**Important:** If the current branch already has commits for this epic (e.g., you ran `/pm:work-on` for earlier tasks), stay on the current branch instead of switching. Check:
```bash
current_branch=$(git branch --show-current)
```
If the current branch contains work for this epic and is NOT main, use it directly by renaming or continuing on it.

### 2. Display Plan

Show what will be executed, then begin immediately:

```
🤖 Autopilot: $ARGUMENTS
══════════════════════════════════════

Branch: {branch_name}
Total tasks: {total}
Remaining: {remaining}
Task order: {ordered list based on dependencies}

Starting unattended execution...
```

### 3. Task Loop

Process tasks in dependency order. For each task:

#### 3a. Select Next Task

Find the next ready task:
- Status is `open` or `in-progress`
- All tasks in `depends_on` are `closed` or `completed`
- Lowest task number among ready tasks

If no ready task but uncompleted tasks remain: some tasks are blocked. Report and stop.

#### 3b. Mark In-Progress

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Update task frontmatter:
```yaml
status: in-progress
updated: {current_datetime}
```

Display:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 Task {number}/{total}: {task_name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 3c. Research

Launch the researcher agent to analyze the codebase and produce a change plan:

```yaml
Task:
  description: "Research: {task_name}"
  subagent_type: "general-purpose"
  prompt: |
    You are the researcher agent. Read the agent definition at .claude/agents/researcher.md and follow its instructions.

    Task file: {task_file_path}
    Task requirements:
    {task_description}

    Acceptance criteria:
    {acceptance_criteria}

    Technical details:
    {technical_details}

    Analyze the codebase and produce a file-by-file change plan following the researcher agent format.
    Focus on: what files to change, what to change in each, dependency order, and risks.
```

#### 3d. Implement

Follow the testing tier recommended by the researcher (see `.claude/rules/testing-discipline.md`):

```
Testing discipline: Tier {1/2/3} — {Full TDD / Test-after / Verify-only}
```

**Tier 1 (Full TDD):** For each change unit, write a failing test first, then implement to pass it, then commit both together.

**Tier 2 (Test-after):** Implement all changes, then write tests covering new/changed behavior.

**Tier 3 (Verify-only):** Implement directly — existing test suite provides coverage.

For all tiers:
- Follow the dependency ordering from the plan
- Make changes file by file
- Commit after each logical unit of work: `Task {number}: {specific change}`
- If something is unclear, make a reasonable decision and note it — do not stop to ask

#### 3e. Test

Launch the test-runner agent:

```yaml
Task:
  description: "Test: {task_name}"
  subagent_type: "general-purpose"
  prompt: |
    You are the test-runner agent. Read the agent definition at .claude/agents/test-runner.md and follow its instructions.

    Run relevant tests for the changes made to implement: {task_name}

    Changed files:
    {list_of_changed_files}

    Run the project's test suite and report results.
    If no test infrastructure exists, note this and skip.
```

#### 3f. Dual Review

Launch both reviewers in parallel to catch spec and quality issues before verification.

**Spec compliance review** — launch the spec-reviewer agent:

```yaml
Task:
  description: "Spec review: {task_name}"
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

    Implementation summary:
    {brief_summary_of_what_was_implemented}

    Independently verify that the code changes satisfy every acceptance criterion.
    Do not trust the implementation summary — read the actual code.
```

**Code quality review** — launch the code-analyzer agent (in parallel):

```yaml
Task:
  description: "Quality review: {task_name}"
  subagent_type: "general-purpose"
  prompt: |
    You are the code-analyzer agent. Read the agent definition at .claude/agents/code-analyzer.md and follow its instructions.

    Review the changes made to implement: {task_name}

    Changed files:
    {list_of_changed_files}

    Analyze all changed files for bugs, security issues, and pattern violations.
    Produce your analysis in the standard code-analyzer output format.
```

**Evaluate results in autopilot mode:**
- **Critical issues from either reviewer:** Attempt to fix and re-review (up to 1 re-review cycle to keep progress moving)
- **Important/Minor issues only:** Log them in the task result but continue — the user will review in the PR
- **Both pass cleanly:** Proceed to verification

Log review results briefly:
```
  Review: Spec {PASS/FAIL} | Quality {risk level}
  {if issues: count and severity}
```

#### 3g. Verify Completion

**Follow `.claude/rules/verification-before-completion.md` — this gate is mandatory even in autopilot.**

Before marking a task closed:

1. **Re-read the task's acceptance criteria** from the task file
2. **Run fresh verification** — run tests/build again even if the test-runner agent just ran them
3. **Check each acceptance criterion** against real output
4. **If verification passes:** proceed to record result as closed
5. **If verification fails:** attempt to fix the issue and re-verify (up to 2 attempts in autopilot mode to keep progress moving). If still failing after 2 attempts, mark the task as `in-progress` (not closed), log the failure, and continue to the next task.

#### 3h. Record Result

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

If verification passed — update task frontmatter:
```yaml
status: closed
updated: {current_datetime}
```

Log the result briefly:
```
  ✅ Task {number}: {task_name}
     Files: {count} changed
     Tests: {pass/fail/skipped}
     Verification: passed
     Commits: {count}
```

If verification failed after retries:
```
  ⚠️ Task {number}: {task_name} — verification failed
     {brief failure summary}
     Status: left as in-progress
     Continuing to next task...
```

Do NOT mark a task as closed if verification failed. Leave it as `in-progress` so the user knows it needs attention. Continue to the next task.

#### 3i. Advance

Go back to step 3a for the next task.

### 4. Push and Create PR

After all tasks are processed:

```bash
git push -u origin {branch_name}
```

Create a pull request:

```bash
gh pr create --title "{epic_name}: {brief description}" --body "$(cat <<'PREOF'
## Summary

Autopilot execution of epic: {epic_name}

### Tasks Completed
- {task 1 name} ✅
- {task 2 name} ✅
- {task 3 name} ⚠️ (tests failed)
- ...

### Files Changed
{consolidated list of all changed files}

### Test Results
- Passed: {count}
- Failed: {count}
- Skipped: {count}

### Notes
{any decisions made, issues encountered, or warnings}

---
🤖 Generated by CCPM Autopilot
PREOF
)"
```

### 5. Final Summary

```
🤖 Autopilot Complete: $ARGUMENTS
══════════════════════════════════════

Branch: {branch_name}
PR: {pr_url}

Tasks: {completed}/{total} completed
  {list of tasks with status}

Files changed: {total_files}
Commits: {total_commits}
Test results: {summary}

{if any failures}
⚠️ Issues to review:
  {list of tasks with test failures or problems}
{end if}

Review the PR: {pr_url}
```

## Error Handling

- **Task implementation fails:** Log the error, mark task as `in-progress` (not closed), continue to next task
- **No ready tasks but uncompleted tasks remain:** Report blocked tasks and stop
- **Git conflicts:** Attempt to resolve automatically. If unable, log and continue with other tasks
- **Agent failures:** Fall back to direct implementation without research phase

Do NOT stop execution for recoverable errors. Log everything and keep going. The user will review the full PR.

## Important Notes

- This command runs unattended — never prompt for input
- Make reasonable decisions when facing ambiguity
- Commit frequently with descriptive messages
- Log all decisions, warnings, and issues in the PR body
- Test failures are warnings, not blockers — the user reviews everything at the end
- If a task seems too risky to implement without human input, skip it and note why
