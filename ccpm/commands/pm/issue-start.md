---
allowed-tools: Bash, Read, Write, LS, Task, Glob, Grep
---

# Issue Start

Begin work on a GitHub issue — either single-threaded (default) or with parallel agents.

## Usage
```
/pm:issue-start <issue_number> [--parallel]
```

## Required Rules

**IMPORTANT:** Before executing this command, read and follow:
- `.claude/rules/datetime.md` - For getting real current date/time
- `.claude/rules/branch-operations.md` - For git branch operations

## Quick Check

1. **Get issue details:**
   ```bash
   gh issue view $ARGUMENTS --json state,title,labels,body
   ```
   If it fails: "❌ Cannot access issue #$ARGUMENTS. Check number or run: gh auth login"

2. **Find local task file:**
   - First check if `.claude/epics/*/$ARGUMENTS.md` exists (new naming)
   - If not found, search for file containing `github:.*issues/$ARGUMENTS` in frontmatter (old naming)
   - If not found: "❌ No local task for issue #$ARGUMENTS. This issue may have been created outside the PM system."

3. **Check for uncommitted changes:**
   ```bash
   git status --porcelain
   ```
   If not empty: "⚠️ You have uncommitted changes. Commit or stash them before starting work."

4. **Determine mode:**
   - If `--parallel` flag is present: Use parallel mode (Section B below)
   - Otherwise: Use single-threaded mode (Section A below)

## Section A: Single-Threaded Mode (Default)

### A1. Create or Switch to Branch

```bash
# Extract issue title for branch name
issue_title=$(gh issue view $ARGUMENTS --json title -q .title)
slug=$(echo "$issue_title" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//' | sed 's/-$//' | head -c 50)
branch_name="task/$ARGUMENTS-$slug"

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

### A2. Update Status

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Update task file frontmatter:
```yaml
status: in-progress
updated: {current_datetime}
```

```bash
gh issue edit $ARGUMENTS --add-assignee @me --add-label "in-progress" 2>/dev/null || true
```

### A3. Begin Work

Read the task file and understand requirements, then begin implementation directly.

Display:
```
✅ Started work on issue #$ARGUMENTS

Branch: {branch_name}
Task: {task_name}

Ready to implement. The task requirements are:
{task_summary}
```

Proceed with implementation — read the task's acceptance criteria and technical details, then work through them.

## Section B: Parallel Mode (--parallel flag)

### B1. Check for Analysis

```bash
test -f .claude/epics/*/$ARGUMENTS-analysis.md || echo "❌ No analysis found for issue #$ARGUMENTS

Run: /pm:issue-analyze $ARGUMENTS first
Or remove --parallel to work single-threaded"
```
If no analysis exists, stop execution.

### B2. Create or Switch to Branch

Same as Section A1 above.

### B3. Read Analysis

Read `.claude/epics/{epic_name}/$ARGUMENTS-analysis.md`:
- Parse parallel streams
- Identify which can start immediately
- Note dependencies between streams

### B4. Setup Progress Tracking

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Create workspace structure:
```bash
mkdir -p .claude/epics/{epic_name}/updates/$ARGUMENTS
```

Update task file frontmatter `updated` field with current datetime.

### B5. Launch Parallel Agents

For each stream that can start immediately:

Create `.claude/epics/{epic_name}/updates/$ARGUMENTS/stream-{X}.md`:
```markdown
---
issue: $ARGUMENTS
stream: {stream_name}
agent: {agent_type}
started: {current_datetime}
status: in_progress
---

# Stream {X}: {stream_name}

## Scope
{stream_description}

## Files
{file_patterns}

## Progress
- Starting implementation
```

Launch agent using Task tool:
```yaml
Task:
  description: "Issue #$ARGUMENTS Stream {X}"
  subagent_type: "{agent_type}"
  prompt: |
    You are working on Issue #$ARGUMENTS in branch: {branch_name}

    Your stream: {stream_name}

    Your scope:
    - Files to modify: {file_patterns}
    - Work to complete: {stream_description}

    Requirements:
    1. Read full task from: .claude/epics/{epic_name}/{task_file}
    2. Work ONLY in your assigned files
    3. Commit frequently with format: "Issue #$ARGUMENTS: {specific change}"
    4. Update progress in: .claude/epics/{epic_name}/updates/$ARGUMENTS/stream-{X}.md
    5. Follow coordination rules in /rules/agent-coordination.md

    If you need to modify files outside your scope:
    - Check if another stream owns them
    - Wait if necessary
    - Update your progress file with coordination notes

    Complete your stream's work and mark as completed when done.
```

### B6. GitHub Assignment

```bash
gh issue edit $ARGUMENTS --add-assignee @me --add-label "in-progress" 2>/dev/null || true
```

### B7. Output

```
✅ Started parallel work on issue #$ARGUMENTS

Branch: {branch_name}

Launching {count} parallel agents:
  Stream A: {name} (Agent-1) ✓ Started
  Stream B: {name} (Agent-2) ✓ Started
  Stream C: {name} - Waiting (depends on A)

Progress tracking:
  .claude/epics/{epic_name}/updates/$ARGUMENTS/

Monitor with: /pm:epic-status {epic_name}
Sync updates: /pm:issue-sync $ARGUMENTS
```

## Error Handling

If any step fails, report clearly:
- "❌ {What failed}: {How to fix}"
- Continue with what's possible
- Never leave partial state

## Important Notes

Follow `/rules/datetime.md` for timestamps.
Follow `/rules/branch-operations.md` for git operations.
Keep it simple - trust that GitHub and file system work.