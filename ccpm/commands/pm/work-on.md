---
allowed-tools: Bash, Read, Write, LS, Task, Glob, Grep
---

# Work On

Begin focused work on a single task — research, plan, implement, and test.

## Usage
```
/pm:work-on <task-identifier>
```

Where `<task-identifier>` can be:
- A GitHub issue number (e.g., `1234`)
- A local task reference (e.g., `feature-name/003`)

## Required Rules

**IMPORTANT:** Before executing this command, read and follow:
- `.claude/rules/datetime.md` - For getting real current date/time
- `.claude/rules/branch-operations.md` - For git branch operations

## Preflight Checklist

Before proceeding, complete these validation steps.
Do not bother the user with preflight checks progress. Just do them and move on.

### Input Validation
1. **Validate task identifier was provided:**
   - If not, tell user: "❌ Task identifier required. Usage: /pm:work-on <issue-number> or /pm:work-on <epic-name/task-number>"

2. **Resolve the task file:**
   - If identifier looks like a number: search for `.claude/epics/*/$ARGUMENTS.md` (new naming) or file with `github:.*issues/$ARGUMENTS` in frontmatter (old naming)
   - If identifier contains `/`: treat as `<epic-name>/<task-number>` and look for `.claude/epics/<epic-name>/<task-number>.md`
   - If not found: "❌ No task found for '$ARGUMENTS'. Check with: /pm:epic-show <epic-name>"

3. **Parse task frontmatter:**
   - Read status, name, depends_on, github fields
   - If status is `closed` or `completed`: "⚠️ Task '$ARGUMENTS' is already closed. Reopen it first or choose another task."

4. **Check dependencies:**
   - If `depends_on` is not empty, check each dependency's status
   - If any dependency is not closed/completed: "⚠️ Task has unmet dependencies: {list}. Complete those first or use /pm:next to find ready tasks."

5. **Check for uncommitted changes:**
   ```bash
   git status --porcelain
   ```
   If not empty: "⚠️ You have uncommitted changes. Commit or stash them before starting a new task."

## Instructions

### 1. Resolve Task

Read the task file and extract:
- Task name and description
- Acceptance criteria
- Technical details
- Epic name (from file path)
- GitHub issue number (from frontmatter `github` field, if present)

Display:
```
📋 Task: {task_name}
   Epic: {epic_name}
   Issue: #{github_number} (if synced)
   Status: {current_status}
```

### 2. Create or Switch to Branch

Follow `/rules/branch-operations.md`:

```bash
# Determine branch name
# If GitHub issue number exists: task/<issue>-<slugified-name>
# Otherwise: task/<epic>-<task-number>

branch_name="task/{identifier}"

if git branch -a | grep -q "$branch_name"; then
  git checkout "$branch_name"
  git pull origin "$branch_name" 2>/dev/null || true
  echo "✅ Switched to existing branch: $branch_name"
else
  git checkout main
  git pull origin main
  git checkout -b "$branch_name"
  echo "✅ Created branch: $branch_name"
fi
```

### 3. Update Task Status

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Update task file frontmatter:
```yaml
status: in-progress
updated: {current_datetime}
```

If GitHub issue number is known:
```bash
gh issue edit {number} --add-label "in-progress" 2>/dev/null || true
```

### 4. Run Researcher Agent

Launch the researcher agent to analyze the codebase and produce a change plan:

```yaml
Task:
  description: "Research task: {task_name}"
  subagent_type: "general-purpose"
  prompt: |
    You are the researcher agent. Read the agent definition at ccpm/agents/researcher.md and follow its instructions.

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

### 5. Present Plan

Display the researcher's change plan to the user:

```
📝 Implementation Plan for: {task_name}
═══════════════════════════════════════

{researcher_output}

───────────────────────────────────────
Proceed with implementation? (yes/no/modify)
```

- If **yes**: Continue to step 6
- If **modify**: Ask what to change, update plan, re-present
- If **no**: "Okay, branch `{branch_name}` is ready when you want to continue. Task status remains in-progress."

### 6. Implement

Execute the change plan step by step:
- Follow the dependency ordering from the researcher's plan
- Make changes file by file
- Commit after each logical unit of work with format: `Issue #{number}: {specific change}` (or `Task {id}: {change}` if no issue number)
- If any step is unclear or risky, ask the user before proceeding

### 7. Run Tests

Launch the test-runner agent:

```yaml
Task:
  description: "Test task: {task_name}"
  subagent_type: "general-purpose"
  prompt: |
    You are the test-runner agent. Read the agent definition at ccpm/agents/test-runner.md and follow its instructions.

    Run relevant tests for the changes made to implement: {task_name}

    Changed files:
    {list_of_changed_files}

    Run the project's test suite and report results.
    If no test infrastructure exists, note this and skip.
```

### 8. Summarize

Display final summary:

```
✅ Work completed on: {task_name}

Branch: {branch_name}
Commits: {commit_count}

Files changed:
  {list_of_files_with_change_type}

Test results:
  {test_summary}

Next steps:
  • Review changes: git diff main...HEAD
  • Run /pm:review for code analysis
  • Push branch: git push -u origin {branch_name}
  • Close task: /pm:issue-close {identifier}
```

### 9. Update Task Status

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Update task file frontmatter `updated` field with current datetime.

If tests passed, optionally note progress but do NOT auto-close the task.

## Error Handling

If any step fails, report clearly:
- "❌ {What failed}: {How to fix}"
- Never leave the task in an inconsistent state
- If implementation fails partway, summarize what was done and what remains

## Important Notes

Follow `/rules/datetime.md` for timestamps.
Follow `/rules/branch-operations.md` for git operations.
Follow `/rules/frontmatter-operations.md` for frontmatter updates.
The user is watching — explain what you're doing at each major step.
