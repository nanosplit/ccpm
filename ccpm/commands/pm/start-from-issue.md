---
allowed-tools: Bash, Read, Write, LS, Task, Glob, Grep
---

# Start From Issue

End-to-end feature workflow starting from an existing GitHub issue: fetch issue, generate PRD, create epic, decompose into tasks, optionally sync as sub-issues, and begin work on the first task.

## Usage
```
/pm:start-from-issue <issue_number> [feature-name]
```

## Required Rules

**IMPORTANT:** Before executing this command, read and follow:
- `.claude/rules/datetime.md` - For getting real current date/time
- `.claude/rules/branch-operations.md` - For git branch operations
- `.claude/rules/frontmatter-operations.md` - For frontmatter updates
- `.claude/rules/github-operations.md` - For GitHub CLI operations
- `.claude/rules/testing-discipline.md` - For testing tier selection

## Preflight Checklist

Before proceeding, complete these validation steps.
Do not bother the user with preflight checks progress. Just do them and move on.

### Input Validation
1. **Validate issue number was provided:**
   - Parse the first argument as the issue number
   - If not provided, tell user: "❌ Issue number required. Usage: /pm:start-from-issue <issue_number> [feature-name]"

2. **Fetch the GitHub issue:**
   ```bash
   gh issue view $ISSUE_NUMBER --json state,title,body,labels,url
   ```
   - If it fails: "❌ Cannot access issue #$ISSUE_NUMBER. Check the number or run: gh auth login"

3. **Check issue state:**
   - If issue state is `CLOSED`: "❌ Issue #$ISSUE_NUMBER is already closed. Reopen it first or use a different issue."

4. **Derive feature name:**
   - If the user provided a second argument, use that as the feature name
   - Otherwise, slugify the issue title: lowercase, replace non-alphanumeric with hyphens, collapse multiple hyphens, trim leading/trailing hyphens, truncate to 50 chars
   ```bash
   feature_name=$(echo "$ISSUE_TITLE" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//' | sed 's/-$//' | head -c 50)
   ```

5. **Validate feature name format:**
   - Must contain only lowercase letters, numbers, and hyphens
   - Must start with a letter
   - If invalid: "❌ Derived feature name '$feature_name' is invalid. Provide one explicitly: /pm:start-from-issue $ISSUE_NUMBER <feature-name>"

6. **Check for existing PRD:**
   - If `.claude/prds/$feature_name.md` exists, ask: "⚠️ PRD '$feature_name' already exists. Use existing PRD? (yes/no/overwrite)"
   - If **yes**: Skip to Step 2 (epic creation)
   - If **overwrite**: Continue with Step 1 (PRD creation)
   - If **no**: Stop execution

7. **Check for existing epic:**
   - If `.claude/epics/$feature_name/epic.md` exists, ask: "⚠️ Epic '$feature_name' already exists. Skip to task work? (yes/no)"
   - If **yes**: Skip to Step 6 (kick off first task)
   - If **no**: Continue normally

8. **Check for uncommitted changes:**
   ```bash
   git status --porcelain
   ```
   If not empty: "⚠️ You have uncommitted changes. Commit or stash them before starting a new feature."

9. **Verify directory structure:**
   - Ensure `.claude/prds/` and `.claude/epics/` exist or create them

## Instructions

### Step 1: Create PRD from Issue (Auto-Generated + Review)

You are a product manager creating a PRD from GitHub issue #$ISSUE_NUMBER for feature: **$feature_name**

#### 1a. Extract Issue Content

Parse the issue body, title, and labels fetched during preflight. Extract:
- Problem statement / motivation
- Requirements and acceptance criteria
- Any user stories or use cases mentioned
- Constraints, dependencies, or out-of-scope items
- Labels that indicate type (bug, feature, enhancement, etc.)

#### 1b. Write PRD

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Save to `.claude/prds/$feature_name.md`:
```markdown
---
name: $feature_name
description: [Brief one-line description from issue title]
status: backlog
created: [Current ISO date/time]
source_issue: $ISSUE_NUMBER
---

# PRD: $feature_name

> Auto-generated from GitHub issue #$ISSUE_NUMBER

## Executive Summary
[Value proposition derived from issue]

## Problem Statement
[Extracted from issue body]

## User Stories
[Derived from issue requirements/acceptance criteria]

## Requirements
### Functional Requirements
[Core features from issue]

### Non-Functional Requirements
[Performance, security, scalability — inferred or from issue]

## Success Criteria
[Measurable outcomes from issue acceptance criteria]

## Constraints & Assumptions
[From issue or inferred]

## Out of Scope
[Items explicitly excluded or reasonably out of scope]

## Dependencies
[External and internal]
```

