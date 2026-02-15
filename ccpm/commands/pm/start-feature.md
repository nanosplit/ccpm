---
allowed-tools: Bash, Read, Write, LS, Task, Glob, Grep
---

# Start Feature

End-to-end feature workflow: brainstorm PRD, create epic, decompose into tasks, and begin work on the first task.

## Usage
```
/pm:start-feature <feature-name>
```

## Required Rules

**IMPORTANT:** Before executing this command, read and follow:
- `.claude/rules/datetime.md` - For getting real current date/time
- `.claude/rules/branch-operations.md` - For git branch operations
- `.claude/rules/frontmatter-operations.md` - For frontmatter updates

## Preflight Checklist

Before proceeding, complete these validation steps.
Do not bother the user with preflight checks progress. Just do them and move on.

### Input Validation
1. **Validate feature name was provided:**
   - If not, tell user: "❌ Feature name required. Usage: /pm:start-feature <feature-name>"

2. **Validate feature name format:**
   - Must contain only lowercase letters, numbers, and hyphens
   - Must start with a letter
   - If invalid: "❌ Feature name must be kebab-case (lowercase letters, numbers, hyphens only). Examples: user-auth, payment-v2"

3. **Check for existing PRD:**
   - If `.claude/prds/$ARGUMENTS.md` exists, ask: "⚠️ PRD '$ARGUMENTS' already exists. Use existing PRD? (yes/no/overwrite)"
   - If **yes**: Skip to Step 2 (epic creation)
   - If **overwrite**: Continue with Step 1 (PRD creation)
   - If **no**: Stop execution

4. **Check for existing epic:**
   - If `.claude/epics/$ARGUMENTS/epic.md` exists, ask: "⚠️ Epic '$ARGUMENTS' already exists. Skip to task work? (yes/no)"
   - If **yes**: Skip to Step 5 (kick off first task)
   - If **no**: Continue normally

5. **Check for uncommitted changes:**
   ```bash
   git status --porcelain
   ```
   If not empty: "⚠️ You have uncommitted changes. Commit or stash them before starting a new feature."

6. **Verify directory structure:**
   - Ensure `.claude/prds/` and `.claude/epics/` exist or create them

## Instructions

### Step 1: Create PRD (Interactive)

You are a product manager creating a PRD for: **$ARGUMENTS**

#### 1a. Discovery
- Ask the user clarifying questions about the feature
- Understand the problem being solved
- Identify target users and use cases
- Gather constraints and requirements

**Keep the brainstorming focused and efficient.** Ask the most important questions first. Don't over-ask — 3-5 focused questions is usually enough.

#### 1b. Write PRD

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Save to `.claude/prds/$ARGUMENTS.md`:
```markdown
---
name: $ARGUMENTS
description: [Brief one-line description]
status: backlog
created: [Current ISO date/time]
---

# PRD: $ARGUMENTS

## Executive Summary
[Value proposition]

## Problem Statement
[What problem, why now]

## User Stories
[Primary personas and journeys with acceptance criteria]

## Requirements
### Functional Requirements
[Core features]

### Non-Functional Requirements
[Performance, security, scalability]

## Success Criteria
[Measurable outcomes]

## Constraints & Assumptions
[Limitations]

## Out of Scope
[What we're NOT building]

## Dependencies
[External and internal]
```

Confirm: "✅ PRD created: .claude/prds/$ARGUMENTS.md"

### Step 2: Create Epic

You are a technical lead converting the PRD into an implementation epic.

Read the PRD from `.claude/prds/$ARGUMENTS.md` and create the epic.

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Create `.claude/epics/$ARGUMENTS/epic.md`:
```markdown
---
name: $ARGUMENTS
status: backlog
created: [Current ISO date/time]
progress: 0%
prd: .claude/prds/$ARGUMENTS.md
github: [Will be updated when synced]
---

# Epic: $ARGUMENTS

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

Confirm: "✅ Epic created: .claude/epics/$ARGUMENTS/epic.md"

### Step 3: Decompose into Tasks

Break the epic into concrete, actionable tasks.

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

For each task, create `.claude/epics/$ARGUMENTS/{number}.md`:
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

Confirm: "✅ Created {count} tasks for epic: $ARGUMENTS"

### Step 4: Create Feature Branch

Follow `/rules/branch-operations.md`:

```bash
git checkout main
git pull origin main
git checkout -b feature/$ARGUMENTS
echo "✅ Created branch: feature/$ARGUMENTS"
```

### Step 5: Kick Off First Task

Identify the first ready task:
- No unmet dependencies (`depends_on` is empty or all deps are closed)
- Lowest task number among ready tasks
- Status is `open`

If no ready task found: "✅ Feature setup complete but no ready tasks found. Check task dependencies."

Otherwise, begin the `/pm:work-on` flow for the first task:

#### 5a. Update task status
```yaml
status: in-progress
updated: {current_datetime}
```

#### 5b. Run researcher agent

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

#### 5c. Present plan and implement

Display the change plan and ask for approval:
```
📝 Implementation Plan for: {task_name}
═══════════════════════════════════════
{researcher_output}
───────────────────────────────────────
Proceed with implementation? (yes/no/modify)
```

On approval, implement the changes following the plan. Commit with format: `Task {number}: {specific change}`

#### 5d. Run tests

Launch test-runner agent to validate changes:
```yaml
Task:
  description: "Test first task"
  subagent_type: "general-purpose"
  prompt: |
    You are the test-runner agent. Read the agent definition at ccpm/agents/test-runner.md.
    Run relevant tests for the changes made. Report results.
```

#### 5e. Summarize

```
✅ Feature kickoff complete: $ARGUMENTS

PRD: .claude/prds/$ARGUMENTS.md
Epic: .claude/epics/$ARGUMENTS/epic.md
Tasks: {count} created
Branch: feature/$ARGUMENTS

First task completed: {task_name}
  Files changed: {list}
  Tests: {pass/fail summary}

Remaining tasks: {count}

Next steps:
  • Continue work: /pm:work-on $ARGUMENTS/{next_task}
  • Sync to GitHub: /pm:epic-sync $ARGUMENTS
  • View progress: /pm:epic-status $ARGUMENTS
```

## Error Recovery

If any step fails:
- Clearly explain what went wrong
- Provide specific steps to fix the issue
- Never leave partial or corrupted files
- If PRD brainstorming needs to restart, the user can run `/pm:prd-edit $ARGUMENTS`

## Important Notes

- Steps 1-3 are interactive and require user input
- Step 5 transitions into the work-on flow
- The user is watching throughout — explain what you're doing at each phase
- Keep task count low: fewer, well-scoped tasks are better than many tiny ones
