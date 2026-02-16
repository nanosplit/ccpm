---
allowed-tools: Bash, Read, Write, LS
---

# Issue Close

Mark an issue as complete and close it on GitHub.

## Usage
```
/pm:issue-close <issue_number> [completion_notes]
```

## Required Rules

**IMPORTANT:** Before executing this command, read and follow:
- `.claude/rules/verification-before-completion.md` - Verification gate before closing

## Instructions

### 1. Verify Before Closing

**Follow `.claude/rules/verification-before-completion.md` — this gate is mandatory.**

Before closing any issue, verify the work is actually done:

1. **Read the task file** and extract acceptance criteria
2. **Run the project's test suite** (or relevant subset) and read the output
3. **Check each acceptance criterion** is satisfied with fresh evidence
4. If verification fails: "❌ Cannot close — verification failed: {what failed}. Fix the issues and try again."
5. If no acceptance criteria exist: at minimum confirm the build succeeds and relevant tests pass

Only proceed to close if verification passes.

### 2. Find Local Task File

First check if `.claude/epics/*/$ARGUMENTS.md` exists (new naming).
If not found, search for task file with `github:.*issues/$ARGUMENTS` in frontmatter (old naming).
If not found: "❌ No local task for issue #$ARGUMENTS"

### 3. Update Local Status

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Update task file frontmatter:
```yaml
status: closed
updated: {current_datetime}
```

### 4. Update Progress File

If progress file exists at `.claude/epics/{epic}/updates/$ARGUMENTS/progress.md`:
- Set completion: 100%
- Add completion note with timestamp
- Update last_sync with current datetime

### 5. Close on GitHub

Add completion comment and close:
```bash
# Add final comment
echo "✅ Task completed

$ARGUMENTS

---
Closed at: {timestamp}" | gh issue comment $ARGUMENTS --body-file -

# Close the issue
gh issue close $ARGUMENTS
```

### 6. Update Epic Task List on GitHub

Check the task checkbox in the epic issue:

```bash
# Get epic name from local task file path
epic_name={extract_from_path}

# Get epic issue number from epic.md
epic_issue=$(grep 'github:' .claude/epics/$epic_name/epic.md | grep -oE '[0-9]+$')

if [ ! -z "$epic_issue" ]; then
  # Get current epic body
  gh issue view $epic_issue --json body -q .body > /tmp/epic-body.md
  
  # Check off this task
  sed -i "s/- \[ \] #$ARGUMENTS/- [x] #$ARGUMENTS/" /tmp/epic-body.md
  
  # Update epic issue
  gh issue edit $epic_issue --body-file /tmp/epic-body.md
  
  echo "✓ Updated epic progress on GitHub"
fi
```

### 7. Update Epic Progress

- Count total tasks in epic
- Count closed tasks
- Calculate new progress percentage
- Update epic.md frontmatter progress field

### 8. Offer Branch Merge

Check if currently on a task branch:
```bash
current_branch=$(git branch --show-current)
```

If the branch matches the `task/*` pattern:

Ask the user:
```
🔀 Currently on branch: {current_branch}
   Merge this branch back to main? (yes/no)
```

If **yes**:
```bash
git checkout main
git pull origin main
git merge --no-ff "$current_branch" -m "Merge $current_branch: Issue #$ARGUMENTS completed"
git push origin main
git branch -d "$current_branch"
git push origin --delete "$current_branch" 2>/dev/null || true
echo "✅ Branch merged and cleaned up"
```

If **no**: "Okay, branch `{current_branch}` left as-is."

### 9. Output

```
✅ Closed issue #$ARGUMENTS
  Local: Task marked complete
  GitHub: Issue closed & epic updated
  Epic progress: {new_progress}% ({closed}/{total} tasks complete)
  Branch: {merged or left as-is}

Next: Run /pm:next for next priority task
```

## Important Notes

Follow `/rules/frontmatter-operations.md` for updates.
Follow `/rules/github-operations.md` for GitHub commands.
Follow `/rules/branch-operations.md` for merge operations.
Always sync local state before GitHub.