#### 1c. Ask User to Review

Display the generated PRD and ask:
```
📝 PRD auto-generated from issue #$ISSUE_NUMBER
════════════════════════════════════════════════
{PRD content summary}
────────────────────────────────────────────────
Would you like to add or modify anything before proceeding? (yes/no)
```

If **yes**: Let the user provide additions/modifications, then update the PRD.
If **no**: Continue to Step 2.

Confirm: "✅ PRD created: .claude/prds/$feature_name.md"

### Step 2: Create Epic

You are a technical lead converting the PRD into an implementation epic.

Read the PRD from `.claude/prds/$feature_name.md` and create the epic.

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Get the issue URL from the fetched issue data (or construct it):
```bash
issue_url=$(gh issue view $ISSUE_NUMBER --json url -q .url)
```

Create `.claude/epics/$feature_name/epic.md`:
```markdown
---
name: $feature_name
status: backlog
created: [Current ISO date/time]
progress: 0%
prd: .claude/prds/$feature_name.md
github: $issue_url
---

# Epic: $feature_name

## Overview
[Technical summary]

## Architecture Decisions
[Key technical decisions]

## Technical Approach
[Implementation details by component]

## Implementation Strategy
[Phases, risk mitigation, testing]

## Task Breakdown Preview
[High-level task categories]

## Dependencies
[Technical dependencies]

## Success Criteria (Technical)
[Benchmarks and quality gates]
```

Note: The `github` field is set to the source issue URL immediately — no "will be updated" placeholder.

Confirm: "✅ Epic created: .claude/epics/$feature_name/epic.md (linked to issue #$ISSUE_NUMBER)"

### Step 3: Decompose into Tasks

Break the epic into concrete, actionable tasks.

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

For each task, create `.claude/epics/$feature_name/{number}.md`:
```markdown
---
name: [Task Title]
status: open
created: [Current ISO date/time]
updated: [Current ISO date/time]
github: [Will be updated when synced]
depends_on: []
parallel: true
conflicts_with: []
---

# Task: [Task Title]

## Description
[What needs to be done]

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Technical Details
[Implementation approach, files affected]

## Effort Estimate
- Size: XS/S/M/L/XL
```

After creating all tasks, update epic.md with a "Tasks Created" section.

**Guidelines:**
- Aim for as few tasks as possible (max 10)
- Each task should be completable in 1-3 days
- Mark parallel-safe tasks with `parallel: true`
- Set `depends_on` for tasks that require others to finish first

Confirm: "✅ Created {count} tasks for epic: $feature_name"

### Step 4: Offer to Sync Tasks as Sub-Issues

Ask the user:
```
🔗 Sync tasks to GitHub?
═══════════════════════
Created {count} tasks locally. Would you like to sync them as sub-issues
under issue #$ISSUE_NUMBER on GitHub? (yes/no)
```

If **yes**:

#### 4a. Check Remote Repository

Follow `/rules/github-operations.md` to ensure we're not syncing to the CCPM template:
```bash
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$remote_url" == *"automazeio/ccpm"* ]] || [[ "$remote_url" == *"automazeio/ccpm.git"* ]]; then
  echo "❌ ERROR: You're trying to sync with the CCPM template repository!"
  # ... (full check from github-operations.md)
fi
```

#### 4b. Detect Repository

```bash
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
REPO=$(echo "$remote_url" | sed 's|.*github.com[:/]||' | sed 's|\.git$||')
[ -z "$REPO" ] && REPO="user/repo"
```

#### 4c. Check for gh-sub-issue Extension

```bash
if gh extension list | grep -q "yahsan2/gh-sub-issue"; then
  use_subissues=true
else
  use_subissues=false
  echo "⚠️ gh-sub-issue not installed. Creating as regular issues linked to #$ISSUE_NUMBER."
fi
```

#### 4d. Create Sub-Issues

