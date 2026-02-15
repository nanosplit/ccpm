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

### 1. Setup Branch

```bash
branch_name="feature/$ARGUMENTS"

if git branch -a | grep -q "$branch_name"; then
  git checkout "$branch_name"
  git pull origin "$branch_name" 2>/dev/null || true
  echo "✅ Using existing branch: $branch_name"
else
  git checkout main
  git pull origin main
  git checkout -b "$branch_name"
  echo "✅ Created branch: $branch_name"
fi
```

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

Execute the researcher's change plan directly — no approval step:
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

#### 3f. Record Result

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Update task frontmatter:
```yaml
status: closed
updated: {current_datetime}
```

Log the result briefly:
```
  ✅ Task {number}: {task_name}
     Files: {count} changed
     Tests: {pass/fail/skipped}
     Commits: {count}
```

If tests failed:
```
  ⚠️ Task {number}: {task_name} — tests failed
     {brief failure summary}
     Continuing to next task...
```

Do NOT stop on test failures. Log them and continue. The user will review everything at the end.

#### 3g. Advance

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