For each task file:
```bash
for task_file in .claude/epics/$feature_name/[0-9][0-9][0-9].md; do
  [ -f "$task_file" ] || continue

  task_name=$(grep '^name:' "$task_file" | sed 's/^name: *//')
  sed '1,/^---$/d; 1,/^---$/d' "$task_file" > /tmp/task-body.md

  if [ "$use_subissues" = true ]; then
    task_number=$(gh sub-issue create \
      --parent "$ISSUE_NUMBER" \
      --title "$task_name" \
      --body-file /tmp/task-body.md \
      --label "task,epic:$feature_name" \
      --json number -q .number)
  else
    task_number=$(gh issue create \
      --repo "$REPO" \
      --title "$task_name" \
      --body-file /tmp/task-body.md \
      --label "task,epic:$feature_name" \
      --json number -q .number)
  fi

  echo "$task_file:$task_number" >> /tmp/task-mapping.txt
done
```

#### 4e. Rename Task Files and Update References

Follow the same rename/reference-update pattern as `epic-sync.md` Step 3:
- Build old→new ID mapping
- Rename files from `001.md` → `{issue_id}.md`
- Update `depends_on` and `conflicts_with` references
- Update `github` field in each task's frontmatter
- Update `updated` timestamp

#### 4f. Update Epic Tasks Section

Update the "Tasks Created" section in epic.md with real issue numbers.

#### 4g. Create Mapping File

Create `.claude/epics/$feature_name/github-mapping.md` with issue mappings.

If **no**: Skip syncing — tasks remain local with sequential numbering.

### Step 5: Create Feature Branch

Follow `/rules/branch-operations.md`:

```bash
git checkout main
git pull origin main
git checkout -b feature/$feature_name
echo "✅ Created branch: feature/$feature_name"
```

### Step 6: Kick Off First Task

Identify the first ready task:
- No unmet dependencies (`depends_on` is empty or all deps are closed)
- Lowest task number among ready tasks
- Status is `open`

If no ready task found: "✅ Feature setup complete but no ready tasks found. Check task dependencies."

Otherwise, begin the `/pm:work-on` flow for the first task:

#### 6a. Update task status

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

```yaml
status: in-progress
updated: {current_datetime}
```

#### 6b. Run researcher agent

Launch researcher to analyze codebase and produce change plan:

```yaml
Task:
  description: "Research first task: {task_name}"
  subagent_type: "general-purpose"
  prompt: |
    You are the researcher agent. Read the agent definition at ccpm/agents/researcher.md and follow its instructions.

    Task file: {task_file_path}
    Task requirements:
    {task_description}

    Acceptance criteria:
    {acceptance_criteria}

    Analyze the codebase and produce a file-by-file change plan.
```

#### 6c. Present plan and implement

Display the change plan and ask for approval:
```
📝 Implementation Plan for: {task_name}
═══════════════════════════════════════
{researcher_output}
───────────────────────────────────────
Proceed with implementation? (yes/no/modify)
```

On approval, implement the changes following the plan. Commit with format: `Task {number}: {specific change}`

#### 6d. Run tests

Launch test-runner agent to validate changes:
```yaml
Task:
  description: "Test first task"
  subagent_type: "general-purpose"
  prompt: |
    You are the test-runner agent. Read the agent definition at ccpm/agents/test-runner.md.
    Run relevant tests for the changes made. Report results.
```

#### 6e. Summarize

```
✅ Feature kickoff complete: $feature_name (from issue #$ISSUE_NUMBER)

Source: Issue #$ISSUE_NUMBER — $issue_url
PRD: .claude/prds/$feature_name.md
Epic: .claude/epics/$feature_name/epic.md
Tasks: {count} created {synced_status}
Branch: feature/$feature_name

First task completed: {task_name}
  Files changed: {list}
  Tests: {pass/fail summary}

Remaining tasks: {count}

Next steps:
  • Continue work: /pm:work-on $feature_name/{next_task}
  • Sync to GitHub: /pm:epic-sync $feature_name
  • View progress: /pm:epic-status $feature_name
```

Where `{synced_status}` is either "(synced as sub-issues of #$ISSUE_NUMBER)" or "(local only)".

## Error Recovery

If any step fails:
- Clearly explain what went wrong
- Provide specific steps to fix the issue
- Never leave partial or corrupted files
- If the issue cannot be fetched, suggest checking `gh auth status`
- If PRD generation needs revision, the user can run `/pm:prd-edit $feature_name`

## Important Notes

- Steps 1-4 are interactive and require user input
- Step 6 transitions into the work-on flow
- The user is watching throughout — explain what you're doing at each phase
- Keep task count low: fewer, well-scoped tasks are better than many tiny ones
- The `github` field on the epic is populated immediately with the source issue URL
- Feature name is derived from the issue title unless explicitly provided
